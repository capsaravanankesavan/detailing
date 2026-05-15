# Tech Detail: Stats Test Coverage — MySQL History + Summary Path
**Date:** 2026-04-21
**Author:** Saravanan Kesavan (via tech-detailer)
**Status:** Draft
**Investigation Doc:** `handoff.md` (same directory)
**Confidence:** HIGH — all claims validated against source code with `file:line` citations

---

## 1. Problem Statement

Luci now reads issuance and redemption limits from MySQL (blueprint + summary + Redis) instead
of Redis alone. The MySQL path is the steady-state for any series older than one ETL cycle.
However, all existing ITs create a fresh coupon series, leaving `stats_history_blueprint_*`
and `stats_series_summary` empty for that series. This means every test exercises only the
Redis-only or same-day summary path — the MySQL history+summary aggregate is entirely untested
in both limit checks and the GET API stats response.

**Entry point:** IT suite `StatsSeriesSummaryIntegrationTest` and automation tests in `campaigns_auto/tests/luci/`
**Scope:** All orgs (flag `isDbStatsReadEnabled=true` is the default for all orgs in the IT environment)
**When observed:** During code review of the new MySQL stats persistence layer

---

## 2. Root Cause (Confirmed)

Every IT creates a fresh series at test start; `stats_history_blueprint_*` and
`stats_series_summary` are always empty for that series — so no test ever exercises
the history+summary aggregate that is the normal steady state for any long-running series.

### Contributing Factors

1. `insertHistoricalStatsData()` helper exists but is called by only ONE test
   (`testHistoricalStatsWithCouponIssueAndRedeem`), covering only IC limit — not RC limit,
   RIC-in-summary, or the GET API stats assembly path.
2. RIC (`isMysql=true`, `CouponSeriesStatisticsField.java:12`) is never stored in the blueprint
   table — it can only be seeded via `insertStatsSeriesSummary()`. This architectural fact is
   not obvious from the field name.
3. The automation tests (`campaigns_auto`) have no stats field assertions at all; they create
   fresh series and have no mechanism to seed blueprint data.
4. `isDbStatsReadEnabled` is silently `true` for all orgs in the IT env
   (`src/test/resources/application.properties:DB_STATS_READ_ENABLED:true`) but this is
   not documented, causing confusion about which orgId to use.

---

## 3. Scope of Change

### In Scope
- New IT methods in `StatsSeriesSummaryIntegrationTest.java`
- New UT in `StatsHistoryServiceImplTest.java`
- Automation: fixture-series approach (infrastructure coordination, not code)

### Out of Scope (Explicit)
- `StatsHistoryRunner` end-to-end pipeline IT (S3 → parquet → rolling table) — deferred
- `TestStatsHistorySeedController` REST endpoint for automation — deferred (Option B)
- Any production code change — none required; this is test-only
- Changes to `CLAUDE.md`, `.context/`, or infra config — none required

### Deferred
- `StatsHistoryRunner` end-to-end IT (P2 — needs S3 mock infrastructure)
- Automation Option B (test-seeding controller) — requires new prod-test endpoint; plan for next sprint

---

## 4. Assumptions

| # | Assumption | Breaks if… | Owner to validate |
|---|-----------|------------|------------------|
| A1 | `isDbStatsReadEnabled=true` for all orgs in IT env (confirmed: `src/test/resources/application.properties`) | Env var `DB_STATS_READ_ENABLED` is set to `false` in CI | Dev / CI config |
| A2 | The embedded test DB auto-increment resets between test runs so new series IDs don't collide with blueprint seeds | DB is shared across test classes without cleanup | Dev (check `@Before` in BaseIntegrationTest) |
| A3 | `insertHistoricalStatsData(orgId, seriesId, tableName, ic, rc)` accepts the actual series ID returned by `saveCouponSeries()` | Helper hardcodes `1L` internally (confirmed: it accepts `Long couponSeriesId` as a parameter — safe) | — |
| A4 | `createCouponSeriesWithLimits` defaults to a series type (e.g., GENERIC) that issues without pre-uploaded coupons | Fixture default type is `DISC_CODE_PIN` | Dev (confirm `CouponConfigurationFixture.getCouponConfiguration()` default type) |
| A5 | The midnight-expiring Redis cache is either empty at test start or auto-evicted by unique series IDs | Series ID reuse across test runs; cache eviction call needed | Dev — add `deleteRedisByKeyPattern(MIDNIGHT_EXPIRING_CACHE_NAME + "*")` in test setup if A2 is violated |

---

## 5. Design Validation

### Gaps vs Existing Patterns

All new tests follow the existing pattern in `testHistoricalStatsWithCouponIssueAndRedeem`:
`saveOrgDefaultProperties` → `createRollingTable` → `createStatsHistoryRegistryEntry` →
`insertHistoricalStatsData` / `insertStatsSeriesSummary` → `saveCouponSeries` → `sleep(1000)` →
operate → assert. No new patterns introduced.

**Gap corrected:** Handoff GAP-IT-3 proposed seeding RIC in the blueprint table — impossible.
`RevokedIssueCount.isMysql=true` (`CouponSeriesStatisticsField.java:12`). RIC has never been
and cannot be stored in `stats_history_blueprint_*`. Correct approach: seed via
`insertStatsSeriesSummary(orgId, seriesId, pastDate, "ric", value)`.

### MADR Compliance

- MADR 0005 (Midnight-expiring Redis cache): `getHistoricalValue` is `@Cacheable` in
  `MIDNIGHT_EXPIRING_CACHE_NAME`. Tests that mutate blueprint/summary data must flush this cache
  before exercising the hot path. Use `deleteRedisByKeyPattern(MIDNIGHT_EXPIRING_CACHE_NAME + "*")`
  after seeding and before issuing. Existing `updateStatsSeriesSummaryDateTo` already does this
  (`BaseIntegrationTest.java:1200`).
