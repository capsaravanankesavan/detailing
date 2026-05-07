# Tech Detail: Expiry Date Change Job — MongoDB Load Reduction
**Date:** 2026-04-28
**Author:** Saravanan Kesavan
**Status:** Final
**Investigation Doc:** `CAP-184618_expiry_job_tuning_analysis.md`
**Confidence:** HIGH (inherited and validated against worktree source)

---

## 1. Problem Statement

The `ExpiryDateChangeJobListener` processes bulk expiry-date updates for three MongoDB
collections (`customerIssuedPromotion`, `customerEarnedPromotion`, `codeBasedPromotion`).
When a brand updates 300 promotion end dates in a single batch, up to 600 RMQ messages land
on `PROMOTION_EXPIRY_UPDATE_QUEUE`. All 6 pods concurrently pull and execute multi-iteration
bulk update loops, producing sustained CPU and IOPS spikes on MongoDB.

Two orthogonal optimisations are in scope:
1. **Query inefficiency** — `AbstractExpiryDateChangeDao.updateValidTill()` uses an `_id`
   range predicate on `updateMulti`, which can span documents from other tenants.
2. **Uncontrolled cross-pod concurrency** — no mechanism limits how many jobs run in
   parallel across pods; the value is `pods × consumers-per-queue` and cannot be tuned per
   queue without impacting all other listeners.

---

## 2. Root Cause (Confirmed)

**Optimisation 1:** `AbstractExpiryDateChangeDao.getAggregation()` returns only the last
`_id` of the batch. The subsequent `updateMulti` uses `_id > startId AND _id <= endId` as
a range, where `endId` is an ObjectId whose value spans other tenants' documents in global
time-ordered ID space, causing unnecessary index traversal.

**Optimisation 2:** `PROMOTION_EXPIRY_UPDATE_QUEUE`'s listener uses the global Spring AMQP
factory — configured via `spring.rabbitmq.listener.simple.*` — which applies uniformly to
all queues. No per-queue concurrency control is possible without a dedicated factory.

### Contributing Factors

1. `group().last(_id)` discards the full list of matching IDs, forcing the update to re-derive
   the set via range instead of targeting exact documents.
2. The global `consumers-per-queue=1` / `prefetch=1` setting cannot be decoupled per queue
   without a named container factory.
3. No distributed semaphore to cap total concurrent jobs across all pods.

---

## 3. Scope of Change

### In Scope
- Replace range-based `updateMulti` with `$in` on primary key in `AbstractExpiryDateChangeDao`.
- Same fix in `CustomerEarnedCollectionDynamicExpiryDateChangeDao` (override).
- Dedicated `SimpleRabbitListenerContainerFactory` for `PROMOTION_EXPIRY_UPDATE_QUEUE`.
- Redis atomic semaphore in `ExpiryDateChangeJobFacade` to cap cross-pod concurrency.
- Semaphore counter reconciliation in `ExpiryDateChangeHelper` (post-shard-loop).
- New `countByStatus` query on `ExpiryDateChangeJobDao`.
- One new env-var-backed property in `application.properties` (`EXPIRY_MAX_GLOBAL_CONCURRENT_JOBS`, default 3).

### Out of Scope (Explicit)
- `CustomerIssuedCollectionStaticExpiryDateChangeDao` — inherits abstract class; zero changes.
- `CustomerEarnedCollectionStaticExpiryDateChangeDao` — same.
- `CodeBasedPromotionCollectionStaticExpiryDateChangeDao` — same.
- All `ExpiryDateChangeProcessor` implementations — interface and impls unchanged.
- All MongoDB collection indexes — no changes.
- RMQ queue/exchange/binding declarations — no changes.
- `PromotionFacade.updatePromotion()` duplicate-job protection — already correct.
- Pre-existing `ExpiryDateChangeJobFacadeTest` missing `@Mock ExpiryDateChangeJobDao` — not fixed here.

### Deferred
- Per-queue dead-letter queue configuration for exhausted retries.
- Observability dashboard for semaphore counter.

---

## 4. Assumptions

| # | Assumption | Breaks if… | Owner to validate |
|---|-----------|-------------|------------------|
| 1 | `EXPIRY_DATE_CHANGE_BATCH_SIZE=2000` in prod | Larger value → $in list exceeds memory budget or query plan changes | Saravanan — confirm prod env var |
| 2 | `"cacheEvictionRedisTemplate"` bean is always present when app starts | Bean removed or renamed → NPE on injection | Implementer — verified in `RedisCacheUtil.java:204` ✅ |
| 3 | `RabbitListenerContainerFactoryConfigurer` is auto-configured by Spring Boot AMQP starter | Starter version doesn't provide it → must manually wire retry | Implementer — verify on Spring AMQP version in pom.xml |
| 4 | Redis Sentinel is reachable from all pods | Redis outage → semaphore acquire fails; fail-open behaviour required | Implementer — confirm with infra team |
| 5 | In USHC, orgs are distributed across multiple MongoDB shards | Single-shard assumption in reconciliation → wrong counter | Confirmed multi-shard in USHC per `.context/infra.md` |
| 6 | `@Scheduled(fixedRate=1h)` in `ExpiryDateChangeHelper` runs on all pods | Scheduler disabled on some pods → reconciliation not triggered | Implementer — verify `@EnableScheduling` scope |

