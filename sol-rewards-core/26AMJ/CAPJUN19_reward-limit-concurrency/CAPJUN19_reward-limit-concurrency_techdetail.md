# Tech Detail: Reward Constraint Limit Breach & Duplicate Summary Rows Under Parallel Issue Calls

**Date:** 2026-06-24  
**Author:** Saravanan Kesavan  
**Status:** Implementation Complete — Pending Staging Validation  
**Investigation Doc:** `CAPJUN19_reward-limit-concurrency_analysis.md` (same directory)  
**Confidence:** HIGH — all three races fixed and verified; both post-implementation gaps (Gap 1: lock timeout, Gap 2: multi-FIXED dedup) confirmed resolved. Integration tests T1–T8 all pass.  
**Branch:** `claude/CAPJUN19_reward-limit-concurrency`  
**Commit:** `6404131d9 — CAPJUN19 | Fix reward-limit concurrency races with distributed lock + atomic SQL`

---

## 1. Problem Statement

Under parallel `issueReward` API calls for the same reward from different customers, two distinct failures surface in production:

1. **Limit breach** — the constraint's `limitValue` is exceeded even when each individual request passes the constraint evaluation check.
2. **Duplicate rows** — `TBL_REWARD_ISSUE_SUMMARY` accumulates multiple rows for the same logical key `(orgId, rewardId, constraintLevel, kpi, userId, issueDate)`, causing the consumed value to be under-counted on subsequent reads, compounding the breach over time.

Scope: affects all constraint levels (CUSTOMER, REWARD) but is most severe for REWARD-level constraints — which have no per-constraint serialization at all. ORG-level constraints are explicitly deferred to Phase 2. Existing duplicate rows in `TBL_REWARD_ISSUE_SUMMARY` cannot be removed by adding a DB unique index; the Redis lock prevents new duplicates without touching existing data.

---

## 2. Root Cause (Confirmed)

**The constraint check-then-act sequence is not atomic, and the customer-level Redis lock serialises only per-customer calls, leaving REWARD-level constraints fully exposed to concurrent violations from parallel requests by different customers. Within the lock, the UPDATE path was a read-compute-write (not atomic), and duplicate INSERTs could occur when the row did not yet exist.**

### Contributing Factors

1. **Race A — Wrong lock granularity.** `@CustomerLockable` lock key = `orgId + "_" + customerId` ([CustomerLockManager.java:62](src/main/java/com/capillary/solutions/rewards/lock/CustomerLockManager.java)). For REWARD-level constraints (shared across all customers), two different customers' threads hold different locks and run the check concurrently.
2. **Race B — Concurrent INSERT on first issuance.** `RewardConstraintFacade.updateSummaries()` re-reads the DB, runs `buildUniqueSummaries()`, and calls `bulkSaveOrUpdate()`. When no row exists yet, two concurrent threads both read `[]` and both INSERT — producing duplicate rows.
3. **Race C — Lost UPDATE on concurrent increments.** `bulkUpdate()` computed the new value in application memory then issued a blind `UPDATE SET CONSUMED = :computedValue`. Two threads reading `consumed = 5` both computed `6`, both wrote `6` — one increment silently lost.
4. **CQ9 — `updateSummaries()` double-write.** Even after the pos-13 atomic write, the post-issuance `updateSummaries()` path would re-fetch the row and overwrite it with stale accumulated values from pos-12's read, reintroducing Race C through the back door.

### Why Existing Customer Lock Did Not Catch It

`@CustomerLockable` on `UserRewardUtils.issueRewardBulk()` serialises only same-customer concurrent calls. Different customers for the same reward each hold a different lock key and run concurrently with no shared serialization for the REWARD-level shared budget row.

---

## 3. Scope of Change

### In Scope (Phase 1)

- REWARD-level constraint enforcement under concurrent cross-customer calls
- CUSTOMER-level constraint atomic writes (shares the same new write path; no lock needed since `@CustomerLockable` already serialises same-customer calls)
- Compensation decrement on downstream failure (coupon issue, vendor, points redeem fail after summary write)
- CQ9 coordination: `updateSummaries()` skip for rows already written atomically

### Out of Scope (Explicit)

- ORG-level constraints — deferred to Phase 2; they use a different lock strategy (`org_constraint:{orgId}:{constraintId}` with `acquireBulkLock()`)
- DB schema changes — no new columns or indexes
- Existing duplicate rows in `TBL_REWARD_ISSUE_SUMMARY` — not cleaned up; existing data is left as-is
- Databricks ETL pipeline — no change required (`LAST_UPDATED_ON` auto-updated by MySQL `ON UPDATE CURRENT_TIMESTAMP`)

### Deferred (Future Consideration)

- Phase 1b: hist + today split optimization for NO_LIMIT constraints (detailed in CQ5 of investigation doc)
- Phase 2: ORG-level constraint protection (ROLLING window REWARD/CUSTOMER-level is already covered by Phase 1 — the lock is per `orgId:rewardId`, not per WindowType)

---

## 4. Assumptions

| # | Assumption | Breaks if... | Validation |
|---|-----------|-------------|-----------|
| 1 | `RedisLockService.acquireLock(key)` uses `redisLockRegistry` (customer lock registry, TTL = `redis.lock.ttl`) | A different registry is used — different TTL, different key namespace | **Confirmed**: `acquireLock(String key)` uses `redisLockRegistry` ([RedisLockService.java:50](src/main/java/com/capillary/solutions/rewards/lock/RedisLockService.java)) |
| 2 | `BulkRewardIssueContext` is a shared mutable object passed by reference through the entire processor chain | Context is cloned or snapshotted between processors | **Confirmed**: loop at BulkIssueService.java:126 reassigns reference; same object flows |
| 3 | Processor chain order is defined in `buildBulkIssueProcessors()` and is not validated at startup | Chain order constraint added that rejects insertion of `nonOrgSummaryWriteProcessor` at pos 13 | **Confirmed**: order is a plain `ArrayList` at BulkIssueService.java:98–121 |
| 4 | `LAST_UPDATED_ON` is auto-managed by MySQL `ON UPDATE CURRENT_TIMESTAMP` | MySQL trigger or auto-update is removed — Databricks ETL stops picking up atomic increments | **Confirmed**: DDL at `TBL_REWARD_ISSUE_SUMMARY.sql:13` — `ON UPDATE CURRENT_TIMESTAMP` |
| 5 | Compensation decrement path covers all partial-failure scenarios | A failure path exits without calling the `finally` block in `UserRewardUtils` | **Confirmed**: `try { issueBulk() } finally { compensateFailedSummaryWrites() }` at UserRewardUtils.java:60–63 |
| 6 | ROLLING window REWARD/CUSTOMER-level constraints are covered by Phase 1 | `NonOrgSummaryWriteProcessor` filters by Level, not WindowType; `writeNonOrgSummariesAtomically()` resolves `issualDate` for ROLLING — correct row key. Phase 2 urgency is driven by ORG-level coverage only. | **Confirmed** — no open action |

---

## 5. Design Validation

### 5a. Gaps Found vs Handoff Doc

**GAP [SEVERITY: HIGH] — T3/T4 integration tests failing (lock acquire timeout)**  
Category: Configuration  
What the tests reveal: T3 (2 concurrent customers, limit=5, both must succeed) and T4 (5 concurrent customers, limit=10, all must succeed) fail: only Thread A succeeds; threads B–E fail with `CONSTRAINT_EVALUATION_FAILED`.  

Root cause (confirmed via investigation): `redis.lock.maxWaitTime=${REDIS_LOCK_MAX_WAIT_TIME:10}` defaults to **10 ms** in both test and production properties. The reward-level Redis lock serialises ALL customers of the same reward through a single key (`reward_constraint:{orgId}:{rewardId}`). Thread A holds the lock for the duration of its DB operations (SELECT + INSERT/UPDATE on MySQLContainer ≈ 50–200 ms in tests). Threads B–E call `tryLock(10ms)` and time out before Thread A finishes. The catch block in `NonOrgSummaryWriteProcessor.processWrapper()` adds the timed-out wrapper to `toFail` with `CONSTRAINT_EVALUATION_FAILED` — identical status code to a legitimate limit rejection. Tests assert `successCount=5` but get `1`.  

Prior hypothesis (incorrect): "Race B under outer transaction" — the theory that JPA REPEATABLE_READ snapshot prevents Thread B's JDBC SELECT from seeing Thread A's committed INSERT. **Investigation disproved this**: `solutionDbDatasource` (`SimpleTestDataSourceManager` → `BaseIntegrationTest.dataSource`) and Spring's auto-configured JPA DataSource are **separate HikariDataSource objects**. `DataSourceUtils.getConnection(solutionDbDatasource)` finds no `TransactionSynchronizationManager` binding for this object → returns a fresh auto-commit connection. Thread B's JDBC SELECT, once it acquires the lock, correctly sees Thread A's committed INSERT. Race B does NOT occur.  

