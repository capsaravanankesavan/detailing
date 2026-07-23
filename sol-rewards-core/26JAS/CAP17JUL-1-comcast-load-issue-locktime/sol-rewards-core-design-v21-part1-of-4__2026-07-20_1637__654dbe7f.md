---
title: "issueReward Lock Removal — Complete Design v21 (Part 1 of 4)"
subtitle: "Problem, Solution Narrative A-B.5 — see Parts 2-4"
---

* * *
title: "issueReward Lock Removal — Complete Design v21 (Part 1 of 4)"
subtitle: "Problem, Solution Narrative A-B.5 — see Parts 2-4 for the rest"
---

# sol-rewards-core: Removing the issueReward Redis Lock — Complete Design Reference (v21, self-contained) — PART 1 of 4

**Status:** v21 fixes 2 real gaps QA found in v20's fixes to v19's ETERNAL-sync mechanism (v19 replaced a fragile "sentinel row" approach — which needed 3 arbiter REFINE cycles across v16-v18 — with a simpler approach that mirrors legacy's own daily-bucketed table shape for TBL_REWARD_ISSUE_SUMMARY, confirmed via tracing all callers of Utils.getDateWithoutTimestampInSpecifiedZone). v20 fixed 5 real bugs in v19's pseudocode (wrong bulkAtomicIncrement signature assumption, missing org-timezone resolution in the cron's background context, unsafe two-store write ordering, a first-tick double-count bug, unspecified negative-delta handling) plus a corrupted job body. v21 fixes: (1) the find-or-create sequence itself had an unaddressed concurrent-writer race — fixed with a duplicate-key-catch-and-reconcile pattern mirroring writeNonOrgSummariesAtomically's own bulkSaveOrUpdate; (2) a migration-seed doc inconsistency (§B.12 still said lastSyncedCumulativeTotal=null contradicting the getOrCreate fix). 118 tests total. **Last arbiter PASS was v14.2 — v15 through v21 are real, unarbitrated changes; this design still needs a fresh senior+QA review + mandatory arbiter verdict before Phase B (implementation) can start.**

**⚠️ NOTE ON THIS SPLIT:** This design is split across 4 parts due to its size (~150KB total, built up over 21 revisions). This is Part 1 (Problem statement + solution narrative through §B.5). Parts 2-4 continue the solution (§B.6 onward), scaling, risks, code-paths/criticality tables, the full 118-test list, and the complete version-by-version changelog. All 4 parts together are the single, complete, self-contained design — read them in order.

* * *

## PART A — THE PROBLEM (why we're doing this at all)

### A.1 Symptom
`POST /v1/user/rewards/issue` and `POST /v1/user/reward/{rewardId}/issue` suffer severe latency under concurrent load against a single popular reward.

### A.2 Root cause (confirmed via NewRelic, org 2000102, 2026-07-14 load test)
PR #1277 added `RedisLockService.acquireRewardLock` to `NonOrgSummaryWriteProcessor`, guarding **reward-level** ("NonOrg" — a limit that applies across ALL customers issuing a given reward, as opposed to a per-customer limit) constraint enforcement. This lock is the bottleneck:

- **Coarse key**: `reward_constraint:{orgId}:{rewardId}` — serializes every customer issuing the same reward, not just concurrent writers to the same summary row (`NonOrgSummaryWriteProcessor.java:53,102`).
- **5000ms blocking wait, no backoff** (`RedisLockService.java:76-102`).
- **Wide critical section**: a re-evaluation read + N-row atomic write all happen INSIDE the lock (`NonOrgSummaryWriteProcessor.java:166`).
- **Nested inside** the pre-existing, unrelated per-customer `@CustomerLockable` lock (`UserRewardUtils.java:56,61`) — compounding the concurrency loss.

NewRelic evidence: `acquireRewardLock` = 60.45% of total transaction time; lock-acquire p95 hit **91–101 seconds** under load, with the issue-POST endpoints never recovering while GET endpoints recovered normally — ruling out a generic shared-resource bottleneck and confirming this specific lock as the cause.

### A.3 Scope of the fix
**ONLY `Level.REWARD` constraints.** Customer-level constraints (`Level.CUSTOMER`, `Level.TRANSACTION`, and all `ORG_CONSTRAINT_ALLOWED_LEVELS`) are completely out of scope and untouched — already protected by the separate `@CustomerLockable` lock, which has no contention problem.

### A.4 Non-functional target
Sustain **5,000 requests/minute (≈83.3 req/sec) against a single reward**, with **strict** limit-breach prevention (no tolerance for over-issuance) and no application-level lock.

### A.5 Design evolution — why there were so many versions
- **v1–v3** (superseded): enforcement read a periodically cron-synced MySQL value, tolerating a bounded "overshoot" window. **User explicitly rejected this** — Mongo must actively *prevent* a breach on the write path, not just track for later reconciliation.
- **v4–v6**: rebuilt around strict, atomic, budget-partitioned Mongo shards ("slots" from v13 onward). v6 passed arbiter — but its counter *key* was wrong.
- **v7**: fixed the wrong key; added full per-KPI and window/frequency coverage, and the 5,000 rpm scaling math.
- **v8**: fixed a critical multi-counter atomicity gap; made the cron aware Mongo access is org-sharded. **Arbiter PASS.**
- **v9**: user's own design review surfaced real gaps (LIMIT_VALUE changes weren't handled, cron design underspecified, missing doc-lifecycle fields) — folded in as a mechanism change.
- **v10**: reviewer fan-out on v9 found **5 genuine implementation-blocking bugs** in the new cron/locking/metrics plumbing (things that would not have compiled, or would have silently done nothing). All fixed here. **Arbiter PASS.**
- **v11-v12**: user Q&A + review found real gaps in POINTS-KPI reachability, TTL index provisioning, and a fixed shard-count-16 sizing issue for low-limit rewards — fixed via an adaptive slot-count formula, itself needing a fix for ETERNAL counters (which have no rollover boundary to self-heal a bad sizing choice).
- **v13**: a 13-question user Q&A round — renamed "shard"→"slot" (avoiding confusion with MongoDB's own real sharding concept), fixed a real existing-reward-seeding gap, fixed a real cron-scaling gap at ~100K-doc scale, specified `evenSplit`'s exact algorithm for the first time, hardened compensation to never resurrect a TTL-deleted doc.
- **v14-v14.2**: senior+QA review found v13's claimed fixes were narrative-only (the code didn't actually do what the prose said) — v14 built the real code; a verification pass (v14.1) then found the real code introduced a NEW performance bug (N sequential Mongo round-trips); the arbiter (v14.2) then found the v14.1 fix for THAT still had a residual correctness race, closed properly. **Arbiter PASS at v14.2.**
- **v15-v18**: a user-directed e2e review across all prior versions found a real, previously-unspecified gap: the ETERNAL (independent NO_LIMIT) counter's MySQL-sync mechanism was undefined. v15-v18 tried a "sentinel row" approach (one fixed MySQL row holding the cumulative total) — this went through 3 consecutive arbiter REFINE cycles fixing: a call to a nonexistent repository method, a migration-vs-live-cron race, a factually-wrong `ON DUPLICATE KEY UPDATE` claim against the real DDL (the table has no unique key on the relevant columns), and a cross-writer sentinel-date-agreement problem.
- **v19**: **the breakthrough.** The user asked to check all callers of `Utils.getDateWithoutTimestampInSpecifiedZone` — this confirmed `TBL_REWARD_ISSUE_SUMMARY` is actually a genuinely daily-bucketed table for ETERNAL constraints too (a new row every day, an unfiltered all-time SUM on read) — exactly like ROLLING. The sentinel-row approach had been fighting the schema's real shape. v19 replaced it entirely with a much simpler mechanism that mirrors legacy's own daily bucketing.
- **v20**: senior+QA review found 5 real bugs in v19's pseudocode (a repository-method signature mismatch, missing org-timezone resolution in the cron's background context, unsafe two-datastore write ordering, a first-tick double-count bug, unspecified negative-delta handling) — all fixed with real code, plus a corrupted job body (leftover text from an earlier edit) rewritten cleanly.
- **v21 (this document)**: QA found 2 more real gaps in v20's fixes — the find-or-create sequence itself had an unaddressed race between two concurrent writers, and a migration-seeding inconsistency. Both fixed.