---

## 5. Design Validation

### Gaps vs Existing Patterns

**GAP 1 [BLOCKER] — Redis bean qualifier**
- Proposed: `@Autowired RedisTemplate<String, String> redisTemplate` (no qualifier)
- Required: `@Autowired @Qualifier("cacheEvictionRedisTemplate") RedisTemplate<String, String> redisTemplate`
- Evidence: `RedisCacheUtil.java:204` — only one `RedisTemplate<String,String>` bean exists, named `"cacheEvictionRedisTemplate"`. `CacheEvictionPublisher.java:28-29` shows the correct pattern.
- Resolution: Add `@Qualifier("cacheEvictionRedisTemplate")` on the injected field in `ExpiryDateChangeJobFacade`.

**GAP 2 [HIGH] — Shard-aware semaphore reconciliation**
- Proposed: Reconcile counter inside `ExpiryDateChangeJobService.markAllJobAsExpiredIfNotRunning()`
- Problem: `ExpiryDateChangeHelper.updateExpiryDateChangeJobs()` calls this method inside a per-shard loop (`ExpiryDateChangeHelper.java:37-44`) with MDC shard key set. `countByStatus(RUNNING)` inside the method executes against only the current shard. Counter is reset to last shard's count — incorrect in multi-shard deployments (USHC).
- Resolution: Move reconciliation to `ExpiryDateChangeHelper.updateExpiryDateChangeJobs()` AFTER the shard loop. Sum `countByStatus(RUNNING)` across all shards, then call a new `ExpiryDateChangeJobFacade.reconcileSemaphoreCounter(long totalRunning)` method that sets the Redis counter. This keeps Redis dependency out of the helper.

**GAP 3 [MEDIUM] — Dedicated factory retry interceptor**
- Proposed: Custom factory inherits global retry policy
- Problem: `grep -rn RabbitListenerContainerFactoryConfigurer` returns no results in the codebase. Spring Boot wires retry interceptor only to the auto-configured default factory. Custom `@Bean("expiryDateChangeContainerFactory")` gets no retry by default.
- Resolution: Inject `RabbitListenerContainerFactoryConfigurer configurer` into `SpringAmqpConfig.expiryDateChangeContainerFactory(...)` and call `configurer.configure(factory, connectionFactory)` first, then set custom overrides (`setConcurrentConsumers`, `setMaxConcurrentConsumers`, `setPrefetchCount`). This is the Spring Boot-idiomatic approach.

**GAP 4 [MEDIUM] — Redis failure resilience**
- Proposed: `tryAcquireSemaphore` throws if Redis unavailable
- Problem: Redis outage → all expiry jobs fail retry loop → messages dead-lettered after 10 attempts (~17 min window). 600 queued messages would be permanently lost.
- Resolution: Wrap Redis operations in `tryAcquireSemaphore` with `try/catch`. On exception, log WARN and return `true` (fail-open). Jobs process normally; MongoDB concurrency control is temporarily lost, but no data loss.

**GAP 5 [LOW] — Stale `ObjectId endId` variable**
- `AbstractExpiryDateChangeDao.java:60`: `ObjectId endId;` is declared before the loop.
- Must be removed in the new implementation. `batchIds` is extracted inline from `mappedResults`.

### MADR Compliance
MADR 0001 (PromotionMeta two-tier cache) — not affected. This change does not touch
`PromotionMetaCacheService`, `CacheEvictionPublisher`, or any cache keys. ✅

### Guardrail Compliance (`.context/code.md`)

| Guardrail | Check | Result |
|---|---|---|
| **GC Health — pre-size collections** | `List<ObjectId> batchIds` cast from MongoDB `BasicDBList` (already sized) | PASS |
| **GC Health — no unbounded static maps** | Semaphore uses a single Redis counter key, not a local collection | PASS |
| **Caching — Redis via `@Qualifier`** | `"cacheEvictionRedisTemplate"` required | FAIL → fixed by Gap 1 |
| **Observability — include `orgId` in logs** | `log.info("Semaphore acquired jobId={}")` lacks orgId | FAIL → add `orgId` to semaphore log lines |
| **MongoDB — `lastUpdatedOn` must be set on every write** | `update.set(LAST_UPDATED_ON, now)` unchanged in both DAOs | PASS |
| **MongoDB — cross-org queries never issued** | `$in(batchIds)` uses pre-filtered IDs; `orgId` added as defensive filter | PASS |
| **Logging — no INFO in per-request hot paths** | Semaphore logs are per-job (not per-request); INFO is acceptable | PASS |

---

## 6. Alternative Designs Considered

| Alternative | Pros | Cons | Decision |
|------------|------|------|----------|
| **Keep `group().last()`, extract all IDs with a separate projection query** | Familiar aggregation shape | Two aggregation calls per batch = 2× index reads | Rejected — more expensive |
| **No `group()`, use `match+sort+limit+project(_id)` returning 1000 slim docs** | No in-memory accumulation; slightly lower memory | Java must iterate 1000 result maps to collect IDs; slightly more code change | Deferred — valid but adds more diff without meaningful gain |
| **Redisson RSemaphore instead of INCR/DECR** | Built-in fairness, TTL, lease renewal | New dependency (`redisson`), more complex config | Rejected — `spring-data-redis` already present; INCR/DECR is sufficient |
| **MongoDB-based semaphore (count RUNNING docs before processing)** | No Redis dependency | MongoDB I/O per job start; cross-shard coordination complex; not atomic | Rejected — Redis is faster and already available |