T-10 fix (already applied): `writeNonOrgSummariesAtomically()` uses `rewardIssueSummaryJdbcRepository.findExistingForNonOrgLevel()` (JDBC) rather than JPA — correct and future-safe regardless of DataSource wiring, but NOT the cause of the T3/T4 test failure.  

Fix required: **T-19** — introduce `redis.reward.lock.maxWaitTime` property (default 5000 ms) distinct from the existing 10 ms customer-lock timeout. `NonOrgSummaryWriteProcessor` uses this longer timeout when acquiring the per-reward lock. 10 ms is appropriate for same-customer serialisation; 5000 ms is needed for cross-customer reward-level serialisation where N threads queue sequentially.

**✅ Resolution (T-19 implemented):**

The two timeouts serve fundamentally different semantics — which is why they must be separate properties:

| | Customer lock (`redis.lock.maxWaitTime=10ms`) | Reward lock (`redis.reward.lock.maxWaitTime=5000ms`) |
|---|---|---|
| **Key** | `{orgId}_{customerId}` | `reward_constraint:{orgId}:{rewardId}` |
| **Scope** | Per-customer (one key per customer) | Per-reward shared across all customers |
| **Threads contending** | Same customer hitting concurrently (rare; most apps retry) | Different customers, all legitimate concurrent requests |
| **Desired behaviour on timeout** | Fail fast — same customer retry is cheap | Queue — legitimate concurrent requests must not be rejected |

Raising `redis.lock.maxWaitTime` (the shared property) to 5000 ms would cause same-customer lock contention to hold threads open for 5 seconds instead of failing fast — unacceptable latency under pathological same-customer concurrency. The separate property keeps each lock at the right operating point.

Changes delivered:
- `redis.reward.lock.maxWaitTime=${REDIS_REWARD_LOCK_MAX_WAIT_TIME:5000}` added to both `src/main/resources/application.properties` (line 40) and `src/test/resources/application.properties` (line 41)
- `RewardsApplicationConfiguration.getRedisRewardLockAcquireMaxWaitTime()` — new `@Value`-bound getter
- `RedisLockService.acquireRewardLock(key)` — new method using `redisLockRegistry` + `redisRewardLockAcquireMaxWaitTime`; all other lock methods unchanged
- `NonOrgSummaryWriteProcessor.processWrapper()` — `acquireLock(lockKey)` → `acquireRewardLock(lockKey)`
- `RewardConstraintConcurrencyIntegrationTest` `@TestPropertySource` — replaced `redis.lock.maxWaitTime=5000` (wrong property, was overriding customer lock) with `redis.reward.lock.maxWaitTime=5000` (correct property for reward lock)

**GAP [SEVERITY: HIGH] — `writeNonOrgSummariesAtomically()` double-increments when multiple FIXED-window constraints share the same `(Level, KPI)` row**  
Category: Implementation regression introduced by CAPJUN19  
Root cause: `isEquivalentToConstraint()` ([RewardConstraint.java:144](src/main/java/com/capillary/solutions/rewards/db/RewardConstraint.java)) matches on `level + kpi` only — it does NOT distinguish `windowType` or `repeatFrequencyType`. When two FIXED-window constraints (e.g., FIXED/DAYS and FIXED/MONTHS) are configured at the same `(REWARD, QUANTITY)` level — a valid API configuration — both constraints resolve to the same physical `TBL_REWARD_ISSUE_SUMMARY` row. The loop in `writeNonOrgSummariesAtomically()` adds a separate entry to `toAtomicIncrement` (a `List`) for each, causing `bulkAtomicIncrement()` to run `CONSUMED = CONSUMED + delta` twice on the same row ID. For first issuance (no existing row), both constraints call `save()` on separate `newRow` objects → two rows inserted with identical `(level, kpi, userId, issueDate)`.  
The old `buildUniqueSummaries()` path was immune because it used a `Set` of object references — both constraints resolved to the same object in the Set, so the consumed was set only once in the final loop.  
Validated against: a brand configuring `REWARD-level QUANTITY FIXED/DAYS limit=5` + `REWARD-level QUANTITY FIXED/MONTHS limit=30` is allowed by `RewardConstraintValidation` (distinct constraint keys `QUANTITY_DAYS_FIXED` ≠ `QUANTITY_MONTHS_FIXED`). This is a realistic production combination (daily cap + monthly cap on same reward).  
Fix (deduplication before flush): deduplicate `toAtomicIncrement` by row ID (first match wins — delta is the same KPI quantity for all constraints sharing the row); deduplicate `toInsert` by `(level, kpi, userId, issueDate)` surrogate key before calling `save()`. See T-18.

**✅ Resolution (T-18 implemented):**

**Why `Map.putIfAbsent` with typed keys, not "check `toInsert` during the loop":**

The user raised the question: "should we look up `toInsert` along with `existingRows` when searching for a match?" — mirroring the old `buildUniqueSummaries()` accumulating-Set approach. Both strategies are semantically equivalent in outcome, but they differ in correctness risk:

*Old approach (accumulating Set):* `stream().filter(isEquivalentToConstraint).findFirst()` checked both DB rows AND previously created rows in the same collection. This worked despite `RewardIssueSummary` having no `@EqualsAndHashCode` (only identity equality) because the lookup used a custom predicate — `Set.contains()` was never called. The `HashSet<RewardIssueSummary>` was effectively a mutable list.

*T-18 approach (typed Map keys):* Deduplication happens at the accumulation step, not the lookup step. The Map keys are `Long` (rowId) and `String` (level|kpi|issueDate surrogate) — both JDK types with correct `equals()`/`hashCode()`. `RewardIssueSummary.equals()` is never called. `putIfAbsent` keeps the first match and silently drops duplicates.

**Why "look up `toInsert` in the loop" would be more complex:** Rows in `toInsert` have no database ID yet (pre-INSERT). If C2 found C1's staged row there, the only valid action is to skip C2 — you cannot add C2 to `toAtomicIncrement` (no ID). `putIfAbsent` achieves exactly that skip, at the accumulation step, without the loop-level lookup. Same outcome, simpler code.

**Why dropping C2's delta is correct:** C1 and C2 at the same `(level, kpi)` share the same logical consumed counter. The delta is derived from `quantity`/`points`/`redemptionValue` — identical for both, because they represent the same issuance event, not two independent events. One issuance = one increment on the shared counter, regardless of how many constraints map to that counter.

**`RewardIssueSummary.equals()` investigation:** Confirmed by reading [`RewardIssueSummary.java`](src/main/java/com/capillary/solutions/rewards/db/RewardIssueSummary.java) — annotations are `@Getter @Setter @Builder @Entity`. No `@EqualsAndHashCode`, no `@Data`. Default identity equality only. This does NOT affect the T-18 fix (values in the Map, not keys). It also did not affect the old code (Set backed by stream filter, not `contains()`). Documenting to prevent future developers from introducing a `Set<RewardIssueSummary>` dedup assuming structural equality would work.

Changes delivered ([RewardConstraintFacade.java:432–478](src/main/java/com/capillary/solutions/rewards/service/impl/RewardConstraintFacade.java)):
- `List<RewardIssueSummary> toAtomicIncrement` → `Map<Long, RewardIssueSummary>` keyed by `existing.getId()`; `putIfAbsent(existing.getId(), deltaEntry)` — second FIXED constraint finding same row is dropped
- `List<RewardIssueSummary> toInsert` → `Map<String, RewardIssueSummary>` keyed by `constraint.getConstraintLevel() + "|" + constraint.getKpi() + "|" + expectedDate.getTime()`; `putIfAbsent(key, newRow)` — second INSERT for same logical row is dropped
- Flush changed from `for newRow : toInsert` → `for newRow : toInsert.values()`; `bulkAtomicIncrement(toAtomicIncrement)` → `bulkAtomicIncrement(new ArrayList<>(toAtomicIncrement.values()))`
- Integration tests T7 (REWARD-level dual-FIXED concurrent, asserts 1 row consumed=3 not 6) and T8 (dual-FIXED at both REWARD + CUSTOMER level, asserts 1 row each consumed=3) added to `RewardConstraintConcurrencyIntegrationTest`

**GAP [SEVERITY: LOW] — `RewardConstraint.equals()`/`hashCode()` contract violation (pre-existing)**  
Category: Domain model design smell  
Finding: `RewardConstraint` ([RewardConstraint.java:27](src/main/java/com/capillary/solutions/rewards/db/RewardConstraint.java)) declares `@EqualsAndHashCode(of = {"id"})` at the class level, but also defines a manual `equals()` at lines 99–110 (business fields: `orgId, kpi, constraintLevel, repeatFrequencyType, limitValue, windowType`) and a manual `hashCode()` at lines 112–115 (`Objects.hash(id, orgId)`). Lombok skips generating `equals()` when a manual one exists — so the annotation is dead letter. The resulting contract is broken: two objects that are `equals()` by business fields can have different `hashCode()` (different `id`), violating the Java spec. Additionally, `rewardId` is absent from `equals()`, meaning constraints for different rewards with identical business fields are considered equal — a latent cross-reward collision if ever used outside single-reward context.  
Why safe now: `getIndependentRewardCustomerNoLimitRestrictions()` builds its `Set<RewardConstraint>` from the same object references that the loop checks with `contains()`. Same reference → same `id` → same `hashCode()` → correct bucket lookup. Safe as long as constraints are never reconstructed from separate queries or deserialized.  
This is a pre-existing bug; CAPJUN19 did not introduce it. However, the T-18 fix adds another `independentNoLimitConstraints.contains(constraint)` call path, making the risk worth documenting.  
No action required for Phase 1 — the existing usage pattern is safe. Note for future: if `RewardConstraint` equality is ever needed in a cross-reward context, fix the contract by including `rewardId` in `equals()` and aligning `hashCode()` with the same fields.

