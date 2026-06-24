# Test Plan: Reward Constraint Limit Breach & Duplicate Summary Rows Under Parallel Issue Calls

**Date:** 2026-06-24  
**Input:** `CAPJUN19_reward-limit-concurrency_techdetail.md` (same directory)  
**Scope:** Concurrency fix for Race A (wrong lock granularity), Race B (concurrent INSERT), Race C (blind UPDATE), and CQ9 (updateSummaries double-write) in `TBL_REWARD_ISSUE_SUMMARY`. REWARD-level and CUSTOMER-level constraints. ORG-level deferred to Phase 2.  
**Confidence:** HIGH — root cause confirmed; fix committed; T1/T5/T6 passing; T2/T3/T4 need T-10 resolution; T-13 pending.

---

## Requirements Summary

### Functional Requirements

| ID | Requirement | Source |
|----|-------------|--------|
| FR-001 | REWARD-level constraint limit must not be breached under concurrent cross-customer calls | UC-1, UC-2 |
| FR-002 | No duplicate `TBL_REWARD_ISSUE_SUMMARY` rows produced under concurrent first-issuance | UC-4, UC-9 |
| FR-003 | `CONSUMED` value must be exact after concurrent atomic increments (no lost updates) | UC-1, UC-4 |
| FR-004 | Compensation decrement (`CONSUMED = CONSUMED - delta`) must fire on any downstream failure after pos-13 write | UC-6 |
| FR-005 | CUSTOMER-level constraint serial enforcement unchanged (regression) | UC-3 |
| FR-006 | `updateSummaries()` must not overwrite atomically-written summary rows (CQ9 filter) | UC-7 |
| FR-007 | `CONSTRAINT_EVALUATION_FAILED` (code 12005) returned at per-reward level for both lock contention and re-evaluation failure inside lock | §10 API |
| FR-008 | ROLLING window REWARD-level constraints covered by Phase 1 lock (same write path as FIXED via `issualDate` resolution) | UC-10 |
| FR-009 | Mixed REWARD + CUSTOMER level constraints: REWARD-level limit enforced correctly while CUSTOMER-level also checked | UC-5 |
| FR-010 | FIXED/WEEKS and FIXED/MONTHS window constraints: concurrent cross-customer calls honour window boundaries correctly | UC-1 variant |
| FR-011 | NO_LIMIT independent constraint: single REWARD-level row; atomic increment correct under concurrency | UC-4 variant |
| FR-012 | NO_LIMIT dependent constraint (paired with FIXED window): concurrent threads produce exactly 1 shared row | UC-4 variant |
| FR-013 | FIXED/DAYS cross-window: issue on Day 0 (row at `ISSUE_DATE=day0`); issue on Day 31 creates a new row at `ISSUE_DATE=day31`; day0 consumed unchanged | Date boundary |
| FR-014 | `KPI.REDEMPTION_VALUE` REWARD-level concurrent: consumed = sum of `intouchPoints` values, no lost updates | UC-1 KPI variant |
| FR-015 | `KPI.POINTS` at Reward/Customer level: rejected at API validation (HTTP 400) before reaching processor chain | Validation gate |
| FR-016 | Multi-FIXED-window constraints at same `(Level, KPI)` (e.g., FIXED/DAYS + FIXED/MONTHS at REWARD/QUANTITY): consumed incremented exactly once per successful issuance; exactly 1 `TBL_REWARD_ISSUE_SUMMARY` row produced on first issuance — no duplicate rows, no double-increment | §5a GAP HIGH, Risk #8, T-18 |

### Non-Functional Requirements

| ID | Requirement | Source |
|----|-------------|--------|
| NFR-001 | `NonOrgSummaryWriteProcessor.process()` P99 ≤ 100ms under normal load (lock hold time ≤ 50ms) | §15 SLA Impact |
| NFR-002 | Lock key must be org-scoped: `reward_constraint:{orgId}:{rewardId}` — org A cannot block org B | §5d Tenant Isolation |
| NFR-003 | `LAST_UPDATED_ON` auto-updated by MySQL `ON UPDATE CURRENT_TIMESTAMP` for Databricks ETL delta pickup | §9 Data Model |
| NFR-004 | New Relic attribute `PARALLEL_CALL_LOCK_ACQUIRED` emitted on both lock acquisition and lock contention paths | §16 Observability |

