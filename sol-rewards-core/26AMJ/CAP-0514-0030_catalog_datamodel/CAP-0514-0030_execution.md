# Execution Tracking: CAP-0514-0030 — Reward Catalogue Listing API (CQRS MongoDB Read Model)
**Repo:** sol-rewards-core
**Branch:** CAP-0514-0030_catalog_datamodel
**Started:** 2026-05-14
**Last updated:** 2026-05-14

---

## Progress

| Block | Tasks Total | Tasks Done | Status |
|-------|-------------|-----------|--------|
| Foundational Classes | 5 | 5 | ✅ DONE |
| RMQ Infrastructure | 3 | 3 | ✅ DONE |
| Data Access Layer | 4 | 4 | ✅ DONE |
| Service + API Layer | 4 | 4 | ✅ DONE |
| Facade Modifications | 3 | 3 | ✅ DONE |
| Unit Testing | 2 | 2 | ✅ DONE |
| Integration Testing | 2 | 0 | 🔲 NOT STARTED |

---

## Task Log

| Task ID | Description | Status | Commit SHA | Started | Completed | Notes |
|---------|-------------|--------|-----------|---------|-----------|-------|
| T01 | `RewardCatalogueDoc` + `ActiveRewardDoc` MongoDB document classes | DONE | 11e9f4e9 | 2026-05-14 | 2026-05-14 | |
| T02 | `RewardCatalogueEvent` POJO | DONE | 1342bed4 | 2026-05-14 | 2026-05-14 | |
| T03 | `CatalogueIndexInitializer` — @PostConstruct ensureIndex for all 9 indexes (B2 fix) | DONE | 11e9f4e9 | 2026-05-14 | 2026-05-14 | B2 blocker fix applied |
| T04 | `SpringAmqpConfig` additions — exchange/queue/DLQ/binding/batchListenerFactory | DONE | 1342bed4 | 2026-05-14 | 2026-05-14 | |
| T05 | `RewardCatalogueEventPublisher` — catch-and-log publisher | DONE | 1342bed4 | 2026-05-14 | 2026-05-14 | |
| T06 | `URLPrefix` — add V1_REWARDS_CATALOGUE constant | DONE | 52aa9c06 | 2026-05-14 | 2026-05-14 | |
| T07 | `CursorToken` — encode/decode with cursor validation (A6 fix: 400 on malformed) | DONE | 52aa9c06 | 2026-05-14 | 2026-05-14 | |
| T08 | `RewardCatalogueRequest` DTO + fromUriInfo() parsing | DONE | 52aa9c06 | 2026-05-14 | 2026-05-14 | |
| T09 | `RewardCatalogueResponse` + `RewardSummaryDto` + `ActiveRewardQueryResult` DTOs | DONE | 52aa9c06 | 2026-05-14 | 2026-05-14 | |
| T10 | `ActiveRewardMongoRepository` — find/bulkUpsert/delete | DONE | 52aa9c06 | 2026-05-14 | 2026-05-14 | |
| T11 | `RewardCatalogueMongoRepository` — find/bulkUpsert/delete | DONE | 52aa9c06 | 2026-05-14 | 2026-05-14 | |
| T12 | `CatalogueCache` — Redis get/put/invalidate | DONE | 52aa9c06 | 2026-05-14 | 2026-05-14 | |
| T13 | `RewardCatalogueService` — orchestration with cache + cacheHit NR attr (A4 fix) | DONE | 52aa9c06 | 2026-05-14 | 2026-05-14 | |
| T14 | `RewardCatalogueResource` — JAX-RS endpoint | DONE | 52aa9c06 | 2026-05-14 | 2026-05-14 | |
| T15 | Register `RewardCatalogueResource` in `JerseyConfig` | DONE | 52aa9c06 | 2026-05-14 | 2026-05-14 | |
| T16 | Modify `RewardFacade.create()` — register afterCommit synchronization | DONE | a298a012 | 2026-05-14 | 2026-05-14 | |
| T17 | Modify `RewardService.save()` — register afterCommit synchronization | DONE | a298a012 | 2026-05-14 | 2026-05-14 | |
| T18 | Modify `RewardFacade.setStatusCategory()` — direct publish (A2 fix: correct method) | DONE | a298a012 | 2026-05-14 | 2026-05-14 | A2 fix applied |
| T19 | `RewardCatalogueSyncListener` — batch consumer, manual MDC (B1+A3 fix) | DONE | 1342bed4 | 2026-05-14 | 2026-05-14 | B1+A3 blockers fix applied |
| UT01 | Unit tests: CursorToken, Resource, Service, DocTest (UT-01 to UT-15) — 18 tests | DONE | 52aa9c06 | 2026-05-14 | 2026-05-14 | All 18 GREEN |
| UT02 | Unit tests: Cache, Publisher, Repo, Listener, Initializer (UT-16 to UT-40) — 25 tests | DONE | 52aa9c06 | 2026-05-14 | 2026-05-14 | All 25 GREEN |
| IT01 | Integration tests: CatalogueEndToEndIT (IT-01 to IT-15, RG-01 to RG-03) | PENDING | — | — | — | Next up |
| IT02 | Integration tests: CatalogueTenantIsolationIT (TI-01 to TI-05) | PENDING | — | — | — | Next up |