---

## 7. Use Cases

### B2B (Brand Admin Flows)

| ID | Flow | Today | After Fix | Risk | Test Type |
|----|------|-------|-----------|------|-----------|
| UC-1 | Brand updates 1 promotion end date → 2 jobs (ISSUED + EARNED) queued → processed → `impactedCount` matches actual documents | Range scan on `_id`, extra index traversal | `$in` point lookups; same correctness, lower IOPS | Wrong `impactedCount` if batchIds extraction fails | Integration |
| UC-2 | Brand updates 300 promotions → 600 messages → all 6 pods process concurrently | All 6 pods run simultaneously → MongoDB spike | Semaphore caps active jobs to `maxGlobalConcurrentJobs`; others retry | Deadlock if maxJobs=0 (misconfiguration) | Manual / load test |
| UC-3 | Brand updates same promotion end date twice while first job is OPEN | Second update blocked with `UPDATE_EXPIRY_DATE_NOT_ALLOWED` | Behaviour unchanged | None | Unit (existing) |
| UC-4 | Pod crashes mid-job (semaphore counter inflated by 1) | N/A | Counter healed by `ExpiryDateChangeHelper` reconciliation after ≤1 hour | If reconciliation doesn't run (scheduler off), counter stays inflated | Integration |
| UC-5 | `maxGlobalConcurrentJobs` slots full; new job arrives | N/A | RuntimeException thrown → RMQ retry with exponential backoff → eventually processes | If max-attempts exhausted before slot opens → message lost | Unit |
| UC-6 | Promotion has 0 documents in the collection (new promotion with no issuances) | Loop exits immediately (empty `mappedResults`) | Same — `CollectionUtils.isEmpty(batchIds)` exits loop | None | Unit |
| UC-7 | Multi-shard env (USHC): pods across shards running jobs | N/A | Semaphore counter is global (Redis is shared); cross-shard job count correctly limited | If Redis connection routing differs per shard — verify same Redis instance | Manual in USHC env |

### B2C (End Customer Flows)

| ID | Flow | Today | After Fix | Risk | Test Type |
|----|------|-------|-----------|------|-----------|
| UC-8 | Customer queries active promotions during job execution → sees old `validTill` for documents not yet processed in the current batch | Batch in progress; some docs updated, some not | Same eventual consistency window; no change | Customer sees stale expiry until batch reaches their document | N/A (acceptable) |
| UC-9 | Customer queries active promotions AFTER job completes → sees updated `validTill` | Correct | Correct — `impactedCount` on job equals number of documents updated | Wrong if `$in` criteria is mis-formed | Integration |

---

## 8. Low-Level Design

### 8a. `AbstractExpiryDateChangeDao.java`

```
Class: AbstractExpiryDateChangeDao  [dao/AbstractExpiryDateChangeDao.java]
Existing responsibility: Cursor-paginated bulk updateMulti loop for all ExpiryDateChange DAOs
Change: Aggregation returns all IDs in batch; updateMulti uses $in on primary key

Constants:
  REMOVE: public static final String END_ID = "last_id";
  ADD:    public static final String BATCH_IDS = "ids";

+ getAggregation(ExpiryDateChangeJob job): Aggregation
    Pipeline:
      match: _id > job.startId, promotionId = job.promotionId, orgId = job.orgId
      sort:  _id ASC  ← REMOVE (redundant; covered index already sorted)
      limit: batchSize
      group: push(_id) as "ids"            ← was last(_id) as "last_id"
      project: "ids"                       ← was "last_id"
    DB: IXSCAN on {orgId, promotionId, _id} — covered query, zero heap I/O
    Guardrail: PASS — index-covered, no WiredTiger doc pages loaded

+ updateValidTill(ExpiryDateChangeJob job, PromotionMeta meta): void
    Loop change at line 67:
      REMOVE: ObjectId endId;  [line 60 — stale declaration]
      REMOVE: endId = (ObjectId) mappedResults.get(0).get(END_ID);
      ADD:    List<ObjectId> batchIds = (List<ObjectId>) mappedResults.get(0).get(BATCH_IDS);
              if (CollectionUtils.isEmpty(batchIds)) break;
      REMOVE: Long count = updateValidTillDate(job, meta, endId, zoneOffset);
      ADD:    Long count = updateValidTillDate(job, meta, batchIds, zoneOffset);
      CHANGE: expiryDateChangeJob.setStartId(endId)
           → expiryDateChangeJob.setStartId(batchIds.get(batchIds.size() - 1));
    Guardrail: PASS

+ updateValidTillDate(ExpiryDateChangeJob job, PromotionMeta meta,
                       List<ObjectId> batchIds, ZoneOffset zoneOffset): Long  [was ObjectId endId]
    Criteria change:
      REMOVE: andOperator(orgId.is, promotionId.is, _id.gt(startId), _id.lte(endId))
      ADD:    Criteria.where(ID_COLUMN).in(batchIds)
              .and(ORG_ID_COLUMN).is(job.getOrgId())   ← defensive, zero I/O cost
    DB: updateMulti — _id $in (batchIds), orgId = Y
        Uses primary _id index; O(batchSize) point lookups
    Emits: log.info("Time taken to update {} documents is {}", count, timeTakenToUpdate)  [unchanged]
    Guardrail: PASS
```

