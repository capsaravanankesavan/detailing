# Test Plan: Stats Test Coverage — MySQL History + Summary Path
**Date:** 2026-04-22
**Input:** `CAP-184615_stat_test_techdetail.md` (same directory)
**Scope:** New ITs and UTs to cover the MySQL stats history path (blueprint + summary aggregate)
for IC limit, RC limit, GET API stats assembly, DISC_CODE_PIN upload-count stats,
and multi-operation sequences with historical blueprint data.
**Confidence:** HIGH — all helper availability and formula derivations verified against source.

---

## Requirements Summary

### Functional Requirements

| ID | Requirement |
|----|-------------|
| FR-001 | `num_issued` in GET API = `(summary_IC − summary_RIC) + blueprint_IC` (MySQLCouponSeriesStatisticsReadService.java `getSumFromCache`) |
| FR-002 | `num_redeemed` in GET API = `(summary_RC − summary_RRC) + blueprint_RC` |
| FR-003 | RC limit check = `mysql_RC_today(post-increment) + historicalRC(blueprint+summary→yesterday) − historicalRRC(summary→today)` (MaxRedemptionForSeriesProcessor) |
| FR-004 | IC limit check = `Redis_IC_today(post-increment) + historicalIC − historicalRIC` where `historicalRIC = getMysqlSummaryValue(RIC)` scoped to summary→today (SeriesLockAcquireProcessor) |
| FR-005 | Historical RIC rows in `stats_series_summary` (past days) reduce `effectiveIssued` in the IC limit check; RIC is **never** stored in the blueprint table (`isMysql=true`) |
| FR-006 | When `stats_history_registry` has no entry for an org, `getHistoricalValue()` falls back to `MAX_LOOK_BACK_DAYS=−100` and reads summary only — no NPE |
| FR-007 | OU-level RC limit is enforced correctly with blueprint OU-RC + today's MySQL OU-RC (`MaxRedemptionForSeriesProcessor` OU branch: `ouId != 0`) |
| FR-008 | `getHistoricalValue()` skips summary aggregation when snapshot date = today (DataBricks early run causes `startDate > endDate`) |
| FR-009 | `getAllCouponConfigurations` / `getCouponConfigForSeries` returns `numTotal = summary_TC + blueprint_TC` for DISC_CODE_PIN series |
| FR-010 | Same API returns `num_uploaded_total = summary_UTC + blueprint_UTC` for DISC_CODE_PIN series |
| FR-011 | upload → issue → revoke → issue sequence: `numTotal` and `num_issued` reflect blueprint history correctly across all steps |
| FR-012 | issue → redeem → reactivate sequence: RC limit correctly reduces after reactivation (historical RRC offsets effective RC); `num_redeemed` = `(summary_RC − summary_RRC) + blueprint_RC` |
| FR-013 | upload → redeem → reactivate sequence: `numTotal` + `num_redeemed` via getAllCouponConfigurations correctly combine blueprint TC/RC with today's summary |

### Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| NFR-001 | Tenant isolation: `orgId=0` operations (new tests) must not bleed into `orgId=123` operations (existing test `testHistoricalStatsWithCouponIssueAndRedeem`) via shared registry rows, blueprint tables, or Redis cache |
| NFR-002 | `@Cacheable` on `getHistoricalValue()` must not cause stale reads across tests; all new ITs must evict `MIDNIGHT_EXPIRING_CACHE_NAME` after seeding |
| NFR-003 | No hardcoded series IDs (never `1L`); all new tests must pass `(long) savedSeries.getId()` to all seed helpers |

### Ambiguities & Open Questions — **All Resolved (2026-04-27)**

| # | Question | Resolution | Test Impact |
|---|---------|------------|-------------|
| Q1 ✅ | Default `client_handling_type` from `createCouponSeriesWithLimits`? | **`DISC_CODE`** — confirmed from `CouponConfigurationFixture#getCouponConfiguration`. `DISC_CODE` auto-generates coupons on issue; no `uploadCoupons()` needed for T1–T4. `numTotal` for DISC_CODE returns `couponSeriesEntity.getMaxCreate()`, NOT blueprint+summary TC — do not assert `numTotal` from history in IT-01 through IT-04. | T7/T8/T10 must still explicitly call `.setClient_handling_type("DISC_CODE_PIN")` + `uploadCoupons()` |
| Q2 ✅ | Does `getSumFromCache(IssuedCount)` clamp negative `currentValue` (IC < RIC)? | **Never clamps.** `currentValue = IC − RIC` can be negative; `getSumFromCache` returns `currentValue + blueprint_IC` even when negative. IT-03: after issue(1) today with past RIC=5 → `getSumFromCache = (1−5) + 20 = 16`. This is a **known production risk**: if `blueprint_IC < RIC`, `num_issued` in the GET API will be wrong. | IT-03 assertion: `num_issued=16`; add a comment flagging this as unclamped behavior |
| Q3 ✅ | How are OU-level series created in existing ITs? | Pattern is in `RedeemCouponIntegrationTest#shouldValidateRedemptionPerCoupounMaxLimitForOuWithDiffTillId`. Reference that test for series creation + OU redemption config setup. IT-05 spike is **unblocked**. | Developer copies OU series creation from that test |
| Q4 ✅ | Does `redeemCoupon()` accept `ouId`, or is a separate helper needed? | **OU is inferred from `tillId`** by `OrgEntityRelationsService#getOUIdForTill`. No separate `redeemCouponAtOU()` needed. IT-05 must use a `tillId` (storeUnitId) that maps to the target `ouId` in the test environment, and blueprint OU-RC must be seeded for that same `ouId`. | IT-05 setup: verify `storeUnitId` → `ouId` mapping before seeding blueprint OU-RC |
| Q5 ✅ | Is there `@Before`/`@After` cleanup for `stats_history_registry` and rolling tables in `BaseIntegrationTest`? | **Not confirmed** ("ideally should be there"). Cannot rely on framework cleanup. Each new IT must own its teardown explicitly — delete registry rows and DROP rolling tables in `@After` or at test end. | Add `@After` cleanup to all new IT methods (see Test Data Requirements below) |

---

## Risk Assessment