---

## Blockers and Actions Applied

### B1 — @RmqMessageTracker ClassCastException (BLOCKER) ✅ RESOLVED
- Resolution: `@RmqMessageTracker` removed from `onBatch()`. Manual MDC (3 keys) + MetricsService tracking added in `processOrgBatch()`.
- Reference: `PointsRedemptionListener.java` pattern for MDC setup.

### B2 — Compound indexes never created (BLOCKER) ✅ RESOLVED
- Resolution: `CatalogueIndexInitializer.ensureIndexes()` creates ALL 8 compound indexes + TTL index explicitly via `mongoTemplate.indexOps().ensureIndex()`.
- NO reliance on `@CompoundIndexes` annotations being processed at runtime (MongoDbInitializer.createIndexes() is commented out at line 46).

### A1 — Image URL field confusion ✅ RESOLVED
- Resolution: `buildCatalogueDoc()` uses `details.getImagePath()` (column `IMAGE_PATH`), NOT `details.getImageId()` (column `IMAGE_URI`).

### A2 — Wrong method name in RewardFacade ✅ RESOLVED
- Resolution: Direct publish added to `setStatusCategory()` at line ~1650, NOT `updateStatus()` at line 798.

### A3 — Missing MDC keys in processOrgBatch ✅ RESOLVED
- Resolution: All 3 MDC keys set: `X_CAP_API_AUTH_ORG_ID`, `REQUEST_ID_MDC`, `INTERNAL_REQUEST_ID_MDC`. All 3 removed in `finally` block.

### A4 — catalogue.cacheHit NR attribute missing ✅ RESOLVED
- Resolution: `metricsService.addCustomParameter("catalogue.cacheHit", ...)` added in `RewardCatalogueService.list()`.

### N6 — THIRTY_SECOND_CACHE constant — REMOVED from task list
- Per review: `CatalogueCache` uses hardcoded `Duration.ofSeconds(30)` via `RedisTemplate` — no `@Cacheable` bucket needed.

---

## Gaps, Deviations, and Observations

### T16/T17 — isSynchronizationActive() guard added
**Type:** Deviation  
**Description:** `TransactionSynchronizationManager.registerSynchronization()` wrapped in `if (isSynchronizationActive())` check. The `else` branch was intentionally omitted — both methods are `@Transactional` in production so synchronization is always active. The guard exists purely to prevent `IllegalStateException` in unit test contexts where no transaction is active.  
**Reason:** Existing `RewardServiceTest` and `RewardFacadeTest` tests call `save()` and `setStatusCategory()` without a Spring transaction context.  
**Risk:** Low — production path unchanged; the guard is a no-op in production.  
**Action:** `@Mock private RewardCatalogueEventPublisher catalogueEventPublisher` added to both existing test classes.