### 8b. `CustomerEarnedCollectionDynamicExpiryDateChangeDao.java`

```
Class: CustomerEarnedCollectionDynamicExpiryDateChangeDao
Existing responsibility: Override updateValidTillDate() with AggregationUpdate for dynamic validity
Change: Signature change + Criteria change only; AggregationUpdate pipeline is UNCHANGED

+ updateValidTillDate(ExpiryDateChangeJob job, PromotionMeta meta,
                       List<ObjectId> batchIds, ZoneOffset zoneOffset): Long  [was ObjectId endId]
    Criteria change (lines 55-59):
      REMOVE: andOperator(orgId.is, promotionId.is, _id.gt(startId), _id.lte(endId))
      ADD:    Criteria.where(ID_COLUMN).in(batchIds)
              .and(ORG_ID_COLUMN).is(job.getOrgId())
    Rest of AggregationUpdate (DateFromParts + ArithmeticOperators.Add) is UNCHANGED
    DB: updateMulti with AggregationUpdate pipeline on _id $in (batchIds)
    Guardrail: PASS
```

### 8c. `SpringAmqpConfig.java`

```
Class: SpringAmqpConfig  [queue/SpringAmqpConfig.java]
Existing responsibility: Queue, exchange, binding bean declarations
Change: Add named container factory for PROMOTION_EXPIRY_UPDATE_QUEUE

Purpose: PROTECTIVE ISOLATION — not tuning. RabbitMQ is shared with redemption and other
flows whose future throughput demands may push the global prefetch/consumers settings higher.
This factory hard-pins the expiry queue at prefetch=1 / consumers=1 so it is never affected
by global broker config changes.

+ expiryDateChangeContainerFactory(
      ConnectionFactory connectionFactory,
      RabbitListenerContainerFactoryConfigurer configurer
      // No @Value params — values are intentionally fixed, not operator-tunable.
      // The point is isolation, not flexibility.
  ): SimpleRabbitListenerContainerFactory
    @Bean("expiryDateChangeContainerFactory")
    Steps:
      1. factory = new SimpleRabbitListenerContainerFactory()
      2. configurer.configure(factory, connectionFactory)
           ← MUST be called first — inherits global retry interceptor, message converter,
             error handler. No RabbitListenerContainerFactoryConfigurer usage existed anywhere
             in the codebase before this; omitting silently drops retry policy.
             Spring Boot 2.6.6 / Spring AMQP 2.4.x — configurer is auto-configured ✅
      3. factory.setConcurrentConsumers(1)    ← hard-pinned
      4. factory.setMaxConcurrentConsumers(1) ← prevents dynamic scale-up
      5. factory.setPrefetchCount(1)          ← hard-pinned
    Guardrail: PASS
```

### 8d. `ExpiryDateChangeJobListener.java`

```
Class: ExpiryDateChangeJobListener  [queue/ExpiryDateChangeJobListener.java]
Change: One-line — add containerFactory to @RabbitListener

@RabbitListener(
    queues = {SpringAmqpConfig.PROMOTION_EXPIRY_UPDATE_QUEUE},
    containerFactory = "expiryDateChangeContainerFactory")   ← ADD
```

### 8e. `ExpiryDateChangeJobFacade.java`

```
Class: ExpiryDateChangeJobFacade  [service/impl/ExpiryDateChange/ExpiryDateChangeJobFacade.java]
Existing responsibility: Orchestrate job lifecycle (find → RUNNING → process → COMPLETED/ERRORED)
Change: Add Redis semaphore acquire/release wrapping the processing block

New fields:
  private static final String SEMAPHORE_KEY = "promotion_engine:expiry_job:running_count";

  @Autowired
  @Qualifier("cacheEvictionRedisTemplate")          ← REQUIRED — see Gap 1
  private RedisTemplate<String, String> redisTemplate;

  @Value("${expiry.date.change.max.global.concurrent.jobs:3}")
  private int maxGlobalConcurrentJobs;

+ tryAcquireSemaphore(String jobId, Long orgId): boolean
    try {
        Long current = redisTemplate.opsForValue().increment(SEMAPHORE_KEY);
        if (current != null && current <= maxGlobalConcurrentJobs) {
            log.info("Semaphore acquired orgId={} jobId={} count={}/{}", orgId, jobId, current, maxGlobalConcurrentJobs);
            return true;
        }
        redisTemplate.opsForValue().decrement(SEMAPHORE_KEY);
        log.info("Semaphore limit hit orgId={} jobId={} count={}/{}", orgId, jobId, current, maxGlobalConcurrentJobs);
        return false;
    } catch (Exception e) {
        log.warn("Semaphore Redis error orgId={} jobId={} — failing open", orgId, jobId, e);
        return true;   ← fail-open: jobs process normally if Redis unavailable
    }
    Throws: never — exceptions caught internally
    Guardrail: PASS — orgId in log line per .context/code.md

+ releaseSemaphore(String jobId, Long orgId): void
    try {
        Long remaining = redisTemplate.opsForValue().decrement(SEMAPHORE_KEY);
        log.info("Semaphore released orgId={} jobId={} remaining={}", orgId, jobId, remaining);
    } catch (Exception e) {
        log.warn("Semaphore release Redis error orgId={} jobId={}", orgId, jobId, e);
    }
    Throws: never

+ reconcileSemaphoreCounter(long totalRunning): void         ← called by ExpiryDateChangeHelper
    try {
        long safe = Math.max(0, totalRunning);
        redisTemplate.opsForValue().set(SEMAPHORE_KEY, String.valueOf(safe));
        log.info("Semaphore reconciled to actual running count: {}", safe);
    } catch (Exception e) {
        log.warn("Semaphore reconcile Redis error count={}", totalRunning, e);
    }
    Throws: never

Modified changeExpiryForPromotion():
  ExpiryDateChangeJob job = expiryDateChangeJobService.findById(...)
  if (!shouldProceed(job)) return;                          ← unchanged
  if (!tryAcquireSemaphore(jobId, job.getOrgId())) {
      throw new RuntimeException("Expiry job concurrency limit reached, will retry");
                                                            ← RMQ retry kicks in
  }
  try {
      ... existing logic unchanged ...
  } catch (Exception e) {
      ... existing catch unchanged ...
      throw e;
  } finally {
      releaseSemaphore(jobId, job.getOrgId());              ← always releases acquired slot
  }
  Guardrail: PASS
```

