# Test Plan: Expiry Date Change Job — MongoDB Load Reduction
**Date:** 2026-05-04
**Input:** `CAP-184618_expiry_job_tuning_techdetail.md` (Status: Final, 2026-04-28)
**Scope:** Two orthogonal optimisations — (1) replace range-based MongoDB `updateMulti` with
`$in` point lookups; (2) Redis INCR/DECR semaphore capping cross-pod concurrency at
`EXPIRY_MAX_GLOBAL_CONCURRENT_JOBS=3`, with hourly counter reconciliation and a dedicated
protective-isolation RabbitMQ container factory.
**Confidence:** HIGH — full tech detail with resolved design gaps; automation repo surveyed
(no existing test for this flow — confirmed gap).

---

## Requirements Summary

### Functional Requirements

| ID | Requirement | Source |
|----|-------------|--------|
| FR-001 | `AbstractExpiryDateChangeDao.getAggregation()` pushes all matching `_id` values into a list (`BATCH_IDS`); `updateValidTillDate()` uses `$in(batchIds)` + defensive `orgId` filter instead of the `_id` range predicate | §8a, UC-1 |
| FR-002 | `CustomerEarnedCollectionDynamicExpiryDateChangeDao.updateValidTillDate()` override also switches to `$in(batchIds)` + `orgId` filter; the `AggregationUpdate` pipeline itself is unchanged | §8b, UC-1, Handoff |
| FR-003 | `ExpiryDateChangeJobFacade.tryAcquireSemaphore()` atomically INCRs the Redis counter; returns `false` and DECRs if result > `maxGlobalConcurrentJobs`; returns `true` if within limit | §8e, UC-5 |
| FR-004 | `tryAcquireSemaphore()` is fail-open: any Redis exception is caught; WARN logged; `true` returned so jobs are never blocked by Redis unavailability | §8e, Gap 4 |
| FR-005 | `releaseSemaphore()` is called in the `finally` block of `changeExpiryForPromotion()` — guaranteed release on success, exception, and retry | §8e, UC-4 |
| FR-006 | When semaphore limit is hit, `changeExpiryForPromotion()` throws `RuntimeException("Expiry job concurrency limit reached, will retry")` — RMQ retry kicks in | §8e, UC-5 |
| FR-007 | `ExpiryDateChangeHelper.updateExpiryDateChangeJobs()` sums `countByStatus(RUNNING)` across ALL shards after the shard loop and calls `ExpiryDateChangeJobFacade.reconcileSemaphoreCounter(totalRunning)` | §8f, UC-4, Gap 2 |
| FR-008 | `expiryDateChangeContainerFactory` bean calls `configurer.configure(factory, connectionFactory)` first (inherits retry interceptor), then hard-pins `prefetchCount=1`, `concurrentConsumers=1`, `maxConcurrentConsumers=1` | §8c, Gap 3 |
| FR-009 | `ExpiryDateChangeJobDao.countByStatus(BatchStatus)` derived query is index-covered via `{status:1}` index | §8g |

### Non-Functional Requirements

| ID | Requirement | Source |
|----|-------------|--------|
| NFR-001 | Redis outage must not prevent expiry job processing — fail-open required | §8e, Gap 4, §15 |
| NFR-002 | Cross-tenant data isolation: `$in(batchIds)` IDs are pre-filtered by `orgId`; no cross-org document can be updated | §5, §11, Handoff |
| NFR-003 | Counter drift after pod crash must be bounded to ≤1 hour (scheduler period) | §8f, UC-4 |
| NFR-004 | Redis semaphore key is global (not per-org); all orgs compete for the same `maxGlobalConcurrentJobs` slots | §8e, §11 |
| NFR-005 | Expiry queue concurrency must not be affected by future global broker config changes (prefetch, consumers) | §8c |
| NFR-006 | Peak MongoDB concurrent `updateMulti` operations must stay ≤ `maxGlobalConcurrentJobs` (=3) during a large batch run | §15, UC-2 |
| NFR-007 | All messages in a 300-promotion batch (600 RMQ messages) must drain within the ~17-minute retry window | §15, §17 |

### Ambiguities & Open Questions

| # | Question | Blocks | Owner |
|---|----------|--------|-------|
| Q1 | Should `sort` stage be explicitly removed from `getAggregation()` after change? (Open: recommend removal with comment) | UT-01 | Saravanan — before build |
| Q2 | Does `incentives` Redis cluster key namespace collide with `"promotion_engine:expiry_job:running_count"`? | NFR-004 (prod) | Infra team — before deploy |

---

## Risk Assessment

| Area | Risk | Priority |
|------|------|----------|
| DAO — $in extraction | `batchIds` cast from `BasicDBList` fails silently; `updateValidTill` loop exits early with 0 docs updated | P0 |
| DAO — AggregationUpdate | `$in` criteria change breaks `AggregationUpdate.updateMulti()` for the EARNED-dynamic path (different update type vs simple Update) | P0 |
| Semaphore — fail-open | Redis exception path returns wrong value or re-throws — all 600 queued jobs dead-letter | P0 |
| Semaphore — counter leak | `releaseSemaphore()` not called on exception; counter grows permanently; all jobs eventually blocked | P0 |
| Semaphore — reconciliation placement | Reconcile called inside shard loop (per-shard count, not global sum) — USHC counter always wrong | P0 (fixed in design; test confirms it) |
| Factory — retry inheritance | `configurer.configure()` not called first — custom factory silently has no retry policy; jobs dead-letter instead of retrying | P0 |
| Factory — listener routing | `containerFactory` annotation change re-routes listener to wrong queue or breaks message delivery | P0 |
| Tenant isolation | `$in` IDs contain cross-org documents due to wrong `orgId` filter in aggregation | P0 |
| Load — semaphore throttle | Under large batch, >3 jobs run concurrently; MongoDB CPU spike not reduced | P1 (NFR-006) |
| Load — message loss | Jobs exhausting retry window dead-letter when `maxGlobalConcurrentJobs` is too low | P1 (NFR-007) |

---

## Test Case Specifications

### Unit Tests — `src/test/java/`

All new unit tests for facade go in **`ExpiryDateChangeJobFacadeTest.java`**. Tasks T1–T4 in
§21 map directly to the files noted.

