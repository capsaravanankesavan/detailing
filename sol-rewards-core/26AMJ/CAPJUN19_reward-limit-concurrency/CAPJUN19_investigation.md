# Investigation: Reward Constraint Limit Breach & Duplicate Summary Rows Under Parallel Issue Calls
**Date:** 2026-06-19
**Investigator:** Saravanan Kesavan
**Status:** Ready for Tech Detail
**Confidence:** HIGH

---

## Problem Statement

Under parallel `issueReward` API calls for the same or different customers on the same reward, two
distinct failures surface:

1. **Limit breach** — the constraint's limitValue is exceeded even when each individual request
   passes the constraint evaluation check.
2. **Duplicate rows** — `TBL_REWARD_ISSUE_SUMMARY` accumulates multiple rows for the same logical
   key (orgId + rewardId + constraintLevel + kpi + userId + issueDate), causing the consumed value
   to be under-counted on subsequent reads, which compounds the breach over time.

Scope: affects all constraint levels (CUSTOMER, REWARD, ORG) but is most severe for REWARD- and
ORG-level constraints, which have no per-constraint serialization at all. Observed in production.
Existing duplicate rows in `TBL_REWARD_ISSUE_SUMMARY` prevent adding a DB unique index now.

---

## Root Cause

**The constraint check-then-act sequence is not atomic, and the customer-level Redis lock
serializes only per-customer calls, leaving reward-level and org-level constraints fully exposed
to concurrent violations from parallel requests by different customers. Even within the customer
lock, the UPDATE path is a read-compute-write (not atomic) and duplicate INSERTs can occur when
the row does not yet exist.**

### Three Contributing Races

#### Race A — Wrong lock granularity for reward/org-level constraints

The `@CustomerLockable` Redis lock key is `orgId + "_" + customerId`
([CustomerLockManager.java:62](src/main/java/com/capillary/solutions/rewards/lock/CustomerLockManager.java)).
For a reward-level constraint (one that limits total issuances of a reward across all customers),
two different customers' threads hold different locks and run the check concurrently:

```
Customer A thread (lock: org1_custA):
  getConsumedValueFromDB(reward_R, level=REWARD) → consumed = 9  (limit = 10)
  evaluate: 9 + 1 ≤ 10 → PASS

Customer B thread (lock: org1_custB) [concurrent, no shared lock]:
  getConsumedValueFromDB(reward_R, level=REWARD) → consumed = 9  (limit = 10)
  evaluate: 9 + 1 ≤ 10 → PASS

Both proceed → 11 total issued against a limit of 10
```

#### Race B — Duplicate INSERT on concurrent first-issuance

`RewardConstraintFacade.updateSummaries()` ([RewardConstraintFacade.java:328](src/main/java/com/capillary/solutions/rewards/service/impl/RewardConstraintFacade.java))
re-reads the DB, runs `buildUniqueSummaries()` over the in-memory result set, and calls
`bulkSaveOrUpdate()`. When no row yet exists for today's constraint window:

```
T1: getExistingRewardIssueSummariesForNonOrgLevel() → []
T2: getExistingRewardIssueSummariesForNonOrgLevel() → []    (T1 hasn't written yet)
T1: buildUniqueSummaries() creates new object (id=null) → bulkSave() → INSERT row #1
T2: buildUniqueSummaries() creates new object (id=null) → bulkSave() → INSERT row #2

DB now has two rows for the same (orgId, rewardId, level, kpi, userId, issueDate).
Next call's getConsumedValueFromDB() SUMs or picks one row → under-counts consumed.
```

#### Race C — Lost UPDATE on concurrent increments

`bulkUpdate()` ([RewardIssueSummaryJdbcRepository.java:137](src/main/java/com/capillary/solutions/rewards/jdbc/RewardIssueSummaryJdbcRepository.java))
computes the new value in application memory then issues a blind `UPDATE SET CONSUMED = :computedValue`.
If two threads fetch `consumed = 5`, both compute `6`, both write `6` — one increment is silently
lost.

### Why the customer lock didn't catch it

