# Tech Detail Review: Reward Catalogue Listing API
**Date:** 2026-05-14
**Reviewer:** tech-detail-reviewer
**Document Reviewed:** `tech-detail.md`
**Overall Verdict:** APPROVED WITH CONDITIONS

---

## Executive Summary

The tech detail is thorough, well-structured, and catches the key design gaps (enable/disable toggle path, batch orgId grouping, TopicExchange → DirectExchange). It correctly follows MADR-0001 and MADR-0002, implements cursor pagination with correct `$or` predicate, and applies `maxTimeMS(100)` across all queries. However, two blockers must be resolved before implementation starts: (1) `@RmqMessageTracker` on the batch listener will cause a `ClassCastException` at runtime because the existing `RMQMessageTrackerAspect` casts `args[0]` to `Message` — a batch listener receives `List<String>`, not `Message`; (2) `CatalogueIndexInitializer.ensureIndexes()` only creates the TTL index but `MongoDbInitializer.createIndexes()` is commented out globally, so the 8 performance-critical compound indexes will never be created on production startup. Both are codebase-verified. One additional medium-severity action: `RewardDetails.imageId` maps to `IMAGE_URI` (a URI reference ID), while the actual URL lives in `RewardDetails.imagePath` (column `IMAGE_PATH`) — the tech detail incorrectly references `IMAGE_URI` for the image URL.

---

## Phase 1 — Completeness Gaps

**COMPLETENESS GAP [SEVERITY: LOW]**
Section: §13 Internal Architecture Changes — Modified Classes
Issue: Present but incomplete
Detail: `JerseyConfig.java` must have `register(RewardCatalogueResource.class)` added. It is named in the task breakdown (last item) but absent from the Modified Classes list in §13.
Risk if unfixed: Developer forgets to register the resource; endpoint returns 404 silently.

**COMPLETENESS GAP [SEVERITY: LOW]**
Section: §16 Observability
Issue: Present but incomplete — `catalogue.cacheHit` attribute is listed as a mandatory New Relic attribute, but the corresponding `metricsService.addCustomParameter("catalogue.cacheHit", ...)` call is absent from §8's `RewardCatalogueResource` and `RewardCatalogueService` code sketches.
Risk if unfixed: Developer skips the attribute; cache-hit ratio unobservable in production.

All other required sections (§1–§21) are present and substantive.

---

## Phase 2 — Codebase Discrepancies

**CODEBASE DISCREPANCY [SEVERITY: HIGH]**
Document claims: `@RmqMessageTracker(name = "rewards.catalogue.sync")` on `RewardCatalogueSyncListener.onBatch(List<String> messages)` provides RMQ tracking.
Actual code: `RMQMessageTrackerAspect.java:52` does `Message message = (Message) pjp.getArgs()[0]`. Every existing listener (`RewardExpiryReminderListener.java:33`, `TransactionActionLogsListener.java`, `PointsRedemptionListener.java`) takes a single `Message` as `args[0]`. A batch listener delivers `List<String>` as `args[0]`. The cast will throw `ClassCastException` at runtime on first batch message.
Impact: The catalogue sync listener crashes on every batch, all events go to DLQ, MongoDB read model never populates.
Correction needed: Remove `@RmqMessageTracker` from `onBatch()`. Implement tracking manually in `processOrgBatch()`: add `MDC.put(Constants.REQUEST_ID_MDC, Utils.generateRequestId())`, `MDC.put(Constants.INTERNAL_REQUEST_ID_MDC, Utils.generateRequestId())`, `MDC.put(Constants.X_CAP_API_AUTH_ORG_ID, orgId.toString())`, emit New Relic attributes and success/failure tracking via `MetricsService.markAsSuccessfulRequest("rewards.catalogue.sync")` / `markAsFailedRequest(...)` directly.