---

#### Group A — DAO: `$in` Aggregation & Criteria (`AbstractExpiryDateChangeDao`)

| ID | Req | Description | Class Under Test | Mock Strategy | Priority |
|----|-----|-------------|-----------------|---------------|----------|
| UT-01 | FR-001 | `getAggregation()` pipeline uses `$push(_id)` as group output key `"ids"`, not `$last("_id")` as `"last_id"`. Verify `$sort` stage is absent (or confirmed removed). | `CustomerIssuedCollectionStaticExpiryDateChangeDao` (concrete subclass) | Mock `mongoOperation.aggregate()`; capture `Aggregation` arg; assert pipeline stages and group expression | P0 |
| UT-02 | FR-001 | `updateValidTill()` loop: when aggregation returns a non-empty `BasicDBList` of IDs, extracts `batchIds`, advances `startId` to `batchIds.get(last)`, and calls `updateValidTillDate` with the list. | `CustomerIssuedCollectionStaticExpiryDateChangeDao` | Mock aggregate to return one batch of 3 IDs then empty; verify `updateValidTillDate` called once; verify `startId` advanced to 3rd ID | P0 |
| UT-03 | FR-001, UC-6 | `updateValidTill()` loop exits immediately (no `updateValidTillDate` call) when `batchIds` is empty on first iteration. | `CustomerIssuedCollectionStaticExpiryDateChangeDao` | Mock aggregate returns empty list on first call | P1 |
| UT-04 | FR-001, NFR-002 | `updateValidTillDate()` calls `updateMulti` with criteria `{_id: {$in: batchIds}, orgId: Y}` — NOT a range predicate. Stale `ObjectId endId` variable must not appear. | `CustomerIssuedCollectionStaticExpiryDateChangeDao` | Mock `mongoOperation.updateMulti()`; capture `Query` arg; assert `Criteria` uses `$in` and `orgId == job.getOrgId()` | P0 |

---

#### Group B — DAO: EARNED-Dynamic Path (`CustomerEarnedCollectionDynamicExpiryDateChangeDao`)

| ID | Req | Description | Class Under Test | Mock Strategy | Priority |
|----|-----|-------------|-----------------|---------------|----------|
| UT-05 | FR-002 | `updateValidTillDate()` override: same `$in(batchIds)` + `orgId` criteria. Update object is an `AggregationUpdate` (not `Update`) — verify the update type is not changed. | `CustomerEarnedCollectionDynamicExpiryDateChangeDao` | Mock `mongoOperation.updateMulti()`; capture both `Query` and `UpdateDefinition` args; assert criteria correct AND update is `instanceof AggregationUpdate` | P0 |
| UT-06 | FR-002 | T1 (task): Update existing `shouldUpdateValidTillDate` test to pass `List<ObjectId>` instead of `ObjectId`. Existing assertions must still pass after signature change. | `CustomerEarnedCollectionDynamicExpiryDateChangeDaoTest` | Existing mocks, updated call signature | P0 |

---

#### Group C — Semaphore: `ExpiryDateChangeJobFacade`

| ID | Req | Description | Class Under Test | Mock Strategy | Priority |
|----|-----|-------------|-----------------|---------------|----------|
| UT-07 | FR-003 | `tryAcquireSemaphore()` returns `true` when `redisTemplate.opsForValue().increment()` returns a value ≤ `maxGlobalConcurrentJobs` (e.g., increment returns 2, max=3). | `ExpiryDateChangeJobFacade` | `@Mock RedisTemplate<String,String>` + `ValueOperations`; stub `increment()` → 2L | P0 |
| UT-08 | FR-003 | `tryAcquireSemaphore()` DECRs and returns `false` when increment returns > max (e.g., 4, max=3). Verify `decrement()` called exactly once. | `ExpiryDateChangeJobFacade` | Stub `increment()` → 4L; verify `decrement()` called | P0 |
| UT-09 | FR-004 | `tryAcquireSemaphore()` returns `true` (fail-open) and does NOT re-throw when `redisTemplate.opsForValue().increment()` throws `RedisConnectionFailureException`. | `ExpiryDateChangeJobFacade` | Stub `increment()` throws `RedisConnectionFailureException` | P0 |
| UT-10 | FR-005 | `releaseSemaphore()` calls `decrement()` on success path. | `ExpiryDateChangeJobFacade` | Stub `decrement()` → 1L; verify called with `SEMAPHORE_KEY` | P0 |
| UT-11 | FR-005 | `releaseSemaphore()` does NOT throw when `redisTemplate` throws (swallows exception). | `ExpiryDateChangeJobFacade` | Stub `decrement()` throws; assert no exception propagates | P0 |
| UT-12 | FR-006 | `changeExpiryForPromotion()` throws `RuntimeException` with message containing `"concurrency limit reached"` when `tryAcquireSemaphore()` returns `false`. | `ExpiryDateChangeJobFacade` | Spy facade; stub `tryAcquireSemaphore` → false | P0 |
| UT-13 | FR-005 | `changeExpiryForPromotion()` calls `releaseSemaphore()` in `finally` even when the processing block throws a `RuntimeException`. | `ExpiryDateChangeJobFacade` | Mock service to throw; verify `releaseSemaphore` called via spy | P0 |
| UT-14 | FR-007 | `reconcileSemaphoreCounter(n)` calls `redisTemplate.opsForValue().set(SEMAPHORE_KEY, "n")`. Verify `"promotion_engine:expiry_job:running_count"` is the key used. | `ExpiryDateChangeJobFacade` | Stub `set()`; capture args | P0 |
| UT-15 | FR-007 | `reconcileSemaphoreCounter(-1)` clamps to 0 and sets `"0"` (not `"-1"`). | `ExpiryDateChangeJobFacade` | Stub `set()`; assert set called with `"0"` | P1 |
| UT-16 | FR-007 | `reconcileSemaphoreCounter()` does NOT throw when `redisTemplate.opsForValue().set()` throws. | `ExpiryDateChangeJobFacade` | Stub `set()` throws; assert no exception | P0 |

---

#### Group D — Reconciliation: `ExpiryDateChangeHelper`

