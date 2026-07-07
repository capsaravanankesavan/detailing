# Investigation: DataBricks Stats Snapshot — Mutable Timestamp Cutoff Miscompute
**Date:** 2026-06-11
**Investigator:** Saravanan Kesavan
**Status:** Ready for Tech Detail
**Confidence:** HIGH — all findings confirmed against notebook source and table schemas

---

## Problem Statement

The DataBricks stats notebook (`luci_calculate_stats_incrm.py`) computes point-in-time snapshot
counts for `issuedCount` (IC), `redeemedCount` (RC), `uploadedCount` (UC), and
`uploadedTotalCount` (UTC) using **mutable last-modification timestamps**
(`auto_update_time` on `coupons_issued`/`coupon_redemptions`, `last_updated_on` on
`coupons_created`) as the snapshot cutoff filter instead of the **immutable original event
timestamps** (`issued_date`, `redeemed_date`, `created_on`).

As a result, any post-event state change — metadata updates (mergeUser, bill number correction,
resend) OR business operations (revoke, invalidate, reactivate) — that refreshes a mutable
timestamp or flag past the DataBricks cutoff causes the affected record to **fall out of the
snapshot's count and be misclassified**. Depending on the field and operation:

| Operation | Table updated | Field affected | Direction |
|---|---|---|---|
| mergeUser / bill update | `coupons_issued.auto_update_time` pushed past cutoff | IC under, UC over | Does not self-correct |
| `revokeCoupons` | `coupons_issued.active=FALSE` + `auto_update_time` updated | IC under, UC over (DISC_CODE_PIN) | Does not self-correct |
| `invalidateCoupons` (issued) | `coupons_issued.active=FALSE` | IC under | Does not self-correct |
| `invalidateCoupons` (unissued, DISC_CODE_PIN) | `coupons_created.is_valid=FALSE` + `last_updated_on` updated | UC under, UTC under | Does not self-correct |
| `reactivateCoupon` | `coupon_redemptions.active=FALSE` + `auto_update_time` updated | RC under (`num_redeemed` negative) | Does not self-correct |
| `is_queued` update (upload queue) | `coupons_created.last_updated_on` pushed past cutoff | UC under | Does not self-correct |

This miscompute is silent — no alert or error. The **IC/UC/RC miscomputes do NOT self-correct**
across DataBricks runs once the snapshotDate advances past the original event date (the event's
summary row falls into the blueprint zone; the wrong blueprint is the only source of truth for
that date).

> **Timezone handling — NOT a bug (known intentional design):** The UTC conversion in the notebook
> is intentionally disabled (`luci_calculate_stats_incrm.py:39`). The DB stores `auto_update_time`
> epoch millis using the JVM's server-local clock (e.g., CST/IST) treated as if UTC. DataBricks
> reads those epochs via `TIMESTAMP_MILLIS()` which treats them as UTC — consistent with how they
> were written. The cutoff is also passed in server-local time without UTC conversion, keeping both
> sides of every comparison in the same frame of reference. The `Z` suffix in the SQL is misleading
> but correct for this convention. ETL team has documented this in
> `databricks/stats/luci_timezone_descrepency.txt` and confirmed correction will happen in Fact
> computation. Java `ParquetFileService` and `StatsHistoryServiceImpl` consume the resulting
> `max_auto_update_time_issued` epoch consistently. **This issue is out of scope for this fix.**

---

## Root Cause

DataBricks uses mutable state fields (`auto_update_time`, `active`, `is_valid`, `last_updated_on`)
as point-in-time snapshot filters, but reads the **current** state of those fields when the ETL runs
2 days later. Any business operation (revoke, invalidate, reactivate) or metadata update that changes
these fields after the snapshot date but before the ETL run causes the affected record to be
mis-counted. The Java formula `(IC_window − RIC_window) + blueprint_IC` produces **negative
`num_issued`** when `blueprint_IC = 0` (record excluded by `active=FALSE`) but `RIC_window = 1`
(revocation captured in Java summary).

### Contributing Factors

1. **`coupons_issued.auto_update_time` is refreshed by mergeUser, bill number updates, and resend
   operations** — common operational flows that run after issuance. There is no safeguard preventing
   these from pushing `auto_update_time` past the DataBricks snapshot cutoff.

2. **The `unissued` anti-join uses `auto_update_time` on the right side** — so a coupon whose
   `auto_update_time > cutoff` is invisible in the join, making `ci.coupon_code IS NULL` true,
   classifying it as unissued even though it IS issued.

3. **`coupons_created.last_updated_on` is an `ON UPDATE CURRENT_TIMESTAMP` column AND is
   explicitly re-written during queue operations** — `CouponsCreatedDaoImpl.java:244` explicitly
   sets `paramMap.put(LAST_UPDATED_ON, new Date())` when a coupon is placed into the issuance
   queue (`is_queued=true`). Subsequently, the dequeue UPDATE (`updateIsQueuedStatusForUnissuedCoupons`)
   triggers `ON UPDATE CURRENT_TIMESTAMP` again. A coupon uploaded Jun 4 and issued Jun 9 will
   have `last_updated_on = Jun 9` at the time DataBricks runs on Jun 10. With `cutoff = Jun 8`,
   the coupon is EXCLUDED from `unissued`, producing `Blueprint_UC = 0` instead of 1. The Java
   summary already wrote `uc = -1` on Jun 9 for the issuance. Net: `num_uploaded_nonIssued = 0 + (−1) = −1`.

4. **`revokeCoupons` sets `coupons_issued.active=FALSE`** — `CouponDaoImpl.revokeCoupon:709`
   sets `active=false` and triggers `ON UPDATE CURRENT_TIMESTAMP` on `auto_update_time`.
   DataBricks `issued` CTE uses `active=TRUE`, so a coupon revoked AFTER the snapshot date is
   excluded from `blueprint_IC`. Java writes `+ric` to `stats_series_summary` (NOT `−ic` for the
   MySQL path — confirmed `updateStatsPostRevoke:2820`). Formula: `(0 − 1) + 0 = −1`. **`num_issued`
   goes negative per revoked-after-snapshot coupon.**

5. **`invalidateCoupons` sets `coupons_created.is_valid=FALSE` + updates `last_updated_on`** —
   DataBricks `created`/`unissued` CTEs filter `is_valid=1 AND last_updated_on <= cutoff`. Both
   filters exclude an invalidated coupon even if it was valid on the snapshot date. Java writes
   `−uc` and `−utc` to `stats_series_summary`. Formula: `0 + (−1) = −1`. **`num_uploaded_nonIssued`
   and `num_uploaded_total` go negative per invalidated-after-snapshot coupon (DISC_CODE_PIN).**

6. **`reactivateCoupon` sets `coupon_redemptions.active=FALSE`** — `CouponRedemptionDaoImpl
   .invalidateRedemption:491` sets `active=0`. DataBricks `redeemed` CTE uses `active=TRUE`.
   Java writes `+rrc` to `stats_series_summary` via `incrementReactivatedRedemptionCountInDb`.
   Formula: `(0 − 1) + 0 = −1`. **`num_redeemed` goes negative per reactivated-after-snapshot
   redemption.**

7. **The Java counter-diff checker already has the correct compensation** (`CouponDaoImpl.java:
   1322–1328`, `CouponRedemptionDaoImpl.java:861–867`): if `active=FALSE` but `auto_update_time
   >= cutoff`, count it (meaning it was active at snapshot time). DataBricks is missing this logic.

8. **UTC conversion disabled** (`luci_calculate_stats_incrm.py:39`) — intentional design, NOT a
   bug. Out of scope. See timezone note above.

### Why It Surfaces Under These Conditions

Any series where, between the snapshot date and the DataBricks ETL run date (2-day gap), any of
the following occurs for a coupon/redemption with event date ≤ snapshot date:
- Metadata update: mergeUser, bill number correction, resend → `auto_update_time` pushed past cutoff
- `revokeCoupons` → `coupons_issued.active=FALSE`
- `invalidateCoupons` → `coupons_issued.active=FALSE` and/or `coupons_created.is_valid=FALSE`
- `reactivateCoupon` → `coupon_redemptions.active=FALSE`
- `is_queued` state change → `coupons_created.last_updated_on` pushed past cutoff

High-traffic series with frequent revocations, invalidations, or reactivations are at greater risk.
Fresh series with no post-event operations are unaffected.

---

## Evidence Trail

| File | Line | Finding |
|------|------|---------|
| `databricks/stats/luci_calculate_stats_incrm.py` | 63 | `issued` CTE: `AND TIMESTAMP_MILLIS(auto_update_time) <= TIMESTAMP(cutoff)` — mutable field used as point-in-time filter |
| `databricks/stats/luci_calculate_stats_incrm.py` | 89 | `redeemed` CTE: same pattern — `auto_update_time` cutoff on `coupon_redemptions` |
| `databricks/stats/luci_calculate_stats_incrm.py` | 108 | `created` CTE: `TIMESTAMP_MILLIS(last_updated_on) <= cutoff` — `ON UPDATE` column used as creation filter |
| `databricks/stats/luci_calculate_stats_incrm.py` | 124, 126 | `unissued` anti-join: both sides filtered by mutable timestamps — issued coupon with `auto_update_time > cutoff` makes `ci.coupon_code IS NULL` true → misclassified as unissued |
| `databricks/stats/luci_timezone_descrepency.txt` | — | Timezone convention documented — intentional: DB stores server-local epoch as-if-UTC; DataBricks and Java both consume it in the same frame of reference; UTC conversion correctly disabled |
| `src/main/java/com/capillary/luci/data/entity/CouponEntity.java` | 26 | `issuedDate` field confirmed — immutable original issue timestamp available |
| `src/test/resources/cc-stack-crm/schema/dbmaster/luci/coupons_created.sql` | 9–10 | `created_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP` (immutable), `last_updated_on ON UPDATE CURRENT_TIMESTAMP` (mutable) — fix column exists |
| `src/main/java/com/capillary/luci/data/dao/impl/CouponsCreatedDaoImpl.java` | 244 | `paramMap.put(LAST_UPDATED_ON, new Date())` — explicit Java write sets `last_updated_on = now()` when coupon is queued for issuance (`is_queued=true`). This is the primary real-world trigger that pushes `last_updated_on` past the DataBricks cutoff for coupons issued after the snapshot date. |
| `src/main/java/com/capillary/luci/data/dao/impl/CouponsCreatedDaoImpl.java` | 494–519 | `updateIsQueuedStatusForUnissuedCoupons` — UPDATE on `is_queued=0` triggers `ON UPDATE CURRENT_TIMESTAMP` on `last_updated_on` even without an explicit set. Second touch on the same day as issuance. |
| `src/main/java/com/capillary/luci/impl/statistics/impl/StatsHistoryServiceImpl.java` | 80 | `startDate = DateUtil.addDaysToDate(snapshotDate, 1)` — summary window begins day AFTER snapshotDate; once snapshotDate advances past event date, that event's summary row is in the blueprint zone |
| `src/main/java/com/capillary/luci/impl/ParquetFileService.java` | 383 | `saveRegistry(orgId, rollingTable, new Date(snapShotDate))` — snapshotDate = `max_auto_update_time_issued` from parquet |
| `src/main/java/com/capillary/luci/data/dao/impl/CouponDaoImpl.java` | 697–716 | `revokeCoupon()` — `UPDATE coupons_issued SET active=false ...` triggers `ON UPDATE CURRENT_TIMESTAMP` on `auto_update_time`. DataBricks `issued` CTE (`active=TRUE`) will exclude this coupon from blueprint_IC after the revoke. |
| `src/main/java/com/capillary/luci/impl/CouponRuntimeServiceImpl.java` | 2819–2820 | `updateStatsPostRevoke()` — MySQL path writes `+ric` to `stats_series_summary` (NOT `−ic`). Formula `(IC_window − RIC_window) + blueprint_IC = (0 − 1) + 0 = −1` when blueprint excludes revoked coupon. |
| `src/main/java/com/capillary/luci/impl/CouponRuntimeServiceImpl.java` | 1267–1272 | `updateCouponSeriesStats()` (invalidateCoupons) — same RIC write pattern as revoke; for DISC_CODE_PIN also calls `decrementUploadedCount` and `incrementUploadedTotalCount(−N)`. |
| `src/main/java/com/capillary/luci/data/dao/impl/CouponRedemptionDaoImpl.java` | 488–497 | `invalidateRedemption()` (called by reactivateCoupon) — `UPDATE coupon_redemptions SET active=0 ...` triggers `auto_update_time` update. DataBricks `redeemed` CTE (`active=TRUE`) excludes this redemption from blueprint_RC. |
| `src/main/java/com/capillary/luci/impl/CouponRuntimeServiceImpl.java` | — | `reactivateCoupon` processor chain calls `incrementReactivatedRedemptionCountInDb` → `+rrc` in `stats_series_summary`. Formula `(0 − 1) + 0 = −1` for `num_redeemed`. |
| `src/main/java/com/capillary/luci/data/dao/impl/CouponDaoImpl.java` | 1322–1328 | **Compensation already in Java diff-checker:** `if (active=FALSE AND auto_update_time >= cutoff) → count it`. DataBricks notebook is missing this exact logic. |
| `src/main/java/com/capillary/luci/data/dao/impl/CouponRedemptionDaoImpl.java` | 861–867 | Same compensation in redemption diff-checker. Same logic needed in DataBricks `redeemed` CTE. |