| Area | Risk | Priority |
|------|------|----------|
| Blueprint seeding for TC/UTC (DISC_CODE_PIN) | `insertHistoricalStatsData(orgId, seriesId, table, ic, rc)` does not seed TC/UTC. Must use `insertStatsHistoryData(orgId, table, List<StatsHistory>)` with manually built `StatsHistory` objects (key="tc", key="utc"). If `Fixture.getStatsHistory()` does not support TC/UTC keys, test will silently insert zero blueprint TC | P0 |
| `@Cacheable` stale reads | If Redis cache holds data from a prior test's series, `getHistoricalValue` returns a stale cached value. **Q5 confirmed: no guaranteed `@After` cleanup exists** — each new IT must evict Redis in its own `@After` | P0 |
| Series ID assumption | If blueprint seeds use wrong series ID, limit check sees 0 historical IC/RC — test looks like it passes but is actually testing the empty-history path | P0 |
| **[Q1 resolved]** DISC_CODE default — do not assert `numTotal` from history in IT-01–IT-04 | `createCouponSeriesWithLimits` produces a DISC_CODE series. `numTotal` for DISC_CODE = `couponSeriesEntity.getMaxCreate()`, not blueprint+summary TC. Any assertion of `numTotal=blueprint_TC+summary_TC` in IT-01 through IT-04 is wrong and will fail | P0 |
| **[Q2 confirmed]** `getSumFromCache` IC−RIC never clamped — production risk | When `summary_IC < summary_RIC` (more historical revokes than issues in summary window), `currentValue` goes negative. `num_issued` in GET API = `(negative) + blueprint_IC`. If `blueprint_IC` is also small, merchants can see a lower-than-actual `num_issued`. IT-03 assertion must reflect this: `num_issued = (1−5)+20 = 16`, not 21. Flag explicitly in test comment | P0 |
| DISC_CODE_PIN without uploadCoupons | If T7/T8/T10 create a DISC_CODE_PIN series but forget `uploadCoupons()`, `getTotalCount()` runs `getSumFromCache(TC)` with TC_summary=0 and blueprint TC from seed — assertion may pass accidentally without exercising the real upload-count path | P1 |
| RC check `shouldCommit=false` path | RC limit has two branches. T1/T9/T10 must use `redeemCoupon(…, true)` (shouldCommit=true) — the only branch that increments MySQL RC before the limit check | P0 |
| **[Q4 resolved]** OU inferred from `tillId` — must verify mapping | For IT-05, the `storeUnitId` passed to `redeemCoupon()` is looked up in `OrgEntityRelationsService#getOUIdForTill`. Blueprint OU-RC must be seeded for the `ouId` that maps to the chosen `storeUnitId`. If the mapping is absent in the test environment, OU resolution returns 0 and the OU branch is silently skipped | P1 |
| Revoke decrements TC/UTC in today's summary | Confirmed by existing test (numTotal=997 after 3 revokes from 1000 uploads). T8 and T10 final TC assertions must subtract the revoke count from today's TC | P1 |
| **[Q5 confirmed]** No guaranteed registry/rolling table cleanup | Each new IT must clean up `stats_history_registry` rows and DROP its rolling table in `@After`. Without this, a second test run will find stale registry entries pointing to non-existent tables | P0 |

---

## Coverage Gap Analysis (vs Existing Tests)

| Existing test | What it covers | What it misses |
|---------------|---------------|----------------|
| `testHistoricalStatsWithCouponIssueAndRedeem` | IC limit + RC limit with blueprint+summary history | RIC in summary, GET API stats assembly, TC/UTC, uses hardcoded `couponSeriesId=1L` |
| `shouldIssueRevokeRedeemReactivateCouponsAndValidateResponses` | RIC/RRC today-path, TC/UTC today-path for DISC_CODE_PIN | No blueprint history at all |
| `shouldReflectRicAndRrcWhenRevokeOrReactivateOnDifferentDay` | RIC/RRC cross-day (no blueprint), IC limit with today's RIC | No blueprint IC/RC, no blueprint TC/UTC |
| `shouldRedeemACouponSuccessfullyAndReduceTheRedemptionLeftCount` | Today-path only | No history |
| Automation (`campaigns_auto/tests/luci/`) | Issue/redeem flow | No `num_issued`/`num_redeemed`/`numTotal` assertions; no history seeding |

---

## Test Case Specifications

### Unit Tests — `src/test/java/`

| ID | Req | Description | Class Under Test | Mock Strategy | Priority |
|----|-----|-------------|-----------------|---------------|----------|
| UT-01 | FR-008 | `getHistoricalValue()` when snapshot date = today: `startDate > endDate` → `sumValuesForDateRange` is never called; blueprint value returned directly | `StatsHistoryServiceImpl` | Mock `statsHistoryRegistryDao.findLastActiveByOrgId` → returns registry with `snapShotDate=today`; mock `statsHistoryDao.findHistoricalValue` → returns 42L; assert `statsSeriesSummaryDao.sumValuesForDateRange` never invoked | P0 |
| UT-02 | FR-005 | `isMysql()` on `CouponSeriesStatisticsField`: `RevokedIssueCount.isMysql()=true`, `IssuedCount.isMysql()=false`, `ReactivatedRedemptionCount.isMysql()=true` | `CouponSeriesStatisticsField` (enum) | No mocks — pure enum state assertion | P0 |
| UT-03 | FR-001 | `getLongFields()` excludes RIC and RRC (isMysql=true fields); `getMysqlLongFields()` includes them | `CouponSeriesStatisticsField` | No mocks — assert list sizes and membership | P1 |
| UT-04 | FR-006 | `getHistoricalValue()` when no registry entry (returns null): falls back to `today − 100 days` as startDate, returns summary sum without NPE | `StatsHistoryServiceImpl` | Mock `findLastActiveByOrgId` → returns null; mock `statsSeriesSummaryDao.sumValuesForDateRange` → returns 7L; assert result = 7L, no NPE | P0 |
| UT-05 | FR-004 | `bulkGetHistoricalValues()` returns blueprint values ONLY (not summary values); confirms the three-path split of `bulkFetchHistoryValues()` | `StatsHistoryServiceImpl` | Mock `statsHistoryDao.bulkFindHistoricalValues` → returns map with key=IC, value=20L; assert result map contains only the mocked key (no summary fan-out) | P1 |

