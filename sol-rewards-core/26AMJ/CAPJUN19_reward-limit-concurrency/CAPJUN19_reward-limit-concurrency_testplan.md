# Test Plan: Reward Constraint Limit Breach & Duplicate Summary Rows Under Parallel Issue Calls

**Date:** 2026-06-24  
**Input:** `CAPJUN19_reward-limit-concurrency_techdetail.md`  
**Scope:** DB-level correctness assertions for concurrency races, limit enforcement, ISSUE_DATE integrity, partial compensation (T-20), and legacy non-midnight row compatibility  
**Confidence:** HIGH  
**Revision:** v2 — adds ISSUE_DATE midnight assertions, non-midnight compatibility test, partial compensation test, and limit-breach boundary assertions

---

## Requirements Summary

### Functional Requirements (from §7 Use Cases)

- FR-001: REWARD-level limit enforced atomically under concurrent cross-customer calls (Race A fix)
- FR-002: No duplicate TBL_REWARD_ISSUE_SUMMARY rows for same logical key under concurrency (Race B fix)
- FR-003: CONSUMED = CONSUMED + delta; no lost updates under concurrent increments (Race C + CQ9 fix)
- FR-004: Compensation decrement fires on downstream failure; CONSUMED returns to pre-write value
- FR-005: ISSUE_DATE stored as midnight-stripped date (no time component) for every new row
- FR-006: Exact row count per (rewardId, level, kpi) — never more than 1 for same logical key
- FR-007: Limit rejection (CONSTRAINT_EVALUATION_FAILED) must not increment CONSUMED
- FR-008: Legacy rows with non-midnight ISSUE_DATE must be found by the lookup (or clearly documented as a compatibility gap requiring resolution before deploy)
- FR-009: Partial vendor issuance (qty=2, 1 success + 1 failure) → CONSUMED=1 after compensation (T-20)
- FR-010: T-18 dual-FIXED dedup — exactly 1 DB row per (level, kpi), consumed=N not 2N

### Non-Functional Requirements

- NFR-001: Existing tests T1–T8 must not regress
- NFR-002: New assertions must add < 50ms each to test execution (prefer appending to existing tests over new Spring context setups)
- NFR-003: All ISSUE_DATE values in new rows must be midnight in the org's timezone

### Ambiguities & Open Questions

- [ ] **FR-008 (non-midnight compatibility)**: `findExistingForNonOrgLevel()` uses exact equality `ISSUE_DATE = :date` (line 214 in `RewardIssueSummaryJdbcRepository`). A pre-existing row with a non-midnight timestamp will NOT be found; a new midnight row will be inserted instead → duplicate. **Decision needed before deploy**: fix the SQL to use `DATE(ISSUE_DATE) = DATE(:date)` (but be aware function-on-column disables index) OR run a one-time migration `UPDATE TBL_REWARD_ISSUE_SUMMARY SET ISSUE_DATE = DATE(ISSUE_DATE) WHERE TIME(ISSUE_DATE) != '00:00:00'`. Test T9 below is written to FAIL until this is resolved. — owner: implementer + DBA
- [ ] **T-20 partial compensation stub**: T11 requires configuring the vendor stub to return 1 success + 1 failure for qty=2. Confirm whether `IntouchServiceStub.issueBulkFunction1` can express partial success at qty level, or a different injection is needed. — owner: implementer

---

## Risk Assessment

| Area | Risk | Priority |
|------|------|----------|
| ISSUE_DATE non-midnight (new rows) | New rows written with time component bypass future dedup → new duplicate rows accumulate | P0 |
| Legacy non-midnight rows (FR-008) | Pre-existing rows invisible to new lookup → spurious second row per subsequent issuance | P0 |
| Partial compensation not covering partial qty (T-20) | Failed vendor qty not unwound → CONSUMED over-counts → limit reached early | P0 |
| Limit breach boundary after concurrent success | CONSUMED correct but boundary behavior untested | P1 |
| T-18 dual-FIXED dedup correctness | Double row or double increment accumulates silently | P1 |

---

## Test Case Specifications