**CODEBASE DISCREPANCY [SEVERITY: HIGH]**
Document claims: "All @CompoundIndexes on both doc classes are created by MongoDbInitializer (which calls ensureIndex via createIndexes). Only the TTL index needs special handling here." — `CatalogueIndexInitializer.ensureIndexes()` therefore only creates the TTL index.
Actual code: `MongoDbInitializer.java:46` — `createIndexes()` is commented out: `//createIndexes(mongoDocumentClasses, mongoTemplate)`. It is NOT called on startup. `MongoClientConfiguration.java:56–59` — `mongoMappingContext()` explicitly prevents Spring Data's auto index creator from running ("By not calling mappingContext.afterPropertiesSet(), we prevent the index creator from being registered"). No mechanism exists that creates `@CompoundIndexes` at startup.
Impact: All 8 compound indexes (3 on `rewards_catalogue`, 5 on `active_rewards`) are never created. Every catalogue query degrades to a full collection scan. Performance target (< 150ms p95) will not be met.
Correction needed: `CatalogueIndexInitializer.ensureIndexes()` must explicitly call `mongoTemplate.indexOps(...).ensureIndex(...)` for all 8 compound indexes in addition to the TTL index. See §9 of tech-detail for the complete index list.

**CODEBASE DISCREPANCY [SEVERITY: MEDIUM]**
Document claims: `RewardCatalogueDoc.imageUrl` populated "from REF_REWARD_DETAILS.IMAGE_URI (default lang)"
Actual code: `RewardDetails.java:49–51` — `IMAGE_URI` maps to field `imageId` (a URI reference ID, not a URL string). `RewardDetails.java:61–63` — `IMAGE_PATH` maps to field `imagePath` which is the actual HTTP URL. Method `toRewardDetailsManagementListingRO()` at line 82 confirms: `.imageUrl(getImagePath())`.
Impact: If the developer uses `details.getImageId()` for `imageUrl`, MongoDB stores a URI ID string instead of the actual image URL. All catalogue API responses show IDs instead of URLs.
Correction needed: `buildCatalogueDoc()` must use `details.getImagePath()` (from column `IMAGE_PATH`), not `details.getImageId()` (from column `IMAGE_URI`). Update §8 comment to say "from `IMAGE_PATH` via `getImagePath()`".

**CODEBASE DISCREPANCY [SEVERITY: MEDIUM]**
Document claims: §8 — "Modifications to `RewardFacade` — `updateStatus()` path (line ~1651)". Describes adding `catalogueEventPublisher.publishUpsert(orgId, id)` to the `updateStatus()` method body.
Actual code: `RewardFacade.java:798–800` — `updateStatus(Long id, Long orgId, boolean status)` is a one-liner: `return rewardJdbcRepository.updateStatus(orgId, id, status);`. The `if (updateCount > 0)` block is in `setStatusCategory()` at line 1649–1652. The publish should go into `setStatusCategory()`, not `updateStatus()`.
Impact: Developer adds the publish to the wrong method (the simple delegate at line 798), missing the `if (updateCount > 0)` guard — catalogue sync fires on failed status changes too.
Correction needed: Name the correct method: `setStatusCategory()` at line 1636. The publish call should be inside the `if (updateCount > 0)` block at line 1649 of `setStatusCategory()`, not in `updateStatus()`.

**CODEBASE DISCREPANCY [SEVERITY: LOW]**
Document claims: `RewardCatalogueSyncListener.processOrgBatch()` sets up MDC.
Actual code: All existing listeners set 3 MDC keys: `X_CAP_API_AUTH_ORG_ID`, `REQUEST_ID_MDC`, and `INTERNAL_REQUEST_ID_MDC`. The tech-detail's `processOrgBatch()` only sets `X_CAP_API_AUTH_ORG_ID`. `REQUEST_ID_MDC` and `INTERNAL_REQUEST_ID_MDC` are missing, breaking log correlation for catalogue sync operations.
Impact: Catalogue sync log lines will have no `requestId` or `internalRequestId`, making it impossible to trace individual batch executions in log aggregation.
Correction needed: Add `MDC.put(Constants.REQUEST_ID_MDC, Utils.generateRequestId())` and `MDC.put(Constants.INTERNAL_REQUEST_ID_MDC, Utils.generateRequestId())` at the top of `processOrgBatch()`. Remove both in the `finally` block alongside `X_CAP_API_AUTH_ORG_ID`.