> **Testing instinct:** Unit tests pin the contract of each method in isolation — when an IT fails, you know the issue is in wiring, not in method logic.

---

### Integration Tests — `src/test/java/…/integration/StatsSeriesSummaryIntegrationTest`

#### Tests from Tech Detail (T1–T6)

| ID | Req | Description | Setup | Assertion | Priority |
|----|-----|-------------|-------|-----------|----------|
| IT-01 (T1) | FR-003 | RC limit with blueprint history: blueprint RC=14, maxRedeem=15; redeem 1 → success (15=limit), redeem 2 → fail | `createRollingTable(4)` + registry + `insertHistoricalStatsData(orgId, seriesId, table, 0L, 14L)` + `deleteRedisByKeyPattern(MIDNIGHT_EXPIRING_CACHE_NAME+"*")` | 1st redeem: no exception; `getCouponConfigForSeries` → `num_redeemed=15`; 2nd redeem: exception with errorCode=605 | P0 |
| IT-02 (T2) | FR-001, FR-002 | GET API assembly: blueprint IC=20, RC=8; past-day summary IC=3, RC=1 → num_issued=23, num_redeemed=9 | Blueprint seeded; `insertStatsSeriesSummary` for twoDaysAgo("ic"=1) + yesterday("ic"=2, "rc"=1); `deleteRedisByKeyPattern` | `getCouponConfigForSeries` → `num_issued=23`, `num_redeemed=9`; `fetchTodayStats` → IC=0 (no real ops) | P0 |
| IT-03 (T3) | FR-004, FR-005 | IC limit adjusted by historical RIC: blueprint IC=20, summary RIC=5; maxCreate=16; issue 1 → effectiveIssued=16 → success; issue 2 → fail | Blueprint IC=20; `insertStatsSeriesSummary` for threeDaysAgo("ric"=3) + yesterday("ric"=2); `deleteRedisByKeyPattern` | 1st issue: no exception; 2nd issue: errorCode=626. **GET API after 1st issue:** `num_issued = (cachedStat.IC−cachedStat.RIC) + blueprint_IC = (1−5) + 20 = 16` (**no clamping — confirmed Q2**); assert `num_issued=16` and add comment: *"IC−RIC goes negative (−4); result is blueprint_IC+negative = 16, not 21"* | P0 |
| IT-04 (T4) | FR-006 | Absent registry fallback: no rolling table, no registry; seed summary IC=4 for yesterday; maxCreate=5; issue 1 → success, issue 2 → fail | No `createRollingTable`, no `createStatsHistoryRegistryEntry`; `insertStatsSeriesSummary(orgId, seriesId, yesterday, "ic", 4L)` | 1st issue: no exception; 2nd issue: errorCode=626, errorCode≠500 (NPE guard) | P0 |
| IT-05 (T5) | FR-007 | OU-level RC limit with blueprint OU-RC=2, maxOuRedeem=3; redeem 1 at the OU-mapped till → success (3=limit), redeem 2 → fail | **Pattern from `RedeemCouponIntegrationTest#shouldValidateRedemptionPerCoupounMaxLimitForOuWithDiffTillId`** (Q3 resolved). Create OU-level series using that test's setup. Identify `storeUnitId` → `ouId` mapping via `OrgEntityRelationsService#getOUIdForTill` (Q4 resolved — no separate `redeemCouponAtOU` needed; OU inferred from `tillId`). Seed blueprint OU-RC=2 via `insertStatsHistoryData` with `StatsHistory.redemptionConfigEntity=OU, redemptionConfigEntityId=<ouId>`. | 1st redeem (historical OU-RC=2 + today=1 = 3 = limit): no exception; 2nd redeem (total=4 > 3): errorCode=605 | P1 |
| IT-06 (T6) | FR-008 | UT: `getHistoricalValue` snapshot=today → summary skipped; only blueprint value returned | Class: `StatsHistoryServiceImplTest` (unit, not IT) | See UT-01 above | P0 |

#### New Tests — Upload Count & Operation Sequences (T7–T10)

| ID | Req | Description | Setup | Assertion | Priority |
|----|-----|-------------|-------|-----------|----------|
| IT-07 | FR-009, FR-010 | GET API: DISC_CODE_PIN `numTotal` and `num_uploaded_total` combine blueprint TC/UTC with past-day summary TC/UTC (no today ops) | DISC_CODE_PIN series + blueprint TC=800, UTC=800 via `insertStatsHistoryData`; past-day summary "tc"=200, "utc"=200; `deleteRedisByKeyPattern` | `getCouponConfigForSeries` → `numTotal=1000`, `num_uploaded_total=1000`; `num_issued=0+blueprint_IC` | P0 |
| IT-08 | FR-009, FR-010, FR-011 | upload→issue→revoke→issue with blueprint history: DISC_CODE_PIN; blueprint TC=500, UTC=500, IC=70; past-day summary IC=10, RIC=5; today: upload(50), issue 3, revoke 1, issue 1 more | Blueprint seeded via `insertStatsHistoryData`; past-day summary seeded; series created with `setClient_handling_type("DISC_CODE_PIN")` + `uploadCoupons` for activation; `deleteRedisByKeyPattern` | After all ops: `getCouponConfigForSeries` → `numTotal=549`, `num_uploaded_total=549`, `num_issued=78` (see derivation §Appendix); `fetchTodayStats` → IC=4, RIC=1, TC=49 | P0 |
| IT-09 | FR-002, FR-012 | issue→redeem→reactivate with historical blueprint RC: blueprint IC=30, RC=18, past-day summary RRC=5; maxRedeem=20; issue 5, redeem 5; reactivate 3 → allows 3 more redemptions; verify num_redeemed after all ops | Blueprint IC=30, RC=18; `insertStatsSeriesSummary(past-day, "rrc", 5L)`; GENERIC series; issue 5 customers → redeem 5 (all succeed, 18+5-5=18 ≤ 20); reactivate 3 → RRC_today=3 | After reactivate+3 more redeems: total=8 redeems today; `getCouponConfigForSeries` → `num_redeemed=20` (formula: (8-3-5)+18=20); RC limit test: 11th total attempt fails with errorCode=605 | P0 |
| IT-10 | FR-009, FR-013 | upload→redeem→reactivate with blueprint TC/RC: DISC_CODE_PIN; blueprint TC=500, UTC=500, IC=20, RC=10; past-day summary TC=200, UTC=200; maxRedeem=20; today: upload(50), issue 5, redeem 3, reactivate 2, redeem 2 more | Blueprint TC=500, UTC=500, IC=20, RC=10 seeded via `insertStatsHistoryData`; past-day "tc"=200, "utc"=200 via `insertStatsSeriesSummary`; DISC_CODE_PIN + `uploadCoupons` | `getCouponConfigForSeries` → `numTotal=750`, `num_uploaded_total=750`, `num_issued=25`, `num_redeemed=13` (see §Appendix derivation) | P0 |