### Ambiguities & Open Questions

| # | Question | Blocks | Owner |
|---|---------|--------|-------|
| OQ-1 | Does `solutionDbDatasource` participate in the outer Spring transaction (`DataSourceUtils.getConnection()`)? If so, `findExistingForNonOrgLevel()` JDBC SELECT sees uncommitted INSERTs from concurrent threads — T2/T3/T4 still fail | IT-T2, IT-T3, IT-T4 | Implementer (T-10) |
| OQ-2 | Current `redis.lock.ttl` and `redisLockAcquireMaxWaitTime` values in production — required ≥ 500ms and ≥ 5s respectively | NFR-001, IT-T2/T3/T4 test setup | DevOps |
| OQ-3 | Production DDL for `TBL_REWARD_ISSUE_SUMMARY` has `ON UPDATE CURRENT_TIMESTAMP` on `LAST_UPDATED_ON`? | NFR-003 | DBA |

---

## Risk Assessment

| Area | Risk | Priority |
|------|------|----------|
| T-10 (outer transaction context) | T2/T3/T4 still fail if `solutionDbDatasource` is transaction-managed — JDBC SELECT inside lock sees uncommitted concurrent INSERTs | P0 |
| CQ9 regression | `updateSummaries()` overwrites atomically-written rows if `atomicWrittenConstraintIds` filter is bypassed | P0 |
| Compensation path | `compensateFailedSummaryWrites()` silently fails or is not called on all failure paths — CONSUMED over-counted | P0 |
| Lock contention error code | Lock contention must return code 12005 at per-reward level (not a 500) — same as re-evaluation failure | P0 |
| ORG-level still unprotected | Phase 1 does NOT cover ORG-level (constraint level = ORG with `rewardId = -1L`) — existing Race A/C still applies | P1 |
| ROLLING window REWARD-level | Covered by Phase 1 but no explicit integration test for ROLLING window path | P1 |
| Tenant isolation via lock key | `reward_constraint:orgA:X` must not block `reward_constraint:orgB:X` | P1 |

---

## Test Case Specifications

### Unit Tests — `src/test/java/`

*Framework: JUnit 5 + Mockito — match existing IT class conventions. All DAO/service dependencies mocked.*