### 8f. `ExpiryDateChangeHelper.java`

```
Class: ExpiryDateChangeHelper  [util/ExpiryDateChangeHelper.java]
Existing responsibility: @Scheduled job to expire stale OPEN/RUNNING jobs (every 1 hour, all shards)
Change: After shard loop, sum RUNNING counts across shards and reconcile semaphore counter

New field:
  @Autowired
  private ExpiryDateChangeJobFacade expiryDateChangeJobFacade;

Modified updateExpiryDateChangeJobs():
  // Existing shard loop — UNCHANGED:
  for (String shardKey : allShardKeys) {
      MDC.put(SHARD_KEY, shardKey);
      try { expiryDateChangeJobService.markAllJobAsExpiredIfNotRunning(); }
      finally { MDC.remove(SHARD_KEY); }
  }
  // NEW — reconcile after all shards processed:
  long totalRunning = 0;
  for (String shardKey : allShardKeys) {
      MDC.put(SHARD_KEY, shardKey);
      try { totalRunning += expiryDateChangeJobDao.countByStatus(BatchStatus.RUNNING); }
      finally { MDC.remove(SHARD_KEY); }
  }
  expiryDateChangeJobFacade.reconcileSemaphoreCounter(totalRunning);

New field:
  @Autowired
  private ExpiryDateChangeJobDao expiryDateChangeJobDao;

Guardrail: PASS — orgId not available here (cross-org), reconcile log is sufficient
```

### 8g. `ExpiryDateChangeJobDao.java`

```
Interface: ExpiryDateChangeJobDao  [dao/ExpiryDateChangeJobDao.java]
Change: Add one derived query method

+ countByStatus(BatchStatus status): long
  Spring Data derived query — uses @CompoundIndex(def = "{'status' : 1}") on ExpiryDateChangeJob
  → index-covered count query, no heap I/O
  No custom @Query needed — Spring Data derives: db.expiryDateChangeJob.countDocuments({status: ?})
```

### 8h. `application.properties`

```
# Expiry date change job — cross-pod concurrency throttle
# Default 3 = half of 6 pods. Raises to 6 to disable throttle; lower to 2 only with
# retry window review (10 attempts × exponential backoff = ~17 min total window).
expiry.date.change.max.global.concurrent.jobs=${EXPIRY_MAX_GLOBAL_CONCURRENT_JOBS:3}

# NOTE: No per-queue prefetch/consumers properties.
# The dedicated factory hard-pins concurrentConsumers=1 and prefetchCount=1.
# These are fixed, not operator-tunable — the factory's purpose is protective isolation
# from global broker setting changes, not flexibility.
```

---

## 9. Data Model Changes

No changes to any MongoDB collection schema, documents, or indexes.

`ExpiryDateChangeJob.java` BO unchanged. `ExpiryDateChangeJobDao.java` gets a new derived
method (`countByStatus`) — this is query-only, no schema change.

---

## 10. API Changes

No API changes. This is a pure internal background job processing path.

---

## 11. Security

No new attack surface is introduced. This change is entirely internal to background job
processing — no new API endpoints, no new user input surfaces, no auth/role check changes.

| Check | Finding |
|-------|---------|
| New API endpoints requiring auth/role checks | **None** |
| PII in new log lines | **None** — `orgId` and `jobId` are operational identifiers (not personal data); semaphore count is a plain integer |
| New fields storing PII | **None** — Redis key stores a plain integer counter; `batchIds` are MongoDB internal ObjectIds |
| Token / credential handling | **None** — Redis connection uses existing Lettuce client configuration; no new credentials introduced |
| Tenant data leak risk | **Low / Acceptable** — semaphore counter is intentionally global (not per-org); it limits total concurrency, not per-tenant throughput. `$in(batchIds)` IDs are pre-filtered by `orgId` in the aggregation phase — no cross-tenant document access is possible |
| User-supplied input in new code paths | **None** — `batchIds` are produced internally by the aggregation pipeline; `maxGlobalConcurrentJobs` is a server-side env var, not user input |