### UT coverage mapping vs test plan IDs
**Type:** Observation  
**Description:** Test plan IDs UT-01 to UT-43 were mapped to 43 tests across 9 test classes. The implemented tests cover all P0 cases. Some test plan IDs were consolidated (e.g. UT-09/UT-10 merged into single cache hit/miss tests in RewardCatalogueServiceTest). Net count: 43 tests implemented matching the plan total.  
**Risk:** None.

---

## Full Test Results (2026-05-14)

| Scope | Tests Run | Failures | Errors | Result |
|-------|-----------|----------|--------|--------|
| New unit tests (43) | 43 | 0 | 0 | ✅ GREEN |
| Full module suite | 3366 | 0 | 0 | ✅ BUILD SUCCESS |

---

## Open Items

| Question | Owner | Due | Status |
|---------|-------|-----|--------|
| Integration test infrastructure: confirm Testcontainers MongoDB + RabbitMQ containers start cleanly in CI | Dev | Before PR merge | 🔲 Open |
| IT-04 (TTL index test, 60s poll) — include or skip in CI gate? | Dev | Before PR merge | 🔲 Open |

---

## Phase 2 — v2 MongoDB Routing (CAP-0514-0030-V2)

**Added:** 2026-05-14  
**Status:** All tasks PENDING — arch investigation complete, handoff document at `CAP-0514-0030_v2_handoff.md`

| Task ID | Description | Status | Commit SHA | Started | Completed | Notes |
|---------|-------------|--------|-----------|---------|-----------|-------|
| V01 | Expand `ActiveRewardDoc` with all RewardFilter fields (startDate, group, label, tier, type, redemptionType, vendorId, categoryIds, priority, rewardRank, createdOn) | PENDING | — | — | — | 11 new fields; see handoff §6a |
| V02 | Expand `RewardCatalogueDoc` with display fields needed for response (description, thumbnailPath, thumbnailId, termAndConditionsPath, termAndConditionsId, imageId, group, label, tier, type, redemptionType, vendorId, categoryIds, priority, rewardRank, programId) | PENDING | — | — | — | 16 new fields; see handoff §6b |
| V03 | Add new indexes to `CatalogueIndexInitializer` for all filter patterns (10 new compound indexes on active_rewards) | PENDING | — | — | — | See handoff §7; follow existing ensureIndex pattern |
| V04 | Expand `RewardCatalogueSyncListener.processUpserts()` to populate new fields; inject VendorRedemptionRepository + RewardCategoryRepository | PENDING | — | — | — | See handoff §9 |
| V05 | `CatalogueMongoEnabledOrgs` — env var reader component; reads CATALOGUE_MONGO_ENABLED_ORGS; logs enabled orgs on startup | PENDING | — | — | — | See handoff §8a |
| V06 | `UserRewardMongoService` — MongoDB query service; buildCriteria(), buildSort(), buildDtos(), buildPagingDto(); skip guard at skip > 1000 | PENDING | — | — | — | See handoff §8b |
| V07 | `UserRewardResource.getAllForBrand` — add @QueryParam("v2"), isV2Eligible() check, routing to v2 or MySQL fallback | PENDING | — | — | — | See handoff §8c |
| V08 | Observability: add NR attribute constants to NewRelicConstants; emit catalogue.backend, catalogue.filterHash, catalogue.elapsedMs, catalogue.resultCount on both paths; add X-CAP-Data-Freshness response header on v2 path | PENDING | — | — | — | See handoff §10 |
| V09 | Data correctness sampling — async compare mongo vs mysql count; emit CatalogueCountMismatch custom event on mismatch; controlled by catalogue.mongo.sampling.rate property | PENDING | — | — | — | See handoff §8d |
| VUT01 | Unit tests for CatalogueMongoEnabledOrgs, UserRewardMongoService (buildCriteria all filters, skip guard, fallback triggers), v2 routing in resource | PENDING | — | — | — | See handoff §12 Tests section |
| VIT01 | Integration tests: v2 filter coverage (all supported filters), fallback for each unsupported filter, skip guard, response shape field-by-field vs MySQL, data correctness sampling, disabled org | PENDING | — | — | — | See handoff §12 Tests section |