| ID | Requirement | Description | Class Under Test | Mock Strategy | Priority |
|----|-------------|-------------|-----------------|---------------|----------|
| UT-01 | FR-001 | `process()`: REWARD-level constraint present → lock acquired, re-eval passes, `writeNonOrgSummariesAtomically()` called, wrappers recorded | `NonOrgSummaryWriteProcessor` | Mock `redisLockService`, `rewardConstraintFacade`, `levelServiceProvider`, `metricsService` | P0 |
| UT-02 | FR-001 | `process()`: re-eval fails inside lock → wrapper added to `toFail`, no write called | `NonOrgSummaryWriteProcessor` | Same mocks; `levelService.evaluate()` returns false | P0 |
| UT-03 | FR-007 | `process()`: `acquireLock()` throws `MarvelException` → wrapper gets status code 12005, message contains "rewardId" | `NonOrgSummaryWriteProcessor` | `redisLockService.acquireLock()` throws `MarvelException` | P0 |
| UT-04 | FR-005 | `process()`: CUSTOMER-level constraint only (no REWARD-level) → no lock acquired; `writeNonOrgSummariesAtomically()` called without lock | `NonOrgSummaryWriteProcessor` | Mock `rewardConstraintFacade`; verify `acquireLock()` NOT called | P0 |
| UT-05 | FR-005 | `process()`: no non-org constraints at all → returns ctx immediately, no lock, no write | `NonOrgSummaryWriteProcessor` | `wrapper.getValidRewardConstraintList()` returns only ORG-level constraints | P1 |
| UT-06 | FR-002 | `writeNonOrgSummariesAtomically()`: existing row found → `bulkAtomicIncrement()` called; `save()` NOT called | `RewardConstraintFacade` | Mock `rewardIssueSummaryJdbcRepository.findExistingForNonOrgLevel()` returning 1 row; mock `bulkAtomicIncrement` | P0 |
| UT-07 | FR-002 | `writeNonOrgSummariesAtomically()`: no existing row → `save()` called; `bulkAtomicIncrement()` NOT called | `RewardConstraintFacade` | Mock `findExistingForNonOrgLevel()` returning empty list; mock `save()` | P0 |
| UT-08 | FR-006 | `updateSummaries()`: constraints in `atomicWrittenConstraintIds` are excluded from `buildUniqueSummaries()` | `RewardConstraintFacade` | Mock `getExistingRewardIssueSummariesForNonOrgLevel()`; populate `ctx.addHandledConstraintId()` with constraint ID | P0 |
| UT-09 | FR-006 | `updateSummaries()`: row IDs in `writtenSummaryRowIds` are excluded from non-org summary list passed to `bulkSaveOrUpdate()` | `RewardConstraintFacade` | Populate `ctx.recordWrittenSummaryRows()`; assert filtered rows not in save call | P0 |
| UT-10 | FR-004 | `compensateFailedSummaryWrites()`: failed wrappers' written rows decremented; successful wrappers not decremented | `RewardConstraintFacade` | Mock `rewardIssueSummaryJdbcRepository.bulkAtomicDecrement()`; ctx with 1 success + 1 failure | P0 |
| UT-11 | FR-004 | `compensateFailedSummaryWrites()`: no written rows for reward → `bulkAtomicDecrement()` not called | `RewardConstraintFacade` | `ctx.getWrittenSummaryRowsForCompensation()` returns empty list | P1 |
| UT-12 | FR-001 | `BulkRewardIssueContext.recordWrittenSummaryRows()` accumulates correctly across multiple `rewardId` keys | `BulkRewardIssueContext` | Pure unit — no mocks | P0 |
| UT-13 | FR-006 | `BulkRewardIssueContext.getAtomicWrittenConstraintIds()` contains IDs added via `addHandledConstraintId()` | `BulkRewardIssueContext` | Pure unit — no mocks | P0 |
| UT-14 | FR-004 | `BulkRewardIssueContext.getWrittenSummaryRowIds()` returns IDs from recorded summary rows for a given rewardId | `BulkRewardIssueContext` | Pure unit — no mocks | P0 |

---

### Integration Tests — `src/test/java/com/capillary/solutions/rewards/integeration/`

*Framework: Spring context, embedded MySQL, embedded Redis. Base class: `BaseIntegrationTest`. All tests in `RewardConstraintConcurrencyIntegrationTest`. Test property override: `redis.lock.maxWaitTime=5000`.*

