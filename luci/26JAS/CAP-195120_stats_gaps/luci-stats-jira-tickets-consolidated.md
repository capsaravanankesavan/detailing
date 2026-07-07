---
title: "Luci Coupon Series Stats — Jira Ticket Drafts for Review"
subtitle: "CAP-195120 (JAS26 — Calculate Stats Release) | 7 New Tickets | Ready for Author Review"
---


# Luci Coupon Series Stats — Jira Ticket Drafts for Review

**Epic:** CAP-195120 (JAS26 — Calculate Stats Release)
**Prepared by:** Explore (Peer Architect)
**Date:** 2026-07-04
**Status:** Awaiting author review before Jira creation

* * *

## Context & Ticket Map

### Already Done (no new tickets needed)
| Ticket | What | Status |
|---|---|---|
| CAP-195082 | Phase 0 `created_on` swap in DataBricks + Option 2 SQL filter fix — 5 clusters (STAGE, INCRM, ASIACRM, EUCRM, USCRM) | ✅ CLOSED |
| CAP-190758 | Root cause bug tracker — mutable timestamp miscompute | Open P1 — enrich with solution options from analysis doc |
| CAP-191062 | USER_ID upload Java counter fix | In Dev — add scope note: DataBricks blueprint UTC gap still open until T4 ships |
| CAP-188284 | Redis dual Redisson client spike | Open P2 |
| CAP-184614 | Stop Redis writes + clean Redis keys | Open P2, blocked by CAP-188284 |
| CAP-189119 | 20ms latency spike investigation | Open P2 |

### New Tickets to Create (this document)
| # | Title | Priority | Repo | Depends On |
|---|-------|----------|------|-----------|
| **T1-new** | Deploy DataBricks stats script to leftover clusters | P2 | Luci/DataBricks | CAP-195082 |
| **T2-new** | Automation: reversal flow coverage SM_09–SM_12 | P2 | campaigns_auto | CAP-190373 |
| **T3** | Option 3A Java — `coupon_event_log` + DAO + 6 write points | P1 | Luci | — |
| **T4** | Option 3B DataBricks — event_log CTEs + null-bridge | P1 | Luci/DataBricks | T3 |
| **T5** | Dracarys batchStartTime fix | P2 | dracarys | Luci §N branch |
| **T6** | `stats_series_summary` idempotency — dedup table | P2 | Luci + dracarys | T5 (partial) |
| **T7** | Consistent `snapshotDate` per request — cache staleness fix | P3 | Luci | — |

### Dependency Graph
```
CAP-190758 (root cause P1)
    ├── T1-new  DataBricks leftover clusters      [XS — independent]
    ├── T2-new  Automation SM_09–SM_12            [M  — acceptance gate for T3+T4]
    ├── T3      Java coupon_event_log             [L  — Luci]
    │    └── enables → T4
    └── T4      DataBricks event_log CTEs         [M  — DataBricks]

T5  Dracarys batchStartTime                      [S  — independent]
    └── enables cross-day dedup in → T6

T6  stats_series_summary dedup table             [M  — Luci + Dracarys]
    └── concurrent same-day dedup independent of T5

T7  Cache staleness fix                          [XS — independent]

CAP-188284 → unblocks → CAP-184614 (Redis cleanup)
```

* * *

## T1-new — Deploy DataBricks Stats Script to Leftover Clusters

| Field | Value |
|---|---|
| **Type** | Task |
| **Epic** | CAP-195120 |
| **Priority** | P2 — Medium |
| **Component** | Incentives, Luci |
| **Labels** | JAS26Planned, incentives-tech-debt |
| **Repo** | `Capillary/Luci` — `databricks/stats/luci_calculate_stats.py` |
| **Suggested Assignee** | saravanan.kesavan |

### Problem
As part of AMJ26 (CAP-195082), the Luci calculate stats DataBricks script was fixed and deployed to 5 clusters. Three clusters were explicitly skipped at closure:

| Cluster | Reason skipped | Current state |
|---|---|---|
| **USCHS** | No DataBricks access at the time | Script exists, fix not applied |
| **TATA** | Script not set up | No script at all |
| **SEACRM** | Script not set up | No script at all |