**CODEBASE DISCREPANCY [SEVERITY: LOW]**
Document claims: `@Autowired private RewardRepository rewardDetailsRepository;` (the "same bean" comment is on a second `@Autowired RewardRepository`).
Actual code: Two `@Autowired` fields of the same type in the same class would cause Spring injection ambiguity or be silently resolved to the same bean. The `// same bean` comment signals this is intentional but it is redundant and confusing.
Correction needed: Remove the duplicate `rewardDetailsRepository` field. Use only `rewardRepository` throughout `RewardCatalogueSyncListener`.

---

## Phase 3 — Solution Approach Gaps

**SOLUTION GAP [SEVERITY: MEDIUM]**
Category: Guardrail — Observability
What the document proposes: `catalogue.cacheHit` is listed as a mandatory New Relic attribute in §16 but absent from the §8 code sketches for `RewardCatalogueResource` and `RewardCatalogueService`.
What is required: `metricsService.addCustomParameter("catalogue.cacheHit", cached.isPresent())` must be called in `RewardCatalogueService.list()` before the cache check returns.
Risk if not addressed: Cache-hit ratio invisible in production. Cannot validate that the 30s cache is functioning correctly post-deploy.
Recommended correction: Add `metricsService.addCustomParameter("catalogue.cacheHit", true/false)` in the cache-hit and cache-miss branches of `RewardCatalogueService.list()`.

**SOLUTION GAP [SEVERITY: MEDIUM]**
Category: API Contract — Undocumented `total` semantics
What the document proposes: The `count` query in `ActiveRewardMongoRepository.findFiltered()` applies only `orgId` and `isEnabled:true` filters — NOT the cursor, groupId, labelId, or points filters. The returned `total` is the org-total active reward count, not the filter-match count.
What is required: The API spec in §10 must explicitly document this: "`total` = total active rewards for this org, not the count matching the current filter combination." If callers expect `total` to match the filter, this is a contract bug.
Risk if not addressed: Clients build pagination UI using `total` as the filtered count — shows incorrect "Page 1 of N" when N is based on unfiltered total.
Recommended correction: Document `total` semantics in §10 API spec. If filter-aware total is needed, apply the same criteria to the count query (adds ~5ms latency).

**SOLUTION GAP [SEVERITY: LOW]**
Category: Guardrail — Observability / Infra
What the document proposes: `@Trace(dispatcher = true, metricName = "rewards.catalogue.sync")` on `RewardCatalogueSyncListener.onBatch()`.
What is required: New Relic `@Trace(dispatcher = true)` creates a transaction boundary. Since `onBatch()` processes multiple orgs per batch, the trace will show aggregate latency for all orgs in the batch, making per-org debugging harder. The existing `@Trace` on the processor entry (`processOrgBatch`) would be more appropriate for per-org tracing.
Risk if not addressed: Low — a single per-batch trace is still useful.
Recommended correction: Move `@Trace(dispatcher = true)` to `processOrgBatch()` to trace per-org.

---

## Phase 4 — Alternate Solution Assessment

None — all alternatives (DirectExchange, single collection, synchronous dual-write, @Cacheable) are adequately evaluated with clear rationale. The choices are sound.

**Note on `@Cacheable` rejection:** The rationale ("cannot do prefix-based per-org invalidation") is correct and well-stated.

---

## Phase 5 — Performance Gaps