### A.6 Acceptance criteria (final)
1. `acquireRewardLock` removed entirely for the REWARD-level path.
2. Customer-level rows/enforcement untouched — structurally leak-proof.
3. `TBL_REWARD_ISSUE_SUMMARY` remains the durable GET-API reporting source, cron-synced.
4. Strict, atomic breach prevention by construction — including the multi-counter AND-gate case.
5. Correct for every KPI/window/constraint-multiplicity case, not just the happy path.
6. Scales to 5,000 rpm on one reward with acceptable p99 latency.
7. Cron correctness across the org-sharded Mongo topology.
8. A `LIMIT_VALUE` edit propagates to enforcement within a bounded, short window (~1 cron tick), not "next cycle only."

Criticality: **High**.

* * *

## PART B — THE SOLUTION, END TO END (narrative + literal implementation)

### B.1 The core idea in one paragraph
Instead of one Redis lock serializing every issuance against a reward, we split the reward's `LIMIT_VALUE` into N independent "sub-budgets," one per Mongo document ("slot"). Each issuance does a single **atomic conditional increment** (`consumed += delta` only if `consumed <= subBudget - delta`) against one slot — no lock, no waiting, sub-millisecond. Because the sub-budgets always sum to the reward's true limit, the reward's true total consumption can never exceed the limit, no matter how many requests hit it concurrently. A lightweight background job keeps the sub-budgets fairly distributed across slots, and keeps MySQL's `TBL_REWARD_ISSUE_SUMMARY` table updated for reporting/GET APIs — but MySQL is now purely a *read-side mirror*, never consulted for the enforcement decision itself.

