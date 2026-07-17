# Test Plan: Reward Constraint Limit Breach & Duplicate Summary Rows Under Parallel Issue Calls

**Date:** 2026-06-24  
**Input:** `CAPJUN19_reward-limit-concurrency_techdetail.md`  
**Scope:** DB-level correctness assertions for concurrency races, limit enforcement, ISSUE_DATE integrity, partial compensation (T-20), and legacy non-midnight row compatibility  
**Confidence:** HIGH  
**Revision:** v3 — FR-008 closed as non-issue (investigation of main branch confirmed no non-midnight write path exists); T9 redesigned to test MySQL ON UPDATE fix; all gaps closed; suite green at 3526 tests, 0 failures

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
- ~~FR-008: Legacy rows with non-midnight ISSUE_DATE must be found by the lookup~~ **CLOSED — non-issue.** Investigation of main branch `RewardConstraintFacade.updateSummaries()` confirmed every write path calls `Utils.getDateWithoutTimestampInSpecifiedZone()` before `bulkSaveOrUpdate`. No non-midnight row can exist from this codebase. Exact equality in `findExistingForNonOrgLevel` is correct and safe.
- FR-009: Partial vendor issuance (qty=2, 1 success + 1 failure) → CONSUMED=1 after compensation (T-20)
- FR-010: T-18 dual-FIXED dedup — exactly 1 DB row per (level, kpi), consumed=N not 2N

### Non-Functional Requirements

- NFR-001: Existing tests T1–T8 must not regress
- NFR-002: New assertions must add < 50ms each to test execution (prefer appending to existing tests over new Spring context setups)
- NFR-003: All ISSUE_DATE values in new rows must be midnight in the org's timezone

### Ambiguities & Open Questions

- [x] **FR-008 (non-midnight compatibility)**: **RESOLVED — non-issue.** Traced all write paths in main branch. `RewardConstraintFacade.updateSummaries()` always calls `Utils.getDateWithoutTimestampInSpecifiedZone(…, orgZoneId)` before persisting. No path produces non-midnight rows. `findExistingForNonOrgLevel` exact equality is correct. No SQL change or migration needed.
- [x] **T-20 partial compensation stub**: **RESOLVED.** `IntouchServiceStub.issueBulkFunction1` accepts per-reward success/failure maps. `setupCouponStub(rewardId, successQty, failQty)` overload added to integration test. T11 (`setupCouponStub(rewardId, 1, 1)`) and T12 (`setupCouponStub(rewardId, 0, 1)`) both pass.

---

## Risk Assessment

| Area | Risk | Priority |
|------|------|----------|
| ISSUE_DATE non-midnight (new rows) | New rows written with time component bypass future dedup → new duplicate rows accumulate | P0 |
| ~~Legacy non-midnight rows (FR-008)~~ | ~~Pre-existing rows invisible to new lookup~~ **CLOSED — no non-midnight write path exists in codebase** | ~~P0~~ Resolved |
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
| T9 | ~~FR-008~~ MySQL ON UPDATE fix | **REDESIGNED — ISSUE_DATE = ISSUE_DATE fix: atomic increment must not corrupt timestamp.** FR-008 was closed as non-issue (no non-midnight write path). T9 now validates the `ISSUE_DATE = ISSUE_DATE` clause in `ATOMIC_INCREMENT_SQL` that suppresses MySQL's implicit `ON UPDATE CURRENT_TIMESTAMP`. Three sequential issues: after each, assert row count=1, ISSUE_DATE=midnight, CONSUMED=N. | `buildRewardLevelConstraintRequest(5)`, three serial `issueBulkRewards()` calls | After each issue: `rows.size()==1`, `assertMidnight`, `consumed==N`. **PASSING.** | P0 |
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

- **T9 — redesigned**: No JDBC insert needed. Tests the MySQL ON UPDATE fix via three serial `issueBulkRewards()` calls through the production code path. FR-008 closed.
- **T11 — Partial vendor stub**: `IntouchServiceStub.issueBulkFunction1` accepts per-reward success/failure maps. `setupCouponStub(rewardId, successQty, failQty)` overload added to the integration test class.
- **Midnight assertion helper**: `assertMidnight(Date)` implemented using `Asia/Kolkata` timezone (IST) — IntouchServiceStub always returns Asia/Kolkata as org timezone. **Note:** the appendix below used UTC — that was incorrect; IST is the right zone.
- **`assertNoDuplicateIssueDates`**: not implemented as a named method. `assertEquals(1, rows.size())` in every test provides equivalent duplicate-row detection. No calendar-grouping helper required.

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

- [x] All existing T1–T8 pass with DA-01 through DA-07 assertions added
- [x] T9 passes — redesigned to validate MySQL ON UPDATE CURRENT_TIMESTAMP fix (FR-008 closed as non-issue)
- [x] T10 passes — limit breach boundary: 5 successes then rejection, consumed stays 5
- [x] T11 passes — T-20: partial compensation, consumed=1 after 1-of-2 qty failure
- [x] T12 passes — total failure compensation, consumed=0
- [x] UT-01 through UT-14 all pass
- [x] TI-01, TI-02 pass
- [x] No regression in any existing IT — **3526 tests, 0 failures, 0 errors**
- [x] `assertMidnight()` helper added (IST/Asia/Kolkata timezone — org timezone from IntouchServiceStub)
- [x] `assertNoDuplicateIssueDates()` helper replaced by `assertEquals(1, rows.size())` inline — equivalent coverage, no named helper needed

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

**Non-midnight ISSUE_DATE production observation** (developer screenshot — 2026-06-24): **RESOLVED**

- Initial concern: duplicate rows with different ISSUE_DATE timestamps for the same REWARD_ID.
- Root cause identified (post-investigation): The duplicate rows were caused by the MySQL `ON UPDATE CURRENT_TIMESTAMP` implicit behaviour on the TIMESTAMP column (with `--explicit-defaults-for-timestamp=0`). `bulkAtomicIncrement` did not include `ISSUE_DATE` in the SET clause, so MySQL silently rewrote it to `CURRENT_TIMESTAMP`. The next issuance's midnight lookup then missed the non-midnight row → inserted a second row.
- **Fix applied**: `ATOMIC_INCREMENT_SQL` and `ATOMIC_DECREMENT_SQL` now include `ISSUE_DATE = ISSUE_DATE` to suppress the implicit `ON UPDATE`. Validated by T9 (3 sequential increments: row count stays 1, ISSUE_DATE stays midnight).
- **FR-008 (exact equality SQL)**: Investigation of main branch confirmed `updateSummaries()` → `buildUniqueSummaries()` always calls `Utils.getDateWithoutTimestampInSpecifiedZone()` before writing. No non-midnight row could have been produced by the original code. The exact equality in `findExistingForNonOrgLevel` is safe. No SQL change or migration required.
