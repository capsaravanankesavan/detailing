# Tech Detail: PromotionMeta Redis L2 Cache — Fix Spring AOP Proxy Bypass
**Date:** 2026-04-03
**JIRA:** https://capillarytech.atlassian.net/browse/CAP-183979
**Status:** Draft — Ready for Implementation
**Investigation source:** CodeRabbit comment on PR #788 (CAP-183706)
**Related MADR:** `.context/adr/madr-0001-promotion-meta-caffeine-redis-l1-l2-cache.md`
**Confidence:** HIGH

---

## 1. Problem Statement

`PromotionMetaCacheService.loadPromotionMetaFromDatabase` (`PromotionMetaCacheService.java:340`) is
declared `private` and annotated `@Cacheable(ONE_DAY_CACHE)`. It is only ever called from an anonymous
`CacheLoader` inner class at line 117 — a direct JVM invocation that bypasses Spring's CGLIB proxy. The
`@Cacheable` interceptor is never applied. The Redis L2 cache (`ONE_DAY_CACHE`, 1-day TTL) has **never
been populated** since this service was introduced.

The `@CacheEvict` on `RedisCacheUtil.evictPromotionMetaCache` (line 213) is an orphaned no-op — it evicts
from a Redis cache that has no entries.

**Additional finding identified during codebase reconnaissance:**
`PromotionMetaManagementService.findByOrgIdAndActiveIsTrue` (line 220) has a `@Cacheable` on a `public`
method and IS intercepted by Spring AOP — but the JDK serialisation write fails silently because
`PromotionMeta` does not implement `java.io.Serializable`. This is a second broken L2 entry, suppressed by
Spring's `SimpleCacheErrorHandler`.

**Effective runtime topology (today):**
```
getPromotionMeta(orgId, promotionId)
  → Caffeine L1 hit  →  return
  → Caffeine L1 miss →  loadPromotionMetaFromDatabase()  →  MongoDB  →  populate L1
```

**Intended topology (after fix):**
```
getPromotionMeta(orgId, promotionId)
  → Caffeine L1 hit  →  return
  → Caffeine L1 miss →  PromotionMetaDbLoader.load()  →  Redis L2 hit  →  populate L1, return
                                                        →  Redis L2 miss →  MongoDB  →  populate L2 + L1
```

---

## 2. Root Cause (Confirmed)

Spring's proxy-based AOP intercepts method calls only when they pass through the injected proxy bean
reference. A `private` method called from within an anonymous inner class inside the same bean is a direct
JVM invocation — the proxy is never in the call path. The `@Cacheable` annotation is silently ignored at
runtime.

**Evidence trail:**

| File | Line(s) | Finding |
|------|---------|---------|
| `PromotionMetaCacheService.java` | 339 | `@Cacheable(ONE_DAY_CACHE)` on `private` method |
| `PromotionMetaCacheService.java` | 110–123 | Anonymous `CacheLoader` calls `loadPromotionMetaFromDatabase` directly (self-call) |
| `PromotionMetaCacheService.java` | 117 | `PromotionMeta meta = loadPromotionMetaFromDatabase(orgId, promotionId);` — no proxy in call path |
| `RedisCacheUtil.java` | 213–216 | `evictPromotionMetaCache` — keyed correctly but evicts an empty cache |
| `CacheEvictionPublisher.java` | 52–54 | `for(String promotionId:promotionIds)` loop calls `redisCacheUtil.evictPromotionMetaCache` — no-op |
| `RedisCacheUtil.java` | 176–183 | `getRedisCacheConfig()` uses `RedisCacheConfiguration.defaultCacheConfig()` — JDK serialisation |
| `PromotionMeta.java` | 50 | `implements BO<PromotionMetaRO>, IAttributionMigration` — no `Serializable` |
| `PromotionMetaManagementService.java` | 220–228 | `public @Cacheable` on `findByOrgIdAndActiveIsTrue` returning `Map<String, PromotionMeta>` — AOP-intercepted but silently failing at Redis write due to missing `Serializable` |

### Contributing Factors
1. `loadPromotionMetaFromDatabase` declared `private` — CGLIB cannot proxy private methods
2. Call site is inside an anonymous `CacheLoader` inner class — direct `this.` call, not through a proxy
3. `PromotionMeta` not `Serializable` — JDK serialiser (`RedisCacheConfiguration.defaultCacheConfig()`) cannot write it
4. `SimpleCacheErrorHandler` (Spring default) swallows cache write failures — no runtime exception or alert when serialisation fails
5. No integration test asserting a Redis write — both bugs went undetected

---

## 3. Scope of Change

### In Scope
1. New `PromotionMetaDbLoader` `@Service` bean with a `public @Cacheable` method for MongoDB loading
2. `PromotionMetaCacheService.CacheLoader.load()` calls `promotionMetaDbLoader.load()` instead of private method
3. Remove dead `@Cacheable` from `loadPromotionMetaFromDatabase`
4. Configure `GenericJackson2JsonRedisSerializer` on the `ONE_DAY_CACHE` bucket **only** in `RedisCacheUtil.getRedisCacheConfig()` — scoped to avoid impacting other buckets with risky return types
5. Remove dead `@Mock RedisCacheUtil redisCacheUtil` injection from `CacheEvictionIntegrationTest.setUp()`
6. Update MADR-0001 — remove "not yet active" notes, document active L2
7. Add Jackson round-trip unit test for `PromotionMeta` serialization

