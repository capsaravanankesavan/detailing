# Test Plan: Reward Catalogue Listing API (CQRS MongoDB Read Model)
**Date:** 2026-05-14
**Input:** `tech-detail-reviewed.md` + codebase reconnaissance
**Scope:** New `GET /core/v1/rewards/catalogue` endpoint backed by MongoDB CQRS read model (`active_rewards` + `rewards_catalogue`), fed via RabbitMQ batch listener on reward mutations, cached in Redis at 30s TTL.
**Confidence:** HIGH

---

## Test Infrastructure

**Unit tests**: JUnit 5 (Jupiter) + Mockito — `@ExtendWith(MockitoExtension.class)` — confirmed from `RedisLockServiceTest.java`.
**Integration tests**: `@SpringBootTest` extending `BaseIntegrationTest` — Testcontainers (MongoDB 5.0, MySQL, RabbitMQ, Redis) — confirmed from `BaseIntegrationTest.java:242`. Package: `com.capillary.solutions.rewards.integeration` (existing typo preserved).
**No automation / Python test suite** in this repo — all test coverage is unit + integration.

---

## Requirements Summary

### Functional Requirements

| ID | Requirement |
|----|-------------|
| FR-001 | `GET /core/v1/rewards/catalogue` returns active rewards for an org with optional filters (groupIds, labelIds, pointsMin, pointsMax, pointsExact) |
| FR-002 | Reward create triggers afterCommit UPSERT event; `active_rewards` + `rewards_catalogue` populated async |
| FR-003 | Reward update (pointsValue, dates, etc.) triggers afterCommit UPSERT; both collections re-synced |
| FR-004 | Reward enable/disable toggle triggers direct UPSERT publish; disabled reward removed from `active_rewards` |
| FR-005 | Cursor-based pagination with stable compound key `(pointsValue ASC, _id ASC)` — no page skips or duplicates across calls |
| FR-006 | Redis 30s TTL cache for listing results; invalidated on any mutation for the affected org |
| FR-007 | Batch RMQ listener groups events by orgId before MySQL IN queries |
| FR-008 | `rewards_catalogue` stores denormalised reward detail (name, imageUrl via `getImagePath()`, langDetails, customFields) for projection |
| FR-009 | MongoDB TTL index on `active_rewards.expiresAt` auto-purges expired rewards |
| FR-010 | Listener failure → re-throw → RMQ retry → DLQ; idempotent replay safe |
| FR-011 | `pointsExact` mutually exclusive with `pointsMin` / `pointsMax` — 400 on conflict |
| FR-012 | Malformed cursor returns 400 not 500 |
| FR-013 | `afterCommit` registration must NOT fire on MySQL transaction rollback |
| FR-014 | Publish failure in `RewardCatalogueEventPublisher` must not affect the reward write API response |
| FR-015 | All MongoDB compound indexes created on startup via `CatalogueIndexInitializer` |

### Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| NFR-001 | p95 < 150ms for `GET /core/v1/rewards/catalogue` — cache miss target 30–45ms |
| NFR-002 | Tenant isolation: Org-A query must not return Org-B rewards via any path |
| NFR-003 | `maxTimeMS(100)` enforced on all MongoDB queries |
| NFR-004 | Eventual consistency lag acceptable (seconds); stale listing during RMQ processing window is expected |
| NFR-005 | `add("isEnabled").is(true)` is redundant in queries (all docs in `active_rewards` are active) — no functional impact |

### Ambiguities and Assumptions

| # | Assumption | Breaks if |
|---|-----------|-----------|
| 1 | `total` in API response = total active rewards for org, NOT filter-match count | If clients expect filter-aware total, pagination math is wrong |
| 2 | A reward with null startDate or endDate is not considered active | Business allows rewards with open-ended dates |
| 3 | `isEnabled=true` default filter — catalogue never returns disabled rewards | Business needs a "preview disabled rewards" mode |
| 4 | Labels and groupIds from MySQL are `Long` type; cursor `_id` is MongoDB ObjectId hex | Type mismatch on the API contract side |

---

## Risk Assessment

