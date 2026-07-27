---
title: "Peer Architect Review: Luci Coupon Series Stats — Redis → MySQL + Databricks Migration"
subtitle: "Analysis Validation, Issue Decomposition & Solution Assessment | JAS26 / CAP-195120"
---


# Peer Architect Review: Luci Coupon Series Stats — Redis → MySQL + Databricks Migration

**Epic:** CAP-195120 (JAS26 — Calculate Stats Release)  
**Source document:** `2026-06-11-databricks-stats-cutoff.md` (v1, Saravanan Kesavan)  
**Reviewer:** Explore (Peer Architect)  
**Review date:** 2026-07-04  
**Status:** READY FOR AUTHOR REVIEW — awaiting sign-off before implementation

* * *

## 1. Executive Summary

The investigation document is **architecturally sound, well-evidenced, and largely correct**. The root cause identification — two incompatible counting paradigms (DataBricks current-state query model vs. Java event-log delta model) producing negative counts — is precise and supported by file:line evidence throughout. The three-option solution framework is well-structured.

**Key validation findings:**

| Area | Verdict | Notes |
|------|---------|-------|
| Root cause (mutable timestamp paradigm mismatch) | ✅ CONFIRMED CORRECT | §I architectural diagnosis is excellent |
| Scenario A–E miscompute walkthroughs | ✅ CONFIRMED CORRECT | Each scenario correctly traces the formula breakdown |
| Option 1 (Time Travel) | ⚠️ CONDITIONAL | Valid IF CDC-sync confirmed (Q6 open); rejected in CAP-190758 comments — needs clarification |
| Option 2 (SQL filter fix) | ✅ CORRECT | Safe fallback; mirrors Java diff-checker logic correctly |
| Option 3 (Immutable Event Log) | ✅ CORRECT AND PREFERRED | Eliminates paradigm mismatch; null-bridge is elegant |
| Phase 0 (created_on swap) | ⚠️ PARTIAL OVERLAP | CAP-195082 (CLOSED) already did `last_updated_on → created_on` for USCRM; need to confirm cluster coverage |
| §K Cache staleness (MaxRedemptionForSeriesProcessor) | ✅ NEW VALID FINDING | Not yet ticketed; low-severity but real |
| §M Late-notify double-count | ✅ NEW VALID FINDING | Not yet ticketed; cross-day double-count is real |
| §N batchStartTime implementation | ✅ WELL-DESIGNED | Luci side done; Dracarys side PENDING |
| Timezone (intentional design) | ✅ CONFIRMED OUT OF SCOPE | Correct call — don't disturb the frame-of-reference convention |

**5 new issues identified that need tickets under CAP-195120.** Details below.

* * *

## 2. Document Validation — Section by Section

### 2.1 Root Cause — VALIDATED ✅

The diagnosis in §I ("Two Incompatible Counting Paradigms") is the single most important insight in the document, and it is correct:

```
num_issued = blueprint_IC + Σ(window summary IC)
             ↑ current-state query    ↑ event-log delta
```

A revoked-after-snapshot coupon is **absent from blueprint** (`active=FALSE` at ETL run time) but **present as `+ric` in Java window** → `(0 − 1) + 0 = −1`. This is not fixable by column substitution within the current-state model — it requires either compensation logic (Options 1/2) or paradigm replacement (Option 3).

**One clarification needed on the root cause statement:**

The document says "DataBricks uses `auto_update_time` as the snapshot cutoff filter." More precisely:
- For IC/RC: `auto_update_time <= cutoff` is used for **both** the window filter AND the active-state proxy. The problem is two-fold: (a) `auto_update_time` is mutable, (b) `active` is a current-state flag, not a point-in-time flag.
- For UC/UTC: `last_updated_on <= cutoff` is the window filter, plus `is_valid=1` is a current-state flag. Same dual problem.

This distinction matters for Option 2: the compensation `OR auto_update_time > cutoff` fixes (a) but relies on `issued_date` / `redeemed_date` to fix (b). The document covers this correctly in the Option 2 SQL — just flagging it as something to emphasize in the tech-detail brief.