| ID | Req | Description | Setup | Assertion | Status | Priority |
|----|-----|-------------|-------|-----------|--------|----------|
| IT-T1 | FR-005, FR-001 | Serial REWARD-level limit=2: 3 customers serially; 3rd rejected | Create reward (limit=2), stub coupon, issue MOBILE_1, MOBILE_2, MOBILE_3 serially | rows=1, consumed=2, r3 isConstraintFailure | ✅ Pass | P0 |
| IT-T2 | FR-001, FR-007 | Concurrent Race A: limit=1, 2 customers fire simultaneously; exactly 1 passes | Create reward (limit=1), CountDownLatch(1), 2 async threads | successCount=1, failCount=1, rows=1, consumed=1 | 🔄 Fix T-10 | P0 |
| IT-T3 | FR-002 | Concurrent Race B: limit=5, 2 customers fire simultaneously; no duplicate rows | Create reward (limit=5), CountDownLatch(1), 2 async threads | rows=1 (not 2), consumed=2, successCount=2 | 🔄 Fix T-10 | P0 |
| IT-T4 | FR-003, FR-006 | Concurrent Race C+CQ9: limit=10, 5 customers fire simultaneously; consumed=5 exact | Create reward (limit=10), CountDownLatch(1), 5 async threads | rows=1, consumed=5, successCount=5 | 🔄 Fix T-10 | P0 |
| IT-T5 | FR-005 | CUSTOMER-level serial enforcement: same customer issues 3×; 3rd rejected | Create reward CUSTOMER-level limit=2, MOBILE_1 issues 3× | customerRows=1, consumed=2, r3 isConstraintFailure | ✅ Pass | P0 |
| IT-T6 | FR-001 | Re-eval inside lock: limit=1, 1st customer issues; 2nd enters lock, re-reads consumed=1=limit, rejected without writing | Create reward (limit=1), serial issue MOBILE_1 then MOBILE_2 | rows=1, consumed=1, r2 isConstraintFailure code 12005 | ✅ Pass | P0 |
| IT-T7 | FR-004 | Compensation: downstream fails after pos-13 write; CONSUMED decremented back | Fault-inject coupon failure for MOBILE_1; reward with REWARD limit | CONSUMED = 0 after compensation; no rows OR consumed back to 0 | ⬜ Pending (T-13) | P0 |
| IT-T8 | FR-007 | Lock contention error path: simulate lock held externally; thread gets CONSTRAINT_EVALUATION_FAILED code 12005 | Set `redis.lock.maxWaitTime=1ms` (override); hold lock externally before issuing | r.getRewards().get(0).getStatus().getCode() == 12005; message contains "Concurrent request in progress" | ⬜ Pending | P1 |
| IT-T9 | NFR-002 | Tenant isolation: org A issues do not create rows in org B's `TBL_REWARD_ISSUE_SUMMARY` | Two distinct org IDs; reward created under org A; issue under org A | org B summary rows = 0; lock key distinct (no cross-org block) | ⬜ Pending | P1 |
| IT-T10 | FR-009 | Mixed REWARD + CUSTOMER level constraints: 5 customers concurrent, REWARD limit=3 | Create reward with both REWARD limit=3 and CUSTOMER limit=2; 5 different customers fire concurrently | successCount=3, REWARD-level rows=1, consumed=3; rejected customers get code 12005 | ⬜ Pending | P1 |
| IT-T11 | FR-008 | ROLLING window REWARD-level: 2 customers concurrent, ROLLING/30-day limit=1 | Create reward with `WindowType.ROLLING`, 2 concurrent customers | successCount=1; rows=1, consumed=1 — same enforcement path as FIXED | ⬜ Pending | P1 |
| IT-T12 | FR-010 | FIXED/WEEKS REWARD-level concurrent: limit=1, 2 customers fire simultaneously within the same weekly window | Create reward with `WindowType.FIXED`, `RepeatFrequencyType.WEEKS`, limit=1; CountDownLatch(1), 2 async threads | successCount=1; rows=1; consumed=1; `ISSUE_DATE` falls in same week | ⬜ Pending | P1 |
| IT-T13 | FR-010 | FIXED/MONTHS REWARD-level concurrent: limit=1, 2 customers fire simultaneously within same monthly window | Create reward with `WindowType.FIXED`, `RepeatFrequencyType.MONTHS`, limit=1; 2 async threads | successCount=1; rows=1; consumed=1 | ⬜ Pending | P1 |
| IT-T14 | FR-011 | NO_LIMIT independent REWARD-level concurrent: limit=1, 2 customers race; only 1 passes | Create reward with `RepeatFrequencyType.NO_LIMIT` (no other FIXED constraint at same level+KPI), limit=1; 2 async threads | successCount=1; REWARD-level rows=1; consumed=1 — confirms independent NO_LIMIT write path correct under concurrency | ⬜ Pending | P1 |
| IT-T15 | FR-012 | NO_LIMIT dependent REWARD-level concurrent: NO_LIMIT + FIXED/DAYS both present; 2 customers race | Create reward with FIXED/DAYS + NO_LIMIT at REWARD level, limit=2 each; 2 async threads | successCount=2; REWARD-level rows=1 (shared row); consumed=2 — dependent NO_LIMIT does not produce a second row | ⬜ Pending | P1 |
| IT-T16 | FR-013 | Cross-day FIXED/DAYS: issue on Day 0 (row at `ISSUE_DATE=day0`); advance clock by 31 days; issue concurrently on Day 31 | `DateTimeServiceStub.setClock(Clock.fixed(now, zone))` for Day 0; issue MOBILE_1; advance clock by 31 days; concurrent 2-thread issue on Day 31; `DateTimeServiceStub.restoreClock()` in `@AfterEach` | Day 0 row: consumed=1, `ISSUE_DATE=day0`. Day 31 rows: 1 row, consumed=1 (or 2 if limit allows both), `ISSUE_DATE=day31` ≠ `ISSUE_DATE=day0` — proves window boundary creates new row | ⬜ Pending | P1 |
| IT-T17 | FR-014 | `KPI.REDEMPTION_VALUE` REWARD-level concurrent: limit=10 (redemption value), 2 customers fire concurrently with redemptionValue=3 each | Create reward with `KPI.REDEMPTION_VALUE`, limit=10; configure `redemptionValue` on issue request; 2 async threads | rows=1; consumed = sum of `redemptionValue` (e.g., 6); no lost updates; successCount=2 | ⬜ Pending | P2 |
| IT-T18 | FR-015 | `KPI.POINTS` rejected at API level for Reward/Customer constraint | Create reward with `KPI.POINTS` on REWARD-level constraint | HTTP 400 with code 400, message "Reward/Customer Level do not support POINTS KPI" — never reaches processor chain | ✅ Covered (RewardConstraintIntegrationTest:1500) | P0 (regression guard) |
| IT-T19 | FR-016 | **Multi-FIXED-window REWARD-level (T-18 regression gate):** FIXED/DAYS limit=5 + FIXED/MONTHS limit=20 + NO_LIMIT at REWARD/QUANTITY; 2 concurrent customers; assert no double-increment and no duplicate row | Create reward with 3 REWARD-level constraints: `FIXED/DAYS limit=5`, `FIXED/MONTHS limit=20`, `NO_LIMIT`; 2 async threads; `CountDownLatch(1)` | `rows=1` (exactly 1 REWARD-level summary row); `consumed=2`; `successCount=2`; assert `SELECT COUNT(*) WHERE level=REWARD AND kpi=QUANTITY = 1` — confirms deduplication fix prevents duplicate INSERT and double-increment | ⬜ Pending (T-18 fix required) | **P0** |
| IT-T20 | FR-008, FR-012 | **NO_LIMIT + ROLLING/DAYS concurrent:** NO_LIMIT is dependent when ROLLING/DAYS is present; assert NO_LIMIT does NOT produce a second row | Create reward with `ROLLING/DAYS limit=1` + `NO_LIMIT` at REWARD/QUANTITY; 2 async threads | `rows=1` (only ROLLING row at `issualDate`); `consumed=1`; `successCount=1`; `SELECT COUNT(*) WHERE level=REWARD AND kpi=QUANTITY = 1` — confirms dependent NO_LIMIT skip produces no separate row | ⬜ Pending | P1 |
| IT-T21 | FR-016, FR-009 | **Multi-FIXED-window at both REWARD and CUSTOMER level concurrent:** FIXED/DAYS + FIXED/MONTHS + NO_LIMIT at REWARD/QUANTITY and CUSTOMER/QUANTITY; 2 concurrent customers; assert no double-increment at either level | Create reward with REWARD-level: `FIXED/DAYS limit=5`, `FIXED/MONTHS limit=20`, `NO_LIMIT`; CUSTOMER-level: `FIXED/DAYS limit=3`, `FIXED/MONTHS limit=10`; 2 async threads | REWARD-level `rows=1`, `consumed=2`; CUSTOMER-level: 1 row per customer (`consumed=1` each); no duplicate rows at any level; successCount=2 | ⬜ Pending (T-18 fix required) | P1 |