---

## System Map (Affected Path)

**Data flow:**
```
coupons_issued / coupon_redemptions / coupons_created
    ↓ (DataBricks reads with auto_update_time / last_updated_on cutoff)
luci_calculate_stats_incrm.py → S3 parquet
    ↓ (StatsHistoryRunner triggers)
ParquetFileService.readParquetFileAndBulkInsertInToDb()
    ↓
stats_history_blueprint_YYYYMMDD + stats_history_registry (snapshotDate = max_auto_update_time)
    ↓
StatsHistoryServiceImpl.getHistoricalValue() / bulkGetHistoricalValues()
    ↓ (blueprint_IC + summary_window_IC → getSumFromCache)
MySQLCouponSeriesStatisticsReadService → CouponSeriesStatisticsDetails
    ↓
LuciThriftAdapter → CouponConfiguration.num_issued / num_uploaded_nonIssued
```

**Tenant isolation:** `org_id` is correctly threaded through all DataBricks CTEs and Java paths. No cross-tenant bleed risk from this bug.

**Upstream:** `coupons_issued`, `coupon_redemptions`, `coupons_created` DB tables (written by Java at event time)

**Downstream:** `stats_history_blueprint_*` rolling tables, `stats_history_registry` → Java GET API stats assembly → `CouponConfiguration` response fields `num_issued`, `num_redeemed`, `num_uploaded_nonIssued`, `num_uploaded_total`, `numTotal`

---

## Miscompute Scenarios

### Scenario A: Issued coupon with post-issue metadata update

| Step | Event | IC impact | UC impact |
|------|-------|-----------|-----------|
| Jun 5 | Coupon issued; `auto_update_time=Jun 5` | +1 in Jun 5 summary | −1 in Jun 5 summary (DISC_CODE_PIN) |
| Jun 7 | MergeUser/bill update; `auto_update_time=Jun 7` | — | — |
| Jun 8 ETL (cutoff=Jun 6) | `auto_update_time=Jun 7 > Jun 6` → excluded from `issued` CTE; ci IS NULL in anti-join | Blueprint IC: −1 (coupon missing) | Blueprint UC: +1 (coupon counted as unissued) |
| Java after snapshotDate=Jun 6 | Jun 5 summary row in blueprint zone (not in window Jun 7+) | Jun 5 summary −1 correction not in window; blueprint −1 → **API: IC under by 1 permanently** | Jun 5 summary −1 correction not in window; blueprint +1 → **API: UC over by 1 permanently** |

### Scenario B: Uploaded coupon with is_valid update

| Step | Event | UC impact | UTC impact |
|------|-------|-----------|-----------|
| Jun 4 | Coupon uploaded; `created_on=Jun 4`, `last_updated_on=Jun 4` | +1 in Jun 4 summary | +1 in Jun 4 summary |
| Jun 9 | `is_valid` changed; `last_updated_on=Jun 9` | — | — |
| Jun 10 ETL (cutoff=Jun 8) | `last_updated_on=Jun 9 > Jun 8` → excluded from `created` and `unissued` CTEs | Blueprint UC: −1 (coupon missing) | Blueprint UTC: −1 |
| Next ETL (cutoff≥Jun 9) | `last_updated_on=Jun 9 ≤ new cutoff` → included again | Self-corrects | Self-corrects |

### Scenario C: Coupon revoked after snapshot date

| Step | Event | IC impact | UC impact (DISC_CODE_PIN) |
|------|-------|-----------|--------------------------|
| Jun 4 | Coupon issued; active=TRUE | +1 in Jun 4 summary | −1 in Jun 4 summary |
| Jun 9 | `revokeCoupons` → `active=FALSE`, `auto_update_time=Jun 9` | — | — |
| Jun 10 ETL (cutoff=Jun 8) | `active=FALSE` → EXCLUDED from `issued` CTE | `blueprint_IC = 0` (wrong, should be 1) | `blueprint_UC = 0` (coupon no longer in unissued either — correctly 0) |
| Java formula | Summary window Jun 9+: `ric=+1` (from `updateStatsPostRevoke`) | `(IC=0 − RIC=1) + blueprint_IC(0) = −1` ❌ | For UC: `coupons_created.is_valid` set FALSE → same exclusion; `uc=−1` in summary → `0+(−1)=−1` ❌ |

**Note:** Java writes `+ric` (NOT `−ic`) for MySQL path on revoke. Blueprint must include the coupon for the formula to work. Removing `active=TRUE` filter and adding `OR auto_update_time > cutoff` compensation fixes this.

### Scenario D: Redemption reactivated after snapshot date

| Step | Event | RC impact |
|------|-------|-----------|
| Jun 4 | Redemption; `active=TRUE` on `coupon_redemptions` | +1 in Jun 4 RC summary |
| Jun 9 | `reactivateCoupon` → `invalidateRedemption` → `active=0`, `auto_update_time=Jun 9` | — |
| Jun 10 ETL (cutoff=Jun 8) | `active=FALSE` → EXCLUDED from `redeemed` CTE | `blueprint_RC = 0` (wrong, should be 1) |
| Java formula | Summary window Jun 9+: `rrc=+1` (from `incrementReactivatedRedemptionCountInDb`) | `(RC=0 − RRC=1) + blueprint_RC(0) = −1` ❌ `num_redeemed` goes negative |

### Scenario E: Unissued DISC_CODE_PIN coupon invalidated after snapshot date

| Step | Event | UC impact | UTC impact |
|------|-------|-----------|------------|
| Jun 4 | Coupon uploaded; `is_valid=1`, `created_on=Jun 4` | +1 in Jun 4 UC summary | +1 in Jun 4 UTC summary |
| Jun 9 | `invalidateCoupons` → `is_valid=0`, `last_updated_on=Jun 9` | Java: `uc=−1` in Jun 9 summary | Java: `utc=−1` in Jun 9 summary |
| Jun 10 ETL (cutoff=Jun 8) | `is_valid=0` → EXCLUDED from `created` and `unissued` CTEs (both `is_valid=1` and `last_updated_on<=cutoff` filters fail) | `blueprint_UC = 0` (wrong, should be 1) | `blueprint_UTC = 0` (wrong, should be 1) |
| Java formula | `0 + (−1) = −1` for both UC and UTC ❌ | | |

### Timezone convention (not a bug — intentional design)

DB stores `auto_update_time` as server-local-time epoch treated as UTC. DataBricks reads and
compares in the same frame. Cutoff is server-local end-of-day without UTC conversion. Both sides
of every temporal comparison are in the same frame of reference. Documented in
`databricks/stats/luci_timezone_descrepency.txt`. ETL team will correct in Fact computation.
**No miscompute from timezone. Out of scope for this fix.**

---

## Solution Chosen

> **Important constraint:** `coupons_issued` and `coupons_created` have many millions of rows.
> No `ALTER TABLE` schema changes on these tables. All options below satisfy this constraint —
> they are DataBricks-only or DataBricks + small table changes.

**Three options ranked by scope and timeline.** Option 3 (Immutable Event Log) is the recommended
long-term solution that fixes all bugs including FR-014. Option 1 (Time Travel) or Option 2 (SQL
filter fix) are DataBricks-only interim fixes if Option 3 is not yet ready. Phase 0 (`created_on`
swap in DataBricks) is an independent quick fix for upload-only orgs (e.g., Comcast).

---

### Option 1 — Delta Lake Time Travel *(Recommended if CDC confirmed)*

**How it works:** Delta Lake natively supports querying a table's exact state at a past timestamp
via `TIMESTAMP AS OF`. Querying `source_delta.luci__coupons_issued TIMESTAMP AS OF cutoff` returns
the row state as it existed at that moment — `active=TRUE` was accurate then, `auto_update_time`
was pre-revoke, `is_valid` was pre-invalidation. **No compensation logic needed at all.**

```sql
-- issued CTE (line 61) — add TIMESTAMP AS OF:
FROM source_delta.luci__coupons_issued TIMESTAMP AS OF '{cutoff}'
WHERE active = TRUE          -- correct at that point in time

-- redeemed CTE (line 87):
FROM source_delta.luci__coupon_redemptions TIMESTAMP AS OF '{cutoff}'
WHERE active = TRUE

-- created CTE (line 107):
FROM source_delta.luci__coupons_created TIMESTAMP AS OF '{cutoff}'
WHERE CAST(is_valid AS INT) = 1

-- unissued left side (line 119) and right side join (line 120):
FROM source_delta.luci__coupons_created TIMESTAMP AS OF '{cutoff}' cc
LEFT JOIN source_delta.luci__coupons_issued TIMESTAMP AS OF '{cutoff}' ci
```

**What changes:** `databricks/stats/luci_calculate_stats_incrm.py` — 4 table references get
`TIMESTAMP AS OF '{cutoff_end_utc.strftime(...)}'` appended.

**Schema changes:** ZERO on any production MySQL table.
**Java changes:** ZERO.
**Effort:** S — 4 line additions once CDC is confirmed.
**Rollback:** trivially easy — remove the 4 `TIMESTAMP AS OF` clauses.

**Critical prerequisite (Q6):** `source_delta` tables must be **CDC/incremental merge** ingested,
NOT full daily overwrite. If full-refresh, Delta history contains only one snapshot and time travel
is a no-op. Delta `logRetentionDuration` must be ≥ 3 days (default is 30 days — likely fine for
`run_days_offset=2`).