---

### Automation Tests — Development Cluster (`campaigns_auto/tests/luci/`)

> **Fixture strategy — Month-based self-managing series**
>
> Each cluster uses two fixture series identified by a monthly seriesCode convention.
> They are created automatically on the first test run of the month via
> `getAllCouponConfigurations(seriesCodes=[monthly_code])` and are never deleted.
> Because they accumulate data across runs, all count assertions use either:
> - **Delta pattern:** `api_after − api_before == N` — survives indefinite accumulation
> - **Pipeline integrity pattern:** `api.num_issued == getCouponsIssued_Count(active=1)` — proves DataBricks has aggregated coupons_issued into blueprint+summary correctly
>
> Monthly series codes:
> - `STATS_GEN_{year}_{month:02d}` — DISC_CODE series (coupons auto-generated on issue; no upload needed)
> - `STATS_DCP_{year}_{month:02d}` — DISC_CODE_PIN series (`uploadCoupons(N)` called on creation only)
>
> **Pipeline integrity skip rule:** When `is_new=True` (series created this run), DataBricks has not yet
> processed it. Skip the `api == db_active` assertion for that run only; the delta assertion still applies.

| ID | Req | Description | Series | Assertion Strategy | Dev-only? | Cleanup Required | Priority |
|----|-----|-------------|--------|--------------------|-----------|-----------------|----------|
| AT-DEV-01 | FR-001, FR-002 | Pipeline integrity: `num_issued` and `num_redeemed` aggregate blueprint + summary + today correctly; proves DataBricks aggregation is live | `STATS_GEN_{year}_{month:02d}` (DISC_CODE). Get-or-create via `getAllCouponConfigurations(seriesCodes=[code])`. | **DB ground truth:** `before_db = getCouponsIssued_Count(active=1)`; **API baseline:** `before_api = api.num_issued`; issue N coupons → redeem M; assert `api.num_issued − before_api == N` (delta); assert `api.num_redeemed − before_api_rc == M` (delta); **Pipeline integrity (skip if is_new):** assert `api.num_issued == getCouponsIssued_Count(active=1)` | Yes | None — series accumulates across runs by design | P0 |
| AT-DEV-02 | FR-009, FR-010 | Pipeline integrity: `numTotal` and `num_uploaded_total` aggregate blueprint TC/UTC correctly via DISC_CODE_PIN monthly series | `STATS_DCP_{year}_{month:02d}` (DISC_CODE_PIN). Get-or-create; call `uploadCoupons(N)` on first-creation only (check `is_new`). | **API baseline:** `before_api = api.numTotal`; upload M more coupons; assert `api.numTotal − before_api == M` (delta); **Pipeline integrity (skip if is_new):** assert `api.numTotal` reflects blueprint TC from prior DataBricks run (validate `api.numTotal > before_api` after upload confirms today-path write) | Yes | None — series accumulates across runs by design | P0 |
| AT-DEV-03 | FR-001, FR-002, FR-005, FR-012 | Cross-day operation sequence using day-of-month parity: even days accumulate IC+RC only; odd days add RIC+RRC (with K_revoke > N_issue to use prior-day coupons). Over time exercises `sum(RIC_window) > sum(IC_window)` → negative `currentValue` in `getSumFromCache` → validates the unclamped path and DataBricks pipeline integrity | `STATS_SEQ_{year}_{month:02d}` (DISC_CODE, dedicated sequence series; separate from STATS_GEN to avoid polluting pipeline integrity counts). **Even day:** `issue(5)`, `redeem(3)` — expected deltas: +5, +3. **Odd day:** `issue(3)`, `redeem(2)`, `revoke(min(5, 3+pool))`, `reactivate(min(2, 2+prior_redeemed))` — expected issued delta: `3 − actual_K` (negative once pool has prior coupons); expected redeemed delta: `2 − actual_J`. Guard: first odd day of month uses `actual_K = N` if pool empty. | Delta assertion for both day types; **pipeline integrity (skip if is_new):** `api.num_issued == getCouponsIssued_Count(active=1)` — on odd days where `currentValue < 0`, this specifically catches any clamping regression (`Math.max(0, currentValue)` would make API > DB active → fail). ⚠️ Requires `getCouponsIssuedList(series_id, active=1, limit=K−N)` helper in `luciDBHelper.py` to fetch prior-day coupon codes for over-revoke; verify or add. | Yes | None — net active per 2-day cycle = +3; series grows slowly, never exhausts | P1 |
| AT-DEV-04 | FR-003 | RC limit enforcement on a live cluster: fresh single-run series with `maxRedeem=3`; assert limit is enforced (success up to 3, errorCode=605 on 4th) | Fresh series per run (not monthly — needs a known small `maxRedeem`). Create with `maxRedeem=3`, issue 3 coupons, redeem all 3. | Issue 3 → redeem 3 → assert success (HTTP 200); attempt 4th redeem → assert HTTP 4xx with `errorCode=605` | Yes | Series naturally exhausts its limit; no deletion needed (all redeems already rejected) | P1 |

> **RC limit boundary vs. blueprint history:** AT-DEV-04 covers today-path RC limit only (fresh series, no blueprint RC).
> RC limit enforcement **with blueprint history** is P0-covered by IT-01. If DataBricks pipeline RC aggregation is
> suspected, AT-DEV-01's `num_redeemed` delta + pipeline integrity assertion is the cluster-level signal to use.