**Notes on IT-T16 (cross-day time travel) implementation guidance:**

```java
// Pattern: freeze clock at a known instant, issue, advance 31 days, issue again
Instant day0 = Instant.now();
DateTimeServiceStub.setClock(Clock.fixed(day0, ZoneId.of(orgZone)));
issueBulkRewards(buildIssueRequest(rewardId, MOBILE_1));  // writes row with ISSUE_DATE = day0

Instant day31 = day0.plus(31, ChronoUnit.DAYS);
DateTimeServiceStub.setClock(Clock.fixed(day31, ZoneId.of(orgZone)));
// now fire concurrent threads for Day 31
CountDownLatch latch = new CountDownLatch(1);
// ... concurrent issue for MOBILE_2, MOBILE_3
```
Always restore the clock in `@AfterEach`:
```java
@AfterEach
void restoreClock() { DateTimeServiceStub.restoreClock(); }
```
Pattern from `RewardConstraintIntegrationTest.java:207` and `BrandResourceIntegrationTest.java:1165`.

**Notes on IT-T7 (compensation test) implementation guidance:**
- Inject fault at coupon issue level (e.g., configure `IntouchServiceStub.issueBulkFunction1` to return 0 coupons after the pos-13 write for a target rewardId)
- Assert that `TBL_REWARD_ISSUE_SUMMARY.CONSUMED = 0` (decrement restores to pre-issuance state) OR the row does not exist (if it was a new row and the compensation sets consumed=0)
- Verify `compensateFailedSummaryWrites()` log line is emitted