---

### Option 2 — SQL filter fix with immutable columns *(Safe fallback — always available)*

Uses existing immutable columns already confirmed in Delta Lake:
- `issued_date` — **already in `source_delta.luci__coupons_issued`** (notebook line 60) ✓
- `redeemed_date` — **already in `source_delta.luci__coupon_redemptions`** (notebook line 80) ✓
- `created_on` — exists in production `coupons_created` MySQL table but **not yet in Delta Lake**
  → requires ETL team to add to `source_delta.luci__coupons_created` sync config (not a MySQL schema change)

**What changes:** `databricks/stats/luci_calculate_stats_incrm.py` — 6 filter conditions:

**Bug 1a (IC)** — `issued` CTE (line 63):
```sql
-- Before: active = TRUE AND auto_update_time <= cutoff
-- After (mirrors CouponDaoImpl.java:1322-1328):
WHERE TIMESTAMP_MILLIS(issued_date) <= TIMESTAMP(cutoff)
  AND (active = TRUE OR TIMESTAMP_MILLIS(auto_update_time) > TIMESTAMP(cutoff))
```

**Bug 1b (RC)** — `redeemed` CTE (line 89):
```sql
-- After (mirrors CouponRedemptionDaoImpl.java:861-867):
WHERE TIMESTAMP_MILLIS(redeemed_date) <= TIMESTAMP(cutoff)
  AND (active = TRUE OR TIMESTAMP_MILLIS(auto_update_time) > TIMESTAMP(cutoff))
```

**Bug 1c (UC anti-join)** — `unissued` CTE right side (line 124):
```sql
AND TIMESTAMP_MILLIS(ci.issued_date) <= TIMESTAMP(cutoff)
AND (ci.active = TRUE OR TIMESTAMP_MILLIS(ci.auto_update_time) > TIMESTAMP(cutoff))
```

**Bug 2 (UC/UTC)** — `created` CTE (line 108) and `unissued` left side (line 126):
```sql
-- After (requires created_on in Delta Lake):
WHERE TIMESTAMP_MILLIS(created_on) <= TIMESTAMP(cutoff)
  AND (CAST(is_valid AS INT) = 1 OR TIMESTAMP_MILLIS(last_updated_on) > TIMESTAMP(cutoff))
```

**Interim fix for Bug 2 if `created_on` is NOT yet in Delta Lake:**
```sql
-- Partial fix — may slightly over-count UC for series with very recent uploads,
-- but strictly better than current state (negative counts):
WHERE (CAST(cc.is_valid AS INT) = 1
       OR TIMESTAMP_MILLIS(cc.last_updated_on) > TIMESTAMP(cutoff))
```

**Schema changes:** ZERO on production MySQL. One Delta sync config change (`created_on`).
**Java changes:** ZERO.
**Effort:** S (DataBricks SQL) + ETL team coordination for `created_on`.
**Rollback:** revert 6 filter conditions.

---

### Option 3 — Immutable Event Log *(Long-term recommended — fixes all bugs including FR-014)*

**Core idea:** Create a new small table `coupon_event_log` with one immutable row per business
event (upload, issue, redemption, revoke, invalidate, reactivate). DataBricks reads this table
instead of querying mutable state from `coupons_issued` / `coupons_created`. Counter formulas
become pure event-time arithmetic — no `active` flag, no mutable timestamps, no paradigm mismatch.

**New table (small — no ALTER on any large table):**
```sql
CREATE TABLE coupon_event_log (
  org_id      INT NOT NULL,
  series_id   INT NOT NULL,
  coupon_code VARCHAR(200) NOT NULL,
  event_type  ENUM('UPLOADED','ISSUED','REDEEMED',
                   'REVOKED','INVALIDATED_UNISSUED','REACTIVATED') NOT NULL,
  event_time  TIMESTAMP NOT NULL,
  KEY idx_snapshot_query (org_id, series_id, event_type, event_time)
) PARTITION BY RANGE (UNIX_TIMESTAMP(event_time)) ( ... )
```

**DataBricks counter formulas (pure event arithmetic — from event_log):**
```sql
IC  = COUNT(*) FILTER (event_type='ISSUED'               AND event_time <= cutoff)
    - COUNT(*) FILTER (event_type='REVOKED'              AND event_time <= cutoff)

RC  = COUNT(*) FILTER (event_type='REDEEMED'             AND event_time <= cutoff)
    - COUNT(*) FILTER (event_type='REACTIVATED'          AND event_time <= cutoff)

UTC = COUNT(*) FILTER (event_type='UPLOADED'             AND event_time <= cutoff)
    - COUNT(*) FILTER (event_type='REVOKED'              AND event_time <= cutoff)
    - COUNT(*) FILTER (event_type='INVALIDATED_UNISSUED' AND event_time <= cutoff)

UC  = COUNT(*) FILTER (event_type='UPLOADED'             AND event_time <= cutoff)
    - COUNT(*) FILTER (event_type='ISSUED'               AND event_time <= cutoff)
    - COUNT(*) FILTER (event_type='INVALIDATED_UNISSUED' AND event_time <= cutoff)
```

**FR-014 fix built in:** USER_ID uploads write BOTH `UPLOADED` + `ISSUED` events simultaneously.
DataBricks counts `UPLOADED` events → UTC now includes USER_ID codes. The 705-code gap on series
811192 resolves automatically without any branching change.

**Java write points (6):**
- `notifyCouponsUploadRequest`: write `UPLOADED` event (all import types including USER_ID)
- `notifyCouponsUploadRequest` USER_ID branch: additionally write `ISSUED` event
- `revokeCoupons`: write `REVOKED` event
- `invalidateCoupons` on issued coupon: write `REVOKED` event (sets active=FALSE on issued row)
- `invalidateCoupons` on unissued DISC_CODE_PIN coupon: write `INVALIDATED_UNISSUED` event
- `reactivateCoupon`: write `REACTIVATED` event

**Null = pre-cutoff bridge (eliminates backfill and waiting period):** After Release 1 ships,
DataBricks deploys same day using a LEFT JOIN null bridge:

```sql
FROM source_delta.luci__coupons_issued ci
LEFT JOIN source_delta.luci__coupon_event_log el
  ON ci.coupon_code = el.coupon_code AND el.event_type = 'ISSUED'
WHERE ci.active = TRUE
  AND (
    el.event_time IS NULL            -- null = pre-Release-1 = always before any cutoff
    OR el.event_time <= TIMESTAMP(cutoff)
  )
```

A `null` entry means the coupon was issued before Release 1 and is guaranteed to predate any
cutoff. No backfill needed. No waiting period. Both releases can ship on the same day.

**Retention policy:**
- `ISSUED`, `REDEEMED`, `REVOKED`, `REACTIVATED`: clean after 7–30 days (table remains small)
- `UPLOADED`, `INVALIDATED_UNISSUED`: retain until code is ISSUED or series is deactivated

**Schema changes:** One new small table `coupon_event_log`. No ALTER on large tables.
**Java changes:** 6 event write points + new DAO.
**DataBricks changes:** New CTEs using `event_log`; null=pre-cutoff bridge query.
**ETL coordination:** `coupon_event_log` onboarded to Delta sync.
**Effort:** L (3 coordinated releases).
**Rollback:** Feature-flagged per org. Remove `event_log` CTEs from DataBricks to roll back.

---

### What Does NOT Change (Explicit Out of Scope — all options)

- `max_auto_update_time_issued` computation — operational marker for "last ETL run"; correctly uses `auto_update_time`. No change.
- Java-side `stats_series_summary` writes — correct at event time. No change.
- `stats_history_registry`, `stats_history_blueprint_*` table structure — no schema changes.
- Java stats assembly path (`StatsHistoryServiceImpl`, `MySQLCouponSeriesStatisticsReadService`) — no Java changes.
- Java diff-checker jobs (`CouponDaoImpl:1322-1328`, `CouponRedemptionDaoImpl:861-867`) — already correct; no change.
- ~~Timezone (line 39)~~ — intentional design, out of scope.

### Upstream / Downstream Coordination Required

- DataBricks change requires **backfill run** for affected date range after deploy. Existing wrong blueprint data does not self-correct on the next regular run.
- **No Java deploy required.**
- For Option 2: ETL team to add `created_on` to `source_delta.luci__coupons_created` sync.
- For Option 1: ETL team to confirm CDC sync mode and `logRetentionDuration`.

### Tenant Isolation Considerations

No cross-tenant risk. All CTEs are `GROUP BY org_id, coupon_series_id`. `orgId` threaded correctly throughout.

---

## Alternatives Considered

| Option | Approach | Schema change on large tables | Java change | DataBricks change | Effort | Decision |
|--------|----------|-------------------------------|-------------|-------------------|--------|---------|
| **Phase 0 — created_on swap** | Swap `last_updated_on` → `created_on` in `created` + `unissued` CTEs | None | None | 2 line changes | XS | **Ship now for upload-only orgs (Comcast)** — gate on Q7 |
| **Option 1 — Time Travel** | `TIMESTAMP AS OF cutoff` on Delta tables | None | None | 4 line additions | S | Interim — if CDC sync confirmed and Option 3 not yet ready |
| **Option 2 — SQL filter fix** | Immutable columns + `OR auto_update_time > cutoff` compensation | None | None | 6 filter conditions + ETL sync for `created_on` | S-M | Interim fallback — always available; use while Option 3 ships |
| **Option 3 — Immutable Event Log** | New `coupon_event_log` table; pure event-time counter formulas; null=pre-cutoff bridge | New small table only | 6 write points + DAO | New CTEs + null bridge | L | **Recommended long-term — fixes all bugs including FR-014; no backfill** |
| Option 4 — Java correction job | Scheduled job re-computes counts from MySQL and patches `stats_history_blueprint_*` | None | New job + DAO | None | M-L | Deferred — parallel-write risk; use only if DataBricks pipeline unavailable |
| Delayed cutoff offset | Increase `run_days_offset` from 2 to 3+ | None | None | Config only | S | Rejected — reduces frequency but does not fix; mergeUser can happen months after issue |
| Hybrid auto_update_time | Keep `auto_update_time` for delta detection, use `issued_date` only for attribution | None | None | Minor | S | Rejected — notebook does full re-query each run; `auto_update_time` not needed for change detection |

### Phase 0 detail (created_on swap — quick fix for upload-only orgs)

Change two lines in the DataBricks notebook to use `created_on` instead of `last_updated_on`:
- `created` CTE line 108: `TIMESTAMP_MILLIS(last_updated_on) <= cutoff` → `TIMESTAMP_MILLIS(created_on) <= cutoff`
- `unissued` left side line 126: `TIMESTAMP_MILLIS(cc.last_updated_on) <= cutoff` → `TIMESTAMP_MILLIS(cc.created_on) <= cutoff`

`created_on` is a `DEFAULT CURRENT_TIMESTAMP` column — immutable, set at upload time only. This
correctly attributes uploaded codes to their upload date. Safe for orgs that only upload (no revoke/
invalidate) because the `OR last_updated_on > cutoff` compensation logic is not needed in that case.
Gate: Q7 (`created_on` present in `source_delta.luci__coupons_created`).

### Option 3 detail (Immutable Event Log — long-term preferred)