> **Paths not coverable in Python automation:** Revoke-then-issue against blueprint history and OU-level RC limit
> with blueprint OU-RC require blueprint seeding (`insertHistoricalStatsData`) which has no Python equivalent.
> These remain IT-only coverage areas (IT-05, IT-08).

---

### Automation Tests — Production Cluster (prod-safe)

| ID | Req | Description | Prod-safe? | Validation Method | Priority |
|----|-----|-------------|-----------|-------------------|----------|
| AT-PROD-01 | NFR-001 | Health-check: `getAllCouponConfigurations` on a known production series returns non-zero `num_issued`; validates stats read path is live | Yes — read-only | Assert HTTP 200 and `num_issued > 0` for a known high-volume series | P2 |

---

## Tenant Isolation Test Cases

| ID | Scenario | Org-A action | Org-B assertion | Expected | Type |
|----|----------|-------------|-----------------|----------|------|
| TI-01 | Blueprint seeded for orgId=0; existing test uses orgId=123 | Seed blueprint table + registry for orgId=0, issue 1 coupon for orgId=0 | `testHistoricalStatsWithCouponIssueAndRedeem` (orgId=123) registry + rolling table state unchanged | orgId=0 ops do not add rows to orgId=123 tables; no cross-shard bleed | IT (implicit in IT-01 through IT-10 running in same test run) |
| TI-02 | Redis cache keys must be org-scoped | IT-01 evicts `MIDNIGHT_EXPIRING_CACHE_NAME + "*"` for orgId=0 after seeding | `testHistoricalStatsWithCouponIssueAndRedeem` Redis cache for orgId=123 is unaffected | Pattern-based eviction must not wildcard across orgs | IT (verify key pattern in `RedisCacheKeys.MIDNIGHT_EXPIRING_CACHE_NAME`) |

> **Key confirmed:** `isDbStatsReadEnabled=true` for all orgs in IT env (`src/test/resources/application.properties:DB_STATS_READ_ENABLED:true`) — no per-test flag setup needed. OrgId=0 and orgId=123 both qualify.

---

## Regression Coverage

| ID | Existing Behaviour | Assertion | Risk if Broken | Type |
|----|-------------------|-----------|---------------|------|
| RG-01 | `testHistoricalStatsWithCouponIssueAndRedeem` (orgId=123, couponSeriesId=1L): IC limit with blueprint+summary history still enforced | Run existing test unmodified; assert first issue succeeds (errorCode=null), second fails (errorCode=626); num_issued=24, num_redeemed=15 in GET API | Main IC limit regression | IT |
| RG-02 | `shouldReflectRicAndRrcWhenRevokeOrReactivateOnDifferentDay`: RIC incremented on cross-day revoke; RRC incremented on cross-day reactivate; IC unchanged on revoke | Run existing test; assert yesterday_stats.IC=3, today_stats.RIC=2 after revoke; assert yesterday_stats.RC=3, today_stats.RRC=2 after reactivate | RIC/RRC summary write regression | IT |
| RG-03 | `shouldIssueRevokeRedeemReactivateCouponsAndValidateResponses`: TC/UTC decrements correctly after revoke (numTotal=997 from 1000); num_issued correct after revoke (22 from 25) | Run existing test unmodified | TC/UTC write path regression | IT |
| RG-04 | `StatsHistoryServiceImplTest` existing UTs (all passing): `getHistoricalValue`, `getMysqlSummaryValue`, `bulkGetHistoricalValues` unit-level behaviour | All pre-existing UT assertions pass | Service-layer logic regression | UT |

---

## Test Data Requirements

### Automation Fixture Series (monthly self-managing — AT-DEV-01–03 only)

| Field | DISC_CODE pipeline (AT-DEV-01) | DISC_CODE_PIN pipeline (AT-DEV-02) | Cross-day sequence (AT-DEV-03) |
|-------|-------------------------------|-----------------------------------|-------------------------------|
| seriesCode | `STATS_GEN_{year}_{month:02d}` | `STATS_DCP_{year}_{month:02d}` | `STATS_SEQ_{year}_{month:02d}` |
| client_handling_type | DISC_CODE | DISC_CODE_PIN | DISC_CODE |
| maxRedeem | -1 (unlimited) | -1 (unlimited) | -1 (unlimited) |
| Creation | `getAllCouponConfigurations(seriesCodes=[code])` — get or create | Same; `uploadCoupons(N)` on `is_new=True` | Same as STATS_GEN |
| Lifecycle | Accumulates all month; new series on month rollover | Same | Same |
| Deletion | Never deleted | Never deleted | Never deleted |
| Purpose | Pure IC/RC pipeline integrity | TC/UTC pipeline integrity | Cross-day RIC/RRC distribution; negative currentValue path |

**Two-layer assertion pattern (Python):**
```python
# Step 1: capture DB ground truth and API baseline before the operation
before_db  = luciDBHelper.getCouponsIssued_Count(series_id, active=1)
before_api = getAllCouponConfigurations(org_id, series_id).num_issued

# Step 2: perform the operation
issueCoupons(series_id, N)

# Step 3: assert
after_db  = luciDBHelper.getCouponsIssued_Count(series_id, active=1)
after_api = getAllCouponConfigurations(org_id, series_id).num_issued

# Delta assertion: survives indefinite accumulation across test runs
assert after_api - before_api == N

# Pipeline integrity assertion: proves DataBricks aggregated coupons_issued into blueprint+summary
# SKIP when is_new=True (DataBricks has not yet run on a series created this run)
if not is_new:
    assert after_api == after_db, \
        f"Pipeline integrity FAIL: API={after_api}, DB active={after_db}"
```

> **Why `api == db_active` is the pipeline gate:** `getCouponsIssued_Count(active=1)` reads from `coupons_issued`
> directly. DataBricks reads the same table to build blueprint IC. If `api.num_issued != db_active`, then
> blueprint IC is stale or the summary aggregation is wrong. This is the assertion that cannot be made by ITs.

**Day-parity pattern for STATS_SEQ series (AT-DEV-03):**