### Assertions to ADD to Existing Tests (zero new test-class overhead)

These assertions are missing from T1–T8 and should be added inline. Each adds < 5ms.

| ID | Existing Test | Missing Assertion | Assert Sketch | Priority |
|----|--------------|-------------------|---------------|----------|
| DA-01 | `t1_serialRewardLevelLimitEnforcedCorrectly` | ISSUE_DATE is midnight-stripped | `assertMidnight(rows.get(0).getIssueDate())` | P0 |
| DA-02 | `t1_serialRewardLevelLimitEnforcedCorrectly` | No duplicate calendar-date rows | `assertNoDuplicateIssueDates(rows)` | P0 |
| DA-03 | `t3_concurrentIssueNoDuplicateSummaryRowsInserted` | ISSUE_DATE = midnight on the 1 row | `assertMidnight(rows.get(0).getIssueDate())` | P0 |
| DA-04 | `t3_concurrentIssueNoDuplicateSummaryRowsInserted` | No duplicate calendar-date rows | `assertNoDuplicateIssueDates(rows)` | P0 |
| DA-05 | `t6_pos13RejectsWhenLimitAlreadyReachedInsideLock` | Consumed unchanged after rejection (already asserted — add explicit comment) | `assertEquals(0, BigDecimal.ONE.compareTo(rows.get(0).getConsumed()), "rejection must not write")` | P0 |
| DA-06 | `t7_fixedDaysPlusFixedMonthsAtRewardLevelNoDoubleRowOrDoubleIncrement` | ISSUE_DATE = midnight; no duplicate dates | `assertMidnight(rows.get(0).getIssueDate())` + `assertNoDuplicateIssueDates(rows)` | P0 |
| DA-07 | `t8_fixedDaysPlusFixedMonthsAtBothLevelsNoDoubleIncrement` | Both reward + customer rows have midnight ISSUE_DATE | `assertMidnight` on both `rewardRows.get(0)` and `customerRows.get(0)` | P0 |

---

### New Integration Tests (add to `RewardConstraintConcurrencyIntegrationTest`)