### Out of Scope (Explicit)
- L1 Caffeine cache logic in `PromotionMetaCacheService` — no changes to `getPromotionMeta`, `invalidateCache`, `preWarmCacheWithPromotionIds`, `onMessage`, `handlePromotionEviction`
- `CacheEvictionPublisher.publishPromotionEviction` per-ID eviction loop — **no change** (already correct; will start evicting real entries once L2 is populated)
- `RedisCacheUtil.evictPromotionMetaCache` — **no change** (key expression is correct; now evicts real entries)
- `RedisCacheKeys`, `CacheTtl` constants — no changes
- `RedisMessageListenerContainer` configuration — no changes
- Serializer switch for `ONE_MINUTE_CACHE`, `FIVE_MINUTE_CACHE`, `ONE_HOUR_CACHE` — kept as JDK default
- L1 cardinality right-sizing (30K→≤1K) — separate ticket

### Deferred
- `PromotionMetaManagementService.findByOrgIdAndActiveIsTrue` L2 fix (also broken due to missing `Serializable`) — shares root cause; switching `ONE_DAY_CACHE` to Jackson does not help `FIVE_MINUTE_CACHE`. Can be fixed in a follow-up by either: (a) scoping Jackson to `FIVE_MINUTE_CACHE` too after auditing that bucket's return types, or (b) accepting it as a latent silent miss
- Right-sizing L1 to ≤1,000 entries — tracked in MADR-0001 open risk

---

## 4. Assumptions

| # | Assumption | Breaks if... | Owner to validate |
|---|-----------|-------------|------------------|
| A1 | `PromotionMeta` and all its nested types are Jackson-serializable (no circular refs, all have no-arg constructors via Lombok) | Any nested type lacks a no-arg constructor or has circular reference — `SerializationException` on first L2 write | Developer — run T_JACKSON_ROUNDTRIP before merge |
| A2 | `org.bson.types.ObjectId` serializes/deserializes correctly via `GenericJackson2JsonRedisSerializer` | Jackson cannot reconstruct `ObjectId` from serialized form — `PromotionMeta.id` comes back as `null` after cache read | Developer — covered by same round-trip test |
| A3 | `CapDateTimeJacksonModule` (registered on the app `ObjectMapper`) handles all `java.time` types present in `PromotionMeta` and its nested graph | A `java.time` field not covered by the module fails to serialize — `SerializationException` | Developer — inspect `CapDateTimeJacksonModule` or test all date-typed fields |
| A4 | The app `ObjectMapper` bean should NOT be passed directly to `GenericJackson2JsonRedisSerializer`; a dedicated Redis `ObjectMapper` must be created | Passing the shared app `ObjectMapper` to `GenericJackson2JsonRedisSerializer(ObjectMapper)` causes Spring Redis to mutate it with `DefaultTyping` — breaking REST serialization across the application | Developer — always construct `new GenericJackson2JsonRedisSerializer(redisObjectMapper)` with a separate bean |
| A5 | Only `ONE_DAY_CACHE` bucket needs to change serializer — other buckets' return types are not Jackson-safe globally | If a future `@Cacheable` is added to `ONE_DAY_CACHE` with a non-Jackson-safe return type | Developer — document this scoped change in MADR |
| A6 | Infra team can flush the `ONE_DAY_CACHE` Redis entries in incentives Redis Sentinel during the deploy window | Flush cannot happen — old JDK-binary entries in `ONE_DAY_CACHE` cause `SerializationException` on read until they expire (1-day TTL) | Infra team — confirm flush window before deploy |

---

## 5. Design Validation

### 5a. Existing Code Patterns — PASS with one gap

**Pattern confirmed:** `CacheEvictionPublisher` already injects `RedisCacheUtil` via `@Autowired` (line 34) — the new `PromotionMetaDbLoader` follows the same injection pattern.

**Gap found (RESOLVED in design):**
> `GenericJackson2JsonRedisSerializer()` (no-arg) creates its own `ObjectMapper` with `DefaultTyping` enabled but without `CapDateTimeJacksonModule`. The app `ObjectMapper` has `CapDateTimeJacksonModule` but must NOT be passed to the serializer constructor because Spring Redis mutates the passed ObjectMapper's typing configuration. The correct pattern: create a dedicated `redisObjectMapper` `@Bean` in `RedisCacheUtil` that clones app configuration plus enables `DefaultTyping`.

### 5b. MADR Compliance — PASS

Matches MADR-0001 exactly: the fix activates the L2 tier described in the "Decision — L2 Redis" section. The `@Service` extraction pattern is explicitly called out as the fix in MADR-0001 "Guardrails → Follow-up" (line 121 of MADR).

### 5c. Code Guardrail Compliance