| ID | Req | Description | Class Under Test | Mock Strategy | Priority |
|----|-----|-------------|-----------------|---------------|----------|
| UT-17 | FR-007, UC-4 | `updateExpiryDateChangeJobs()` calls `reconcileSemaphoreCounter` AFTER the shard loop with the sum of `countByStatus(RUNNING)` across all shards — not inside the loop. Verify call order: shard loop first, then reconcile. | `ExpiryDateChangeHelper` | Mock `expiryDateChangeJobDao.countByStatus()` → 1 per shard (2 shards = 2 total); verify `reconcileSemaphoreCounter(2)` called; verify it is not called until after the shard loop | P0 |
| UT-18 | FR-007 | When only 1 shard present, `reconcileSemaphoreCounter` called with that shard's count (not double-counted). | `ExpiryDateChangeHelper` | 1 shard key; stub `countByStatus` → 3; assert `reconcileSemaphoreCounter(3)` | P1 |

---

#### Group E — Factory: `SpringAmqpConfig`

| ID | Req | Description | Class Under Test | Mock Strategy | Priority |
|----|-----|-------------|-----------------|---------------|----------|
| UT-19 | FR-008 | `expiryDateChangeContainerFactory()` calls `configurer.configure(factory, connectionFactory)` as the FIRST operation before any `set*` call. | `SpringAmqpConfig` | Mock `ConnectionFactory` + `RabbitListenerContainerFactoryConfigurer`; verify `configure()` called before `setConcurrentConsumers()` via `InOrder` | P0 |
| UT-20 | FR-008, NFR-005 | Factory returned by `expiryDateChangeContainerFactory()` has `prefetchCount=1`, `concurrentConsumers=1`, `maxConcurrentConsumers=1`. | `SpringAmqpConfig` | Call method with mocks; assert via `ReflectionTestUtils.getField` or accessor if accessible | P1 |

---

### Integration Tests — `src/test/java/.../integration/`

Use `SpringJUnit4ClassRunner` + `BaseIntegrationTest`. These use embedded MongoDB.
Redis behaviour is verified via unit tests (Group C); ITs focus on end-to-end DB correctness and cross-org isolation.

| ID | Req | Description | Setup | Assertion | Priority |
|----|-----|-------------|-------|-----------|----------|
| IT-01 | FR-001, UC-1, UC-9 | **Regression gate:** Existing `PromotionExpiryDateChangeIntegrationTest` suite passes unchanged after `$in` change. `impactedCount` matches seeded doc count; `validTill` updated correctly on ISSUED path. | No change to setup — use as-is | All existing assertions pass; `impactedCount == seedCount` | P0 |
| IT-02 | FR-002, UC-1 | **EARNED-dynamic path:** Create promotion with dynamic validity, issue earning, trigger expiry date change job; assert `validTill` on earned coupon updated to new promotion end date. | Seed earned coupon document; trigger expiry job via facade | `validTill` on earned doc = new end date; `impactedCount = 1` | P0 |
| IT-03 | NFR-002 | **Tenant isolation:** Seed org-A and org-B issued coupons for the same `promotionId`. Run expiry date change job scoped to org-A. Assert org-B `validTill` is unchanged. | Seed 2 issued coupons for orgId-A and 1 for orgId-B with same promotionId | org-A coupon `validTill` updated; org-B coupon `validTill` unchanged | P0 |
| IT-04 | FR-007, UC-4 | **Counter reconciliation:** Directly set Redis semaphore key to an inflated value (e.g., 5) before the scheduler runs. Run `updateExpiryDateChangeJobs()`. Assert Redis key equals actual RUNNING count (e.g., 0 if no jobs are RUNNING). | Set Redis key `promotion_engine:expiry_job:running_count` = 5 in embedded Redis; assert no jobs in RUNNING state | Redis key value = 0 after reconciliation | P0 |
| IT-05 | FR-001 | **Multi-batch iteration:** Seed more issued coupons than `batchSize` (e.g., `batchSize + 1` documents). Run job; assert all coupons' `validTill` updated and `impactedCount` = total seeded. | Seed `batchSize + 1` documents for one promotion | `impactedCount` = total seeded; all `validTill` updated | P1 |

---

### Automation Tests — `campaigns_auto/tests/promotion_engine/`

> **Confirmed gap:** No existing automation test exercises `PROMOTION_EXPIRY_UPDATE_QUEUE`
> processing. `test_Expiry_Redeem.py` only asserts `validTill` at issue time. Two new test
> files are required.

---

#### File: `test_ExpiryJob_EndToEnd.py` — Functional correctness

| ID | Req | Description | Dev-only? | Poll Strategy | Cleanup | Priority |
|----|-----|-------------|-----------|---------------|---------|----------|
| AT-DEV-01 | FR-001, UC-1, UC-9 | **End-to-end job flow:** Create promotion (end date T+30d), issue coupon, assert initial `validTill`. Call `PeHelper.updatePromotionExpiry()` to move end date to T+15d. Poll issued coupon `validTill` every 5s; assert equals T+15d within 60s TTL. | Yes | `for _ in range(12): time.sleep(5); check; pytest.fail("…60s")` | Delete promotion + coupon | P1 |
| AT-DEV-02 | FR-001, FR-002 | **EARNED-dynamic path smoke:** Create promotion with dynamic validity, issue earning, call `updatePromotionExpiry()`. Poll earned coupon `validTill`; assert updated within 60s TTL. | Yes | Same poll-with-TTL as AT-DEV-01 | Delete promotion + earned coupon | P1 |

---

#### File: `test_ExpiryJob_Load.py` — Load & throughput validation (Nightly)

> **Purpose:** Validate NFR-006 (semaphore caps MongoDB concurrency) and NFR-007 (no messages
> lost under large batch). These tests use a larger data volume and longer TTL than the
> end-to-end tests. Run on Nightly cluster as part of the before/after load comparison protocol
> described in the Load Test Protocol section below.