### B.2 Key identity — the counter's "primary key"

**This was a real bug fixed in v7.** An earlier draft (v6) keyed each counter by `(orgId, rewardId, constraintId, windowCycleKey, slot)`. This is WRONG: verified against the real code (`RewardConstraint.isEquivalentToConstraint()`, `RewardConstraint.java:136-166`), the system treats two `Level.REWARD` constraints as "the same counter" if they share the same `(level, kpi)` — **never** by `constraintId`, and never by window/frequency type (`REWARD_CONSTRAINT_ALLOWED_LEVELS = [REWARD, CUSTOMER, TRANSACTION]`, `Constants.java:214`). The existing-row match key in `writeNonOrgSummariesAtomically` (`RewardConstraintFacade.java:661-666`) is `(level, kpi, userId, issueDate)` — `userId` is always null for REWARD level.

**Corrected, current key:**
```
(orgId, rewardId, kpi, windowCycleKey)
```

**What this means practically**: if a reward has TWO active constraints on the same KPI (e.g. one "max 100/day" and one "max 1000/month," both on `QUANTITY`), these are **different counters** (different `windowCycleKey`s) that must BOTH be checked before allowing an issuance (an AND-gate — deny if EITHER would be breached), and BOTH incremented on success. If two constraints resolve to the exact SAME `windowCycleKey` (degenerate case), they share ONE counter and get ONE increment.

⚠️ **Honesty check**: this AND-gate-across-multiple-counters behavior is **new design intent**, extrapolated from the code's row-sharing/dedup mechanics (verified: `writeNonOrgSummariesAtomically`'s `(level|kpi|issueDate)` dedup key, `RewardConstraintFacade.java:658-666`) — but the current legacy code's actual pass/fail path, `reEvaluateConstraints` (`NonOrgSummaryWriteProcessor.java:152-183`), is a simple `for` loop returning `false` on the FIRST failing constraint (`.java:170-179`) and never formally specs an AND-gate across distinct Mongo counters. **Needs explicit product/business sign-off** (Risks §D.2).

**Dependent vs. independent NO_LIMIT constraints**: verified via `getIndependentRewardCustomerNoLimitRestrictions` (`RewardConstraintFacade.java:326-355`). A `NO_LIMIT` constraint sharing its `(level,kpi)` group with a windowed sibling is "dependent" and gets **no counter of its own** — the windowed sibling's counter is authoritative. Only a `NO_LIMIT` constraint that is the *sole* member of its group is "independent" and gets its own **eternal** counter (see B.5).

### B.3 Per-KPI handling — the four KPI types and their quirks

Verified: `KPI.java` has exactly 4 values.