**Important finding from code investigation:** There is ONE canonical `luci_calculate_stats.py` script (PR #827), parameterised via Databricks job widgets (`server_timezone`, `run_days_offset`, `s3_bucket_name`). No per-cluster files exist. Setup for TATA/SEACRM means creating new Databricks job configs pointing to the canonical script, not creating new script files.

### Scope
1. **USCHS** — Obtain DataBricks access. Create Databricks job config pointing to canonical `luci_calculate_stats.py` with USCHS-specific widget values. Verify script runs and produces correct parquet.
2. **TATA** — Create Databricks job config with TATA-specific widget values (`server_timezone`, S3 bucket, `run_days_offset`). Trigger first run. Validate.
3. **SEACRM** — Same as TATA.

The job configs deployed must reflect **the canonical script version at time of deployment** — i.e., include all fixes that have landed (Phase 0 `created_on` swap already in the script via CAP-195082). Any subsequent fixes (T4 event_log CTEs) automatically apply to these clusters too since they share the same script.

### Acceptance Criteria
- [ ] **USCHS** — DataBricks access obtained. Job config created and deployed. First successful run confirmed.
- [ ] **TATA** — Databricks job config created with correct cluster-specific widgets. First successful run confirmed.
- [ ] **SEACRM** — Same as TATA.
- [ ] For each cluster post-deploy: trigger manual `statsHistory` run → confirm new parquet on S3 → confirm `stats_history_registry` updated → confirm `getCouponConfiguration` UC values non-negative for a known series.
- [ ] Job config parameters documented in `databricks/stats/readme.md` for all 3 clusters.

### Dependencies
| Ticket | Type |
|---|---|
| CAP-195082 | Predecessor — canonical script baseline |
| T4 | Linked — when event_log CTEs ship, these clusters get the fix automatically (shared script) |

### Notes
- USCHS access: check with infra/ops whether DataBricks workspace access has been granted since AMJ26 closure.
- Cluster-specific widget values needed: `server_timezone` (UTC offset per cluster), `s3_bucket_name` (per-cluster S3), `run_days_offset=2` (standard).

* * *

## T2-new — Automation: Reversal Flow Coverage in Luci Stats MySQL Pipeline

| Field | Value |
|---|---|
| **Type** | Task |
| **Epic** | CAP-195120 |
| **Priority** | P2 — Medium |
| **Component** | Incentives, Luci |
| **Labels** | JAS26Planned, incentives-tech-debt |
| **Repo** | `Capillary/campaigns_auto` — `tests/luci/test_stats_mysql_pipeline.py` |
| **Suggested Assignee** | Suryaprakash Palanisamy |

### Problem
The existing stats pipeline suite (SM_01–SM_08) validates the happy path only. The core bug class in CAP-190758 is triggered by **post-snapshot reversal and mutation operations** — none of which are covered by the current suite:

| Flow | Currently covered? | Why it matters |
|---|---|---|
| `revokeCoupons` after snapshot date | ⚠️ SM_03 uses revoke but NOT in post-snapshot context | Sets `active=FALSE` → `blueprint_IC=0`, Java writes `+ric` → `num_issued = −1` |
| `invalidateCoupons` on issued coupon after snapshot | ❌ Not covered | Same `active=FALSE` trigger → same negative IC |
| `invalidateCoupons` on unissued DISC_CODE_PIN after snapshot | ❌ Not covered | `is_valid=FALSE` + `last_updated_on` pushed past cutoff → `num_uploaded_nonIssued = −1` |
| `reactivateCoupon` after snapshot | ❌ Not covered | `coupon_redemptions.active=FALSE` → `num_redeemed = −1` |
| `mergeUser` / metadata update pushing `auto_update_time` past cutoff | ❌ Not covered | IC undercount without self-correction |

### New Test Cases

#### SM_09 — Revoke after snapshot: `num_issued` must not go negative
```
Setup:    Persistent monthly DISC_CODE series (STATS_RC_HIST pattern)
Day D:    Issue N coupons → wait for DataBricks ETL (blueprint_IC = N for snapshot D)
Day D+1:  Revoke 1 coupon
Assert:   api.num_issued >= 0  (NEVER negative)
Assert:   api.num_issued == DB active count (invariant)
```

#### SM_10 — invalidateCoupons on issued coupon after snapshot: `num_issued` must not go negative
```
Setup:    DISC_CODE series
Day D:    Issue N coupons → blueprint absorbs on D+2
Day D+1:  Call invalidateCoupons on 1 issued coupon
Assert:   api.num_issued >= 0
Assert:   api.num_issued == DB active count
```

#### SM_11 — invalidateCoupons on unissued DISC_CODE_PIN after snapshot: `num_uploaded_nonIssued` must not go negative
```
Setup:    DISC_CODE_PIN series, upload M codes
Day D:    Upload M codes → blueprint absorbs UC on D+2
Day D+1:  Call invalidateCoupons on 1 unissued code
Assert:   api.num_uploaded_nonIssued >= 0
Assert:   api.num_uploaded_nonIssued + api.num_issued == api.num_uploaded_total (invariant)
```

#### SM_12 — reactivateCoupon after snapshot: `num_redeemed` must not go negative
```
Setup:    DISC_CODE series
Day D:    Issue + redeem N coupons → blueprint absorbs RC on D+2
Day D+1:  Call reactivateCoupon on 1 redeemed coupon
Assert:   api.num_redeemed >= 0
Assert:   api.num_redeemed == DB active redemption count
```

### Key Invariants — shared helper `_assert_stats_invariants(series_id)`
| Invariant | Formula | Violation signal |
|---|---|---|
| No negative issued count | `num_issued >= 0` | Paradigm mismatch bug present |
| No negative redeemed count | `num_redeemed >= 0` | Paradigm mismatch bug present |
| No negative unissued count | `num_uploaded_nonIssued >= 0` | Paradigm mismatch bug present |
| Upload total conservation | `num_uploaded_nonIssued + num_issued == num_uploaded_total` | IC/UC double-count or miss |
| API == DB active count | `api.num_issued == COUNT(*) FROM coupons_issued WHERE active=1` | Blueprint corruption |

### Acceptance Criteria
- [ ] SM_09 added and passing on at least one cluster (INCRM or USCRM)
- [ ] SM_10, SM_11, SM_12 added and passing
- [ ] Shared helper `_assert_stats_invariants()` added; retrofitted into SM_01–SM_08 where applicable
- [ ] All 4 tests tagged `smoke` — run in hourly regression suite
- [ ] Tests are pre-fix aware — expected to **fail before T3+T4 ship** (add `@pytest.mark.xfail` or `isDbStatsReadEnabled` skip gate so they don't block CI on unfixed clusters)
- [ ] Tests pass after T3+T4 activate `event_log_enabled=true` — serves as the acceptance gate

### Dependencies
| Ticket | Type |
|---|---|
| CAP-190373 | Predecessor — stats regression investigation; same file, same assignee |
| T3 + T4 | These tests are the acceptance gate — must pass after T3+T4 ship |
| CAP-190758 | Context |

### Notes for Implementer
- Follow SM_03's cross-day pattern — use persistent monthly series (`STATS_GEN_YYYY_MM` prefix).
- Reversal must happen on Day D+1 — after blueprint has absorbed Day D events.
- Poll `stats_history_registry` to confirm blueprint advanced past Day D before executing reversal (DataBricks gate — reuse from SM_01).
- Pre-fix: SM_09–SM_12 will fail on `num_issued >= 0` assertion. This is intentional — they are the bug detector AND the fix gate.

* * *

## T3 — Option 3A (Java): `coupon_event_log` Table, DAO, and 6 Event Write Points

| Field | Value |
|---|---|
| **Type** | Story |
| **Epic** | CAP-195120 |
| **Priority** | P1 — High |
| **Component** | Incentives, Luci |
| **Labels** | JAS26Planned, incentives-okr |
| **Repo** | `Capillary/Luci` |
| **Suggested Assignee** | saravanan.kesavan |

### Background
CAP-190758 identified two incompatible counting models:
- **DataBricks**: current-state query (`active=TRUE`, `auto_update_time ≤ cutoff`)
- **Java**: event-log delta (`+1` on issue, `−1` on revoke in `stats_series_summary`)

Any post-snapshot reversal makes the blueprint exclude a coupon Java has already counted → **negative `num_issued` / `num_redeemed`**.

Option 3 replaces the current-state model in DataBricks with pure event-time arithmetic driven by a new `coupon_event_log` table written by Java. This is **Part A — the Java side**.

**Code investigation confirmed:** `CouponEventLogDao`, `CouponEventLogEntity`, and `coupon_event_log` table do **not yet exist** — clean greenfield. The `created_on` field changes from §N of the analysis doc are on a **local branch, not yet merged to master** — coordinate branch merge before starting.

### Scope

#### 1. New MySQL Table
```sql
CREATE TABLE luci.coupon_event_log (
  id            BIGINT NOT NULL AUTO_INCREMENT,
  org_id        INT NOT NULL,
  series_id     INT NOT NULL,
  coupon_code   VARCHAR(200) NOT NULL,
  event_type    ENUM(
                  'UPLOADED', 'ISSUED', 'REDEEMED',
                  'REVOKED', 'INVALIDATED_UNISSUED', 'REACTIVATED'
                ) NOT NULL,
  event_time    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY idx_snapshot_query (org_id, series_id, event_type, event_time)
) PARTITION BY RANGE (UNIX_TIMESTAMP(event_time)) ( ... );
```
- Add to `src/test/integration/resources/h2-schema.sql`
- DBA provisioning required on all prod clusters before deploy

#### 2. New Entity + DAO
- `CouponEventLogEntity.java` — fields: id, orgId, seriesId, couponCode, eventType (enum), eventTime
- `CouponEventLogDao.java` — `writeEvent()` (single) + `writeEvents()` (batch for uploads)
- `CouponEventLogDaoImpl.java` — **write failure must NOT propagate to caller** (try/catch, log warn, continue — event log is auxiliary)

#### 3. Six Event Write Points

| # | Location | File:Line | Event |
|---|---|---|---|
| W1 | `notifyCouponsUploadRequest` — NOT_TAGGED upload | `LuciThriftServiceImpl.java:1733` | `UPLOADED` per code |
| W2 | `notifyCouponsUploadRequest` — USER_ID branch | `LuciThriftServiceImpl.java:1752` | `UPLOADED` + `ISSUED` per code (simultaneously) |
| W3 | `revokeCoupon` | `CouponDaoImpl.java:701` | `REVOKED` per revoked code |
| W4 | `invalidateCoupons` on **issued** coupon | `CouponRuntimeServiceImpl.java:1214` | `REVOKED` (same IC formula — both subtract 1 from IC) |
| W5 | `invalidateCoupons` on **unissued** DISC_CODE_PIN | `CouponRuntimeServiceImpl.java:1263` | `INVALIDATED_UNISSUED` |
| W6 | `reactivateCoupon` → `invalidateRedemption` | `CouponRedemptionDaoImpl.java:488` | `REACTIVATED` |

> **Note on W4 semantics:** `invalidateCoupons` on issued coupon writes `REVOKED` (not `INVALIDATED_ISSUED`) because the DataBricks IC formula treats both identically: `IC = ISSUED − REVOKED`. If future audit requirements need distinction, extend the ENUM — formula unchanged.

#### 4. `event_time` Convention
Use `new Date()` (server JVM clock) — consistent with `auto_update_time` frame of reference. For batch uploads (W1/W2): use `batchStartTime` from `UploadCouponEntity.getCreatedOn()` once T5 ships; until then fall back to `new Date()`.

#### 5. Feature Flag
```
coupon.event.log.enabled=true  (default: false)
```
Read via `CouponConfigKeyService`. Per-org enablement before T4 is ready. Safe rollback by flipping flag.

#### 6. Retention Cleanup Job
```sql
-- Nightly @Scheduled job:
DELETE FROM luci.coupon_event_log
WHERE event_time < NOW() - INTERVAL 30 DAY
  AND event_type IN ('ISSUED','REDEEMED','REVOKED','REACTIVATED')
LIMIT 10000;  -- batched to avoid lock pressure
```
`UPLOADED` and `INVALIDATED_UNISSUED` events retain until series deactivated (longer TTL, controlled separately).

### Pre-conditions to Verify Before Build
| # | Assumption | Verification needed |
|---|---|---|
| A1 | `issued_date` on `coupons_issued` never overwritten by resend/re-issue | Grep `CouponRuntimeServiceImpl` issue flow for any `issued_date` UPDATE |
| A2 | `redeemed_date` on `coupon_redemptions` immutable on reversal (Q1/Q5) | Check schema + `invalidateRedemption` flow |
| A3 | No bulk revoke/batch invalidate path bypasses `CouponDaoImpl.revokeCoupon` | Grep all `UPDATE coupons_issued SET active` across Luci and member-care (Q2) |
| A4 | `created_on` branch (§N) not yet on master | Confirm branch name + merge status before starting to avoid DAO conflict |

### Acceptance Criteria
- [ ] `luci.coupon_event_log` DDL created + added to `h2-schema.sql`
- [ ] `CouponEventLogEntity`, `CouponEventLogDao`, `CouponEventLogDaoImpl` implemented
- [ ] All 6 write points (W1–W6) implemented, wrapped in try/catch (non-blocking)
- [ ] Feature flag `coupon.event.log.enabled` gates all writes; default=false
- [ ] Unit tests for each write point — assert correct `event_type` and `event_time`
- [ ] Integration test: seed known events → assert IC/RC/UC formula produces correct values
- [ ] Nightly retention cleanup job implemented and tested
- [ ] DBA sign-off on partition scheme before prod deploy
- [ ] Does NOT require T4 to be live — write points are inert until T4 switches DataBricks to read this table

### Dependencies
| Ticket | Type |
|---|---|
| CAP-190758 | Parent problem |
| **T4** | T3 must deploy first; T4 deploys same day or after (null bridge handles all pre-T3 history) |
| **T5** | Soft — T5 improves `event_time` accuracy for upload events; T3 ships independently |
| **T2-new** | SM_09–SM_12 are the acceptance gate |
| CAP-191062 | Linked — W2 write point fixes DataBricks blueprint UTC gap that CAP-191062 doesn't cover |

### Risks
| Risk | Mitigation |
|---|---|
| Write failure blocks main transaction | try/catch on all writes; log warn; never re-throw |
| Missing bulk revoke/batch invalidate path | Q2 grep required before build |
| `created_on` branch conflict with DAO changes | Coordinate branch merge order before starting |
| High write throughput for large orgs | Batch INSERT for upload flows (W1/W2); partitioned table |

* * *

## T4 — Option 3B (DataBricks): `coupon_event_log` CTEs + Null-Bridge

| Field | Value |
|---|---|
| **Type** | Story |
| **Epic** | CAP-195120 |
| **Priority** | P1 — High |
| **Component** | Incentives, Luci |
| **Labels** | JAS26Planned, incentives-okr |
| **Repo** | `Capillary/Luci` — `databricks/stats/luci_calculate_stats.py` |
| **Suggested Assignee** | saravanan.kesavan |

### Background
T3 writes `coupon_event_log`. This ticket replaces the 4 mutable-state CTEs in `luci_calculate_stats.py` with pure event-time arithmetic driven by `coupon_event_log`, eliminating the paradigm mismatch.

**Single file:** `databricks/stats/luci_calculate_stats.py` (one canonical script, all clusters share it via Databricks job widget parameters — confirmed by code investigation).

**`source_delta.luci__coupon_event_log` does not yet exist** — ETL/data-eng team must onboard it.

### Scope

#### ETL Pre-requisite — `coupon_event_log` Delta Sync
- Table: `luci.coupon_event_log` → Delta table: `source_delta.luci__coupon_event_log`
- Sync mode: **CDC / incremental merge** (NOT full daily overwrite — full overwrite wipes the history the null bridge depends on)
- `logRetentionDuration`: ≥ 30 days
- Raise with ETL team as soon as T3 is in dev

#### New Event-Log CTEs (replace lines 55–129)

**`el_ic` CTE — replaces `issued` CTE (line 55):**
```sql
el_ic AS (
  SELECT ci.org_id, ci.coupon_series_id,
         COUNT(*) AS issued_count,
         MAX(TIMESTAMP_MILLIS(ci.issued_date)) AS last_issued_time
  FROM source_delta.luci__coupons_issued ci
  LEFT JOIN source_delta.luci__coupon_event_log el
    ON ci.org_id = el.org_id AND ci.coupon_series_id = el.series_id
    AND ci.coupon_code = el.coupon_code AND el.event_type = 'ISSUED'
  WHERE (el.event_time IS NULL                                  -- null = pre-T3 = always before cutoff
         OR TIMESTAMP_MILLIS(el.event_time) <= TIMESTAMP('{cutoff_end_utc}'))
    AND ci.active = TRUE
  GROUP BY ci.org_id, ci.coupon_series_id
),
```

**`el_rc` CTE — replaces `redeemed` CTE (line 75):**
```sql
el_rc AS (
  SELECT cr.org_id, cr.coupon_series_id,
         COUNT(*) AS redeemed_count,
         MAX(TIMESTAMP_MILLIS(cr.redeemed_date)) AS last_redeemed_time,
         COALESCE(cr.redemption_config_entity, 'SERIES') AS redemption_config_entity,
         CASE WHEN cr.redemption_config_entity_id IS NULL
               OR cr.redemption_config_entity_id = 0 THEN -1
              ELSE cr.redemption_config_entity_id END AS redemption_config_entity_id
  FROM source_delta.luci__coupon_redemptions cr
  LEFT JOIN source_delta.luci__coupon_event_log el
    ON cr.org_id = el.org_id AND cr.coupon_series_id = el.series_id
    AND cr.coupon_code = el.coupon_code AND el.event_type = 'REDEEMED'
  WHERE (el.event_time IS NULL
         OR TIMESTAMP_MILLIS(el.event_time) <= TIMESTAMP('{cutoff_end_utc}'))
    AND cr.active = TRUE
  GROUP BY cr.org_id, cr.coupon_series_id,
    COALESCE(cr.redemption_config_entity, 'SERIES'),
    CASE WHEN cr.redemption_config_entity_id IS NULL
          OR cr.redemption_config_entity_id = 0 THEN -1
         ELSE cr.redemption_config_entity_id END
),
```

**`el_uc` CTE — replaces `created` + `unissued` CTEs (lines 102–129):**
```sql
el_uc AS (
  SELECT cc.org_id, cc.coupon_series_id,
         COUNT(CASE WHEN CAST(cc.is_valid AS INT) = 1 THEN 1 END) AS created_valid,
         COUNT(CASE WHEN CAST(cc.is_valid AS INT) = 1
                     AND ci.coupon_code IS NULL THEN 1 END) AS unissued_count
  FROM source_delta.luci__coupons_created cc
  LEFT JOIN source_delta.luci__coupon_event_log el_cc
    ON cc.org_id = el_cc.org_id AND cc.coupon_series_id = el_cc.series_id
    AND cc.coupon_code = el_cc.coupon_code AND el_cc.event_type = 'UPLOADED'
  LEFT JOIN source_delta.luci__coupons_issued ci
    ON cc.org_id = ci.org_id AND cc.coupon_series_id = ci.coupon_series_id
    AND cc.coupon_code = ci.coupon_code
  LEFT JOIN source_delta.luci__coupon_event_log el_ci
    ON ci.org_id = el_ci.org_id AND ci.coupon_series_id = el_ci.series_id
    AND ci.coupon_code = el_ci.coupon_code AND el_ci.event_type = 'ISSUED'
  WHERE (el_cc.event_time IS NULL
         OR TIMESTAMP_MILLIS(el_cc.event_time) <= TIMESTAMP('{cutoff_end_utc}'))
  GROUP BY cc.org_id, cc.coupon_series_id
),
```

#### Null Bridge — How It Works
```
Pre-T3 coupon  →  no row in coupon_event_log  →  el.event_time IS NULL
               →  NULL condition = TRUE  →  coupon INCLUDED  ✅

Post-T3 coupon →  row exists with event_time = issue time
               →  event_time <= cutoff checked  →  included only if before cutoff  ✅
```
**No backfill required.** T4 can deploy the same day as T3 — all pre-T3 coupons covered by the null bridge.

#### Feature Flag
```python
# New widget in notebook:
dbutils.widgets.text("event_log_enabled", "false")
event_log_enabled = dbutils.widgets.get("event_log_enabled") == "true"

# Python conditional selects CTE block:
cte_sql = EVENT_LOG_CTE_BLOCK if event_log_enabled else LEGACY_CTE_BLOCK
```
Rollback = flip widget to `"false"` in Databricks job config. No notebook redeploy needed.

#### What Does NOT Change
| Component | Status |
|---|---|
| `issued_meta` CTE (line 67) — `max_auto_update_time_issued` | Unchanged — operational ETL marker, not a count input |
| UTC conversion line 39 (intentionally disabled) | Unchanged — intentional design per `luci_timezone_descrepency.txt` |
| S3 parquet write logic (lines 190–218) | Unchanged |
| All downstream Java (`ParquetFileService`, `StatsHistoryServiceImpl`) | Unchanged — parquet schema unchanged |

#### Backfill Run After First Enable
When `event_log_enabled=true` first activated for a cluster:
- Run notebook manually for last `run_days_offset + 2` = 4 days minimum
- Existing wrong blueprints do NOT self-correct — backfill is required
- Verify using post-deploy validation query (see Acceptance Criteria)

### Pre-conditions
| # | Pre-condition | Owner |
|---|---|---|
| P1 | T3 deployed; `coupon_event_log` has ≥ 1 ETL cycle of data | Luci Java team |
| P2 | `source_delta.luci__coupon_event_log` CDC sync active | ETL/data-eng team |
| P3 | `event_log_enabled` widget added to all cluster Databricks job configs (default=false) | DataBricks team |
| P4 | Q6 resolved — confirm `source_delta` tables are CDC-synced (not full-refresh) | ETL/data-eng team |

### Acceptance Criteria
- [ ] New CTEs `el_ic`, `el_rc`, `el_uc` added to `luci_calculate_stats.py`
- [ ] Legacy CTEs retained under `event_log_enabled=false` path for safe rollback
- [ ] `event_log_enabled` widget added; all cluster job configs default `false`
- [ ] `source_delta.luci__coupon_event_log` Delta sync active (ETL team sign-off)
- [ ] Null bridge validated: pre-T3 `coupons_issued` row with no `event_log` entry → appears in `el_ic` count
- [ ] Post-enable validation query passes for each cluster:
```sql
SELECT b.org_id, b.coupon_series_id, b.value AS blueprint_ic, d.db_count
FROM stats_history_blueprint_<YYYYMMDD> b
JOIN (SELECT org_id, coupon_series_id, COUNT(*) AS db_count
      FROM coupons_issued WHERE active = 1
      GROUP BY org_id, coupon_series_id) d
  USING (org_id, coupon_series_id)
WHERE b.stats_key = 'ic' AND b.value != d.db_count;
-- Zero rows = fix confirmed
```
- [ ] SM_09–SM_12 (T2-new) pass after `event_log_enabled=true` activated
- [ ] Backfill runbook documented and executed per cluster at activation
- [ ] Rollback verified: `event_log_enabled=false` restores legacy CTE behaviour

### Dependencies
| Ticket | Type |
|---|---|
| **T3** | Hard prerequisite — must deploy before T4 goes live |
| **T2-new** | Acceptance gate |
| **T1-new** | USCHS/TATA/SEACRM automatically covered (shared script) |

### Risks
| Risk | Mitigation |
|---|---|
| `source_delta.luci__coupon_event_log` full-refresh (not CDC) | Null bridge loses history on each sync → confirm CDC (P4) before T4 dev starts |
| T4 deployed before T3 has run ≥ 1 ETL cycle | All `el.event_time IS NULL` → all coupons counted as pre-cutoff → inflated counts. Enforce deploy order. |
| Backfill not run at `event_log_enabled=true` activation | Stale wrong blueprints persist. Backfill runbook must execute at activation. |
| `redeemed_date` null for some `coupon_redemptions` rows | Add `COALESCE(redeemed_date, auto_update_time)` in `el_rc` CTE (Q1/Q5) |

* * *

## T5 — Dracarys: batchStartTime Fix (`uploadedOn` + `createdOn` Canonicalisation)

| Field | Value |
|---|---|
| **Type** | Story |
| **Epic** | CAP-195120 |
| **Priority** | P2 — Medium |
| **Component** | Incentives, Dracarys |
| **Labels** | JAS26Planned, incentives-tech-debt |
| **Repo** | `Capillary/dracarys` |
| **Suggested Assignee** | Dracarys team |

### Background
Two problems stem from the same root — **no canonical stable timestamp for when a batch started**:

| Field | Current value | Problem |
|---|---|---|
| `uploadedOn` in `NotifyCouponsUploadRequest` (`PostUploadConvertorImpl.java:34`) | `System.currentTimeMillis()` — notification **send** time | Cross-day retry (Day D+2): `uploadedOn = D+2` → Luci writes `summary_date = D+2` → double-count with blueprint |
| `created_on` in `coupons_created` rows (`BaseCouponUploaderImpl.java:75`) | `new Date()` at INSERT time | Codes inserted across midnight get split `created_on` dates → DataBricks splits batch across two cutoff windows → transient 24h overcount |
| `requestId` (`PostUploadConvertorImpl.java:38`) | `"post_upload_" + ... + new Date().getTime()` | Changes on every retry — cannot be used for dedup |

**Fix:** Use `UploadCouponEntity.createdOn` (Q15 confirmed: set once at `ContextMapper.java:80`, persisted to DB, retry-stable) as canonical `batchStartTime`.

**Luci-side pre-requisite:** §N implementation already done on local branch. Must merge to master and deploy before Dracarys deploys.

### Scope — 4 Changes in `Capillary/dracarys`

#### Change 1 — `PostUploadConvertorImpl.java:34`
```java
// BEFORE:
.setUploadedOn(System.currentTimeMillis())

// AFTER:
.setUploadedOn(uploadCouponEntity.getCreatedOn() != null
    ? uploadCouponEntity.getCreatedOn().getTime()
    : System.currentTimeMillis())  // null-safe fallback
```

#### Change 2 — `BaseCouponUploaderImpl.java:68`
```java
// Add batchStartTime param to constructCouponCreatedEntities():
protected List<CouponsCreatedEntity> constructCouponCreatedEntities(
    List<TempTableEntity> validRows, boolean isQueued, Date batchStartTime)

// Inside loop, after setLastUpdatedOn(new Date()):
if (batchStartTime != null) { tempEntity.setCreatedOn(batchStartTime); }
```

#### Change 3 — `NonCustomerTaggedUploader.java:84`
```java
// Thread batchStartTime from context:
Date batchStartTime = couponUploadContext.getUploadCouponEntity().getCreatedOn();
List<CouponsCreatedEntity> entities =
    constructNonTaggedCouponCreatedEntities(tempTableEntityList, batchStartTime);
```

#### Change 4 — `CustomerTaggedUploader.java:570`
```java
// In constructGeneratedCouponCreatedEntities(), add at method start:
Date batchStartTime = couponUploadContext.getUploadCouponEntity().getCreatedOn();
// Inside loop, after setValid(true):
if (batchStartTime != null) { couponCreated.setCreatedOn(batchStartTime); }
```

### Luci-side Pre-requisite (§N — already implemented, pending merge)
| File | Change | Status |
|---|---|---|
| `CouponsCreatedEntity.java` | `createdOn` field + getter/setter | ✅ Implemented on local branch |
| `CouponsCreatedDaoImpl.java` | `created_on` in valueMap when non-null | ✅ Implemented |
| `StatsSeriesSummaryDao.java` | New overload with `summaryDate` param | ✅ Implemented |
| `LuciThriftServiceImpl.java` | Uses `uploadedOn` as `summaryDate` for TC/UC/UTC writes | ✅ Implemented |

**Deploy order:** Luci first (or simultaneous). If Luci deploys first with `createdOn=null`, MySQL DEFAULT applies — no regression.

### Combined Impact Once Both Live
| Problem | Status |
|---|---|
| Midnight batch split-attribution overcount (§H) | ✅ Eliminated |
| DataBricks cutoff split across batch boundary | ✅ Eliminated |
| Cross-day late-notify double-count (§M Option 2) | ✅ Eliminated |
| Concurrent same-day retry double-count | ❌ Requires dedup table (T6) |

### Acceptance Criteria
- [ ] `PostUploadConvertorImpl.java:34` — `uploadedOn` uses `getCreatedOn().getTime()` with null fallback
- [ ] `BaseCouponUploaderImpl.constructCouponCreatedEntities()` — accepts `batchStartTime`, sets `createdOn` when non-null
- [ ] `NonCustomerTaggedUploader.commitToCreated()` — threads `batchStartTime` from `UploadCouponEntity.getCreatedOn()`
- [ ] `CustomerTaggedUploader.constructGeneratedCouponCreatedEntities()` — sets `createdOn = batchStartTime`
- [ ] Unit test: same `UploadCouponEntity` called twice → `uploadedOn` identical both times
- [ ] Unit test: midnight batch → all entities have same `createdOn = batchStartTime`
- [ ] Integration test: cross-day retry → no duplicate `summary_date` rows in `stats_series_summary`
- [ ] Backward compatibility: `createdOn=null` → `System.currentTimeMillis()` fallback, no regression
- [ ] `constructCouponCreatedEntities` all existing 2-arg callers updated (compile-time check)

### Dependencies
| Ticket | Type |
|---|---|
| **Luci §N branch** | Hard prerequisite — merge and deploy before Dracarys |
| **T6** | T5 is prerequisite for T6's cross-day dedup path |

### Risks
| Risk | Mitigation |
|---|---|
| Dracarys deploys before Luci §N branch is merged | `createdOn` ignored by Luci DAO → MySQL DEFAULT → no regression, no benefit yet |
| `UploadCouponEntity.createdOn` null for in-flight jobs at deploy | Null-safe fallback covers this |
| `constructCouponCreatedEntities` other callers break | Compile error will catch all call sites |

* * *

## T6 — `stats_series_summary` Idempotency: `coupon_upload_notify_log` Dedup Table

| Field | Value |
|---|---|
| **Type** | Story |
| **Epic** | CAP-195120 |
| **Priority** | P2 — Medium |
| **Component** | Incentives, Luci |
| **Labels** | JAS26Planned, incentives-tech-debt |
| **Repos** | `Capillary/Luci` (primary), `Capillary/dracarys` (pass `uploadJobId`), `thrift-ifaces-luci` (optional field) |
| **Suggested Assignee** | Luci Java team |

### Background
Every write to `stats_series_summary` is a delta-add (`value = value + delta`). Three Dracarys-triggered paths have **no idempotency guard** — vulnerable to double-counting on retry:

| Write path | File:Line | Fields | Retry risk |
|---|---|---|---|
| `notifyCouponsUploadRequest` — NOT_TAGGED | `LuciThriftServiceImpl.java:1734-1736` | TC, UC, UTC | 🔴 HIGH — `requestId` at line 37-39 is `new Date().getTime()`, changes every call, never checked |
| `DracarysCouponSeriesStatisticsService.addIssuedCount` | `DracarysCouponSeriesStatisticsService.java:34` | IC | 🔴 HIGH — no guard; called directly by `CustomerTaggedUploader.preCommitOperations():94` |
| `DracarysCouponSeriesStatisticsService.addRedeemedCount` | `DracarysCouponSeriesStatisticsService.java:56` | RC | 🔴 HIGH — same pattern |

**Double-count mechanism (cross-day late retry):**
```
Day D:    notify call fails → stats_series_summary NOT written
Day D+2:  ETL ran; blueprint_UC = N
          Retry notify → summaryDate = D+2, uc = +N  (second write)
Assembly: blueprint_UC(N) + window_UC(D+2)(+N) = 2N  ← DOUBLE COUNT
Self-corrects: when next ETL cutoff ≥ D+2 → D+2 row absorbed into blueprint
```
**Same-day concurrent retry:** Two threads, each calling `addIssuedCount(+100)` → IC += 200. Does NOT self-correct. Affects limit enforcement (false "limit exhausted" rejections).

### Scope

#### Part A — New Dedup Table
```sql
CREATE TABLE luci.coupon_upload_notify_log (
  id             BIGINT NOT NULL AUTO_INCREMENT,
  org_id         INT NOT NULL,
  series_id      INT NOT NULL,
  upload_job_id  BIGINT NOT NULL,
  call_type      ENUM('NOTIFY_UPLOAD', 'ADD_ISSUED', 'ADD_REDEEMED') NOT NULL,
  processed_at   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uq_dedup (org_id, series_id, upload_job_id, call_type)
) ENGINE=InnoDB;
```
Add to `h2-schema.sql`. Nightly TTL cleanup: `DELETE WHERE processed_at < NOW() - INTERVAL 30 DAY LIMIT 10000`.

#### Part B — `uploadJobId` field in `NotifyCouponsUploadRequest` Thrift struct
**Source of stable ID:** `UploadCouponEntity.id` — DB primary key, confirmed Q12 (survives restarts, retry-stable).
```thrift
// thrift-ifaces-luci — add optional field (backward compatible):
optional i64 uploadJobId
```

#### Part C — Dedup guard in 3 Luci write paths

**Path 1 — `LuciThriftServiceImpl.java:1708` (`notifyCouponsUploadRequest`):**
```java
Long uploadJobId = notifyCouponsUploadRequest.isSetUploadJobId()
    ? notifyCouponsUploadRequest.getUploadJobId() : null;
if (uploadJobId != null) {
    int inserted = couponUploadNotifyLogDao.insertIfAbsent(
        orgId, couponSeriesId, uploadJobId, CallType.NOTIFY_UPLOAD);
    if (inserted == 0) { return; }  // idempotent no-op
}
// ... existing TC/UC/UTC writes
```

**Path 2 — `DracarysCouponSeriesStatisticsService.java:28` (`addIssuedCount`):**
```java
public long addIssuedCount(int orgId, int couponSeriesId,
                           long issuedCount, Long uploadJobId) throws LuciException {
    if (uploadJobId != null) {
        int inserted = couponUploadNotifyLogDao.insertIfAbsent(
            orgId, couponSeriesId, uploadJobId, CallType.ADD_ISSUED);
        if (inserted == 0) { return 0L; }
    }
    // ... existing DB + Redis writes
}
```

**Path 3 — `DracarysCouponSeriesStatisticsService.java:50` (`addRedeemedCount`):** Same pattern with `CallType.ADD_REDEEMED`.

#### Part D — New DAO: `CouponUploadNotifyLogDao`
```java
// INSERT IGNORE → returns 1 (first call) or 0 (duplicate):
int insertIfAbsent(int orgId, int seriesId, long uploadJobId, CallType callType);
```
```java
// Implementation:
return jdbcTemplate.update(
    "INSERT IGNORE INTO luci.coupon_upload_notify_log " +
    "(org_id, series_id, upload_job_id, call_type) VALUES (?, ?, ?, ?)",
    orgId, seriesId, uploadJobId, callType.name());
```

#### Part E — Dracarys: pass `uploadJobId` at 2 call sites
- `PostUploadConvertorImpl.java:34` — add `.setUploadJobId(uploadCouponEntity.getId())`
- `CustomerTaggedUploader.preCommitOperations():94` — pass `uploadCouponEntity.getId()` to `addIssuedCount()`

### Deploy Order
```
1. Luci deploys first:
   - coupon_upload_notify_log table (DBA provision)
   - CouponUploadNotifyLogDao + Impl
   - Guards in LuciThriftServiceImpl + DracarysCouponSeriesStatisticsService
   - thrift-ifaces-luci updated (optional field)

2. Dracarys deploys after Luci is live:
   - Pass uploadJobId in PostUploadConvertorImpl
   - Pass uploadJobId in CustomerTaggedUploader.preCommitOperations()

Until Dracarys deploys: uploadJobId = null → dedup guard skipped → no regression
```

### What This Fixes
| Problem | Status after T6 |
|---|---|
| Cross-day late-notify double-count | ✅ Fixed |
| Concurrent same-day retry double-count | ✅ Fixed (`INSERT IGNORE` is atomic) |
| `addIssuedCount` duplicate on tagged upload retry | ✅ Fixed |
| Single-coupon issue/redeem/revoke writes (non-Dracarys) | ⚠️ Not covered — LOW risk (at-most-once RabbitMQ) |

### Acceptance Criteria
- [ ] `luci.coupon_upload_notify_log` DDL + `h2-schema.sql`
- [ ] `CouponUploadNotifyLogDao` + `Impl` with `insertIfAbsent()` implemented
- [ ] `uploadJobId` optional field in `NotifyCouponsUploadRequest` Thrift struct
- [ ] Dedup guard in `notifyCouponsUploadRequest`, `addIssuedCount`, `addRedeemedCount`
- [ ] Dracarys: passes `uploadCouponEntity.getId()` as `uploadJobId` in both call sites
- [ ] Unit test: same `uploadJobId` twice → second call returns immediately, no second DB write
- [ ] Unit test: different `uploadJobId` values → both writes proceed
- [ ] Integration test: cross-day retry → no duplicate delta row in `stats_series_summary`
- [ ] Integration test: concurrent same-day retry → IC incremented exactly once
- [ ] Nightly TTL cleanup job implemented
- [ ] Backward compatibility: `isSetUploadJobId() = false` → guard skipped silently

### Dependencies
| Ticket | Type |
|---|---|
| **T5** | Soft — T5 fixes cross-day double-count via `summaryDate`; T6 adds hard dedup guard. Can ship independently. |
| `thrift-ifaces-luci` | New optional field — coordinate with thrift release cycle |

### Risks
| Risk | Mitigation |
|---|---|
| `INSERT IGNORE` fails (DB down) | Wrap in try/catch; fail-open (proceed with stat write) — better a duplicate than a missed count |
| Dracarys deploys before Luci thrift struct updated | `isSetUploadJobId() = false` → dedup skipped → no regression |
| TTL cleanup deletes row before retry after 30 days | Retry proceeds as first call — correct behaviour |

* * *

## T7 — Consistent `snapshotDate` Per Request: Cache Staleness Fix in Limit Enforcement Processors

| Field | Value |
|---|---|
| **Type** | Task |
| **Epic** | CAP-195120 |
| **Priority** | P3 — Low |
| **Component** | Incentives, Luci |
| **Labels** | JAS26Planned, incentives-tech-debt |
| **Repo** | `Capillary/Luci` |
| **Suggested Assignee** | Luci Java team |

### Background
Two limit-enforcement processors combine `getHistoricalValue()` (cached, date-prefix key) and `getMysqlSummaryValue()` (uncached, fresh registry lookup) in one formula. When `statsHistory` advances the registry mid-day, these two calls anchor to **different snapshots within the same request**:

```
@CacheEvict on StatsHistoryRegistryDaoImpl.save():
  Evicts:     statsRegistryKey:{orgId}          ✓
  Does NOT evict: statsHistoryPerKey:...         ✗

→ getMysqlSummaryValue() sees new snapshot S2 (startDate = S2+1 = Jun17)
→ getHistoricalValue()   still cached with S1   (covers through Jun16)
→ RRC on Jun16: in getHistoricalValue window BUT NOT in getMysqlSummaryValue window
→ Net RC over-counted → false "max redemption reached" → valid redemption BLOCKED
```

**Duration:** Until midnight (date-prefix key rotates → natural cache miss).
**Severity:** LOW in practice — RRC/RIC events are rare.

### Blast Radius — 2 Processors Affected (wider than original doc stated)

| Processor | File | Lines | Formula | Effect |
|---|---|---|---|---|
| `MaxRedemptionForSeriesProcessor` | `MaxRedemptionForSeriesProcessor.java` | 124–129 (commit), 138–143 (non-commit) | `RC + getHistoricalValue(RC) − getMysqlSummaryValue(RRC)` | False "max redemption reached" |
| `SeriesLockAcquireProcessor` | `SeriesLockAcquireProcessor.java` | 62–67 | `IC + getHistoricalValue(IC) − getMysqlSummaryValue(RIC)` | False "max issuance reached" |
| `NotifyLimitExhaustedService` | `NotifyLimitExhaustedService.java` | 202–205 | `IC + getHistoricalValue(IC)` (no subtraction) | **Not affected** — no RIC subtraction |

> **Note:** `SeriesLockAcquireProcessor` was NOT in the original analysis document. This ticket expands the fix scope to cover both affected processors.

### Fix — Consistent Registry Per Request (5–10 lines per processor)

Fetch `findLastActiveByOrgId()` **once** at the top of each affected method. Pass `snapshotDate` to both calls.

**`StatsHistoryServiceImpl` — new `snapshotDate`-aware overloads (additive, existing callers unchanged):**
```java
// getHistoricalValue overload — bypasses @Cacheable (caller owns registry):
public long getHistoricalValue(Integer orgId, Integer couponSeriesId,
        CouponSeriesStatisticsField key, String entityType, long entityId,
        Date snapshotDate) {
    if (snapshotDate == null)
        return getHistoricalValue(orgId, couponSeriesId, key, entityType, entityId);
    // use snapshotDate directly — no cache lookup needed
    Date endDate = DateUtil.addDaysToDate(snapshotDate, 0);
    Date startDate = DateUtil.addDaysToDate(snapshotDate, 1);
    return bulkSumValuesForDateRange(orgId, couponSeriesId, key, entityType, entityId, startDate, endDate);
}

// getMysqlSummaryValue overload — uses caller-supplied snapshotDate:
public long getMysqlSummaryValue(Integer orgId, Integer couponSeriesId,
        CouponSeriesStatisticsField key, String entityType, long entityId,
        Date snapshotDate) {
    if (snapshotDate == null)
        return getMysqlSummaryValue(orgId, couponSeriesId, key, entityType, entityId);
    Date startDate = DateUtil.addDaysToDate(snapshotDate, 1);
    Date endDate = new Date();
    return statsSeriesSummaryDao.bulkSumValuesForDateRange(
        orgId, couponSeriesId, key, entityType, entityId, startDate, endDate);
}
```

**`MaxRedemptionForSeriesProcessor.getRedeemedCouponCount()` — fetch registry once:**
```java
// ADD at top of method, before commit/non-commit branch:
StatsHistoryRegistryEntity registry =
    statsHistoryRegistryDao.findLastActiveByOrgId(ctx.getCouponSeriesConfig().getOrgId());
Date snapshotDate = (registry != null && registry.getSnapShotDate() != null)
    ? registry.getSnapShotDate() : null;

// Pass snapshotDate to BOTH calls at lines 124-129 AND 138-143:
long historicalValue   = statsHistoryService.getHistoricalValue(..., snapshotDate);
long historicalRRCValue = statsHistoryService.getMysqlSummaryValue(..., snapshotDate);
```

**`SeriesLockAcquireProcessor` — same pattern at lines 62–67.**

### What Does NOT Change
| Component | Status |
|---|---|
| `NotifyLimitExhaustedService` | Unaffected — uses `getHistoricalValue()` only |
| `@Cacheable` on existing `getHistoricalValue()` | Unchanged — existing callers still benefit from caching |
| `@CacheEvict` on `StatsHistoryRegistryDaoImpl.save()` | Unchanged — this fix bypasses the staleness rather than fixing the eviction |
| Redis path orgs (`isDbStatsReadEnabled=false`) | Unaffected |

**Note on net DB calls:** `findLastActiveByOrgId()` is already called inside `getMysqlSummaryValue()`. Hoisting it to the top of the method removes it from inside the method — net extra DB calls = 0.

### Acceptance Criteria
- [ ] `StatsHistoryServiceImpl` — new `snapshotDate`-aware overloads for `getHistoricalValue()` and `getMysqlSummaryValue()`; existing overloads unchanged
- [ ] `MaxRedemptionForSeriesProcessor.getRedeemedCouponCount()` — registry fetched once at top; `snapshotDate` passed consistently to both calls in commit path (lines 124–129) and non-commit path (lines 138–143)
- [ ] `SeriesLockAcquireProcessor` — same pattern at lines 62–67
- [ ] Unit test: simulate registry advance mid-day → formula uses same `snapshotDate` for both calls in single invocation
- [ ] Unit test: `snapshotDate=null` fallback → both overloads delegate to existing behaviour
- [ ] Integration test: `reactivateCoupon` event on gap date → `getRedeemedCouponCount` does not over-count after registry advance
- [ ] `NotifyLimitExhaustedService` — no regression (uses original non-`snapshotDate` overload)
- [ ] Confirm Q10 before closing: query `rrc > 0` rows on DB-stats-read orgs to validate severity

### Dependencies
| Ticket | Type |
|---|---|
| CAP-190758 | Context |
| Q10 | Confirm `reactivateCoupon` frequency on DB-stats-read orgs before elevating priority |

### Risks
| Risk | Mitigation |
|---|---|
| `endDate` computation difference between old and new overload | Add explicit unit tests comparing old vs new path output for same inputs |

* * *

## Summary: All Open Questions

| # | Question | Urgency | Owner |
|---|----------|---------|-------|
| **Q1/Q5** | Is `redeemed_date` on `coupon_redemptions` always populated? Immutable on reversal? | 🔴 High — blocks T3/T4 RC fix | Developer |
| **Q2** | Full list of ops updating `coupons_issued.auto_update_time` beyond known ones | 🔴 High — needed for T3 write point completeness | Developer |
| **Q6** | Are `source_delta` tables CDC-synced or full-refresh? Why was time travel rejected in CAP-190758? | 🔴 Urgent — determines T4 viability | DataBricks/ETL team |
| **Q7** | Is `created_on` in `source_delta.luci__coupons_created` sync? | 🔴 Urgent — needed for T4 `el_uc` CTE | DataBricks/ETL team |
| **Q9** | Which orgs are upload-only (Phase 0 safe)? | 🟡 Medium — scopes T1-new rollout | Developer / data-eng |
| **Q10** | Frequency of `reactivateCoupon` calls for DB-stats-read orgs? | 🟡 Medium — informs T7 priority | Developer |
| Q8 | Dracarys `batchStartTime` source | ✅ CLOSED — `UploadCouponEntity.createdOn` confirmed |
| Q11 | `uploadedOn` = send time confirmed | ✅ CLOSED |
| Q12 | `UploadCouponEntity.id` as stable job ID | ✅ CLOSED |
| Q13 | Dracarys calls `addIssuedCount` directly | ✅ CLOSED |
| Q14 | Dracarys writes `coupons_created` directly | ✅ CLOSED |
| Q15 | `UploadCouponEntity.createdOn` retry-stable | ✅ CLOSED |

* * *

## Recommended Execution Order

### Immediate (this week)
1. Close **Q6** (DataBricks team — CDC vs full-refresh, time travel rejection reason)
2. Close **Q7** (ETL team — `created_on` in Delta sync)
3. Merge **Luci §N branch** to master — prerequisite for T5

### Short Term (next 2 weeks)
4. Create and assign all 7 tickets in Jira under CAP-195120
5. Close **Q1/Q5** (Developer — `redeemed_date` immutability)
6. Close **Q2** (Developer — full UPDATE grep on `coupons_issued`)
7. Start **T7** (5-line change, independent, low risk)
8. Start **T5** (Dracarys team, independent)

### Sprint Work (JAS26)
9. **T3** (Java `coupon_event_log`) — longest lead time, start first
10. **T2-new** (Automation SM_09–SM_12) — in parallel with T3
11. **T6** (`stats_series_summary` dedup) — in parallel with T3
12. **T4** (DataBricks event_log CTEs) — after T3 has ≥1 ETL cycle
13. **T1-new** (leftover clusters) — after T4 approach confirmed

### Unblocking CAP-184614 (Redis cleanup)
14. **CAP-188284** (Redis dual Redisson client) → resolve → unblocks → **CAP-184614** (stop Redis writes + clean keys)