### 2.2 Evidence Trail — VALIDATED ✅

All file:line citations are internally consistent. Key ones I want to highlight for the implementer:

| Citation | Importance | Validation note |
|----------|-----------|-----------------|
| `CouponDaoImpl.java:1322–1328` — Java diff-checker compensation | **CRITICAL** | Option 2 SQL is literally a DataBricks port of this Java logic. This is the right template. |
| `CouponRedemptionDaoImpl.java:861–867` — same for redemptions | **CRITICAL** | Same port pattern for RC CTE |
| `CouponsCreatedDaoImpl.java:244` — explicit `LAST_UPDATED_ON = new Date()` on queue | **CRITICAL** | This is why Phase 0 / `created_on` swap matters — `last_updated_on` is poisoned by the queue operation itself |
| `CouponRuntimeServiceImpl.java:2819–2820` — `+ric` NOT `−ic` on revoke | **CRITICAL** | This is the exact reason the formula goes negative. Must NOT be changed — it's correct for the Redis path. DataBricks must compensate. |
| `StatsHistoryServiceImpl.java:80` — `startDate = snapshotDate + 1` | IMPORTANT | Defines the boundary: once snapshotDate advances past event date, the event is in blueprint zone permanently |

### 2.3 Scenario Walkthroughs — VALIDATED ✅

All 5 scenarios (A–E) are correct. Two observations:

**Scenario A (metadata update → IC under, UC over):**  
The UC overcount (coupon re-classified as unissued) is a subtle consequence that deserves more prominence. A coupon that is both issued AND showing as unissued violates the invariant `issuedCount + unissuedCount = uploadedTotal`. This invariant check (`num_uploaded_nonIssued + num_issued = num_uploaded_total`) should be in the post-deploy monitoring suite.

**Scenario C (revoke after snapshot):**  
The note that Java writes `+ric` NOT `−ic` on the MySQL path is crucial and **must be verified** by the implementer. If any code path writes `−ic` on revoke, the negative-count formula above would be `(−1 − 0) + 0 = −1` — same result but different fix needed. Reference: `CouponRuntimeServiceImpl.java:2819–2820`.

### 2.4 Extended Analysis §A–§I — VALIDATED ✅

**§A (Three-Layer Assembly Verified):** The verification against org 2000000, series 811192 is solid. Blueprint + window = API value confirmed for all 4 fields. Good baseline to replicate in post-deploy validation.

**§B (issued_date timezone convention):**  
Correct — `issued_date` is a DATETIME column storing org-local wall-clock time, **not** a TIMESTAMP. This is exactly why §C correctly concludes `issued_date` cannot replace `auto_update_time` as the cutoff column. This is a non-obvious but critical constraint that will confuse implementers — it should be prominently called out in the tech-detail brief.

**§C (issued_date cannot replace auto_update_time as cutoff):**  
**VALIDATED and CRITICAL.** The document's conclusion is correct: `issued_date` (DATETIME, org-local) vs `auto_update_time` (TIMESTAMP, server-local) are in different frames of reference. Using `issued_date <= cutoff` would produce cross-org timezone drift bugs. The document's Option 2 correctly retains `auto_update_time` for the window boundary and uses `issued_date` only for the compensation condition `OR auto_update_time > cutoff`. This is the right approach.

**§D (summary_date uses server JVM date):**  
Noted correctly as a lower-severity attribution issue, not a count correctness issue. Agreed it is out of scope for this fix.

**§E (Midnight Race — transient 24h undercount, self-correcting):**  
Valid finding. The ~24h self-correcting undercount from DB clock / JVM clock skew is real. Agreed it does not need a fix — self-corrects with next blueprint. Document correctly distinguishes this from the permanent undercounts in Scenarios A–E.

**§F (FR-014 — USER_ID upload UTC gap):**  
**Confirmed valid.** The 705-code gap for series 811192 is correctly diagnosed: USER_ID uploads bypass `coupons_created`, so DataBricks `created` CTE can never count them. CAP-191062 is IN DEV for the Java counter fix. However, CAP-191062 only fixes the Java `uploadedTotalCount` field — the **DataBricks blueprint UTC gap** for USER_ID series still exists and is NOT covered by CAP-191062 (which is a Java-side fix). Option 3's event log is the correct DataBricks fix. This needs to be explicitly documented in CAP-191062 or a linked ticket.