---

## 12. DB & Infra Considerations

```
CONSIDERATION [INFRA] [ACTION]
Finding: Dedicated container factory adds one additional AMQP connection channel per pod.
Impact: Negligible — shared RabbitMQ server; 6 pods × 1 channel = 6 additional channels.
Action: None required.
Owner: tech-detailer

CONSIDERATION [INFRA] [ACTION]
Finding: Redis semaphore key "promotion_engine:expiry_job:running_count" has no TTL.
  A Redis flush or expiry would reset it to 0 — this is the DESIRED behaviour (counter resets).
  Key persists across pod restarts since Redis is a shared Sentinel cluster.
Impact: None. Counter drift is bounded to pod crash window; reconciled every 1 hour.
Action: Do NOT set a TTL on this key. Document in runbook.
Owner: tech-detailer

CONSIDERATION [INFRA] [NOTE]
Finding: reconcileSemaphoreCounter runs on ALL pods (scheduler on all pods).
  Multiple pods may call reconcile concurrently every hour.
  Race: pod A reads countByStatus=2 and writes 2; pod B reads countByStatus=2 and writes 2.
  Since both read the same actual count and both write the same value, the race is benign —
  no corruption, no inflated value.
Impact: None (idempotent write of the same value).
Action: No mitigation needed.
Owner: tech-detailer

CONSIDERATION [DB] [NOTE]
Finding: countByStatus(RUNNING) is a new query issued once per shard per hour.
  Uses the {status:1} compound index on ExpiryDateChangeJob — index-covered count.
  Zero document heap I/O. Negligible MongoDB load.
Impact: None.
Owner: tech-detailer
```

---

## 13. Internal Architecture Changes

**New dependency introduced:** `ExpiryDateChangeHelper` now depends on `ExpiryDateChangeJobFacade`.
Check for circular dependency: `ExpiryDateChangeHelper → ExpiryDateChangeJobFacade → ExpiryDateChangeFactory → processors → DAOs → ExpiryDateChangeJobDao`. No cycle back to `ExpiryDateChangeHelper`. ✅

**New pattern introduced:**
- Redis atomic counter semaphore (`INCR/DECR` on a named key) for cross-pod rate control.
- Fail-open Redis resilience: catch Redis exceptions → return `true` to avoid blocking on Redis unavailability.
- Factory configurer pattern (`RabbitListenerContainerFactoryConfigurer.configure()`) for inheriting Spring Boot AMQP defaults on custom factories. **This should be the repo convention** for any future custom RabbitMQ listener factories.

---

## 14. Upstream / Downstream Impact

### Upstream
- RabbitMQ `PROMOTION_EXPIRY_UPDATE_QUEUE` — no schema change. Queue name unchanged. Message structure unchanged.

### Downstream
- MongoDB `customerIssuedPromotion`, `customerEarnedPromotion`, `codeBasedPromotion` — same documents updated, same fields set. `impactedCount` on `ExpiryDateChangeJob` is unchanged in meaning.
- Redis `incentives` cluster — new key `"promotion_engine:expiry_job:running_count"` written by all pods. Low-frequency writes (1 INCR per job start, 1 DECR per job end, 1 SET per shard-loop reconciliation per hour).

---

## 15. SLA Impact

**Optimisation 1 (IN clause):** Lower MongoDB IOPS and CPU per job run. Bulk update wall-clock time should be same or lower. `impactedCount` is unchanged.

**Optimisation 2 (concurrency throttle):** Default `maxGlobalConcurrentJobs=3` (half of 6 pods) throttles from day 1 — peak MongoDB pressure is halved relative to today. A 300-promotion batch (600 messages) drains at 3 concurrent jobs; all messages complete within the ~17-minute retry window. Operator can raise `EXPIRY_MAX_GLOBAL_CONCURRENT_JOBS` to 6 to disable throttling if MongoDB headroom allows.

**Degraded mode:** If Redis is unavailable, semaphore fails open — all 6 pods process concurrently, same as today. No regression.

---

## 16. Observability

### New log lines
| Location | Level | Content |
|---|---|---|
| `ExpiryDateChangeJobFacade.tryAcquireSemaphore` | INFO | `"Semaphore acquired orgId={} jobId={} count={}/{}"` |
| `ExpiryDateChangeJobFacade.tryAcquireSemaphore` | INFO | `"Semaphore limit hit orgId={} jobId={} count={}/{}"` |
| `ExpiryDateChangeJobFacade.tryAcquireSemaphore` | WARN | `"Semaphore Redis error orgId={} jobId={} — failing open"` |
| `ExpiryDateChangeJobFacade.releaseSemaphore` | INFO | `"Semaphore released orgId={} jobId={} remaining={}"` |
| `ExpiryDateChangeJobFacade.reconcileSemaphoreCounter` | INFO | `"Semaphore reconciled to actual running count: {}"` |