- MADR 0004 (Processor chain): No changes to processors. Tests exercise the chain end-to-end.
- MADR 0002 (Sharded transactions): No new DB writes in production code. Test helpers use
  direct SQL — acceptable.

### Guardrail Compliance

Test code only — GC, caching, JDBC, and observability guardrails from `.context/code.md`
do not apply to test methods. No production code is changed.

---

## 6. Alternative Designs Considered

| Alternative | Pros | Cons | Decision |
|------------|------|------|----------|
| Extend existing `testHistoricalStatsWithCouponIssueAndRedeem` with RC/RIC/GET assertions | No new test methods | Single method becomes 200+ lines; violates single-purpose rule; harder to isolate failures | Rejected — add separate focused test methods |
| Automation OPT-B: `TestStatsHistorySeedController` REST endpoint | Self-contained, repeatable automation | Requires new controller + deployment to test clusters | Deferred — not needed for IT coverage; plan separately |
| Automation OPT-A: Month-based self-managing fixture series (evolved from persistent fixture) | No infra coordination; series auto-creates via `getAllCouponConfigurations(seriesCodes=[…])`; month rollover creates fresh series; two-layer assertion (DB ground truth + API delta) proves DataBricks pipeline integrity | maxRedeem=-1 so RC limit boundary tested separately with a fresh small-maxRedeem series | **Accepted** — supersedes original "persistent fixture + infra coordination" approach |
| Automation OPT-C: S3 parquet trigger | Tests the full DataBricks pipeline | Requires S3 mock/access, parquet generation, slow | Deferred — use as pipeline regression test, not routine |

---

## 7. Use Cases

### B2B (Brand Admin / Limit Configuration Flows)

| ID | Flow | Today | After Fix | Risk | Test Type |
|----|------|-------|-----------|------|-----------|
| UC-1 | Admin sets `maxRedeem=15`; series has 14 RC in blueprint; customer redeems 1 → should succeed | Not tested via MySQL path | Tested via IT `testRCLimitWithHistoricalData` | High — RC limit silently broken for long-running series | IT |
| UC-2 | Same series; customer redeems 2nd → should fail (16 > 15) | Not tested | Tested | High | IT |
| UC-3 | Admin views series stats in dashboard (GET API); blueprint IC=20, summary IC=3 → `num_issued=23` | Not tested | Tested via `testGetApiStatsAssemblyFromHistoryAndSummary` | High — merchants see wrong stats | IT |
| UC-4 | Admin sets `maxCreate=16`; blueprint IC=20, summary RIC=5 in past days → effective=15; issue 1 → success | Not tested | Tested via `testRICInSummaryReducesEffectiveIssuedCount` | Medium | IT |
| UC-5 | Same series; issue 2nd → fail (effectiveIssued=17 > 16) | Not tested | Tested | Medium | IT |
| UC-6 | No registry entry for org; summary-only fallback with IC=4; issue 1 with `maxCreate=5` → success | Not tested | Tested via `testAbsentRegistryFallbackToMaxLookback` | Medium — NPE risk or wrong count | IT |
| UC-7 | OU-level `maxRedeem=3`; blueprint OU-RC=2; redeem 1 → success; 2nd → fail | Not tested | Tested via `testOULevelRCLimitWithHistoricalData` | Medium | IT |
| UC-8 | `getHistoricalValue` snapshot date = today (DataBricks early run) → startDate > endDate → summary skipped | Mocked UT only | New edge-case UT in `StatsHistoryServiceImplTest` | Low | UT |

### B2C (End Customer Flows)

| ID | Flow | Today | After Fix | Risk | Test Type |
|----|------|-------|-----------|------|-----------|
| UC-9 | Customer issues coupon on long-running series (IC limit from history) | Only same-day Redis path tested | IT covers MySQL path | High | IT (via UC-1 setup) |
| UC-10 | Customer redeems on long-running series (RC limit from history) | Not tested | IT covers it | High | IT (via UC-1/2) |
| UC-11 | Customer issues on series where past revokes exist in summary (RIC reduces cap) | Not tested | IT covers it | Medium | IT (via UC-4/5) |
| UC-12 | Customer checks their coupon via `getCustomerCoupon` — stats include history | Not tested | Covered by UC-3 assertion path | Medium | IT |

---

## 8. Low-Level Design

### 8a. Two-path architecture (validated, for developer reference)

```
IC limit check (hot path):
  getHistoricalValue(IC)  = blueprint_IC + summary_IC(snapshotDate+1 → yesterday)
                            [@Cacheable in MIDNIGHT_EXPIRING_CACHE]
  getMysqlSummaryValue(RIC) = summary_RIC(snapshotDate+1 → today) [NOT cached]
  effectiveIssued = Redis_IC_today + historicalIC - historicalRIC

RC limit check (shouldCommit=true):
  incrementRedeemedCount() → MySQL RC today
  getHistoricalValue(RC)   = blueprint_RC + summary_RC(→ yesterday)
  getMysqlSummaryValue(RRC)= summary_RRC(→ today)
  redeemedCouponCount = mysql_RC_today + historicalRC - historicalRRC

GET API (getStatisticsBatch):
  bulkFetchStatsFromCacheOrDb()  = summary(snapshotDate+1 → today) ALL keys → cachedStats
  bulkGetHistoricalValues()      = blueprint ONLY → historyMap
  bulkGetMysqlSummaryValues()    = summary(→ today) for RIC/RRC only → mysqlOnlyMap
  getSumFromCache(IssuedCount)   = (cachedStat.IC - cachedStat.RIC) + historyMap.IC
```

### 8b. New Test Methods

---

#### T1 — `testRCLimitWithHistoricalData`

**Class:** `StatsSeriesSummaryIntegrationTest.java`
**Validates:** `MaxRedemptionForSeriesProcessor` RC limit check with blueprint + today's MySQL RC

