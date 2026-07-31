---
title: "Luci Coupon Series Stats — Immutable Event Log: Consolidated Design"
subtitle: "Problem, Solution, Reversal-Flow Coverage, and Status — for Developer Handover (CAP-195120)"
---

# Luci Coupon Series Stats — Immutable Event Log: Consolidated Design for Handover

**Epic:** CAP-195120 (JAS26 — Calculate Stats Release)
**Scope:** Refines the previously-ticketed T3 (Java `coupon_event_log`) + T4 (DataBricks event-log CTEs) design
**Status:** Design finalized through senior/arch/QA review; **arbiter verdict = REFINE (1 open gap)** — see "Outstanding Before Build" at the end. Not yet implementation-approved.
**Audience:** Handover document for a junior developer picking this up. All claims below are grounded against the real `Capillary/Luci` codebase (file:line cited) — not assumptions.

* * *

## 1. The Problem

### 1.1 Symptom
The public API `getCouponConfiguration` sometimes returns **permanently negative** values for `num_issued`, `num_redeemed`, or `num_uploaded_nonIssued`. This is silent — no alert, no error — and does not self-correct.

### 1.2 Root cause
Two systems compute the same counts using **two incompatible models**, and the API blends them:

```
num_issued = blueprint_IC (from DataBricks) + Σ(window summary IC) (from Java, stats_series_summary)
```

| Layer | Model | Behavior |
|---|---|---|
| **DataBricks** (`databricks/stats/luci_calculate_stats.py`) | **Current-state query** | Reads `active`/`is_valid` flags and mutable timestamps (`auto_update_time`, `last_updated_on`) AS THEY ARE when the ETL runs (~2 days after the snapshot date), filtered against a cutoff |
| **Java** (`stats_series_summary`) | **Event-log delta** | Writes `+1` at issue, `-1` at revoke, etc., dated at the moment the event happened |

**The failure mode:** any business operation that changes a mutable state field — `revokeCoupons`, `invalidateCoupons`, `reactivateCoupon`, or even an unrelated metadata update (mergeUser, bill correction) — *after* the snapshot date but *before* the DataBricks ETL run, causes DataBricks to see a state that doesn't match what was true on the snapshot date. Java's delta, however, is already correctly dated. Combining a wrong current-state blueprint with a correct delta produces a negative number.

**Concrete example (revoke):** A coupon is issued on day D. It's revoked on day D+1 (`active` flips to `false`). DataBricks runs on D+2 with `cutoff=D`. Because DataBricks reads `active=TRUE` as an additional filter (not just the timestamp), the now-revoked coupon is **excluded** from the D blueprint (`blueprint_IC = 0`), even though it genuinely was issued and active on day D. Meanwhile Java already wrote a `+1` on issue (day D, correctly) and a compensating `+ric=1` on revoke (day D+1, inside the "window" Java sums going forward). Result: `(IC_window=0 − RIC_window=1) + blueprint_IC(0) = −1`. **`num_issued` is negative for a coupon that was legitimately issued.**

### 1.3 The five reversal/mutation scenarios that trigger this

| # | Scenario | Trigger | Field mutated | Effect |
|---|---|---|---|---|
| **A** | Metadata update (mergeUser, bill correction, resend) | Touches `coupons_issued.auto_update_time` without any real business-state change | `auto_update_time` pushed past cutoff | `num_issued` UNDER by 1 (permanent), `num_uploaded_nonIssued` OVER by 1 |
| **B** | `revokeCoupons` after snapshot | Coupon revoked | `coupons_issued.active=FALSE` + `auto_update_time` refreshed | `num_issued` goes **negative** |
| **C** | `invalidateCoupons` on an already-issued coupon | Same effect as revoke | `coupons_issued.active=FALSE` | `num_issued` goes **negative** |
| **D** | `invalidateCoupons` on an unissued DISC_CODE_PIN coupon | Coupon never issued, invalidated after upload | `coupons_created.is_valid=FALSE` + `last_updated_on` refreshed | `num_uploaded_nonIssued` goes **negative** |
| **E** | `reactivateCoupon` after snapshot | A redemption is reversed | `coupon_redemptions.active=FALSE` + `auto_update_time` refreshed | `num_redeemed` goes **negative** |