| ID | Req | Description | Dev-only? | Poll Strategy | Cleanup | Priority |
|----|-----|-------------|-----------|---------------|---------|----------|
| AT-DEV-03 | FR-001, NFR-006 | **Single-job timing baseline:** Create 1 promotion, issue 200 coupons (≈ `batchSize / 10`), call `updatePromotionExpiry()`. Record elapsed time from trigger to all 200 coupons showing updated `validTill`. Assert completes < 90s. Capture timing as pre/post-deploy comparison datapoint. | Yes | `for _ in range(18): time.sleep(5); check all 200; break if done; else pytest.fail("…90s")` | Delete promotion + 200 coupons | P1 |
| AT-DEV-04 | NFR-006, NFR-007, UC-2 | **Large batch — all jobs drain within throttle window:** Create 20 promotions, issue 50 coupons each (= 40 RMQ messages). Call `updatePromotionExpiry()` for all 20 in rapid succession. Poll until all 1,000 coupons show updated `validTill`. Assert all complete < 180s (= 3 concurrent × ceil(40/3) × avg_job_time + buffer). This proves the throttle does not stall the system and all messages drain without dead-lettering. | Yes | `for _ in range(36): time.sleep(5); count updated coupons; break if count == 1000; else pytest.fail("…180s")` | Delete 20 promotions + 1,000 coupons | P1 |
| AT-DEV-05 | NFR-007 | **No message loss under large batch:** After AT-DEV-04 completes, assert that EVERY one of the 1,000 issued coupons has `validTill` == new promotion end date — no coupon left on old date. Proves no RMQ message was dead-lettered or silently dropped. | Yes | Run immediately after AT-DEV-04 drain; assert `count(updated) == 1000` | Shared cleanup with AT-DEV-04 | P1 |

---

### Automation Tests — Production Cluster

No prod-safe automation tests are appropriate for this feature. The change is internal
background job processing; validating it in production requires issuing coupons and mutating
promotion data. Marked **N/A** — post-deploy validation is covered by the Load Test Protocol
section below (Redis key monitoring + `opcounters.update` baseline comparison in Nightly).

---

## Tenant Isolation Test Cases

| ID | Scenario | Org-A Action | Org-B Assertion | Expected | Type |
|----|----------|-------------|-----------------|----------|------|
| TI-01 | Expiry date change job for org-A must not update org-B documents | Run expiry job for org-A (promotionId P, orgId=A) with org-A issued coupons seeded | Query org-B issued coupons for same promotionId P | `validTill` on org-B docs unchanged | IT (IT-03) |
| TI-02 | Redis semaphore is global — org-A job consuming a slot must reduce available slots visible to org-B job | Set Redis counter = `maxGlobalConcurrentJobs` via org-A job; attempt org-B job | org-B `tryAcquireSemaphore()` returns `false` | `false` returned; counter not double-incremented | UT (UT-08 — use different orgIds in log assertion) |
| TI-03 | `$in(batchIds)` list is pre-filtered by orgId in the aggregation phase — no cross-tenant ObjectId can appear | Inspect aggregation pipeline criteria in `getAggregation()` | Match filter includes `orgId = job.getOrgId()` | `$match` has `orgId` condition | UT (UT-01 captures aggregation arg) |

---

## Regression Coverage

| ID | Existing Behaviour | Assertion | Risk if Broken | Type |
|----|-------------------|-----------|---------------|------|
| RG-01 | `impactedCount` on `ExpiryDateChangeJob` equals number of documents actually updated per promotion | Assert `job.getImpactedCount() == seedDocCount` in IT-01 and IT-02 | Wrong `impactedCount` → ops team unable to verify batch completion | IT |
| RG-02 | `PromotionExpiryDateChangeIntegrationTest` — ISSUED static path end-to-end | Full suite passes with zero changes | $in change breaks existing test → merge blocker | IT (IT-01) |
| RG-03 | EARNED-dynamic path uses `AggregationUpdate` — criteria change must not collapse to `Update` type | Assert `instanceof AggregationUpdate` in UT-05 | Dynamic validity calculation (DateFromParts) silently replaced by simple date set | UT |
| RG-04 | `ExpiryDateChangeJobListener` continues to consume from `PROMOTION_EXPIRY_UPDATE_QUEUE` after `containerFactory` switch | AT-DEV-01 exercises full queue consumption | Routing break → messages silently drop | AT-DEV |
| RG-05 | Duplicate promotion update blocked (`UPDATE_EXPIRY_DATE_NOT_ALLOWED`) — `shouldProceed()` gate unchanged | Existing test UT-14 (`shouldProceed()` path) passes unchanged | Duplicate jobs run, double-update | UT (existing) |
| RG-06 | `releaseSemaphore` in `finally` — existing `changeExpiryForPromotion` error path must still call release | UT-13 asserts release called on exception | Counter permanently inflated; all expiry jobs blocked within minutes | UT |

---

## Load Test Protocol (Nightly — Before/After MongoDB CPU Comparison)

> **Goal:** Prove Optimisation 1 ($in) reduces MongoDB `docsExamined` per batch, and
> Optimisation 2 (semaphore) flattens the concurrent write spike during large brand batches.
> This is a structured before/after protocol run on Nightly cluster, not a CI gate.

### What To Measure

| Signal | Tool | What it proves |
|--------|------|----------------|
| `docsExamined` per update batch | MongoDB explain plan (Step 1 below) | Opt-1: $in eliminates range-scan overshoot |
| `opcounters.update` per second (peak vs sustained) | New Relic → MongoDB metrics | Opt-2: Semaphore flattens spike |
| Concurrent active update operations | `db.currentOp()` snapshot (Step 3 below) | Opt-2: At most 3 concurrent `updateMulti` at any time |
| Max Redis semaphore counter observed | AT-DEV-04 poll assertion | Opt-2: Counter never exceeds `maxGlobalConcurrentJobs` |

---

### Step 1 — Explain Plan Comparison (run on Nightly or local dev MongoDB, before and after deploy)

Run these two commands against `customerIssuedPromotion` with a real batch of ObjectIds. Compare output.

**Before deploy (range query):**
```javascript
db.customerIssuedPromotion.explain("executionStats").updateMany(
  { _id: { $gt: ObjectId("<startId>"), $lte: ObjectId("<endId>") } },
  { $set: { validTill: ISODate("2026-06-01") } }
)
// Key metrics to record:
//   executionStats.totalDocsExamined   ← will be > batchSize if IDs span other orgs
//   executionStats.totalKeysExamined
//   executionStats.executionTimeMillis
```