| Area | Risk | Priority |
|------|------|----------|
| `@RmqMessageTracker` on batch listener | ClassCastException crashes listener; all events go to DLQ; MongoDB never populated | **P0** |
| `CatalogueIndexInitializer` missing compound indexes | All queries are collection scans; NFR-001 cannot be met | **P0** |
| afterCommit registration fires on rollback | Catalogue syncs data that was never committed to MySQL | **P0** |
| Batch orgId grouping bug | Org-A MySQL IN query contaminated with Org-B reward IDs — cross-tenant data | **P0** |
| `getImagePath()` vs `getImageId()` confusion | Catalogue returns URI IDs instead of actual image URLs | **P0** |
| Cursor tampered / malformed → 500 | Unnecessary 5xx alert noise; client cannot recover | **P1** |
| Count query does not apply filters | `total` misleads cursor-aware clients | **P1** |
| `catalogue.cacheHit` NR attribute missing | Cache effectiveness unobservable post-deploy | **P1** |
| Empty `active_rewards` on first deploy | Load test returns 0 results until backfill runs | **P1** |
| TTL lag (≤60s) | Expired rewards appear briefly after `endDate` | **P2** |

---

## Test Case Specifications

### Unit Tests — `src/test/java/com/capillary/solutions/rewards/`

#### CursorToken

| ID | Req | Description | Class Under Test | Mock Strategy | Priority |
|----|-----|-------------|-----------------|---------------|----------|
| UT-01 | FR-005 | `encode(doc)` → `decode(encoded)` round-trip: same `pointsValue` and `id` hex | `CursorToken` | None (pure function) | **P0** |
| UT-02 | FR-012 | `decode("not-base64!!!")` throws exception; verify caller can catch it and return 400 | `CursorToken` | None | **P0** |
| UT-03 | FR-012 | `decode("")` throws exception | `CursorToken` | None | **P1** |
| UT-04 | FR-012 | `decode` of valid base64 but invalid JSON throws exception | `CursorToken` | None | **P1** |

#### RewardCatalogueResource

| ID | Req | Description | Class Under Test | Mock Strategy | Priority |
|----|-----|-------------|-----------------|---------------|----------|
| UT-05 | FR-011 | `pointsExact` + `pointsMin` in same request → 400 | `RewardCatalogueResource` | Mock `authDetailsProvider`, `catalogueService` | **P0** |
| UT-06 | FR-011 | `pointsExact` + `pointsMax` in same request → 400 | `RewardCatalogueResource` | Same | **P0** |
| UT-07 | FR-001 | Valid request with no filters → 200, delegates to `catalogueService.list()` | `RewardCatalogueResource` | Mock `catalogueService` returning empty response | **P0** |
| UT-08 | FR-001 | `orgId` always taken from `authDetailsProvider.get().getOrgId()`, never from query param | `RewardCatalogueResource` | Verify `catalogueService.list()` is called with provider orgId | **P0** |

#### RewardCatalogueService

| ID | Req | Description | Class Under Test | Mock Strategy | Priority |
|----|-----|-------------|-----------------|---------------|----------|
| UT-09 | FR-006 | Cache HIT → `activeRewardRepo.findFiltered()` NOT called, deserialized response returned | `RewardCatalogueService` | Mock `CatalogueCache.get()` returning cached JSON; mock `activeRewardRepo` | **P0** |
| UT-10 | FR-006 | Cache MISS → MongoDB queried → `cache.put()` called with serialized response | `RewardCatalogueService` | Mock `CatalogueCache.get()` returning empty; verify `cache.put()` invoked | **P0** |
| UT-11 | FR-001 | Empty org → `{items: [], total: 0, nextCursor: null}` | `RewardCatalogueService` | Mock repo returning empty list | **P0** |
| UT-12 | FR-005 | `activeDocs.size() == limit + 1` → `hasMore=true`, `nextCursor != null`, list trimmed to `limit` | `RewardCatalogueService` | Mock repo returning `limit+1` docs | **P0** |
| UT-13 | FR-005 | `activeDocs.size() <= limit` → `hasMore=false`, `nextCursor == null` | `RewardCatalogueService` | Mock repo returning exactly `limit` docs | **P0** |
| UT-14 | FR-001 | groupIds filter passed to `activeRewardRepo.findFiltered()` when provided | `RewardCatalogueService` | Verify `findFiltered()` receives request with `groupIds` | **P1** |
| UT-15 | FR-001 | labelIds filter passed to `activeRewardRepo.findFiltered()` | `RewardCatalogueService` | Verify request forwarded | **P1** |