```python
day_of_month  = datetime.date.today().day
is_odd        = (day_of_month % 2 == 1)

before_issued = api.num_issued
before_rdm    = api.num_redeemed
pool_active   = getCouponsIssued_Count(series_id, active=1)   # confirmed helper

if is_odd:
    N, M, K, J = 3, 2, 5, 2
    # Guard: first odd day of month (pool may be empty after new series creation)
    actual_K = min(K, N + pool_active)
    # Revoke actual_K: today's N issued + (actual_K - N) from prior days
    # ⚠️ getCouponsIssuedList(series_id, active=1, limit=actual_K-N) needed for prior-day codes
    actual_J = min(J, M)      # reactivate from today's redeemed only until prior-day helper confirmed
    issueCoupons(N); redeemCoupons(M); revokeCoupons(actual_K); reactivateCoupons(actual_J)
    expected_issued_delta = N - actual_K    # negative once pool_active > 0 (from day 2 onward)
    expected_redeem_delta = M - actual_J
else:
    N, M = 5, 3
    issueCoupons(N); redeemCoupons(M)
    expected_issued_delta = N
    expected_redeem_delta = M

after_api = getCouponConfig(org_id, series_id)
assert after_api.num_issued   - before_issued == expected_issued_delta
assert after_api.num_redeemed - before_rdm    == expected_redeem_delta

# Pipeline integrity — on odd days where currentValue = (IC_window − RIC_window) < 0:
# api.num_issued = negative + blueprint_IC == db_active  (no clamping)
# If getSumFromCache ever adds Math.max(0, currentValue), api > db_active → this fails
if not is_new:
    after_db = getCouponsIssued_Count(series_id, active=1)
    assert after_api.num_issued == after_db, \
        f"Pipeline FAIL day={day_of_month} is_odd={is_odd}: API={after_api.num_issued} DB_active={after_db}"
```

**Negative `currentValue` mechanics over a month (snapshotDate = yesterday):**

| Day | IC window | RIC window | currentValue | num_issued |
|-----|-----------|------------|--------------|------------|
| 1 (odd, pool=0) | 3 | 3 (actual_K=N) | 0 | blueprint_IC |
| 2 (even) | 3+5=8 | 3 | +5 | blueprint_IC+5 |
| 3 (odd, pool=5) | 8+3=11 | 3+5=8 | +3 | blueprint_IC+3 |
| 4 (even) | 11+5=16 | 8 | +8 | blueprint_IC+8 |
| 5 (odd, pool≥5) | 16+3=19 | 8+5=13 | +6 | blueprint_IC+6 |
| … | … | … | converges around +3 per 2-day cycle | stable |

> Short snapshot window (snapshotDate = yesterday): window contains only today.
> On odd day 3+: `IC_today=3 < RIC_today=5` → `currentValue=−2` → `num_issued = blueprint_IC − 2`.
> `db_active` = total_active = same value if DataBricks and formula are correct.
> The pipeline integrity assertion is the guard.

---

### Blueprint Seeding (via `insertStatsHistoryData`)

For IT-07, IT-08, IT-10 (DISC_CODE_PIN TC/UTC in blueprint):
```java
// Creates StatsHistory objects for TC/UTC/IC — insert into rolling table
StatsHistory tcRow  = Fixture.getStatsHistory(orgId, seriesId, "tc",  800L, DEFAULT_ENTITY_TYPE, DEFAULT_ENTITY_ID);
StatsHistory utcRow = Fixture.getStatsHistory(orgId, seriesId, "utc", 800L, DEFAULT_ENTITY_TYPE, DEFAULT_ENTITY_ID);
StatsHistory icRow  = Fixture.getStatsHistory(orgId, seriesId, "ic",   50L, DEFAULT_ENTITY_TYPE, DEFAULT_ENTITY_ID);
insertStatsHistoryData(orgId, activeTable, Lists.newArrayList(tcRow, utcRow, icRow));
```

> ⚠️ **Developer action required:** Verify `Fixture.getStatsHistory(orgId, seriesId, statsKey, value, entityType, entityId)` exists and creates a valid `StatsHistory` object. If it does not exist, create it or use direct `new StatsHistory()` construction before calling `insertStatsHistoryData`.

For IT-01, IT-02, IT-03, IT-04 (IC/RC in blueprint):
```java
// Shorthand helper handles IC and RC only
insertHistoricalStatsData(orgId, (long) savedSeries.getId(), activeTable, icValue, rcValue);
```

### Summary Seeding (via `insertStatsSeriesSummary`)

| Test | Keys to seed | Date |
|------|-------------|------|
| IT-02 | "ic"=1 (twoDaysAgo), "ic"=2 + "rc"=1 (yesterday) | Past days within snapshotDate+1→today window |
| IT-03 | "ric"=3 (threeDaysAgo), "ric"=2 (yesterday) | RIC in summary — NOT in blueprint |
| IT-04 | "ic"=4 (yesterday) | Summary fallback path |
| IT-07 | "tc"=200, "utc"=200 (twoDaysAgo) | DISC_CODE_PIN TC/UTC history |
| IT-08 | "ic"=10, "ric"=5 (twoDaysAgo) | Past-day IC/RIC |
| IT-09 | "rrc"=5 (twoDaysAgo) | Past-day RRC reduces effective RC cap |
| IT-10 | "tc"=200, "utc"=200 (twoDaysAgo) | Past TC/UTC for DISC_CODE_PIN |

### Cache Eviction

All new ITs must call after seeding and before triggering the limit/GET path:
```java
deleteRedisByKeyPattern(RedisCacheKeys.MIDNIGHT_EXPIRING_CACHE_NAME + "*");
```
Source: `BaseIntegrationTest.java`; method evicts the midnight-expiring cache used by `@Cacheable getHistoricalValue()`.

### Series ID Rule

```java
// ALWAYS:
long seriesId = (long) savedSeries.getId();
insertHistoricalStatsData(orgId, seriesId, activeTable, icValue, rcValue);

// NEVER:
insertHistoricalStatsData(orgId, 1L, activeTable, icValue, rcValue);  // fragile, breaks on non-empty DB
```

### Series Type Rule (resolved Q1)