Full design in Solution Chosen §Option 3 above. Key invariants:
- **IC**: `ISSUED − REVOKED` (both filtered by `event_time <= cutoff`)
- **UTC**: `UPLOADED − REVOKED − INVALIDATED_UNISSUED`
- **UC**: `UPLOADED − ISSUED − INVALIDATED_UNISSUED`
- **RC**: `REDEEMED − REACTIVATED`
- **null bridge**: pre-Release-1 rows have `event_time IS NULL` → treated as before any cutoff → no backfill required
- **FR-014**: USER_ID uploads write UPLOADED + ISSUED simultaneously → UTC automatically includes these codes

### Option 4 detail (Java correction job — fallback only)
A new `@Scheduled` job runs after each DataBricks parquet ingestion. Queries production MySQL with:
`SELECT COUNT(*) FROM coupons_issued WHERE issued_date <= snapshotDate AND (active=1 OR auto_update_time > snapshotDate) GROUP BY org_id, coupon_series_id`. Compares against current `stats_history_blueprint_*` values and issues corrections. **Risk:** concurrent writes if DataBricks run and job overlap; job must be idempotent with mutex. **Use only if** DataBricks pipeline is unavailable or CDC cannot be confirmed and Option 3 timeline is blocked.

---

## Rollout Strategy

### Phase 0 — Comcast Quick Fix (Independent, ships whenever Q7 confirmed)

**Target:** Upload-only orgs (no revoke / invalidate operations) waiting for correct UC/UTC counts.

**Change:** 2 lines in `luci_calculate_stats_incrm.py` (see Phase 0 detail above).

**Gate:** Q7 confirmed (`created_on` in `source_delta.luci__coupons_created`).

**Risk:** Low. IC/RC paths are unchanged. Upload-only orgs have no revoke/invalidate so the
`OR last_updated_on > cutoff` compensation is not needed.

**No Java deploy, no backfill, no waiting period.**

---

### Release 1+2+3 — Java + Dracarys + DataBricks (can ship together)

**Release 1 — Java: event_log writes**
- New `coupon_event_log` MySQL table + DAO
- 6 event write points in `LuciThriftServiceImpl` and processor chain
- USER_ID uploads: write `UPLOADED` + `ISSUED` simultaneously

**Release 2 — DataBricks: event_log CTEs with null=pre-cutoff bridge**
- Replace `coupons_issued`/`coupons_created` CTEs with `event_log` aggregations
- LEFT JOIN null bridge: `el.event_time IS NULL OR el.event_time <= cutoff`
- `coupon_event_log` onboarded to `source_delta` ETL sync by data-eng team
- **Deploys same day as Release 1** — null entries handle all pre-Release-1 historical records; no backfill

**Release 3 — Dracarys: canonical batchEventTime**
- Dracarys captures timestamp at batch START (before first `INSERT`)
- Passes `batchEventTimeMillis` in Luci notification API request
- Java `StatsSeriesSummaryDaoImpl:248` uses this for `summary_date` instead of `Instant.now()`
- Eliminates midnight batch split-attribution overcount (~24h transient; see Extended Analysis §H)

All three releases can ship in a single coordinated deploy. Releases 1 and 3 have no ordering
dependency between them. Release 2 requires Release 1 to have run at least once (for `event_log`
rows to exist), but the null bridge handles all older records.

**DataBricks compensation logic in the notebook is NOT needed** — the midnight overcount is
transient (~24h, self-correcting), and Option 3 eliminates the underlying mutable-timestamp
paradigm entirely.

---

### Comparison of All Options

| Option | Fixes IC/RC/UC/UTC | Fixes FR-014 (UTC USER_ID gap) | Fixes midnight overcount | Backfill needed | Effort | When to use |
|--------|-------------------|-------------------------------|--------------------------|-----------------|--------|-------------|
| **Phase 0** (`created_on` swap) | UC/UTC only (upload-only) | No | No | No | XS | **Now — unblocks Comcast** |
| **Option 1** (Time Travel) | Yes | No | No | Yes | S | Bridge if CDC confirmed + Option 3 not ready |
| **Option 2** (SQL filter fix) | Yes | No | No | Yes | S-M | Bridge if CDC unconfirmed + Option 3 not ready |
| **Option 3** (Immutable Event Log) | Yes | Yes | Yes (with Release 3) | No | L | **Long-term fix — all bugs, all orgs** |

**Recommendation:**
1. **Ship Phase 0 now** for Comcast and other upload-only orgs — 2 lines, low risk, independent
2. **Build Option 3** — structurally correct, fixes all three bug dimensions including FR-014
3. **Use Option 1 or Option 2 as a bridge** for orgs needing IC/RC fix while Option 3 ships

---

## Assumptions Going Into Tech Detail

1. `issued_date` on `coupons_issued` is set at issuance time and never updated — confirmed in `CouponEntity.java:26`; breaks if: a resend or re-issue flow overwrites `issued_date` (verify in `CouponRuntimeServiceImpl` issue flow)
2. `created_on` on `coupons_created` is truly immutable (`DEFAULT CURRENT_TIMESTAMP` only, no triggers) — confirmed in schema `coupons_created.sql:9`; breaks if: any UPDATE statement sets `created_on` explicitly
3. `redeemed_date` on `coupon_redemptions` is the immutable original redemption timestamp — needs confirmation (not read directly in this investigation); breaks if: redeemed_date is updated on reversal/reactivation
4. `coupons_issued.auto_update_time` is reliably refreshed by mergeUser/bill/resend operations — confirmed by user observation and `CouponEntity.java:36` field presence; this is the trigger for the bug
5. The DataBricks notebook does a **full re-query** from source tables each run — confirmed from notebook structure (no incremental state stored); breaks if: an incremental watermark is added later
6. `source_delta.*` tables are **CDC/incremental merge** synced (not full daily overwrite) — needed for Option 1 time travel to work; breaks if: tables are full-refresh overwrite (in which case use Option 2)

---

## Risks for Implementer

| Risk | Likelihood | Mitigation |
|------|-----------|-----------|
| `redeemed_date` on `coupon_redemptions` is null for some records (partial migration or old data) | Medium | Add `COALESCE(redeemed_date, auto_update_time)` fallback in `redeemed` CTE; validate in DataBricks before deploy |
| Changing the filter from `auto_update_time` to `issued_date` + compensation shifts which records appear in a given snapshot — backfill required | High | Plan a targeted re-run for the last N days; identify affected orgs via `SELECT org_id, COUNT(*) FROM coupons_issued WHERE auto_update_time > issued_date + interval 1 day GROUP BY org_id` (for metadata updates) and `SELECT org_id, COUNT(*) FROM coupons_issued WHERE active=0 AND auto_update_time > issued_date + interval 1 day GROUP BY org_id` (for revoke/invalidate) |
| `OR auto_update_time > cutoff` compensation on `active=FALSE` rows (Option 2 only): a coupon revoked before the snapshot but with `auto_update_time` accidentally refreshed after the cutoff by a background process would be incorrectly included | Low | Validate pre-deploy: `SELECT COUNT(*) FROM coupons_issued WHERE active=0 AND issued_date < :cutoff AND auto_update_time > :cutoff` — should be near-zero; investigate any unexpected rows |
| Timezone handling | N/A | Intentional design per ETL team — `databricks/stats/luci_timezone_descrepency.txt`. No change required. |
| Option 1: `source_delta` tables are full-refresh overwrite, not CDC | Medium — unknown until confirmed | Confirm with ETL/DataBricks team (Q6). If full-refresh, Delta history is not meaningful → fall back to Option 2. |
| Option 1: Delta `logRetentionDuration` < 3 days | Low — default is 30 days | Verify with ETL team. `run_days_offset=2` needs at minimum 2 days of history. |
| Option 2: `created_on` not available in `source_delta.luci__coupons_created` Delta sync | Medium | ETL team adds `created_on` to sync config (not a MySQL schema change). Use interim Bug 2 fix (no `created_on`) in the meantime. |

---

## Extended Analysis: Stats Assembly, Timezone, and Race Condition (2026-06-14)

*Findings from follow-on investigation on series 811192, org 2000000 (uscrm cluster).*

### A. Three-Layer Assembly Verified

Blueprint (snapshot Jun 11, active from Jun 13 06:01) + window rows (Jun 12, Jun 13) from
`stats_series_summary` produce exactly the values returned by `getCouponConfiguration`:

| Field | Blueprint | Window Σ | Assembly | API value | Match |
|-------|-----------|-----------|----------|-----------|-------|
| `num_issued` (IC) | 1212 | +198 | 1410 | 1410 | ✅ |
| `num_uploaded_total` (UTC) | 1020 | +165 | 1185 | 1185 | ✅ |
| `numTotal` (TC) | 1626 | +264 | 1890 | 1890 | ✅ |
| `num_uploaded_nonIssued` (UC) | 414 | +66 | 480 | 480 | ✅ |

All four fields match exactly. Formula `blueprint + Σ window` confirmed correct.
Window = `summary_date > snapshot_date` (Jun 12 and Jun 13 rows are live; Jun 11
row was absorbed into blueprint).

### B. latestIssualTime Timezone Convention — Intentional Design

`coupons_issued.issued_date` stores org/store local wall-clock time reinterpreted through the
JVM timezone. Set via `DateUtil.getCurrentTimeOfZoneAsLocalTimeZone(eventTimeZoneOffset)`
([DateUtil.java:85-116](../../../../src/main/java/com/capillary/luci/impl/utils/DateUtil.java)):

```java
// Takes org-local fields (e.g. IST 23:21:23), creates LocalDateTime,
// calls .toDate() in JVM timezone (CDT) → epoch = IST wall-clock as CDT epoch
DateTime dateInSourceTimeZone = new DateTime(sourceZone);
org.joda.time.LocalDateTime dateInLocalZone = new LocalDateTime(
    dateInSourceTimeZone.getYear(), ..., dateInSourceTimeZone.getSecondOfMinute());
return dateInLocalZone.toDate();
```

All three layers are consistent by design:
- `coupons_issued.issued_date` = org-local epoch (as above)
- `stats_series_summary.lit` = `.getTime()` of the same Date object (same epoch)
- `getCouponConfiguration.latestIssualTime` = lit = same epoch

The API correctly exposes issued time in org timezone. `issued_date` appears in the notebook
only at line 60 as `MAX(TIMESTAMP_MILLIS(issued_date)) AS last_issued_time` (lit computation).
It is NOT used for cutoff filtering or daily bucketing.

**Org 2000000 timezone confirmed from DB:** `Asia/Kolkata` (IST, UTC+5:30). Although tests run
on uscrm (CDT server), this test org was created by the Indian team and defaults to IST.

### C. issued_date Cannot Replace auto_update_time as Cutoff

`issued_date` is a `DATETIME` column — MySQL stores it as-is (no timezone conversion). It holds
org-local wall-clock time (IST for org 2000000). Using `issued_date <= cutoff` would mix
org-local time with a server-local cutoff — a different frame of reference for each org.

`auto_update_time` is a `TIMESTAMP` column — MySQL converts to UTC on write. DataBricks reads
it via `TIMESTAMP_MILLIS()` and both sides of the comparison are in the same server-local
frame (intentional per `luci_timezone_descrepency.txt`).

**Conclusion:** `auto_update_time` is the correct column for the cutoff. The bug is not which
column — it is that `auto_update_time` is mutable. Fix = keep `auto_update_time` but add
compensation logic (Option 2) or use Delta time travel (Option 1).