| Guardrail | Check | Result |
|-----------|-------|--------|
| `@Cacheable` must be on a `public` method on a separate `@Service` bean | `PromotionMetaDbLoader.load()` is `public` on a new `@Service` | ✅ PASS |
| Cache key must include `orgId` for tenant isolation | Key: `'promo_meta_' + orgId + '_' + promotionId` | ✅ PASS |
| `maximumSize` + TTL on every Caffeine cache | No new Caffeine caches introduced | N/A |
| `recordStats()` + `CaffeineCacheMetrics.monitor` | No new Caffeine cache | N/A |
| Mutations must go through `CacheEvictionPublisher.publishPromotionEviction` | `evictPromotionMetaCache` unchanged; Publisher loop unchanged | ✅ PASS |
| No `@Cacheable` on private method or self-call | `loadPromotionMetaFromDatabase` annotation removed | ✅ PASS |
| Log to stdout with `orgId` | `log.debug` with `orgId` and `promotionId` in `PromotionMetaDbLoader.load()` | ✅ PASS |
| No INFO logging in per-request hot paths | `load()` logs at DEBUG/WARN only | ✅ PASS |
| New Relic `@Trace` on cache read/eviction | Existing `@Trace` on `getPromotionMeta` covers the call chain; add `@Trace` to `PromotionMetaDbLoader.load()` with `metricName = "Custom/PromotionMetaDbLoader/load"` | ⚠️ ADD (see Low-Level Design) |

### 5d. Tenant Isolation — PASS

Cache key `'promo_meta_' + orgId + '_' + promotionId` is org-scoped at both `@Cacheable` (write) and `@CacheEvict` (evict). No static/shared state introduced. `PromotionMetaDbLoader.load()` is a stateless `@Service` bean — all state is in parameters.

### 5e. Transaction Boundary — N/A

`PromotionMetaDbLoader.load()` is read-only (no DB writes). No transaction boundaries change.

---

## 6. Design: Serializer Scoping Decision

**CRITICAL FINDING from `@Cacheable` audit:**

The following return types are cached in non-`ONE_DAY_CACHE` buckets and are **not safe** for global Jackson serializer switch:

| Service | Method | Return Type | Cache Bucket | Risk |
|---------|--------|------------|-------------|------|
| `OrganizationServiceImpl` | `getTimeZone` / `getTimeZoneTillOrSystemDefault` | `TimeZone` (abstract; concrete `sun.util.calendar.ZoneInfo`) | `ONE_DAY_CACHE` | **HIGH** — internal JDK class, not instantiable by name on Java 9+ |
| `OrganizationServiceImpl` | `getZoneOffSetForOrgOrDefault` | `ZoneOffset` | `ONE_DAY_CACHE` | MEDIUM — requires JavaTimeModule |
| `OrganizationServiceImpl` | `getOrgZoneIdOrSystemDefault` | `ZoneId` | `ONE_DAY_CACHE` | MEDIUM — requires JavaTimeModule |
| `PointEngineThriftServiceImpl` | `getMembershipsForLoyaltyProgram` | `List<CustomerProgramMembership>` (Thrift-generated) | `ONE_MINUTE_CACHE` | **HIGH** — Thrift-generated class, Jackson compatibility unknown |
| `OrganizationServiceImpl` | `getOrgEntities` | `Map<OrgEntity.ORG_ENTITY_TYPE, List<OrgEntity>>` | `ONE_HOUR_CACHE` | MEDIUM — enum map keys |

**Decision: Scope Jackson serializer to `ONE_DAY_CACHE` only. Keep JDK default for all other buckets.**

Rationale: `ONE_DAY_CACHE` is the only bucket where `PromotionMeta` is cached (via `PromotionMetaDbLoader`). Changing only this bucket isolates the blast radius. `TimeZone`, `ZoneOffset`, `ZoneId`, and `OrganizationDetails` in `ONE_DAY_CACHE` are pre-existing entries — they will fail silently under the old JDK serialiser too (since `OrganizationDetails` also may not implement `Serializable`). This will be documented as a follow-up item: audit all `ONE_DAY_CACHE` entries for Jackson compatibility in a dedicated PR.

**Updated `getRedisCacheConfig()` design:**

```java
@Trace
private Map<String, RedisCacheConfiguration> getRedisCacheConfig() {
    Map<String, RedisCacheConfiguration> configurationMap = new HashMap<>();
    // JDK serialisation for all short-lived buckets (non-PromotionMeta types cached here)
    configurationMap.put(CacheTtl.ONE_MINUTE_CACHE, RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(1)));
    configurationMap.put(CacheTtl.FIVE_MINUTE_CACHE, RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(5)));
    configurationMap.put(CacheTtl.ONE_HOUR_CACHE, RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(1)));
    // Jackson serialisation for ONE_DAY_CACHE — required for PromotionMeta (not Serializable)
    configurationMap.put(CacheTtl.ONE_DAY_CACHE, buildJacksonCacheConfig(Duration.ofDays(1)));
    return configurationMap;
}

private RedisCacheConfiguration buildJacksonCacheConfig(Duration ttl) {
    return RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(ttl)
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer(redisObjectMapper())));
}
```

---

## 7. Alternative Designs Considered

