# CAP-184615 — Stats Test Coverage Handoff
**Prepared by:** arch-investigator  
**Date:** 2026-04-21  
**For:** tech-detailer  
**Scope:** IT, UT, and automation test gaps for the MySQL stats persistence layer (history + summary + Redis)

---

## 1. Background & Concern

Luci now persists `CouponSeriesStatisticsField` counters in MySQL alongside Redis:

| Layer | Table(s) | Written by |
|-------|----------|-----------|
| Blueprint (ETL snapshot) | `stats_history_blueprint_YYYYMMDD` | `StatsHistoryRunner` (daily RMQ trigger from DataBricks S3) |
| Daily summary | `stats_series_summary` | In-process on every issue/redeem/revoke/reactivate |
| Today in-flight | Redis counters | In-process on every issue/redeem/revoke/reactivate |
| Registry | `stats_history_registry` | `StatsHistoryRunner` — records last snapshot date per org |

**Effective count formula used in limit checks and GET API:**
```
effectiveIssued  = Redis(IC today)
                 + blueprint(IC up to snapshot date)
                 + summary(IC from snapshot+1 → yesterday)
                 - historicalRIC (summary snapshot+1 → today)

effectiveRedeemed = blueprint(RC up to snapshot date)
                  + summary(RC from snapshot+1 → yesterday)
                  + Redis(RC today)
                  - historicalRRC
```

**Core concern (from requirement.md):**
> Most test cases create a new series, which won't cover stats from history. Need IT, UT, and automation coverage for history + summary + Redis aggregate — the normal steady state for any long-running series.

---

## 2. Key Source Files

### Limit-check processors
| File | Responsibility |
|------|----------------|
| `SeriesLockAcquireProcessor.java` | Issuance (IC) limit check — assembles `effectiveIssued` from Redis + `historicalIssuedValue` (blueprint+summary) − `historicalRICValue` |
| `MaxRedemptionForSeriesProcessor.java` | Redemption (RC) limit check — calls `statsHistoryService.getHistoricalValue(RC)` for blueprint+summary |
| `MaxRedemptionForEntityProcessor.java` | OU-level RC limit — same MySQL path but filtered by `redemption_config_entity_id` |

### Stats assembly service
| File | Responsibility |
|------|----------------|
| `MySQLCouponSeriesStatisticsReadService.java` | GET API path — `getStatisticsBatch()` calls `bulkFetchHistoryValues()` → blueprint+summary |
| `StatsHistoryServiceImpl.java` | Core aggregate logic: `getHistoricalValue()`, `getMysqlSummaryValue()`, `bulkGetHistoricalValues()` |
| `StatsHistoryDaoImpl.java` | SQL execution for blueprint+summary queries |
| `StatsSeriesSummaryServiceImpl.java` | Daily increment writes for IC/RC/RIC/RRC |

### Existing test classes (reference)
| File | What it covers |
|------|----------------|
| `StatsSeriesSummaryIntegrationTest.java` | IT — IC limit with history; RIC/RRC on same-day path; `insertHistoricalStatsData()` helper already present |
| `StatsHistoryServiceImplTest.java` | UT — mocked DAO calls for `getHistoricalValue`, edge cases like start > end date |
| `StatsCounterDiffCheckerJobTest.java` | UT — diff checker logic |
| `campaigns_auto/tests/luci/` | Automation — black-box API tests; **no stats field assertions at present** |

---

## 3. Root Cause of Coverage Gap

```
ROOT CAUSE [CONFIDENCE: HIGH]

Every existing IT creates a fresh coupon series at test start.
stats_history_blueprint_YYYYMMDD and stats_series_summary are therefore
always empty for that series — so every test exercises only the
"today-only Redis path" or the "same-day summary path", never the
history+summary aggregate that is the normal steady state for any series
older than one ETL cycle.

Contributing factors:
1. insertHistoricalStatsData() helper exists in StatsSeriesSummaryIntegrationTest
   but is called by only ONE test (testHistoricalStatsWithCouponIssueAndRedeem).
2. That one test covers IC limit but NOT RC limit, RIC-in-blueprint, or
   the GET API stats assembly path.
3. Automation tests have no mechanism to control DB state across days or to
   seed blueprint data via the API.
```

---

## 4. Identified Gaps

### 4.1 IT Gaps