**§G (Org timezone — not a concern for count correctness):**  
Agreed. Internal consistency of the three-layer pipeline is the only invariant that matters for correct cumulative counts.

**§H (Midnight batch split-attribution overcount — transient 24h, self-correcting):**  
Valid finding. The direction (OVERCOUNT, not undercount) is important and correctly stated. The self-correcting property is confirmed. Release 3 (Dracarys batchEventTimeMillis) is the right fix. No DataBricks compensation needed — agreed.

**§I (Fundamental Design Problem — Two Incompatible Paradigms):**  
**The strongest section of the document.** The three-dimensional breakdown of the problem is precisely correct:
1. Mutability → Options 1/2 fix
2. State-vs-event → Options 1/2 compensate; Option 3 eliminates
3. Upload-dimension (FR-014) → Only Option 3 fixes

This is the right framing and should be the opening of any engineering design doc or RFC for this work.

### 2.5 Option Analysis — VALIDATED WITH NOTES

#### Option 1 (Delta Lake Time Travel)

**⚠️ NEEDS CLARIFICATION — conflict with CAP-190758 comment.**

CAP-190758 has a comment from saravanan.kesavan (2026-07-03): *"Assessed Delta Lake time travel using TIMESTAMP AS OF — we can't rely on Databricks time travel."*

The document (dated 2026-06-11) was written before this assessment. The document correctly identifies the CDC prerequisite (Q6) but the in-ticket rejection suggests CDC sync mode may have been confirmed as full-refresh overwrite.

**Questions for the author before proceeding:**
1. Was Q6 confirmed? Are `source_delta` tables CDC-synced or full-refresh?
2. If full-refresh, the "TIMESTAMP AS OF" is a no-op — Option 1 is off the table.
3. The document should be updated to reflect this finding from CAP-190758.

**If CDC is confirmed:** Option 1 remains the easiest interim fix (4 line changes). The comment in CAP-190758 is ambiguous — it says "can't rely on" which could mean CDC was not confirmed OR that even with CDC it had other issues (Delta file compaction, vacuum settings, etc.).

#### Option 2 (SQL Filter Fix)

**✅ VALIDATED — always safe, always available.**

The SQL logic correctly mirrors `CouponDaoImpl.java:1322-1328`. The `OR auto_update_time > cutoff` compensation is the right approach. One note:

**Edge case to validate for Scenario C (revoke):** The document notes a risk: "a coupon revoked before the snapshot but with `auto_update_time` accidentally refreshed after the cutoff by a background process would be incorrectly included." This risk is real but the doc correctly assesses it as low. The pre-deploy validation query is good.

**Q7 dependency (created_on in Delta sync):** The document correctly notes this as a blocker for Bug 2 fix and provides an interim partial fix. This should be raised with the ETL team immediately.

#### Option 3 (Immutable Event Log)

**✅ RECOMMENDED — architecturally correct.**

The null=pre-cutoff bridge is elegant and solves the backfill problem cleanly. The pure event-arithmetic formulas are correct:

```
IC  = ISSUED_events − REVOKED_events           (event_time ≤ cutoff)
RC  = REDEEMED_events − REACTIVATED_events
UTC = UPLOADED_events − REVOKED − INVALIDATED_UNISSUED
UC  = UPLOADED_events − ISSUED − INVALIDATED_UNISSUED
```

**One gap in the Option 3 design that needs addressing:**

The document lists 6 Java write points for `coupon_event_log`. However, the `invalidateCoupons` on an **issued** coupon (which sets `active=FALSE` on `coupons_issued`) is described as writing a `REVOKED` event. This could create semantic confusion with the actual `revokeCoupons` flow (which also writes `REVOKED`). Two questions:

1. Should `invalidateCoupons` on issued coupons write `INVALIDATED_ISSUED` instead of `REVOKED`? Or is the count formula identical for both? (For IC: both subtract 1 — so formula is identical. For UTC: revoke subtracts from UTC but invalidate of an issued coupon does not subtract from UTC because the code was already issued, so UTC was already decremented at issuance. This needs to be verified.)

2. Is there a `bulkRevoke` or batch invalidation path that bypasses the 6 write points listed? A grep for all UPDATE statements on `coupons_issued.active = FALSE` would confirm.

**Retention policy gap:**  
The document says ISSUED/REDEEMED/REVOKED/REACTIVATED events can be cleaned after 7–30 days. However, if DataBricks runs with `run_days_offset=2`, the event log must retain at least `2 + buffer` days for all ETL runs to see events. The 7-day floor is fine for a 2-day offset, but if `run_days_offset` ever increases, this must be revisited.

### 2.6 §K Cache Staleness (MaxRedemptionForSeriesProcessor) — VALIDATED ✅

The analysis is correct. The staleness window is real:
- `getHistoricalValue()` is cached with a date-prefix key → snapshot can advance within the same calendar day when `statsHistory` job runs
- `getMysqlSummaryValue()` uses a fresh registry lookup → gets new snapshot
- The gap between the two causes RRC events to be missed in the enforcement formula

The severity assessment (LOW in practice) is correct — RRC events are rare. Option B (fetch registry once per request, pass snapshot consistently) is the right fix and is a 5-line change. The recommendation to ship this with Option 3 work is sound.

### 2.7 §M Late-Notify Double-Count (stats_series_summary Idempotency) — VALIDATED ✅

The analysis is correct. The core issue:
- `stats_series_summary` writes are `value = value + delta` (not SET)
- `notifyCouponsUploadRequest` has no idempotency guard
- A Dracarys retry on day D+2 after a D-day failure double-counts UC/UTC for ~24h

The duration analysis is correct — self-corrects when next ETL absorbs the D+2 summary row into the blueprint. But the ~24h window where display shows 2× is a real customer-impacting bug for high-volume orgs.

**One gap:** The document recommends shipping Option 2 (uploadedOn as summaryDate) immediately, but notes this requires the Dracarys fix first (Q11 CLOSED — uploadedOn is currently notification send time, not batch start time). The dependency chain needs to be explicit in tickets:

```
Dracarys: fix uploadedOn = batchStartTime (PostUploadConvertorImpl:34)
    ↓ prerequisite for ↓
Luci: use uploadedOn as summaryDate (already implemented per §N)
    ↓ together eliminate ↓
Cross-day late-notify double-count
```

The concurrent same-day retry double-count still requires the dedup table (Option 1 of §M), which is a separate ticket.

### 2.8 §N batchStartTime Implementation — VALIDATED ✅

The Luci-side implementation is well-designed. The additive overload pattern (backward-compatible, null fallback) is correct. The deploy order (Luci first, then Dracarys) is correct.

The Dracarys change diffs are precise and complete. No issues with the implementation design.

* * *

## 3. Independent Issues — Full Decomposition

Based on the document and cross-checking against existing Jira tickets, here are **all independent issues** with their current ticket status:

### Issue 1 — DataBricks Mutable Timestamp Miscompute (IC/RC/UC/UTC)
**Existing ticket:** CAP-190758 (Open, P1)  
**Status:** Solutioning in progress. Delta Lake time travel rejected (per 2026-07-03 comment). SQL filter fix (Option 2) and Immutable Event Log (Option 3) remain active paths.  
**Action:** Enrich CAP-190758 with the three-option framework from the document. Confirm Q6 (CDC vs full-refresh) to definitively close or reopen Option 1.