**GAP [SEVERITY: NOTE] — Compensation leaves zero-consumed rows (design decision)**  
Category: Data lifecycle design  
Finding: `bulkAtomicDecrement` SQL is `CONSUMED = CONSUMED - :delta`. For a newly inserted row (consumed=delta from INSERT), compensation brings consumed to 0. No DELETE path exists. The row stays with consumed=0.  
Why this is correct: consumed=0 < limitValue → future limit checks still pass. The next issuance finds the existing row and increments it — no second INSERT race. Semantically clean.  
Risk: On high-volume rewards with frequent downstream failures (coupon service flapping), zero-consumed rows accumulate indefinitely. No functional impact on correctness. Data hygiene concern only — monitoring `CONSUMED=0` rows over time can signal downstream failure rate spikes.  
Design decision confirmed: decrement (not delete) is intentional. DELETE would require additional coordination to avoid racing against concurrent reads inside the lock.

**GAP [SEVERITY: LOW] — `pos13HandledConstraintIds` naming leaks positional implementation detail**  
Category: Domain vocabulary  
What it is: Field name `pos13HandledConstraintIds` and accessor `getPos13HandledConstraintIds()` in `BulkRewardIssueContext` reference "pos-13" — a processor chain position that can change.  
Required correction: Rename to `atomicWrittenConstraintIds` and `getAtomicWrittenConstraintIds()` throughout.  
Files affected: `BulkRewardIssueContext.java:93`, `RewardConstraintFacade.java:350`, all callers.

**GAP [SEVERITY: LOW] — Comment stale references in `NonOrgSummaryWriteProcessor.java`**  
Category: Code hygiene  
Line 136: comment says `"CQ4: no refreshSummary() here — pos-12's read is authoritative"` — "pos-12" should be "RewardConstraintProcessor".  
Required correction: Replace positional reference with class name.

**GAP [SEVERITY: HIGH] — `compensateFailedSummaryWrites()` does not handle partial vendor issuance**  
Category: Implementation correctness — compensation correctness  
Identified: 2026-06-24 (tech lead review)

`writeNonOrgSummariesAtomically()` writes the full `delta = resolveAtomicDelta(kpi, quantityToBeProcessed, ...)` before vendor outcome is known — correct by design (atomic constraint check + write under Redis lock). `compensateFailedSummaryWrites()` is responsible for unwinding this pre-write for any quantity that vendor issuance failed. It has two coupled bugs:

**Bug A — filter:** `.filter(w -> !w.isAnySuccessful())` only fires when `successCount == 0`. `VendorIssueProcessor` makes one vendor API call per qty unit; `successCount` is incremented per success. For qty=2, successCount=1: `isAnySuccessful()` returns `true` → `!isAnySuccessful()` = `false` → the wrapper is excluded from compensation. The 1 failed qty's pre-written `CONSUMED` is **never decremented**.

**Bug B — delta:** Even if Bug A were fixed, `bulkAtomicDecrement(rows)` receives rows whose `consumed` field holds the original full-qty delta. For qty=2, failedQty=1: the decrement would use delta=2 (full) instead of 1 (failed only) — over-decrement by 1.

**Why not use `failureCount`?**  
`VendorIssueProcessor.java:163` — inside `catch(Exception e)` after `getVendorRewardDetails()` throws (wrapper-level exception covering all qty) — increments `failureCount` by 1 instead of `quantityToBeProcessed`. This breaks the invariant `successCount + failureCount == quantityToBeProcessed` for that path. `failureCount` is unreliable. See T-21.

**Fix (T-20):** Add `getFailedQuantity()` = `quantityToBeProcessed - successCount` to `RewardIssueWrapper`. `successCount` is always reliable (strictly per-unit increment). Fix the filter to `w.getFailedQuantity() > 0` and compute a per-KPI proportional delta:
- `QUANTITY`: delta = `failedQty`
- `REDEMPTION_VALUE`: delta = `w.getRedemptionValue() × failedQty`
- `TRANSACTION_COUNT`: delta = `BigDecimal.ONE` if `failedQty == totalQty` (total failure), else `BigDecimal.ZERO` (partial — the transaction DID happen)
- `POINTS`: not applicable — rejected at constraint creation time by `RewardConstraintValidation.java:68-69` (`POINTS_KPI_NOT_SUPPORTED`); cannot exist in `writtenSummaryRows` for non-org constraints

**Separate pre-existing bug (T-21):** `VendorIssueProcessor.java:163` — `setFailureCount(failureCount + 1)` should be `setFailureCount(failureCount + quantityToBeProcessed)`. Fix independently; T-20 is immune to this bug because it uses `getFailedQuantity()` (not `failureCount`).

### 5b. MADR Compliance

MADRs from `.context/overview.md`:
- **MADR-0001 (write-via-JDBC)** — PASS. All writes in `NonOrgSummaryWriteProcessor` and `RewardConstraintFacade.writeNonOrgSummariesAtomically()` use `RewardIssueSummaryJdbcRepository` directly (no JPA `.save()`).
- **MADR-0002 (datetime format)** — PASS. Dates trimmed to date-without-timestamp using `Utils.getDateWithoutTimestampInSpecifiedZone()` (consistent with existing pattern).

### 5c. Code Guardrail Compliance (from `.context/code.md`)

| Guardrail | Status | Finding |
|-----------|--------|---------|
| Redis Distributed Lock — use `RedisLockService` / `CustomerLockManager`, do not create bare `ReentrantLock` | PASS | `NonOrgSummaryWriteProcessor` uses `redisLockService.acquireRewardLock()` (T-19); customer lock still via `CustomerLockManager` |
| Redis Distributed Lock — always release in `finally` | PASS | `releaseLock(lock)` in finally at NonOrgSummaryWriteProcessor.java:131 |
| Redis Distributed Lock — lock TTL and max-wait-time are configured via properties | PASS | Reward lock: `redis.lock.ttl` (TTL) + `redis.reward.lock.maxWaitTime` (acquire wait, T-19). Customer lock: `redis.lock.ttl` + `redis.lock.maxWaitTime`. Both from `RewardsApplicationConfiguration`. |
| Processor Chain — keep processors single-purpose | PASS | `NonOrgSummaryWriteProcessor` has one responsibility: atomic write of non-org summaries |
| Processor Chain — carry all request-scoped state in context object | PASS | `writtenSummaryRows` and `atomicWrittenConstraintIds` carried in `BulkRewardIssueContext` |
| JDBC — correct parameter source for batch updates | PASS | `HashMap<String, Object>` arrays passed to `jdbc().batchUpdate()` |
| GC — no object churn in hot paths | PASS | Lists pre-allocated; no repeated allocation inside loops |
| Logging — no PII in log output | PASS | Logs only `rewardId`, `constraintId`, `orgId` — no customerId, mobile, email |
| Observability — New Relic attributes emitted for new code paths | PASS | `metricsService.addCustomParameter(NewRelicConstants.PARALLEL_CALL_LOCK_ACQUIRED, ...)` at NonOrgSummaryWriteProcessor.java:108,123 |

### 5d. Tenant Isolation Audit

- `orgId` is in every lock key: `reward_constraint:{orgId}:{rewardId}` — org A's lock cannot collide with org B.
- `orgId` parameter in all JDBC writes: `ATOMIC_INCREMENT_SQL` includes `AND ORG_ID = :orgId` ([RewardIssueSummaryJdbcRepository.java:170](src/main/java/com/capillary/solutions/rewards/jdbc/RewardIssueSummaryJdbcRepository.java)) — prevents cross-org row mutation.
- `writeNonOrgSummariesAtomically()` receives `orgId` explicitly and passes it to every JDBC call.
- No shared static state introduced. Context object is per-request.

### 5e. Transaction Boundary Audit

- `bulkSaveOrUpdate()` has `@Transactional` at the method level — INSERT and UPDATE are atomic within each call.
- `bulkAtomicIncrement()` and `bulkAtomicDecrement()` are batch SQL calls with no outer `@Transactional` — each row UPDATE is individually committed. This is acceptable: the Redis lock prevents concurrent increments; compensation handles downstream failures.
- No outer `@Transactional` added to the processor chain — consistent with the existing design (no long transactions wrapping external calls).
- Redis lock is external to Spring transaction management — no interaction with `@Transactional` propagation.

---

## 6. Alternative Designs Considered