| Tests | Series type | `uploadCoupons()` needed? | `numTotal` source |
|-------|------------|--------------------------|------------------|
| IT-01, IT-02, IT-03, IT-04 | **DISC_CODE** (default from `CouponConfigurationFixture`) | No — coupons auto-generated on issue | `maxCreate` field, NOT blueprint+summary TC — do NOT assert `numTotal` from history |
| IT-05 | OU-configured — follow `RedeemCouponIntegrationTest` pattern | Per that test's convention | N/A |
| IT-07, IT-08, IT-10 | **DISC_CODE_PIN** (must call `.setClient_handling_type("DISC_CODE_PIN")`) | **Yes — `uploadCoupons()` required** for TC/UTC summary rows to be realistic | `summary_TC + blueprint_TC` |
| IT-09 | DISC_CODE (no upload needed) | No | N/A |

### Mandatory `@After` Cleanup (resolved Q5)

**No framework-level cleanup is guaranteed.** Each new IT must clean up after itself:

```java
@After
public void cleanupStatsTestData() {
    // 1. Delete registry entries for orgId=0 created in this test
    statsHistoryRegistryDao.deleteByOrgId(0);

    // 2. Drop rolling tables created by createRollingTable()
    //    (store table names as instance fields in @Before/@Test setup)
    if (activeTableName != null) {
        jdbcTemplate.execute("DROP TABLE IF EXISTS " + activeTableName);
    }

    // 3. Evict Redis cache to prevent stale entries bleeding into next test
    deleteRedisByKeyPattern(RedisCacheKeys.MIDNIGHT_EXPIRING_CACHE_NAME + "*");
}
```

> If `statsHistoryRegistryDao.deleteByOrgId()` does not exist, use a direct SQL delete:
> `jdbcTemplate.execute("DELETE FROM stats_history_registry WHERE org_id = 0")`.

### Feature Flags
`isDbStatsReadEnabled=true` for all orgs by default in IT env. No `ReflectionTestUtils.setField` needed.

### Org IDs
- IT-01 through IT-10: use `orgId=0`
- Regression tests RG-01: use `orgId=Fixture.orgId=123` (unchanged)
- TI-01/TI-02: both orgs run in the same test run; isolation is structural (different org IDs = different DB shards + different Redis key prefixes)

---

## Appendix: Value Derivations for IT-08 and IT-10

### IT-08 Final Value Derivation