### Issue 2 — Phase 0: created_on swap in DataBricks (upload-only orgs / Comcast)
**Existing ticket:** CAP-195082 (CLOSED in AMJ26 for USCRM — but §L shows incrm applied and uscrm PENDING)  
**⚠️ Conflict:** §L says uscrm Phase 0 is "Pending" as of 2026-06-18. But CAP-195082 is closed. Either:
- CAP-195082 was closed prematurely before all clusters were covered, OR
- A new ticket is needed for the remaining cluster deployments  
**Action:** Verify which clusters have the Phase 0 fix applied. Create a new ticket under CAP-195120 for any remaining clusters (track separately from the broader Option 2/3 work since it's a 2-line change).

### Issue 3 — FR-014: USER_ID Uploads Not Counted in UTC (DataBricks blueprint gap)
**Existing ticket:** CAP-191062 (In Dev) — covers the Java `uploadedTotalCount` fix only  
**Gap:** CAP-191062 does NOT fix the DataBricks blueprint UTC gap for USER_ID series. Blueprint UTC will still be wrong for USER_ID-upload series until Option 3 ships.  
**Action:** Link CAP-191062 to Option 3 work. Add explicit scope note: "Java fix covers incremental window; blueprint fix requires Option 3 (Immutable Event Log)."

### Issue 4 — Immutable Event Log (Option 3): coupon_event_log Table + Java Write Points
**Existing ticket:** NONE  
**Scope:** New `coupon_event_log` table, DAO, 6 Java write points in LuciThriftServiceImpl + processor chain, feature flag per org.  
**New ticket needed:** YES — under CAP-195120

### Issue 5 — DataBricks: Immutable Event Log CTEs + Null Bridge (Option 3 DataBricks half)
**Existing ticket:** NONE  
**Scope:** Replace `coupons_issued`/`coupons_created` CTEs in all cluster DataBricks notebooks with `event_log` aggregations. Deploy same day as Issue 4. Requires `coupon_event_log` in Delta sync.  
**New ticket needed:** YES — under CAP-195120 (can be child of Issue 4 or separate)

### Issue 6 — DataBricks: SQL Filter Fix (Option 2 — interim bridge while Option 3 ships)
**Existing ticket:** NONE (CAP-190758 exists for the problem; no ticket for the SQL fix itself)  
**Scope:** 6 SQL filter changes in all cluster notebooks. Requires ETL team to add `created_on` to `source_delta.luci__coupons_created` (Q7). Backfill run needed post-deploy.  
**New ticket needed:** YES — under CAP-195120 (or resolve as sub-task of CAP-190758)

### Issue 7 — Dracarys: batchStartTime Fix (PostUploadConvertorImpl + BaseCouponUploaderImpl)
**Existing ticket:** NONE  
**Scope:** `PostUploadConvertorImpl.java:34` — change `uploadedOn = batchStartTime`; `BaseCouponUploaderImpl.constructCouponCreatedEntities()` — set `createdOn = batchStartTime`. Exact diffs in §N.  
**New ticket needed:** YES — under CAP-195120 (Dracarys team, depends on Q15 CLOSED ✅)

### Issue 8 — Late-Notify Double-Count: stats_series_summary Idempotency (upload dedup table)
**Existing ticket:** NONE  
**Scope:** New `coupon_upload_notify_log` table + dedup guard in `notifyCouponsUploadRequest` + `addIssuedCount` (Dracarys calls this directly — Q13 CLOSED). Covers concurrent same-day retry double-count that Release 3 does NOT fix.  
**New ticket needed:** YES — under CAP-195120

### Issue 9 — MaxRedemptionForSeriesProcessor: Cache Staleness on Registry Advance (§K)
**Existing ticket:** NONE  
**Scope:** Fetch `findLastActiveByOrgId()` once at top of `getRedeemedCouponCount()`, pass snapshotDate consistently to both `getHistoricalValue()` and `getMysqlSummaryValue()`. 5-line change (Option B from §K).  
**New ticket needed:** YES — under CAP-195120

### Issue 10 — Redis Spike: Dual Redisson Client Cleanup (CAP-188284)
**Existing ticket:** CAP-188284 (Open, P2)  
**Status:** Root cause identified (extra Redisson client for midnight cache manager). Fix: remove additional Redisson client.  
**Action:** Already tracked. Unblocks CAP-184614 (Stop Redis write + clean keys).

### Issue 11 — Stop Redis Writes + Clean Redis Keys (CAP-184614)
**Existing ticket:** CAP-184614 (Open, P2, blocked by CAP-188284)  
**Status:** Blocked. Must follow CAP-188284 resolution.

### Issue 12 — Response Spike Assessment Post Stats Release (CAP-189119)
**Existing ticket:** CAP-189119 (Open, P2)  
**Status:** ~20ms latency spike investigation ongoing.

* * *

## 4. Tickets to Create Under CAP-195120

### Tickets Already Existing (no new ticket needed)

| Issue | Existing Ticket | Status |
|-------|----------------|--------|
| DataBricks mutable timestamp root cause | CAP-190758 | Open P1 — enrich with solution options |
| USER_ID upload Java counter fix | CAP-191062 | In Dev — add scope clarification |
| Redis dual Redisson client | CAP-188284 | Open P2 |
| Stop Redis writes + clean keys | CAP-184614 | Open P2, blocked |
| Latency spike investigation | CAP-189119 | Open P2 |

### New Tickets to Create

| # | Title | Priority | Depends On | Owner |
|---|-------|----------|-----------|-------|
| T1 | Phase 0 cluster coverage gap: apply created_on swap to remaining clusters | P2 | Q7 (ETL confirm created_on in Delta) | DataBricks/ETL team |
| T2 | Option 2 (DataBricks SQL filter fix — interim bridge): immutable column cutoff compensation across all clusters | P1 | Q6 (CDC confirm), Q7 (created_on in Delta), backfill plan | DataBricks/ETL team |
| T3 | Option 3 Part A — Java: coupon_event_log table + DAO + 6 write points | P1 | None | Luci Java team |
| T4 | Option 3 Part B — DataBricks: event_log CTEs with null-bridge + Delta sync onboarding | P1 | T3 deployed (at least 1 run) | DataBricks team + ETL |
| T5 | Option 3 Part C — Dracarys: batchStartTime fix (uploadedOn + createdOn) | P2 | None (independent) | Dracarys team |
| T6 | stats_series_summary idempotency: coupon_upload_notify_log dedup table | P2 | T5 (Dracarys uploadedOn fix) | Luci Java team + Dracarys |
| T7 | MaxRedemptionForSeriesProcessor: consistent snapshotDate per request (§K cache staleness) | P3 | None | Luci Java team |

* * *

## 5. Dependency Graph

```
CAP-190758 (root cause — P1)
    ├── T1  Phase 0 cluster coverage         [XS, independent, DataBricks]
    ├── T2  Option 2 SQL filter fix           [S, interim bridge, DataBricks]
    │        └── blocks → backfill run
    ├── T3  Option 3A Java event_log          [L, Luci]
    │        └── enables → T4
    └── T4  Option 3B DataBricks event_log   [M, DataBricks]
             └── T3 prerequisite (null bridge handles history)

CAP-191062 (USER_ID Java fix — In Dev)
    └── NOTE: DataBricks blueprint UTC gap still open until T4 ships

T5  Dracarys batchStartTime fix              [S, Dracarys — independent]
    └── enables → T6 (cross-day dedup)

T6  stats_series_summary dedup table         [M, Luci + Dracarys]
    └── prerequisite: T5 (for cross-day case)
    └── Note: concurrent same-day dedup is independent of T5

T7  MaxRedemptionForSeriesProcessor cache    [XS, Luci — independent]

CAP-188284 Redis dual Redisson client
    └── unblocks → CAP-184614 Redis cleanup
```

* * *

## 6. Open Questions — Current Status & Owners

| # | Question | Status | Owner | Urgency |
|---|----------|--------|-------|---------|
| Q1 | Is `redeemed_date` on `coupon_redemptions` always populated? Immutable on reversal? | OPEN | Developer (verify schema + reactivate flow) | High — blocks Option 2 RC fix |
| Q2 | Full list of operations updating `coupons_issued.auto_update_time` beyond confirmed ones | OPEN | Developer (grep all UPDATEs on table) | High — needed for backfill scope |
| Q5 | Confirm `redeemed_date` immutability on reversal/reactivation | OPEN | Developer | High — same as Q1 |
| Q6 | Are `source_delta` tables CDC-synced or full-refresh overwrite? Delta logRetentionDuration? | **CRITICAL BLOCKER for Option 1** | DataBricks/ETL team | URGENT — determines Option 1 viability. Note: CAP-190758 comment suggests "can't rely on" time travel — confirm if this means CDC not available |
| Q7 | Is `created_on` in `source_delta.luci__coupons_created` Delta sync? | **CRITICAL BLOCKER for Phase 0 + Option 2 Bug 2** | DataBricks/ETL team | URGENT |
| Q9 | Which orgs are upload-only (Phase 0 safe)? | OPEN | Developer / data-eng | Medium — scopes Phase 0 rollout |
| Q10 | Frequency of `reactivateCoupon` calls for DB-stats-read orgs? | OPEN | Developer (query stats_series_summary for rrc > 0 rows) | Low — determines §K urgency |
| Q11 | uploadedOn = send time confirmed | ✅ CLOSED | — | — |
| Q12 | UploadCouponEntity.id as stable job ID | ✅ CLOSED | — | — |
| Q13 | Dracarys calls addIssuedCount directly | ✅ CLOSED | — | — |
| Q14 | Dracarys writes to coupons_created directly | ✅ CLOSED | — | — |
| Q15 | UploadCouponEntity.createdOn is retry-stable | ✅ CLOSED | — | — |

* * *

## 7. Risks & Gaps — Architect Concerns

### Risk 1 — Option 1 / Time Travel Conflict ⚠️ HIGH
The document proposes Option 1 as an interim fix. CAP-190758 (2026-07-03 comment) says time travel "can't be relied on." This is a direct contradiction. **Must be resolved before T2 (Option 2) is scoped** — if the rejection is because `source_delta` is full-refresh, Option 2 also needs a clarification on whether the SQL filter fix alone (without time travel) is sufficient.

**Recommended action:** Saravanan to add a comment to CAP-190758 clarifying the exact reason for time travel rejection (CDC not confirmed? Vacuum settings? GDPR delete complications?). Then close Q6.

### Risk 2 — invalidateCoupons Semantic Ambiguity in Option 3 ⚠️ MEDIUM
The document maps `invalidateCoupons` on an issued coupon to a `REVOKED` event in `coupon_event_log`. The IC formula `ISSUED − REVOKED` would then subtract both revoke AND invalidate operations from IC. Verify:
1. Is the business intent that an invalidated-issued coupon should decrement IC? (Currently it does: `active=FALSE` removes it from issued CTE.)
2. Should there be a separate `INVALIDATED_ISSUED` event type for clearer semantics, even if the formula treatment is identical?

This is low-risk for count correctness but important for auditability of the event log.

### Risk 3 — Backfill Coordination ⚠️ HIGH
Options 1 and 2 require a backfill run for affected date ranges. The backfill:
- Must cover `run_days_offset * 2` days minimum (2-day window × 2 = 4 days)
- Must be coordinated across all clusters simultaneously (or staggered with monitoring)
- Must NOT run concurrently with a regular ETL run

The document provides the right pre-backfill validation query. A formal backfill runbook should be created as part of T2 ticket scope.

### Risk 4 — Option 3 Null Bridge Window Narrowing ⚠️ MEDIUM
The null bridge (`el.event_time IS NULL` → treat as pre-cutoff) is elegant but has a subtle risk: if `coupon_event_log` is loaded into Delta Lake with a delay (ETL sync lag), a recently-written `ISSUED` event might still appear as NULL in Delta Lake even though it was written after Release 1 shipped. This would cause a newly-issued coupon to be counted as if it were a pre-Release-1 coupon — which is actually correct (it IS before any cutoff) — but only if the ETL sync lag is less than the `run_days_offset` (2 days). With a 2-day offset and a typical ETL sync lag of hours, this is safe. Confirm with ETL team.

### Risk 5 — `coupon_event_log` at Scale ⚠️ MEDIUM
For large orgs with millions of coupons, the `coupon_event_log` table will receive high-velocity writes. The proposed partition scheme (by `event_time`) is correct. Additional considerations:
- Write path: all 6 event write points are synchronous. If the INSERT fails, the main operation (issue/revoke/etc.) must not be blocked. **Recommend: async write or accept-and-log-on-failure semantics. Event log is auxiliary to the main transaction.**
- Retention cleanup job: must be a scheduled job, not a trigger. The document mentions 7–30 days for ISSUED/REDEEMED events — confirm with the DataBricks team that `run_days_offset` will never exceed retention minus buffer.

### Risk 6 — Phase 0 on USCRM still Pending ⚠️ LOW
§L confirms uscrm Phase 0 was pending as of 2026-06-18. This is the cluster with the Comcast org (2000101). If this is still not applied, Comcast sees wrong UC/UTC counts today. This is the quickest win and should be the first action item.

* * *

## 8. Recommended Action Plan

### Immediate (this week)
1. **Resolve Q6** — DataBricks team confirms CDC vs full-refresh for `source_delta` tables. Clarify the CAP-190758 time travel rejection reason.
2. **Resolve Q7** — DataBricks/ETL team confirms `created_on` in Delta sync for `coupons_created`.
3. **Apply Phase 0 to uscrm** — 2-line change, unblocks Comcast. Gate on Q7.
4. **Create T1 ticket** for tracking remaining Phase 0 cluster coverage.

### Short Term (next 2 weeks)
5. **Resolve Q1/Q5** — Developer verifies `redeemed_date` immutability.
6. **Resolve Q2** — Developer greps all UPDATE paths on `coupons_issued.auto_update_time`.
7. **Create T2 ticket** (Option 2 SQL filter fix) — interim DataBricks bridge. Scope backfill plan.
8. **Create T5 ticket** (Dracarys batchStartTime) — coordinate with Dracarys team.
9. **Create T7 ticket** (MaxRedemptionForSeriesProcessor cache staleness — §K) — 5-line fix, low risk.

### Medium Term (JAS26 sprint)
10. **Create T3 ticket** (Option 3A Java `coupon_event_log`).
11. **Create T4 ticket** (Option 3B DataBricks event_log CTEs).
12. **Create T6 ticket** (stats_series_summary dedup table — §M Option 1).
13. **Resolve CAP-188284** (Redis dual Redisson client) → unblocks CAP-184614 (Redis cleanup).

### Decision Gates Before Each Phase
- **T2 (Option 2 — DataBricks interim fix):** Q6 closed, Q7 confirmed, backfill plan ready.
- **T3/T4 (Option 3 — Immutable Event Log):** Q1/Q5 closed, `redeemed_date` confirmed immutable, `coupon_event_log` schema reviewed by DBA, ETL sync plan confirmed.
- **T4 deploy:** T3 must have been live for at least 1 full ETL cycle before T4 deploys (so null bridge covers all history).
- **CAP-184614 (Redis cleanup):** CAP-188284 must be resolved first.

* * *

## 9. Summary Verdict

| Dimension | Assessment |
|-----------|-----------|
| **Root cause diagnosis** | ✅ Correct and thorough |
| **Solution options** | ✅ Well-structured; Option 3 is the right long-term choice |
| **New issues identified (§K, §M, §N)** | ✅ Valid, actionable, well-evidenced |
| **Implementation readiness (Luci side)** | ✅ §N implementation is complete and compile-clean |
| **Implementation readiness (Dracarys side)** | ⚠️ Pending — exact diffs provided in §N, needs Dracarys team pickup |
| **Ticket coverage** | ⚠️ 5 net-new issues need tickets; 2 existing tickets need enrichment |
| **Open questions blocking build** | ⚠️ Q1/Q2/Q5 (redeemed_date), Q6 (CDC), Q7 (created_on Delta) must close before build starts |
| **Overall confidence** | HIGH — ready to proceed to ticketing and tech detail once open questions close |

**Bottom line:** The document is a high-quality investigation. The analysis is correct. The Option 3 architecture is the right long-term answer. The immediate blocker is closing Q6 (CDC vs full-refresh) and Q7 (created_on in Delta) with the DataBricks/ETL team — these two answers determine whether Phase 0 + Option 2 can ship as interim fixes while Option 3 is built.

* * *

*Review by Explore (Peer Architect) | 2026-07-04 | Awaiting author review before proceeding to implementation planning*