### New Relic
- No new `@Trace` annotations required — the facade method is already traced via `@Profiled`.
- Recommended: add `metricsService.addCustomParameter("semaphoreCount", current)` in `tryAcquireSemaphore` to surface semaphore level in New Relic transaction traces.

### Redis monitoring
- Key to watch: `promotion_engine:expiry_job:running_count`
- Expected: value stays ≤ `maxGlobalConcurrentJobs` during a batch run; returns to 0 after all jobs complete.

---

## 17. Rollout Plan

1. **No feature flag required** — both changes are internal; no API behaviour change.
2. **Deploy order:** Standard rolling deploy — no migration scripts, no schema changes.
3. **Env vars to set at deploy:**
   - `EXPIRY_MAX_GLOBAL_CONCURRENT_JOBS=3` (default; throttles from day 1)
   - No other new env vars — factory pins `prefetch=1` and `consumers=1` in code; not operator-configurable by design.
4. **Post-deploy validation:**
   - Watch Redis key `promotion_engine:expiry_job:running_count` during next brand batch update.
   - Confirm value stays ≤ 3 and returns to 0 after completion.
   - Confirm MongoDB `opcounters.update` per second is lower during job runs vs pre-deploy baseline.
   - Confirm `ExpiryDateChangeJob.impactedCount` on a sample job matches expected doc count.
5. **Throttle tuning (if MongoDB headroom is comfortable):** raise `EXPIRY_MAX_GLOBAL_CONCURRENT_JOBS` to 4–6 to increase throughput. Do not lower below 2 without reviewing the retry window — at limit=1 with a 5-minute job, the 10-attempt retry window (~17 min) may not be sufficient for queued messages.
6. **Rollback:** revert single commit. No data migration to undo. Redis counter drifts to 0 within 1 hour via reconciliation even without rollback-specific cleanup.

---

## 18. Risks

| # | Description | Likelihood | Mitigation |
|---|------------|-----------|-----------|
| 1 | `RabbitListenerContainerFactoryConfigurer` not available in project's Spring AMQP version → compile error | Low | Check `spring-amqp` version in `pom.xml`; fallback: manually configure `StatefulRetryOperationsInterceptor` |
| 2 | `configurer.configure(factory, connectionFactory)` sets `concurrentConsumers=1` from global config, then our override sets it again — order matters | Low | Our overrides are applied AFTER `configure()` — correct order per Spring docs |
| 3 | Circular dependency: `ExpiryDateChangeHelper` → `ExpiryDateChangeJobFacade` | Low | Verified: no cycle exists in the dependency graph |
| 4 | `countByStatus` Spring Data derived method name conflicts with existing method | Low | Check `ExpiryDateChangeJobDao` interface — no collision found |
| 5 | Semaphore counter drifts negative if `releaseSemaphore` called without prior acquire | Very Low | `finally` is only reached after `tryAcquireSemaphore` returns `true`; no acquire = no finally block |
| 6 | USHC multi-shard: reconciliation runs concurrently on multiple pods — count is summed before write, not atomic | Very Low | Race is benign (idempotent — both pods write same value derived from same actual count) |

---

## 19. Open Questions

| Question | Owner | Due | Status |
|---------|-------|-----|--------|
| ~~Spring AMQP version — does it include `RabbitListenerContainerFactoryConfigurer`?~~ | — | — | **Resolved** — Spring Boot 2.6.6 → AMQP 2.4.x. Configurer is auto-configured since AMQP 2.0. ✅ |
| Should `sort` stage be explicitly removed from `getAggregation()` after change, or left as a no-op? | Saravanan | Before build | **Open** — recommend removing with an explanatory comment |
| ~~Initial production value for `EXPIRY_MAX_GLOBAL_CONCURRENT_JOBS`?~~ | — | — | **Resolved** — default set to **3**. Raise post-deploy if headroom allows. |
| Does the `incentives` Redis cluster key namespace collide with `"promotion_engine:expiry_job:running_count"`? | Infra team | Before deploy | **Open** |

---

## 20. Questions for arch-investigator — Resolved

**Q1 — Semaphore reconciliation placement** ✅ Resolved
Reconciliation moved to `ExpiryDateChangeHelper.updateExpiryDateChangeJobs()` post-shard-loop
with cross-shard sum. `reconcileSemaphoreCounter(long totalRunning)` exposed on Facade.
Confirmed acceptable; design updated accordingly.

**Q2 — `"cacheEvictionRedisTemplate"` bean uniqueness** ✅ Resolved
Confirmed sole `RedisTemplate<String,String>` bean in codebase at `RedisCacheUtil.java:204`.
No unnamed bean exists. `@Qualifier("cacheEvictionRedisTemplate")` is the correct and only
injection pattern. No ambiguity risk.

---

## 21. Task Breakdown

### Backend