**Series:** DISC_CODE_PIN, maxCreate=90
**Blueprint:** TC=500, UTC=500, IC=70
**Past-day summary:** IC=10, RIC=5 (seeded before today's ops)
**Today ops:** `uploadCoupons(50)` → TC_today=50, UTC_today=50; issue 3 → IC_today=3; revoke 1 → RIC_today=1, TC_today=49; issue 1 more → IC_today=4

```
cachedStat (bulkFetchStatsFromCacheOrDb, snapshotDate+1 → today):
  IC  = 10(past) + 4(today)  = 14
  RIC = 5(past)  + 1(today)  = 6
  TC  = 0(past)  + 49(today) = 49   [upload 50 minus 1 revoke decrement]
  UTC = 0(past)  + 49(today) = 49

getSumFromCache(IC):  (14 − 6) + 70  = 78  → num_issued   = 78
getSumFromCache(TC):  49       + 500  = 549 → numTotal      = 549
getSumFromCache(UTC): 49       + 500  = 549 → num_uploaded_total = 549
```

> Note: past-day TC/UTC are NOT seeded in IT-08 for simplicity (only past-day IC/RIC are seeded).
> If past-day TC/UTC were also seeded at 100 each, numTotal would be 649. Developer should choose one variant and stick with it.

### IT-10 Final Value Derivation

**Series:** DISC_CODE_PIN, maxCreate=unlimited, maxRedeem=20
**Blueprint:** TC=500, UTC=500, IC=20, RC=10
**Past-day summary:** TC=200, UTC=200 (seeded)
**Today ops:** `uploadCoupons(50)` → TC_today=50, UTC_today=50; issue 5 → IC_today=5; redeem 3 → RC_today=3; reactivate 2 → RRC_today=2; redeem 2 more → RC_today=5

```
cachedStat (snapshotDate+1 → today):
  IC  = 0(past)  + 5(today)  = 5
  RIC = 0        + 0          = 0
  RC  = 0(past)  + 5(today)  = 5   [3 + 2 more]
  RRC = 0(past)  + 2(today)  = 2
  TC  = 200(past) + 50(today) = 250  [upload 50, no revokes in this test]
  UTC = 200(past) + 50(today) = 250

getSumFromCache(IC):  (5 − 0)  + 20  = 25  → num_issued         = 25
getSumFromCache(RC):  (5 − 2)  + 10  = 13  → num_redeemed       = 13
getSumFromCache(TC):  250      + 500  = 750 → numTotal           = 750
getSumFromCache(UTC): 250      + 500  = 750 → num_uploaded_total = 750
```

### IT-09 Final Value Derivation

**Series:** GENERIC, maxCreate=unlimited, maxRedeem=20
**Blueprint:** IC=30, RC=18
**Past-day summary:** RRC=5 (seeded — 5 historical reactivations reduce effective RC)
**Today ops:** Issue 5 → IC_today=5; redeem 5 → RC_today=5; reactivate 3 → RRC_today=3; redeem 3 more → RC_today=8

```
RC limit check during 8th redeem attempt:
  mysql_RC_today(post-increment) = 8
  historicalRC = blueprint_RC(18) + summary_RC_past(0) = 18
  historicalRRC = summary_RRC_past(5) + summary_RRC_today(3) = 8
  redeemedCouponCount = 8 + 18 − 8 = 18 ≤ 20 → success

9th attempt: 9 + 18 − 8 = 19 ≤ 20 → success
10th attempt: 10 + 18 − 8 = 20 ≤ 20 → success  [AT limit]
11th attempt: 11 + 18 − 8 = 21 > 20 → FAIL (errorCode=605)

GET API after 10 successful redeems, 3 reactivations:
  cachedStat: RC=10(today), RRC=3+5(past)=8 total
  getSumFromCache(RC): (10 − 8) + 18 = 20 → num_redeemed = 20
  getSumFromCache(IC): (5 − 0) + 30   = 35 → num_issued   = 35
```

---

## Test Coverage Summary

| Layer | Count | % of Total |
|-------|-------|-----------|
| Unit | 5 (UT-01–UT-05) | ~24% |
| Integration | 10 (IT-01–IT-10) | ~48% |
| Automation (dev) | 4 (AT-DEV-01–04) | ~14% |
| Automation (prod) | 1 (AT-PROD-01) | ~5% |
| Regression | 4 (RG-01–04) | ~19% |
| **Total distinct** | **~22** | **100%** |

Requirements coverage: 13/13 FRs covered (100%), 3/3 NFRs covered (100%)

---

## Minimum Viable Test Set (Fast-Release Gate)

If release is time-constrained, these P0 tests form the minimum gate before merging:

**Unit:**
- UT-01 (snapshot=today edge case)
- UT-02 (isMysql enum correctness)
- UT-04 (absent registry no NPE)

**Integration:**
- IT-01 (RC limit with blueprint history)
- IT-02 (GET API blueprint+summary assembly)
- IT-03 (RIC in summary reduces effectiveIssued)
- IT-04 (absent registry fallback)
- IT-07 (DISC_CODE_PIN numTotal/UTC from blueprint)
- IT-08 (upload→issue→revoke→issue with history)
- IT-09 (issue→redeem→reactivate with history)
- IT-10 (upload→redeem→reactivate with history)

**Regression (must not break):**
- RG-01, RG-02, RG-03, RG-04 (all existing tests pass unmodified)

Full coverage: add IT-05 (OU-level), UT-03 (getLongFields), UT-05 (bulkGet), AT-DEV-01 and AT-DEV-02 (pipeline integrity, P0) after QA deploy — no separate infra coordination needed; monthly series are self-creating.

---

## Recommended Implementation Order

```
1. UT-01, UT-02, UT-04  — fastest; zero DB setup; validates service-layer contracts
2. IT-04                — simplest IT (no rolling table, no registry); validates fallback path first
3. IT-01                — RC limit with blueprint; builds the blueprint-seeding pattern for all other ITs
4. IT-02                — GET API assembly; reuses IT-01 setup pattern
5. IT-03                — RIC in summary; short test once blueprint pattern is established
6. IT-07                — DISC_CODE_PIN blueprint TC/UTC; first test using insertStatsHistoryData for TC/UTC
7. IT-08                — upload→issue→revoke→issue sequence; builds on IT-07 + IT-03
8. IT-09                — issue→redeem→reactivate with history; validate RRC offsets RC cap
9. IT-10                — upload→redeem→reactivate; combines IT-07 + IT-09 patterns
10. IT-05 (after spike) — OU-level; requires resolved Q3/Q4
11. UT-03, UT-05        — supplementary unit tests
12. AT-DEV-01–02 (P0)   — run post-deploy to QA; monthly series auto-created on first run; assert pipeline integrity after DataBricks has processed the series (is_new=False)
13. AT-DEV-03–04 (P1)   — full operation sequence + RC limit boundary; run after AT-DEV-01/02 confirm pipeline is live
```

---

## Definition of Done

- [ ] All UT-01 through UT-05 pass in `mvn test`
- [ ] All IT-01 through IT-10 pass in `mvn test -pl luci -Dtest=StatsSeriesSummaryIntegrationTest`
- [ ] `getCouponConfigForSeries` returns correct `num_issued` from blueprint + summary for IC path
- [ ] `getCouponConfigForSeries` returns correct `num_redeemed` from blueprint + summary for RC path
- [ ] `getCouponConfigForSeries` returns correct `numTotal` and `num_uploaded_total` from blueprint + summary TC/UTC for DISC_CODE_PIN
- [ ] RC limit enforced correctly when blueprint RC + today's MySQL RC crosses maxRedeem
- [ ] IC limit enforced correctly when historical RIC from summary reduces effectiveIssued
- [ ] Operation sequences (upload→issue→revoke→issue, issue→redeem→reactivate, upload→redeem→reactivate) produce correct stats in GET API with blueprint history
- [ ] Tenant isolation verified: orgId=0 operations do not affect orgId=123 registry/blueprint/Redis state
- [ ] All regression tests (RG-01–RG-04) still pass unmodified
- [ ] No hardcoded series ID `1L` in any new test method
- [ ] `deleteRedisByKeyPattern(MIDNIGHT_EXPIRING_CACHE_NAME + "*")` called in every new IT after seeding
- [ ] AT-DEV-01 passes on QA cluster: `api.num_issued − before == N` (delta) and `api.num_issued == getCouponsIssued_Count(active=1)` (pipeline integrity, post-DataBricks run)
- [ ] AT-DEV-02 passes on QA cluster: `api.numTotal − before == M` (delta) for DISC_CODE_PIN monthly series

---

## Signals for Arch-Investigator Inbox

The following should be added to the arch-investigator's investigation checklist:

1. **Always check `isMysql()` on `CouponSeriesStatisticsField` before asserting where a stat field lives** — RIC and RRC are `isMysql=true` and cannot appear in blueprint tables under any circumstances.

2. **For DISC_CODE_PIN TC/UTC blueprint seeding, `insertHistoricalStatsData(orgId, seriesId, table, ic, rc)` is insufficient** — only IC and RC are supported. Use `insertStatsHistoryData(orgId, table, List<StatsHistory>)` with explicit `StatsHistory` objects for TC/UTC/UC keys.

3. **`bulkFetchHistoryValues()` in `MySQLCouponSeriesStatisticsReadService` is NOT blueprint+summary** — it is blueprint-only for IC/RC/TC/UTC. Summary range (snapshotDate+1 → today) is owned by `bulkFetchStatsFromCacheOrDb()`.

4. **Upload count stats (`num_uploaded`, `num_uploaded_total`, `numTotal`) have NO blueprint+summary IT coverage** — only today-path is tested in existing suite. Any change to `getUploadedCount()`, `getUploadedTotalCount()`, or `getTotalCount()` in `MySQLCouponSeriesStatisticsReadService` is currently unprotected by history-path tests.

5. **Revoke operation decrements TC and UTC in today's `stats_series_summary`** — confirmed by existing test (numTotal=997 after 3 revokes from 1000 uploads). Any TC assertion after a revoke must account for this decrement.

---

*End of test plan document.*