**PERFORMANCE GAP [SEVERITY: MEDIUM]**
Area: DB — Count query on every cache miss
Finding: `ActiveRewardMongoRepository.findFiltered()` issues a separate count query (with `maxTimeMS(100)`) on every cache miss, applying only `orgId + isEnabled` filters. For an org with 2000 active rewards, this count hits the `{ orgId: 1, isEnabled: 1 }` compound index — efficient. However, running two MongoDB queries per request (count + find) doubles MongoDB load.
Risk: At 8K RPM with 30s TTL and cold start (empty cache), both queries fire simultaneously on every pod. With a single MongoDB replica, 8K × 2 = 16K ops/s.
Recommended action: Consider whether `total` is required on every request, or whether a separate "count-only" endpoint reduces load. Alternatively, compute count from the find result (`total = results.size()` if fetching all) or accept stale total from a separate cached count.

**PERFORMANCE GAP [SEVERITY: LOW]**
Area: GC — `Collectors.groupingBy` with downstream `Collectors.mapping` creates intermediate lambdas
Finding: `processUpserts()` uses `Collectors.groupingBy` with `Collectors.mapping` for `groupsByReward` and `labelsByReward`. For batch sizes of 200 rewards with 5 groups each, this allocates ~1000 intermediate stream objects in Eden per batch.
Risk: Minor GC pressure at high batch rates. Not a correctness issue.
Recommended action: Accept for initial release. If profiling shows allocation pressure, replace with explicit `for` loops as the `.context/code.md` GC guardrail recommends for multi-pass collections.

**PERFORMANCE GAP [SEVERITY: LOW]**
Area: Cache — `keys()` command on Redis Sentinel is O(N) blocking
Finding: Already documented in §12. `keys()` scans all keys on the Redis primary. With catalogue keys TTLing at 30s and fewer than 200 keys per org at any time, this is acceptable.
Risk: At peak (2000 reward batch → 10 batches → 10 invalidation calls per org), each call scans the full Redis keyspace. 30s TTL self-heals.
Recommended action: Accept per §12 mitigation. No new action.

---

## Phase 6 — Missing Use Cases

**MISSING USE CASE [B2C] [NEGATIVE]**
ID: UC-REVIEW-1
Flow: Client sends `cursor=<malformed-base64>` → `CursorToken.decode()` throws `IllegalArgumentException` or Gson parse error → caught by `catch (Exception e)` in `RewardCatalogueResource` → returns 500.
Why it matters: A malformed cursor (network corruption, client bug, manual manipulation) should return 400, not 500. Returning 500 causes alerting noise.
After fix: Validate cursor in `RewardCatalogueRequest.fromUriInfo()` or catch `CursorDecodeException` specifically and return 400 in the resource layer.
Test type needed: Unit (resource validation)