| KPI | What increments the counter | What reverses it | The gotcha |
|---|---|---|---|
| `QUANTITY` | `quantity` | `failedQty` | none — the simple baseline case |
| `POINTS` | `intouchPoints × quantity` (BigDecimal) | `intouchPoints × failedQty`, null-guarded | pre-multiplied once, standard scale/rounding |
| `REDEMPTION_VALUE` | `redemptionValue × quantity` | `redemptionValue × failedQty` | if `redemptionValue` is **null** (no payment config), `resolveAtomicDelta` returns null (`RewardConstraintFacade.java:717`) → the write must be SKIPPED entirely, and the constraint must FAIL (deny), not silently pass. A **revoke** must fetch a fresh per-unit value from the DB FIRST via `fetchPerUnitRedemptionValue(orgId, transactionId)` (`RewardConstraintFacade.java:461-465`) — it must NOT reuse the in-memory delta computed at issuance time. |
| `TRANSACTION_COUNT` (`@Deprecated`, still live) | always `BigDecimal.ONE`, regardless of quantity | `+1` reversal ONLY if `failedQty == totalQty` (full failure), else `0` (`RewardConstraintFacade.java:754-755`) | evaluation for this KPI is **hardcoded to `1`** (`setIsValidForTransaction(1)`, `RewardLevelService.java:84`) — this counter is written for consistency/future-proofing but is never actually the reason an issuance gets denied under current semantics |

⚠️ **Correction:** verified via code inspection — `POINTS` is currently **UNREACHABLE for `Level.REWARD`** via the supported creation path. `RewardConstraintValidation.validateEachLevel()` (`RewardConstraintValidation.java:68-69`) unconditionally rejects `KPI.POINTS` for BOTH `REWARD` and `CUSTOMER` levels at creation/update time (`POINTS_KPI_NOT_SUPPORTED`) — a flat KPI blacklist, not a per-level matrix; the validator's own Javadoc ("only TRANSACTION_COUNT KPI is supported") is stale/inaccurate. HOWEVER the evaluator (`RewardLevelService.evaluate()` → `RewardIssueSummaryContext.setIsValidForPoints()`, `RewardLevelService.java:86-87`, `RewardIssueSummaryContext.java:80-91`) is **fully functional** for REWARD+POINTS — a real limit check, no degenerate handling (only `Level.TRANSACTION` is force-passed). **This design implements POINTS's write/compensate logic defensively anyway** — identical to the other proportional KPIs — for forward-compatibility if that validator is ever relaxed. Test #76 covers this explicitly.

**Structural safeguard**: `compensate()` takes an explicit `CompensationReason{REVOKE, DOWNSTREAM_FAILURE}` parameter — a `REVOKE` call with a null `originalDelta` throws `IllegalStateException` rather than silently reusing a stale in-memory number.

### B.4 Window/frequency matrix — every currently-supported combination

Verified: `WindowType={ROLLING,FIXED}`, `RepeatFrequencyType={DAYS,WEEKS,MONTHS,NO_LIMIT}` (`value_in_days=1/7/30/-1`).

| Window type | Frequency | Where the WRITE lands (`windowCycleKey`) | What the READ (enforcement check) covers |
|---|---|---|---|
| FIXED | DAYS / WEEKS / MONTHS | `eventDate`, via `DailyFixedWindowCycleCalculator`/`WeeklyFixedWindowCycleCalculator` (respects org `weekStartDay`)/`MonthlyFixedWindowCycleCalculator` (calendar `[1st,next-1st)`) | the whole current cycle's range |
| ROLLING | DAYS / WEEKS / MONTHS | `issualDate` (today) — **one slot-set per calendar day** | `[today − (interval × value_in_days − 1) days, today+1)` — **MONTHS here means `interval × 30 calendar days`, NOT calendar months** (existing legacy math, preserved exactly) |
| Independent NO_LIMIT | NO_LIMIT | `eventDate` if a FIXED-non-NO_LIMIT sibling exists in the same request batch, else `issualDate` | ALL TIME, no date filter — never resets |
| Dependent NO_LIMIT | NO_LIMIT | — | no counter exists (§B.2) |

**Backdated request dates (`eventDateTime`)** — confirmed via code inspection: `IssueRewardValidator.validateIssualEventDate()` only rejects a FUTURE date beyond `maxSupportedFutureEventDateSeconds` — there is NO lower/past bound. For **FIXED** windows, a backdated date correctly pivots which cycle the write/read lands in (`LevelService.java:60-88` uses the supplied `eventDate`). For **ROLLING** windows, a backdated date is **silently ignored** — the rolling window is always computed from "now" (`getCurrentInstant()`, `RewardConstraintHelper.java:96-104`), never from `eventDate`. This is pre-existing legacy behavior, preserved exactly by construction. Test #52 proves this explicitly.

**Mid-cycle FIXED-window frequency/interval change** — handled correctly **by construction**: a changed calculator produces a DIFFERENT `windowCycleKey` for the next issuance → naturally creates a NEW meta doc under the new key. The OLD meta doc gets marked `EXPIRED` by the cron's existing rollover-detection (`cycleHasRolledOver`) once the calculator stops mapping back to it. Test #53 proves this explicitly.

