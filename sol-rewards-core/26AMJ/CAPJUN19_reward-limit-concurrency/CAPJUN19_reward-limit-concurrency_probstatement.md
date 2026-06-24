# Problem Statement: CAPJUN19 — reward-limit-concurrency

## Symptom

Two distinct data correctness failures observed under parallel `issueReward` calls for the same reward:

1. **Reward-level limit breach** — the configured constraint limit (e.g., total issuances = 100) is exceeded. Multiple concurrent calls each read the same consumed value, independently determine the limit is not reached, and all proceed to issue.
2. **Duplicate rows in `TBL_REWARD_ISSUE_SUMMARY`** — multiple rows with identical `(ORG_ID, REWARD_ID, KPI, CONSTRAINT_LEVEL, CONSTRAINT_ATTRIBUTE, USER_ID, ISSUE_DATE)` appear in the summary table. Consumed values across the duplicate rows undercount the true total, masking the breach in subsequent reads.

Both symptoms occur together under concurrent load. They are not independent bugs — they share the same root cause.

---

## Scope

- **Service:** `sol-rewards-core`
- **Affected constraint levels:** `REWARD` (global reward budget) and `ORG` level constraints. `CUSTOMER`-level constraints are partially protected by the existing per-customer Redis lock but remain vulnerable to the INSERT-race described below.
- **Affected window types:** `FIXED` window constraints confirmed. `ROLLING` window constraints likely affected by the same races but not yet validated.
- **Affected KPI types:** `QUANTITY`, `POINTS`, `TRANSACTION_COUNT`, `REDEMPTION_VALUE` — all go through the same constraint check and summary update path.
- **Tenants:** Any org using reward-level or org-level issuance limits with concurrent traffic.
- **Not affected:** Single-threaded or low-QPS orgs where concurrent calls are rare.

---

## Entry Point

**API:** `POST /core/v1/reward/issue` (and the bulk variant)

**Call chain:**
```
UserRewardUtils.issueReward()                       ← @CustomerLockable (lock key = orgId + "_" + customerId)
  └── BulkIssueService.issueBulk()
        └── processor chain (fixed order, PostConstruct)
              ├── [pos 12] RewardConstraintProcessor.process()
              │     ├── validateForOrgLevelConstraints()
              │     └── validateForNonOrgLevelConstraints()
              │           └── LevelService.evaluate()
              │                 └── LevelService.getConsumedValueFromDB()  ← live MySQL read, no row lock
              ├── [pos 14] CouponIssueProcessor
              ├── [pos 16] VendorIssueProcessor
              └── [pos 17] PointsRedeemProcessor
  └── updateSummaries()                             ← AFTER all external calls
        └── RewardConstraintFacade.updateSummaries()
              └── RewardIssueSummaryJdbcRepository.bulkSaveOrUpdate()
```

---

## Expected vs Actual Behaviour

### Expected

- For a reward with limit = 100 and current consumed = 98, exactly 1 request with delta = 2 should succeed and 1 request with delta = 2 should be rejected if two arrive simultaneously.
- `TBL_REWARD_ISSUE_SUMMARY` should have exactly one row per `(orgId, rewardId, kpi, level, attribute, userId, windowDate)` combination, reflecting the true cumulative consumed value.

### Actual

**Race A — wrong lock granularity (reward/org-level breach):**  
The `@CustomerLockable` lock key is `orgId + "_" + customerId`. Two requests from _different_ customers concurrently issuing the same reward are NOT serialized. Both enter `RewardConstraintProcessor` simultaneously, both read `consumed = 98`, both compute `98 + 2 ≤ 100 → pass`, both issue. True consumed = 102 against limit 100.

**Race B — concurrent INSERT duplicate rows:**  
Both threads reach `updateSummaries()` with no existing DB row. `buildUniqueSummaries()` finds no row → creates a new object with `id = null` → both threads execute `INSERT`. No DB unique constraint exists, so both INSERTs succeed. Two rows now exist with identical keys but each showing `consumed = 2` (not `consumed = 4`). Subsequent reads aggregate incorrectly.