| # | Task | File | Size | Depends on |
|---|------|------|------|-----------|
| B1 | Change `getAggregation()` to `group().push(_id)`, remove `sort`, update `BATCH_IDS` constant | `AbstractExpiryDateChangeDao.java` | S | — |
| B2 | Change `updateValidTill()` loop: extract `List<ObjectId> batchIds`, update `setStartId` call, remove `ObjectId endId` declaration | `AbstractExpiryDateChangeDao.java` | S | B1 |
| B3 | Change `updateValidTillDate()` signature to `List<ObjectId> batchIds`; replace range criteria with `Criteria.where(ID_COLUMN).in(batchIds).and(ORG_ID_COLUMN).is(...)` | `AbstractExpiryDateChangeDao.java` | S | B2 |
| B4 | Update `updateValidTillDate()` override: same signature + criteria change; AggregationUpdate pipeline unchanged | `CustomerEarnedCollectionDynamicExpiryDateChangeDao.java` | S | B3 |
| B5 | Add `expiryDateChangeContainerFactory` bean using `RabbitListenerContainerFactoryConfigurer` pattern | `SpringAmqpConfig.java` | S | — |
| B6 | Add `containerFactory = "expiryDateChangeContainerFactory"` to `@RabbitListener` | `ExpiryDateChangeJobListener.java` | S | B5 |
| B7 | Add `tryAcquireSemaphore()`, `releaseSemaphore()`, `reconcileSemaphoreCounter()` + `@Qualifier` `RedisTemplate` + `@Value` max jobs; wrap `changeExpiryForPromotion()` | `ExpiryDateChangeJobFacade.java` | M | — |
| B8 | Add post-loop shard-summing reconciliation; inject `ExpiryDateChangeJobFacade` + `ExpiryDateChangeJobDao` | `ExpiryDateChangeHelper.java` | S | B7 |
| B9 | Add `countByStatus(BatchStatus status): long` | `ExpiryDateChangeJobDao.java` | S | — |
| B10 | Add 1 new env-var-backed property (`expiry.date.change.max.global.concurrent.jobs`, default 3) | `application.properties` | S | — |

### Testing

| # | Task | File | Size | Depends on |
|---|------|------|------|-----------|
| T1 | Update `shouldUpdateValidTillDate` test: signature `List<ObjectId>` not `ObjectId` | `CustomerEarnedCollectionDynamicExpiryDateChangeDaoTest.java` | S | B4 |
| T2 | Add unit tests for new semaphore behaviour in `ExpiryDateChangeJobFacade`: acquire on success path, release on exception path, RuntimeException on limit hit, fail-open on Redis error. Add `@Mock RedisTemplate<String,String>` | `ExpiryDateChangeJobFacadeTest.java` | M | B7 |
| T3 | Add test: `markAllJobAsExpiredIfNotRunning` scenario should NOT trigger reconcile (reconcile is in Helper, not Service). Add `countByStatus` stub to `ExpiryDateChangeJobServiceTest`. | `ExpiryDateChangeJobServiceTest.java` | S | B9 |
| T4 | Add test: `updateExpiryDateChangeJobs` calls `reconcileSemaphoreCounter` with correct summed count after shard loop | `ExpiryDateChangeHelperTest.java` | S | B8 |
| T5 | Verify existing integration tests pass unchanged (`PromotionExpiryDateChangeIntegrationTest`) | — | S | B1–B4 |

---

## Handoff Notes for test-plan-architect

**Tech detail doc location:**
`CAP-184618_expiry_job_tuning/CAP-184618_expiry_job_tuning_techdetail.md`

### Critical B2B Flows to Test
- UC-1: Single promotion end-date update → 2 jobs → correct `impactedCount` and `validTill` → Integration
- UC-2: 300 promotions batch update → semaphore limits active jobs → MongoDB load controlled → Load test
- UC-4: Pod crash scenario → counter healed after reconciliation → Integration
- UC-5: Semaphore full → job retried → eventually processed → Unit

### Critical B2C Flows to Test
- UC-9: After job completes, customer sees updated `validTill` → Integration (via `PromotionExpiryDateChangeIntegrationTest`)

### Regression Risks (must not break)
- `impactedCount` on `ExpiryDateChangeJob` must equal the number of documents updated per promotion — verify in existing integration tests.
- EARNED-dynamic path (`CustomerEarnedCollectionDynamicExpiryDateChangeDao`) uses `AggregationUpdate` (not simple `Update`) — the `$in` criteria change must be validated to work with `AggregationUpdate.updateMulti()`, not just simple `Update.updateMulti()`.
- `ExpiryDateChangeJobListener` must still consume from `PROMOTION_EXPIRY_UPDATE_QUEUE` after `containerFactory` switch — verify no message routing breakage.

### Tenant Isolation Tests
- After processing job for orgId=A, verify no documents for orgId=B are modified.
- Verify Redis semaphore key is GLOBAL (not org-scoped) — all orgs compete for the same `maxGlobalConcurrentJobs` slots.

### Suggested Test Emphasis
- **Unit:** `AbstractExpiryDateChangeDao` via `CustomerIssuedCollectionStaticExpiryDateChangeDao` — mock `mongoOperation.aggregate()` returning a result with `ids` array; verify `updateMulti` called with `$in` criteria.
- **Unit:** `ExpiryDateChangeJobFacade` — 4 semaphore scenarios (acquire/release/limit-hit/fail-open).
- **Unit:** `ExpiryDateChangeHelper` — verify `reconcileSemaphoreCounter` called with correct cross-shard sum.
- **Integration:** `PromotionExpiryDateChangeIntegrationTest` — existing tests must pass without modification.
- **Integration:** Verify EARNED-dynamic path works end-to-end with `AggregationUpdate` + `$in` criteria.