---

### Automation Tests — Development Cluster (Python)

*Location: `campaigns_auto/tests/luci/`  
Run against: crm-nightly-new / devenv-crm  
These are dev-cluster-only — not prod-safe.*

| ID | Req | Description | Dev-only? | Cleanup Required | Priority |
|----|-----|-------------|-----------|-----------------|----------|
| AT-DEV-01 | FR-001 | Staging smoke: 50 concurrent `issueReward` for reward with REWARD limit=10; confirm exactly 10 succeed, 40 get code 12005; 1 DB row with consumed=10 | Yes | Delete test reward, summary rows | P0 (staging gate — §17 Rollout step 3) |
| AT-DEV-02 | NFR-001 | Latency: `NonOrgSummaryWriteProcessor.process()` P99 ≤ 100ms under AT-DEV-01 load — check New Relic `@Trace` data | Yes | None (read-only check) | P1 |
| AT-DEV-03 | NFR-004 | Observability: verify `PARALLEL_CALL_LOCK_ACQUIRED=true/false` attributes appear in New Relic after AT-DEV-01 run | Yes | None (read-only check) | P1 |

---

### Automation Tests — Production Cluster

*No production-destructive automation for this feature. All prod validation is observational post-deploy.*

| ID | Req | Description | Prod-safe? | Validation Method | Priority |
|----|-----|-------------|-----------|-------------------|----------|
| AT-PROD-01 | NFR-004 | Post-deploy: `PARALLEL_CALL_LOCK_ACQUIRED` attribute appears in New Relic within 15 min of first REWARD-level issue call (confirms new code path is live) | Yes — read-only | Query New Relic API for attribute presence; no writes | P1 |
| AT-PROD-02 | NFR-001 | Post-deploy 24h: `NonOrgSummaryWriteProcessor.process()` P99 ≤ 100ms in New Relic traces (confirms no lock contention buildup) | Yes — read-only | Query New Relic trace percentiles | P1 |

---

## Tenant Isolation Test Cases

| ID | Scenario | Org-A Action | Org-B Assertion | Expected | Type |
|----|----------|-------------|-----------------|----------|------|
| TI-01 | Separate-org lock isolation | Org A issues reward under lock `reward_constraint:orgA:rewardId` | Org B issues same logical rewardId under `reward_constraint:orgB:rewardId` simultaneously | No blocking; both complete independently | IT-T9 |
| TI-02 | DB row isolation | Org A issues reward X | Query `TBL_REWARD_ISSUE_SUMMARY WHERE ORG_ID = orgB AND REWARD_ID = X` | 0 rows — org A write cannot appear in org B namespace | IT-T9 |
| TI-03 | Atomic increment org scope | Org A atomically increments row | `ATOMIC_INCREMENT_SQL` includes `AND ORG_ID = :orgId` — org B row for same ID physically impossible | CONSUMED of org B row unchanged | Unit (UT) — already guaranteed by SQL |