For same-customer calls (Race A can't happen), the lock serializes correctly **if** the lock
TTL survives the full chain. But vendor, coupon-issue, and points-redeem external calls
happen inside the lock and can be slow. If the lock TTL (`redis.lock.ttl`) is shorter than the
P99 of the full issue chain, the second call acquires the lock while the first is still running.
Confirm by checking the configured TTL vs observed P99 for `/issueReward`.

---

## Evidence Trail

| File | Line | Finding |
|------|------|---------|
| `CustomerLockManager.java` | 62 | Lock key = `orgId + "_" + customerId` only |
| `CustomerLockManager.java` | 35 | Lock skipped entirely if `lockKey == null` |
| `RewardConstraintProcessor.java` | 100–121 | `loadKPI → refreshSummary → evaluate` is non-atomic (pure DB read) |
| `LevelService.java` | 53–146 | `getConsumedValueFromDB` — live MySQL read with no row lock |
| `RewardConstraintFacade.java` | 328–373 | `updateSummaries` re-reads DB, no lock on the summary row |
| `RewardIssueSummaryJdbcRepository.java` | 153–167 | `bulkSaveOrUpdate` splits on `id==null`, plain INSERT for new rows |
| `RewardIssueSummaryJdbcRepository.java` | 96–103 | UPDATE is a blind `SET CONSUMED=:computedValue` (not `consumed + delta`) |
| `BulkIssueService.java` | 96–117 | Processor chain order: constraint check at position 12, well after customer lock acquired |
| `UserRewardUtils.java` | 56 | `@CustomerLockable` on `issueRewardBulk()` — lock held over full chain |
| `UserRewardUtils.java` | 73 | `updateSummaries()` called inside the lock but AFTER external calls |

---

## System Map (Affected Path)

**Call chain:**
```
Controller
  → UserRewardUtils.issueReward()           @CustomerLockable(orgId, customerId)
      → BulkIssueService.issueBulk()
          → [processors 1–11: payment, catalog, org settings, idempotency, ...]
          → RewardConstraintProcessor         ← constraint check (Race A, C occur here)
          → [couponIssue, vendorIssue, pointsRedeem — external I/O, slow]
      → UserRewardFacade.updateSummaries()   ← DB write (Race B occurs here)
      → [save user rewards, links, idempotency]
                                             [lock released]
```

**Tenant isolation:** `orgId` is in every constraint query and Redis lock key. No cross-org
bleed risk. The gap is cross-customer-within-same-org for shared reward budgets.

**Upstreams called (inside lock):** MySQL (constraint read), Intouch API (coupon issue),
vendor APIs, points engine. All slow paths.

**Downstreams affected:** `TBL_REWARD_ISSUE_SUMMARY` (constraint accounting),
`TBL_USER_REWARD` (issued rewards), MongoDB `issued_transactions`.

---

## Solution Approaches Considered

Two approaches were evaluated. **Approach A is chosen for Phase 1** — lower implementation surface, no new infrastructure. Approach B remains the right escalation path if lock contention becomes a bottleneck at scale.

---

### Approach A — Redis Distributed Lock + Atomic DB Increment *(Chosen)*

Serialize the reward-level constraint check-and-write window at the DB layer. Eliminates Gaps A, B, and C without introducing new infrastructure dependencies.

#### Phase 1 — Reward-level constraints (`REWARD` + `CUSTOMER` levels)

**High-level flow:**

```
[pos 12] RewardConstraintProcessor — existing, unchanged evaluation
│
│  ← acquire reward_constraint:{orgId}:{rewardId}   (conditional — see below)
│
[pos 13] NonOrgSummaryWriteProcessor — NEW, runs inside lock
│     refreshSummary()       ← live SUM(consumed) from MySQL
│     evaluate()             ← in-memory limit check
│     FAIL → release lock, reject reward
│     PASS → writeNonOrgSummaries() with atomic increment
│  ← release lock
│
[pos 14] CouponIssueProcessor   ← external calls run OUTSIDE lock
[pos 16] VendorIssueProcessor
[pos 17] PointsRedeemProcessor
│
└── on any failure after pos 13:
    compensate: UPDATE SET CONSUMED = CONSUMED - :delta WHERE ID = :id
```

`updateSummaries()` in `UserRewardUtils` continues to handle org-level summary writes post-issuance, unchanged in Phase 1.

**Lock key and conditional acquisition:**

```
Key:  reward_constraint:{orgId}:{rewardId}
```

Acquired only when the reward has at least one `REWARD`-level constraint (`userId = null` in summary). Rewards with only `CUSTOMER`-level constraints have no cross-customer row conflict and skip the lock.

```java
boolean hasRewardLevelConstraint = validConstraints.stream()
    .anyMatch(c -> c.getConstraintLevel() == Level.REWARD);
if (hasRewardLevelConstraint) {
    redisLockService.acquireLock("reward_constraint:" + orgId + ":" + rewardId);
}
```

Why not finer granularity (`rewardId + kpi`): multiple `REWARD`-level constraints across different KPIs are processed in the same loop — a thread would need all fine-grained keys simultaneously, giving identical serialization with no throughput gain.

`CUSTOMER`-level rows are per-user (different customers write different rows) — no cross-customer conflict. They fall inside the lock when a `REWARD`-level constraint exists but cause no additional contention.

Same-customer concurrent calls: already serialized by `@CustomerLockable`. No change.

Lock hold duration: ~10–50ms (DB read + evaluate + DB write only).

**Fixes applied:**

| Gap | Fix |
|---|---|
| A | `reward_constraint:{orgId}:{rewardId}` lock serializes cross-customer access to `REWARD`-level shared rows |
| B | Lock ensures only one thread reaches `writeNonOrgSummaries()` per rewardId at a time — INSERT race eliminated structurally |
| C | `UPDATE SET CONSUMED = CONSUMED + :delta WHERE ID = :id` replaces blind overwrite |
| D | `buildUniqueSummaries()` working set rebuilt from scratch; `HashSet` reference-identity deduplication removed |

**Compensating decrement:**

If any processor after position 13 fails, the summary rows are already incremented. Compensation:
```sql
UPDATE TBL_REWARD_ISSUE_SUMMARY SET CONSUMED = CONSUMED - :delta WHERE ID = :id
```
Written row IDs must be retained in `BulkRewardIssueContext.writtenSummaryRows` and passed to the failure handler. Scope: all constraints that passed for this reward (both `REWARD`-level and `CUSTOMER`-level rows).

**Performance prerequisite — index on `TBL_REWARD_ISSUE_SUMMARY`:**

The `refreshSummary()` SUM query runs inside the lock. Lock hold time is proportional to query latency.

Query is bounded by window size (WEEKLY ≤ 7 rows, MONTHLY ≤ 31 rows) regardless of reward age — but only if this composite index exists:

```sql
(REWARD_ID, CONSTRAINT_LEVEL, KPI, ISSUE_DATE)
```

Without it, the query degrades to a full reward-history scan and directly extends lock hold time. **Verify this index exists before deploy. Add as a DBA prerequisite migration if missing.**

#### Phase 2 — Org-level constraints (deferred)

Org-level constraints use `rewardId = -1L` and may have multiple constraints per org. Lock strategy:

- One key per org-level constraint: `org_constraint:{orgId}:{constraintId}`
- Acquire all relevant keys atomically via existing `acquireBulkLock(List<String> keys)` — already deadlock-safe (sorts keys alphabetically, releases all on any failure)

Phase 2 begins only after Phase 1 is stable in production.

---

### Approach B — 3-Tier Atomic Counter (Redis → MongoDB → MySQL)

Since we cannot add a DB unique constraint (existing duplicate rows block it), and Redis has no
AOF persistence, the design uses three tiers with clear responsibility:

```
┌──────────────────────────────────────────────────────────────────┐
│  TIER 1 — Redis (hot path, volatile)                             │
│  Atomic check-and-reserve via Lua script.                        │
│  Source of truth for "can this issuance proceed right now?"      │
│  No persistence (AOF off) — loses state on restart.             │
├──────────────────────────────────────────────────────────────────┤
│  TIER 2 — MongoDB (durable intra-day journal)                    │
│  Append-only log of COMMITTED / ROLLED_BACK increment events.    │
│  Used to re-seed Redis on restart or cold-start.                 │
│  Source of truth for "what actually happened today?"             │
├──────────────────────────────────────────────────────────────────┤
│  TIER 3 — MySQL TBL_REWARD_ISSUE_SUMMARY (end-of-day settle)    │
│  Receives one aggregated row per constraint per window date.     │
│  Written by a scheduled flush job after the day closes.          │
│  Source of truth for Databricks ETL delta loads.                 │
└──────────────────────────────────────────────────────────────────┘
```

### Tier 1: Redis Atomic Check-and-Reserve

**Key pattern:**
```
constraint_counter:{orgId}:{constraintId}:{windowAnchorDate}
e.g.  constraint_counter:1001:42:2026-06-19
```

`windowAnchorDate` = the start of the current constraint window in org timezone:
- `DAILY` → today's date (ISO, "2026-06-19")
- `WEEKLY` → ISO date of the week's configured start day
- `MONTHLY` → ISO date of the 1st of the month
- `NO_LIMIT` → constraint creation date (acts as a permanent single bucket)
- `ROLLING` → **not handled in Phase 1; see Scope / Out of Scope below**

**Redis key TTL:**
```
TTL = secondsRemainingInCurrentWindow + 3_600   // 1-hour tail buffer for in-flight requests
```
This ensures in-flight requests that were validated against the old window can still write their
MongoDB journal entry without the Redis key expiring under them.

**Lua check-and-reserve (atomic at Redis level):**
```lua
-- KEYS[1] = counter key
-- ARGV[1] = delta (as integer × 10000 for 4-decimal precision)
-- ARGV[2] = limit (as integer × 10000)
local current = tonumber(redis.call('GET', KEYS[1]) or '0')
local delta    = tonumber(ARGV[1])
local limit    = tonumber(ARGV[2])
if current + delta > limit then
    return -1   -- limit would be exceeded
end
return redis.call('INCRBY', KEYS[1], delta)
```

KPI values are stored as `BigDecimal × 10000` (long integer) to avoid `INCRBYFLOAT` precision
issues. `QUANTITY` and `TRANSACTION_COUNT` KPIs multiply by 10000 too (always integers anyway).

**On Lua return = -1:** constraint evaluation fails — reject the request (same StatusCode as today:
`CONSTRAINT_EVALUATION_FAILED`).

**On Lua return > 0:** capacity reserved — proceed with issuance. Store `reservedDelta` in
`RewardIssueSummaryContext` for potential rollback.

### Redis Cold-Start / Key-Miss Initialization

When the Redis key does not exist (new day start OR Redis restart):

```
1. Acquire a short init lock:
     "constraint_init_lock:{orgId}:{constraintId}:{windowAnchorDate}"  TTL=10s

2. Double-check: if key now exists (raced to init), release lock, proceed to Lua.

3. Aggregate MongoDB journal for this (orgId, constraintId, windowAnchorDate, status=COMMITTED):
     db.constraint_increments.aggregate([
       { $match: { orgId, constraintId, windowAnchorDate, status: "COMMITTED" } },
       { $group: { _id: null, total: { $sum: "$kpiDelta" } } }
     ])
   → gives the actual consumed so far today.

4. SET constraint_counter key (initialValue × 10000) NX EX <ttl>
   (SET NX ensures atomicity — only one thread's value wins even under race).

5. Release init lock.
6. Retry the Lua check-and-reserve.
```

**Why this handles Redis restart mid-day correctly:**
MongoDB journal has all COMMITTED events for today. SUM gives the exact consumed. Redis is
re-seeded to the correct value. No capacity is given away that was already consumed.

**Why this handles new-day start correctly:**
At midnight, the new day's windowAnchorDate changes. No MongoDB journal entries exist yet
for the new date → SUM = 0 → Redis initialized to 0. Clean slate for the new day.

### Tier 2: MongoDB Journal

**Collection:** `constraint_increments` (new — discuss naming with team before creating)

```
{
  _id:              ObjectId,
  orgId:            Long,
  constraintId:     Long,
  windowAnchorDate: String,     // "2026-06-19" — same value used in Redis key
  kpiDelta:         Long,       // stored as integer × 10000 (same as Redis)
  kpi:              String,     // KPI enum name
  requestId:        String,     // for deduplication / audit
  status:           String,     // "COMMITTED" | "ROLLED_BACK"
  createdAt:        Date,
  lastUpdatedAt:    Date
}

Required indexes:
  { orgId: 1, constraintId: 1, windowAnchorDate: 1, status: 1 }  // aggregation queries
  { createdAt: 1, expireAfterSeconds: 604800 }                    // TTL: auto-delete after 7 days
```

**Write on successful issuance (synchronous):**

```
ConstraintIncrementDocument doc = new ConstraintIncrementDocument(
    orgId, constraintId, windowAnchorDate,
    kpiDelta × 10000, kpi, requestId, "COMMITTED");
constraintIncrementDao.save(doc);
```

This write is synchronous. If it fails, roll back Redis (`DECRBY` by the reserved delta)
and fail the request. This keeps Redis ≤ MongoDB at all times (Redis reflects only what MongoDB
has durably recorded).

**Write on rollback (issuance failed after constraint check):**

```
constraintIncrementDao.save(... status="ROLLED_BACK");
redisConstraintCounter.decrBy(key, reservedDelta);
```

### Tier 3: End-of-Day MySQL Flush

A scheduled job runs at **01:00 AM in org timezone** (1-hour buffer past midnight to allow
in-flight requests from 23:59 to complete their MongoDB writes):

```
For each unique (orgId, constraintId, windowAnchorDate) in yesterday's MongoDB journal:
  total = SUM(kpiDelta WHERE status=COMMITTED)
  
  INSERT INTO TBL_REWARD_ISSUE_SUMMARY
    (ORG_ID, REWARD_ID, KPI, CONSTRAINT_LEVEL, CONSTRAINT_ATTRIBUTE,
     USER_ID, CONSUMED, ISSUE_DATE, LAST_UPDATED_ON, ...)
  VALUES
    (:orgId, :rewardId, :kpi, :level, :attribute,
     :userId, :total / 10000.0, :windowAnchorDate, NOW(), ...)
```

No `ON DUPLICATE KEY UPDATE` needed — this is an append-only write of one new row per
constraint per day. Databricks delta ETL reads by `LAST_UPDATED_ON > yesterday` and will pick
up this new row. Existing duplicate rows from before the fix continue to exist and are harmless
for ETL (Databricks SUM aggregates them correctly).

**The existing `updateSummaries()` MySQL write path is DISABLED** for constraints tracked
via Redis (see `What Changes` section). The flush job becomes the sole MySQL writer for
the summary table going forward.

### Day Transition Edge Cases

```
┌──────────────────────────┬────────────────────────────────────────────────────┐
│ Scenario                 │ Handling                                           │
├──────────────────────────┼────────────────────────────────────────────────────┤
│ Request straddles         │ windowAnchorDate computed ONCE at constraint check  │
│ midnight (starts 23:59,  │ time and stored in RewardIssueSummaryContext.       │
│ MongoDB write at 00:00)  │ MongoDB entry written with old day's date → correct │
│                          │ flush job picks it up the next day at 01:00 AM.    │
├──────────────────────────┼────────────────────────────────────────────────────┤
│ Multiple threads at       │ All call SET NX for new day key. Only one succeeds. │
│ 00:00:00 (new day burst) │ Others proceed normally since key now exists.       │
│                          │ No duplicate init because SET NX is atomic.         │
├──────────────────────────┼────────────────────────────────────────────────────┤
│ Redis restart mid-day    │ Key miss triggers init path. MongoDB journal has    │
│                          │ all COMMITTED events → SUM re-seeds Redis to exact  │
│                          │ consumed value. No capacity given away incorrectly. │
├──────────────────────────┼────────────────────────────────────────────────────┤
│ Redis unavailable         │ Fall back to per-constraint Redis lock (lock key:   │
│ (complete outage)        │ "constraint_lock:{orgId}:{constraintId}") with      │
│                          │ MySQL as the check source. Slower but correct.      │
│                          │ Implement behind a circuit-breaker flag.            │
├──────────────────────────┼────────────────────────────────────────────────────┤
│ Flush job misses an entry │ Entries with createdAt near midnight for windowDate │
│ (timing race)            │ = D are caught because flush runs at 01:00 AM, not  │
│                          │ at 00:00. 1-hour buffer is intentional.             │
├──────────────────────────┼────────────────────────────────────────────────────┤
│ MongoDB write fails after │ Redis DECRBY rolls back reserved capacity.          │
│ Redis increment          │ No MongoDB entry → Redis and MongoDB stay in sync.  │
│                          │ Request fails with error (retry by caller).         │
└──────────────────────────┴────────────────────────────────────────────────────┘
```

### Out of Scope — ROLLING Window Constraints (Phase 2)

For `WindowType.ROLLING` constraints (e.g., "max 5 per 7 rolling days"), the Redis key
cannot use a fixed anchor date because the window slides with each call. These are handled
in Phase 2 with one of:
- Redis sorted-set per day accumulator + ZRANGEBYSCORE sum
- Per-constraint Redis lock + MySQL read (simpler, lower throughput)

Verify with the team what percentage of active constraints use `WindowType.ROLLING` before
prioritising Phase 2. Phase 1 covers `WindowType.FIXED` and `RepeatFrequencyType.NO_LIMIT`.

---

## What Changes

### Approach A (Chosen) — Phase 1

| Component | Change | File |
|-----------|--------|------|
| New | `NonOrgSummaryWriteProcessor` — processor at position 13; acquires `reward_constraint:{orgId}:{rewardId}` lock (conditional), runs `refreshSummary + evaluate + writeNonOrgSummaries`, releases lock | `service/impl/issue/processors/NonOrgSummaryWriteProcessor.java` |
| Modify | `RewardConstraintFacade.buildUniqueSummaries()` — rebuild working set from scratch (remove `HashSet` reference-identity dedup); add field-equality check for dedup | `service/impl/RewardConstraintFacade.java` |
| Modify | `RewardIssueSummaryJdbcRepository.bulkUpdate()` — change `SET CONSUMED = :computedValue` to `SET CONSUMED = CONSUMED + :delta` | `jdbc/RewardIssueSummaryJdbcRepository.java` |
| Modify | `BulkRewardIssueContext` — add `writtenSummaryRows: List<RewardIssueSummary>` to carry written row IDs through to the failure compensation handler | `dto/wrapper/BulkRewardIssueContext.java` |
| Modify | `UserRewardUtils.issueReward()` / `issueRewardBulk()` — add compensation call in the failure path: `UPDATE SET CONSUMED = CONSUMED - :delta` for all rows in `writtenSummaryRows` | `service/UserRewardUtils.java` |
| Config | Add `reward_constraint:` lock TTL to `sol-rewards-core-a.json`; propagate to all 8+ prod clusters | `sol-rewards-core-a.json` |

### Approach B (Alternative) — for reference if Approach A escalation is needed

| Component | Change | File |
|-----------|--------|------|
| New | `ConstraintCounterService` — Redis Lua check-and-reserve + init-from-Mongo | `service/constraint/ConstraintCounterService.java` |
| New | `ConstraintIncrementDocument` — MongoDB document entity | `db/mongo/ConstraintIncrementDocument.java` |
| New | `ConstraintIncrementDao` — Spring Data MongoRepository | `mongoDao/ConstraintIncrementDao.java` |
| New | `DailyConstraintFlushJob` — scheduled end-of-day MySQL write | `service/constraint/DailyConstraintFlushJob.java` |
| Modify | `RewardConstraintProcessor.process()` — replace `loadKPI + refreshSummary + evaluate` with `ConstraintCounterService.checkAndReserve()` for FIXED/NO_LIMIT window constraints; keep existing path for ROLLING | `service/impl/issue/processors/RewardConstraintProcessor.java` |
| Modify | `RewardConstraintFacade.updateSummaries()` — write to MongoDB journal; disable MySQL write for non-ROLLING constraints | `service/impl/RewardConstraintFacade.java` |
| Modify | `RewardIssueSummaryContext` — add fields: `windowAnchorDate` (Date), `reservedKpiDelta` (Long scaled), `redisKey` (String) | `dto/wrapper/RewardIssueSummaryContext.java` |
| Modify | `UserRewardUtils.issueReward()` / `issueRewardBulk()` — add rollback call in the failure path (post-issueBulk, before response) | `service/UserRewardUtils.java` |

## What Does NOT Change (Explicit Out of Scope)

- DB schema of `TBL_REWARD_ISSUE_SUMMARY` — no new columns, no index changes
- `LevelService` implementations and `refreshSummary` — retained for ROLLING window path
  and for the `fetchCurrentRewardSummary` read API (get-user-rewards)
- `CustomerLockManager` — customer lock stays as-is (still useful for per-customer CUSTOMER-level constraints)
- `bulkSaveOrUpdate()` — retained as-is; pos 13 calls it for REWARD/CUSTOMER-level writes inside the lock; ROLLING constraints still go through `updateSummaries()` unchanged
- Databricks ETL pipeline — no change; it reads `TBL_REWARD_ISSUE_SUMMARY` the same way
- Existing duplicate rows in `TBL_REWARD_ISSUE_SUMMARY` — not cleaned up in this ticket;
  MySQL remains the authoritative store; the Redis lock prevents new duplicates from forming

---

## Assumptions Going Into Tech Detail

1. **`RedisLockService.acquireLock()` accepts a per-call TTL** — if TTL is global-only config
   (`redis.lock.ttl`), the lock TTL is fixed for the entire process and cannot be tuned per
   constraint type. Breaks if Phase 2 needs different TTLs for ROLLING vs FIXED windows.
2. **Processor chain order is configurable between pos 12 and pos 14** — pos 13 (`NonOrgSummaryWriteProcessor`)
   must be inserted after pos 12 and before pos 14 (CouponIssueProcessor). Breaks if the chain
   order is hardcoded or validated by a sequence check at startup.
3. **`BulkRewardIssueContext` is accessible to all processors in the chain** — pos 13 writes
   `writtenSummaryRows`; the updated `updateSummaries()` reads the same object to filter.
   Breaks if context is not propagated as a shared mutable object in the chain.
4. **`updateSummaries()` can be modified to skip non-org-level rows** without breaking org-level
   writes — the filter on `writtenSummaryRows` must only exclude REWARD and CUSTOMER-level
   rows; org-level rows (`userId == null AND rewardId != -1L` for REWARD, `userId != null` for
   CUSTOMER) must continue as before.
5. **Compensation decrement path covers all partial-failure scenarios** — if any processor after
   pos 13 throws and the request is abandoned, `BulkRewardIssueContext.writtenSummaryRows` is
   read to issue `CONSUMED = CONSUMED - delta` for each written row. Breaks if the failure path
   exits without calling the compensation logic.
6. **ROLLING window constraints are a known minority** — Phase 1 explicitly leaves ROLLING
   unprotected. If ROLLING is a majority (>50% of active constraints), Phase 2 urgency changes
   and should be raised to the same sprint rather than deferred.

---

## Risks for Implementer

| Risk | Likelihood | Mitigation |
|------|-----------|-----------|
| Compensation path not triggered on all failure paths — partial bulk failures leave CONSUMED over-decremented | Medium | Wrap pos 13 write and compensation in a try-finally keyed on `writtenSummaryRows`; call compensation from the top-level error handler, not only from pos 13. |
| `lastUpdatedOn` not updated by new compensation `bulkUpdate()` SQL — breaks Databricks delta ETL which depends on `lastUpdatedOn` for daily delta loads | High | Add `lastUpdatedOn = NOW()` to both the increment SQL and the compensation decrement SQL. Verify via `.context/infra.md` guardrail: "Every rewards_utf8_db table write must populate `lastUpdatedOn`". |
| Lock TTL misconfigured below GC pause headroom — lock expires while thread is in GC stop-the-world, another thread enters | Low | Set lock TTL = MAX(current `redis.lock.ttl`, P99 of `/issueReward` + 5s buffer). Confirm via New Relic before build. |
| ROLLING window constraints still exposed to Race A and Race C under Phase 1 — no fix in this ticket | High (known) | Explicitly note in code comment on `NonOrgSummaryWriteProcessor` that ROLLING constraints are excluded. Add monitoring alert on CONSUMED breach for ROLLING-type constraints as a proxy to detect Phase 2 urgency. |
| `updateSummaries()` double-write if CQ9 coordination is implemented incorrectly — filter uses wrong field (e.g., `rewardId` instead of row `id`) | Medium | Unit-test filter logic with a mix of REWARD-level (excluded) and ORG-level (included) rows in the same context. Integration test T4b should catch this. |

---

## Open Questions (Must Resolve Before Build)

- [ ] **What is the current `redis.lock.ttl` value, and what is the P99 of `/issueReward` end-to-end?**
  — Critical: TTL must exceed P99 + GC pause buffer. If TTL < P99, two threads can hold the lock simultaneously. Owner: DevOps / New Relic dashboard.
- [ ] **Does `RedisLockService.acquireLock()` accept a custom TTL per call, or is TTL global-only?**
  — Determines whether Phase 1 can tune lock TTL per constraint type. Owner: implementer (read `RedisLockService.java`).
- [ ] **What percentage of active reward constraints use `WindowType.ROLLING`?**
  — Determines Phase 2 urgency; if > 30%, Phase 2 should not be deferred. Owner: data team.

---

## What to Watch Post-Deploy

- **New Relic custom attribute** `constraint_lock_acquired` — emit from `NonOrgSummaryWriteProcessor`
  so we can measure how often the lock path is exercised vs skipped (no REWARD-level constraint).
- **Redis lock contention** — if `acquireLock()` returns false (lock already held), log a
  `LOCK_CONTENTION` warning with `rewardId`. A spike means concurrent load is higher than expected.
- **Compensation decrement fire rate** — log `CONSTRAINT_COMPENSATION_DECREMENT` with count when
  triggered. Should be near-zero under normal operation; any non-zero value deserves investigation.
- **TBL_REWARD_ISSUE_SUMMARY breach rate** — query for rows where `CONSUMED > limitValue` for
  REWARD-level constraints. Should drop to zero for FIXED/NO_LIMIT after deploy; ROLLING will still show.
- **P99 of `NonOrgSummaryWriteProcessor.process()`** — via New Relic `@NewRelicCustomStats`.
  Alert if > 50ms (indicates DB or lock contention under high load).
- **Validate for one org in staging**: run 50 parallel `issueReward` calls for a reward with
  limit=10; confirm exactly 10 succeed and 40 get `CONSTRAINT_EVALUATION_FAILED`.

---

## Integration Test Plan

**Purpose:** These tests must reproduce every race condition in IT before the fix is applied. Pre-fix, the assertions fail — that failure is the evidence. Post-fix, the same assertions pass. No separate "bug reproduction" script needed; the test IS the reproduction.

**Test class:** `src/test/java/com/capillary/solutions/rewards/integeration/RewardConstraintConcurrencyIntegrationTest.java`  
Extends `BaseIntegrationTest`. `@BeforeEach tearDown()` from the base class clears all tables — each test starts clean.

---

### Why existing tests don't cover this

`ParallelCallIntegrationTest.shouldTakeLockOkOnCustomerLevel` fires concurrent calls from the **same customer** (same mobile, same lock key). It correctly tests the customer lock — but never tests different customers issuing the same reward. That is the exact Race A gap. The mobile number parses directly to `userId` (confirmed at line 182 of `ParallelCallIntegrationTest`), so different mobiles = different customers = different lock keys = the race is real and observable in IT.

---

### Concurrency harness (shared helper method)

Use `CountDownLatch` — not `CompletableFuture.supplyAsync` — to park all threads until all are created, then release them simultaneously. This maximises the race window reproducibly in CI.

```java
private List<UserRewardBulkIssueResponse> fireParallelFromDifferentCustomers(
        Long rewardId, int threadCount, int qtyEach) throws InterruptedException {

    List<UserRewardBulkIssueResponse> responses = Collections.synchronizedList(new ArrayList<>());
    CountDownLatch startGate = new CountDownLatch(1);
    CountDownLatch endGate   = new CountDownLatch(threadCount);

    for (int i = 0; i < threadCount; i++) {
        String mobile = "10000000" + String.format("%02d", i); // distinct customer per thread
        RewardIssueRequest req = buildIssueRequest(rewardId, mobile, qtyEach);
        new Thread(() -> {
            try   { startGate.await(); responses.add(issueBulkRewards(req)); }
            catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            finally { endGate.countDown(); }
        }).start();
    }
    startGate.countDown();              // release all threads simultaneously
    endGate.await(30, TimeUnit.SECONDS);
    return responses;
}
```

---

### T1 — Race A + B + C combined: REWARD-level QUANTITY FIXED MONTHLY limit breach

| | |
|---|---|
| **Reward config** | CART_PROMOTION, REWARD-level QUANTITY FIXED MONTHLY, `limit = 5` |
| **Load** | 6 threads × 6 different customers, `qty = 1` each (total attempted = 6 > 5) |
| **Pre-fix API failure** | `successCount == 6` instead of 5 (all pass because both threads read consumed=0 before either writes) |
| **Pre-fix DB failure** | 6 duplicate REWARD-level rows each showing `consumed = 1`; sum happens to be correct but each row is individually wrong |

```java
@Test
void shouldEnforceRewardLevelQuantityLimit_FixedMonthly_DifferentCustomers() throws Exception {
    RewardConstraintRequest c = new RewardConstraintRequest();
    c.setRewardLevel(List.of(buildConstraint(KPI.QUANTITY, 5, FIXED, MONTHS)));
    Long rewardId = createRewardWithConstraints(c).getReward().getId();

    List<UserRewardBulkIssueResponse> responses = fireParallelFromDifferentCustomers(rewardId, 6, 1);

    // API assertions
    assertEquals(5, countByCode(responses, 200),
        "Exactly 5 must succeed. PRE-FIX: 6 succeed — limit breached.");
    assertEquals(1, countByCode(responses, CONSTRAINT_EVALUATION_FAILED),
        "Exactly 1 must be rejected. PRE-FIX: 0 rejected.");

    // DB assertions
    List<RewardIssueSummary> rewardRows = rewardIssueSummaryRepository.findAll().stream()
        .filter(s -> s.getLevel() == Level.REWARD && s.getKpi() == KPI.QUANTITY)
        .collect(toList());

    assertEquals(1, rewardRows.size(),
        "Must be exactly 1 REWARD-level row. PRE-FIX: 6 duplicate rows.");
    assertEquals(0, BigDecimal.valueOf(5).compareTo(rewardRows.get(0).getConsumed()),
        "Consumed must equal limit exactly. PRE-FIX: each row shows 1 not 5.");
}
```

---

### T2 — Race A: REWARD-level QUANTITY ROLLING window

Same as T1 but `windowType = ROLLING, intervalValue = 7, repeatFrequencyType = DAYS`. Confirms Race A is not specific to FIXED windows.

```java
@Test
void shouldEnforceRewardLevelQuantityLimit_Rolling7Days_DifferentCustomers() throws Exception {
    RewardConstraintRequest c = new RewardConstraintRequest();
    c.setRewardLevel(List.of(buildConstraint(KPI.QUANTITY, 5, ROLLING, DAYS, 7)));
    Long rewardId = createRewardWithConstraints(c).getReward().getId();

    List<UserRewardBulkIssueResponse> responses = fireParallelFromDifferentCustomers(rewardId, 6, 1);

    assertEquals(5, countByCode(responses, 200),
        "Exactly 5 must succeed. PRE-FIX: 6 succeed.");
    assertEquals(1, countByCode(responses, CONSTRAINT_EVALUATION_FAILED),
        "Exactly 1 must be rejected. PRE-FIX: 0 rejected.");

    BigDecimal consumed = rewardIssueSummaryRepository
        .fetchKPIConsumedValueForRewardId(Fixtures.ORG_ID, rewardId, KPI.QUANTITY, Level.REWARD);
    assertTrue(consumed.compareTo(BigDecimal.valueOf(5)) <= 0,
        "Total consumed must not exceed 5. PRE-FIX: consumed = 6.");
}
```

---

### T3 — Race A: REWARD-level POINTS FIXED WEEKLY limit breach

| | |
|---|---|
| **Reward config** | INTOUCH_REWARD, REWARD-level POINTS FIXED WEEKLY, `limit = 100`, `points = 30` per call |
| **Load** | 4 threads × 4 different customers (30×3=90 ≤ 100 passes; 4th: 90+30=120 > 100) |
| **Pre-fix** | All 4 succeed (120 pts issued against 100pt limit) |

```java
@Test
void shouldEnforceRewardLevelPointsLimit_FixedWeekly_DifferentCustomers() throws Exception {
    RewardConstraintRequest c = new RewardConstraintRequest();
    c.setRewardLevel(List.of(buildConstraint(KPI.POINTS, 100, FIXED, WEEKS)));
    Long rewardId = createIntouchRewardWithConstraints(c, 30 /*points per call*/).getReward().getId();

    List<UserRewardBulkIssueResponse> responses = fireParallelFromDifferentCustomers(rewardId, 4, 1);

    assertEquals(3, countByCode(responses, 200),
        "Exactly 3 must succeed (90 pts). PRE-FIX: 4 succeed, 120 pts issued against 100pt limit.");
    assertEquals(1, countByCode(responses, CONSTRAINT_EVALUATION_FAILED),
        "Exactly 1 must be rejected. PRE-FIX: 0 rejected.");

    BigDecimal consumed = rewardIssueSummaryRepository
        .fetchKPIConsumedValueForRewardId(Fixtures.ORG_ID, rewardId, KPI.POINTS, Level.REWARD);
    assertTrue(consumed.compareTo(BigDecimal.valueOf(100)) <= 0,
        "Consumed must not exceed 100. PRE-FIX: consumed = 120.");
}
```

---

### T4 — Race B isolated: duplicate summary rows, API succeeds but DB is corrupt

**Critical test.** All API calls succeed (limit is high enough), yet the DB is silently wrong. This mirrors exactly how the bug hid in production.

| | |
|---|---|
| **Reward config** | CART_PROMOTION, REWARD-level QUANTITY FIXED MONTHLY, `limit = 100` (not a bottleneck) |
| **Load** | 5 threads × 5 different customers, `qty = 1` each — all should succeed |
| **Pre-fix API** | Passes (all 5 succeed) |
| **Pre-fix DB failure** | 5 duplicate REWARD-level rows exist, each `consumed = 1`; future UPDATE will only fix one row |

```java
@Test
void shouldProduceSingleRewardLevelSummaryRow_NoDuplicates() throws Exception {
    RewardConstraintRequest c = new RewardConstraintRequest();
    c.setRewardLevel(List.of(buildConstraint(KPI.QUANTITY, 100, FIXED, MONTHS)));
    Long rewardId = createRewardWithConstraints(c).getReward().getId();

    List<UserRewardBulkIssueResponse> responses = fireParallelFromDifferentCustomers(rewardId, 5, 1);

    assertEquals(5, countByCode(responses, 200), "All 5 must succeed.");

    List<RewardIssueSummary> rewardRows = rewardIssueSummaryRepository.findAll().stream()
        .filter(s -> s.getLevel() == Level.REWARD && s.getKpi() == KPI.QUANTITY)
        .collect(toList());

    assertEquals(1, rewardRows.size(),
        "Must be exactly 1 REWARD-level row. PRE-FIX: 5 duplicate rows — one per concurrent INSERT.");
    assertEquals(0, BigDecimal.valueOf(5).compareTo(rewardRows.get(0).getConsumed()),
        "Single row must accumulate all 5 deltas. PRE-FIX: row shows consumed=1 only.");
}
```

---

### T5 — Race C isolated: atomic consumed accumulation (no lost UPDATEs)

Seeds one row first (so the UPDATE path, not INSERT path, is exercised), then fires concurrent UPDATEs.

| | |
|---|---|
| **Reward config** | CART_PROMOTION, REWARD-level QUANTITY FIXED MONTHLY, `limit = 100` |
| **Load** | 1 sequential seed call, then 9 concurrent from 9 different customers, `qty = 1` each |
| **Pre-fix DB failure** | Multiple threads read `consumed = 1` (seed value), all compute `1+1=2`, last write wins → `consumed = 2` instead of `10` |

```java
@Test
void shouldAccumulateConsumedAtomically_NoLostUpdates() throws Exception {
    RewardConstraintRequest c = new RewardConstraintRequest();
    c.setRewardLevel(List.of(buildConstraint(KPI.QUANTITY, 100, FIXED, MONTHS)));
    Long rewardId = createRewardWithConstraints(c).getReward().getId();

    // seed: one sequential call so the summary row exists before concurrent calls hit UPDATE
    issueBulkRewards(buildIssueRequest(rewardId, "10000000999", 1));

    List<UserRewardBulkIssueResponse> responses = fireParallelFromDifferentCustomers(rewardId, 9, 1);
    assertEquals(9, countByCode(responses, 200), "All 9 concurrent calls must succeed.");

    // sum across all rows (handles any duplicate rows from Race B for completeness)
    List<RewardIssueSummary> rewardRows = rewardIssueSummaryRepository.findAll().stream()
        .filter(s -> s.getLevel() == Level.REWARD)
        .collect(toList());
    BigDecimal total = rewardRows.stream().map(RewardIssueSummary::getConsumed)
        .reduce(BigDecimal.ZERO, BigDecimal::add);

    assertEquals(0, BigDecimal.valueOf(10).compareTo(total),
        "Total consumed must be 10 (1 seed + 9 concurrent). PRE-FIX: consumed = 2 — blind UPDATE lost 8 increments.");
}
```

---

### T6 — Mixed REWARD + CUSTOMER level constraints

Verifies REWARD-level limit blocks new customers even when their individual CUSTOMER-level counter is still under their own limit.

| | |
|---|---|
| **Reward config** | CART_PROMOTION, REWARD-level QUANTITY `limit = 3`, CUSTOMER-level QUANTITY `limit = 5` |
| **Load** | 5 different customers, `qty = 1` each |
| **Pre-fix** | All 5 succeed at API (REWARD-level limit ignored under concurrency) |

```java
@Test
void shouldEnforceMixedRewardAndCustomerLevelConstraints_DifferentCustomers() throws Exception {
    RewardConstraintRequest c = new RewardConstraintRequest();
    c.setRewardLevel(List.of(buildConstraint(KPI.QUANTITY, 3, FIXED, MONTHS)));
    c.setCustomerLevel(List.of(buildConstraint(KPI.QUANTITY, 5, FIXED, MONTHS)));
    Long rewardId = createRewardWithConstraints(c).getReward().getId();

    List<UserRewardBulkIssueResponse> responses = fireParallelFromDifferentCustomers(rewardId, 5, 1);

    // API: exactly 3 succeed (REWARD-level=3 binds before CUSTOMER-level=5)
    assertEquals(3, countByCode(responses, 200),
        "Exactly 3 must succeed. PRE-FIX: all 5 succeed, REWARD-level limit ignored.");
    assertEquals(2, countByCode(responses, CONSTRAINT_EVALUATION_FAILED),
        "2 must be rejected by REWARD-level constraint. PRE-FIX: 0 rejected.");

    List<RewardIssueSummary> all = rewardIssueSummaryRepository.findAll();

    // REWARD-level: 1 shared row, consumed = 3
    List<RewardIssueSummary> rewardRows = all.stream()
        .filter(s -> s.getLevel() == Level.REWARD).collect(toList());
    assertEquals(1, rewardRows.size());
    assertEquals(0, BigDecimal.valueOf(3).compareTo(rewardRows.get(0).getConsumed()));

    // CUSTOMER-level: exactly 3 per-customer rows (only 3 customers succeeded)
    List<RewardIssueSummary> customerRows = all.stream()
        .filter(s -> s.getLevel() == Level.CUSTOMER).collect(toList());
    assertEquals(3, customerRows.size(),
        "Only 3 customers issued — 3 CUSTOMER-level rows expected. PRE-FIX: 5 rows.");
    customerRows.forEach(r ->
        assertEquals(0, BigDecimal.valueOf(1).compareTo(r.getConsumed())));
}
```

---

### T7 — Regression: same-customer lock must still serialise concurrent calls

Must pass **both pre-fix and post-fix**. Ensures the fix does not accidentally break the existing customer-level lock path.

```java
@Test
void shouldStillSerializeSameCustomerConcurrentCalls_CustomerLockRegression() {
    Long rewardId = createRewardWithConstraints(Fixtures.getRewardCreateConstraintRequest())
        .getReward().getId();

    String sameMobile = "84793759919"; // same customer for all 4 threads
    List<UserRewardBulkIssueResponse> responses = IntStream.range(0, 4)
        .mapToObj(i -> CompletableFuture.supplyAsync(
            () -> issueBulkRewards(buildIssueRequest(rewardId, sameMobile, 1))))
        .map(CompletableFuture::join).collect(toList());

    assertEquals(3, countByCode(responses, PARALLEL_CALL_NOT_ALLOWED),
        "3 of 4 same-customer calls must be rejected by customer lock — both pre- and post-fix.");
    assertEquals(1, countByCode(responses, 200));
}
```

---

### T8 — NO_LIMIT REWARD-level: limit enforced correctly across multiple days of history

Uses `DateTimeServiceStub` to simulate issuances on Day 1 and Day 2, producing two distinct `issueDate` rows in `TBL_REWARD_ISSUE_SUMMARY`. Then on Day 3, fires concurrent requests and asserts the limit is enforced against the correct multi-day aggregate.

**What it proves:** The authoritative `refreshSummary()` inside the Redis lock reads accumulated history from prior-day rows — not just today's row. Pre-fix, the SUM may miss prior-day rows due to duplicate row corruption or Race A on the remaining capacity.

| | |
|---|---|
| **Reward config** | CART_PROMOTION, REWARD-level QUANTITY NO_LIMIT, `limit = 8` |
| **Load** | Day 1: 3 sequential (consumed = 3); Day 2: 3 sequential (consumed = 6); Day 3: 4 concurrent from 4 different customers (only 2 remaining) |
| **Pre-fix API failure** | All 4 Day-3 calls succeed (consumed = 10 > limit 8) |
| **Pre-fix DB failure** | Total across all rows > 8; Race B duplicates cause under-count on re-read |

```java
@Test
void shouldEnforceNoLimitConstraint_AcrossMultipleDaysOfHistory() throws Exception {
    Instant base = LocalDate.now().atStartOfDay(ZoneId.systemDefault()).toInstant();
    RewardConstraintRequest c = new RewardConstraintRequest();
    c.setRewardLevel(List.of(buildConstraint(KPI.QUANTITY, 8, RepeatFrequencyType.NO_LIMIT)));
    Long rewardId = createRewardWithConstraints(c, Date.from(base.plus(365, DAYS))).getReward().getId();

    // Day 1 — 3 sequential issuances; each writes issueDate = Day 1
    DateTimeServiceStub.setClock(Clock.fixed(base, ZoneId.systemDefault()));
    issueBulkRewards(buildIssueRequest(rewardId, "10000000001", 1));
    issueBulkRewards(buildIssueRequest(rewardId, "10000000002", 1));
    issueBulkRewards(buildIssueRequest(rewardId, "10000000003", 1));

    // Day 2 — 3 more sequential issuances; new issueDate row for Day 2
    DateTimeServiceStub.setClock(Clock.fixed(base.plus(1, DAYS), ZoneId.systemDefault()));
    issueBulkRewards(buildIssueRequest(rewardId, "10000000004", 1));
    issueBulkRewards(buildIssueRequest(rewardId, "10000000005", 1));
    issueBulkRewards(buildIssueRequest(rewardId, "10000000006", 1));

    // Day 3 — 4 concurrent attempts; only 2 capacity remaining (limit 8 − consumed 6)
    DateTimeServiceStub.setClock(Clock.fixed(base.plus(2, DAYS), ZoneId.systemDefault()));
    List<UserRewardBulkIssueResponse> responses = fireParallelFromDifferentCustomers(rewardId, 4, 1);

    // API assertions
    assertEquals(2, countByCode(responses, 200),
        "Only 2 remaining capacity. PRE-FIX: all 4 succeed — limit breached.");
    assertEquals(2, countByCode(responses, CONSTRAINT_EVALUATION_FAILED),
        "2 must be rejected. PRE-FIX: 0 rejected.");

    // DB assertions — 3 distinct issueDate rows, aggregate exactly 8
    List<RewardIssueSummary> rows = rewardIssueSummaryRepository.findAll().stream()
        .filter(s -> s.getLevel() == Level.REWARD && s.getKpi() == KPI.QUANTITY)
        .collect(toList());
    assertEquals(3, rows.size(),
        "One row per distinct issueDate. PRE-FIX: Race B may produce duplicates.");
    BigDecimal total = rows.stream().map(RewardIssueSummary::getConsumed)
        .reduce(BigDecimal.ZERO, BigDecimal::add);
    assertEquals(0, BigDecimal.valueOf(8).compareTo(total),
        "Aggregate = limit exactly. PRE-FIX: total = 10 — limit breached.");
}
```

---

### T9 — Race C across day boundary: atomic increment holds when prior-day row exists

Extends T5. Seeds a prior-day row (Day 1), then fires concurrent updates on Day 2. Verifies:
- The prior-day row is not mutated by Day 2's concurrent writes
- Day 2's row accumulates atomically (no lost updates across the day boundary)

**What it proves:** Race C (`SET CONSUMED = :computedValue` blind overwrite) corrupts data when multiple threads concurrently update a row — including when there is already a prior-day row whose SUM feeds the evaluation. Pre-fix, Day 2's row ends up with `consumed = 1` instead of `9` (8 increments silently lost).

| | |
|---|---|
| **Reward config** | CART_PROMOTION, REWARD-level QUANTITY FIXED MONTHLY, `limit = 100` |
| **Load** | Day 1: 1 sequential seed; Day 2: 9 concurrent from 9 different customers |
| **Pre-fix API** | All 9 succeed (limit not a bottleneck) |
| **Pre-fix DB failure** | Day 2 row: `consumed = 1` instead of `9` (8 lost updates); Day 1 row: unaffected |

```java
@Test
void shouldAccumulateAtomically_WhenPriorDaySummaryRowAlreadyExists() throws Exception {
    Instant base = LocalDate.now().atStartOfDay(ZoneId.systemDefault()).toInstant();
    RewardConstraintRequest c = new RewardConstraintRequest();
    c.setRewardLevel(List.of(buildConstraint(KPI.QUANTITY, 100, WindowType.FIXED, RepeatFrequencyType.MONTHS)));
    Long rewardId = createRewardWithConstraints(c, Date.from(base.plus(365, DAYS))).getReward().getId();

    // Day 1 — seed 1 sequential issuance → 1 REWARD-level row with Day 1 issueDate
    DateTimeServiceStub.setClock(Clock.fixed(base, ZoneId.systemDefault()));
    issueBulkRewards(buildIssueRequest(rewardId, "10000000099", 1));

    // Day 2 — 9 concurrent issuances; must accumulate atomically into a NEW Day 2 row
    DateTimeServiceStub.setClock(Clock.fixed(base.plus(1, DAYS), ZoneId.systemDefault()));
    List<UserRewardBulkIssueResponse> responses = fireParallelFromDifferentCustomers(rewardId, 9, 1);
    assertEquals(9, countByCode(responses, 200), "All 9 must succeed (limit = 100).");

    List<RewardIssueSummary> rows = rewardIssueSummaryRepository.findAll().stream()
        .filter(s -> s.getLevel() == Level.REWARD && s.getKpi() == KPI.QUANTITY)
        .collect(toList());

    // Exactly 2 rows: one per day (no Race B duplicates)
    assertEquals(2, rows.size(),
        "Day 1 row + Day 2 row. PRE-FIX: Race B produces up to 9 duplicate rows for Day 2.");

    // Day 1 row must be unchanged
    RewardIssueSummary day1Row = rows.stream()
        .min(Comparator.comparing(RewardIssueSummary::getIssueDate)).orElseThrow();
    assertEquals(0, BigDecimal.valueOf(1).compareTo(day1Row.getConsumed()),
        "Day 1 row must remain 1. PRE-FIX: may be mutated by Day 2's blind UPDATE.");

    // Day 2 row must accumulate all 9 increments atomically
    RewardIssueSummary day2Row = rows.stream()
        .max(Comparator.comparing(RewardIssueSummary::getIssueDate)).orElseThrow();
    assertEquals(0, BigDecimal.valueOf(9).compareTo(day2Row.getConsumed()),
        "Day 2 row must show 9. PRE-FIX: consumed = 1 (8 increments silently lost).");

    // Total must be 10
    BigDecimal total = rows.stream().map(RewardIssueSummary::getConsumed)
        .reduce(BigDecimal.ZERO, BigDecimal::add);
    assertEquals(0, BigDecimal.valueOf(10).compareTo(total));
}
```

---

### Clock discipline for T8 and T9

Both tests manipulate `DateTimeServiceStub.clock`, which is a **static field shared across all threads**. Two non-negotiable guards:

```java
@AfterEach
void resetClock() {
    DateTimeServiceStub.restoreClock();   // mandatory — a stuck clock corrupts subsequent tests
}
```

**Reward `endTime` must be set well beyond the simulated dates.** A reward created with default `endTime = now + 7 days` will appear expired when the clock is advanced to Day 8+, and all issuance calls will be rejected at validation before reaching constraint checks. Set `endTime = base.plus(365, DAYS)` for all multi-day tests.

---

### Coverage matrix

| Test | Race | KPI | Window | Threads | Clock travel | Pre-fix API fails | Pre-fix DB fails |
|------|------|-----|--------|---------|--------------|-------------------|------------------|
| T1 | A + B + C | QUANTITY | FIXED MONTHLY | 6 diff | No | ✗ success count wrong | ✗ duplicate rows + wrong consumed |
| T2 | A | QUANTITY | ROLLING 7d | 6 diff | No | ✗ success count wrong | ✗ consumed > limit |
| T3 | A | POINTS | FIXED WEEKLY | 4 diff | No | ✗ success count wrong | ✗ consumed > 100 |
| T4 | B only | QUANTITY | FIXED MONTHLY | 5 diff | No | ✓ all succeed (hidden bug) | ✗ 5 duplicate rows |
| T5 | C only | QUANTITY | FIXED MONTHLY | 9+1 diff | No | ✓ all succeed (hidden bug) | ✗ consumed = 2 not 10 |
| T6 | A + B | QUANTITY | FIXED MONTHLY | 5 diff, mixed level | No | ✗ success count wrong | ✗ extra CUSTOMER rows |
| T7 | Control | QUANTITY | FIXED MONTHLY | 4 same | No | ✓ must pass always | ✓ must pass always |
| T8 | A + B + C | QUANTITY | NO_LIMIT | 4 diff + 2 seeded days | Yes — 3 days | ✗ success count wrong | ✗ aggregate > limit |
| T9 | C cross-day | QUANTITY | FIXED MONTHLY | 9+1 diff + 1 seeded day | Yes — 2 days | ✓ all succeed (hidden bug) | ✗ Day 2 row = 1 not 9 |

T4, T5, T8, and T9 are the hidden-corruption tests — API returns 200 but DB is silently wrong. T8 and T9 add the multi-day dimension that most closely mirrors production: rewards that have been active for days or months before the race window is hit.

**Clock discipline column** marks tests that call `DateTimeServiceStub.setClock()` and therefore require `@AfterEach restoreClock()` and a far-future reward `endTime`.

---

## Technical Clarifications for Implementer

The following questions were raised during design review. Answers are recorded here so the implementer does not re-derive them during tech detail.

---

### CQ1 — Why is the lock key `reward_constraint:{orgId}:{rewardId}` and not finer (e.g., per-KPI)?

A single thread processes all REWARD-level KPI constraints in one loop inside `validateForNonOrgLevelConstraints()`. If you used finer keys (`reward_constraint:{orgId}:{rewardId}:{kpi}`), a thread would need to hold all KPI-level keys simultaneously — giving the same effective serialisation as one coarser key with no throughput improvement. One key per `rewardId` is the minimum granularity that eliminates Race A across all KPIs in a single lock acquisition.

---

### CQ2 — Is it fine to acquire the lock in one processor and release it in another?

No. The lock must be **acquired and released entirely within `NonOrgSummaryWriteProcessor.process()`** using a try-finally block. The call chain diagram shows the lock's logical position relative to pos 12 and pos 13 — not code ownership split across them. Splitting lock acquisition across processor boundaries means: if pos 12 exits abnormally, the finally block in pos 12 never runs and the lock is never released. The processor framework has no lock-cleanup hook.

```java
// NonOrgSummaryWriteProcessor.process()
String lockKey = "reward_constraint:" + orgId + ":" + rewardId;
boolean acquired = redisLockService.acquireLock(lockKey);
try {
    refreshSummaryForRewardLevelConstraints();  // authoritative read, inside lock
    evaluate();
    if (fails) throw CONSTRAINT_EVALUATION_FAILED;
    writeNonOrgSummaries();                    // atomic CONSUMED + delta
} finally {
    if (acquired) redisLockService.releaseLock(lockKey);
}
```

---

### CQ3 — Why does `refreshSummary()` appear to be called twice (pos 12 and pos 13)?

They serve **different purposes and must not be merged**:

| Call site | Lock held | Purpose | Authoritative? |
|---|---|---|---|
| Pos 12 `RewardConstraintProcessor` | No | Speculative early exit — reject clearly-over-limit requests without ever acquiring the lock | No — result may be stale by the time pos 13 runs |
| Pos 13 `NonOrgSummaryWriteProcessor` | Yes | Authoritative check — the real enforcement that drives the write decision | Yes — no concurrent write can interleave |

Without the pos 12 call, every request at 2K RPM queues on the lock before discovering an exhausted reward. Without the pos 13 call, the lock protects nothing — Race A remains because the evaluation was done before acquiring the lock.

---

### CQ4 — Should pos 13 call `refreshSummary` for CUSTOMER-level constraints too?

**No.** `@CustomerLockable` on `issueRewardBulk()` is held throughout the entire chain — from before pos 12 through past pos 13. For the same customer, no two threads can interleave between pos 12 and pos 13. This means **pos 12's customer-level SUM is already authoritative** for that customer by the time pos 13 runs.

Calling `refreshSummary` again inside pos 13 for CUSTOMER-level constraints is a dead read — it always returns the same value pos 12 already fetched, at the cost of one extra DB round trip per customer-level constraint per request.

**Implementation rule:** pos 13 calls `refreshSummary` only for REWARD-level constraints (the shared rows that have no per-customer lock). CUSTOMER-level constraint contexts from pos 12 are passed forward via `rewardIssueSummaryContext` without re-reading.

At 2K RPM with 1 REWARD-level + 1 CUSTOMER-level constraint:
- Incorrect (refresh all in pos 13): 4 SUM queries/request = 132/sec
- Correct (refresh REWARD-level only in pos 13): 3 SUM queries/request = 99/sec

---

### CQ5 — Will `refreshSummary` degrade for long-running rewards with a `NO_LIMIT` constraint?

The `NO_LIMIT` SUM query ([`RewardIssueSummaryRepository.java:59`](src/main/java/com/capillary/solutions/rewards/repository/RewardIssueSummaryRepository.java)) has **no date filter**:

```sql
SELECT SUM(consumed) FROM TBL_REWARD_ISSUE_SUMMARY
WHERE orgId = ? AND rewardId = ? AND userId IS NULL AND kpi = ? AND level = ?
-- no ISSUE_DATE bound — scans all rows ever written for this reward
```

This query runs inside the Redis lock at pos 13. Lock hold time is proportional to its latency.

**Actual row count for REWARD-level NO_LIMIT:**

`LevelService.loadKPI()` sets `issueDate = getCurrentDateWithoutTimestamp()` — one row per day of issuance. For a REWARD-level NO_LIMIT constraint, row count = number of active days since reward creation:

```
1-year reward with daily issuances  → ~365 rows
3-year reward                       → ~1095 rows  (+ Race B duplicates: maybe 2×)
5-year reward                       → ~1825 rows
```

With the covering index `(ORG_ID, REWARD_ID, USER_ID, CONSTRAINT_LEVEL, KPI, ISSUE_DATE)`, 365 rows fits in ~36KB of index leaf pages — held warm in MySQL buffer pool for active rewards. **Warm read latency: ~1–3ms.**

At 33 req/sec with 3ms lock hold: lock utilisation = **10%** — not a problem today.

The concern becomes real only after several years with accumulated Race B duplicates under sustained high RPM:

```
5-year reward × 2 duplicates = 3650 rows → ~8ms warm
33 req/sec × 8ms = 264ms/sec = 26% lock utilisation → manageable but worth watching
```

**The "time bomb" fuse is years long, not months.** It does not manifest in load testing on new rewards and is unlikely to surface within the first year of running the Phase 1a fix.

**Required monitoring (non-negotiable):** Emit `refreshSummary` latency as a New Relic custom attribute on the `NEWRELIC_CUSTOM_REWARDS_EVENT` event for REWARD-level NO_LIMIT constraints. Alert if P99 exceeds 10ms. This is the signal that triggers Phase 1b — do not ship Phase 1a without this instrumentation.

---

**Phase 1b — hist + today split (DEFERRED — data-gated, not time-gated):**

A well-designed optimisation exists if monitoring proves the problem. The principle: consumed through yesterday is stable for the full day; only today's delta is live.

```
total_consumed = consumed_through_yesterday  +  consumed_today
                        ↑                              ↑
               fixed for the day               changes per issuance
               → cache in Redis (MGET)         → today-only DB query, same shape
                                                 for DAILY / MONTHLY / NO_LIMIT
```

**Why Phase 1b is deferred and not shipped with Phase 1a:**

1. **Benefit is speculative at current reward ages.** At 1–3ms per NO_LIMIT SUM with a 10% lock utilisation, there is no production bottleneck to solve today.

2. **Phase 1b introduces a new silent-corruption risk.** The hist cache TTL must expire precisely at the org-timezone window boundary — not wall-clock 24h. `RedisCacheUtil` provides fixed TTL buckets (`ONE_DAY_CACHE = Duration.ofDays(1)`) that are not timezone-aligned. A TTL that expires one second too early means pos 12 fetches "hist for today" as empty, undercounting consumed — the same silent corruption Pattern 1a is fixing, reintroduced through the cache layer.

3. **Implementation surface is material.** New Redis key namespace, custom per-window-type TTL calculation in org timezone, SET NX cache miss handling, bulk GROUP BY repository method, `rewardIssueSummaryContext` interface change to carry `histConsumed` and `todayConsumed` separately — touching all `LevelService` implementations.

**Gate to unlock Phase 1b:** New Relic P99 for `refreshSummary` inside lock exceeds 10ms for at least one reward in production. That is the first confirmed data point proving the problem is real. Phase 1b becomes justified at that moment, not before.

**Design reference (when Phase 1b is unlocked):**

```
[Pos 12 — outside lock]
  1. MGET hist for all REWARD-level KPIs        → 1 Redis round trip
  2. Bulk today-only DB query (all REWARD KPIs) → 1 DB query (GROUP BY kpi)
  3. total = hist + today; evaluate; store hist in context

[Pos 13 — inside lock]
  4. Reuse hist from context                    → 0 Redis calls (hist unchanged since pos 12)
  5. Bulk today-only DB query (inside lock)     → 1 DB query (authoritative)
  6. total = hist_from_context + today_pos13; evaluate → write

Lock hold time: flat ~2–5ms regardless of reward age.
DB calls: 2 bulk queries/request regardless of N constraints (vs 2N individual queries today).
```

---

### CQ6 — Why was `SELECT FOR UPDATE` evaluated and rejected?

`SELECT FOR UPDATE` was considered as an alternative to the Redis lock for the UPDATE path (existing row in `TBL_REWARD_ISSUE_SUMMARY`). It was rejected for three reasons:

1. **INSERT path is unprotected.** `SELECT FOR UPDATE` only locks existing rows. When no row exists (first issuance), it returns 0 rows, acquires no lock, and Race B is completely unaffected. InnoDB gap locks can cause DEADLOCKs in this scenario. The Redis lock covers both INSERT and UPDATE paths uniformly.

2. **Requires a new transaction boundary in the processor chain.** `SELECT FOR UPDATE` is useless without an active transaction spanning both the SELECT and the subsequent UPDATE — without one, MySQL auto-commits after the SELECT and releases the row lock before the UPDATE runs. There is **no outer `@Transactional`** on `issueRewardBulk()` or `BulkIssueService` — only fine-grained method-level `@Transactional` on individual JDBC repository methods. Adding `@Transactional` to `NonOrgSummaryWriteProcessor.process()` would be a new transaction boundary introduction into the processor chain.

3. **Dual-datasource coordination.** The SELECT would go through the JPA `RewardIssueSummaryRepository` (EntityManager) while the UPDATE goes through `RewardIssueSummaryJdbcRepository` (JDBC, `DataSourceUtils.getConnection(dataSource)`). For both to participate in the same transaction and for `SELECT FOR UPDATE` to hold the lock across both, JPA and JDBC must bind to the same `DataSource` bean. In a dual-datasource setup (read replica + write master, implied by `getDataSource(false)` in `BaseJdbcRepository`), this coordination is non-trivial and easy to get wrong silently.

The Redis lock avoids all three concerns: it requires no transaction boundary change, works for both INSERT and UPDATE paths, and is independent of the datasource routing configuration.

---

### CQ7 — Does the Redis lock interact with `@Transactional` in the issue flow?

No interaction. The Redis lock is **external to Spring's transaction management** — it is a distributed mutex over the network, not a JDBC connection-level lock. It does not open, join, or suspend any database transaction.

The existing `@Transactional` on `RewardIssueSummaryJdbcRepository.bulkSaveOrUpdate()` (method-level) remains unchanged and is sufficient for atomicity of the INSERT/UPDATE operation. The Redis lock wraps around that method call — the sequence is:

```
acquire Redis lock
  → call bulkSaveOrUpdate()  [opens own @Transactional, writes, commits, closes]
release Redis lock
```

No `REQUIRES_NEW`, no transaction propagation change, no new `@Transactional` annotation needed anywhere in the existing code.

---

### CQ8 — Is the compensation decrement mandatory or optional?

**Mandatory.** The summary write inside pos 13 commits via `bulkSaveOrUpdate()`'s own `@Transactional` — the transaction closes before pos 13 returns. There is no outer transaction wrapping the processor chain. If any processor after pos 13 fails (`CouponIssueProcessor`, `VendorIssueProcessor`, `PointsRedeemProcessor`), Spring has nothing to roll back — the CONSUMED increment is already durably committed.

The compensation must be:
```sql
UPDATE TBL_REWARD_ISSUE_SUMMARY SET CONSUMED = CONSUMED - :delta WHERE ID = :id
```

Row IDs must be stored in `BulkRewardIssueContext.writtenSummaryRows` immediately after the write. The failure handler in `UserRewardUtils` iterates this list and issues the decrement for each row. For bulk requests, compensate only the rows belonging to the failed reward — not the entire batch.

---

### CQ9 — What happens to `updateSummaries()` after pos 13 has already written the summary?

This is the most critical coordination point in the implementation. Without an explicit fix here, pos 13's correct atomic write is silently overwritten by `updateSummaries()` running after the external calls — reintroducing Race C through the back door.

**The problem if `updateSummaries()` is left unchanged:**

```
pos 13:  REWARD-level row written → CONSUMED = CONSUMED + delta (atomic, inside lock)
         lock released

pos 14+: CouponIssueProcessor, VendorIssueProcessor, PointsRedeemProcessor

updateSummaries() [unchanged]:
  → getExistingRewardIssueSummariesForNonOrgLevel()  ← finds pos 13's row (id != null)
  → buildUniqueSummaries()  ← creates UPDATE object with same row id
  → bulkUpdate()  ← either blind SET CONSUMED = :computedValue (old bug)
                     or CONSUMED + delta again (double-counts the delta)
```

Either way is wrong. The blind overwrite resurrects Race C. The double increment double-counts.

**The fix — `updateSummaries()` must skip rows already written by pos 13:**

`BulkRewardIssueContext.writtenSummaryRows` carries the row IDs written inside the lock. `RewardConstraintFacade.updateSummaries()` (or the caller in `UserRewardUtils`) must exclude these rows from the non-org-level write path:

```java
// In RewardConstraintFacade.updateSummaries() or its caller
Set<Long> alreadyWrittenIds = bulkRewardIssueContext.getWrittenSummaryRows().stream()
    .map(RewardIssueSummary::getId).collect(toSet());

// Filter: only write rows NOT already handled by pos 13
List<RewardIssueSummary> pendingWrites = allSummaries.stream()
    .filter(s -> !alreadyWrittenIds.contains(s.getId()))
    .collect(toList());
```

**Phase 1 split of write responsibility:**

| Constraint level | Writer | When |
|---|---|---|
| REWARD-level (FIXED / NO_LIMIT) | pos 13 `NonOrgSummaryWriteProcessor` | Inside lock, before external calls |
| CUSTOMER-level (FIXED / NO_LIMIT) | pos 13 `NonOrgSummaryWriteProcessor` | Inside lock (customer lock already held; no cross-customer conflict) |
| ORG-level | `updateSummaries()` (unchanged) | After external calls, as today |
| ROLLING window (any level) | `updateSummaries()` (unchanged) | After external calls, as today — Race A and C remain for ROLLING in Phase 1 |

**Explicit out-of-scope for Phase 1:** ROLLING window constraints are not protected by pos 13's lock. Race A (cross-customer breach) and Race C (blind overwrite) remain for ROLLING constraints. The test-plan architect must NOT write ROLLING post-fix assertions expecting them to pass — those belong to Phase 2.

---

## Handoff Note for tech-detailer

Root cause is confirmed across three races:
- Race A: [CustomerLockManager.java:62](src/main/java/com/capillary/solutions/rewards/lock/CustomerLockManager.java) — wrong lock granularity for reward/org-level constraints
- Race B: [RewardConstraintFacade.java:342–372](src/main/java/com/capillary/solutions/rewards/service/impl/RewardConstraintFacade.java) — concurrent INSERT on first issuance
- Race C: [RewardIssueSummaryJdbcRepository.java:96–103](src/main/java/com/capillary/solutions/rewards/jdbc/RewardIssueSummaryJdbcRepository.java) — blind UPDATE loses concurrent increments

**Chosen approach: Approach A (Redis Distributed Lock + Atomic DB Increment).** Phase 1 covers `REWARD` and `CUSTOMER` level constraints only. Org-level deferred to Phase 2.

The fix is bounded to:
1. New `NonOrgSummaryWriteProcessor` at processor position 13 — acquires `reward_constraint:{orgId}:{rewardId}` lock (conditional on `REWARD`-level constraint presence), runs refresh + evaluate + write, releases lock. Follow existing processor pattern from `RewardConstraintProcessor`.
2. Modified `RewardConstraintFacade.buildUniqueSummaries()` — remove `HashSet` reference-identity dedup; rebuild working set with explicit field-equality check. Also fix the result set to be built from scratch, not by mutating the fetched list.
3. Modified `RewardIssueSummaryJdbcRepository.bulkUpdate()` — change `SET CONSUMED = :computedValue` to `SET CONSUMED = CONSUMED + :delta`. Delta is the KPI value of this request only; not recomputed from a prior fetch.
4. Modified `BulkRewardIssueContext` — add `writtenSummaryRows` to carry row IDs through to compensation.
5. Modified `UserRewardUtils.issueReward()` / `issueRewardBulk()` — add compensation (`CONSUMED - :delta`) in the failure path for any processor failure after position 13.
6. No DBA migration needed — `idx_rewardIssue_summary_user_id (ORG_ID, REWARD_ID, USER_ID, CONSTRAINT_LEVEL, KPI, ISSUE_DATE)` already covers all SUM queries in `RewardIssueSummaryRepository`. Confirmed from `TBL_REWARD_ISSUE_SUMMARY.sql` DDL. Exception: `NO_LIMIT` constraints scan this index without a date bound — see CQ5 for the long-term degradation concern.

Key risks to explore during tech detail:
- Compensation path must cover partial successes in bulk requests (some rewards succeed, some fail — only the failed reward's rows are compensated)
- `NonOrgSummaryWriteProcessor` must be idempotent: if the processor itself throws after the write but before returning, the failure path must still trigger compensation
- `lastUpdatedOn` column — verify the JDBC `bulkUpdate()` SQL touches `LAST_UPDATED_ON` so Databricks delta ETL picks up the increment
- Redis lock TTL must be tuned to cover GC pause headroom (nominal hold ~10–50ms, configure TTL ≥ 500ms)

Contracts to validate:
- `redisLockService.acquireLock("reward_constraint:" + orgId + ":" + rewardId)` must not collide with existing lock key prefixes in the Sentinel cluster
- `CONSUMED + :delta` UPDATE — confirm `delta` type is `BigDecimal` consistent with `CONSUMED` column precision (13,4)
- `acquireBulkLock()` (Phase 2 path) — already exists in `RedisLockService`; validate deadlock-safe key sort behaviour in unit test before Phase 2

Suggested test coverage emphasis:
- Unit: `NonOrgSummaryWriteProcessor` — lock acquired, evaluate passes, write succeeds; lock acquired, evaluate fails, lock released; lock not acquired when no REWARD-level constraint
- Unit: `RewardConstraintFacade.buildUniqueSummaries()` — no pre-existing duplicates in working set; correct date selection per constraint type
- Integration (concurrency): see **`## Integration Test Plan`** section above — 7 tests in `RewardConstraintConcurrencyIntegrationTest` cover all three races across QUANTITY/POINTS KPIs and FIXED/ROLLING windows. T1–T3 fail pre-fix at the API level. T4–T5 fail only at the DB level (API returns 200) — these are the critical hidden-corruption cases.
- Integration (compensation): simulate `CouponIssueProcessor` failure after pos 13; assert `CONSUMED` is decremented back to pre-request value
- Integration (`lastUpdatedOn`): assert Databricks-visible column updated after atomic increment