**After deploy ($in query):**
```javascript
db.customerIssuedPromotion.explain("executionStats").updateMany(
  { _id: { $in: [ ObjectId("..."), ObjectId("..."), /* batchIds */ ] },
    orgId: <orgId> },
  { $set: { validTill: ISODate("2026-06-01") } }
)
// Key metrics to record:
//   executionStats.totalDocsExamined   ← must equal len(batchIds) exactly
//   executionStats.totalKeysExamined   ← must equal len(batchIds)
//   executionStats.executionTimeMillis
```

**Pass condition:**

| Metric | Before | After | Target |
|--------|--------|-------|--------|
| `totalDocsExamined` | > `batchSize` (range overshoot) | = `batchSize` | Reduction to batchSize |
| `totalKeysExamined` | ≥ `batchSize` | = `batchSize` | Equal |
| `executionTimeMillis` | Baseline | ≤ baseline | No regression |

> Record both sets of numbers in the PR description as evidence.

---

### Step 2 — Automated Load Test (AT-DEV-03, AT-DEV-04, AT-DEV-05)

Run `test_ExpiryJob_Load.py` on Nightly **before** and **after** the deploy.

| Test | Before deploy (expected) | After deploy (expected) |
|------|--------------------------|-------------------------|
| AT-DEV-03 (1 job, 200 coupons) | Completes within 90s (baseline timing) | Same or faster |
| AT-DEV-04 (20 jobs, 1,000 coupons) | Completes within 180s | Same or faster; Redis counter ≤ 3 |
| AT-DEV-05 (no message loss) | All 1,000 coupons updated | All 1,000 coupons updated |

> Capture AT-DEV-03 elapsed time as a before/after comparison datapoint in the PR.
> If after-deploy timing is significantly higher, investigate — the semaphore introduces
> queuing delay but should not increase per-job time.

---

### Step 3 — MongoDB `currentOp()` Snapshot During AT-DEV-04 (manual, ~30s into the run)

While AT-DEV-04 is running (large batch draining), execute on the Nightly MongoDB:

```javascript
db.currentOp({
  "ns": /customerIssuedPromotion|customerEarnedPromotion|codeBasedPromotion/,
  "op": "update",
  "active": true
})
```

**Before deploy:** Up to 6 concurrent update operations visible.
**After deploy:** ≤ 3 concurrent at any point (semaphore enforced).

---

### Step 4 — New Relic MongoDB Dashboard

During AT-DEV-04 batch run, capture a screenshot of the `opcounters.update` rate graph.

**Before deploy:** Sharp spike — all 6 pods hit MongoDB at once.
**After deploy:** Flatter profile — 3 pods max concurrent; same total work spread over longer window.

> Save both screenshots. Attach to PR as evidence of load reduction.

---

## Test Implementation Scaffolding

> **For `/test-case-sheet`:** This section maps every test ID to its target file, class,
> method name, required annotations, numbered setup steps, and exact assertion. An implementor
> can write the test directly from this table without reading the tech detail.

---

### Unit Tests

#### Group A — DAO `$in` (`CustomerIssuedCollectionStaticExpiryDateChangeDaoTest`)

**File:** `src/test/java/com/capillary/promotionengine/dao/CustomerIssuedCollectionStaticExpiryDateChangeDaoTest.java`
**Annotations on class:** `@RunWith(MockitoJUnitRunner.class)`
**Fields:** `@InjectMocks CustomerIssuedCollectionStaticExpiryDateChangeDao dao;` `@Mock MongoOperations mongoOperation;`

| ID | Method Name | Setup Steps | Assertion |
|----|-------------|-------------|-----------|
| UT-01 | `getAggregation_pushesAllIds_noSort` | 1. Build a stub `ExpiryDateChangeJob` with orgId=100, promotionId=500, startId=null. 2. Stub `mongoOperation.aggregate(any(), any(), any())` → empty `AggregationResults`. 3. Call `dao.getAggregation(job)`. 4. Capture `Aggregation` arg via `ArgumentCaptor<Aggregation>`. | Assert pipeline contains a `GroupOperation` with `push(_id)` as field `"ids"`. Assert no `SortOperation` in pipeline. Assert `MatchOperation` has orgId and promotionId criteria. |
| UT-02 | `updateValidTill_multipleIds_advancesStartId` | 1. Build stub job. 2. Build `BasicDBList` with 3 `ObjectId`s → `AggregationResults` with `BATCH_IDS` key. 3. Stub aggregate: first call → 3-ID result; second call → empty. 4. Spy `dao` to capture `updateValidTillDate` args. | Assert `updateValidTillDate` called exactly once with the 3-ID list. Assert `job.getStartId()` == 3rd ObjectId after loop. |
| UT-03 | `updateValidTill_emptyBatchIds_exitsImmediately` | 1. Build stub job. 2. Stub aggregate → `AggregationResults` with empty `BasicDBList`. | Assert `mongoOperation.updateMulti()` never called. |
| UT-04 | `updateValidTillDate_usesInCriteria_notRange` | 1. Build stub job (orgId=100). 2. Build `List<ObjectId>` of 3 IDs. 3. Build stub `PromotionMeta`. 4. Stub `mongoOperation.updateMulti()` → `UpdateResult.acknowledged(1,1L,null)`. 5. Capture `Query` arg. | Assert captured query's criteria document contains `$in` key under `_id`. Assert criteria contains `orgId: 100`. Assert no `$gt`/`$lte` range operators in criteria. |

---

#### Group B — EARNED-Dynamic Path

**File:** `src/test/java/com/capillary/promotionengine/dao/CustomerEarnedCollectionDynamicExpiryDateChangeDaoTest.java`
**Annotations on class:** `@RunWith(MockitoJUnitRunner.class)`