#### ActiveRewardMongoRepository

| ID | Req | Description | Class Under Test | Mock Strategy | Priority |
|----|-----|-------------|-----------------|---------------|----------|
| UT-16 | NFR-002 | `orgId` is first predicate in `Criteria` for `findFiltered()` — verify using `ArgumentCaptor<Query>` | `ActiveRewardMongoRepository` | Mock `MongoTemplate`; capture `Query` arg | **P0** |
| UT-17 | NFR-003 | `maxTimeMS(100)` applied to find query — verify via captured `Query` | `ActiveRewardMongoRepository` | Same | **P0** |
| UT-18 | FR-005 | Cursor predicate: when cursor provided, `$or` contains `(pointsValue > V)` and `(pointsValue = V AND _id > id)` | `ActiveRewardMongoRepository` | Mock `MongoTemplate`; capture `Query` criteria | **P0** |
| UT-19 | FR-001 | groupId filter in `Criteria` when `request.getGroupIds()` non-empty | `ActiveRewardMongoRepository` | Capture `Query` | **P1** |
| UT-20 | FR-001 | labelId filter in `Criteria` when `request.getLabelIds()` non-empty | `ActiveRewardMongoRepository` | Capture `Query` | **P1** |
| UT-21 | FR-001 | `bulkUpsert()`: `BulkOperations.upsert()` called for each doc; verify orgId and rewardId in upsert filter | `ActiveRewardMongoRepository` | Mock `MongoTemplate.bulkOps()`; capture ops | **P0** |

#### RewardCatalogueSyncListener

| ID | Req | Description | Class Under Test | Mock Strategy | Priority |
|----|-----|-------------|-----------------|---------------|----------|
| UT-22 | FR-007 | Mixed-org batch `[{orgId:1, ...}, {orgId:2, ...}]` → `findByOrgIdAndIdIn` called once for orgId=1 and once for orgId=2 (never combined) | `RewardCatalogueSyncListener` | Mock `rewardRepository`; verify calls per org | **P0** |
| UT-23 | FR-002 | UPSERT event with active reward → `activeRewardRepo.bulkUpsert()` called with doc containing correct orgId | `RewardCatalogueSyncListener` | Mock repos; inject known reward with `isEnabled=true`, active dates | **P0** |
| UT-24 | FR-004 | UPSERT event with disabled reward → `activeRewardRepo.deleteByOrgIdAndRewardIdIn()` called; `bulkUpsert()` NOT called for this reward | `RewardCatalogueSyncListener` | Mock `Reward.isEnabled()` → false | **P0** |
| UT-25 | FR-004 | UPSERT event with expired reward (`endDate` past) → removed from `active_rewards` | `RewardCatalogueSyncListener` | Mock `Reward.endDate` before now | **P0** |
| UT-26 | FR-004 | UPSERT event with reward not yet started (`startDate` future) → removed from `active_rewards` | `RewardCatalogueSyncListener` | Mock `Reward.startDate` after now | **P0** |
| UT-27 | FR-010 | Reward not found in MySQL → WARN logged, no MongoDB writes, method returns (no exception) | `RewardCatalogueSyncListener` | Mock `rewardRepository.findByOrgIdAndIdIn()` returning empty | **P0** |
| UT-28 | FR-010 | Listener exception during `processOrgBatch()` → exception re-thrown from `onBatch()` | `RewardCatalogueSyncListener` | Mock `catalogueRepo.bulkUpsert()` throwing RuntimeException | **P0** |
| UT-29 | FR-006 | After successful UPSERT → `catalogueCache.invalidateOrg(orgId)` called exactly once per org in batch | `RewardCatalogueSyncListener` | Mock `catalogueCache`; verify invocation count | **P0** |
| UT-30 | FR-007 | Event with null `orgId` in batch → filtered out, never processed | `RewardCatalogueSyncListener` | Include null-orgId event in batch; verify no repo calls | **P1** |
| UT-31 | FR-007 | Event with null `rewardId` → filtered out | `RewardCatalogueSyncListener` | Same | **P1** |
| UT-32 | FR-010 | DELETE event → `catalogueRepo.deleteByOrgIdAndRewardIdIn()` + `activeRewardRepo.deleteByOrgIdAndRewardIdIn()` called | `RewardCatalogueSyncListener` | Mock repos; verify delete calls | **P0** |
| UT-33 | FR-007 | Same `rewardId` appears twice in batch (UPSERT+UPSERT duplicate) → idempotent; processed once (upsert semantics handle it) | `RewardCatalogueSyncListener` | Send two identical events; verify single MySQL fetch per reward | **P1** |