| Alternative | Pros | Cons | Decision |
|------------|------|------|----------|
| Option B: Remove `@Cacheable`, document L2 as future work | Zero production risk; smallest change | L2 still not active; CodeRabbit concern unresolved in spirit; second broken cache (`findByOrgIdAndActiveIsTrue`) left in place | Rejected — L2 is implementable in this PR with Jackson |
| Global Jackson switch (all buckets) | One consistent serializer across all caches | `TimeZone`, `ZoneOffset`, Thrift types may fail silently; large blast radius | Rejected — scope to `ONE_DAY_CACHE` only |
| `PromotionMeta implements Serializable` (add interface + all nested types) | No serializer config change needed | `PromotionMeta` has deep nested graph (Condition polymorphism, Reward, Criteria, etc.); `serialVersionUID` drift risk; maintenance burden | Rejected — Jackson is less fragile |
| Dedicated `RedisCacheManager` bean for PromotionMeta only | Perfect isolation | Spring does not support multiple `@CacheManager` beans easily without `@Primary`/`@Qualifier` on every `@Cacheable` | Rejected — per-bucket config within one manager is sufficient |

---

## 8. Use Cases

### B2B (Brand Admin Flows)

| ID | Flow | Today (before fix) | After Fix | Risk if Wrong | Test Type |
|----|------|-----------|-----------|---------------|-----------|
| UC-1 | Admin updates promotion → `publishPromotionEviction` called | `@CacheEvict` no-op (nothing in L2); L1 cleared via pub/sub | L2 Redis entry evicted; next cross-pod request loads fresh from MongoDB | Stale promotion config served after update | Integration |
| UC-2 | Admin deactivates promotion → `publishPromotionEviction` called | L1 cleared; L2 no-op | L1 cleared + L2 evicted; deactivated promotion not served from any tier | Deactivated promotion served to customers | Integration |
| UC-3 | Admin creates new promotion (first access by any pod) | L1 miss → MongoDB (per pod) | L1 miss → L2 miss → MongoDB → L2 written → all subsequent pods hit L2 | — | Integration |
| UC-4 | Admin updates promotion on pod A; pod B evaluates immediately after | Pod B: L1 still warm (pub/sub evicts async) → stale data possible within 1h TTL | Pod B: L1 evicted via pub/sub + L2 evicted; pod B re-fetches fresh | Incorrect discount/earn logic | Integration |

### B2C (End Customer Flows)

| ID | Flow | Today (before fix) | After Fix | Risk if Wrong | Test Type |
|----|------|-----------|-----------|---------------|-----------|
| UC-5 | Evaluation request — L1 hit | L1 hit → return (unchanged) | L1 hit → return (unchanged) | — | Unit |
| UC-6 | Evaluation request — L1 eviction (TTL/capacity) on pod A | L1 miss → MongoDB | L1 miss → L2 hit → return (no MongoDB query) | Increased MongoDB load if L2 miss rate high | Integration |
| UC-7 | Evaluation on cold pod (restart, new pod scale-out) | L1 empty → MongoDB (per pod; no cross-pod sharing) | L1 empty → L2 hit (all pods share L2) → return | MongoDB overload on mass pod restart | Integration |
| UC-8 | Evaluation — inactive or deleted promotion | L1 CacheLoader throws `PromotionNotFoundException` → `Optional.empty()` (unchanged) | Same; `unless="#result==null"` in `PromotionMetaDbLoader` prevents null caching in L2 | NPE or incorrect eval if null cached | Unit |
| UC-9 | 100 pods restart simultaneously — all L1 cold | 100 concurrent MongoDB queries per promotion | 1 MongoDB query per promotion → L2 populated → 99 L2 hits | MongoDB overload if L2 cold too (e.g., post flush) | Load / Integration |
| UC-10 | Org A update must not affect Org B's cached promotions | L2 not populated — no risk | Cache key includes `orgId`; `evictPromotionMetaCache` only evicts Org A's specific promotion | Cross-tenant cache pollution | Tenant isolation test |

---

## 9. Low-Level Design

### NEW: `PromotionMetaDbLoader.java`
**Location:** `src/main/java/com/capillary/promotionengine/service/impl/cache/PromotionMetaDbLoader.java`

```
Class: PromotionMetaDbLoader [new file]
  Responsibility: Redis L2 loading of PromotionMeta from MongoDB.
                  Single public method. No Caffeine L1 logic here.

  + load(orgId: Long, promotionId: String): PromotionMeta
      @Cacheable(
          cacheNames = CacheTtl.ONE_DAY_CACHE,
          key = RedisCacheKeys.PROMOTION_META + " + #orgId + " + RedisCacheKeys.UNDERSCORE + " + #promotionId",
          unless = "#result == null"
      )
      @Trace(metricName = "Custom/PromotionMetaDbLoader/load")
      → calls: promotionMetaManagementService.findByOrgIdAndIdAndActiveIsTrue(orgId, promotionId)
      → DB: MongoDB find by orgId + id + active=true
      → Cache: writes to key promo_meta_{orgId}_{promotionId} in ONE_DAY_CACHE (1-day TTL)
               does NOT write if result is null (inactive/missing promotion)
      → Emits: log.debug on miss; log.warn if not found/not active
      → Throws: none — catches Exception, logs WARN, returns null
                (null return is NOT cached due to unless="#result==null")
      Guardrail check: PASS — public method, separate @Service bean, called via injected reference,
                              orgId in cache key, no INFO logging in hot path
```