| ID | FR | Description | Setup | Key Assertions | Priority |
|----|----|-------------|-------|----------------|----------|
| T9 | FR-008 | **Non-midnight ISSUE_DATE compatibility** — seed a row with full-timestamp ISSUE_DATE directly via JDBC (bypassing the service layer's midnight-stripping), then issue via API; assert no second row is created | Direct `jdbcTemplate.update(INSERT_SQL)` with `ISSUE_DATE = new Date()` (full timestamp, not midnight); then `issueBulkRewards()` | `rewardLevelRows(rewardId).size() == 1` AND `consumed.compareTo(BigDecimal.valueOf(2)) == 0`. **Test WILL FAIL** until FR-008 is resolved — that is intentional; it is a blocking gate test. | P0 |
| T10 | FR-001, FR-007 | **Limit breach boundary after concurrent success** — T3 proves 2 concurrent succeed with consumed=2 (limit=5); this test verifies the boundary: 5 total succeed, 6th fails with consumed unchanged | Reuse `buildRewardLevelConstraintRequest(5)`, issue MOBILE_1 through MOBILE_5 serially (no concurrency needed for boundary test), then issue MOBILE_6 | First 5: `isSuccess`; consumed after 5: `== 5`; MOBILE_6: `isConstraintFailure`; consumed after rejection: still `== 5` (not 6) | P1 |
| T11 | FR-009 | **Partial vendor compensation (T-20)** — vendor returns 1 success + 1 failure for qty=2; CONSUMED should be 1 after compensation | Configure vendor stub for partial success (1 of 2 qty issued); issue with `quantity=2` | `rewardLevelRows(rewardId).size() == 1`; `consumed.compareTo(BigDecimal.ONE) == 0` (pre-write was 2, compensation decremented by 1 for failed qty) | P0 |
| T12 | FR-004 | **Total failure compensation: CONSUMED returns to 0** — coupon issue fails for all; compensation fires in finally; consumed unwound to 0 | Configure `IntouchServiceStub` coupon failure for the reward; issue normally | `rows.size() == 1` (zero-consumed row by design); `consumed.compareTo(BigDecimal.ZERO) == 0` (NOT negative) | P1 |

---

### Unit Tests (add to `RewardConstraintFacadeTest` or `RewardIssueWrapperTest`)

| ID | FR | Description | Class Under Test | Mock Strategy | Priority |
|----|----|-------------|-----------------|---------------|----------|
| UT-01 | FR-009 | `getFailedQuantity()` = `quantityToBeProcessed - successCount` for partial success (qty=2, success=1) | `BulkRewardIssueContext.RewardIssueWrapper` | None — pure logic | P0 |
| UT-02 | FR-009 | `getFailedQuantity()` = `quantityToBeProcessed` when `successCount=0` (total failure) | `BulkRewardIssueContext.RewardIssueWrapper` | None | P0 |
| UT-03 | FR-009 | `getFailedQuantity()` = 0 when `successCount = quantityToBeProcessed` (all success) | `BulkRewardIssueContext.RewardIssueWrapper` | None | P0 |
| UT-04 | FR-009 | Compensation filter: wrapper with `getFailedQuantity()>0` is included; `bulkAtomicDecrement` called | `RewardConstraintFacade.compensateFailedSummaryWrites` | Mock `ctx` (partial failure), mock JDBC repo; verify `bulkAtomicDecrement` called once | P0 |
| UT-05 | FR-009 | Compensation filter: wrapper with `getFailedQuantity()==0` is excluded; `bulkAtomicDecrement` NOT called | `RewardConstraintFacade.compensateFailedSummaryWrites` | Mock `ctx` (all-success wrapper) | P0 |
| UT-06 | FR-009 | Compensation delta for QUANTITY KPI = `failedQty` (not original full delta) | `RewardConstraintFacade.compensateFailedSummaryWrites` | Mock rows with QUANTITY KPI; verify `bulkAtomicDecrement` receives `consumed=failedQty` | P0 |
| UT-07 | FR-009 | Compensation delta for REDEMPTION_VALUE KPI = `redemptionValue × failedQty` | `RewardConstraintFacade.compensateFailedSummaryWrites` | Mock rows with REDEMPTION_VALUE KPI | P0 |
| UT-08 | FR-009 | TRANSACTION_COUNT: compensate ONE only on total failure (`failedQty == totalQty`) | `RewardConstraintFacade.compensateFailedSummaryWrites` | Mock partial+total failure wrapper with TRANSACTION_COUNT row | P0 |
| UT-09 | FR-009 | TRANSACTION_COUNT: NO compensation on partial success (`failedQty < totalQty`) | `RewardConstraintFacade.compensateFailedSummaryWrites` | Verify `bulkAtomicDecrement` not called for TRANSACTION_COUNT row when partial success | P0 |
| UT-10 | FR-002 | `writeNonOrgSummariesAtomically()` — `toInsert` dedup: 2 constraints at same (level,kpi), empty existing rows → exactly 1 `save()` call | `RewardConstraintFacade` | Mock `findExistingForNonOrgLevel` returns empty; verify `rewardIssueSummaryJdbcRepository.save()` called once | P0 |
| UT-11 | FR-003 | `writeNonOrgSummariesAtomically()` — `toAtomicIncrement` dedup: 2 constraints resolve to same existing row → exactly 1 entry in `bulkAtomicIncrement` | `RewardConstraintFacade` | Mock `findExistingForNonOrgLevel` returns 1 existing row | P0 |
| UT-12 | FR-007 | `NonOrgSummaryWriteProcessor` — re-evaluation failure: lock released, `writeNonOrgSummariesAtomically` NOT called | `NonOrgSummaryWriteProcessor` | Mock `reEvaluateConstraints` returns false; verify `writeNonOrgSummariesAtomically` not invoked; verify `releaseLock` invoked | P0 |
| UT-13 | FR-001 | `NonOrgSummaryWriteProcessor` — lock timeout: wrapper added to `toFail` with `CONSTRAINT_EVALUATION_FAILED` | `NonOrgSummaryWriteProcessor` | Mock `acquireRewardLock` throws `MarvelException` | P0 |
| UT-14 | FR-005 | `writeNonOrgSummariesAtomically()` — new row's `issueDate` is midnight-stripped | `RewardConstraintFacade` | Mock `dateTimeService.getCurrentDate()` to return non-midnight; verify `save()` receives midnight date | P0 |

---

### Tenant Isolation Tests

| ID | Scenario | Org-A action | Org-B assertion | Expected | Type |
|----|----------|-------------|-----------------|----------|------|
| TI-01 | Lock key scoping | Org A issues reward X (lock: `reward_constraint:orgA:X`) | Org B issues same rewardId (different org) — different lock key | Requests proceed independently, no cross-org lock contention | IT (use `MOBILE_1` on two distinct orgId setups) |
| TI-02 | CONSUMED row isolation | Org A issues → row with `ORG_ID=orgA` written | Directly query `TBL_REWARD_ISSUE_SUMMARY` for `ORG_ID=orgB, REWARD_ID=X` | Row count = 0 for org B | IT (add to T1 or T9 with a secondary org assertion) |

---

## Regression Coverage

| ID | Existing Behaviour | Assertion | Risk if Broken | Type |
|----|-------------------|-----------|---------------|------|
| RG-01 | T1–T8 all pass with new DA assertions | All DA assertions green | Race fix regressed | IT |
| RG-02 | CUSTOMER-level serial enforcement | T5: consumed=2, 3rd rejected | CUSTOMER limit bypass | IT |
| RG-03 | CQ9 filter: updateSummaries skips atomically-written rows | T4: consumed=5 not drifted | Stale pos-12 double-write | IT |
| RG-04 | Pos-13 re-evaluation rejects inside lock | T6: consumed stays 1 | Limit breach on serialized concurrent calls | IT |
| RG-05 | T-18 dual-FIXED dedup | T7+T8: 1 row, consumed=N | Double row or double increment | IT |

---

## Test Coverage Summary

| Layer | Count | % |
|-------|-------|---|
| Unit (new) | 14 (UT-01 to UT-14) | ~54% |
| Integration (inline DA additions) | 7 (DA-01 to DA-07) | ~27% |
| Integration (new tests) | 4 (T9 to T12) | ~15% |
| Tenant isolation | 2 (TI-01, TI-02) | ~8% |
| **Total new** | **27** | 100% |

---

## Test Data Requirements

- **T9 — Direct JDBC insert**: Use `@Autowired JdbcTemplate` or the `BaseIntegrationTest` datasource to insert a row with full-timestamp `ISSUE_DATE`. Do NOT use `rewardIssueSummaryJdbcRepository.save()` — it would go through the service layer. Use raw SQL matching the `INSERT_SQL` in `RewardIssueSummaryJdbcRepository`.
- **T11 — Partial vendor stub**: `IntouchServiceStub.issueBulkFunction1` returns success/failure counts at the reward level (not qty level). Verify whether partial qty is achievable via the existing stub interface. If not, stub `VendorIssueProcessor` at field level via `ReflectionTestUtils`.
- **Midnight assertion helper**: add `assertMidnight(Date)` and `assertNoDuplicateIssueDates(List<RewardIssueSummary>)` to `RewardConstraintConcurrencyIntegrationTest` (see Appendix).

---

## Minimum Viable Test Set (fast-release gate)

**Must pass before merge:**
- DA-01, DA-02, DA-03, DA-04, DA-06, DA-07 (midnight + no-duplicate-date assertions on existing tests)
- T9 (non-midnight compatibility — blocking gate; must either pass or FR-008 decision documented)
- T11 (partial compensation — blocking gate for T-20)
- UT-01, UT-02, UT-03, UT-04, UT-05, UT-06, UT-07, UT-08, UT-09 (getFailedQuantity + compensation delta)
- UT-12, UT-13, UT-14 (processor behavior unit tests)

**Ship after release:**
- T10, T12 (boundary and zero-consumed design tests)
- TI-01, TI-02 (tenant isolation confirmations)
- UT-10, UT-11 (dedup unit tests for writeNonOrgSummariesAtomically)

---

## Recommended Execution Order

1. Unit tests (UT-01 to UT-14) — gate on zero failures
2. Inline DA assertions added to T1–T8 — run existing IT suite; should be green
3. T9 (non-midnight compatibility) — run independently; designed to FAIL until FR-008 resolved
4. T10, T12 — run as part of full IT suite
5. T11 (partial compensation) — run after T-20 implementation complete

---

## Definition of Done

- [ ] All existing T1–T8 pass with DA-01 through DA-07 assertions added
- [ ] T9 passes (FR-008 resolved: SQL changed to date-only comparison OR migration applied)
- [ ] T11 passes (T-20 implementation complete: partial compensation CONSUMED=1 confirmed)
- [ ] UT-01 through UT-14 all pass
- [ ] No regression in any existing IT
- [ ] `assertMidnight()` and `assertNoDuplicateIssueDates()` helpers added to test class

---

## Appendix: Proposed Helper Additions to `RewardConstraintConcurrencyIntegrationTest`

```java
/** Asserts stored ISSUE_DATE has no time component (midnight in UTC). */
private void assertMidnight(Date date) {
    Calendar cal = Calendar.getInstance(TimeZone.getTimeZone("UTC"));
    cal.setTime(date);
    assertEquals(0, cal.get(Calendar.HOUR_OF_DAY), "ISSUE_DATE hour must be 0 — non-midnight date indicates time-stripping bug");
    assertEquals(0, cal.get(Calendar.MINUTE),       "ISSUE_DATE minute must be 0");
    assertEquals(0, cal.get(Calendar.SECOND),        "ISSUE_DATE second must be 0");
}

/**
 * Asserts all rows for a reward+level have no two rows sharing the same calendar date
 * but with different stored timestamps. Guards against the non-midnight duplicate-row bug:
 * one row at 11:25:57 + one row at 00:00:00 on the same date = same logical row, two DB rows.
 */
private void assertNoDuplicateIssueDates(List<RewardIssueSummary> rows) {
    Map<String, Long> dateCount = rows.stream().collect(
        Collectors.groupingBy(
            r -> {
                Calendar c = Calendar.getInstance(TimeZone.getTimeZone("UTC"));
                c.setTime(r.getIssueDate());
                return c.get(Calendar.YEAR) + "-" + c.get(Calendar.DAY_OF_YEAR);
            },
            Collectors.counting()
        )
    );
    dateCount.forEach((day, count) ->
        assertEquals(1L, count,
            "More than 1 row for calendar date " + day +
            " — non-midnight duplicate row: one row has time component, another has midnight. " +
            "Fix: DATE(ISSUE_DATE) = DATE(:date) in findExistingForNonOrgLevel SQL"));
}
```

---

## Findings for arch-investigator-inbox

**Non-midnight ISSUE_DATE production observation** (developer screenshot — 2026-06-24):

- `TBL_REWARD_ISSUE_SUMMARY` shows rows with `ISSUE_DATE = 2026-06-24 11:25:57` (non-midnight) alongside rows with `ISSUE_DATE = 2026-06-24 00:00:00` (midnight) for the same `REWARD_ID`.
- Root cause: `findExistingForNonOrgLevel()` uses `ISSUE_DATE = :date` (exact equality). The query parameter is always midnight-stripped. A pre-existing non-midnight row is invisible → new midnight row inserted → duplicate.
- **What created the non-midnight rows?** Likely the old `updateSummaries()` → `buildUniqueSummaries()` path, which ran before the `NonOrgSummaryWriteProcessor` was in the chain. Check `buildUniqueSummaries()` for date-stripping on the stored `issueDate`.
- **Resolution options:** (a) `DATE(ISSUE_DATE) = DATE(:date)` in SQL (check index impact), (b) one-time migration to strip time from existing rows, (c) accept as pre-existing data gap with monitoring.
- This is a **pre-deploy blocking issue** for any org with existing non-midnight rows in production.