### D. summary_date Uses Server JVM Date (Not Org Timezone)

`stats_series_summary.summary_date` is set via
([StatsSeriesSummaryDaoImpl.java:248-249](../../../../src/main/java/com/capillary/luci/data/dao/impl/StatsSeriesSummaryDaoImpl.java)):

```java
ZonedDateTime zonedDateTime = Instant.now().atZone(ZoneId.systemDefault());
Date summaryDate = Date.from(zonedDateTime.truncatedTo(ChronoUnit.DAYS).toInstant());
```

`ZoneId.systemDefault()` = JVM server timezone (CDT on uscrm). DataBricks computes **lifetime
totals** per org/series — there is no day-level bucketing in the notebook. Daily rows in
`stats_series_summary` are exclusively written by Java event processors using `Instant.now()`
(server JVM clock). For IST orgs, issues between CDT 19:00–24:00 (= IST 00:30–05:30 next day)
are attributed to the CDT date. Separate lower-severity issue, not the mutable-timestamp bug.

### E. Midnight Race Condition — Transient 24h Undercount, Self-Correcting

A race exists between Java counter increment (`Instant.now()` → JVM clock) and DB
`auto_update_time` update (`ON UPDATE CURRENT_TIMESTAMP` → DB server clock). Under load these
two clocks can straddle midnight by 1–2 seconds.

**Race scenario (coupon C issued at CDT 23:59:59.9):**
- Java writes: `summary_date = Jun 13`, `ic += 1` in Jun 13 row
- DB writes: `auto_update_time = Jun 14 00:00:00` (DB clock fractionally ahead)

| Blueprint active | Coupon C in blueprint? | Jun 13 window row? | num_issued |
|-----------------|----------------------|-------------------|------------|
| Jun 13 blueprint (active Jun 15–Jun 16) | No — `auto_update_time=Jun14 > cutoff` | No — absorbed | −1 (undercount) |
| Jun 14 blueprint (active Jun 16+) | Yes — `auto_update_time=Jun14 ≤ Jun14 cutoff` | Not in Jun14 window | Correct ✅ |

**Duration:** ~24 hours. **Self-corrects** when the next day's blueprint activates. NOT
permanent. No double-count risk — the Jun 13 summary row is absorbed into the Jun 13
blueprint before coupon C reappears in the Jun 14 blueprint.

### F. TC vs UTC Gap: FR-014 — USER_ID Uploads Not Counted in UTC (Confirmed)

For series 811192, org 2000000:
- `numTotal` (TC) = 1890 — counts NONE + USER_ID codes
- `num_uploaded_total` (UTC) = 1185 — counts only codes with `coupons_created` rows
- **Gap = 705 = exactly the USER_ID code count confirmed from DB**

USER_ID upload jobs issue coupons directly into `coupons_issued` without `coupons_created`
rows. DataBricks `uploadedTotalCount` = `created_valid` from the `created` CTE — it can
never count USER_ID codes.

235 USER_ID upload jobs, 705 codes, all 705 issued. Gap is permanent and will not be resolved
by Option 1 or Option 2. Separate pre-existing bug, tracked as FR-014.

### G. Org Timezone — NOT a concern for count correctness

*Question: does org timezone (e.g., Comcast = America/Chicago CDT) affect counts when the
Phase 0 fix uses `created_on` / `last_updated_on`?*

**No. The pipeline is internally consistent — org timezone is irrelevant to count correctness.**

All three layers operate in the same reference frame (server-local time):

| Layer | Timestamp used | Reference frame |
|---|---|---|
| MySQL `coupons_created` | `created_on`, `last_updated_on` | Server-local (IST / CDT per cluster) |
| Java `stats_series_summary` | `summary_date` bucket assigned at write time | Server-local JVM clock |
| DataBricks cutoff | `TIMESTAMP_MILLIS(created_on) <= cutoff` | Server-local (cutoff passed in same TZ) |

Since all three layers use the same server-local timestamps, a code written at 23:30 server-time
on Jun 14 lands in the Jun 14 bucket in both Java and DataBricks — regardless of what time that
is in any org's local timezone. The `blueprint + Σ window` total is always correct.

**What org timezone affects:** per-day drilldown attribution only (a code at IST 01:00 lands in
the previous CDT-date bucket). Cumulative API fields (`num_issued`, `num_uploaded_total`,
`num_uploaded_nonIssued`, etc.) are unaffected. This is a display/reporting concern, not a
pipeline correctness concern, and is out of scope for this fix.

**Verified against Comcast Production (org 2000101, uscrm):** `default_time_zone_id=372`
(America/Chicago, UTC−5 CDT in summer). All counts confirmed correct despite CDT ≠ IST
server timezone.

### H. Midnight Batch Split-Attribution — Transient 24h OVERCOUNT

*Question: if a Dracarys batch upload straddles midnight, do codes end up attributed to different
dates — some to the day before midnight in DataBricks, all to the day after midnight in Java?*

**Yes — and the effect is a transient OVERCOUNT, not an undercount.**

**Mechanism:** Dracarys uploads 17,000 codes starting at CDT 23:55. Due to volume the job writes
rows over several minutes:
- 7,000 codes land with `created_on` / `auto_update_time` = Jun 13 (before midnight)
- 10,000 codes land with `created_on` / `auto_update_time` = Jun 14 (after midnight)

Dracarys calls the Luci notification API at Jun 14 00:05 — after midnight. Java records:
- `summary_date = Jun 14` (uses `Instant.now()` = JVM clock at notification time)
- `uc += 17,000` (entire batch in one Jun 14 summary row)

Two days later DataBricks runs with `cutoff = Jun 13`:
- Jun 13 blueprint: includes 7,000 codes (`auto_update_time ≤ Jun 13`) → UC = +7,000
- Java window (Jun 14+ rows): `uc = +17,000` (notification arrived Jun 14)
- **Net: UC = 24,000 for a 17,000-code batch → OVERCOUNT of 7,000**

**Duration:** ~24 hours. Self-corrects when Jun 14 blueprint activates:
- Jun 14 blueprint: includes all 17,000 codes — UC = 17,000
- Java window (Jun 15+): no UC entries for this batch
- **Net: UC = 17,000 ✅**

**Direction is OVERCOUNT, not undercount.** DataBricks sees the 7,000 pre-midnight codes PLUS
Java's full-batch summary row together. The 7,000 are double-counted for ~24h until the Jun 14
blueprint replaces the Jun 13 one.

**Fix — Canonical batchEventTime (Release 3):**

Dracarys captures a timestamp at batch START (before the first `INSERT`), passes it as
`batchEventTimeMillis` in the Luci notification API call. Java uses this for `summary_date`
instead of `Instant.now()` ([StatsSeriesSummaryDaoImpl.java:248](../../../../src/main/java/com/capillary/luci/data/dao/impl/StatsSeriesSummaryDaoImpl.java)):

```java
ZonedDateTime zonedDateTime = (batchEventTimeMillis != null)
    ? Instant.ofEpochMilli(batchEventTimeMillis).atZone(ZoneId.systemDefault())
    : Instant.now().atZone(ZoneId.systemDefault());
Date summaryDate = Date.from(zonedDateTime.truncatedTo(ChronoUnit.DAYS).toInstant());
```

Result: if the batch started Jun 13, ALL 17,000 codes get `summary_date = Jun 13` in Java.
DataBricks sees the 7,000 pre-midnight codes in the Jun 13 blueprint AND Java summary says Jun 13
for the same batch. No split-attribution overcount.

**DataBricks compensation logic is NOT needed** for this case — the overcount is transient (~24h)
and self-corrects on its own without any notebook change. Release 3 (Dracarys batchEventTime)
is the correct fix.

### I. Fundamental Design Problem — Two Incompatible Counting Paradigms

*Question: what is the architectural root cause driving all these counter bugs?*

**The Single Root Cause: two incompatible counting paradigms are combined in one formula.**

```
num_issued = blueprint_IC + Σ(window summary IC)
```

| Layer | Model | What it counts |
|-------|-------|----------------|
| DataBricks blueprint | **Current-state query** | Rows with `active=TRUE` AND `auto_update_time ≤ cutoff` — queries CURRENT row state for a past window |
| Java stats_series_summary | **Event-log delta** | `+1` when issued, `−1` when revoked — records EVENTS as they happened |

**Why they are incompatible:** A coupon revoked after `snapshotDate` but before the DataBricks ETL
run appears:
- In blueprint: **ABSENT** (`active=FALSE` at ETL run time)
- In Java window: **`+ric = 1`** (revoke event happened in the window)
- Net: `(0 − 1) + 0 = −1` ← negative count from combining incompatible models

No amount of column substitution within the current-state query model fixes this — only switching
the blueprint to an event-log model resolves the paradigm mismatch.

**Three dimensions of the problem:**

1. **Mutability dimension:** `auto_update_time` is mutable. It cannot represent "when did this
   event happen" — only "when was this row last touched." Using it as a point-in-time filter is
   structurally wrong for a snapshot system. *Options 1 and 2 fix this dimension.*

2. **State vs. event dimension:** DataBricks reads current `active` flag (state model). Java writes
   deltas at event time (event model). These cannot be mixed in one formula without explicit
   reconciliation. *Options 1 and 2 fix this via compensation; Option 3 eliminates the mismatch entirely.*

3. **UTC vs. upload dimension (FR-014):** USER_ID uploads write to `coupons_issued` only — no
   `coupons_created` row. DataBricks `created` CTE reads only `coupons_created`. UTC from
   DataBricks can never count USER_ID codes. The 705-code gap on series 811192 is the observed
   symptom. *Options 1 and 2 do NOT fix this. Option 3 fixes it via UPLOADED event for all import types.*

**What Option 3 fixes:** By replacing the current-state query with pure event-time arithmetic,
Option 3 eliminates all three dimensions. The formulas become:

```
IC  = ISSUED_events − REVOKED_events        (both filtered by event_time ≤ cutoff)
UTC = UPLOADED_events − REVOKED_events − INVALIDATED_UNISSUED_events
UC  = UPLOADED_events − ISSUED_events − INVALIDATED_UNISSUED_events
RC  = REDEEMED_events − REACTIVATED_events
```

No state flags. No mutable timestamps. No paradigm mismatch.

---

## Open Questions (Must Resolve Before Build)