| Alternative | Pros | Cons | Decision |
|------------|------|------|----------|
| **A (Chosen): Redis Distributed Lock + Atomic SQL** | No new infrastructure; lock hold time ~10–50ms; atomic SQL handles UPDATE and INSERT races uniformly for both FIXED and ROLLING windows | Lock contention under very high RPM | **Accepted for Phase 1** |
| **B: 3-Tier Atomic Counter (Redis Lua + MongoDB Journal + MySQL flush job)** | True atomic check-and-reserve; no lock contention; horizontally scalable | New MongoDB collection; new scheduled job; complex cold-start/restart recovery; 3–4× implementation surface | **Deferred** — right escalation if lock contention observed in production at scale |
| **SELECT FOR UPDATE** | No new infrastructure; DB-native | Does not protect INSERT path; requires new `@Transactional` boundary; dual-datasource coordination complex (read replica vs write master) | **Rejected** — CQ6 in investigation doc |

---

## 7. Use Cases

### B2B (Brand Admin Flows)

| ID | Flow | Before Fix | After Fix | Risk | Test Type |
|----|------|------------|-----------|------|-----------|
| UC-1 | Admin creates reward with REWARD-level QUANTITY limit=10; 8 customers issue concurrently | Up to 10+ succeed (Race A); consumed value under-counted (Race B/C) | Exactly 10 succeed; consumed = 10 atomically | Limit breach affecting brand's reward budget | Integration (T4) |
| UC-2 | Admin creates reward with REWARD-level limit=1; 2 customers issue simultaneously | Both succeed (limit = 2 observed) | Exactly 1 succeeds; other gets CONSTRAINT_EVALUATION_FAILED | Budget overshoot; brand pays for 2 instead of 1 | Integration (T2) |
| UC-3 | Admin creates reward with CUSTOMER-level limit=2; same customer issues 3× serially | Third correctly rejected (existing customer lock) | Unchanged behavior | N/A — regression risk | Integration (T5) |
| UC-4 | Admin creates reward; 5 customers issue concurrently; no duplicate rows in DB | 5 duplicate REWARD-level rows inserted (Race B) | Exactly 1 REWARD-level row; consumed = 5 | Under-count on subsequent reads compounds future breach | Integration (T3) |
| UC-5 | Admin adds REWARD + CUSTOMER level constraints; 5 customers issue concurrently against REWARD limit=3 | All 5 succeed (REWARD-level ignored under concurrency) | Exactly 3 succeed; REWARD-level limits | Whole class of org budget controls bypassed | Integration (T1 extended) |

### B2C (End Customer Flows)

| ID | Flow | Before Fix | After Fix | Risk | Test Type |
|----|------|------------|-----------|------|-----------|
| UC-6 | Customer issues reward; downstream coupon issue fails; summary row already written | CONSUMED incremented but reward not issued; customer deceived | Compensation decrement fires in finally; CONSUMED decremented back | Customer's future issuance wrongly blocked | Integration (compensation scenario) |
| UC-7 | Customer issues reward; all processors succeed | Summary written inside lock; atomic increment | Unchanged happy path — faster (no updateSummaries double write) | N/A — regression risk | Integration (T1, T5, T6) |
| UC-8 | Customer at limit exactly; concurrent request arrives | Both pass (Race A), limit breached | Second request re-evaluates inside lock; sees full limit; CONSTRAINT_EVALUATION_FAILED | Customer over-issues; brand budget breached | Integration (T2) |
| UC-9 | Two customers race to be the first issuance (no existing summary row) | Two INSERTs; each row shows consumed=1; aggregate under-counts | First thread INSERTs inside lock; second thread finds existing row and increments atomically | Silent corruption — future evaluations under-count consumed | Integration (T3) |
| UC-10 | ROLLING window REWARD-level constraint; 2 customers race | Race A + C expose limit breach (identical root cause as FIXED window) | Per-reward lock serializes; row written at `issualDate` via atomic increment — same path as FIXED window | No residual risk for ROLLING REWARD-level | Integration (T2 variant in analysis doc) |

---

## 8. Low-Level Design

### Call Chain (Actual — Confirmed from Code)

```
HTTP Controller
  → UserRewardFacade.issueBulk()
      → UserRewardUtils.issueRewardBulk()          @CustomerLockable(orgId, customerId)
          [customer lock acquired]
          try {
            BulkIssueService.issueBulk()
              → [processors 1–11: paymentConfig, catalog, orgSettings, idempotency, ...]
              → RewardConstraintProcessor.process()     [speculative eval, no lock]
              → NonOrgSummaryWriteProcessor.process()   [authoritative eval + atomic write]
                   if hasRewardLevelConstraint:
                     redisLockService.acquireLock("reward_constraint:{orgId}:{rewardId}")
                     try {
                       reEvaluateConstraints()           [refreshSummary inside lock]
                       rewardConstraintFacade.writeNonOrgSummariesAtomically()
                     } finally {
                       redisLockService.releaseLock(lock)
                     }
                   if hasCustomerLevelConstraint:
                     rewardConstraintFacade.writeNonOrgSummariesAtomically()  [no extra lock needed]
              → loyaltyProgramValidatorProcessor
              → couponIssueProcessor                [external I/O — OUTSIDE lock]
              → vendorIssueProcessor
              → pointsRedeemProcessor
          } finally {
            userRewardFacade.compensateFailedSummaryWrites()
              → rewardConstraintFacade.compensateFailedSummaryWrites()
                   for each failed wrapper:
                     rewardIssueSummaryJdbcRepository.bulkAtomicDecrement(rows)
          }
          [customer lock released]
      → UserRewardFacade.updateSummaries()          [CQ9-filtered — skips atomically-written rows]
```

### Class/Method Sketch

**`NonOrgSummaryWriteProcessor`** — [NonOrgSummaryWriteProcessor.java:53](src/main/java/com/capillary/solutions/rewards/service/impl/issue/processors/NonOrgSummaryWriteProcessor.java)

```
Responsibility: Authoritative enforcement of REWARD- and CUSTOMER-level constraints.
                Acquires per-reward lock (REWARD only), re-evaluates, writes atomically.

+ process(ctx: BulkRewardIssueContext): BulkRewardIssueContext
    → redisLockService.acquireRewardLock("reward_constraint:{orgId}:{rewardId}")  [conditional — T-19]
        uses: redisRewardLockAcquireMaxWaitTime (5000 ms default, NOT the 10 ms customer-lock timeout)
    → reEvaluateConstraints(): boolean    [refreshSummary inside lock for REWARD-level]
    → rewardConstraintFacade.writeNonOrgSummariesAtomically()
    → ctx.recordWrittenSummaryRows(rewardId, written)
    → ctx.addHandledConstraintId(constraintId)
    → Emits: NewRelicConstants.PARALLEL_CALL_LOCK_ACQUIRED
    → Throws: CONSTRAINT_EVALUATION_FAILED (re-eval failed OR lock timeout)
    Guardrail check: PASS
```

**`RewardConstraintFacade.writeNonOrgSummariesAtomically()`** — [RewardConstraintFacade.java:403](src/main/java/com/capillary/solutions/rewards/service/impl/RewardConstraintFacade.java)

```
Responsibility: Atomic write for non-org summary rows.
                INSERT for new rows (via save() returning ID).
                CONSUMED = CONSUMED + delta for existing rows (via bulkAtomicIncrement()).

+ writeNonOrgSummariesAtomically(orgId, orgZoneId, userId, wrapper, levelsToWrite): List<RewardIssueSummary>
    → rewardIssueSummaryJdbcRepository.findExistingForNonOrgLevel()  [JDBC — bypasses JPA REPEATABLE_READ snapshot; inside caller's lock]
    → for each constraint (skipping dependent NO_LIMIT):
        → find existing row by (level, kpi, userId, issueDate) match
        → if existing: toAtomicIncrement.putIfAbsent(existing.getId(), deltaEntry)
                    Key = Long(rowId) — Long.equals() is correct; RewardIssueSummary.equals() never called
                    First FIXED-window constraint wins; FIXED/DAYS + FIXED/MONTHS at same (level,kpi) deduplicated to one increment
        → if not existing: insertKey = level+"|"+kpi+"|"+issueDate.getTime()
                    toInsert.putIfAbsent(insertKey, newRow)
                    Key = String — String.equals() is correct; second constraint for same logical row dropped silently
    → for newRow in toInsert.values(): rewardIssueSummaryJdbcRepository.save(newRow)  [INSERT — returns ID for compensation]
    → rewardIssueSummaryJdbcRepository.bulkAtomicIncrement(new ArrayList<>(toAtomicIncrement.values()))
                    [CONSUMED = CONSUMED + delta — one entry per unique row ID, no double-increment]
    Returns: entries with consumed = delta (for compensation — caller stores these)
    Guardrail check: PASS
    Bug pre-T-18: toAtomicIncrement was a List; two FIXED-window constraints at same (level, kpi) produced two entries for the same row ID → CONSUMED += delta twice.
                  toInsert was a List; same scenario on first issuance produced two save() calls → duplicate rows with identical (level, kpi, userId, issueDate).
    Note: RewardIssueSummary has no @EqualsAndHashCode (identity equality only). The Map key approach is correct precisely because it uses String/Long keys — NOT RewardIssueSummary equality. Any future use of Set<RewardIssueSummary> for structural dedup would silently fail.
```