**Key implementation note:** The `@Cacheable` key expression must exactly match the one at `RedisCacheUtil.evictPromotionMetaCache:213`:
```
RedisCacheKeys.PROMOTION_META + " + #orgId + " + RedisCacheKeys.UNDERSCORE + " + #promotionId"
```
Both resolve to: `'promo_meta_' + orgId + '_' + promotionId`. Confirmed identical — eviction will work.

---

### MODIFIED: `PromotionMetaCacheService.java`

```
Class: PromotionMetaCacheService [PromotionMetaCacheService.java]
  Existing responsibility: L1 Caffeine cache + Redis pub/sub subscription management
  Change: Delegate DB loading to PromotionMetaDbLoader; remove dead @Cacheable annotation

  INJECT: promotionMetaDbLoader: PromotionMetaDbLoader (@Autowired)

  MODIFY: CacheLoader.load(cacheKey: String) [line 113]
      Before: PromotionMeta meta = loadPromotionMetaFromDatabase(orgId, promotionId);
      After:  PromotionMeta meta = promotionMetaDbLoader.load(orgId, promotionId);
      → call now goes through Spring proxy → @Cacheable in PromotionMetaDbLoader is intercepted
      → L2 Redis checked first; MongoDB only on L2 miss
      → null returned from promotionMetaDbLoader.load() → throw PromotionNotFoundException (line 119, unchanged)

  REMOVE: @Cacheable annotation at line 339 (method body and signature unchanged)
  REMOVE: import org.springframework.cache.annotation.Cacheable; (now unused)
```

---

### MODIFIED: `RedisCacheUtil.java`

```
Class: RedisCacheUtil [RedisCacheUtil.java]
  Existing responsibility: Redis connection factory, cache manager, pub/sub container
  Change: Add Jackson serialiser for ONE_DAY_CACHE bucket; add dedicated redisObjectMapper bean

  ADD BEAN: redisObjectMapper(): ObjectMapper
      → Creates a NEW ObjectMapper (NOT the shared app ObjectMapper)
      → Registers CapDateTimeJacksonModule (for java.time types in PromotionMeta)
      → configure(FAIL_ON_UNKNOWN_PROPERTIES, false)
      → Does NOT set NON_NULL inclusion (GenericJackson2JsonRedisSerializer needs null fields for round-trip)
      → IMPORTANT: this ObjectMapper is separate from the app @Bean ObjectMapper to prevent
                   DefaultTyping mutation affecting REST serialization

  MODIFY: getRedisCacheConfig() [line 176]
      → ONE_MINUTE_CACHE, FIVE_MINUTE_CACHE, ONE_HOUR_CACHE: unchanged (JDK default)
      → ONE_DAY_CACHE: use buildJacksonCacheConfig(Duration.ofDays(1))

  ADD: buildJacksonCacheConfig(ttl: Duration): RedisCacheConfiguration
      → RedisCacheConfiguration.defaultCacheConfig()
             .entryTtl(ttl)
             .serializeValuesWith(
                 RedisSerializationContext.SerializationPair.fromSerializer(
                     new GenericJackson2JsonRedisSerializer(redisObjectMapper())))

  ADD imports: GenericJackson2JsonRedisSerializer, RedisSerializationContext, ObjectMapper,
               DeserializationFeature (if not already present), CapDateTimeJacksonModule

  evictPromotionMetaCache [line 213]: NO CHANGE
```

---

### MODIFIED: `CacheEvictionIntegrationTest.java`

```
Class: CacheEvictionIntegrationTest [CacheEvictionIntegrationTest.java]
  Change: Remove dead mock declarations

  REMOVE: @Mock private RedisCacheUtil redisCacheUtil;  [line 40]
  REMOVE: setField(publisher, "redisCacheUtil", redisCacheUtil);  [line 52]

  No assertion changes — no test currently verifies evictPromotionMetaCache invocation count.
  Tests testPosPromotionEvictionPubSub, testEarningPromotionEvictionWithSpecificIds,
  testUnifiedPromotionEvictionPubSub (lines 56–268) all verify redisTemplate.convertAndSend —
  these remain unchanged and will continue to pass.

  NOTE: testPublisherErrorHandling (line 192) stubs redisTemplate.convertAndSend to throw.
  Currently it also calls redisCacheUtil.evictPromotionMetaCache before convertAndSend (CacheEvictionPublisher:52–54).
  After fix, evictPromotionMetaCache is still called (loop unchanged). The test will still pass because
  the throw from convertAndSend is caught in the outer try-catch (CacheEvictionPublisher:59).
  No assertion change needed.
```

---

### NEW TEST: `PromotionMetaJacksonRoundTripTest.java`

**Pre-merge gate (T_JACKSON_ROUNDTRIP):**