- [ ] **Q1:** Is `redeemed_date` on `coupon_redemptions` always populated, or can it be null? Does it hold the original redemption timestamp or is it updated on reversal? — **Owner:** Developer (verify against `coupon_redemptions` schema and `CouponRuntimeServiceImpl` redemption flow)
- [ ] **Q2:** Full list of operations that update `coupons_issued.auto_update_time` (confirmed: mergeUser, bill update, revoke, invalidate) and `coupon_redemptions.auto_update_time` (confirmed: reactivate). Any others? Needed for backfill scope. — **Owner:** Developer (grep all `UPDATE` statements on these tables across Luci and member-care services)
- [ ] **Q3:** ~~`issued_date` in Delta Lake~~ — **CLOSED**: `issued_date` already used at notebook line 60 (`MAX(TIMESTAMP_MILLIS(issued_date)) AS last_issued_time`). Available in `source_delta.luci__coupons_issued` ✓. `redeemed_date` confirmed at line 80 ✓. **Additional note:** `issued_date` cannot be used as a cutoff alternative — it is a DATETIME column storing org-local time, incompatible with the server-local cutoff frame.
- [ ] **Q4:** ~~Timezone~~ — **CLOSED**: intentional design per ETL team. No action required on Luci side.
- [ ] **Q5:** For `coupon_redemptions`, confirm `redeemed_date` is immutable (not updated on reversal/reactivation). If null for some records, use `COALESCE(redeemed_date, auto_update_time)` fallback. — **Owner:** Developer
- [ ] **Q6 [BLOCKER FOR OPTION 1]:** Are `source_delta.luci__coupons_issued`, `source_delta.luci__coupon_redemptions`, and `source_delta.luci__coupons_created` ingested via **CDC/incremental Delta merge** or **full daily overwrite**? If CDC: use Option 1 (time travel). If full-refresh: use Option 2 (SQL filter fix). Also confirm `delta.logRetentionDuration` ≥ 3 days. — **Owner:** DataBricks/data-eng team
- [ ] **Q7 [BLOCKER FOR OPTION 2 Bug 2 AND Phase 0]:** Is `created_on` column from `coupons_created` MySQL table included in the `source_delta.luci__coupons_created` Delta sync? If not, ETL team must add it (no MySQL schema change — column already exists). — **Owner:** DataBricks/data-eng team
- [x] **Q8 — CLOSED (2026-06-18):** Dracarys does **NOT** capture a dedicated batch-start
  timestamp. However, `UploadCouponEntity.createdOn` (job creation date, persisted in DB, set
  at context creation in `ContextMapper.java:80`) is the correct proxy. It is stable across all
  retries of the same job (confirmed Q15). Dracarys team must use `uploadCouponEntity.getCreatedOn()`
  as the canonical `batchStartTime` for both `coupons_created` INSERTs and the `uploadedOn` field
  in the notify call. No new timestamp field needs to be introduced.
- [ ] **Q9 [FOR PHASE 0 SCOPE]:** Which orgs are "upload-only" (never use revoke / invalidateCoupons on `coupons_issued`)? Phase 0 is safe for these orgs. A quick query: `SELECT DISTINCT org_id FROM coupons_issued WHERE active=0` on each cluster gives revoke/invalidate orgs — orgs absent from this list are upload-only safe. — **Owner:** Developer / data-eng team

---

## What to Watch Post-Deploy

- **Before backfill:** Monitor `num_issued` for a known high-activity DISC_CODE_PIN series across a 3-day window. Confirm it no longer drops after metadata-update events.
- **After backfill:** Compare blueprint IC vs direct DB count:
  `SELECT org_id, coupon_series_id, COUNT(*) FROM coupons_issued WHERE issued_date <= :snapshotDate AND (active=1 OR auto_update_time > :snapshotDate) GROUP BY org_id, coupon_series_id`
  vs `SUM(stats_history_blueprint_*.ic)`. Delta should be zero.
- **UC invariant (DISC_CODE_PIN):** `num_uploaded_nonIssued + num_issued` = `num_uploaded_total`. A negative or zero `num_uploaded_nonIssued` on an active series with unissued coupons is a signal the bug is present.
- **num_issued / num_redeemed negative check:** Monitor for any `CouponConfiguration` response where `num_issued < 0` or `num_redeemed < 0`. These are definitively broken — a negative count is only possible via the double-subtraction bug.
- **Revoke/reactivate series check:** For a series that had revocations between snapshotDate and snapshotDate+2 days, verify `num_issued` does not drop below the pre-revoke value minus the actual revoke count.

---

## Handoff Note for tech-detailer

Root cause confirmed in `databricks/stats/luci_calculate_stats_incrm.py`. The fundamental issue
is two incompatible counting paradigms in one formula: DataBricks uses a current-state query model
(mutable timestamps, current `active` flag) while Java uses an event-log delta model. See Extended
Analysis §I for the full architectural breakdown.

**Recommended solution: Option 3 (Immutable Event Log).** This fixes all three bug dimensions
including FR-014 (USER_ID upload gap) and requires no backfill. Full design in Solution Chosen §Option 3.

**Immediate action for Comcast / upload-only orgs: Phase 0** — confirm Q7 (`created_on` in Delta
sync), then swap 2 lines in the notebook. No Java deploy, no backfill.

**Interim DataBricks-only fix paths (while Option 3 ships):**

**Path A (Option 1 — Time Travel):** If `source_delta` tables are CDC-synced (Q6):
  - Add `TIMESTAMP AS OF '{cutoff}'` to 4 table references in the notebook
  - Requires backfill run after deploy

**Path B (Option 2 — SQL filter fix):** If full-refresh or CDC unconfirmed:
  - 6 SQL filter changes in the notebook
  - ETL team adds `created_on` to `source_delta.luci__coupons_created` sync (Q7)
  - Requires backfill run after deploy

**Option 3 rollout (3 coordinated releases — can ship together):**
- Release 1 (Java): `coupon_event_log` table + DAO + 6 event write points
- Release 2 (DataBricks): event_log CTEs with null=pre-cutoff bridge — deploy same day as Release 1
- Release 3 (Dracarys): canonical `batchEventTimeMillis` passed to Luci notify API

**Key risks to explore during tech detail:**
- Confirm CDC sync mode and `logRetentionDuration` for Option 1 (Q6)
- Confirm `created_on` in Delta sync for Phase 0 and Option 2 Bug 2 fix (Q7)
- Confirm `redeemed_date` immutability on reversal (Q5)
- Confirm Dracarys can capture batch-start timestamp for Release 3 (Q8)
- Identify upload-only orgs for Phase 0 scope (Q9)
- For Option 1/2 backfill: `SELECT org_id, COUNT(*) FROM coupons_issued WHERE auto_update_time > issued_date + interval 1 day GROUP BY org_id` gives metadata-update affected orgs

**Contracts to validate:**
- `issued_date` ✓ already in `source_delta.luci__coupons_issued` (notebook line 60)
- `redeemed_date` ✓ already in `source_delta.luci__coupon_redemptions` (notebook line 80)
- `created_on` — needs confirmation in `source_delta.luci__coupons_created` (Q7)
- `coupon_event_log` — new table; needs DBA provisioning + ETL sync onboarding
- Java `ParquetFileService` reads `max_auto_update_time_issued` — unchanged; no redeploy

**Suggested test coverage emphasis:**
- Unit (DataBricks / Option 3): all 5 scenario types — metadata update, revoke, invalidate, reactivate, is_queued — verify correct event-time count per scenario
- Integration (Option 3 Release 1): seed `coupon_event_log` with ISSUED + REVOKED events; assert IC formula correct
- Integration (Option 3 null bridge): seed `coupons_issued` row with NO `event_log` entry; assert it appears in IC (null = pre-cutoff)
- Integration (FR-014 fix): USER_ID upload writes both UPLOADED + ISSUED events; assert UTC includes USER_ID codes
- Integration (Release 3): pass `batchEventTimeMillis` in notify call; assert `summary_date` = batch-start date, not notification-arrival date
- Invariants post-deploy: `issuedCount + unissuedCount` preserved; `num_issued ≥ 0`; `num_redeemed ≥ 0`
- Negative count detection: any `num_issued < 0` or `num_redeemed < 0` = fix not complete

---

## Newly Identified Issues (2026-06-18)

*Findings from follow-on investigation during Phase 0 rollout and production validation.*
*These are pre-existing issues not introduced by the DataBricks fix — captured here for tracking.*

---

### J. Stats Read Path Split — Enforcement vs. Display Paths Differ

**Finding:** The DB-stats-read path (`isDbStatsReadEnabled`) controls two different behaviours
that must not be conflated:

| Path | Controlled by | What it reads | Affected by DataBricks bug? |
|------|--------------|---------------|-----------------------------|
| **Stats API display** (`getCouponConfiguration`) | `CouponSeriesStatisticsServiceFactory` | `MySQLCouponSeriesStatisticsReadService`: `bulkGetHistoricalValues()` (blueprint, NO cache) + `bulkGetMysqlSummaryValues()` | YES — blueprint wrong values displayed |
| **Limit enforcement** (issue/redeem checks) | `isDbStatsReadEnabled` guard in processor | `StatsHistoryServiceImpl.getHistoricalValue()` (`@Cacheable MIDNIGHT_EXPIRING`) + `getMysqlSummaryValue()` | YES — wrong limit thresholds used |
| **Redis path** (orgs with `isDbStatsReadEnabled=false`) | `RedisCouponSeriesStatisticsReadService` | Redis `RAtomicLong` counters populated by `calculateAllStats()` from OLTP tables | NO — OLTP counts are not affected by DataBricks bug |

**Implication:** For Redis-path orgs, the DataBricks blueprint miscompute affects only the
`getCouponConfiguration` API display — NOT live issuance/redemption limit enforcement.
For DB-stats-read orgs (e.g., Comcast), both display AND enforcement are affected.

**Key implementation details confirmed:**
- `bulkGetHistoricalValues()` (stats API, bulk path) — **NO `@Cacheable`** — reads rolling
  table fresh on every call. Fix reflects immediately after `statsHistory` job writes new
  blueprint.
- `getHistoricalValue()` (single-key, limit enforcement) — **`@Cacheable(MIDNIGHT_EXPIRING_CACHE_NAME)`**
  with date-prefix key format: `{cacheName}:{YYYY-MM-DD}::{baseKey}`. Key rotates at midnight
  (new date → automatic cache miss). TTL = 86,500 seconds (25 h, insurance only).
- `getMysqlSummaryValue()` — **NOT cached** — reads `stats_series_summary` fresh on every call.

**No action required** — architecture is correct. Documented here to prevent future confusion
when debugging limit enforcement failures vs. display discrepancies.

---

### K. Cache Staleness Boundary in `MaxRedemptionForSeriesProcessor` (Bug 3 candidate)

**Location:** `MaxRedemptionForSeriesProcessor.java:124–129`

**Formula (both commit and non-commit paths):**
```
net_RC = [today's RC counter]
       + getHistoricalValue(RC)      ← @Cacheable, date-prefix key, endDate = YESTERDAY
       - getMysqlSummaryValue(RRC)   ← NOT cached, endDate = TODAY
```

**The boundary case — triggered when statsHistory runs mid-day:**

Suppose statsHistory runs at 10:30 AM on day D, advancing snapshot from S1 to S2 (S2 = S1 + 1 day):

```
Step 1 — 10:00 AM, before statsHistory:
  getHistoricalValue(RC) is called and cached:
    Key: MIDNIGHT_EXPIRING_CACHE_NAME:{D}::statsHistoryPerKey:{orgId}_{seriesId}_rc_...
    Registry used: S1 (e.g., Jun 15)
    Value: blueprint_RC(≤Jun15) + summary_RC(Jun16 only, yesterday=Jun16)
    ← CACHED

Step 2 — 10:30 AM, statsHistory saves new registry (S2 = Jun 16):
  @CacheEvict on StatsHistoryRegistryDaoImpl.save():
    Evicts: MIDNIGHT_EXPIRING_CACHE_NAME::statsRegistryKey:{orgId}   ✓
    Does NOT evict: statsHistoryPerKey:... entries                    ✗

Step 3 — 10:31 AM, redemption check arrives:
  getHistoricalValue(RC) — cache HIT (stale S1-based):
    Still returns: blueprint_RC(≤Jun15) + summary_RC(Jun16)
    RC window covers: Jun16 only (via summary)

  getMysqlSummaryValue(RRC) — NOT cached, reads fresh:
    findLastActiveByOrgId() → cache miss (statsRegistryKey evicted) → fresh S2=Jun16
    startDate = S2+1 = Jun17
    endDate = today = D
    RRC window covers: Jun17 to D

Gap: summary_RRC on Jun16 (= date S1+1 = date S2) is:
  - EXCLUDED from fresh getMysqlSummaryValue (startDate = Jun17, not Jun16)
  - The corresponding RC on Jun16 IS included in the stale getHistoricalValue cache

Net effect: if any coupon_redemption was reactivated on Jun16, that RRC(Jun16) is invisible
to getMysqlSummaryValue, but the original RC is in the stale cache. Formula overcounts net_RC
by RRC(gap_date). Direction: tighter limit → false "max redemption hit" → valid redemption blocked.

Duration: until midnight, when date-prefix key changes → cache miss → both calls re-anchor to S2.
```