**Race C — lost UPDATE (stale computed value):**  
If a row exists, both threads read `consumed = 98`, each computes `new_consumed = 98 + 2 = 100`, both `UPDATE SET CONSUMED = 100`. True consumed should be 102. The UPDATE uses a blind `SET CONSUMED = :computedValue`, not an atomic `consumed = consumed + delta`. Second write silently overwrites the first.

---

## Root Causes

| # | Root Cause | Location |
|---|---|---|
| A | Lock granularity is per-customer; reward/org-level constraints are shared across customers and unprotected | `CustomerLockManager.java:62` — lock key = `orgId + "_" + customerId` |
| B | Concurrent INSERT on summary table with no DB unique constraint and no SELECT FOR UPDATE | `RewardConstraintFacade.java:228` (`buildUniqueSummaries`), `RewardIssueSummaryJdbcRepository.java:153` (`bulkSaveOrUpdate`) |
| C | Summary UPDATE is a blind overwrite of a computed value, not an atomic increment | `RewardIssueSummaryJdbcRepository.java` — `UPDATE … SET CONSUMED = :consumed` |

---

## Chosen Solution Direction

**3-Tier Counter Architecture** — move the live enforcement authority out of MySQL into Redis, with MongoDB as a durable intra-day journal.

```
Redis (volatile, no AOF)   → atomic Lua check-and-reserve  (hot enforcement path)
MongoDB                    → durable intra-day WAL journal  (re-seed source on Redis restart)
MySQL TBL_REWARD_ISSUE_SUMMARY → end-of-day settled value  (Databricks ETL feed, unchanged)
```

Key design decisions:
- Redis Lua script checks ALL constraints for a request atomically in one call (multi-key, safe on Sentinel single-master). Either all constraints pass and all are incremented, or none are incremented. Eliminates Race A and the partial-rollback problem.
- MongoDB journal entry written SYNCHRONOUSLY after Redis Lua, BEFORE external issuance. Status = `PENDING`. Updated to `COMMITTED` after successful issuance. If MongoDB write fails, Redis is DECR'd and request fails. Ensures Redis ≤ MongoDB always for re-seed safety.
- End-of-day flush job (runs at 01:00 AM, 1-hour buffer past midnight) reads MongoDB COMMITTED entries grouped by `(constraintId, windowDate)` and writes aggregated totals to `TBL_REWARD_ISSUE_SUMMARY` for Databricks.
- `windowAnchorDate` is pinned at constraint check time and stored in `RewardIssueSummaryContext`. All downstream writes (MongoDB, MySQL) use this stored date, never `now()`. Prevents day-boundary window assignment drift.
- ROLLING window constraints excluded from Phase 1 (need different window-key strategy).

---

## Constraints

- Cannot add DB unique constraint to `TBL_REWARD_ISSUE_SUMMARY` — existing duplicate rows block `CREATE UNIQUE INDEX`.
- Redis Sentinel has no AOF persistence — Redis data is lost on restart. MongoDB journal is the sole re-seed source.
- `TBL_REWARD_ISSUE_SUMMARY` must be maintained for Databricks daily delta ETL (`lastUpdatedOn` column).
- Redis Sentinel is a shared cluster (sol-rewards-core, PromotionEngine, Badges) on DB index 1 — no namespace collision risk if key prefix is `reward_constraint_counter:`.
- MongoDB is on the shared `emf` cluster, `sol-rewards-core` database — new collection must be registered in `MongoDbInitializer`.
- Any new env vars (flush cron, Redis key TTL) go in `sol-rewards-core-a.json` and propagate to all 8+ prod clusters.

---

## Evidence

- Observed duplicate rows with identical constraint keys in `TBL_REWARD_ISSUE_SUMMARY` in production.
- Reward issued count exceeding configured `limitValue` in production monitoring.
- Code-level: `CustomerLockManager.java:62` shows lock key includes only `customerId`, not `rewardId`.
- Code-level: `RewardIssueSummaryJdbcRepository.java` `bulkSave()` is plain `INSERT` — no `ON DUPLICATE KEY UPDATE`.
- Code-level: `bulkUpdate()` UPDATE SQL uses `SET CONSUMED = :consumed` (computed value), not `SET CONSUMED = CONSUMED + :delta`.
- Code-level: `LevelService.getConsumedValueFromDB()` is a plain JPA read with no `SELECT FOR UPDATE`.