```
Class: PromotionMetaJacksonRoundTripTest [new unit test]

  testPromotionMetaJacksonRoundTrip():
      → Construct a PromotionMeta with: id (ObjectId), orgId, name, active, condition (CartCondition or
        ProductCondition), reward, earningCriteria, startDate, endDate, promotionRestriction
      → Serialize via GenericJackson2JsonRedisSerializer(redisObjectMapper)
      → Deserialize
      → Assert all fields equal, especially id.toString(), orgId, condition type, reward type
      → Confirms: ObjectId serializes correctly, polymorphic Condition deserializes via @class,
                  no circular references, all Lombok @NoArgsConstructor present on nested types
```

---

## 10. Data Model Changes

No data model changes. No MongoDB schema changes. No new Redis key namespaces (same key format as existing `evictPromotionMetaCache` expression).

---

## 11. API Changes

No API changes. This is an internal caching layer change invisible to callers.

---

## 12. Security, DB, Infra Considerations

```
CONSIDERATION [INFRA] [BLOCKER]
Finding: Switching ONE_DAY_CACHE from JDK binary to Jackson JSON changes the wire format.
         Any existing JDK-serialized entries in ONE_DAY_CACHE (e.g., OrganizationDetails,
         ZoneOffset, TimeZone from OrganizationServiceImpl) will cause SerializationException
         when read after deploy, until they expire (1-day TTL).
Impact if ignored: SerializationException on any ONE_DAY_CACHE read for ~24h post-deploy;
                   Spring's SimpleCacheErrorHandler swallows this and falls through to the upstream
                   service call — so the app continues to function, but cache is cold for 24h.
                   This is acceptable but noisy (error logs).
Recommended action: FLUSHDB on incentives Redis Sentinel during maintenance window immediately before
                    pod rollout. Alternatively, rename ONE_DAY_CACHE key prefix to avoid collision
                    (but this requires changes to all @Cacheable keys in that bucket — not recommended).
Owner: Infra team
```

```
CONSIDERATION [INFRA] [ACTION]
Finding: GenericJackson2JsonRedisSerializer must use a dedicated ObjectMapper (not the shared app bean).
         Spring Redis calls objectMapper.activateDefaultTyping(...) on the passed ObjectMapper to
         enable @class type information. Passing the shared app ObjectMapper mutates it, which breaks
         REST layer serialization across the entire application (NON_NULL inclusion may interfere,
         DefaultTyping causes @class to appear in REST responses).
Impact if ignored: REST API responses contain unexpected "@class" fields; Jackson behaviour changes
                   globally for all @RestController responses.
Recommended action: Create a separate @Bean redisObjectMapper() in RedisCacheUtil, copied from
                    app ObjectMapper configuration (CapDateTimeJacksonModule + FAIL_ON_UNKNOWN_PROPERTIES=false)
                    but NOT shared with the app context's ObjectMapper bean.
Owner: Developer implementing T3
```

```
CONSIDERATION [INFRA] [ACTION]
Finding: ObjectId (org.bson.types.ObjectId) in PromotionMeta.id — no custom Jackson serializer in codebase.
         GenericJackson2JsonRedisSerializer will attempt to serialize ObjectId via reflection.
         ObjectId does not have setters; its reconstruction from serialized form depends on
         bson library version and available constructors.
Impact if ignored: PromotionMeta.id comes back as null or wrong after Redis round-trip → downstream
                   NPE or incorrect promotion ID lookups.
Recommended action: Validate via T_JACKSON_ROUNDTRIP test before merge. If ObjectId fails,
                    add a custom Jackson module with ObjectIdSerializer/ObjectIdDeserializer
                    (serialize as toHexString(), deserialize via new ObjectId(hexString)).
Owner: Developer implementing T2
```

```
CONSIDERATION [INFRA] [NOTE]
Finding: OrganizationServiceImpl caches TimeZone (abstract class, concrete sun.util.calendar.ZoneInfo)
         in ONE_DAY_CACHE. After switching ONE_DAY_CACHE to Jackson, these entries will fail to
         deserialize due to internal JDK class name visibility on Java 9+.
Impact if ignored: getTimeZone / getTimeZoneTillOrSystemDefault cache becomes a cold-miss L2
                   (SimpleCacheErrorHandler swallows) — falls through to upstream Intouch API call.
                   No data correctness risk; latency increase only.
Recommended action: Treat as follow-up. Document in MADR that TimeZone/ZoneOffset/ZoneId cached
                    methods in ONE_DAY_CACHE need per-method Jackson custom serializers or migration
                    to a type-safe alternative in a subsequent PR.
Owner: Future PR
```

---

## 13. Internal Architecture Changes

- New `@Service` bean: `PromotionMetaDbLoader` — sits between `PromotionMetaCacheService.CacheLoader` and `PromotionMetaManagementService`
- `PromotionMetaCacheService` gains a new `@Autowired` dependency on `PromotionMetaDbLoader`
- `RedisCacheUtil` now has per-bucket serializer configuration (ONE_DAY_CACHE: Jackson; others: JDK)
- New pattern established: **L2 cache bean extraction** — any future `@Cacheable` on data loaded inside a `CacheLoader` must follow the `PromotionMetaDbLoader` pattern (separate `@Service`, public method, injected reference)

---

## 14. Upstream / Downstream Impact