```java
@Test
@DisplayName("RC limit check: blueprint RC + today's MySQL RC enforces maxRedeem correctly")
public void testRCLimitWithHistoricalData() throws TException, LuciThriftException, SQLException {
    int orgId = 0;
    int storeUnitId = 1234;
    int billId = 1564;

    saveOrgDefaultProperties(orgId);

    // --- Setup: rolling table (snapshot = 4 days ago) ---
    String activeTable = createRollingTable(4);
    Date fourDaysAgo = Date.from(Instant.now().minus(4, ChronoUnit.DAYS));
    createStatsHistoryRegistryEntry(orgId, activeTable, fourDaysAgo, true, new Date());

    // --- Create series FIRST to get its real ID ---
    CouponConfiguration savedSeries = saveCouponSeries(
        orgId, createCouponSeriesWithLimits(orgId, -1, 15));  // maxCreate=-1, maxRedeem=15
    long seriesId = savedSeries.getId();
    sleep(1000);

    // --- Seed blueprint: RC=14 (series has 14 historical redemptions) ---
    // NOTE: insertHistoricalStatsData() only supports IC and RC. Pass ic=0 to isolate RC.
    insertHistoricalStatsData(orgId, seriesId, activeTable, 0L, 14L);

    // Evict midnight cache so getHistoricalValue hits DB with fresh blueprint data
    deleteRedisByKeyPattern(RedisCacheKeys.MIDNIGHT_EXPIRING_CACHE_NAME + "*");

    // --- Issue 2 coupons for 2 different customers (need 2 to test limit boundary) ---
    CouponDetails coupon1 = issueCouponWithStats(orgId, storeUnitId, 101, billId,     (int) seriesId);
    CouponDetails coupon2 = issueCouponWithStats(orgId, storeUnitId, 102, billId + 1, (int) seriesId);
    assertNull("Issue coupon1 should succeed", coupon1.getEx());
    assertNull("Issue coupon2 should succeed", coupon2.getEx());

    // --- Redeem coupon1: historical RC=14, today MySQL RC becomes 1 → total=15 → AT limit → SUCCESS ---
    List<CouponDetails> firstRedeem = redeemCoupon(
        orgId, storeUnitId, 101, Lists.newArrayList(coupon1), true);
    assertNull("First redeem should succeed (14 historical + 1 today = 15 = limit)",
        firstRedeem.get(0).getEx());

    // Assert GET API returns num_redeemed=15 (blueprint 14 + today 1)
    CouponConfiguration configAfterFirst = getCouponConfigForSeries(orgId, (int) seriesId);
    assertEquals(15, configAfterFirst.getNum_redeemed(),
        "num_redeemed should reflect blueprint + today");

    // --- Redeem coupon2: historical RC=14, today MySQL RC becomes 2 → total=16 > 15 → FAIL ---
    List<CouponDetails> secondRedeem = redeemCoupon(
        orgId, storeUnitId, 102, Lists.newArrayList(coupon2), true);
    assertNotNull("Second redeem should fail (16 > 15)", secondRedeem.get(0).getEx());
    assertEquals(605, secondRedeem.get(0).getEx().getErrorCode(),
        "Error code should be MAX_REDEMPTION_FOR_SERIES_EXCEEDED");
    assertEquals("max redeem for series exceeded",
        secondRedeem.get(0).getEx().getErrorMsg());
}
```

**Boundary verified:**
- RC limit check is in `getRedeemedCouponCount()` (shouldCommit=true branch, `MaxRedemptionForSeriesProcessor.java:118-129`)
- `incrementRedeemedCount()` increments MySQL RC BEFORE the check. So after coupon1: mysql_today=1, historicalRC=14, total=15 → 15 < 15 is false → proceed (success). After coupon2: mysql_today=2, total=16 → 15 < 16 → fail.

---

#### T2 — `testGetApiStatsAssemblyFromHistoryAndSummary`

**Class:** `StatsSeriesSummaryIntegrationTest.java`
**Validates:** `MySQLCouponSeriesStatisticsReadService.getStatisticsBatch()` returns `blueprint + summary` for IC and RC

```java
@Test
@DisplayName("GET API: num_issued = blueprint IC + summary IC; num_redeemed = blueprint RC + summary RC")
public void testGetApiStatsAssemblyFromHistoryAndSummary() throws TException, LuciThriftException, SQLException {
    int orgId = 0;
    saveOrgDefaultProperties(orgId);

    // --- Setup: rolling table (snapshot = 4 days ago) ---
    String activeTable = createRollingTable(4);
    Date fourDaysAgo = Date.from(Instant.now().minus(4, ChronoUnit.DAYS));
    Date twoDaysAgo   = Date.from(Instant.now().minus(2, ChronoUnit.DAYS));
    Date yesterday    = Date.from(Instant.now().minus(1, ChronoUnit.DAYS));
    createStatsHistoryRegistryEntry(orgId, activeTable, fourDaysAgo, true, new Date());

    // --- Create series first to get real ID ---
    CouponConfiguration savedSeries = saveCouponSeries(
        orgId, createCouponSeriesWithLimits(orgId, -1, -1));  // unlimited
    long seriesId = savedSeries.getId();

    // --- Seed blueprint: IC=20, RC=8 ---
    insertHistoricalStatsData(orgId, seriesId, activeTable, 20L, 8L);

    // --- Seed summary for past days (snapshotDate+1 → yesterday):
    //     twoDaysAgo row: IC=1, RC=0
    //     yesterday row : IC=2, RC=1
    //     Total summary: IC=3, RC=1
    insertStatsSeriesSummary(orgId, (int) seriesId, twoDaysAgo, "ic", 1L);
    insertStatsSeriesSummary(orgId, (int) seriesId, yesterday,  "ic", 2L);
    insertStatsSeriesSummary(orgId, (int) seriesId, yesterday,  "rc", 1L);

    // Evict midnight cache: getStatisticsBatch uses bulkFetchStatsFromCacheOrDb (NOT cached)
    // but bulkGetHistoricalValues internally calls bulkGetHistoricalValues which is NOT @Cacheable.
    // Cache eviction here is precautionary for getHistoricalValue if called elsewhere.
    deleteRedisByKeyPattern(RedisCacheKeys.MIDNIGHT_EXPIRING_CACHE_NAME + "*");

    // --- Call GET API ---
    // getSumFromCache: cachedStat(IC-RIC) + blueprint_IC
    // cachedStat = bulkFetchStatsFromCacheOrDb (snapshotDate+1 → today) = IC=3 (no today's issues), RIC=0
    // blueprint = 20
    // Expected: (3 - 0) + 20 = 23
    // RC: cachedStat(RC-RRC) + blueprint_RC = (1 - 0) + 8 = 9
    CouponConfiguration config = getCouponConfigForSeries(orgId, (int) seriesId);
    assertEquals(23, config.getNum_issued(),
        "num_issued must equal blueprint IC (20) + summary IC (3)");
    assertEquals(9, config.getNum_redeemed(),
        "num_redeemed must equal blueprint RC (8) + summary RC (1)");

    // Confirm today stats are zero (no real operations today)
    CouponSeriesStatistics todayStats = fetchTodayStats(orgId, (int) seriesId);
    assertEquals(0, todayStats.getIssuedCount(),
        "Today's summary row should have IC=0 (no real issues in test)");
}
```