None of these self-correct — once the DataBricks snapshot date advances past the event date, that event's row is permanently "in the blueprint zone," and the wrong blueprint becomes the only source of truth for that date.

### 1.4 What is explicitly OUT of scope (confirmed intentional, not bugs)
- **Timezone handling** (`luci_calculate_stats.py:39`, UTC conversion intentionally disabled) — this is a deliberate convention keeping both sides of every cutoff comparison in the same frame of reference (server-local time treated as UTC). Confirmed correct, documented by the ETL team, out of scope for this fix.
- **FR-014** (USER_ID uploads not counted in `num_uploaded_total`) — a related but *separate* bug (tracked elsewhere, CAP-191062) caused by USER_ID uploads bypassing `coupons_created` entirely. The design below (§4) *does* structurally fix this as a side effect once the Dracarys-side producer ships, but it is not the primary target and must not be conflated with it in status reporting.

* * *

## 2. Solution Direction: Immutable Event Log

### 2.1 The core idea
Stop asking DataBricks to infer "what was true on date D" by reading *current* mutable state. Instead, give it an **explicit, immutable log of business events**, each stamped with the exact time it happened. DataBricks then does pure event-time arithmetic — no more "guess the past from the present."

```
IC  (num_issued)            = COUNT(ISSUED events, event_time <= cutoff)  −  COUNT(REVOKED events, event_time <= cutoff)
RC  (num_redeemed)          = COUNT(REDEEMED events, event_time <= cutoff) − COUNT(REACTIVATED events, event_time <= cutoff)
UC  (num_uploaded_nonIssued)= COUNT(UPLOADED events, event_time <= cutoff) − COUNT(ISSUED events, ...) − COUNT(INVALIDATED_UNISSUED events, ...)
UTC (num_uploaded_total)    = COUNT(UPLOADED events, event_time <= cutoff) − COUNT(REVOKED, ...) − COUNT(INVALIDATED_UNISSUED, ...)
```

No `active` flag. No mutable timestamp. No current-state guessing. Each event is written once, forever, at the moment it happened — that is what "immutable" means here.

### 2.2 Why this solves every one of the 5 scenarios
| Scenario | How the event log fixes it |
|---|---|
| **A** (metadata update) | The event log never reads `auto_update_time` at all. A mergeUser/bill update touches no event type — it is invisible to the formula. **Eliminated entirely**, not just compensated for. |
| **B** (revoke) | `REVOKED` event is written at revoke time, keyed to the exact `coupons_issued.id` that was `ISSUED`. `IC = ISSUED − REVOKED` nets to exactly the right value regardless of when the ETL runs relative to the revoke. **Fixed.** |
| **C** (invalidate issued) | Same mechanism as B — an issued-coupon invalidation writes the same `REVOKED` event type (the IC formula treats them identically; see §5.1 for the naming rationale). **Fixed.** |
| **D** (invalidate unissued) | `INVALIDATED_UNISSUED` event written at invalidation time, keyed to `coupons_created.id`. `UC`/`UTC` formulas subtract it correctly regardless of ETL timing. **Fixed.** |
| **E** (reactivate) | `REACTIVATED` event written at reactivation time, keyed to `coupon_redemptions.id`. `RC = REDEEMED − REACTIVATED` nets correctly. **Fixed.** |

### 2.3 The "null bridge" — no backfill required
A coupon issued *before* this system existed has no event-log row at all. The design treats **"no row" as "definitely before any cutoff"** (`event_time IS NULL` → always included). This means:
- No backfill of historical data is needed.
- The DataBricks half (below, T4) can deploy on the **same day** as the Java half (T3) — the null-bridge covers all pre-existing history automatically.

* * *

## 3. Key Design Decision: Key on Stable Numeric IDs, Never on `coupon_code`

This is the most important refinement made during this design session, and it's worth your junior developer fully internalizing *why*, because it's not obvious from the original ticket.