**`RewardConstraintFacade.updateSummaries()`** — [RewardConstraintFacade.java:330](src/main/java/com/capillary/solutions/rewards/service/impl/RewardConstraintFacade.java)

```
CQ9 filter added — skips rows already written atomically by NonOrgSummaryWriteProcessor.

+ updateSummaries(orgId, orgZoneId, userId, wrapper, ctx)
    → getExistingRewardIssueSummariesForOrgLevel()
    → getExistingRewardIssueSummariesForNonOrgLevel()
    → ctx.getWrittenSummaryRowIds(rewardId)          [CQ9 exclusion — row ID filter]
    → ctx.getAtomicWrittenConstraintIds()            [CQ9 exclusion — constraint filter]
    → filter nonOrgLevelRewardIssueSummaries         [exclude atomically-written rows]
    → filter constraintsToProcess                    [exclude atomically-handled constraints]
    → buildUniqueSummaries() → bulkSaveOrUpdate()    [org-level only; ROLLING REWARD/CUSTOMER-level excluded by CQ9 atomicWrittenConstraintIds filter]
    Guardrail check: PASS
```

**`RewardConstraintFacade.compensateFailedSummaryWrites()`** — [RewardConstraintFacade.java:506](src/main/java/com/capillary/solutions/rewards/service/impl/RewardConstraintFacade.java)

```
+ compensateFailedSummaryWrites(ctx: BulkRewardIssueContext)
    → ctx.streamAllRewardIssueWrapper().filter(!isAnySuccessful())
    → ctx.getWrittenSummaryRowsForCompensation(rewardId)
    → rewardIssueSummaryJdbcRepository.bulkAtomicDecrement(rows)
                                         [CONSUMED = CONSUMED - delta WHERE ID = :id AND ORG_ID = :orgId]
    Guardrail check: PASS — try/catch wraps entire method to prevent compensation failure from masking original error
```

**`RewardIssueSummaryJdbcRepository`** — [RewardIssueSummaryJdbcRepository.java:169](src/main/java/com/capillary/solutions/rewards/jdbc/RewardIssueSummaryJdbcRepository.java)

```
New SQL constants:
  ATOMIC_INCREMENT_SQL:
    UPDATE TBL_REWARD_ISSUE_SUMMARY SET CONSUMED = CONSUMED + :delta
    WHERE ID = :id AND ORG_ID = :orgId

  ATOMIC_DECREMENT_SQL:
    UPDATE TBL_REWARD_ISSUE_SUMMARY SET CONSUMED = CONSUMED - :delta
    WHERE ID = :id AND ORG_ID = :orgId

+ bulkAtomicIncrement(deltas: List<RewardIssueSummary>)
    → jdbc().batchUpdate(ATOMIC_INCREMENT_SQL, params[])
    Note: consumed field on input objects holds DELTA, not final value.

+ bulkAtomicDecrement(deltas: List<RewardIssueSummary>)
    → jdbc().batchUpdate(ATOMIC_DECREMENT_SQL, params[])
    Guardrail check: PASS — LAST_UPDATED_ON auto-updated by MySQL ON UPDATE CURRENT_TIMESTAMP
```

**`BulkRewardIssueContext`** — [BulkRewardIssueContext.java:86](src/main/java/com/capillary/solutions/rewards/dto/wrapper/BulkRewardIssueContext.java)

```
New fields:
  writtenSummaryRows:          Map<Long, List<RewardIssueSummary>>  (keyed by rewardId)
  atomicWrittenConstraintIds:  Set<Long>

New methods:
  recordWrittenSummaryRows(rewardId, rows)
  addHandledConstraintId(constraintId)
  getWrittenSummaryRowsForCompensation(rewardId): List<RewardIssueSummary>
  getWrittenSummaryRowIds(rewardId): Set<Long>
  getAtomicWrittenConstraintIds(): Set<Long>
```

**`UserRewardUtils`** — [UserRewardUtils.java:56](src/main/java/com/capillary/solutions/rewards/service/UserRewardUtils.java)

```
Both issueRewardBulk() and issueReward() modified:
  try {
    userRewardList = bulkIssueService.issueBulk(bulkRewardIssueContext);
  } finally {
    userRewardFacade.compensateFailedSummaryWrites(bulkRewardIssueContext);
  }
  Guardrail check: PASS — compensation always fires; no exception swallowed
```

**`BulkIssueService.buildBulkIssueProcessors()`** — [BulkIssueService.java:97](src/main/java/com/capillary/solutions/rewards/service/impl/BulkIssueService.java)

```
Chain order (confirmed):
  [0]  paymentConfigIssueProcessor
  [1]  catalogPromotionProcessor
  [2]  orgSettingsProcessor
  [3]  idempotencyCheckIssueProcessor
  [4]  openTransactionProcessor
  [5]  groupRedemptionCheckProcessor
  [6]  issueRewardSegmentValidator
  [7]  issueRewardCardValidator
  [8]  issueRewardLabelValidator
  [9]  isRedeemableProcessor
  [10] isRedeemableExtProcessor
  [11] rewardConstraintProcessor        ← speculative eval (no lock)
  [12] nonOrgSummaryWriteProcessor      ← authoritative eval + atomic write
  [13] loyaltyProgramValidatorProcessor
  [14] couponIssueProcessor             ← external I/O begins here
  [15] promotionEarnProcessor
  [16] vendorIssueProcessor
  [17] pointsRedeemProcessor
  [18] pointsRedeemExtProcessor
  [19] couponRevokeProcessor
  [20] revokeEarnProcessor
```

---

## 9. Data Model Changes

No data model changes. `TBL_REWARD_ISSUE_SUMMARY` schema is unchanged.

The `ATOMIC_INCREMENT_SQL` and `ATOMIC_DECREMENT_SQL` operate on the existing `CONSUMED` column (DECIMAL type, precision 13,4 — confirmed from `save()` mapping `Types.DECIMAL`).

`LAST_UPDATED_ON` is auto-updated by MySQL `ON UPDATE CURRENT_TIMESTAMP` ([TBL_REWARD_ISSUE_SUMMARY.sql:13](src/test/resources/cc-stack-crm/schema/solutions/rewards_utf8_db/TBL_REWARD_ISSUE_SUMMARY.sql)) — no explicit column update needed in SQL.

No DBA migration required. The covering index `idx_rewardIssue_summary_user_id (ORG_ID, REWARD_ID, USER_ID, CONSTRAINT_LEVEL, KPI, ISSUE_DATE)` confirmed present in DDL and covers all `refreshSummary()` SUM queries.

---

## 10. API Changes

No API changes. The `issueReward` endpoint contract is unchanged. The fix is entirely internal to the processing chain.

Error responses for constraint violations continue to use `CONSTRAINT_EVALUATION_FAILED` (code 12005) — same status code as before. The error message text is enriched (mentions lock detection and constraint ID), but the code is stable.

---

## 11. Security

| Check | Result | Notes |
|-------|--------|-------|
| New API endpoints requiring auth/role checks | N/A | No new endpoints |
| PII in new log lines | PASS | Logs: `rewardId`, `constraintId`, `orgId` only — no customerId, mobile, email |
| New fields storing PII | N/A | No new DB columns or fields storing PII |
| Token / credential handling | N/A | No tokens or credentials involved |
| Tenant data leak risk (new shared/global state) | PASS | Lock key = `reward_constraint:{orgId}:{rewardId}` — org-scoped; `ATOMIC_INCREMENT_SQL` includes `AND ORG_ID = :orgId` — org-scoped write |
| User-supplied input in new code paths | PASS | `rewardId` and `orgId` come from authenticated session context, not user-supplied request body; constraint limits come from DB, not request |
| Redis lock key collision with existing prefixes | PASS | Existing keys use `redisLockRegistry` with prefix `rewards-lock`; reward lock key `reward_constraint:` is distinct from customer lock key `{orgId}_{customerId}` |
| Lock acquisition max wait time — DoS risk | NOTE | If `redisLockAcquireMaxWaitTime` is very long, a lock-holder crashing could block callers for the wait duration. Mitigated by `redis.lock.ttl` TTL on the lock itself. Configure TTL appropriately (see §12). |

---

## 12. DB & Infra Considerations

### DB