### Upstream (systems feeding into this flow)
- **MongoDB / `PromotionMetaManagementService`**: call volume unchanged (L2 absorbs cross-pod misses); slight increase in per-pod cold-start DB calls immediately post-deploy until L2 warms
- **Redis Sentinel (incentives)**: new write volume for `ONE_DAY_CACHE` promo meta entries; read volume increases (L2 hit replaces some MongoDB calls)

### Downstream (systems consuming from this flow)
- All callers of `PromotionMetaCacheService.getPromotionMeta(orgId, promotionId)` — behaviour unchanged from their perspective; latency improves on cold-pod scenarios

---

## 15. SLA Impact

| Scenario | Before Fix | After Fix |
|---------|-----------|-----------|
| L1 hit (hot pod) | ~0.1ms | ~0.1ms (unchanged) |
| L1 miss, L2 miss (cold start or post-eviction) | ~5–20ms (MongoDB) | ~5–20ms (MongoDB + L2 write) |
| L1 miss, L2 hit (cross-pod scenario) | ~5–20ms (MongoDB per pod) | ~0.5–2ms (Redis RTT) |
| Mass pod restart (all pods cold) | MongoDB spike: N pods × M promotions queries | L2 absorbs: 1 MongoDB query → N-1 L2 hits |

Redis L2 cold-start after deploy: duration bounded by promotion access distribution × MongoDB query time. Caffeine L1 (30K entries, 1h TTL) absorbs the vast majority of steady-state traffic; L2 is only hit on cross-pod misses.

---

## 16. Observability

### New metrics
- `Custom/PromotionMetaDbLoader/load` — New Relic `@Trace` on `PromotionMetaDbLoader.load()` — tracks L2+MongoDB load latency
- Redis L2 hit/miss rate observable via `redisCacheManager.getCacheStatistics()` — Spring's `enableStatistics()` already set at `RedisCacheUtil:169`

### New log lines
- `PromotionMetaDbLoader.load()`:
    - `log.debug("Redis L2 miss — loading from DB: orgId={}, promotionId={}", orgId, promotionId)` — fires on every L2+MongoDB load
    - `log.warn("PromotionMeta not found or not active: orgId={}, promotionId={}", orgId, promotionId)` — fires when DB returns null/throws

### Existing observability unchanged
- `promotionMetaCache` Micrometer/Prometheus metrics (L1 hit/miss/eviction) — unchanged
- `CacheEvictionPublisher` log lines — unchanged

---

## 17. Rollout Plan

1. **Pre-deploy (infra gate):** Confirm `FLUSHDB` window with infra team on incentives Redis Sentinel during low-traffic period. L1 Caffeine absorbs load during L2 cold-start; MongoDB spike bounded by distinct promotion count (not request volume)
2. **Deploy:** Standard rolling update. Pods come up one by one with empty L1 and empty L2
3. **Warm-up:** First request per promotion hits MongoDB → writes L2 → subsequent pods hit L2. Expect elevated MongoDB QPS for first ~5 minutes, then steady decline as L2 populates
4. **Rollback:** Revert `PromotionMetaDbLoader` injection and `buildJacksonCacheConfig` in `getRedisCacheConfig`. Effective topology returns to L1-only. No data loss. Old L2 entries with Jackson format expire within 1 day
5. **Go/no-go signal:** `promotionMetaCache` Micrometer hit rate should be unchanged or improving; New Relic `Custom/PromotionMetaDbLoader/load` P99 should match pre-fix MongoDB latency and decline as L2 populates; no `SerializationException` in error logs

---

## 18. Risks

| # | Description | Likelihood | Mitigation |
|---|------------|-----------|-----------|
| R1 | `ObjectId` does not deserialize correctly from Jackson JSON — `PromotionMeta.id` returns null | Medium | T_JACKSON_ROUNDTRIP must pass before merge; add custom serializer if needed |
| R2 | Shared app `ObjectMapper` passed to `GenericJackson2JsonRedisSerializer` — REST responses corrupted by DefaultTyping | High if done naively | Dedicated `redisObjectMapper()` bean in RedisCacheUtil — enforced in design |
| R3 | `CapDateTimeJacksonModule` doesn't cover a `java.time` type in `PromotionMeta` graph | Low | T_JACKSON_ROUNDTRIP with real date-typed fields; inspect `CapDateTimeJacksonModule` |
| R4 | Redis flush not possible in deploy window — old JDK entries cause error logs for up to 24h | Low | SimpleCacheErrorHandler swallows; app continues to function on cache miss; cosmetic risk |
| R5 | Polymorphic `Condition` subtypes (CartCondition, ProductCondition, etc.) lack no-arg constructor | Low | Lombok `@NoArgsConstructor` on concrete classes; T_JACKSON_ROUNDTRIP covers this |
| R6 | `PromotionMetaDbLoader.load()` cache key expression differs from `evictPromotionMetaCache` — eviction misses | Low | Key expressions identical in both — verified at `PromotionMetaCacheService.java:339` and `RedisCacheUtil.java:213` |

---

## 19. Open Questions