#### GAP-IT-1 — RC limit check with history data [SEVERITY: HIGH]
**What's missing:** No IT covers `MaxRedemptionForSeriesProcessor` with non-zero blueprint + summary RC values.  
**Risk:** The MySQL RC limit path (`statsHistoryService.getHistoricalValue(RC)`) has never been exercised end-to-end in a test. A silent regression here would allow redemptions beyond the series limit for long-running series.  
**Existing anchor:** `testHistoricalStatsWithCouponIssueAndRedeem` in `StatsSeriesSummaryIntegrationTest.java` covers IC limit with history — RC limit needs a symmetric test.

#### GAP-IT-2 — GET API stats assembly from history + summary [SEVERITY: HIGH]
**What's missing:** No IT verifies that `MySQLCouponSeriesStatisticsReadService.getStatisticsBatch()` returns values that include blueprint + summary counts, not just today's Redis values.  
**Risk:** Stats displayed to merchants in GET responses could silently omit historical counts, showing stale/low values without any test catching it.  
**Note:** `testHistoricalStatsWithCouponIssueAndRedeem` asserts `num_issued=24` at line 678 of `StatsSeriesSummaryIntegrationTest`, but only as a side effect of the issuance response — the dedicated GET stats path (`bulkFetchHistoryValues`) is not explicitly exercised with seeded blueprint data.

#### GAP-IT-3 — RIC in blueprint path reduces IC limit [SEVERITY: MEDIUM]
**What's missing:** No IT seeds RIC values inside a blueprint table and verifies that `SeriesLockAcquireProcessor` correctly subtracts them from `effectiveIssued`.  
**Existing coverage:** `shouldReflectRicAndRrcWhenRevokeOrReactivateOnDifferentDay` tests RIC only in the same-day summary path. Blueprint-era RIC is untested.  
**Risk:** If RIC is not fetched correctly from the blueprint layer, effective issued count would be overstated → series locks earlier than it should.

#### GAP-IT-4 — Absent registry entry fallback [SEVERITY: MEDIUM]
**What's missing:** No IT covers the scenario where `stats_history_registry` has no entry for an org (e.g., DataBricks job never ran) but `stats_series_summary` has data.  
**Fallback path:** `findLastActiveByOrgId` returns null → summary start date falls back to `MAX_LOOK_BACK_DAYS` (-100 days). No test validates what limit check returns in this state.  
**Risk:** Off-by-one in the fallback window or a null-pointer could silently pass all limit checks.

#### GAP-IT-5 — OU-level RC limit with history data [SEVERITY: MEDIUM]
**What's missing:** No IT covers `MaxRedemptionForEntityProcessor` when blueprint + summary have OU-level (`redemption_config_entity_id`) RC values.  
**Risk:** OU-level redemption caps bypass the same MySQL path but with a different filter; untested means OU-cap logic is invisible to regression.

#### GAP-IT-6 — `StatsHistoryRunner` end-to-end pipeline [SEVERITY: LOW]
**What's missing:** `StatsHistoryRunner` is only unit-tested with mocks. No IT verifies: parquet download from S3 → rolling table insert → registry update → correct aggregate counts.  
**Risk:** A schema change in the parquet or a SQL bug in the rolling table insert would not be caught until production.

### 4.2 Implementation Edge Cases Not Covered by Tests

#### EDGE-1 — Date-range asymmetry in `SeriesLockAcquireProcessor` [SEVERITY: MEDIUM]
**Location:** `SeriesLockAcquireProcessor.java` (lines ~62–69)  
**Issue:** `historicalIssuedValue` reads IC through **yesterday** (`getHistoricalValue`), but `historicalRICValue` reads through **today** (`getMysqlSummaryValue`). In the normal case (blueprint snapshot from a past day) this is correct — today's revokes offset today's in-flight Redis IC. But if the DataBricks job runs and writes a snapshot for today (unlikely but possible in early-day runs), today's IC could be double-counted.  
**No test probes this boundary.**

#### EDGE-2 — `getHistoricalValue` when start date = today [SEVERITY: LOW]
**Location:** `StatsHistoryServiceImplTest.java:231` — `testGetHistoricalValue_WhenStartDateAfterEndDate_SkipsSummingFromStatsSeriesSummary`  
**Issue:** The UT verifies the skip when snapshot = yesterday. The scenario where snapshot = today (DataBricks runs early, then a limit check fires before midnight) is not validated by an IT.

### 4.3 Automation Test Gaps (`campaigns_auto/tests/luci/`)

**Finding:** Zero assertions on stats field values (`num_issued`, `num_redeemed`, `numTotal`, or any counter field) exist in any automation test file. The only stats reference is a comment: `# Wait for calculate stats to complete`.  
**Root cause:** All automation tests create a fresh series. Blueprint and summary tables are empty for that series. Asserting `num_issued > 0` from history would require pre-seeded DB state across days.