| Finding | Impact if Ignored | Recommended Action | Owner |
|---------|------------------|-------------------|-------|
| No new schema migration needed | N/A | None | — |
| `LAST_UPDATED_ON` auto-updated via `ON UPDATE CURRENT_TIMESTAMP` | Databricks delta ETL continues to work correctly | Confirm this is present in production DDL (not just test resources) | DBA |
| Covering index `idx_rewardIssue_summary_user_id (ORG_ID, REWARD_ID, USER_ID, CONSTRAINT_LEVEL, KPI, ISSUE_DATE)` — `refreshSummary()` SUM query runs inside the lock; query latency = lock hold time | If index is missing, full-table-scan extends lock hold time; all concurrent callers queue | Verify index exists in production before deploy | DBA |
| NO_LIMIT constraints: `refreshSummary()` has no date filter — scans all rows for a reward's history | Long-running rewards (3+ years) accumulate hundreds of rows; warm read ~1–8ms; lock utilisation < 26% at 33 req/sec | Monitor `refreshSummary` P99 inside lock; alert if > 10ms (Phase 1b trigger) | Implementer (monitoring) |

### Infra

| Finding | Impact if Ignored | Recommended Action | Owner |
|---------|------------------|-------------------|-------|
| New Redis lock key namespace: `reward_constraint:{orgId}:{rewardId}` via existing `redisLockRegistry` | Collides with existing key namespace if same key is used elsewhere | Confirm `reward_constraint:` prefix is unique in Sentinel cluster | Implementer / DevOps |
| Lock TTL = `redis.lock.ttl` (same as customer lock) — lock hold time is ~10–50ms (DB read + evaluate + write) | If TTL < P99 of lock hold time + GC pause buffer, two threads can hold the lock simultaneously | Configure TTL ≥ 500ms; confirm via New Relic before deploy | Implementer / DevOps |
| `redisLockAcquireMaxWaitTime` — concurrent cross-customer threads for same reward queue on this wait time | Too short: threads fail before lock is released (causes T2/T3/T4 test failures). Too long: late-arriving requests wait unnecessarily | Set wait time ≥ nominal lock hold time × expected queue depth; investigate current value | Implementer (T2/T3/T4 fix) |
| `NonOrgSummaryWriteProcessor.process()` P99 — new in processor chain | If P99 > 50ms, indicates DB or lock contention under load | Emit New Relic custom attribute for this method's duration; alert if > 50ms | Implementer |

---

## 13. Internal Architecture Changes

| Type | Change |
|------|--------|
| New class | `NonOrgSummaryWriteProcessor` — implements `IssueRewardProcessor`, inserted at chain position 13 |
| New pattern | Per-reward Redis distributed lock for REWARD-level constraint serialization (distinct from customer lock); acquired via `acquireRewardLock()` using `redis.reward.lock.maxWaitTime` (5000 ms) — separate from the 10 ms `redis.lock.maxWaitTime` customer-lock timeout |
| New pattern | Atomic SQL increment/decrement (`CONSUMED = CONSUMED + :delta`) for race-safe summary writes |
| New pattern | Two-phase write responsibility: pos-13 writes REWARD+CUSTOMER-level (inside lock, regardless of WindowType — FIXED and ROLLING handled identically via `issualDate` resolution); `updateSummaries()` writes ORG-level only |
| New pattern | CQ9 filter in `updateSummaries()` — context carries `writtenSummaryRows` and `pos13HandledConstraintIds` to prevent double-write |
| New pattern | Compensation decrement in try-finally at `UserRewardUtils` — fires on any downstream failure after pos-13 write |
| Modified | `RewardConstraintFacade` — `updateSummaries()` CQ9 filter; `writeNonOrgSummariesAtomically()` new method; `compensateFailedSummaryWrites()` new method |
| Modified | `RewardIssueSummaryJdbcRepository` — `bulkAtomicIncrement()`, `bulkAtomicDecrement()` new methods with CONSUMED+/- SQL |
| Modified | `BulkRewardIssueContext` — `writtenSummaryRows`, `atomicWrittenConstraintIds` fields with recording methods |
| Modified | `UserRewardUtils` — try-finally wraps `issueBulk()` for compensation |
| Modified | `BulkIssueService.buildBulkIssueProcessors()` — `nonOrgSummaryWriteProcessor` injected between pos 12 and pos 14 |

**New codebase convention established:** Processor-internal Redis lock for shared-budget serialization is separate from and coarser than the customer-level `@CustomerLockable` lock. Future processors that write shared cross-customer resources should follow this pattern.

---

## 14. Upstream / Downstream Impact

### Upstream (systems feeding into this flow)

No upstream change. The `issueReward` API contract is unchanged.

### Downstream (systems consuming from this flow)

| System | What Changes | Coordination Needed |
|--------|-------------|---------------------|
| `TBL_REWARD_ISSUE_SUMMARY` (Databricks ETL) | `LAST_UPDATED_ON` auto-updated on each atomic increment — ETL delta loads continue to pick up changes | Confirm production DDL has `ON UPDATE CURRENT_TIMESTAMP`; no ETL pipeline change required |
| Audit log / Envers | `TBL_REWARD_ISSUE_SUMMARY_AUD` — each atomic `UPDATE` is audited normally (Envers triggers on UPDATE) | No coordination needed |
| Monitoring / New Relic | New `PARALLEL_CALL_LOCK_ACQUIRED` attribute emitted — available for alerting | Create alert on `false` values above threshold (indicates lock contention spike) |

---

## 15. SLA Impact

### Latency

- `NonOrgSummaryWriteProcessor` adds one DB read (`refreshSummary`) + one DB write (`bulkAtomicIncrement` or `save`) to the critical path, inside the reward lock.
- Nominal lock hold time: ~10–50ms (DB round trips at P50).
- For rewards with **no REWARD-level constraint** (customer-level only): no lock acquired; DB write is a single `bulkAtomicIncrement` call — latency impact < 5ms.
- For rewards with REWARD-level constraint: P99 of the full chain increases by ~50ms under normal load.

### Throughput (Lock Contention)

- At 33 req/sec peak (current production estimate) with 50ms lock hold: lock utilisation = 1.65 calls × 50ms = 82ms/sec = 8% — well within safe range.
- At higher RPM (future), contention increases linearly. Monitor `PARALLEL_CALL_LOCK_ACQUIRED=false` rate as the throughput signal for Phase 2 escalation.

### Degraded Mode

- If the reward lock cannot be acquired within `redisLockAcquireMaxWaitTime`, the request returns `CONSTRAINT_EVALUATION_FAILED` rather than timing out or returning an ambiguous error. Caller can retry.

---

## 16. Observability

| Signal | Where | Alert Condition |
|--------|-------|----------------|
| `PARALLEL_CALL_LOCK_ACQUIRED = false` | New Relic custom attribute (NonOrgSummaryWriteProcessor.java:123) | > 5 false/min sustained → lock contention spike; investigate RPM |
| `PARALLEL_CALL_LOCK_ACQUIRED = true` | New Relic custom attribute (NonOrgSummaryWriteProcessor.java:108) | No alert — used for lock acquisition rate baseline |
| `NonOrgSummaryWriteProcessor.process()` P99 | New Relic `@Trace` on `process()` method | Alert if P99 > 100ms (indicates DB query latency inside lock) |
| `refreshSummary` P99 (inside lock, NO_LIMIT) | Recommend emitting `NEWRELIC_CUSTOM_REWARDS_EVENT` attribute | Alert if > 10ms — this is the Phase 1b gate signal |
| `CONSTRAINT_COMPENSATION_DECREMENT` log line | `log.warn` in `compensateFailedSummaryWrites()` | Any non-zero rate → downstream failure causing compensation |
| `TBL_REWARD_ISSUE_SUMMARY` CONSUMED > limitValue for REWARD-level constraints | Post-deploy monitoring query | Should drop to zero for FIXED/NO_LIMIT after deploy |

---

## 17. Rollout Plan

1. **Pre-deploy gate:** Confirm covering index `idx_rewardIssue_summary_user_id` exists in production MySQL before deploy. Confirm `redis.lock.ttl ≥ 500ms` is configured in `sol-rewards-core-a.json` for all clusters.
2. **Deploy:** Standard rolling deploy — no feature flag needed (processor chain is always active; new processor is conditional on REWARD-level constraint presence).
3. **Staging validation (required):** Run 50 concurrent `issueReward` calls for a reward with REWARD-level limit=10; confirm exactly 10 succeed and 40 get `CONSTRAINT_EVALUATION_FAILED`. Confirm exactly 1 `TBL_REWARD_ISSUE_SUMMARY` REWARD-level row exists with `consumed = 10`.
4. **Go/no-go criteria:**
   - Integration tests T1–T8 all pass ✅ (T2/T3/T4 fixed by T-19; T7/T8 verify T-18 dedup fix)
   - Staging validation confirms limit enforcement
   - `P99 of NonOrgSummaryWriteProcessor.process()` < 100ms under load
5. **Rollback:** Revert to prior commit. CONSUMED values written atomically will be correct; no corrupted state to clean up. Lock TTL expiry is automatic — Redis cleans up within configured TTL.
6. **Post-deploy watch:** Monitor for 24 hours: `PARALLEL_CALL_LOCK_ACQUIRED = false` rate, `CONSTRAINT_COMPENSATION_DECREMENT` log rate, `NonOrgSummaryWriteProcessor` P99.

---

## 18. Risks