#### CatalogueCache

| ID | Req | Description | Class Under Test | Mock Strategy | Priority |
|----|-----|-------------|-----------------|---------------|----------|
| UT-34 | FR-006 | `get()` returns `Optional.empty()` on Redis exception (does not propagate) | `CatalogueCache` | Mock `RedisTemplate.opsForValue().get()` throwing exception | **P0** |
| UT-35 | FR-006 | `put()` silently absorbs Redis exception (does not propagate) | `CatalogueCache` | Mock `RedisTemplate.opsForValue().set()` throwing exception | **P0** |
| UT-36 | NFR-002 | `invalidateOrg(orgId=A)` only deletes keys matching `catalogue:A:*` — does NOT delete `catalogue:B:*` | `CatalogueCache` | Mock `redisTemplate.keys()` returning {`catalogue:A:f1:0`, `catalogue:B:f2:0`}; verify only A key in delete call | **P0** |
| UT-37 | FR-006 | `invalidateOrg()` silently absorbs Redis exception (does not propagate) | `CatalogueCache` | Mock `redisTemplate.keys()` throwing exception | **P0** |

#### RewardCatalogueEventPublisher

| ID | Req | Description | Class Under Test | Mock Strategy | Priority |
|----|-----|-------------|-----------------|---------------|----------|
| UT-38 | FR-014 | RabbitMQ `convertAndSend()` throws → exception caught and logged, method returns normally | `RewardCatalogueEventPublisher` | Mock `RabbitTemplate.convertAndSend()` throwing exception | **P0** |
| UT-39 | FR-002 | `publishUpsert()` sends JSON string with `orgId`, `rewardId`, `operation=UPSERT` | `RewardCatalogueEventPublisher` | Mock `RabbitTemplate`; capture message body via `ArgumentCaptor<String>` | **P0** |
| UT-40 | FR-002 | Published to correct exchange + routing key constants | `RewardCatalogueEventPublisher` | Verify exchange = `REWARDS_CATALOGUE_EXCHANGE`, key = `REWARDS_CATALOGUE_ROUTING_KEY` | **P0** |

#### CatalogueIndexInitializer

| ID | Req | Description | Class Under Test | Mock Strategy | Priority |
|----|-----|-------------|-----------------|---------------|----------|
| UT-41 | FR-015 | `ensureIndexes()` calls `mongoTemplate.indexOps()` at least 9 times — 8 compound + 1 TTL | `CatalogueIndexInitializer` | Mock `MongoTemplate`; capture all `indexOps()` calls | **P0** |
| UT-42 | FR-015 | TTL index on `active_rewards.expiresAt` has `expireAfterSeconds=0` | `CatalogueIndexInitializer` | Capture `Index` passed to `ensureIndex()` for `expiresAt`; assert `.expire(0)` | **P0** |

#### JerseyConfig (extend existing JerseyConfigTest)

| ID | Req | Description | Class Under Test | Priority |
|----|-----|-------------|-----------------|----------|
| UT-43 | FR-001 | `JerseyConfig` registers `RewardCatalogueResource.class` — verify constructor call does not throw and resource is registered | `JerseyConfig` | **P0** |

---

### Integration Tests — `com.capillary.solutions.rewards.integeration/`

All integration tests extend `BaseIntegrationTest`. Test containers: MongoDB 5.0, MySQL, RabbitMQ, Redis.
HTTP calls via `restTemplate` + `port`. MongoDB state verified via `mongoTemplate`.

#### End-to-End Sync Flow