**Evidence:**

| File | Line | Finding |
|------|------|---------|
| `MaxRedemptionForSeriesProcessor.java` | 124–129 | `getHistoricalValue(RC)` + `getMysqlSummaryValue(RRC)` combined in one formula |
| `StatsHistoryServiceImpl.java` | 49–50 | `@Cacheable(MIDNIGHT_EXPIRING_CACHE_NAME)` on `getHistoricalValue()` |
| `StatsHistoryServiceImpl.java` | 83–84 | `endDate = LocalDate.now().minusDays(1)` — yesterday |
| `StatsHistoryServiceImpl.java` | 197–213 | `getMysqlSummaryValue()` — NOT cached, `endDate = LocalDate.now()` — today |
| `StatsHistoryRegistryDaoImpl.java` (save) | — | `@CacheEvict` evicts `statsRegistryKey:{orgId}` only; `statsHistoryPerKey` NOT evicted |
| `MidnightExpiringCacheManager.java` | — | Key format: `{cacheName}:{YYYY-MM-DD}::{key}` — date prefix causes natural rotation at midnight |

**Severity:** LOW in practice.
- RRC events (bill reversal reactivating a redemption) are rare.
- Gap window = exactly 1 day (DataBricks advances snapshot by 1 day per run).
- Duration bounded by midnight cache rotation.
- Only affects DB-stats-read orgs (Comcast et al.).

**Root cause:** `@CacheEvict` on registry save evicts only the registry lookup cache
(`statsRegistryKey`), not the derived per-key historical values (`statsHistoryPerKey`).
The derived cache is valid for the snapshot it was computed with, but that snapshot can advance
within the same calendar day.

**Fix options (ranked):**

| Option | Approach | Effort | Notes |
|--------|----------|--------|-------|
| **B — Consistent registry per request** | Fetch `findLastActiveByOrgId()` once at the top of `getRedeemedCouponCount()`; pass `snapshotDate` down to both `getHistoricalValue()` and `getMysqlSummaryValue()` | S (5-line change) | Eliminates the class of bug entirely; both calls guaranteed to use the same snapshot |
| A — Pattern-evict `statsHistoryPerKey` on save | Add Redis key-pattern delete matching `MIDNIGHT_EXPIRING_CACHE_NAME:*:statsHistoryPerKey:{orgId}_*` when registry advances | M | Spring Cache has no wildcard evict; requires direct Redis key scan — fragile |
| C — Accept as-is | Document and monitor; add log warning when save() is called during business hours | XS | Acceptable given rarity of RRC events; revisit if redemption limit false-positives are observed |

**Recommendation:** Option B during the Option 3 (Immutable Event Log) build. Until then, Option C.

**Open question:**
- [ ] **Q10:** What is the actual frequency of `reactivateCoupon` calls during business hours for
  DB-stats-read orgs? If < 1/day, Option C is safe indefinitely. If measurable, Option B should
  ship alongside Phase 0. — **Owner:** Developer (query `stats_series_summary` for `rrc > 0` rows
  on uscrm DB-stats-read orgs)

---

### L. Phase 0 Rollout Status (as of 2026-06-18)

| Cluster | File | Lines 108, 126 fix | statsHistory triggered | Validated |
|---------|------|--------------------|------------------------|-----------|
| `incrm` | `luci_calculate_stats_incrm.py` | ✅ Applied | — | — |
| `uscrm` | `luci_calculate_stats_uscrm.py` | ❌ Pending | ❌ Pending | ❌ Pending |

**Comcast production target:** org 2000101, uscrm shard 1, 5 active series.
Pre-fix baseline captured in `2026-06-17-comcast-pre-fix-baseline.md` (same directory).

Expected post-fix delta: UC increase of ~5,000 total across series 712348, 712349, 760256, 812307.
Series 712350 expected unchanged (control — no codes in the gap window).

**Validation steps (after uscrm Phase 0 apply + statsHistory run):**
1. Confirm new parquet written to S3 for correct snapshot date.
2. Trigger `/coupon/statsHistory` on uscrm.
3. Call `getCouponConfiguration` for all 5 series; compare UC to pre-fix baseline.
4. UC increase ≈ 5,000 across 4 affected series; 712350 unchanged → fix confirmed.

---

### M. Late-Notify Double-Count and stats_series_summary Idempotency Gap (Bug 4)

**Context:** Dracarys (coupon upload batch system) calls `notifyCouponsUploadRequest` after
completing an upload job. If this call fails on day D and is retried on day D+2 (after DataBricks
ETL has run with cutoff = D), a permanent double-count occurs in `stats_series_summary` for
DB-stats-read orgs.

#### Why the Redis path auto-corrects but the DB path does not

`notifyCouponsUploadRequest` (line 1740) calls `calculateStatistics()` which invokes
`calculateAllStats()`. This re-reads directly from OLTP (`m_couponShardService.getIssuedVouchersCount()`,
`getCouponsTotalCountForSeries()`, etc.) and **SETS** the Redis `RAtomicLong` to the OLTP truth value.
A second call produces the same result — idempotent by nature. No delta accumulation possible.

The DB stats path has no equivalent mechanism. Every write is:
```java
// StatsSeriesSummaryDaoImpl.java:268-286
UPDATE … SET value = value + ?       // delta ADD, not SET
INSERT … ON DUPLICATE KEY UPDATE value = value + ?
```

`summary_date` is always `Instant.now().truncatedTo(DAYS)` — late retry on D+2 writes a second
`+N` delta on date D+2, a different row from the original (which was never committed on D).

#### Double-count mechanism

```
Day D:   notify call fails → stats_series_summary NOT written for D
Day D+2: ETL ran on D+1 with cutoff=D → blueprint_UC = N (codes are in coupons_created with created_on ≤ D)
         statsHistory: snapshotDate = D, startDate for window = D+1

Day D+2 09:00 AM: retry notify → summary_date = D+2, uc = +N written

Stats API assembly (bulkGetMysqlSummaryValues, endDate = today):
  blueprint_UC(N) + window_UC(D+1 to D+2)(= +N from retry) = 2N ← DOUBLE COUNT ❌

Limit enforcement (getHistoricalValue, endDate = yesterday = D+1):
  window_UC(D+1 to D+1) = 0 (retry row is on D+2, not in yesterday window)
  Total = N + 0 = N ← correct for enforcement ✓ (but stale cache may suppress this until midnight — see §K)

Self-corrects when: next ETL cycle cutoff ≥ D+2 → D+2 summary row absorbed into new blueprint
  → falls below startDate → no longer in window → display returns to N ✓
```

#### All paths writing to stats_series_summary — idempotency audit

| Write path | File | Fields written | Idempotent? | Duplicate risk |
|------------|------|---------------|-------------|----------------|
| `notifyCouponsUploadRequest` | `LuciThriftServiceImpl.java:1726–1737` | UC, UTC, TC | ❌ | **HIGH** — Dracarys retries whole call |
| `DracarysCouponSeriesStatisticsService.addIssuedCount` | `DracarysCouponSeriesStatisticsService.java:34` | IC | ❌ | **HIGH** — same Dracarys caller |
| `DracarysCouponSeriesStatisticsService.addRedeemedCount` | `DracarysCouponSeriesStatisticsService.java:56` | RC | ❌ | **HIGH** — same Dracarys caller |
| Issue flow (`incrementIssuedCount`) | `CouponSeriesConfigImpl.java:573` | IC, UC | ❌ | LOW — single-coupon, RabbitMQ at-least-once only |
| Redeem flow | `CouponSeriesConfigImpl.java:418–446` | RC | ❌ | LOW — single-coupon |
| Revoke/invalidate/reactivate | `CouponSeriesConfigImpl.java:452–482` | RIC, RRC, IC(-) | ❌ | LOW — single-coupon |
| Revoke TC/UTC correction | `CouponSeriesConfigImpl.java:509–510` | TC(-), UTC(-) | ❌ | LOW |

**No write path in Luci has batch-level deduplication today.**

#### `NotifyCouponsUploadRequest` thrift struct — dedup gap

Fields: `requestId`, `orgId`, `couponSeriesId`, `totalIssuedCount`, `totalUploadCount`,
`uploadedOn`, `customerIdentifierType`, `totalRedeemedCount`.

`uploadedOn` (epoch ms) is already in the struct and is set to the batch upload time. It is used
only for `setLastIssueTime` — **never used as `summary_date`**. This is the key missed opportunity.

#### Fix Options

**Option 1 — Upload-job deduplication table (true idempotency)**

New small table: `coupon_upload_notify_log(org_id, series_id, upload_job_id, processed_at)`  
PRIMARY KEY `(org_id, series_id, upload_job_id)`

Add optional `uploadJobId` field to `NotifyCouponsUploadRequest` thrift struct (backward compatible).

In `notifyCouponsUploadRequest`:
```java
if (uploadJobId != null) {
    int inserted = jdbcTemplate.update(
        "INSERT IGNORE INTO coupon_upload_notify_log (org_id, series_id, upload_job_id, processed_at) VALUES (?,?,?,NOW())",
        orgId, couponSeriesId, uploadJobId);
    if (inserted == 0) {
        logger.info("Duplicate notify for job {}, skipping stat write", uploadJobId);
        return; // idempotent no-op
    }
}
// proceed with stat writes
```

Dracarys passes the stable job ID in each notify call. Same pattern for `DracarysCouponSeriesStatisticsService`.

| Dimension | Assessment |
|-----------|-----------|
| Fixes | Same-day concurrent retry + cross-day late retry |
| Backward compatible | Yes — `uploadJobId` is optional; existing callers unaffected |
| Effort | M (half day) |
| Risk | Low |
| TTL cleanup needed | Yes — `DELETE WHERE processed_at < now() - interval 30 day` |
| Dracarys coordination | Dracarys must pass stable job ID (already has it internally) |

**Option 2 — Use `uploadedOn` as `summary_date` (Release 3 prerequisite)**

In `StatsSeriesSummaryDaoImpl.addLongValueToDbWithoutTransaction()`, accept an optional
`Date summaryDate` parameter. In `notifyCouponsUploadRequest`, pass `new Date(uploadedOn)`
as the `summaryDate` for TC, UC, UTC writes.

Effect: A late retry on D+2 with `uploadedOn = D` writes to `summary_date = D`. After ETL
with cutoff = D, that D row is below the new `startDate = D+1` → not in window → no double-count.