| ID | Method Name | Setup Steps | Assertion |
|----|-------------|-------------|-----------|
| UT-05 | `updateValidTillDate_earnedDynamic_usesInCriteriaAndAggregationUpdate` | 1. Build stub job (orgId=200). 2. Build `List<ObjectId>` of 2 IDs. 3. Build stub `PromotionMeta`. 4. Stub `mongoOperation.updateMulti(query, update, class, collection)` → acknowledged result. 5. Capture `Query` and `UpdateDefinition` via `ArgumentCaptor`. | Assert captured `Query` uses `$in` on `_id`. Assert captured `UpdateDefinition instanceof AggregationUpdate`. Assert criteria has orgId=200. |
| UT-06 | `shouldUpdateValidTillDate_existingTest_passesWithListSignature` | Update existing test method signature: replace `ObjectId endId` param with `List<ObjectId>` containing one `ObjectId`. | All existing assertions pass unchanged. |

---

#### Group C — Semaphore (`ExpiryDateChangeJobFacadeTest`)

**File:** `src/test/java/com/capillary/promotionengine/service/impl/ExpiryDateChange/ExpiryDateChangeJobFacadeTest.java`
**Annotations on class:** `@RunWith(MockitoJUnitRunner.class)`
**New fields to add:**
```java
@Mock private RedisTemplate<String, String> redisTemplate;
@Mock private ValueOperations<String, String> valueOperations;
```
**New @Before setup to add:**
```java
when(redisTemplate.opsForValue()).thenReturn(valueOperations);
ReflectionTestUtils.setField(facade, "redisTemplate", redisTemplate);
ReflectionTestUtils.setField(facade, "maxGlobalConcurrentJobs", 3);
```

| ID | Method Name | Setup Steps | Assertion |
|----|-------------|-------------|-----------|
| UT-07 | `tryAcquireSemaphore_withinLimit_returnsTrue` | 1. Stub `valueOperations.increment(SEMAPHORE_KEY)` → `2L`. 2. Call `facade.tryAcquireSemaphore("job-1", 100L)`. | Returns `true`. `decrement()` never called. |
| UT-08 | `tryAcquireSemaphore_atLimit_returnsFalseAndDecrements` | 1. Stub `increment()` → `4L` (max=3). 2. Call `tryAcquireSemaphore("job-1", 100L)`. | Returns `false`. `valueOperations.decrement(SEMAPHORE_KEY)` called exactly once. |
| UT-09 | `tryAcquireSemaphore_redisDown_failOpenReturnsTrue` | 1. Stub `increment()` throws `new RedisConnectionFailureException("down")`. 2. Call `tryAcquireSemaphore("job-1", 100L)`. | Returns `true`. No exception thrown. |
| UT-10 | `releaseSemaphore_success_callsDecrement` | 1. Stub `valueOperations.decrement(SEMAPHORE_KEY)` → `1L`. 2. Call `facade.releaseSemaphore("job-1", 100L)`. | `decrement(SEMAPHORE_KEY)` called once. No exception. |
| UT-11 | `releaseSemaphore_redisDown_noExceptionPropagates` | 1. Stub `decrement()` throws `new RedisConnectionFailureException("down")`. 2. Call `facade.releaseSemaphore("job-1", 100L)`. | No exception thrown (method must swallow). |
| UT-12 | `changeExpiryForPromotion_semaphoreLimitHit_throwsRuntimeException` | 1. Spy on facade. 2. Stub `tryAcquireSemaphore()` → `false` via `doReturn(false).when(spy).tryAcquireSemaphore(any(), any())`. 3. Build a valid `ExpiryDateChangeJob` with status=OPEN. 4. Stub `expiryDateChangeJobService.findById()` → the job. | `assertThrows(RuntimeException.class, () → spy.changeExpiryForPromotion(jobId))`. Message contains `"concurrency limit reached"`. |
| UT-13 | `changeExpiryForPromotion_processingThrows_semaphoreStillReleased` | 1. Spy on facade. 2. Stub `tryAcquireSemaphore()` → `true`. 3. Stub `expiryDateChangeJobService.findById()` → OPEN job. 4. Stub internal processing step to throw `RuntimeException("processing error")`. | `releaseSemaphore` called exactly once (verify via spy). |
| UT-14 | `reconcileSemaphoreCounter_positive_setsRedisKey` | 1. Stub `valueOperations.set(any(), any())`. 2. Call `facade.reconcileSemaphoreCounter(2L)`. | `valueOperations.set("promotion_engine:expiry_job:running_count", "2")` called once. |
| UT-15 | `reconcileSemaphoreCounter_negative_clampsToZero` | 1. Stub `valueOperations.set()`. 2. Call `facade.reconcileSemaphoreCounter(-1L)`. | `set("promotion_engine:expiry_job:running_count", "0")` called — NOT `"-1"`. |
| UT-16 | `reconcileSemaphoreCounter_redisDown_noExceptionPropagates` | 1. Stub `valueOperations.set()` throws `RedisConnectionFailureException`. 2. Call `facade.reconcileSemaphoreCounter(1L)`. | No exception thrown. |

---

#### Group D — Reconciliation (`ExpiryDateChangeHelperTest`)

**File:** `src/test/java/com/capillary/promotionengine/util/ExpiryDateChangeHelperTest.java`
**Annotations on class:** `@RunWith(MockitoJUnitRunner.class)`
**Fields:** `@InjectMocks ExpiryDateChangeHelper helper;` `@Mock ExpiryDateChangeJobFacade facade;` `@Mock ExpiryDateChangeJobDao expiryDateChangeJobDao;` `@Mock ExpiryDateChangeJobService expiryDateChangeJobService;`
**Setup:** Inject 2 shard keys via `ReflectionTestUtils.setField(helper, "shardKeys", Arrays.asList("shard-1", "shard-2"))`

| ID | Method Name | Setup Steps | Assertion |
|----|-------------|-------------|-----------|
| UT-17 | `updateExpiryDateChangeJobs_twoShards_reconcileCalledWithSum` | 1. Stub `expiryDateChangeJobDao.countByStatus(RUNNING)` → `1L` (each call returns 1). 2. Call `helper.updateExpiryDateChangeJobs()`. 3. Use `InOrder inOrder = inOrder(expiryDateChangeJobService, facade)` to verify ordering. | `inOrder.verify(expiryDateChangeJobService, times(2)).markAllJobAsExpiredIfNotRunning()` called (shard loop). Then `inOrder.verify(facade).reconcileSemaphoreCounter(2L)` called (post-loop, sum=2). `reconcileSemaphoreCounter` called exactly once. |
| UT-18 | `updateExpiryDateChangeJobs_oneShard_reconcileCalledWithExactCount` | 1. Inject 1 shard key. 2. Stub `countByStatus(RUNNING)` → `3L`. 3. Call `updateExpiryDateChangeJobs()`. | `facade.reconcileSemaphoreCounter(3L)` called once. Not `6L` (no double-count). |