---

## 5. Proposed Solutions

### 5.1 IT Solutions (in priority order)

#### SOL-IT-1 — Add RC limit IT with history data (addresses GAP-IT-1)
**Target class:** `StatsSeriesSummaryIntegrationTest.java`  
**Approach:**
1. Use existing `insertHistoricalStatsData(orgId, seriesId, snapshotDate, ic, rc, ...)` helper to seed blueprint with `rc = 14`.
2. Seed `stats_series_summary` with RC entries from `snapshotDate+1` to `yesterday` summing to 0 (so blueprint alone provides the historical RC).
3. Create series with `maxRedeem = 15`.
4. Issue 1 coupon, redeem it → should succeed (14 historical + 1 today = 15, at limit).
5. Try to redeem a second → should fail with the "max redeem for series exceeded" error code.
6. Assert the error code matches `ErrorCode.MAX_REDEMPTIONS_FOR_SERIES_EXCEEDED`.

**Reusable helpers already present:** `createRollingTable()`, `insertHistoricalStatsData()`, `insertStatsSeriesSummary()`, `createNewCouponSeries()`, `issueCoupon()`, `redeemCoupon()`.

#### SOL-IT-2 — Add GET API stats IT with history (addresses GAP-IT-2)
**Target class:** `StatsSeriesSummaryIntegrationTest.java` (or a new `StatsGetApiIntegrationTest.java` if that class is growing too large)  
**Approach:**
1. Seed blueprint with `ic = 20, rc = 8` for a series.
2. Seed summary with `ic = 3, rc = 1` from snapshot+1 to yesterday.
3. Call `GET /coupon/getAllCouponConfigurations?couponSeriesStatisticsRequired=true` for that series.
4. Assert: `num_issued == 23` (20+3), `num_redeemed == 9` (8+1) in the response body.
5. This directly exercises `MySQLCouponSeriesStatisticsReadService.getStatisticsBatch()` → `bulkFetchHistoryValues()`.

#### SOL-IT-3 — Add RIC-in-blueprint IT (addresses GAP-IT-3)
**Approach:**
1. Seed blueprint with `ic = 20, ric = 5` (net issued = 15 at snapshot time).
2. Create series with `maxCreate = 16`.
3. Issue 1 → should succeed (15 effective historical + 1 today = 16, at limit).
4. Issue 1 more → should fail with `SERIES_MAX_CREATE_LIMIT_EXCEEDED`.

#### SOL-IT-4 — Add absent registry fallback IT (addresses GAP-IT-4)
**Approach:**
1. Do NOT seed `stats_history_registry` for the test org.
2. Seed `stats_series_summary` with `ic = 4` (entries within last 100 days).
3. Create series with `maxCreate = 5`.
4. Issue 1 → should succeed.
5. Issue 1 more → should fail.
6. Also assert no NPE / exception in logs.

#### SOL-IT-5 — Add OU-level RC limit with history IT (addresses GAP-IT-5)
**Approach:**
1. Create a series with OU-level redemption cap `maxRedemptionPerEntity = 3` for entity `entityId`.
2. Seed blueprint with OU-level RC `rc_entity_entityId = 2`.
3. Redeem 1 for that entity → should succeed (2+1=3, at limit).
4. Redeem again for same entity → should fail.

### 5.2 Automation Test Solutions

Three options ranked by implementation effort and reliability:

#### OPT-A — Persistent fixture series (Quick win, recommended first)
**Concept:** Pre-seed one dedicated coupon series per test cluster during environment setup (not during test execution). The automation test calls GET and asserts known stats values.  
**Steps:**
1. DBA / infra team seeds `stats_history_blueprint_YYYYMMDD` + `stats_history_registry` for a fixed `seriesId` in each cluster during env provisioning.
2. Add a new automation test `test_stats_historical_assertions.py` that:
   - Calls `GET /coupon/getAllCouponConfigurations?couponSeriesId=<FIXTURE_SERIES_ID>&couponSeriesStatisticsRequired=true`
   - Asserts `num_issued >= <seeded_ic>` and `num_redeemed >= <seeded_rc>`.
3. The fixture series never issues/redeems in automation (read-only assertions), so values stay stable.  
**Pro:** No Luci code change. Immediate coverage.  
**Con:** Fragile if cluster DB is reset. Requires coordination with infra.