**⚠️ PREREQUISITE (confirmed Q11):** Today `uploadedOn = System.currentTimeMillis()` at
notification send time (`PostUploadConvertorImpl.java:34`). A retry on D+2 sends `uploadedOn = D+2`
→ Option 2 has zero effect without the Dracarys fix. Dracarys **must** change `uploadedOn` to
pass `uploadCouponEntity.getCreatedOn().getTime()` (job creation date = stable batchStartTime,
confirmed Q15) before Option 2 eliminates cross-day double-counts. Both changes must ship together.

This is the same canonical `batchEventTimeMillis` fix as Release 3 (§H), applied to the notify
path. **If Release 3 ships (with Dracarys fix), this is automatically solved for UC/UTC.**

**Luci-side implementation (2026-06-18):** `StatsSeriesSummaryDao` new overload, `CouponSeriesConfigImpl`
`summaryDate` overloads, and `LuciThriftServiceImpl` notify handler all already implemented — see §N.

| Dimension | Assessment |
|-----------|-----------|
| Fixes | Cross-day late retry only. Does NOT fix same-day concurrent retry |
| Prerequisite | Dracarys must change `uploadedOn` to `uploadCouponEntity.getCreatedOn()` |
| Backward compatible | Yes — fall back to `Instant.now()` if `uploadedOn` is 0 |
| Effort | S Luci (done) + S Dracarys (`PostUploadConvertorImpl:34` one-liner) |
| Risk | Low now that prerequisite is identified |

**Option 3 — OLTP-backed reconciliation (detection + correction, not prevention)**

Post-ETL scheduled job that reads OLTP counts, computes expected `stats_series_summary` values
for recent dates, and corrects drifted rows. Mirrors `calculateAllStats()` for the DB path.
Useful as a monitoring + safety net but complex to implement correctly near boundary dates.
Defer until Options 1+2 are shipped.

#### Recommendation

**Ship Option 2 immediately** (2h, self-contained, solves the cross-day case which is the most
impactful). **Ship Option 1 with the Option 3 / event-log work** (requires Dracarys coordination,
higher effort, solves concurrent retries too). Option 3 as a future monitoring tool.

**For Release 3:** the `batchEventTimeMillis` fix in Option 2 (notify path) and the Dracarys
`summary_date` fix are the same change — coordinate with Release 3 to avoid duplicate implementation.

#### Open Questions

- [x] **Q11 — CLOSED (2026-06-18):** `uploadedOn` = **notification SEND TIME**, not batch start.
  `PostUploadConvertorImpl.java:34`: `.setUploadedOn(System.currentTimeMillis())` — this fires
  at the moment the Luci thrift call is placed, which is after all batches are written.
  `requestId` is also generated fresh each call (`"post_upload_" + orgId + "_" + seriesId + "_" + new Date() + "_" + new Date().getTime()`), so it is NOT stable across retries.
  **Impact on Option 2:** Option 2 (use `uploadedOn` as `summary_date`) only eliminates the
  cross-day double-count IF Dracarys first changes `uploadedOn` to pass `batchStartTime`
  (i.e., `UploadCouponEntity.createdOn`). Without that fix, `uploadedOn = retry send time = D+2`
  and Option 2 has no effect on the double-count. **Dracarys change is a prerequisite for
  Option 2 to work.**

- [x] **Q12 — CLOSED (2026-06-18):** `UploadCouponEntity.id` (DB primary key of the upload job
  record) = stable job ID, survives restarts. Available in `postCommitOperations()` via
  `couponUploadContext.getUploadCouponEntity().getId()`. Dracarys must pass this as
  `uploadJobId` in each notify call and each `addIssuedCount`/`addRedeemedCount` call.
  **`requestId` is not reusable for dedup** — it embeds `System.currentTimeMillis()` and
  generates a new value each call, even on retry.

- [x] **Q13 — CLOSED (2026-06-18):** **YES — Dracarys calls `addIssuedCount` directly**, in
  `CustomerTaggedUploader.preCommitOperations():94`:
  `couponSeriesStatisticsService.addIssuedCount(orgId, couponSeriesId, totalValidUploadCount)`.
  This is the pre-upload Redis count reservation for tagged uploads. It is NOT inside the notify
  call — it is a separate direct call to `DracarysCouponSeriesStatisticsService.java:34`.
  **Option 1 dedup table must cover this endpoint too**, otherwise tagged upload retries can
  double-count IC in `stats_series_summary` even if the notify dedup works.

- [x] **Q14 — CLOSED (2026-06-18) — PATH A confirmed:** Dracarys writes to `coupons_created`
  **directly** via Luci's `CouponsCreatedDao` (autowired in `BaseCouponUploaderImpl.java:33`).
  `couponsCreatedDao.save(entities)` is called in `NonCustomerTaggedUploader.commitToCreated():89`
  and `CustomerTaggedUploader.saveCouponsCreated():472`. **No Luci API intermediary.**
  The change to set `created_on = batchStartTime` is Dracarys-side only:
  - `BaseCouponUploaderImpl.constructCouponCreatedEntities()` — add `entity.setCreatedOn(batchStartTime)`.
  - `CustomerTaggedUploader.constructGeneratedCouponCreatedEntities()` — same.
  Luci DAO changes (`CouponsCreatedEntity` + `CouponsCreatedDaoImpl`) are required to accept
  and persist the `createdOn` field — these are already implemented (see §N).

- [x] **Q15 — CLOSED (2026-06-18):** `UploadCouponEntity.createdOn` (field at line 50) = **job
  creation date**, persisted in DB. Set to `new Date()` in `ContextMapper.java:80` when the
  upload context is first built. On retry, the entity is reloaded from DB and retains the
  original creation date — confirmed by `CouponUploadServiceImpl.java:203` which passes
  `.createdOn(uploadContext.getUploadCouponEntity().getCreatedOn())` through context rebuilds.
  **This is the correct `batchStartTime` to use.** It is immutable once set and identical
  across all retries of the same job. Dracarys should use `uploadCouponEntity.getCreatedOn()`
  as both the `created_on` value for `coupons_created` entities and as `uploadedOn` in the
  notify call.

#### Dracarys batchStartTime fix — combined impact if implemented

If Dracarys (a) sets `created_on = batchStartTime` for all INSERTs in a job and (b) passes `batchStartTime` as `uploadedOn` in the notify call, and Luci uses `uploadedOn` as `summary_date`:

| Problem | Status |
|---------|--------|
| Midnight batch split-attribution overcount (§H) | ✅ Eliminated — all codes in batch have same `created_on` |
| DataBricks cutoff split across batch boundary | ✅ Eliminated — batch is atomically in or out of any cutoff |
| Late-notify double-count cross-day (§M) | ✅ Eliminated — retry writes to same `summary_date = batchStartTime date`, which is in blueprint zone after ETL |
| Concurrent same-day retry double-count | ❌ Requires dedup table (§M Option 1) |

This is the canonical upstream fix. It makes the upload pipeline end-to-end consistent: `created_on` = `summary_date` = `batchStartTime` for every code in every job.

---

### N. batchStartTime Fix — Implementation Status (2026-06-18)

#### Luci-side — IMPLEMENTED, compile-clean

All Luci changes are applied to branch `master` (local) and compile clean (BUILD SUCCESS).

| File | Change |
|------|--------|
| `CouponsCreatedEntity.java` | Added `createdOn` field + getter/setter |
| `CouponsCreatedDao.java` | Added `CREATED_ON_COL = "created_on"` interface constant |
| `CouponsCreatedDaoImpl.java` | `create()` includes `created_on` in valueMap when non-null; `insert(List)` uses extended column map when `getCreatedOn() != null` (backward-compatible — existing callers with null `createdOn` use original column map, MySQL DEFAULT still applies) |
| `StatsSeriesSummaryDao.java` | New interface overload: `addLongValueToDbWithoutTransaction(orgId, seriesId, key, count, Date summaryDate)` |
| `StatsSeriesSummaryDaoImpl.java` | Implements overload: truncates passed `summaryDate` to day-start, then calls existing `addLongValue()` |
| `CouponSeriesConfig.java` (interface) | New overloads: `incrementUploadedCount(..., Date summaryDate)`, `incrementUploadedTotalCount(..., Date summaryDate)`, `incrementTotalCount(..., Date summaryDate)` |
| `CouponSeriesConfigImpl.java` | Implements all three overloads, delegates to `StatsSeriesSummaryDao` new overload |
| `LuciThriftServiceImpl.java` | `notifyCouponsUploadRequest()` now computes `summaryDate = new Date(uploadedOn)` and passes it to TC/UC/UTC increment calls. No change to `calculateStatistics()` Redis path. |

**Backward compatibility:** All changes are additive overloads. Existing callers are unaffected.
The notify handler's TC/UC/UTC writes now use `uploadedOn` date — but since today `uploadedOn`
is still the send time (Dracarys not yet changed), the effective `summaryDate` is still today
for normal runs. The fix becomes active once Dracarys fixes `uploadedOn` (see below).

#### Dracarys-side — PENDING (Dracarys team to implement)

**Source of truth:** `UploadCouponEntity.createdOn` = job creation date, persisted, retry-safe (Q15).

Two files require changes. Exact diffs:

**1. `PostUploadConvertorImpl.java` line 34:**
```java
// BEFORE:
.setUploadedOn(System.currentTimeMillis())

// AFTER:
.setUploadedOn(uploadCouponEntity.getCreatedOn() != null
    ? uploadCouponEntity.getCreatedOn().getTime()
    : System.currentTimeMillis())
```

**2. `BaseCouponUploaderImpl.constructCouponCreatedEntities()` — add `batchStartTime` param:**
```java
// Change signature:
protected List<CouponsCreatedEntity> constructCouponCreatedEntities(
    List<TempTableEntity> validRows, boolean isQueued, Date batchStartTime)

// Add inside the loop, after setLastUpdatedOn:
if (batchStartTime != null) {
    tempEntity.setCreatedOn(batchStartTime);
}
```

**3. `NonCustomerTaggedUploader.commitToCreated()` — thread batchStartTime:**
```java
// Add at top:
Date batchStartTime = couponUploadContext.getUploadCouponEntity().getCreatedOn();
// Change call:
List<CouponsCreatedEntity> couponsCreatedEntities =
    constructNonTaggedCouponCreatedEntities(tempTableEntityList, batchStartTime);

// Update helper signature:
private List<CouponsCreatedEntity> constructNonTaggedCouponCreatedEntities(
    List<TempTableEntity> validRows, Date batchStartTime) {
    return constructCouponCreatedEntities(validRows, false, batchStartTime);
}
// Also add: import java.util.Date;
```

**4. `CustomerTaggedUploader.constructGeneratedCouponCreatedEntities()` — set createdOn:**
```java
// Add at top of method:
Date batchStartTime = couponUploadContext.getUploadCouponEntity().getCreatedOn();

// Add inside the lambda, after setValid(true):
if (batchStartTime != null) {
    couponCreated.setCreatedOn(batchStartTime);
}
```

#### Deploy order

Luci must deploy first (or simultaneously). Dracarys entities call Luci's `CouponsCreatedDao.save()` — the Luci DAO change to accept `createdOn` must be live before Dracarys starts setting it. If Luci deploys first with `createdOn=null`, MySQL DEFAULT applies (current behavior — no regression). Once Dracarys deploys, `createdOn` is explicitly set for all new uploads.