| ID | Req | Description | Setup | Assertion | Priority |
|----|-----|-------------|-------|-----------|----------|
| IT-01 | FR-002 | Create reward via REST → RMQ event consumed → `active_rewards` + `rewards_catalogue` populated | POST to create reward API; wait up to 5s for async sync | `mongoTemplate.findOne(Query.query(Criteria.where("rewardId").is(rewardId)), ActiveRewardDoc.class)` not null; `rewards_catalogue` doc present | **P0** |
| IT-02 | FR-003 | Update reward pointsValue → new value reflected in `active_rewards` | Create reward; wait for sync; PUT update; wait for sync | `active_rewards` doc `pointsValue == newValue` | **P0** |
| IT-03 | FR-004 | Disable reward (`setStatusCategory`) → reward removed from `active_rewards` | Create + sync; disable via REST; wait for sync | `active_rewards` doc no longer exists for that rewardId | **P0** |
| IT-04 | FR-009 | Set `expiresAt` to past time on `active_rewards` doc directly → MongoDB TTL removes it (within 60s) | Insert doc with `expiresAt = Date(System.currentTimeMillis() - 1)` via `mongoTemplate.save()` | Poll up to 70s: `mongoTemplate.count(query, "active_rewards") == 0` | **P2** |

#### Catalogue GET API

| ID | Req | Description | Setup | Assertion | Priority |
|----|-----|-------------|-------|-----------|----------|
| IT-05 | FR-001 | GET catalogue — no filters — returns rewards for org | Pre-seed 3 `active_rewards` + `rewards_catalogue` docs via `mongoTemplate.save()` | Response: `items.size() == 3`, `total == 3` | **P0** |
| IT-06 | FR-001 | GET catalogue with `groupIds=G1` — returns only rewards having G1 | Seed 2 rewards: one with `groupIds=[G1,G2]`, one with `groupIds=[G3]` | Response: `items.size() == 1`, returned reward has `groupIds` containing G1 | **P0** |
| IT-07 | FR-001 | GET catalogue with `labelIds=L1` — returns only rewards having L1 | Seed rewards with different labelIds | Response contains only rewards with L1 | **P0** |
| IT-08 | FR-001 | GET catalogue with `pointsMin=100&pointsMax=500` — range filter | Seed rewards at 50, 200, 600 points | Response: only reward at 200 points returned | **P0** |
| IT-09 | FR-001 | GET catalogue with `pointsExact=300` — exact match | Seed rewards at 200, 300, 400 | Response: only reward at 300 returned | **P1** |
| IT-10 | FR-011 | GET catalogue with `pointsExact=300&pointsMin=100` → 400 | No seed needed | HTTP 400 | **P0** |
| IT-11 | FR-014 | Empty org — GET catalogue → `{items: [], total: 0, nextCursor: null}` | No seed data | Response: `items.isEmpty()`, `total == 0`, `nextCursor == null` | **P0** |

#### Cursor Pagination

| ID | Req | Description | Setup | Assertion | Priority |
|----|-----|-------------|-------|-----------|----------|
| IT-12 | FR-005 | Page 1 → `nextCursor` present → use cursor for page 2 → no duplicates, no skips | Seed 7 docs (default limit=5) | Page 1: 5 items + nextCursor. Page 2 with cursor: 2 items + no nextCursor. All 7 unique rewardIds across pages | **P0** |
| IT-13 | FR-005 | Cursor from page 1 still valid after cache invalidation (cursor decoded from base64 each time) | Same as IT-12; manually invalidate Redis after page 1 | Page 2 returns correct items after cache bust | **P1** |

#### Cache Behaviour

| ID | Req | Description | Setup | Assertion | Priority |
|----|-----|-------------|-------|-----------|----------|
| IT-14 | FR-006 | Two consecutive GET requests within 30s — second is a cache HIT (Redis key exists) | Seed docs; first GET (cache miss); second GET within 1s | After second GET: `redisTemplate.hasKey("catalogue:{orgId}:*")` returns true | **P0** |
| IT-15 | FR-006 | Cache invalidated after reward mutation | Seed docs; first GET (cache miss → sets cache); trigger mutation (disable reward via REST); wait for sync; verify Redis key gone | `redisTemplate.keys("catalogue:{orgId}:*")` returns empty set | **P0** |

#### Regression — Existing API Behaviour Unchanged

