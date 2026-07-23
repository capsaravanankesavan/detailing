---
title: "issueReward Lock Removal — Complete Design v21 (Part 2 of 4)"
subtitle: "Solution Narrative continued — §B.5(cont), §B.6, §B.7, §B.8"
---

* * *
title: "issueReward Lock Removal — Complete Design v21 (Part 2 of 4)"
subtitle: "Solution Narrative continued — §B.5(cont), §B.6 LIMIT_VALUE live-refresh, §B.7 TTL, §B.8 Cron Jobs"
---

# sol-rewards-core: Removing the issueReward Redis Lock — Complete Design Reference (v21, self-contained) — PART 2 of 4

**⚠️ NOTE ON THIS SPLIT:** Continues from Part 1 (Problem + Solution through §B.5's collection definitions). This part covers: the rest of `getOrCreate()`/adaptive slot-count/the dedupe collection, §B.6 (LIMIT_VALUE live-refresh), §B.7 (TTL storage hygiene), and §B.8 (the two cron jobs, including the full corrected ETERNAL-sync mechanism that took v15→v21 to get right). Continues in Part 3 with §B.9 onward.

* * *

### B.5 (continued) — adaptive slot count, the dedupe collection

```java
/** Handles the case where alreadyConsumed > limitValue (a limit was LOWERED after consumption already
 *  happened). Never produces a negative per-slot consumed; the excess is simply not representable per-slot
 *  headroom — every slot ends up with consumed == its own budget (fully saturated), and the meta doc's
 *  allSlotsExhausted flag is set true immediately, so the very next issuance correctly denies without
 *  needing to wait for the next rebalance tick. */
private List<BigDecimal> distributeConsumedAcrossBudgets(BigDecimal alreadyConsumed, List<BigDecimal> slotBudgets) {
    BigDecimal totalBudget = slotBudgets.stream().reduce(BigDecimal.ZERO, BigDecimal::add);
    if (alreadyConsumed.compareTo(totalBudget) >= 0) {
        return slotBudgets; // over-limit case: seed every slot fully saturated (consumed == its own budget), never negative headroom
    }
    BigDecimal ratio = totalBudget.compareTo(BigDecimal.ZERO) == 0 ? BigDecimal.ZERO : alreadyConsumed.divide(totalBudget, MathContext.DECIMAL64);
    return slotBudgets.stream().map(b -> b.multiply(ratio)).collect(toList()); // proportional to each slot's OWN budget share
}

/** Adaptive slot count. A fixed global slot.count=16 wastes retry-to-next-slot cycles on a LOW-limit reward
 *  (e.g. limitValue=100 -> even split of 6.25/slot means frequent, needless probing even though the reward's
 *  TOTAL headroom is fine) — mathematically not broken (BigDecimal handles fractional sub-budgets natively),
 *  but practically wasteful. Computed ONCE, at meta-doc creation, never recomputed mid-cycle. */
public int computeEffectiveSlotCount(BigDecimal limitValue, String windowCycleKey) {
    int maxSlots = configuredMaxSlotCount; // existing config, default 16, UNCHANGED
    // ETERNAL counters have NO cycle-rollover boundary to self-heal at — a windowed counter that outgrows its
    // slot count gets a fresh, correctly-sized count at the NEXT windowCycleKey (new meta doc); an ETERNAL
    // counter's meta doc lives for the reward's entire lifetime, so if it were sized small at creation and
    // LIMIT_VALUE later grows large, it would stay stuck on that small count FOREVER — silently reintroducing
    // the exact hot-document contention this whole design exists to eliminate. FIX: ETERNAL counters skip
    // adaptive sizing entirely and always get the full configuredMaxSlotCount.
    if ("ETERNAL".equals(windowCycleKey)) {
        return maxSlots;
    }
    int minUnitsPerSlot = this.minUnitsPerSlot; // config, default 10
    if (limitValue == null || limitValue.compareTo(BigDecimal.ZERO) <= 0) {
        return 1; // defensive floor for zero/negative/absent limit — never 0, never negative
    }
    int proportional = limitValue.divide(BigDecimal.valueOf(minUnitsPerSlot), 0, RoundingMode.FLOOR).intValue();
    return Math.min(maxSlots, Math.max(1, proportional)); // floors at 1, caps at the existing max — never 0, never over the ceiling
}
```
If two concurrent requests both try to create the SAME brand-new meta doc, Mongo's atomic upsert guarantees only one actually creates it; the other's attempt is a no-op that reads what the winner created.

**Adaptive slot count, worked out:** `effectiveSlotCount = min(configuredMaxSlotCount, max(1, floor(limitValue / minUnitsPerSlot)))`. For `limitValue=100` with `minUnitsPerSlot=10`: `floor(100/10)=10`, capped at 16 → **10 slots**. For `limitValue=100000`: clamped to the ceiling → **16 slots**. For `limitValue=5`: floored to **1 slot** (never zero). Tests #77-79 cover exactly these three cases.

**What does the CRON do to this doc (never creation — always ongoing maintenance)?**
1. **Rebalances `slotBudgets`** — redistributes the fixed total proportionally to observed consumption (closed-form formula, §B.6).
2. **Detects a `LIMIT_VALUE` change** and updates + re-rebalances against the new total (§B.6).
3. **Marks `status=EXPIRED`** once the cycle has rolled over (`now > windowEndAt`) — never pre-creates the NEXT cycle's meta doc, that stays lazy too.

**Advisory `allSlotsExhausted` flag**: set by the cron (not per-request), when every slot's consumption has caught up to its budget. The request path MAY check this first as a fast-fail hint — purely an optimization. If stale/wrong in either direction, correctness is unaffected: the per-slot atomic conditional write remains the sole authoritative gate.

#### 3. `rewardCounterDedupe` — the retry-safety guard (NOT an audit log)

```java
@Document(collection = "rewardCounterDedupe")
public class RewardCounterDedupe {
    @Id
    private String id; // "<orgId>:<rewardId>:<requestId>:<attemptSeq>"
    private DedupeStatus status; // PENDING | COMMITTED
    private Integer landedSlot;
    private Date createdOn;
    private Date updatedAt;
}
```

**What is this FOR?** NOT about preventing a customer from issuing the same reward twice — that's the separate, pre-existing `IssuedTransaction`/idempotency-check's job. This exists to solve one narrow problem: **today's Redis lock accidentally gives retry-safety "for free"** by serializing all writers to a reward — a client retry naturally queues behind the original and (by construction of the lock) doesn't double-apply. Removing the lock removes this free protection: two network-level retries of the exact same request could BOTH succeed at independent, unaware-of-each-other atomic increments, silently counting one real issuance as two.

**Flow (two-phase, closes the gap correctly):**
1. `insertOne({_id, status:"PENDING", createdOn:now})`. Duplicate key on retry → branch: `COMMITTED` → truly applied, skip. `PENDING` and fresh (within `pendingTimeoutMs`) → concurrent in-flight duplicate, deny/backoff. `PENDING` and stale (past timeout) → an orphan from a crash between reserve and increment — **re-attempt the increment now**.
2. Call `increment()` (§B.10).
3. On success: `updateOne({_id,status:"PENDING"}, {$set:{status:"COMMITTED", landedSlot, updatedAt:now}})`.
4. On genuine deny: leave `PENDING` — a retry within timeout is correctly re-denied, not silently skipped.
5. A GC sweep (run by the MySQL-sync cron job, §B.8) deletes `PENDING` markers older than `pendingTimeoutMs × safetyFactor` — pure garbage collection, not a correctness mechanism.

Deliberately NOT the audit-ledger concept considered and explicitly rejected earlier in this design's history — no delta, no business meaning, purely an existence/state check.

### B.6 LIMIT_VALUE live-refresh — handling a limit change mid-cycle

**Why this needed fixing**: `RewardConstraintFacade.getRewardLevelRestrictions()` (`.java:832-840`) issues a fresh JPA SELECT on `TBL_REWARD_CONSTRAINT` per issuance request; no caching exists anywhere on this path. **Today, an admin's `LIMIT_VALUE` edit takes effect on the very next issuance.** An earlier draft had `limitValue` frozen once at counter-creation until the next cycle boundary — a genuine regression, not an accepted gap.

**The fix — `refreshLimitIfChanged`, called by the fast cron job on every tick, for every counter:**
```java
public void refreshLimitIfChanged(RewardCounterCycleMeta meta, Supplier<BigDecimal> currentLimitSupplier,
                                     List<BigDecimal> observedConsumedPerSlot) {
    BigDecimal currentLimit = currentLimitSupplier.get();
    BigDecimal effectiveLimit = currentLimit.compareTo(meta.getLimitValue()) != 0 ? currentLimit : meta.getLimitValue();
    if (currentLimit.compareTo(meta.getLimitValue()) != 0) {
        metricsService.incrementCounter("reward_counter_limit_value_refresh_count",
            "scope", "ETERNAL".equals(meta.getWindowCycleKey()) ? "eternal" : "windowed");
    }
    // Compute the new slotBudgets (+ allSlotsExhausted) IN-MEMORY first, THEN write everything in ONE atomic
    // updateFirst call — removes the window where limitValue is updated but slotBudgets is stale.
    BigDecimal C = observedConsumedPerSlot.stream().reduce(BigDecimal.ZERO, BigDecimal::add);
    List<BigDecimal> newBudgets = C.compareTo(BigDecimal.ZERO) == 0
        ? evenSplitBudgets(effectiveLimit, meta.getSlotCount(), meta.getId().hashCode())
        : observedConsumedPerSlot.stream()
            .map(c -> c.multiply(effectiveLimit.divide(C, MathContext.DECIMAL64)))
            .collect(toList());
    boolean allExhausted = IntStream.range(0, newBudgets.size())
        .allMatch(i -> observedConsumedPerSlot.get(i).compareTo(newBudgets.get(i)) >= 0);
    mongoTemplate.updateFirst(query(where("_id").is(meta.getId())),
        new Update().set("limitValue", effectiveLimit).set("slotBudgets", newBudgets)
                     .set("allSlotsExhausted", allExhausted).set("lastRebalancedAt", new Date()),
        RewardCounterCycleMeta.class);
}
```
The rebalance formula's correctness proof (proven safe at `C<L`, `C==L`, `C>L` — no overshoot possible in any case, including saturation) holds for ANY limit value `L`, not just the one at creation time.

**Bound, not instant**: propagation delay is bounded (~1 cron tick, ~5s default), not instant, and not "next cycle/never" (an earlier regression). For a limit *decrease*, there's a small, bounded window where a request could still be admitted against the stale, higher budget — flagged to communicate to product as "propagates within ~1 tick," not instant parity with legacy (Risks §D.3).

**Cron scaling at high reward-count + event-driven propagation.** At scale (e.g. 100 orgs × 500 rewards × 2 window-limits each ≈ 100,000 potentially-active meta docs), a naive "one `TBL_REWARD_CONSTRAINT` read per meta doc, per tick" loop would not fit in a 5s budget. **Two changes:**
1. **Batch the read**: fetch all currently-relevant `LIMIT_VALUE`s for an org-shard in ONE bulk query, process docs in parallel within the tick.
2. **Event-driven fast path, cron as backstop**: when a `LIMIT_VALUE` edit succeeds, publish an event to a new internal RabbitMQ queue. A listener finds and immediately refreshes matching meta doc(s) — propagation is typically sub-second. The cron's own check is KEPT but demoted to a slower-cadence backstop (e.g. every 60s) for defense-in-depth (RMQ delivery is not unconditionally guaranteed). This changes the Risk framing: the "~1 tick" bound is now the WORST case (RMQ message lost/delayed), not the typical case.

### B.7 Storage hygiene — the TTL fields

Both `rewardCounterSlot` and `rewardCounterCycleMeta` get an `expireAt` field, computed ONCE at creation as `windowEndAt + ~90 days` — EXCEPT `ETERNAL` (independent NO_LIMIT) counters, which get `expireAt = null` and must NEVER expire.

⚠️ **Why this needed extra care**: this repo's own index-provisioning path, `MongoDbInitializer.execute()`'s `createIndexes(...)` call, is **DEAD CODE** (explicitly commented out), and no `spring.data.mongodb.auto-index-creation` property exists anywhere. Declaring `@Indexed(expireAfterSeconds=...)` alone provisions NOTHING. **Required fix**: either re-enable `MongoDbInitializer.createIndexes(...)` for these new collections, or an explicit ops runbook step, as part of the pre-cutover checklist. Flagged as a **"silent failure" risk class** — a broken TTL setup quietly deletes documents with zero application-visible error. Mitigated by a dedicated metric (`reward_counter_ttl_expired_docs_count`) and an integration test verifying the index actually exists (test #66).

**The additive-only index-creator gap.** Spring Data MongoDB's annotation-driven index creator (once enabled) is **additive-only** — it creates a NEW index matching a current annotation, but has **no mechanism to detect/drop a STALE index**. For the INITIAL go-live, this is not a blocker (brand-new collections, nothing stale to reconcile). But if the TTL field's index definition is ever changed AFTER rollout, annotation-driven creation alone will NOT drop the old index — a dedicated reconciliation step or manual ops-runbook drop will be required THEN. Documented as a pre-existing, platform-wide Mongo index-management limitation, **out of scope for this design to solve** — affects every collection using `@Indexed` in this service, not specific to our two new collections.

### B.8 The cron jobs — how many, what do they do, why two, and the bugs found fixing them

**Important context**: this codebase has **zero** in-process `@Scheduled` jobs today — the only existing "cron" concept is an entirely different EXTERNAL mechanism (a Capillary scheduler service over RabbitMQ, used today for reward-expiry reminder emails only). So the jobs below are the **first-ever** in-process scheduled jobs in this service. `@EnableScheduling` is already declared (unused until now) — no new Spring annotation needed, but the thread-pool infrastructure genuinely is new.

There are **two** jobs, deliberately split because they have very different urgency:

**Job 1 — `RewardCounterRebalanceJob`.** Fast cadence (5s default). Per tick: `LIMIT_VALUE`-change detection + rebalance, rollover/expiry detection.

**Job 2 — `RewardCounterMySqlSyncJob`.** Slower cadence (45s default). Per tick: mirrors each counter's current total into `TBL_REWARD_ISSUE_SUMMARY`, ONLY for counters overdue per `lastSyncedAt`. Also runs the dedupe-marker GC sweep. This job's staleness tolerance is much looser (GET-API reporting freshness, not enforcement correctness).

**Why split at all?** A slow MySQL-sync tick sharing one schedule with the fast rebalance tick would delay the more time-sensitive job. Splitting means a slow sync never holds up a fast rebalance.

**Both jobs iterate every "org-shard."** `OrgMongoDBFactory.getDb()` resolves the Mongo database **per operation** from MDC context and **throws `ShardContextNotSetException`** if not set — a genuine per-org-shard access pattern, not a single global database. A background scheduled job must explicitly loop over every known org-shard, set MDC, do its work, clear MDC, move to the next.

**The ETERNAL-sync mechanism — the part that took v15 through v21 to get right.**

An earlier approach (v15-v18) tried a "sentinel row" model — ONE fixed MySQL row holding the cumulative total for an ETERNAL counter. This went through 3 consecutive arbiter REFINE cycles fixing a nonexistent repository method, a migration-vs-cron race, a factually-wrong `ON DUPLICATE KEY UPDATE` claim against the real DDL (the table has NO unique key on the relevant columns — PK is `(ID,ORG_ID)` with auto-increment `ID`), and a cross-writer sentinel-date-agreement problem.

**The v19 breakthrough**: tracing every caller of `Utils.getDateWithoutTimestampInSpecifiedZone` confirmed that for an independent NO_LIMIT constraint, legacy's `TBL_REWARD_ISSUE_SUMMARY` is a **genuinely daily-bucketed table** — BOTH write paths (`updateSummaries`→`buildUniqueSummaries` and `writeNonOrgSummariesAtomically`→`resolveNonOrgSummaryDate`, `RewardConstraintFacade.java:629,704-710`) resolve the write-bucket `ISSUE_DATE` to today's midnight and INSERT a new row whenever no row matches today exactly — mirroring ROLLING's per-day bucketing precisely. The READ side (`LevelService.java:101,123-138` → `RewardIssueSummaryRepository.fetchKPIConsumedValueForRewardId`) SUMs `CONSUMED` across every matching row with NO date filter at all. The sentinel-row approach had been fighting this schema instead of using it.

**The corrected mechanism (v19 → v20 → v21, all real bugs fixed along the way):**

```java
/** Windowed meta docs (FIXED/ROLLING) still map 1:1 onto ONE MySQL row via overwriteMySqlSummary() — unchanged.
 *  An ETERNAL meta doc writes to a single-row-PER-DAY bucket — TODAY's date, the EXACT SAME bucket key legacy's
 *  own write path already computes — incrementing by the delta consumed SINCE THE LAST sync tick. Historical
 *  pre-cutover rows are left completely untouched — legacy's own all-time SUM already includes them
 *  automatically; there is NOTHING to consolidate, migrate, or reconcile.
 *
 *  Fix #1 (real bulkAtomicIncrement(List<RewardIssueSummary>) only UPDATEs an EXISTING row by primary key; it
 *  cannot create today's row on the first tick of a new day) — mirrors writeNonOrgSummariesAtomically's own
 *  existing find-or-create pattern instead of assuming a blind increment suffices.
 *  Fix #2 (org-timezone was never resolved in the cron's context) — resolves per-ORG (a shard hosts multiple
 *  orgs) via the SAME resolveOrgZoneId(orgId) the live write path already uses.
 *  Fix #3 (the two writes, MySQL + Mongo ledger, were not ordered safely) — the Mongo ledger update happens
 *  BEFORE the MySQL write. A crash between them means the NEXT tick recomputes off the already-advanced
 *  ledger — this tick's delta is silently LOST (self-heals fully on the next tick), never DOUBLE-counted.
 *  Losing a delta transiently is the strictly safer failure direction versus double-applying it permanently. */
private void overwriteEternalMySqlSummary(RewardCounterCycleMeta meta, BigDecimal cumulativeTotal, ZoneId orgZoneId) {
    BigDecimal lastSynced = meta.getLastSyncedCumulativeTotal(); // NEVER null post-getOrCreate — seeded, not null-defaulted
    BigDecimal deltaSinceLastSync = cumulativeTotal.subtract(lastSynced);
    int cmp = deltaSinceLastSync.compareTo(BigDecimal.ZERO);
    if (cmp == 0) return; // nothing new since last tick — skip the write entirely
    if (cmp < 0) {
        // A negative delta means Mongo's total DECREASED since last sync (a compensation/revoke landed).
        // Never pass a negative delta to an INCREMENT — route through the existing floor-at-zero DECREMENT path.
        rewardIssueSummaryJdbcRepository.decrementConsumedFloorZero(
            meta.getOrgId(), meta.getRewardId(), meta.getKpi(), Level.REWARD,
            Utils.getDateWithoutTimestampInSpecifiedZone(new Date(), orgZoneId), deltaSinceLastSync.abs());
        mongoTemplate.updateFirst(query(where("_id").is(meta.getId())),
            new Update().set("lastSyncedCumulativeTotal", cumulativeTotal).set("lastSyncedAt", new Date()),
            RewardCounterCycleMeta.class);
        return;
    }
    Date todayIssueDate = Utils.getDateWithoutTimestampInSpecifiedZone(new Date(), orgZoneId);
    // Persist the Mongo ledger FIRST — a crash after this line but before the MySQL write means this tick's
    // delta is transiently LOST (safe direction), never double-applied.
    mongoTemplate.updateFirst(query(where("_id").is(meta.getId())),
        new Update().set("lastSyncedCumulativeTotal", cumulativeTotal).set("lastSyncedAt", new Date()),
        RewardCounterCycleMeta.class);
    // Fix (QA CRITICAL finding — the find-or-create sequence itself had a race: two near-simultaneous writers
    // (two sync-tick retries, or a sync tick racing a live legacy write) could both observe "no row for today"
    // and both attempt an INSERT, producing two rows instead of one consolidated row). FIX: wrap the INSERT
    // attempt in a duplicate-key catch, and on collision, fall back to the UPDATE path against the row the
    // OTHER writer just created — the SAME reconciliation shape writeNonOrgSummariesAtomically's own
    // bulkSaveOrUpdate already uses for its own concurrent-insert race (RewardConstraintFacade.java:412-414).
    Optional<RewardIssueSummary> existingToday = rewardIssueSummaryJdbcRepository.findExistingForNonOrgLevel(
        meta.getOrgId(), meta.getRewardId(), /*userId=*/null, todayIssueDate, meta.getKpi(), Level.REWARD);
    if (existingToday.isPresent()) {
        RewardIssueSummary row = existingToday.get();
        row.setConsumed(row.getConsumed().add(deltaSinceLastSync));
        rewardIssueSummaryJdbcRepository.bulkAtomicIncrement(Collections.singletonList(row));
    } else {
        RewardIssueSummary newRow = new RewardIssueSummary();
        newRow.setOrgId(meta.getOrgId()); newRow.setRewardId(meta.getRewardId()); newRow.setKpi(meta.getKpi());
        newRow.setLevel(Level.REWARD); newRow.setUserId(null); newRow.setIssueDate(todayIssueDate);
        newRow.setConsumed(deltaSinceLastSync);
        try {
            rewardIssueSummaryJdbcRepository.save(newRow); // INSERT branch — first tick of a new calendar day
        } catch (DuplicateKeyException e) {
            // Another writer inserted today's row between our findExistingForNonOrgLevel() call and this
            // save() — re-fetch and fall back to the UPDATE branch against whatever row now exists. No delta
            // is lost: the retry's own delta still gets applied, just via UPDATE instead of INSERT.
            RewardIssueSummary raceWinner = rewardIssueSummaryJdbcRepository.findExistingForNonOrgLevel(
                meta.getOrgId(), meta.getRewardId(), null, todayIssueDate, meta.getKpi(), Level.REWARD)
                .orElseThrow(() -> new IllegalStateException("Duplicate-key collision but no row found on retry — unexpected"));
            raceWinner.setConsumed(raceWinner.getConsumed().add(deltaSinceLastSync));
            rewardIssueSummaryJdbcRepository.bulkAtomicIncrement(Collections.singletonList(raceWinner));
        }
    }
}
```

**Note on the DuplicateKeyException catch**: `TBL_REWARD_ISSUE_SUMMARY`'s existing composite index (`idx_rewardIssue_summary`) is non-unique — a genuine `DuplicateKeyException` can only come from the table's real `PRIMARY KEY (ID,ORG_ID)` (auto-increment, never colliding on a normal INSERT) or a different unique constraint if one exists. In practice, the ONLY realistic way two writers both reach the INSERT branch for the same key is if this design's own single-leader Redis lock on the sync job is somehow bypassed — so this catch-and-reconcile is defense-in-depth for a legacy-write-vs-sync-tick race specifically, not an expected steady-state path.

**Org-zone resolution detail**: `RewardCounterMySqlSyncJob` `@Autowired`s `RewardConstraintFacade` (exposes `resolveOrgZoneId(orgId)`, the SAME resolver the live write path already uses, itself calling an external `@Cacheable` HTTP call to intouch keyed per-orgId). The job resolves this **once per org per tick** (cached within the tick, since a shard can host many meta docs for the same org), and catches `ExternalServiceException` **per meta doc**, not per shard — so one org's intouch failure doesn't abort sync for every other org sharing that Mongo org-shard.

**First-tick double-count fix detail**: `lastSyncedCumulativeTotal` must NEVER start `null` for an ETERNAL meta doc seeded with nonzero `alreadyConsumed` — if it did, the FIRST sync tick's delta would equal the FULL cumulative total, duplicating the already-untouched historical rows on top of them. Fixed: `getOrCreate()` initializes `lastSyncedCumulativeTotal = alreadyConsumed` at creation time (not `null`) for ETERNAL meta docs.

**Why this is correct, not just simpler:** the read side is an unfiltered SUM — it does not care how many rows exist or when they were written. Total = (all pre-existing historical rows, untouched) + (all post-cutover daily increments, correctly excluding the historical amount already counted once) = exactly the true cumulative total, with ZERO migration step and ZERO cross-writer date-agreement problem.

**This ELIMINATES**: the sentinel-row single-fixed-date model and its cross-writer date-agreement problem; the UPDATE-first/INSERT-if-zero-rows sequence built around a nonexistent unique constraint; the ENTIRE migration/cutover consolidation requirement and its cron-race ordering fix.

**Fields required on `RewardCounterCycleMeta`**: `lastSyncedCumulativeTotal` (`BigDecimal`, seeded to `alreadyConsumed` at creation — NEVER left `null` for a nonzero-seeded ETERNAL doc) — populated and consulted ONLY when `windowCycleKey.equals("ETERNAL")`; unused for windowed docs.

*(Continued in Part 3 of 4 — the two full cron job class bodies with leader-election, §B.9 RewardCounterSlotService, §B.10 Multi-counter atomicity, §B.11 Metrics.)*