| # | Description | Likelihood | Mitigation |
|---|------------|-----------|-----------|
| 1 | ~~T2/T3/T4 integration tests not fixed before deploy~~ | ~~Medium~~ | ✅ **Closed** — T4 root cause is 10 ms lock timeout, not JPA REPEATABLE_READ (T-10 was valid JDBC cleanup but not the cause). T-19 adds separate reward-lock wait time (5000 ms). T-10 ✅ Done. |
| 2 | **Reward-level lock timeout too short for production** — `redis.lock.maxWaitTime=10ms` (default) causes threads B–N to time out immediately when Thread A holds the reward lock for its full DB processing time (~10–100 ms in prod). All-but-first concurrent requests fail with `CONSTRAINT_EVALUATION_FAILED` on popular rewards. | **High** | ✅ **Root cause confirmed.** T-19 adds `redis.reward.lock.maxWaitTime` (default 5000 ms) used exclusively in `NonOrgSummaryWriteProcessor`. Existing 10 ms customer-lock timeout unchanged. |
| 9 | **Lock timeout misconfigured in tests (`REDIS_LOCK_MAX_WAIT_TIME=10ms`)** — confirmed root cause of T3/T4 failures. All threads except Thread A time out. Fixed by T-19 property separation. | High (tests) | T-19: test properties get `redis.reward.lock.maxWaitTime=5000`; prod gets env-var default 5000 ms |
| 3 | **Compensation not triggered for partial vendor issuance — `compensateFailedSummaryWrites()` has two coupled bugs.** Bug A: `.filter(w -> !w.isAnySuccessful())` fires only when `successCount==0`; for qty=2 with 1 success, the wrapper is silently excluded. Bug B: decrement delta uses the original full-qty pre-write value instead of the proportional `failedQty` delta. Both bugs leave CONSUMED permanently over-counted for every partial-success vendor issuance. | **High** | T-20: add `getFailedQuantity()` = `quantityToBeProcessed - successCount` to `RewardIssueWrapper`; fix filter to `getFailedQuantity() > 0`; compute per-KPI proportional delta (QUANTITY=failedQty, REDEMPTION_VALUE=rv×failedQty, TRANSACTION_COUNT=1 only on total failure). |
| 12 | **`VendorIssueProcessor.java:163` — `failureCount` incremented by 1 for wrapper-level exception (pre-existing).** The `catch(Exception e)` path after `getVendorRewardDetails()` throws covers all qty units but calls `setFailureCount(failureCount + 1)` instead of `+ quantityToBeProcessed`. Breaks invariant `successCount + failureCount == quantityToBeProcessed` for this failure path. | Low (pre-existing; T-20 fix is immune via `getFailedQuantity()`) | T-21: change `setFailureCount(failureCount + 1)` → `setFailureCount(failureCount + quantityToBeProcessed)` at `VendorIssueProcessor.java:163`. Independent fix; does not block T-20. |
| 4 | ORG-level constraints still exposed to Race A and Race C under Phase 1 | Medium (known, accepted; lower severity than REWARD-level) | Deferred to Phase 2; requires `acquireBulkLock()` strategy distinct from the per-reward lock |
| 5 | `pos13HandledConstraintIds` naming — ~~resolved~~ | — | Renamed to `atomicWrittenConstraintIds` throughout (T-8 ✅ Done) |
| 6 | ~~Production DDL for `TBL_REWARD_ISSUE_SUMMARY` missing `ON UPDATE CURRENT_TIMESTAMP`~~ | ~~Low~~ | ✅ **Closed** — DBA confirmed production DDL has `ON UPDATE CURRENT_TIMESTAMP` on `LAST_UPDATED_ON`. |
| 7 | Lock key `reward_constraint:` prefix collides with future key usage | Low | Document in code; adopt convention of prefix registry for all Redis keys |
| 10 | **`RewardConstraint.equals()`/`hashCode()` contract violated (pre-existing)** — `@EqualsAndHashCode(of = {"id"})` annotation is dead (manual equals() overrides it); manual equals() uses business fields while hashCode() uses id+orgId; `rewardId` absent from equals(). Current usage (`contains()` on same object references) is safe. Risk: latent cross-reward collision if constraints are ever compared across rewards or deserialized. | Low (pre-existing) | No Phase 1 action. Future fix if cross-reward equality ever needed: include `rewardId` in equals() and align hashCode() with same fields. |
| 11 | **Zero-consumed rows after compensation** — decrement (not delete) leaves rows with consumed=0 in `TBL_REWARD_ISSUE_SUMMARY`. Functionally correct; no limit-check impact. Accumulation rate = downstream failure rate. | Low (design decision) | Monitor `SELECT COUNT(*) FROM TBL_REWARD_ISSUE_SUMMARY WHERE CONSUMED=0` as downstream failure rate signal. No cleanup required unless volume becomes operationally significant. |
| 8 | ~~**Multi-FIXED-window double-increment regression**~~ | ~~High~~ | ✅ **Closed** — T-18 applied `Map.putIfAbsent` dedup keyed by `Long(rowId)` for increments and `String(level\|kpi\|issueDate)` for inserts. `RewardIssueSummary.equals()` is NOT used (identity equality only; Map keys are JDK types). Integration tests T7 and T8 confirm 1 row per level, consumed=N not 2N. |

---

## 19. Open Questions

| Question | Owner | Due | Answer |
|---------|-------|-----|--------|
| What percentage of active reward constraints are ORG-level (`rewardId = -1L`)? Determines Phase 2 urgency. ROLLING window is covered by Phase 1 — this question is about ORG-level scope only. | Data team | Before Phase 2 planning | ✅ **Answered:** Not many brands are using ORG-level constraints yet. Phase 2 urgency is LOW — can be deferred without immediate production risk. |
| What is the current `redis.lock.ttl` value for the `redisLockRegistry` (customer lock registry) in production? Is it ≥ 500ms? | DevOps | Before deploy | ⬜ Open |
| Does production `TBL_REWARD_ISSUE_SUMMARY` DDL have `ON UPDATE CURRENT_TIMESTAMP` on `LAST_UPDATED_ON`? | DBA | Before deploy | ✅ **Answered:** Yes — production DDL has `ON UPDATE CURRENT_TIMESTAMP`. Databricks ETL delta pickup via `LAST_UPDATED_ON` is confirmed safe. Risk #6 closed. |
| **T4 fix verification**: Does `solutionDbDatasource` participate in the outer Spring transaction (i.e., does `DataSourceUtils.getConnection(solutionDbDatasource)` return the transaction-bound connection)? If yes, the JDBC SELECT in `findExistingForNonOrgLevel()` still sees the outer transaction snapshot. | Implementer | Immediately (blocks T4 fix) | ✅ **Answered** — `solutionDbDatasource` (`SimpleTestDataSourceManager` → `BaseIntegrationTest.dataSource`) and Spring's auto-configured JPA DataSource are **separate `HikariDataSource` objects**. `DataSourceUtils.getConnection(solutionDbDatasource)` finds no `TransactionSynchronizationManager` binding → returns a fresh auto-commit connection. Thread B's JDBC SELECT, once inside the lock, correctly sees Thread A's committed INSERT. Race B does NOT occur via JPA REPEATABLE_READ snapshot. T4 was failing due to 10ms lock timeout (T-19), not DataSource isolation. |

---

## 20. Questions for arch-investigator

**[CLARIFICATION — before Phase 2]**

Q1: The investigation doc describes `acquireBulkLock()` for Phase 2 ORG-level constraints as "already deadlock-safe (sorts keys alphabetically)". The current `acquireBulkLock()` implementation does sort keys — but confirm: does it release all held locks atomically if any one acquisition fails, or only the last failed lock? The risk is partial acquisition leaving orphaned locks.
- Surfaced in: Phase 3e — transaction boundary audit
- Why it matters: Deadlock risk for Phase 2 ORG-level multi-key acquisition

**[ASSUMPTION CHECK — validate before Phase 1 deploy]**

Q2: Is the `redisLockRegistry` (used by `acquireLock(String key)`) on the same Redis Sentinel cluster as the customer locks, or a different cluster? A split-brain event on the cluster affects both customer and reward locks simultaneously — is that acceptable?
- Surfaced in: Phase 3d — tenant isolation audit
- Why it matters: Single point of failure scope for all reward issuance
- ✅ **Answered:** Same Redis Sentinel cluster. A split-brain event would affect both customer locks and reward locks simultaneously. This is accepted: both lock types use the same cluster today; the blast radius is identical to the existing customer-lock failure mode. No additional mitigation required for Phase 1.

---

## 21. Task Breakdown

### Backend