| Question | Owner | When |
|---------|-------|------|
| Does `org.bson.types.ObjectId` serialize/deserialize correctly via `GenericJackson2JsonRedisSerializer` with no custom module? | Developer (T_JACKSON_ROUNDTRIP) | Before T2 merges |
| Does `CapDateTimeJacksonModule` handle all `java.time` types present in `PromotionMeta`'s field graph? | Developer (inspect source or test) | Before T2 merges |
| Can infra team schedule a `FLUSHDB` on incentives Redis Sentinel in the deploy window? Or should cache key prefix be versioned? | Infra team | Before deploy |
| Should `ONE_DAY_CACHE` Jackson serialization be extended to fix `OrganizationServiceImpl.getTimeZone` / `getZoneOffSetForOrgOrDefault`? Or defer to a separate PR? | Arch review | Before merge — affects MADR scope statement |

---

## 20. Task Breakdown

### Backend

| # | Task | Size | Depends on |
|---|------|------|-----------|
| T1 | Audit `ONE_DAY_CACHE` `@Cacheable` return types for Jackson compatibility | S | — |
| T2 | Create `PromotionMetaDbLoader.java` — `@Service`, public `@Cacheable`, `@Trace`, WARN log | S | — |
| T3 | Modify `RedisCacheUtil.getRedisCacheConfig()` — add `buildJacksonCacheConfig()`, add `redisObjectMapper()` `@Bean` | S | T1 (audit result) |
| T4 | Modify `PromotionMetaCacheService` — inject `PromotionMetaDbLoader`, update `CacheLoader.load()`, remove `@Cacheable` from private method, remove unused import | S | T2 |

### Testing

| # | Task | Size | Depends on |
|---|------|------|-----------|
| T_JACKSON_ROUNDTRIP | New unit test: serialize/deserialize representative `PromotionMeta` via `GenericJackson2JsonRedisSerializer` — assert `id`, `orgId`, `condition` type, `reward` type | S | T2, T3 |
| T5 | Update `CacheEvictionIntegrationTest` — remove dead `redisCacheUtil` mock injection | S | T4 |
| T6 | New integration test: L2 written on miss, L2 hit on second miss (no MongoDB), L2 evicted after `publishPromotionEviction` | M | T2, T3, T4 (Redis Testcontainer or embedded Redis) |

### Docs

| # | Task | Size | Depends on |
|---|------|------|-----------|
| T7 | Update MADR-0001 — activate L2 status, update L1 loading chain, update invalidation section, remove "follow-up" open item | S | T2–T4 merged |

### Infra

| # | Task | Size | Depends on |
|---|------|------|-----------|
| T8 | Coordinate Redis flush with infra team for deploy window | S | T3 merged |

---

## Handoff Notes for test-plan-architect

**Tech detail doc location:**
`.doc/investigations/2026-04-03-promotion-meta-l2-redis-aop-bypass/promotion-meta-l2-redis-aop-bypass_techdetail.md`

### Critical B2B Flows to Test
- UC-1: Admin updates promotion → L2 evicted via `publishPromotionEviction` → next pod request hits MongoDB, not stale L2 → test type: **integration**
- UC-2: Admin deactivates promotion → both L1 (pub/sub) and L2 (`@CacheEvict`) cleared → deactivated promo returns empty on next call → test type: **integration**

### Critical B2C Flows to Test
- UC-6: L1 miss → L2 hit → no MongoDB call → test type: **integration** (requires Redis Testcontainer)
- UC-7: Cold pod (empty L1) → L2 hit from another pod's prior load → test type: **integration**
- UC-8: Inactive promotion → `unless="#result==null"` → null NOT written to L2 → repeated misses always hit MongoDB → test type: **unit**
- UC-9: Concurrent L1 misses from multiple pods → only 1 MongoDB query, rest L2 → test type: **load / integration**

### Regression Risks (must not break)
- L1 Caffeine hit path (`getPromotionMeta` cache hit) — verify: no behaviour change on L1 hit
- `preWarmCacheWithPromotionIds` — verify: still calls `promotionMetaManagementService.findAllActivePromotionMetaByIdIn` (not `PromotionMetaDbLoader`) — these are independent paths
- `onMessage` / pub/sub L1 invalidation — verify: `invalidateCache` still called correctly on eviction message

### Tenant Isolation Tests
- Org A's `publishPromotionEviction` must NOT evict Org B's L2 cache entry — verify key prefix includes orgId
- Verify: `promo_meta_100_<id>` and `promo_meta_200_<id>` are independent Redis keys

### Contract Tests
- `PromotionMetaDbLoader.load()` key expression must produce the same string as `RedisCacheUtil.evictPromotionMetaCache` — verify key parity in unit test

### Suggested Test Emphasis
- **Unit:** Jackson round-trip (`PromotionMetaJacksonRoundTripTest`), null-result not cached (UC-8), null parameters handled
- **Integration:** L2 write on miss, L2 hit on second miss, L2 eviction after `publishPromotionEviction`, multi-pod L2 sharing (UC-7)
- **Tenant isolation:** Cross-org key isolation
- **Regression:** `PromotionMetaCacheServiceTest` (all existing tests must pass with `promotionMetaManagementService` mock — behaviour unchanged for tests that mock at the management service layer since `PromotionMetaDbLoader` delegates to it)