---

## Regression Coverage

| ID | Existing Behaviour | Assertion | Risk if Broken | Type |
|----|-------------------|-----------|---------------|------|
| RG-01 | Serial REWARD-level limit enforcement (T1) | rows=1, consumed=2, 3rd request isConstraintFailure | Core constraint logic regressed | IT-T1 ✅ |
| RG-02 | CUSTOMER-level same-customer serialisation (T5) | customerRows=1, consumed=2, 3rd isConstraintFailure | Customer-level budget bypass | IT-T5 ✅ |
| RG-03 | Re-evaluation inside lock rejects at limit (T6) | rows=1, consumed=1 after r2 rejected | Lock purpose bypassed | IT-T6 ✅ |
| RG-04 | `updateSummaries()` still writes ORG-level rows (CQ9 filter excludes only non-org) | ORG-level summary row exists and consumed correct after issue | ORG-level constraint tracking silently broken | UT-08, UT-09 |
| RG-05 | `CONSTRAINT_EVALUATION_FAILED` message still mentions "Max limit" for re-eval failure | message.contains("Max limit") | API contract change breaks callers | IT-T1, IT-T2, IT-T5, IT-T6 |
| RG-06 | `CONSTRAINT_EVALUATION_FAILED` code is 12005 for lock contention path | code == 12005 | Callers cannot distinguish constraint failure from internal error | IT-T8 |

---

## Test Coverage Summary

| Layer | Count | Notes |
|-------|-------|-------|
| Unit Tests | 14 | UT-01 through UT-14 |
| Integration Tests | 21 | IT-T1 through IT-T21 (IT-T19 and IT-T21 blocked on T-18 fix; IT-T20 pending) |
| Automation (dev cluster) | 3 | AT-DEV-01 through AT-DEV-03 |
| Automation (prod-safe) | 2 | AT-PROD-01 through AT-PROD-02 |
| **Total** | **40** | |

FR coverage: 16/16 FRs (100%), NFR coverage: 4/4 NFRs (100%)

**Window/KPI coverage matrix:**

| Window | RepeatFrequency | KPI | Concurrent IT |
|--------|----------------|-----|---------------|
| FIXED | DAYS | QUANTITY | T1–T4 (core) |
| FIXED | WEEKS | QUANTITY | IT-T12 |
| FIXED | MONTHS | QUANTITY | IT-T13 |
| ROLLING | DAYS | QUANTITY | IT-T11 |
| — | NO_LIMIT (independent) | QUANTITY | IT-T14 |
| FIXED + NO_LIMIT (dependent) | DAYS + NO_LIMIT | QUANTITY | IT-T15 |
| FIXED + FIXED + NO_LIMIT (dependent) | DAYS + MONTHS + NO_LIMIT | QUANTITY | IT-T19 (T-18 fix gate — double-increment regression) |
| ROLLING + NO_LIMIT (dependent) | ROLLING/DAYS + NO_LIMIT | QUANTITY | IT-T20 (asserts NO_LIMIT skip produces 1 row not 2) |
| FIXED + FIXED + NO_LIMIT at REWARD + CUSTOMER | DAYS + MONTHS + NO_LIMIT | QUANTITY | IT-T21 (cross-level multi-window) |
| FIXED | DAYS | REDEMPTION_VALUE | IT-T17 |
| FIXED | DAYS | POINTS | IT-T18 (API rejects — no processor reach) |

---

## Test Data Requirements

| Resource | Requirement |
|---------|-------------|
| Reward creation | `buildRewardLevelConstraintRequest(limit)` helper (exists in test class) |
| Coupon stub | `setupCouponStub(rewardId)` — wires `IntouchServiceStub.issueBulkFunction1` (exists) |
| Customer IDs | `MOBILE_1` through `MOBILE_5` — each resolves to distinct customerId via `IntouchServiceStub` (exists) |
| Fault injection (IT-T7) | Configure `IntouchServiceStub.issueBulkFunction1` to return 0 coupons for target rewardId after pos-13 write |
| Redis max wait time | `redis.lock.maxWaitTime=5000` (set in `@TestPropertySource`) — required for T2/T3/T4; reduce to 1ms for IT-T8 |
| Embedded DB | H2/embedded MySQL seeded fresh per-test — no blueprint seed data needed (constraint limits evaluated from DB summary rows, not stats pipeline) |
| ROLLING window | `WindowType.ROLLING` + `RepeatFrequencyType.DAYS` in `buildRewardLevelConstraintRequest()` variant for IT-T11 |
| Org IDs | Existing embedded DB org IDs for isolation tests (IT-T9): use two distinct org IDs from test fixtures |