### 3.1 The problem with keying on `coupon_code`
It's tempting to think "a coupon is identified by its code" — but that's **false** in this schema:
- `coupons_issued` and `coupons_created` have **no unique constraint on `coupon_code`** anywhere — not globally, not per series. Confirmed: both tables' primary key is `(id, org_id)`; `coupon_code` only has a non-unique secondary index.
- **A coupon code CAN be revoked from series A and legitimately re-uploaded/re-issued under a completely different series B.** When a coupon is revoked (`CouponDaoImpl.revokeCoupon`, `CouponDaoImpl.java:701-719`), the row is **never deleted** — it's soft-deleted (`active=false`, `coupon_uniqueness` negated). Every uniqueness check that guards against duplicate uploads/issuance (`getCouponsPresent`, `couponCodeExists`) filters on `active=true`/`is_valid=true` only — meaning a revoked code is **invisible** to those checks and is free to be reused. This produces a brand new row with a brand new auto-increment `id`.

**If the event log deduped or joined on `coupon_code`,** this revoke-then-reissue-under-a-different-series sequence would risk the two coupons' events being conflated — a REVOKED event meant for series A's coupon could be mistaken for series B's, corrupting both series' counts.

### 3.2 The fix: key on the stable numeric primary key of whichever table the event mutates
| Event type | Correct key (`coupon_ref_id`) | Why |
|---|---|---|
| `UPLOADED`, `INVALIDATED_UNISSUED` | `coupons_created.id` | Stable, unique, assigned at upload time, never reassigned |
| `ISSUED`, `REVOKED` | `coupons_issued.id` | Stable, unique, assigned at issue time. **Confirmed:** revoke never deletes the row or changes its `id` — only flips `active=false`. So `REVOKED` naturally keys to the exact same `id` that was `ISSUED`. |
| `REDEEMED`, `REACTIVATED` | `coupon_redemptions.id` | This isn't a novel idea — it **already exists** in this schema. `coupon_redemptions` has **no `coupon_code` column at all**; it has always linked to the issued coupon via a numeric foreign key `coupon_issued_id`. The event log simply follows a precedent the schema already established. |

`coupon_code` is still stored on each event-log row — purely for human-readable audit/debugging — but it is **never used in a join, a dedup key, or a filter.**

### 3.3 Confirmed: no un-reversal exists, so this key can never legitimately collide
A separate but important verification made this session: **every reversal in Luci's code is a strict, one-way, irreversible state transition.**
- `coupons_issued.active`: only ever flips `true → false` (`revokeCoupon`, `invalidateCoupons`). There is **no** `UPDATE ... SET active = true` anywhere in the codebase that reuses an existing row's id.
- `coupons_created.is_valid`: only ever flips `true → false` (`markAsInvalid`). No un-invalidate exists.
- `coupon_redemptions.active`: only ever flips `1 → 0` (`invalidateRedemption`, called by `reactivateCoupon`). There is no `0 → 1` path, and the validation gate (`getCouponByRedemptionId`, filters `active=1`) actively **rejects** a second `reactivateCoupon` call on an already-reversed id.
- "Resend"/reissue flows always create a **new** `coupons_issued` row; they never flip a revoked row back to active.

**Why this matters:** the event-log's dedup key is `UNIQUE (org_id, series_id, coupon_ref_id, event_type)`. Because no legitimate flow can ever produce a second `REVOKED` (or `INVALIDATED_UNISSUED`, or `REACTIVATED`) event for the same `coupon_ref_id`, this key can only ever be hit twice by an *accidental duplicate* (e.g. RMQ redelivering the same message) — never by two different real business events colliding. This makes the dedup mechanism (§4.3) unambiguous and safe.

### 3.4 The one residual limitation this cannot fix (and why)
There is **no foreign key anywhere** linking a `coupons_created.id` to the `coupons_issued.id` it becomes when that same coupon is issued. The *only* way to know "has this created-but-unissued coupon since been issued?" is to match `(org_id, series_id, coupon_code)` strings between the two tables — this is true in the *existing* production code today (`CouponsCreatedDaoImpl.getUnissuedCouponsCount`) and in the *existing* DataBricks `unissued` CTE (`luci_calculate_stats.py:114-128`). It is not something the event-log design broke — it's a pre-existing schema characteristic that no amount of event-log redesign can bypass without an actual schema migration (out of scope here).