**MISSING USE CASE [B2C] [NEGATIVE]**
ID: UC-REVIEW-2
Flow: MongoDB unavailable → `active_rewards` query throws `MongoException` / `MongoExecutionTimeoutException` → caught by `catch (Exception e)` in `RewardCatalogueResource` → returns 500.
Why it matters: This should log ERROR and return a meaningful error response (or retry from cache). The SLO-guard `maxTimeMS(100)` triggers `MongoExecutionTimeoutException` on slow MongoDB — this must not silence as generic 500.
After fix: Catch `MongoExecutionTimeoutException` specifically in `RewardCatalogueService.list()` → log WARN (not ERROR — it's an SLO breach, not an unexpected error) → propagate as a domain exception that the resource returns as 503 or 500 with error code.
Test type needed: Unit (timeout handling)

**MISSING USE CASE [B2B] [BOUNDARY]**
ID: UC-REVIEW-3
Flow: Admin creates 2000 rewards simultaneously (bulk upload) → 2000 RMQ UPSERT events published → batch listener processes 10 batches of 200 → each batch does 1 MySQL IN query with 200 IDs → all 2000 `active_rewards` docs upserted.
Why it matters: This is the highest-risk production scenario (month-start burst). The MySQL IN query with 200 IDs must be validated against the `findByOrgIdAndIdIn` JPA query definition.
After fix: All 2000 rewards appear in `active_rewards` within seconds of batch processing completing.
Test type needed: Integration (end-to-end batch flow)

---

## Phase 7 — Security, DB, Infra Gaps

**CONSIDERATION GAP [INFRA] [ACTION]**
Finding: `CatalogueIndexInitializer` is annotated `@Profile("production")`. The `MongoClientConfiguration` is also `@Profile("production")`. If test profiles don't have a Mongo bean, the `@Autowired MongoTemplate` in `CatalogueIndexInitializer` won't be injected and the initializer won't run in tests. Compound indexes won't exist in test-profile embedded Mongo, so integration tests don't validate query performance characteristics.
Impact if ignored: Integration tests pass against a scanless collection. Index regressions are invisible until production.
Recommended action: Add a `@Profile("test")` variant or ensure embedded test Mongo has the same indexes created by a `@PostConstruct` test helper. The `@CompoundIndexes` annotations on document classes are sufficient for Spring Data test support if Spring Data auto-index creation is enabled for the test profile.
Owner: developer

**CONSIDERATION GAP [DB] [ACTION]**
Finding: `active_rewards.isEnabled` compound index `{ orgId: 1, isEnabled: 1 }` — since `active_rewards` only stores records with `isEnabled=true`, this index is always queried with `isEnabled=true` and the index provides no selectivity benefit over `{ orgId: 1 }` alone. It wastes write amplification.
Impact if ignored: Minor write overhead on every upsert. Not a correctness issue.
Recommended action: Drop `{ orgId: 1, isEnabled: 1 }` from `ActiveRewardDoc.@CompoundIndexes`. The `isEnabled=true` filter in queries is redundant (all docs are active) and adds nothing. Replace with a plain `{ orgId: 1 }` if needed, but the `{ orgId: 1, pointsValue: 1 }` index already covers unfiltered org queries.
Owner: developer

All other security checks from §11 pass: orgId from `AuthDetails` (not query param), no PII in logs, tenant isolation per org in all queries and cache keys, typed integer inputs (no injection risk), cursor tamper returns wrong page not cross-org leak.

---

## Phase 8 — Observability and Rollout Gaps

**ROLLOUT GAP [SEVERITY: MEDIUM]**
Area: Observability — `catalogue.cacheHit`
Finding: §16 mandates `catalogue.cacheHit` as a New Relic attribute but §8 code sketch does not emit it. (Also captured in Phase 1/3.)
Risk if unfixed: Cache effectiveness invisible post-deploy — cannot confirm 30s TTL is working or diagnose cache miss storms.
Recommended action: Add `metricsService.addCustomParameter("catalogue.cacheHit", cached.isPresent())` in `RewardCatalogueService.list()`.

**ROLLOUT GAP [SEVERITY: LOW]**
Area: Go/No-Go Criteria — initial data backfill
Finding: §17 states "All existing rewards will be synced progressively as they are mutated" and "For the load test environment, manually trigger UPSERT events for all active rewards." There is no backfill job defined, no tooling named, and no estimated time-to-full-coverage.
Risk if unfixed: Load test environment has 0 active rewards in MongoDB at test start. Load test results are invalid.
Recommended action: Add a one-time admin endpoint (or a Gradle task / script) that publishes UPSERT events for all currently active rewards in a given org. Scope this as a small backend task before load test.

**ROLLOUT GAP [SEVERITY: LOW]**
Area: Rollback procedure
Finding: §17 rollback says "Remove `CatalogueIndexInitializer` bean (or `@Profile` guard it)" and "Remove `RewardCatalogueResource` from Jersey config." Both require a code deploy. There is no feature flag — rollback requires a new build.
Risk if unfixed: If a production issue emerges, rollback takes a full deploy cycle.
Recommended action: Accept (new endpoint is additive — existing endpoints are not broken). Document explicitly: "Rollback requires a new deploy. The afterCommit + direct publish additions are safe to leave in place even without the listener running — events accumulate harmlessly in the queue."

---

## Consolidated Action List

### Blockers
| # | Finding | Phase | Recommended Action |
|---|---------|-------|--------------------|
| B1 | `@RmqMessageTracker` on `onBatch(List<String>)` will ClassCastException — aspect casts `args[0]` to `Message` | Phase 2 | Remove `@RmqMessageTracker`; implement tracking manually in `processOrgBatch()` with MDC + MetricsService calls |
| B2 | `CatalogueIndexInitializer` only creates TTL index; compound indexes never created (`createIndexes()` commented out globally) | Phase 2 | Add explicit `ensureIndex` calls for all 8 compound indexes in `CatalogueIndexInitializer.ensureIndexes()` |

### Actions
| # | Finding | Phase | Recommended Action |
|---|---------|-------|--------------------|
| A1 | `RewardDetails.imageId` is `IMAGE_URI` (ID); actual URL is `getImagePath()` from `IMAGE_PATH` | Phase 2 | Use `details.getImagePath()` in `buildCatalogueDoc()` |
| A2 | §8 names `updateStatus()` as the method to modify; actual method is `setStatusCategory()` at line 1636 | Phase 2 | Correct method name reference in §8 to `setStatusCategory()` |
| A3 | `processOrgBatch()` missing `REQUEST_ID_MDC` and `INTERNAL_REQUEST_ID_MDC` setup + cleanup | Phase 2 | Add both MDC keys at entry of `processOrgBatch()`, remove both in `finally` |
| A4 | `catalogue.cacheHit` New Relic attribute in §16 absent from §8 code | Phase 1/3/8 | Add `metricsService.addCustomParameter("catalogue.cacheHit", ...)` in `RewardCatalogueService.list()` |
| A5 | `total` in API response is org-total active rewards, not filter-match count — undocumented | Phase 3/5 | Document in §10 API spec; decide if filter-aware count is required |
| A6 | Malformed cursor returns 500 — should return 400 | Phase 6 | Add cursor validation / specific exception catch → 400 |
| A7 | `active_rewards.{ orgId, isEnabled }` compound index provides no selectivity (all docs have `isEnabled=true`) | Phase 7 | Drop this index; redundant write amplification |
| A8 | Add backfill script/endpoint for load test initial population | Phase 8 | One-time admin endpoint or script to publish UPSERT events for all active rewards of an org |

### Notes
| # | Finding | Phase | Note |
|---|---------|-------|------|
| N1 | `JerseyConfig.java` absent from §13 Modified Classes | Phase 1 | Add to modified classes list |
| N2 | Duplicate `@Autowired RewardRepository rewardDetailsRepository` | Phase 2 | Remove duplicate; use single `rewardRepository` field |
| N3 | `add("isEnabled").is(true)` in MongoDB query is redundant — `active_rewards` only stores active rewards | Phase 5 | Accept or remove; no functional impact |
| N4 | Move `@Trace(dispatcher = true)` to `processOrgBatch()` for per-org New Relic traces | Phase 3 | Preference, not required |
| N5 | Count query on every cache miss doubles MongoDB ops — consider caching count or making it optional | Phase 5 | Accept for initial release; review post load test |
| N6 | `THIRTY_SECOND_CACHE` constant addition to `RedisCacheKeys` + `RedisCacheUtil` noted as "may not be needed." Per guardrail, add both together or add neither. Since `CatalogueCache` uses manual `RedisTemplate`, no `@Cacheable` bucket is needed. Skip adding the constant entirely. | Phase 3 | Do NOT add `THIRTY_SECOND_CACHE` unless `@Cacheable` is used. CatalogueCache hardcodes `Duration.ofSeconds(30)` directly — this is correct. |

---

## Reviewer Notes for tech-detailer

**What was particularly well-specified:**
- orgId-first on every MongoDB query and Redis key — correctly enforced throughout
- Cursor pagination `$or` predicate `(pointsValue > V) OR (pointsValue = V AND _id > id)` is correct and well-specified
- afterCommit vs direct-publish distinction per transaction context is correctly reasoned
- Batch orgId grouping for tenant isolation is the right call and well-explained
- `maxTimeMS(100)` on all MongoDB queries consistently applied
- Upsert semantics on both collections make replay safe
- Exception isolation in publisher (catch + log, never propagate to caller) is correct

**Items to carry into implementation:**

1. **B1 (BLOCKER):** Before writing `RewardCatalogueSyncListener`, confirm you cannot use `@RmqMessageTracker`. Emit New Relic success/failure tracking manually after reading how `MetricsService.markAsSuccessfulRequest()` / `markAsFailedRequest()` is called in `PointsRedemptionListener.java` — that is your reference pattern.

2. **B2 (BLOCKER):** Open `CatalogueIndexInitializer.ensureIndexes()` and add all 8 compound indexes listed in §9 using the same `mongoTemplate.indexOps(...).ensureIndex(new CompoundIndexDefinition(...))` pattern. The TTL index uses `new Index("expiresAt", ASC).expire(0)`. The compound indexes should use `new CompoundIndexDefinition(org.bson.Document.parse("{...}"))`. Cross-reference `MongoDbInitializer.getCompoundIndexesForEntity()` (lines 128–144) for the exact pattern.

3. **A1 (MEDIUM):** In `buildCatalogueDoc()`, `imageUrl` field must be set from `details.getImagePath()`, not `details.getImageId()`. The `IMAGE_PATH` column holds the actual URL. Check `RewardDetailsManagementListingRO` builder pattern in `RewardDetails.java:82` as a reference.

4. **A2 (MEDIUM):** §8 says "Modifications to `RewardFacade` — `updateStatus()` path (line ~1651)". The actual method to modify is `setStatusCategory()` at line 1636. The `if (updateCount > 0)` block starts at line 1649. Add `catalogueEventPublisher.publishUpsert(orgId, id)` at line 1651 (inside `if (updateCount > 0)` after `eventsService.pushRewardUpdatedEvent(reward)`).

5. **Note N6:** Do NOT add `THIRTY_SECOND_CACHE` to `RedisCacheKeys` or `RedisCacheUtil`. `CatalogueCache` hardcodes `Duration.ofSeconds(30)` directly via `RedisTemplate` — this is correct and doesn't need a named cache bucket (those are for `@Cacheable` only). Adding an unused constant violates the "no magic strings without purpose" rule.

---

## Updated Task Breakdown (delta only)

The following tasks from §21 need modification based on this review:

| Task | Change |
|------|--------|
| `CatalogueIndexInitializer` — @PostConstruct ensureIndex | **Expand scope**: create ALL 8 compound indexes + TTL index, not just TTL. See §9 for full list. |
| `RewardCatalogueSyncListener` — batch consumer | **Remove** `@RmqMessageTracker`; **add** manual MDC (3 keys) + MetricsService tracking per-org |
| `buildCatalogueDoc()` helper | **Fix**: use `details.getImagePath()` not `details.getImageId()` for `imageUrl` |
| `RewardFacade.setStatusCategory()` | **Rename** the task: it's `setStatusCategory()` not `updateStatus()` |
| `RewardCatalogueService.list()` | **Add**: `metricsService.addCustomParameter("catalogue.cacheHit", ...)` |
| `RedisCacheKeys` + `RedisCacheUtil` — add THIRTY_SECOND_CACHE | **REMOVE THIS TASK**: Not needed; CatalogueCache uses hardcoded duration |
| New task: cursor validation (400 on malformed cursor) | **Add** [S]: validate cursor format; return 400 on decode failure |
| New task: backfill script/admin endpoint for load test | **Add** [M]: publish UPSERT events for all active rewards of a given org |
| `JerseyConfig.java` — add to Modified Classes | **Add to §13 modified classes list** |