---

#### Group E — Factory (`SpringAmqpConfigTest`)

**File:** `src/test/java/com/capillary/promotionengine/queue/SpringAmqpConfigTest.java`
**Annotations on class:** `@RunWith(MockitoJUnitRunner.class)`
**Fields:** `@InjectMocks SpringAmqpConfig config;` `@Mock ConnectionFactory connectionFactory;` `@Mock RabbitListenerContainerFactoryConfigurer configurer;`

| ID | Method Name | Setup Steps | Assertion |
|----|-------------|-------------|-----------|
| UT-19 | `expiryDateChangeContainerFactory_configurerCalledFirst` | 1. Call `config.expiryDateChangeContainerFactory(connectionFactory, configurer)`. 2. Capture `SimpleRabbitListenerContainerFactory` arg passed to `configurer.configure()`. 3. Use `InOrder inOrder = inOrder(configurer)`. | `inOrder.verify(configurer).configure(any(SimpleRabbitListenerContainerFactory.class), eq(connectionFactory))` is the first call. `configure()` called before any `setConcurrentConsumers` call. |
| UT-20 | `expiryDateChangeContainerFactory_hardPinsValues` | 1. Call `config.expiryDateChangeContainerFactory(connectionFactory, configurer)`. 2. Capture returned `SimpleRabbitListenerContainerFactory`. | `ReflectionTestUtils.getField(factory, "concurrentConsumers") == 1`. `ReflectionTestUtils.getField(factory, "maxConcurrentConsumers") == 1`. `ReflectionTestUtils.getField(factory, "prefetchCount") == 1`. |

---

### Integration Tests

**File:** `src/test/java/.../integration/ExpiryDateChangeIntegrationTest.java`
(Extend `PromotionBaseIntegrationTest` — same pattern as existing `PromotionExpiryDateChangeIntegrationTest`)
**Annotations on class:** `@RunWith(SpringJUnit4ClassRunner.class)` `@SpringBootTest`

| ID | Method Name | Key Setup Steps | Key Assertion |
|----|-------------|-----------------|---------------|
| IT-01 | (existing suite) | Run without any change | All existing assertions pass |
| IT-02 | `earnedDynamicPath_expiryDateChange_updatesValidTill` | 1. Create promotion with dynamic validity (DateFromParts config). 2. Issue earning for org-A coupon. 3. Build `ExpiryDateChangeJob` with status=OPEN for the promotion. 4. Call `expiryDateChangeJobFacade.changeExpiryForPromotion(jobId)`. | `customerEarnedPromotion` document's `validTill` = new end date. `job.getImpactedCount() == 1`. |
| IT-03 | `expiryDateChangeJob_onlyUpdatesOwningOrg` | 1. Seed 2 issued coupons for orgId=100, 1 for orgId=200, same promotionId=500. 2. Create job for orgId=100. 3. Run job. | orgId=100 coupons: `validTill` = new date. orgId=200 coupon: `validTill` unchanged. `job.getImpactedCount() == 2`. |
| IT-04 | `reconcileSemaphoreCounter_inflatedCounter_healsToActualRunningCount` | 1. Inject `RedisTemplate` and set `promotion_engine:expiry_job:running_count` = `"5"`. 2. Ensure no jobs in RUNNING state. 3. Call `expiryDateChangeHelper.updateExpiryDateChangeJobs()`. | `redisTemplate.opsForValue().get("promotion_engine:expiry_job:running_count") == "0"`. |
| IT-05 | `updateValidTill_moreThanBatchSize_allDocumentsUpdated` | 1. Seed `batchSize + 1` issued coupons for one promotion. 2. Create and run job. | `job.getImpactedCount() == batchSize + 1`. All `batchSize + 1` coupons have updated `validTill`. |

---

### Automation Tests

#### `test_ExpiryJob_EndToEnd.py`

**File:** `campaigns_auto/tests/promotion_engine/test_ExpiryJob_EndToEnd.py`
**Class:** `TestExpiryJobEndToEnd`
**Pattern:** `setup_class` → create org context; `teardown_method` → delete created entities

| ID | Method Name | Key Setup Steps | Assertion |
|----|-------------|-----------------|-----------|
| AT-DEV-01 | `test_issued_validtill_updated_after_expiry_date_change` | 1. `PeHelper.createPromotion(end_date=today+30)` → `promo_id`. 2. `PeHelper.issuePromotion(promo_id)` → `coupon_id`. 3. Assert initial `validTill = today+30`. 4. `PeHelper.updatePromotionExpiry(promo_id, new_end=today+15)`. 5. Poll loop: `for _ in range(12): time.sleep(5); vt = PeHelper.getCouponValidTill(coupon_id); if vt == today+15: break; else: pytest.fail(…)`. | `validTill == today+15` within 60s |
| AT-DEV-02 | `test_earned_validtill_updated_after_expiry_date_change` | Same pattern as AT-DEV-01 using earned coupon (dynamic validity promo). `PeHelper.issueEarning(promo_id)` → `coupon_id`. | `validTill == today+15` within 60s |

#### `test_ExpiryJob_Load.py`

**File:** `campaigns_auto/tests/promotion_engine/test_ExpiryJob_Load.py`
**Class:** `TestExpiryJobLoad`
**Note:** These tests seed more data and run longer. Mark with `@pytest.mark.nightly` so they are excluded from fast CI runs and only execute on Nightly cluster.