| ID | Req | Description | Setup | Assertion | Priority |
|----|-----|-------------|-------|-----------|----------|
| RG-01 | FR-013 | Reward create API with forced transaction rollback → NO event published, `active_rewards` not populated | Create reward with invalid data that passes service but fails at final step; or force via exception in test | No `active_rewards` doc for that rewardId after 3s wait | **P0** |
| RG-02 | FR-014 | Reward create succeeds normally; reward create response 200 even if RMQ is down | Stop RMQ container momentarily; create reward | HTTP 200; reward exists in MySQL; no `active_rewards` doc (expected — sync failed) | **P0** |
| RG-03 | FR-003 | Reward update API — response 200 unchanged; no extra latency introduced by afterCommit registration | Update reward via REST | HTTP 200 within 500ms | **P1** |

---

### Tenant Isolation Tests

| ID | Scenario | Org-A Action | Org-B Assertion | Expected | Type |
|----|----------|-------------|-----------------|----------|------|
| TI-01 | Cross-tenant GET | Org-A has 3 active rewards in `active_rewards`; Org-B has none | GET catalogue with `X-CAP-API-AUTH-ORG-ID: B` | `items.size() == 0` — Org-B sees nothing | IT |
| TI-02 | Cross-tenant cache invalidation | Org-A reward mutation → invalidation runs with Org-A orgId | Verify Org-B cache key `catalogue:B:*` still present in Redis | Org-B key NOT deleted | IT |
| TI-03 | Cross-tenant batch listener | Send RMQ batch with `[{orgId:A, rewardId:R1}, {orgId:B, rewardId:R2}]` | Verify `findByOrgIdAndIdIn(A, [R1])` and `findByOrgIdAndIdIn(B, [R2])` — never `findByOrgIdAndIdIn(A, [R1, R2])` | MySQL IN queries are per-org | UT-22 |
| TI-04 | Cross-tenant index isolation | Org-A `active_rewards` has `orgId:A` on every doc | MongoDB query with `orgId:B` returns 0 docs when only Org-A docs exist | 0 results | IT (covered by TI-01) |
| TI-05 | Redis key isolation | Org-A cache key = `catalogue:A:hash:0`; Org-B = `catalogue:B:hash:0` | `invalidateOrg(A)` deletes only keys with `catalogue:A:` prefix | Org-B key survives | UT-36 |

---

## Test Coverage Summary

| Layer | Count | % of Total |
|-------|-------|-----------|
| Unit (UT-01 to UT-43) | 43 | ~72% |
| Integration (IT-01 to IT-15 + RG-01 to RG-03 + TI-01 to TI-05) | 23 | ~28% |
| **Total** | **66** | **100%** |

Requirements coverage: 15/15 FRs covered (100%), 5/5 NFRs covered (100%)

---

## Test Data Requirements

### MySQL (loaded by `DataSourceTestUtils.loadSchemaAndData()` for ITs)
- Standard seed data from `cc-stack-crm/seed_data/solutions`
- Integration tests create rewards via the REST API (using orgId from `X-CAP-API-AUTH-ORG-ID` header)

### MongoDB (via `mongoTemplate.save()` directly in test `@BeforeEach`)
- For IT-05 through IT-13: insert `ActiveRewardDoc` and `RewardCatalogueDoc` with known `orgId`, `rewardId`, `pointsValue`, `groupIds`, `labelIds`, `expiresAt` (future date) directly
- Use `@AfterEach`: `mongoTemplate.dropCollection("active_rewards"); mongoTemplate.dropCollection("rewards_catalogue")` to isolate tests

### Redis
- Tests inherit Redis container from `BaseIntegrationTest`
- Verify keys with `redisTemplate.keys("catalogue:{orgId}:*")` — use the autowired `RedisTemplate` from `BaseIntegrationTest`

### Org IDs
- `orgId = 1L` — standard IT org (seeded by MySQL dump)
- `orgId = 2L` — second org for tenant isolation tests
- Never hardcode `rewardId` — use `reward.getId()` from create response

---

## New Test Classes to Create