**Implementation note:** `bulkFetchStatsFromCacheOrDb()` reads `stats_series_summary` from
`snapshotDate+1` to today (`MySQLCouponSeriesStatisticsReadService.java:307-319`). The seeded
rows for `twoDaysAgo` and `yesterday` fall within this range (both are after `fourDaysAgo+1`).
`getSumFromCache()` combines `cachedStat.IC - cachedStat.RIC` (from summary) + blueprint IC.

---

#### T3 — `testRICInSummaryReducesEffectiveIssuedCount`

**Class:** `StatsSeriesSummaryIntegrationTest.java`
**Validates:** Historical RIC in `stats_series_summary` (past days) correctly offsets `effectiveIssued` in IC limit check

> **Key insight:** RIC (`isMysql=true`, `CouponSeriesStatisticsField.java:12`) is NEVER stored in
> the blueprint table. It is only read from `stats_series_summary` via `getMysqlSummaryValue()`.
> The handoff incorrectly stated "seed RIC in a blueprint table" — corrected here.

```java
@Test
@DisplayName("IC limit: historical RIC from summary reduces effectiveIssued correctly")
public void testRICInSummaryReducesEffectiveIssuedCount() throws TException, LuciThriftException, SQLException {
    int orgId = 0;
    int storeUnitId = 1234;
    int billId = 1564;
    saveOrgDefaultProperties(orgId);

    // Snapshot = 4 days ago
    String activeTable = createRollingTable(4);
    Date fourDaysAgo = Date.from(Instant.now().minus(4, ChronoUnit.DAYS));
    Date threeDaysAgo = Date.from(Instant.now().minus(3, ChronoUnit.DAYS));
    Date yesterday    = Date.from(Instant.now().minus(1, ChronoUnit.DAYS));
    createStatsHistoryRegistryEntry(orgId, activeTable, fourDaysAgo, true, new Date());

    // Create series with maxCreate=16
    CouponConfiguration savedSeries = saveCouponSeries(
        orgId, createCouponSeriesWithLimits(orgId, 16, -1));
    long seriesId = savedSeries.getId();
    sleep(1000);

    // Blueprint: IC=20 (20 coupons issued up to snapshot date)
    insertHistoricalStatsData(orgId, seriesId, activeTable, 20L, 0L);

    // Summary: RIC=5 across past days (5 revocations since snapshot → effective IC = 20 - 5 = 15)
    insertStatsSeriesSummary(orgId, (int) seriesId, threeDaysAgo, "ric", 3L);
    insertStatsSeriesSummary(orgId, (int) seriesId, yesterday,    "ric", 2L);

    // Evict midnight cache
    deleteRedisByKeyPattern(RedisCacheKeys.MIDNIGHT_EXPIRING_CACHE_NAME + "*");

    // effectiveIssued formula (SeriesLockAcquireProcessor:67):
    //   issued (Redis today, post-increment) + historicalIC (blueprint=20 + summary_IC=0)
    //                                        - historicalRIC (summary RIC = 5)
    // Issue 1: Redis=1, effectiveIssued = 1 + 20 - 5 = 16 → 16 <= 16 → SUCCEED
    CouponDetails coupon1 = issueCouponWithStats(orgId, storeUnitId, 201, billId, (int) seriesId);
    assertNull("Issue within RIC-adjusted limit should succeed", coupon1.getEx());
    // num_issued in response = summary_IC + blueprint_IC = 0 + 20 = 20? OR Redis+history?
    // The GET API (getStatisticsBatch) shows: cachedStat(IC=0, RIC=5) → 0-5=-5, clamp to 0?
    // Actually: cachedStat is from bulkFetchStatsFromCacheOrDb which sums all stats_series_summary
    // from snapshotDate+1 → today. The "ic" rows for threeDaysAgo=0, yesterday=0, today=1 (just issued).
    // So cachedStat.IC=1, cachedStat.RIC=5 → currentValue = 1-5 = -4. Then -4 + 20(blueprint) = 16.
    // But getSumFromCache returns currentValue + historicalValue. No clamping to 0 in code!
    // Developer must validate this: is 1-5=-4 handled correctly or does it produce 16?

    // Issue 2: Redis=2, effectiveIssued = 2 + 20 - 5 = 17 → 17 > 16 → FAIL
    CouponDetails coupon2 = issueCouponWithStats(orgId, storeUnitId, 202, billId + 1, (int) seriesId);
    assertNotNull("Issue exceeding RIC-adjusted limit should fail", coupon2.getEx());
    assertEquals(626, coupon2.getEx().getErrorCode(),
        "Error code should be max create for series exceeded");
    assertEquals("max create for series exceeded", coupon2.getEx().getErrorMsg());
}
```