**Practical consequence:** the `UC`/`unissued` calculation (only that one calculation) still has to do a `coupon_code` string match between `coupons_created` and `coupons_issued`. Every other calculation (`IC`, `RC`, `UTC`'s revoke/invalidate subtraction) is fully ID-keyed and immune to the revoke-then-reissue-elsewhere scenario. This is flagged explicitly in the design as an accepted, tested (not silently ignored) residual risk — see §6.3.

* * *

## 4. How the Event Log Gets Populated: RMQ, Not Inline Writes

### 4.1 Why not just write directly to the table inline (the originally-ticketed T3 approach)?
The original T3 ticket scoped this as **6 synchronous inline DAO writes**, each wrapped in try/catch, directly inside the issue/redeem/revoke/reactivate/invalidate call paths. This works, but review flagged a real risk: any new write added to those hot paths — even one wrapped in try/catch — adds latency and a new failure surface to business-critical flows that currently have zero coupling to this new table.

### 4.2 The refined approach: publish an event, let one dedicated consumer write it
Instead, each business flow **publishes an async, fire-and-forget RMQ message** describing the event; a single new consumer is the *sole* writer to `coupon_event_log`. This decouples the write from the hot path entirely.

**Where does the consumer live?** Two candidates were considered:
- **Dracarys** (the upload-flow service) — rejected. Verified via the service graph: Dracarys has **zero** confirmed dependency edge on Luci today, and its own RMQ infrastructure is unconfirmed (code suggests Camel-based queueing exists, but no concrete call sites could be traced). Putting the consumer there would mean either a risky cross-service direct DB write into Luci's schema, or a new synchronous callback into Luci — defeating the point of decoupling.
- **Luci** — chosen. Luci already owns the `coupon_event_log` table and already has working, production-proven `@RabbitListener`/Spring-AMQP infrastructure (`SpringAmqpConfig.java`, with two existing consumers: `CreateCouponJobListener`, `ScheduledJobListener`). The new consumer follows that exact same convention.

### 4.3 The mechanics
- **New exchange/queue** inside Luci: `coupon_event_exchange` / `coupon_event_queue`, added alongside the existing `SpringAmqpConfig` beans, reusing the existing retry container (2s→30s backoff, 10 attempts, `RepublishMessageRecoverer` — note: this republishes to the *same* queue with a retry-count header, it is **not** a true dead-letter queue; don't call it a DLQ in conversation with ops).
- **One new consumer**, `CouponEventLogListener`, the sole writer to the table. It inserts via `INSERT ... ON DUPLICATE KEY UPDATE id = id` (a no-op on the `uq_dedup` unique key) — this is the exact same idempotency pattern already used by `StatsSeriesSummaryDaoImpl` for the *separate*, already-scoped `stats_series_summary` dedup ticket (T6); we deliberately reuse it rather than invent a second mechanism.
- **Four new producers, all inside Luci** (issue, redeem, revoke, reactivate) — each an additive `rabbitTemplate.convertAndSend` call placed **after** the underlying DB write succeeds, never inside the transaction. A publish failure is caught, logged, and counted — **it must never fail or roll back the parent business call.**
- **Non-blocking guarantee (important, added after arch review):** the publish itself runs through a small bounded thread pool with a bounded work queue and an explicit rejection policy (`AbortPolicy`, never `CallerRunsPolicy`) — this guarantees that even if the RMQ broker is down or the pool is fully saturated, `publish()` returns to the caller in microseconds rather than blocking. This was a real gap the architecture reviewer caught in an earlier draft: a naive "async" design can still block if the executor's own queue fills up; this is now explicitly pinned.
- **A fifth producer, out-of-repo:** Dracarys' upload-commit flow needs its own new producer emitting `UPLOADED` events (keyed to `coupons_created.id`, which Dracarys already knows — it writes that row directly into Luci's DB today). This is **not designed in detail here** — Dracarys' own RMQ/Camel infrastructure needs its own investigation by that team — but the payload contract below is what it must emit.

### 4.4 The payload contract (shared by all 5 producers, in-repo and future Dracarys)
```
{
  orgId:       int,
  seriesId:    int,
  couponRefId: long,        // the numeric id per §3.2's table — NEVER a coupon_code
  refIdType:   enum,        // COUPONS_ISSUED_ID | COUPONS_CREATED_ID | COUPON_REDEMPTIONS_ID
  couponCode:  string,      // audit/display only — never joined/filtered on
  eventType:   enum,        // UPLOADED | ISSUED | REDEEMED | REVOKED | INVALIDATED_UNISSUED | REACTIVATED
  eventTime:   timestamp    // the BUSINESS event time, supplied by the producer — never wall-clock at consumer-processing-time
}
```

**Critical detail your junior developer must not get wrong:** `eventTime` must be the time the business event actually happened (e.g., the coupon's issue timestamp, the revoke commit timestamp), captured by the *producer* at the moment it publishes. If it were instead set by the *consumer* at whatever time the message happens to be processed, you'd reintroduce a smaller version of the exact clock-skew bug this whole design exists to eliminate.

### 4.5 Cross-service contract risk
There's no compile-time enforcement that Dracarys' future producer and Luci's consumer agree on field names / enum values, since they're different repos and this isn't a shared protobuf/schema-registry setup (that level of tooling was judged too heavy for this design's scope; the contract above should just be documented and versioned carefully). **The production feature flag must not be flipped to fully "on" until the Dracarys-side producer actually ships** — until then, `UPLOADED` events simply won't flow, and `UC`/`UTC` will be incomplete for uploads (though `IC`/`RC` are fully independent of this and can go live without waiting).

* * *

## 5. Schema and DataBricks Changes

### 5.1 New table: `coupon_event_log`
```sql
CREATE TABLE `luci`.`coupon_event_log` (
  `id`             bigint(20)   NOT NULL AUTO_INCREMENT,
  `org_id`         int(11)      NOT NULL,
  `series_id`      int(11)      NOT NULL,
  `coupon_ref_id`  bigint(20)   NOT NULL,      -- numeric PK of the mutated table, NEVER coupon_code
  `ref_id_type`    varchar(24)  NOT NULL,      -- COUPONS_ISSUED_ID | COUPONS_CREATED_ID | COUPON_REDEMPTIONS_ID
  `coupon_code`    varchar(20)  NOT NULL,      -- audit/display ONLY
  `event_type`     varchar(24)  NOT NULL,      -- UPLOADED|ISSUED|REDEEMED|REVOKED|INVALIDATED_UNISSUED|REACTIVATED
  `event_time`     datetime     NOT NULL,      -- immutable, producer-supplied business-event time
  `created_on`     timestamp    NOT NULL DEFAULT CURRENT_TIMESTAMP,  -- ingestion time, audit/lag-monitoring only
  PRIMARY KEY (`id`, `org_id`),
  UNIQUE KEY `uq_dedup` (`org_id`, `series_id`, `coupon_ref_id`, `event_type`),
  KEY `idx_org_series_time` (`org_id`, `series_id`, `event_time`)
);
```
A note on naming: `invalidateCoupons` on an already-*issued* coupon writes a `REVOKED` event (not a separate `INVALIDATED_ISSUED` type) — because the `IC` formula treats both identically (both simply subtract 1 from issued count). This was reviewed and accepted as correct for count purposes; if audit trails ever need to distinguish "explicitly revoked" from "invalidated-while-issued," the enum can be extended later without breaking the count formulas.

### 5.2 DataBricks CTE rewrite (`databricks/stats/luci_calculate_stats.py`)
The current `issued`/`redeemed`/`created`/`unissued` CTEs (today keyed on `active`/`is_valid`/mutable-timestamp filters) are replaced by `el_ic`/`el_rc`/`el_uc`/`el_utc`, each joining `coupon_event_log` **by the matching ref-id for that event type** (never by `coupon_code`), with the null-bridge (`event_time IS NULL` → treated as pre-cutoff → always included) preserved. The one exception, as noted in §3.4, is the `el_uc` CTE's created↔issued "is this still unissued" check, which necessarily remains a `coupon_code` string anti-join between `coupons_created` and `coupons_issued` directly — there is no ID-based alternative without a schema migration.

The change is gated behind a feature-flag widget (`event_log_enabled`, default `false`) — the legacy current-state CTEs remain intact as the `false` branch, so rollback is a one-line flag flip, no redeploy.

### 5.3 What does NOT change
- Java's `stats_series_summary` delta-write formula and its consumers (`StatsHistoryServiceImpl`, `MySQLCouponSeriesStatisticsReadService`) — confirmed to be pure count arithmetic with zero coupon-level identifiers. No changes needed here at all.
- The timezone handling convention (§1.4) — untouched, confirmed intentional.
- Retention: `ISSUED`/`REDEEMED`/`REVOKED`/`REACTIVATED` events can be cleaned up after 7-30 days (once they're safely inside older blueprints); `UPLOADED`/`INVALIDATED_UNISSUED` are retained until the series is deactivated (since an uploaded-but-unissued coupon can remain "pending" indefinitely).

* * *

## 6. Test Strategy Highlights (full detail in the design artifact — summarized here for handover context)

### 6.1 The single most important test
A DataBricks fixture test asserting that a `coupons_issued` row with `active=false` and **no corresponding `REVOKED` event** (i.e., a hypothetical pure metadata mutation with no real business event) is still counted correctly as issued by the event-log CTEs — this is the literal defining behavioral difference from the legacy model and the direct fix for Scenario A (§1.3).

### 6.2 Per-event-type coverage
Producer unit tests confirming, for each of the 4 in-repo event types, that: the correct `couponRefId`/`refIdType` is published; `eventTime` is the real business time (not publish-time wall-clock) — this was specifically checked across all 4 types, not just ISSUED; empty-batch inputs publish nothing without error; and a publish failure never propagates to fail the parent business call.

### 6.3 The residual limitation, tested not just documented
A specific fixture scenario: coupon code `X` issued under series A, revoked, then code `X` legitimately re-uploaded and re-issued under series B. Assert `IC`/`RC` are fully correct for both series (proving the ID-keying works), while separately characterizing (not "fixing") the `UC`/`unissued` anti-join's actual behavior in this same scenario, since that calculation still relies on the code-string match noted in §3.4.

### 6.4 Idempotency / redelivery
Same RMQ message delivered twice → exactly one row written (dedup key absorbs it, no duplicate, no exception). Two different event types for the same `coupon_ref_id` (e.g. `ISSUED` then later `REVOKED` on the same id) → two distinct rows, correctly not colliding (dedup key includes `event_type`).

### 6.5 Regression guard
With the feature flag off (`coupon.event.log.enabled=false`, `event_log_enabled=false`), zero new RMQ publishes occur and the DataBricks output is byte-for-byte identical to the pre-existing legacy CTEs — this is the rollback safety net.

* * *

## 7. Rollout Sequencing

1. **T3 (Java)** ships first: new table, DAO, consumer, 4 in-repo producers — all behind `coupon.event.log.enabled=false`. Inert until flipped.
2. **T4 (DataBricks)** can ship the **same day** as T3 (the null-bridge means no backfill wait is needed) — also behind `event_log_enabled=false`.
3. Flip flags **per-org** once confidence is established; `IC`/`RC` can go live independent of the Dracarys-side work, since they never depend on `UPLOADED` events.
4. `UC`/`UTC` remain incomplete for upload-heavy series until the **Dracarys producer** (separate, out-of-repo effort) ships — this should be communicated clearly so it isn't mistaken for a T3/T4 defect.
5. Shadow-run/compare DataBricks output against the legacy CTEs for a period before fully cutting over, given this affects a public API's returned numbers.

* * *

## 8. Outstanding Before Build (status honesty for the junior developer)

This design has been through senior-engineer, architecture, and QA review (35 concerns raised, 29 incorporated with substance, 4 soundly rejected with documented reasoning, 1 confirmed already-correct — nothing was silently dropped) and one arbiter pass.

**Current arbiter verdict: `REFINE` (confidence 72%), 2 gaps identified — 1 closed, 1 still open:**
1. ✅ **Closed:** the async RMQ-publish executor's saturation/rejection behavior was underspecified in an earlier draft (risk: it could silently block the hot path exactly like the thing it was meant to fix) — now explicitly pinned to a bounded queue + `AbortPolicy`, never `CallerRunsPolicy`.
2. ❌ **Still open:** the arbiter wants **runtime evidence** (an actual observed negative-count occurrence in production/logs/metrics, or confirmation that no alerting exists to explain why none is findable) confirming the root cause empirically — not just the strong static code-level argument in §1.2-1.3, which is already fully verified against source. Two attempts to get this from the incident-investigation agent timed out (infra issue, not resolved). **This must be closed — with either real evidence or an explicit, reasoned decision to proceed without it — before this design can pass the arbiter gate and move to implementation planning.**

**This document is a complete technical handover of everything designed and reviewed so far** — your junior developer can start familiarizing with the approach, the schema, the RMQ wiring conventions, and the DataBricks CTE shape immediately. It should not be treated as "implementation-approved" until gap 2 above closes and the arbiter re-runs.