| Class | Package | Extends | Purpose |
|-------|---------|---------|---------|
| `CursorTokenTest` | `com.capillary.solutions.rewards.dto` | — | UT-01 to UT-04 |
| `RewardCatalogueResourceTest` | `com.capillary.solutions.rewards.resource` | — | UT-05 to UT-08 |
| `RewardCatalogueServiceTest` | `com.capillary.solutions.rewards.service` | — | UT-09 to UT-15 |
| `ActiveRewardMongoRepositoryTest` | `com.capillary.solutions.rewards.mongoDao` | — | UT-16 to UT-21 |
| `RewardCatalogueSyncListenerTest` | `com.capillary.solutions.rewards.queue` | — | UT-22 to UT-33 |
| `CatalogueCacheTest` | `com.capillary.solutions.rewards.service` | — | UT-34 to UT-37 |
| `RewardCatalogueEventPublisherTest` | `com.capillary.solutions.rewards.queue` | — | UT-38 to UT-40 |
| `CatalogueIndexInitializerTest` | `com.capillary.solutions.rewards.config.mongo` | — | UT-41 to UT-42 |
| `CatalogueEndToEndIT` | `com.capillary.solutions.rewards.integeration` | `BaseIntegrationTest` | IT-01 to IT-15, RG-01 to RG-03 |
| `CatalogueTenantIsolationIT` | `com.capillary.solutions.rewards.integeration` | `BaseIntegrationTest` | TI-01 to TI-05 |

---

## Minimum Viable Test Set (fast-release gate)

P0 unit tests — must all pass before raising PR:
- UT-01, UT-02, UT-05, UT-06, UT-07, UT-08
- UT-09, UT-10, UT-11, UT-12, UT-13
- UT-16, UT-17, UT-18, UT-21
- UT-22, UT-23, UT-24, UT-25, UT-26, UT-27, UT-28, UT-29, UT-32
- UT-34, UT-35, UT-36, UT-37
- UT-38, UT-39, UT-40
- UT-41, UT-42

P0 integration tests — must all pass before deploying to QA:
- IT-01, IT-05, IT-06, IT-10, IT-11, IT-12, IT-14, IT-15
- RG-01, RG-02
- TI-01, TI-02

Full coverage set: all P0 + P1 + P2 tests.

---

## Recommended Execution Order

1. Unit tests (`mvn test -Dtest=CursorTokenTest,RewardCatalogueResourceTest,...`) — gate on zero failures, < 30s
2. Integration tests (`mvn test -Dgroups=integration`) — gate after unit tests pass, ~3-5 min
3. Manual post-deploy checks on QA:
   - `db.active_rewards.getIndexes()` — confirm all 9 indexes
   - `db.rewards_catalogue.getIndexes()` — confirm all 4 indexes
   - Trigger one reward create; verify `active_rewards` populated within 5s

---

## Definition of Done

- [ ] All 43 P0 unit tests pass (`mvn test`)
- [ ] All P0 integration tests pass with real Testcontainers (MongoDB + MySQL + RabbitMQ + Redis)
- [ ] `GET /core/v1/rewards/catalogue` returns data for seeded org in IT
- [ ] Cursor pagination: page 1 → page 2 produces no duplicates and no skips
- [ ] Redis cache HIT confirmed on second identical request within 30s window
- [ ] Cache invalidated after reward mutation (confirmed by Redis key absence)
- [ ] Tenant isolation: Org-A docs not returned for Org-B query
- [ ] Mixed-org batch: MySQL IN queries scoped per org (verified by UT-22)
- [ ] `afterCommit` does NOT fire on transaction rollback (RG-01)
- [ ] `RewardCatalogueEventPublisher` failure does not fail reward write API (RG-02)
- [ ] MongoDB indexes confirmed on startup log (`CatalogueIndexInitializer` INFO line)
- [ ] All existing tests continue to pass (`mvn test`)

---

## Pre-Implementation Checklist (from tech-detail-reviewed.md Blockers)

Before writing a single class, confirm these fixes from `tech-detail-reviewed.md` are in the implementation:

- [ ] **B1**: `@RmqMessageTracker` removed from `onBatch()`. Manual MDC + MetricsService tracking added in `processOrgBatch()`.
- [ ] **B2**: `CatalogueIndexInitializer.ensureIndexes()` creates ALL 8 compound indexes explicitly — not just TTL.
- [ ] **A1**: `buildCatalogueDoc()` uses `details.getImagePath()` not `details.getImageId()`.
- [ ] **A2**: The method modified in `RewardFacade` is `setStatusCategory()` (line 1636), not `updateStatus()`.
- [ ] **A3**: `processOrgBatch()` sets `REQUEST_ID_MDC` and `INTERNAL_REQUEST_ID_MDC` + cleans up in `finally`.