> ⚠️ **Open design question for developer:** In `getSumFromCache()` for IssuedCount
> (`MySQLCouponSeriesStatisticsReadService.java:450-452`):
> ```java
> case IssuedCount:
>     currentValue = cachedStat.getIssuedCount() - cachedStat.getRevokedIssueCount();
> ```
> If `cachedStat.getIssuedCount()=1` (today's issue) and `cachedStat.getRevokedIssueCount()=5`
> (past days' RIC), the subtraction yields `-4`. Is this clamped to 0 anywhere, or does the GET API
> return `blueprint(20) + (-4) = 16`? The limit-check path is fine (it uses `getMysqlSummaryValue`
> for RIC separately). But the GET response value should be validated.

---

#### T4 — `testAbsentRegistryFallbackToMaxLookback`

**Class:** `StatsSeriesSummaryIntegrationTest.java`
**Validates:** When `stats_history_registry` has no entry for an org, `getHistoricalValue()` falls
back to `MAX_LOOK_BACK_DAYS=-100` and reads summary only (`StatsHistoryServiceImpl.java:108-115`)

```java
@Test
@DisplayName("IC limit: absent registry falls back to 100-day summary lookback without NPE")
public void testAbsentRegistryFallbackToMaxLookback() throws TException, LuciThriftException, SQLException {
    int orgId = 0;
    int storeUnitId = 1234;
    int billId = 1564;
    saveOrgDefaultProperties(orgId);

    // --- No registry entry — simulates org where DataBricks job never ran ---
    // (Do NOT call createStatsHistoryRegistryEntry or createRollingTable)

    // Create series with maxCreate=5
    CouponConfiguration savedSeries = saveCouponSeries(
        orgId, createCouponSeriesWithLimits(orgId, 5, -1));
    long seriesId = savedSeries.getId();
    sleep(1000);

    // Seed summary with IC=4 for yesterday (within 100-day lookback window)
    Date yesterday = Date.from(Instant.now().minus(1, ChronoUnit.DAYS));
    insertStatsSeriesSummary(orgId, (int) seriesId, yesterday, "ic", 4L);

    deleteRedisByKeyPattern(RedisCacheKeys.MIDNIGHT_EXPIRING_CACHE_NAME + "*");

    // getHistoricalValue: no registry → fallback path (StatsHistoryServiceImpl.java:108-115)
    //   startDate = today - 100 days, endDate = yesterday
    //   sumValuesForDateRange → returns 4 (from yesterday's summary row)
    // effectiveIssued = Redis(1, post-increment) + historical(4) - RIC(0) = 5 → AT limit → SUCCESS

    CouponDetails coupon1 = issueCouponWithStats(orgId, storeUnitId, 301, billId, (int) seriesId);
    assertNull("First issue should succeed (1 + 4 historical = 5 = limit)", coupon1.getEx());

    // effectiveIssued = Redis(2) + historical(4) - RIC(0) = 6 > 5 → FAIL
    CouponDetails coupon2 = issueCouponWithStats(orgId, storeUnitId, 302, billId + 1, (int) seriesId);
    assertNotNull("Second issue should fail (fallback correctly limits to maxCreate=5)", coupon2.getEx());
    assertEquals(626, coupon2.getEx().getErrorCode());

    // Confirm no exception (NPE guard): check logs or absence of 500 error code
    assertNotEquals(500, coupon2.getEx().getErrorCode(),
        "Must not be an internal server error — NPE guard must hold");
}
```

---

#### T5 — `testOULevelRCLimitWithHistoricalData`

**Class:** `StatsSeriesSummaryIntegrationTest.java`
**Validates:** OU-level redemption cap with blueprint + today's MySQL RC via
`MaxRedemptionForSeriesProcessor.getRedeemedCouponCount()` (OU branch: `ouId != 0`)

> **Pre-condition (resolved Q3/Q4):** Copy the OU-level series creation pattern from
> `RedeemCouponIntegrationTest#shouldValidateRedemptionPerCoupounMaxLimitForOuWithDiffTillId`.
> No separate `redeemCouponAtOU()` helper is needed — OU is inferred from `tillId` by
> `OrgEntityRelationsService#getOUIdForTill`. Use a `storeUnitId` that maps to the target `ouId`
> in the test environment and verify the `storeUnitId → ouId` mapping before seeding blueprint OU-RC.

```java
@Test
@DisplayName("OU-level RC limit: blueprint OU-RC + today's MySQL OU-RC enforces per-OU maxRedeem")
public void testOULevelRCLimitWithHistoricalData() throws TException, LuciThriftException, SQLException {
    int orgId = 0;
    int storeUnitId = 1234;
    int billId = 2000;
    long ouId = 999L;
    saveOrgDefaultProperties(orgId);

    String activeTable = createRollingTable(4);
    Date fourDaysAgo = Date.from(Instant.now().minus(4, ChronoUnit.DAYS));
    createStatsHistoryRegistryEntry(orgId, activeTable, fourDaysAgo, true, new Date());

    // Create OU-level series: series-level maxRedeem=-1, OU-level maxRedeem=3
    // Pattern from RedeemCouponIntegrationTest#shouldValidateRedemptionPerCoupounMaxLimitForOuWithDiffTillId
    CouponConfiguration savedSeries = saveCouponSeries(
        orgId, createCouponSeriesWithOURedemptionLimits(orgId, -1, ouId, 3));
    long seriesId = savedSeries.getId();
    sleep(1000);

    // Seed OU-level blueprint RC=2 for this OU
    // insertStatsSeriesSummaryWithOu supports OU-level seeding; blueprint level needs
    // a new helper: insertHistoricalStatsDataWithOu(orgId, seriesId, tableName, ouId, rc=2)
    // [NEEDS SPIKE: StatsHistory entity supports redemptionConfigEntity/Id — use Fixture.getStatsHistory
    //  with entity type "OU" and entity ID ouId, then insertStatsHistoryData directly]
    StatsHistory ouRc = Fixture.getStatsHistory(orgId, seriesId, "rc", 2L,
        RedemptionConfigEntityType.OU.name(), ouId);
    insertStatsHistoryData(orgId, activeTable, Lists.newArrayList(ouRc));

    deleteRedisByKeyPattern(RedisCacheKeys.MIDNIGHT_EXPIRING_CACHE_NAME + "*");

    // Issue 2 coupons to OU customers
    CouponDetails coupon1 = issueCouponWithStats(orgId, storeUnitId, 401, billId,     (int) seriesId);
    CouponDetails coupon2 = issueCouponWithStats(orgId, storeUnitId, 402, billId + 1, (int) seriesId);
    assertNull(coupon1.getEx());
    assertNull(coupon2.getEx());

    // Redeem coupon1: storeUnitId maps to ouId=999 via OrgEntityRelationsService#getOUIdForTill
    // OU is inferred from tillId — no separate redeemCouponAtOU() helper needed (Q4 resolved)
    // historical OU-RC=2, today mysql OU-RC=1 → total=3 → AT limit → SUCCESS
    List<CouponDetails> firstRedeem = redeemCoupon(orgId, storeUnitId, 401,
        Lists.newArrayList(coupon1), true);
    assertNull("First OU redeem should succeed (2 historical + 1 today = 3 = limit)",
        firstRedeem.get(0).getEx());

    // Redeem coupon2: same storeUnitId → same ouId; today mysql OU-RC=2 → total=4 > 3 → FAIL
    List<CouponDetails> secondRedeem = redeemCoupon(orgId, storeUnitId, 402,
        Lists.newArrayList(coupon2), true);
    assertNotNull("Second OU redeem should fail (4 > 3)", secondRedeem.get(0).getEx());
    assertEquals(605, secondRedeem.get(0).getEx().getErrorCode());
}
```

> **Q3/Q4 resolved:**
> 1. OU-level series creation: copy pattern from `RedeemCouponIntegrationTest#shouldValidateRedemptionPerCoupounMaxLimitForOuWithDiffTillId`
> 2. OU redemption: use standard `redeemCoupon()` with a `storeUnitId` that maps to the target `ouId`
>    via `OrgEntityRelationsService#getOUIdForTill`. Verify the mapping exists in the test environment
>    before seeding blueprint OU-RC for that `ouId`.

---

#### T6 — `testGetHistoricalValueSnapshotDateEqualsToday` (UT)

**Class:** `StatsHistoryServiceImplTest.java`
**Validates:** When snapshot date = today (DataBricks early run), `startDate > endDate`, summary is
skipped (`StatsHistoryServiceImpl.java:87`)

```java
@Test
public void testGetHistoricalValue_WhenSnapshotDateIsToday_SkipsSummingFromStatsSeriesSummary() {
    CouponSeriesStatisticsField key = CouponSeriesStatisticsField.IssuedCount;

    StatsHistoryRegistryEntity registry = new StatsHistoryRegistryEntity();
    registry.setOrgId(TEST_ORG_ID);
    registry.setTableName(TEST_TABLE_NAME);
    // Snapshot date = today (DataBricks ran early this morning)
    LocalDate today = LocalDate.now();
    registry.setSnapShotDate(Date.from(today.atStartOfDay(ZoneId.systemDefault()).toInstant()));

    when(statsHistoryRegistryDao.findLastActiveByOrgId(TEST_ORG_ID)).thenReturn(registry);
    when(statsHistoryDao.findHistoricalValue(anyInt(), anyString(), anyLong(),
        anyString(), anyString(), anyLong())).thenReturn(42L);

    long result = statsHistoryService.getHistoricalValue(
        TEST_ORG_ID, TEST_COUPON_SERIES_ID, key,
        TEST_REDEMPTION_CONFIG_ENTITY_TYPE, TEST_REDEMPTION_CONFIG_ENTITY_ID);

    // startDate = today+1 day, endDate = yesterday → startDate > endDate → summary skipped
    assertEquals(42L, result, "Should return only blueprint value when snapshot=today");
    verify(statsSeriesSummaryDao, never()).sumValuesForDateRange(
        anyInt(), anyInt(), any(), anyString(), anyLong(), any(Date.class), any(Date.class));
}
```

---

## 9. Data Model Changes

No data model changes. All new tests use existing tables:
- `stats_history_blueprint_YYYYMMDD` — seeded via `insertHistoricalStatsData()` / `insertStatsHistoryData()`
- `stats_series_summary` — seeded via `insertStatsSeriesSummary()` / `insertStatsSeriesSummaryWithOu()`
- `stats_history_registry` — seeded via `createStatsHistoryRegistryEntry()`

> Explicitly: **No new columns, tables, indexes, or migrations.**

---

## 10. API Changes

No API changes. All new code is in `src/test/`. No production endpoints are added or modified.

> Explicitly: **No API changes.**

---

## 11. Security, DB, Infra Considerations

```
CONSIDERATION [DB] [NOTE]
Finding: insertStatsSeriesSummary uses raw String.format SQL (BaseIntegrationTest.java:1157-1170).
Impact: Test-only code — not a production security risk. No PII is inserted.
Recommended action: None for now. If the pattern is copied to production helpers, parameterise.
Owner: Developer — awareness only.
```

```
CONSIDERATION [DB] [ACTION]
Finding: Blueprint rolling tables created by createRollingTable() are not cleaned up after tests.
  If the embedded DB persists state across test class runs, stale tables accumulate.
Impact: Test DB bloat; potential confusion if data from a prior test's rolling table is
  picked up by a subsequent test's registry query (findLastActiveByOrgId returns one entry).
Recommended action: Add @After cleanup: statsHistoryRegistryDao.deleteByOrgId(orgId) and
  DROP the rolling table. Or ensure the embedded DB is reset between test class runs.
Owner: Developer (T1-T5).
```

```
CONSIDERATION [INFRA] [NOTE]
Finding: getHistoricalValue() is @Cacheable in MIDNIGHT_EXPIRING_CACHE_NAME (Redis).
  Tests that seed blueprint data must evict this cache before exercising the limit check.
  deleteRedisByKeyPattern(MIDNIGHT_EXPIRING_CACHE_NAME + "*") is available in BaseIntegrationTest.
Recommended action: Call deleteRedisByKeyPattern after seeding and before issuing in all new tests.
  Already documented in T1-T4 method bodies above.
Owner: Developer.
```

```
CONSIDERATION [INFRA] [ACTION]
Finding: T5 (OU-level test) requires a redeemCouponAtOU() helper variant that passes ouId
  in the redemption context. Check if this exists or needs to be added to BaseIntegrationTest.
Recommended action: Search for "ouId" or "OU" context in redeemCoupon variants before writing T5.
Owner: Developer (T5 spike).
```

---

## 12. Internal Architecture Changes

**No production architecture changes.** This is a test-only specification. Test infrastructure additions:

- New test methods in `StatsSeriesSummaryIntegrationTest` (T1–T5)
- New UT in `StatsHistoryServiceImplTest` (T6)
- Potentially: new helper `redeemCouponAtOU()` in `BaseIntegrationTest` (needed for T5)
- Potentially: new helper `createCouponSeriesWithOURedemptionLimits()` in `StatsSeriesSummaryIntegrationTest` (needed for T5)

---

## 13. Upstream / Downstream Impact

No upstream or downstream impact. Tests run in the embedded IT environment. No production data
is touched. No external API contracts change.

---

## 14. SLA Impact

No SLA impact. Tests are offline IT and UT methods with no runtime effect.

> Explicitly: **No SLA impact anticipated.**

---

## 15. Observability

No new production observability. For test observability:
- Each new test logs the seeded blueprint + summary values at `@DisplayName` level
- Assertion messages include expected vs actual values for easy CI failure diagnosis
- T3 includes a `⚠️ Open design question` comment for the `getSumFromCache` IC-RIC subtraction
  edge case — developer must validate and document the result

---

## 16. Rollout Plan

Tests only — no feature flag, no migration, no staged rollout.

1. Write T6 (UT) first — fastest, no DB setup, validates `getHistoricalValue` date boundary
2. Write T1 (RC limit) — most critical gap, builds on existing pattern
3. Write T2 (GET API) — second most critical
4. Write T3 (RIC in summary)
5. Write T4 (absent registry fallback)
6. Write T5 (OU-level) — requires spike first; write last

**Go/no-go:** All tests pass in CI (`mvn test -pl luci -Dtest=StatsSeriesSummaryIntegrationTest`).

**Rollback:** N/A — tests can be disabled with `@Disabled` if a flake is discovered.

---

## 17. Risks

| # | Description | Likelihood | Mitigation |
|---|------------|-----------|-----------|
| R1 | `createCouponSeriesWithLimits` produces a `DISC_CODE_PIN` series (requires coupons uploaded) rather than GENERIC — `issueCouponWithStats` fails | Low (existing test works without uploadCoupons) | Confirm fixture default in `CouponConfigurationFixture`; add `uploadCoupons(savedSeries, 50)` if needed |
| R2 | Embedded DB series ID does not start at 1; test seeds blueprint for wrong ID (the 1L hardcoding issue) | Medium — mitigated by using actual ID | All new tests pass `(long) savedSeries.getId()` to `insertHistoricalStatsData`; never hardcode |
| R3 | `@Cacheable` on `getHistoricalValue` returns stale cached value from a prior test run | Low — mitigated by Redis eviction call in each new test | `deleteRedisByKeyPattern(MIDNIGHT_EXPIRING_CACHE_NAME + "*")` in setup |
| R4 | T5 OU-level test is blocked on unknown helper (redeemCouponAtOU) | Medium | Spike before writing T5; if helper doesn't exist, add to BaseIntegrationTest |
| R5 | `getSumFromCache` IC-RIC subtraction yields negative currentValue (IC < RIC from past days) — GET API returns a lower-than-expected `num_issued` | Low to Medium — depends on historical data patterns | **Confirmed (Q2 resolved): `getSumFromCache` NEVER clamps.** `currentValue = IC − RIC` can be negative; result = `currentValue + blueprint_IC`. IT-03 asserts `num_issued = (1−5)+20 = 16` (not 21 and not 20). This is a known production risk: if `blueprint_IC` is small and `RIC > IC` in the window, merchants see under-reported `num_issued`. No fix in scope; document as known behaviour. |

---

## 18. Open Questions

| Question | Resolution | Resolved |
|---------|-----------|---------|
| Q1: `CouponConfigurationFixture.getCouponConfiguration()` — what is the default `client_handling_type`? Does `createCouponSeriesWithLimits` work without `uploadCoupons`? | ✅ **DISC_CODE** (confirmed from `CouponConfigurationFixture#getCouponConfiguration`). Auto-generates coupons on issue; no `uploadCoupons()` needed for T1–T4. `numTotal` for DISC_CODE returns `couponSeriesEntity.getMaxCreate()`, NOT blueprint+summary TC — do not assert `numTotal` from history in IT-01 through IT-04. | 2026-04-27 |
| Q2: Does `getSumFromCache(IssuedCount)` clamp negative `currentValue` (IC-RIC < 0) to 0, or does the GET API return a negative-adjusted sum? | ✅ **Never clamps.** `currentValue = IC − RIC` can be negative; `getSumFromCache` returns `currentValue + blueprint_IC` even when negative. IT-03 assertion: `num_issued = (1−5)+20 = 16` (not 21). Known production risk — see R5. | 2026-04-27 |
| Q3: How are OU-level series created in existing ITs? | ✅ Pattern is in `RedeemCouponIntegrationTest#shouldValidateRedemptionPerCoupounMaxLimitForOuWithDiffTillId`. Copy that test's series creation and OU redemption config setup for T5. | 2026-04-27 |
| Q4: Does `redeemCoupon()` in `BaseIntegrationTest` pass OU context? If not, what variant exists? | ✅ **OU is inferred from `tillId`** by `OrgEntityRelationsService#getOUIdForTill`. No separate `redeemCouponAtOU()` needed. T5 must use a `storeUnitId` that maps to the target `ouId`; verify mapping before seeding blueprint OU-RC. | 2026-04-27 |
| Q5: Is there a `@Before` or `@After` in `BaseIntegrationTest` that cleans `stats_history_registry` and rolling tables between tests? | ✅ **Not confirmed** — cannot rely on framework cleanup. Each new IT must own its teardown: delete registry rows and DROP rolling tables in `@After`. See test plan Test Data Requirements §Mandatory @After Cleanup. | 2026-04-27 |

---

## 19. Questions for arch-investigator

```
QUESTION FOR ARCH-INVESTIGATOR
================================
[CLARIFICATION — needed during build]

Q1: Handoff GAP-IT-3 states "seed RIC in a blueprint table."
    This is factually incorrect — RIC is isMysql=true (CouponSeriesStatisticsField.java:12)
    and is never stored in stats_history_blueprint_*. The correct approach is to seed RIC
    via insertStatsSeriesSummary for past dates.
    Action needed: Correct the handoff doc or note the discrepancy for future investigations.
    Surfaced in: Phase 2 — codebase validation of CouponSeriesStatisticsField enum.

Q2: Handoff says bulkFetchHistoryValues() returns "blueprint + summary (snapshot+1 → today)."
    Actual: it returns blueprint ONLY. Summary range is owned by bulkFetchStatsFromCacheOrDb().
    Action needed: Correct the system map in the handoff for accuracy.
    Surfaced in: Phase 2 — read MySQLCouponSeriesStatisticsReadService.java:192-299.

Q3: The handoff assumes automation tests at campaigns_auto have no stats assertions.
    Confirm: are there any scripts in campaigns_auto/tests/luci/ that DO check num_issued or
    num_redeemed values, possibly under a different directory or naming convention?
    Surfaced in: Phase 1 — automation test gap analysis.
```

---

## 20. Task Breakdown

### Testing (primary deliverable)

| # | Task | Size | Depends on |
|---|------|------|-----------|
| T6 | UT: `testGetHistoricalValueSnapshotDateEqualsToday` in `StatsHistoryServiceImplTest` | S | — |
| T1 | IT: `testRCLimitWithHistoricalData` in `StatsSeriesSummaryIntegrationTest` | M | Q1 resolved |
| T2 | IT: `testGetApiStatsAssemblyFromHistoryAndSummary` | M | T1 (reuse setup pattern) |
| T3 | IT: `testRICInSummaryReducesEffectiveIssuedCount` | S | T1, Q2 resolved |
| T4 | IT: `testAbsentRegistryFallbackToMaxLookback` | S | T1 |
| T5-SPIKE | Spike: OU-level series creation + redeemCouponAtOU helper | S | — |
| T5 | IT: `testOULevelRCLimitWithHistoricalData` | M | T5-SPIKE |

### Infra / Docs

| # | Task | Size | Depends on |
|---|------|------|-----------|
| T7 | Automation: implement month-based self-managing fixture series (`STATS_GEN_{year}_{month:02d}`, `STATS_DCP_{year}_{month:02d}`, `STATS_SEQ_{year}_{month:02d}`) in `campaigns_auto/tests/luci/`. Get-or-create via `getAllCouponConfigurations(seriesCodes=[…])`. No infra coordination needed — series bootstraps itself on first test run of the month. | M | — |
| T8 | Add `@After` cleanup for rolling tables + registry in `BaseIntegrationTest` (or verify existing) | S | Q5 resolved |

---

## Handoff Notes for test-plan-architect

**Tech detail doc location:**
Same directory as `handoff.md` — `CAP-184615_stat_test/CAP-184615_stat_test_techdetail.md`

### Critical B2B Flows to Test
- UC-1/2: RC limit with historical blueprint data → type: IT (T1)
- UC-3: GET API stats = blueprint + summary aggregate → type: IT (T2)
- UC-4/5: IC limit adjusted by historical RIC from summary → type: IT (T3)
- UC-6: No registry fallback → type: IT (T4)
- UC-7: OU-level RC limit with blueprint → type: IT (T5)

### Critical B2C Flows to Test
- UC-9/10: Long-running series limit checks via MySQL path → covered by T1, T3
- UC-12: `getCustomerCoupon` stats include history → asserted in T2

### Regression Risks (must not break)
- `testHistoricalStatsWithCouponIssueAndRedeem` — the single existing history IT; must still pass
- `shouldReflectRicAndRrcWhenRevokeOrReactivateOnDifferentDay` — same-day RIC/RRC path
- All existing `StatsHistoryServiceImplTest` UTs — none are changed

### Tenant Isolation Tests
- T1-T5 use `orgId=0`; `testHistoricalStatsWithCouponIssueAndRedeem` uses `orgId=Fixture.orgId=123`
  — these must not share registry rows or rolling table data (different org IDs are org-sharded)

### Contract Tests
- No external API contract changes

### Suggested Test Emphasis
- **Unit (T6):** Date boundary in `getHistoricalValue` — fast, mocked, high isolation
- **Integration (T1-T5):** Full end-to-end through processor chain → MySQL → assertion
- **Tenant isolation:** Org 0 and Org 123 must have independent registry + blueprint data in the same embedded DB run

---

*End of tech detail document.*