**ROLLING's trailing-days MySQL read and the accepted drift risk:** computing `limitValue_today` reads MySQL for the trailing *already-closed* days' totals. This read can, in principle, race a compensation/revoke write landing on one of those closed days at the exact moment it's read. This is the concrete mechanism behind the already-documented "ROLLING compensation-drift" risk (Part D, risk #6) — bounded, self-correcting, not a separate unaddressed problem.

### B.5 The three Mongo collections — WHAT gets created, WHEN, and WHY

#### 1. `rewardCounterSlot` — the actual counter documents

```java
@Document(collection = "rewardCounterSlot")
public class RewardCounterSlot {
    @Id
    private String id; // "<orgId>:<rewardId>:<kpi>:<windowCycleKey>:<slot>" — composite, hand-built, NOT a Mongo auto-ObjectId
    private Long orgId;
    private Long rewardId;
    private KPI kpi;
    private String windowCycleKey; // date bucket, or "ETERNAL"
    private int slot;
    private BigDecimal consumed;   // running total for THIS slot only
    private Date createdOn;
    private Date lastUpdatedOn;
    private Date expireAt;         // TTL field, null for ETERNAL (see B.7)
    // NOTE: deliberately NO limitValue field. limitValue is a per-COUNTER concept living once on
    // RewardCounterCycleMeta; a slot doc only needs meta.slotBudgets[slot] to compare against at
    // write time.
}
```

**When is a slot doc created? LAZILY — on first write, never pre-provisioned.**
```java
Query q = query(where("_id").is(slotDocId).and("consumed").lte(subBudget.subtract(delta)));
Update u = new Update().inc("consumed", delta).set("lastUpdatedOn", now).setOnInsert("createdOn", now);
mongoTemplate.upsert(q, u, RewardCounterSlot.class);   // atomic: creates AND increments in one round trip if absent
```
There is no separate "provisioning" step, no admin action, and no cron involvement in creating a slot doc — it springs into existence the moment any request actually lands on it.

**Why composite `_id` instead of a Mongo auto-`ObjectId` + secondary index?** Because we always *know* the full key before we write — this makes the composite string the ideal `_id`: a direct primary-key point lookup, and write-time uniqueness for free.

#### 2. `rewardCounterCycleMeta` — the "how is this counter's budget currently split" document

```java
@Document(collection = "rewardCounterCycleMeta")
public class RewardCounterCycleMeta {
    @Id
    private String id; // "<orgId>:<rewardId>:<kpi>:<windowCycleKey>" — one doc per COUNTER (not per slot)
    private Long orgId;
    private Long rewardId;
    private KPI kpi;
    private String windowCycleKey;
    private BigDecimal limitValue;         // MIN across co-resident constraints; LIVE-REFRESHED, not frozen
    private int slotCount;                 // ADAPTIVE — computed once at creation via computeEffectiveSlotCount(limitValue)
    private List<BigDecimal> slotBudgets;  // per-slot sub-budget; sum always == limitValue; mutable, cron-rebalanced
    private CycleStatus status;            // ACTIVE | EXPIRED (ETERNAL never EXPIRED)
    private Date windowStartAt;             // null for ETERNAL
    private Date windowEndAt;               // null for ETERNAL; cron rollover check compares now > windowEndAt
    private Boolean allSlotsExhausted;      // ADVISORY ONLY — fast-fail hint, cron-set, never authoritative
    private Date lastSyncedAt;              // last successful MySQL-sync timestamp; drives the hybrid staleness filter
    private BigDecimal lastSyncedCumulativeTotal; // seeded to alreadyConsumed at creation — NEVER left null (see v20/v21 fixes below)
    private Date expireAt;                  // TTL field, null for ETERNAL (see B.7)
    private Date createdOn;
    private Date lastRebalancedAt;
}
```

**When is a meta doc created? ALSO LAZILY — on the first issuance request for that `(rewardId,kpi,windowCycleKey)`.** Race-safe atomic upsert. Continues in Part 2 with the full `getOrCreate()` implementation, `computeEffectiveSlotCount()`, the cron's role, and the `rewardCounterDedupe` collection.

*(Continued in Part 2 of 4 — §B.5 continued, §B.6 LIMIT_VALUE live-refresh, §B.7 TTL, §B.8 the cron jobs.)*