| ID | Method Name | Key Setup Steps | Assertion |
|----|-------------|-----------------|-----------|
| AT-DEV-03 | `test_single_job_timing_baseline` | 1. Create 1 promotion. 2. Issue 200 coupons. 3. Record `start = time.time()`. 4. `PeHelper.updatePromotionExpiry(promo_id, new_end)`. 5. Poll until all 200 coupons have `validTill == new_end` (sample check via `PeHelper.getCouponValidTill`). | All 200 coupons updated within 90s. Print `elapsed = time.time() - start` for before/after comparison. |
| AT-DEV-04 | `test_large_batch_drains_within_retry_window` | 1. Create 20 promotions. 2. Issue 50 coupons per promotion (1,000 total). 3. Call `PeHelper.updatePromotionExpiry()` for all 20 in a loop. 4. Poll: count total coupons with updated `validTill` every 5s. TTL = 180s (36 iterations). | `count_updated == 1000` within 180s. If TTL expires: `pytest.fail("Large batch did not drain — possible dead-lettering or semaphore deadlock")`. |
| AT-DEV-05 | `test_large_batch_no_messages_lost` | 1. Reuse same 20 promotions + 1,000 coupons from AT-DEV-04 (or re-seed if independent). 2. After full drain confirmed, query ALL 1,000 coupon `validTill` values. | `assert all(vt == new_end_date for vt in all_valid_tills)`. Zero coupons on old date. |

---

## Test Coverage Summary

| Layer | Count | ~% of Total |
|-------|-------|-------------|
| Unit | 20 | ~61% |
| Integration | 5 | ~15% |
| Automation — E2E (`test_ExpiryJob_EndToEnd.py`) | 2 | ~6% |
| Automation — Load (`test_ExpiryJob_Load.py`) | 3 | ~9% |
| Automation (prod) | 0 | 0% |
| Manual (Load Test Protocol — Nightly) | 3 steps | ~9% |
| **Total** | **33** | **100%** |

Requirements coverage: **9/9 FRs** (100%), **7/7 NFRs** (100%)

---

## Test Data Requirements

### Unit Tests
- `ExpiryDateChangeJob` stubs: any `jobId`, `orgId=100`, `promotionId=500`, `startId=null`
- `PromotionMeta` stubs: any `validTillDate`, `zoneOffset=UTC+5:30`
- `RedisTemplate<String,String>` mock — `@Mock` in `ExpiryDateChangeJobFacadeTest`; add `@Mock ValueOperations<String,String>` and stub `redisTemplate.opsForValue()` → the mock ops
- `ReflectionTestUtils.setField(facade, "maxGlobalConcurrentJobs", 3)` to inject `@Value` field
- `ReflectionTestUtils.setField(helper, "shardKeys", Arrays.asList("shard-1","shard-2"))` for Group D

### Integration Tests
- Use MongoDB-assigned document `_id` values — **never hardcode ObjectId strings**
- Seed issued coupons with distinct `orgId` values (orgId=100 vs orgId=200) for IT-03
- For IT-04: use embedded Redis (`spring.redis.*` in test `application.properties`); set key directly with injected `RedisTemplate` in `@Before`
- No blueprint tables needed — expiry date change job operates on `customerIssuedPromotion`, `customerEarnedPromotion`, `codeBasedPromotion` collections directly

### Automation Tests
- Use existing `PeHelper` setup pattern (class-level `setup_class`, per-test `teardown_method`)
- `PeHelper.updatePromotionExpiry()` confirmed at `PeHelper.py:760`
- AT-DEV-03/04/05: mark `@pytest.mark.nightly` — excluded from fast CI; runs on Nightly cluster only
- AT-DEV-04/05 share seed data (1,000 coupons) — run as a pair; teardown deletes all 20 promotions + coupons

---

## Minimum Viable Test Set (fast-release gate)

These P0 tests are the minimum gate before merging. All must pass.

**Unit (P0):**
UT-01, UT-02, UT-04, UT-05, UT-06, UT-07, UT-08, UT-09, UT-12, UT-13, UT-14, UT-16, UT-17, UT-19

**Integration (P0):**
IT-01, IT-02, IT-03, IT-04

**Automation E2E — run post-deploy to QA (P1, required before merge to staging):**
AT-DEV-01

**Automation Load — run on Nightly post-deploy (P1, required before prod rollout):**
AT-DEV-03 (timing baseline), AT-DEV-04 (drain), AT-DEV-05 (no message loss)

**Full coverage set:** all P0 + P1 (UT-03, UT-10, UT-11, UT-15, UT-18, UT-20, IT-05, AT-DEV-02) + Load Test Protocol (Steps 1–4).

---

## Recommended Execution Order

1. **Unit tests** — run first; gate on zero failures before anything else
2. **Integration tests** — run after unit gate; IT-01 (regression) runs first within the suite
3. **Automation E2E (AT-DEV-01, AT-DEV-02)** — run post-deploy to QA cluster
4. **Automation Load (AT-DEV-03, AT-DEV-04, AT-DEV-05) + Load Test Protocol Steps 1–4** — run on Nightly cluster before prod rollout; capture before/after screenshots for PR
5. **Post-deploy prod validation** — watch Redis key `promotion_engine:expiry_job:running_count` during first real brand batch; confirm stays ≤ 3, returns to 0 after drain

---

## Definition of Done

- [ ] All P0 unit tests pass (Groups A–E)
- [ ] All P0 integration tests pass with embedded MongoDB + embedded Redis
- [ ] `PromotionExpiryDateChangeIntegrationTest` suite passes with zero modifications (RG-02)
- [ ] EARNED-dynamic path (`AggregationUpdate` + `$in`) verified end-to-end in IT-02
- [ ] Tenant isolation confirmed: org-B documents unchanged after org-A job (IT-03)
- [ ] Counter reconciliation confirmed: inflated Redis key corrected to actual RUNNING count (IT-04)
- [ ] `configurer.configure()` verified as first call in factory bean (UT-19)
- [ ] AT-DEV-01 automation smoke passes on QA cluster post-deploy (validTill updated within 60s)
- [ ] AT-DEV-04 large batch drains within 180s on Nightly — no dead-lettering (NFR-007)
- [ ] AT-DEV-03 timing baseline captured before AND after deploy — no regression
- [ ] Explain plan comparison (Load Test Protocol Step 1) shows `docsExamined == batchSize` after deploy
- [ ] New Relic `opcounters.update` screenshot attached to PR showing flattened spike (Step 4)
- [ ] No existing tests broken (RG-01 through RG-06)
- [ ] Redis key `promotion_engine:expiry_job:running_count` stays ≤ 3 during a real batch run (manual post-deploy check)