---

## Minimum Viable Test Set (release gate)

These P0 tests are the minimum before merging. All must pass:

**Unit (P0):** UT-01, UT-02, UT-03, UT-04, UT-06, UT-07, UT-08, UT-09, UT-10, UT-12, UT-13, UT-14

**Integration (P0):** IT-T1 ✅, IT-T2 (after T-10 fix), IT-T3 (after T-10 fix), IT-T4 (after T-10 fix), IT-T5 ✅, IT-T6 ✅, IT-T7 (compensation — T-13), IT-T19 (after T-18 fix — multi-FIXED-window deduplication gate)

**Staging gate:** AT-DEV-01 (50 concurrent, limit=10; 10 succeed, 1 row, consumed=10)

Full coverage set: All P0 + P1 tests above.

---

## Recommended Execution Order

1. **Unit tests** — run first; gate on zero failures
2. **Integration tests** (T-10 fix required for T2/T3/T4; T-13 for T7) — run after unit gate
3. **Staging smoke test** (AT-DEV-01) — run post-deploy to QA cluster; must pass before production
4. **Production observability checks** (AT-PROD-01, AT-PROD-02) — run 15 min and 24h after prod deploy

---

## Definition of Done

- [ ] All 14 P0 unit tests pass (UT-01 through UT-14 at P0)
- [ ] IT-T1, IT-T5, IT-T6 remain passing (regression gate — ✅ currently passing)
- [ ] IT-T2, IT-T3, IT-T4 pass after T-10 fix (OQ-1 resolved)
- [ ] IT-T7 passes: compensation decrement restores CONSUMED after downstream failure
- [ ] `CONSTRAINT_EVALUATION_FAILED` code 12005 confirmed for both re-eval failure and lock contention paths
- [ ] AT-DEV-01 staging smoke passes: 50 concurrent, limit=10; exactly 10 succeed; 1 row; consumed=10
- [ ] `PARALLEL_CALL_LOCK_ACQUIRED` attribute visible in New Relic within 15 min of prod deploy
- [ ] IT-T19 passes after T-18 fix: FIXED/DAYS + FIXED/MONTHS + NO_LIMIT at REWARD/QUANTITY concurrent → rows=1, consumed=2, no duplicate rows (deduplication regression gate)
- [ ] No existing tests broken (run full `RewardConstraintConcurrencyIntegrationTest` suite)
- [ ] OQ-1 (`solutionDbDatasource` transaction participation) resolved before T-10 can be closed

---

## Key Open Item: T-10 Resolution Path

T2, T3, T4 failures share the same root cause: `solutionDbDatasource` may participate in the outer Spring request transaction, so Thread A's JDBC INSERT is not committed when Thread B's JDBC SELECT runs inside the lock. Resolution options (for owner to decide):

| Option | Description | Risk |
|--------|-------------|------|
| A | Confirm `solutionDbDatasource` is NOT transaction-managed (auto-commit) → JDBC SELECT already sees committed rows → T-10 done | Low if confirmed; if wrong, races persist |
| B | `INSERT ... ON DUPLICATE KEY UPDATE` with a unique index on `(ORG_ID, REWARD_ID, KPI, CONSTRAINT_LEVEL, USER_ID, ISSUE_DATE)` — upsert avoids the read-before-write entirely | Medium — requires DB unique index (schema change) + upsert SQL |
| C | Separate datasource / new connection for `findExistingForNonOrgLevel()` to bypass transaction binding | Medium — connection pool management |

Until T-10 is resolved, T2/T3/T4 remain blocked for the integration gate.