| # | Description | Size | Dependencies | Status |
|---|------------|------|--------------|--------|
| T-1 | Add `writtenSummaryRows` + `pos13HandledConstraintIds` to `BulkRewardIssueContext` | S | — | ✅ Done |
| T-2 | Add `bulkAtomicIncrement` + `bulkAtomicDecrement` to `RewardIssueSummaryJdbcRepository` | S | — | ✅ Done |
| T-3 | Add `writeNonOrgSummariesAtomically`, `compensateFailedSummaryWrites`, modify `updateSummaries` CQ9 filter in `RewardConstraintFacade` | M | T-1, T-2 | ✅ Done |
| T-4 | Create `NonOrgSummaryWriteProcessor` | M | T-3 | ✅ Done |
| T-5 | Wire `nonOrgSummaryWriteProcessor` into `BulkIssueService` processor chain at pos 13 | S | T-4 | ✅ Done |
| T-6 | Add compensation `try-finally` in `UserRewardUtils.issueRewardBulk()` and `issueReward()` | S | T-3 | ✅ Done |
| T-7 | Add `compensateFailedSummaryWrites()` delegation in `UserRewardFacade` | S | T-3 | ✅ Done |
| T-8 | Rename `pos13HandledConstraintIds` → `atomicWrittenConstraintIds` throughout | S | — | ✅ Done |
| T-9 | Fix stale "pos-12" references in comments → "RewardConstraintProcessor" | S | — | ✅ Done |
| T-10 | **JDBC SELECT in `writeNonOrgSummariesAtomically()`.** Replaced JPA read with `rewardIssueSummaryJdbcRepository.findExistingForNonOrgLevel()` — correct and future-safe (reads committed data, not JPA snapshot). Investigation confirmed this is NOT the root cause of T3/T4 failure; the actual cause is the 10 ms lock timeout (T-19). | M | — | ✅ Done |
| T-18 | **Multi-FIXED-window deduplication fix in `writeNonOrgSummariesAtomically()`.** `toAtomicIncrement: List` → `Map<Long, RewardIssueSummary>` keyed by `existing.getId()` (`putIfAbsent`). `toInsert: List` → `Map<String, RewardIssueSummary>` keyed by `level+"\|"+kpi+"\|"+issueDate.getTime()` (`putIfAbsent`). Flush via `.values()`. Decision rationale: typed String/Long keys give correct `equals()` without requiring `RewardIssueSummary.equals()` (which is identity-only — no `@EqualsAndHashCode`). Tests T7 + T8 added. See §5a GAP HIGH Resolution. | S | T-3 | ✅ Done |
| T-19 | **Separate reward-lock acquire timeout property.** `redis.reward.lock.maxWaitTime=${REDIS_REWARD_LOCK_MAX_WAIT_TIME:5000}` added to both property files. `RewardsApplicationConfiguration.getRedisRewardLockAcquireMaxWaitTime()` added. `RedisLockService.acquireRewardLock(key)` added using `redisLockRegistry` + new wait time. `NonOrgSummaryWriteProcessor` → `acquireRewardLock(lockKey)`. Customer-level `acquireLock(key)` (10 ms) unchanged. `@TestPropertySource` corrected: `redis.lock.maxWaitTime=5000` was overriding the customer lock (wrong), replaced with `redis.reward.lock.maxWaitTime=5000`. Decision rationale: customer lock must fail-fast (10 ms) while reward lock must queue (5000 ms) — two semantics, two properties. See §5a GAP HIGH Resolution. | S | T-4 | ✅ Done |
| T-20 | **Partial issuance compensation fix.** Add `getFailedQuantity()` = `quantityToBeProcessed - successCount` to `BulkRewardIssueContext.RewardIssueWrapper`. In `RewardConstraintFacade.compensateFailedSummaryWrites()`: (1) change filter from `!w.isAnySuccessful()` → `w.getFailedQuantity() > 0`; (2) build proportional `compensationRows` with delta per KPI (QUANTITY=failedQty, REDEMPTION_VALUE=rv×failedQty, TRANSACTION_COUNT=ONE only on total failure, POINTS=N/A). Pass `compensationRows` (not original rows) to `bulkAtomicDecrement`. Log `(failed={failedQty} of {totalQty})`. See §5a GAP HIGH. | M | T-3 | ⬜ Pending |
| T-21 | **Fix `VendorIssueProcessor` `failureCount` +1 bug (pre-existing).** `VendorIssueProcessor.java:163` — `setFailureCount(failureCount + 1)` in the `catch(Exception e)` path after `getVendorRewardDetails()` throws is a wrapper-level exception covering all qty; should be `setFailureCount(failureCount + quantityToBeProcessed)`. Independent fix — T-20 is immune (uses `getFailedQuantity()`), but T-21 restores the `failureCount` invariant for any future consumers. | S | — | ⬜ Pending |

### Testing

| # | Description | Size | Dependencies | Status |
|---|------------|------|--------------|--------|
| T-11 | Create `RewardConstraintConcurrencyIntegrationTest` (T1–T6) | M | T-4, T-5 | ✅ Done |
| T-12 | T2/T3/T4 validation — T2 (limit=1, 2 customers, exactly 1 passes), T3 (limit=5, 2 customers, no duplicate row), T4 (limit=10, 5 customers, consumed=5 exact). Root cause of T3/T4 failure was 10 ms lock timeout (T-19). T7 (dual-FIXED REWARD-level dedup) and T8 (dual-FIXED both levels) added for T-18 coverage. | M | T-18, T-19 | ✅ Done (T1–T8 all pass) |
| T-13 | Add compensation integration test: simulate downstream failure after write; assert CONSUMED decremented | M | T-6 | ⬜ Pending |

### Observability

| # | Description | Size | Status |
|---|------------|------|--------|
| T-14 | Add New Relic alert on `PARALLEL_CALL_LOCK_ACQUIRED = false` spike | S | ⬜ Pending |
| T-15 | Add `NonOrgSummaryWriteProcessor.process()` P99 alert (> 100ms threshold) | S | ⬜ Pending |

### Infra / DB

| # | Description | Size | Status |
|---|------------|------|--------|
| T-16 | Confirm covering index exists in production `TBL_REWARD_ISSUE_SUMMARY` | S | ⬜ Pre-deploy gate |
| T-17 | Confirm `redis.lock.ttl ≥ 500ms` and `redisLockAcquireMaxWaitTime` configured appropriately across all 8+ prod clusters | S | ⬜ Pre-deploy gate |

---

## Handoff Notes for test-plan-architect

**Tech detail doc location:** This document  
**Investigation doc:** `CAPJUN19_reward-limit-concurrency_analysis.md` (same directory)

### Critical B2B Flows to Test

- UC-1: REWARD-level limit enforced under high concurrency (50 threads, limit 10) → test type: integration + load
- UC-5: Mixed REWARD + CUSTOMER level constraints; REWARD-level binds first → test type: integration

### Critical B2C Flows to Test

- UC-6: Compensation decrement fires when downstream fails after pos-13 write → test type: integration (fault injection)
- UC-8: Re-evaluation inside lock catches limit-at-zero correctly → test type: integration (T2)
- UC-9: Single REWARD-level row produced even under concurrent first-issuance (no Race B) → test type: integration (T3)

### Regression Risks (must not break)

- Same-customer concurrent calls still serialised by `@CustomerLockable` — verify: T5 passes unchanged
- Serial REWARD-level limit enforcement — verify: T1 passes unchanged
- T6: pos-13 re-evaluates and rejects when limit reached under lock — verify unchanged

### Tenant Isolation Tests

- Reward issued by org A must not appear in org B's `TBL_REWARD_ISSUE_SUMMARY` (existing org-id filter, unchanged)
- Lock key `reward_constraint:orgA:rewardId` must not block `reward_constraint:orgB:rewardId` (different keys)

### Contract Tests

- `CONSTRAINT_EVALUATION_FAILED` (code 12005) returned at per-reward response level for lock contention AND re-evaluation failure — both paths must produce same status code at `r.getRewards().get(0).getStatus().getCode()`

### Suggested Test Emphasis

- **Integration (concurrency):** T1–T6 all pass; focus T2/T3/T4 which were failing. After T-10 fix, all 6 must pass.
- **Integration (compensation):** T-13 — simulate downstream failure; assert atomic decrement fires; CONSUMED returns to pre-request value.
- **Integration (ROLLING — positive):** Confirm ROLLING REWARD-level constraints ARE protected by Phase 1; assert Race A does not occur for ROLLING (same lock + `issualDate` atomic write path as FIXED window). T2 in the analysis doc covers this.
- **Unit:** `NonOrgSummaryWriteProcessor.process()` — lock acquired + evaluate passes + write succeeds; lock acquired + evaluate fails + lock released; no REWARD-level constraint present → no lock acquired.
- **Unit:** `RewardConstraintFacade.updateSummaries()` CQ9 filter — mix of REWARD-level (excluded) and ORG-level (included) rows; correct rows passed to `bulkSaveOrUpdate()`.
- **Integration (multi-FIXED-window — T-18 regression gate):** IT-T19 — FIXED/DAYS + FIXED/MONTHS + NO_LIMIT at REWARD/QUANTITY concurrent; assert exactly 1 row, consumed=N (not 2N), no duplicate row. IT-T21 — same at both REWARD and CUSTOMER level simultaneously.
- **Integration (NO_LIMIT + ROLLING):** IT-T20 — NO_LIMIT + ROLLING/DAYS at REWARD/QUANTITY concurrent; assert NO_LIMIT dependent row NOT created (1 row total, not 2).