#### OPT-B — Test-only seeding endpoint in Luci (Recommended long-term)
**Concept:** Add a Luci REST endpoint (Spring profile `test` only) that accepts a JSON payload and inserts rows into `stats_history_blueprint_YYYYMMDD` and `stats_history_registry`.  
**Endpoint sketch:**
```
POST /internal/test/seedStatsHistory
{
  "orgId": 123,
  "seriesId": 456,
  "snapshotDate": "2026-04-20",
  "ic": 20, "rc": 8, "ric": 2, "rrc": 1
}
```
**Steps:**
1. Implement `TestStatsHistorySeedController` (active only on `spring.profiles.active=test`).
2. Automation test calls this endpoint at the start of each stats IT, then issues/redeems, then asserts limit behaviour and GET response.
3. Tear down by calling a `DELETE /internal/test/seedStatsHistory?orgId=...&seriesId=...` at test end.  
**Pro:** Self-contained, repeatable, no infra coordination.  
**Con:** Requires Luci code change and deployment to test clusters.

#### OPT-C — S3 parquet + `StatsHistoryRunner` trigger (Pipeline regression)
**Concept:** Upload a crafted parquet file to S3 in the test environment, then call `POST /coupon/statsHistory` to trigger `StatsHistoryRunner`. Use the result to exercise limit checks.  
**Pro:** Tests the full DataBricks pipeline end-to-end.  
**Con:** Requires S3 access from test setup, parquet generation, and `StatsHistoryRunner` enabled for the test org. Slowest option (~minutes per test).  
**Recommendation:** Use OPT-C only for a dedicated pipeline regression test (not routine automation).

---

## 6. Bounded Scope for Tech Detailing

The tech-detailer should produce specifications for the following, in priority order:

| # | Deliverable | Type | Priority |
|---|------------|------|----------|
| 1 | `testRCLimitWithHistoricalData` in `StatsSeriesSummaryIntegrationTest` | New IT | P0 |
| 2 | `testGetApiStatsAssemblyFromHistoryAndSummary` in `StatsSeriesSummaryIntegrationTest` (or new class) | New IT | P0 |
| 3 | `testRICInBlueprintReducesEffectiveIssuedCount` | New IT | P1 |
| 4 | `testAbsentRegistryFallbackToMaxLookback` | New IT | P1 |
| 5 | `testOULevelRCLimitWithHistoryData` | New IT | P1 |
| 6 | `TestStatsHistorySeedController` + automation test `test_stats_historical_assertions.py` | New controller + automation | P1 |
| 7 | Edge-case UT for date-range boundary in `SeriesLockAcquireProcessor` | New UT | P2 |
| 8 | `StatsHistoryRunner` IT (end-to-end) | New IT | P2 |

---

## 7. Questions for Tech Detailer to Resolve

1. **`insertHistoricalStatsData` helper** — Does it currently support seeding RIC/RRC columns in the blueprint table, or only IC/RC? If not, a helper extension is needed.
2. **Blueprint table naming** — `stats_history_blueprint_YYYYMMDD` — what date should ITs use as the snapshot date? Typically `LocalDate.now().minusDays(1)`. Confirm with the runner logic.
3. **Test profile for OPT-B** — Is there an existing `test` Spring profile in Luci, or would this be a new profile? Check `application-test.yml`.
4. **OU-level blueprint schema** — Does `stats_history_blueprint_YYYYMMDD` have an `entity_id` column for OU-level RC, or is OU-level RC stored only in `stats_series_summary`? Confirm before writing GAP-IT-5 test.
5. **`MAX_LOOK_BACK_DAYS`** — What is the actual constant value and where is it defined? Confirm it is `-100` days.

---

## 8. Files to Read Before Detailing

```
# Limit-check processors
src/main/java/com/capillary/luci/processor/series/SeriesLockAcquireProcessor.java
src/main/java/com/capillary/luci/processor/series/MaxRedemptionForSeriesProcessor.java
src/main/java/com/capillary/luci/processor/series/MaxRedemptionForEntityProcessor.java

# Stats assembly
src/main/java/com/capillary/luci/service/stats/StatsHistoryServiceImpl.java
src/main/java/com/capillary/luci/service/stats/MySQLCouponSeriesStatisticsReadService.java
src/main/java/com/capillary/luci/dao/stats/StatsHistoryDaoImpl.java
src/main/java/com/capillary/luci/service/stats/StatsSeriesSummaryServiceImpl.java

# Existing tests (reference and extension points)
src/test/java/com/capillary/luci/integration/StatsSeriesSummaryIntegrationTest.java
src/test/java/com/capillary/luci/unit/StatsHistoryServiceImplTest.java

# Automation
/Users/saravanankesavan/sara/wsgit/campaigns_auto/tests/luci/
```

---

*End of handoff document. Ready for tech-detailer intake.*
