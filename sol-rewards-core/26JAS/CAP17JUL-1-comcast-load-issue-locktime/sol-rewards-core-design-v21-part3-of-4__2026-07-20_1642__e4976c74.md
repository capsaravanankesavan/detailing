---
title: "issueReward Lock Removal — Complete Design v21 (Part 3 of 4)"
subtitle: "§B.8 job bodies, §B.9 SlotService, §B.10 atomicity, §B.11 metrics, §B.12 migration"
---

* * *
title: "issueReward Lock Removal — Complete Design v21 (Part 3 of 4)"
subtitle: "§B.8 cron job bodies (leader-election, config), §B.9 RewardCounterSlotService, §B.10 Multi-counter atomicity, §B.11 Metrics, §B.12 Migration"
---

# sol-rewards-core: Removing the issueReward Redis Lock — Complete Design Reference (v21, self-contained) — PART 3 of 4

**⚠️ NOTE ON THIS SPLIT:** Continues from Part 2 (§B.8's ETERNAL-sync mechanism). This part covers the three real bugs found and fixed in the cron leader-election/scheduling plumbing, the full corrected job class bodies, §B.9 (`RewardCounterSlotService`), §B.10 (multi-counter atomicity, `incrementAllOrRollback`), §B.11 (metrics — including a critical `MetricsService` API gap that had to be fixed), and §B.12 (the full migration/cutover plan). Continues in Part 4 with Parts C–F (scaling, risks, code-paths table, criticality table, and the complete 118-test strategy).

* * *

### B.8 (continued) — three real bugs found and fixed in the cron plumbing

**Bug 1 [CRITICAL] — leader-election API mismatch.** An earlier draft called `redisLockService.acquireLock(key, 0)` and negated it (`!acquireLock(...)`) as if it returned a boolean. The REAL `RedisLockService.acquireLock(String,Long)` (`RedisLockService.java:104-125`) returns a `Lock` object and **THROWS `MarvelException`** on failure — it never returns a falsy value. As written, this doesn't compile; naively removing the `!` would make every non-leader instance throw on every single tick instead of silently skipping. **Fix — a new non-throwing method:**
```java
/** Non-blocking leader-election helper — tries once, never throws, returns empty on failure. */
public Optional<Lock> tryAcquireNonBlocking(String key, RedisLockRegistry lockRegistry) {
    try {
        Lock lock = lockRegistry.obtain(key);
        boolean acquired = lock.tryLock(0, TimeUnit.MILLISECONDS);
        return acquired ? Optional.of(lock) : Optional.empty();
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        return Optional.empty();
    }
}
```

**Bug 2 [MAJOR] — reused an unrelated 10-minute lock.** Both jobs were hardwired to the existing `reconcileLock` registry, built with a fixed **600,000ms (10-minute) TTL** sized for a completely different, pre-existing reconciliation job. Reusing it means a crashed leader strands the lock for up to 10 minutes (~120 missed rebalance ticks) — directly breaking the "~1 tick" propagation bound §B.6 promises. **Fix — two NEW dedicated registries, each sized to its OWN cadence:**
```java
@Bean
public RedisLockRegistry rewardCounterRebalanceLockRegistry() {
    return new RedisLockRegistry(redisConnectionFactory, "reward-counter-cron-leader:rebalance", 15_000L); // 3x the 5s cadence
}
@Bean
public RedisLockRegistry rewardCounterMySqlSyncLockRegistry() {
    return new RedisLockRegistry(redisConnectionFactory, "reward-counter-cron-leader:mysqlsync", 135_000L); // 3x the 45s cadence
}
```

**Bug 3 [MAJOR] — missing scheduler thread-pool sizing.** Verified: no `TaskScheduler` bean, no `spring.task.scheduling.pool.size` override exists anywhere — Spring Boot's default is a **single-threaded** scheduler pool. With two independently-`@Scheduled` beans sharing that one thread, a slow rebalance sweep would starve the sync job entirely — silently defeating the entire rationale for splitting them. **Fix — REQUIRED config, not a deferred risk item:**
```properties
spring.task.scheduling.pool.size=2
```

**Final, corrected job implementations:**
```java
@Component
public class RewardCounterRebalanceJob {
    @Autowired private OrgMongoDataSourceManager orgMongoDataSourceManager;
    @Autowired private RewardCounterCycleMetaService cycleMetaService;
    @Autowired private RedisLockService redisLockService;
    @Autowired @Qualifier("rewardCounterRebalanceLockRegistry") private RedisLockRegistry rebalanceLockRegistry;

    @Scheduled(fixedDelayString = "${reward.counter.rebalance.fixedDelayMs:5000}")
    public void run() {
        long sweepStart = System.currentTimeMillis();
        Optional<Lock> lock = redisLockService.tryAcquireNonBlocking("reward-counter-cron-leader:rebalance", rebalanceLockRegistry);
        if (lock.isEmpty()) return; // not leader this tick — silently skip, no exception
        try {
            for (String shardKey : orgMongoDataSourceManager.getAllShardKeys()) {
                MDC.put(OrgMongoDBFactory.SHARD_KEY, shardKey);
                try {
                    List<RewardCounterCycleMeta> activeMetaDocs = findActiveOrEternalMetaDocs();
                    // The batching happens BEFORE the per-doc loop, not inside it: a Mongo org-shard can
                    // legitimately host MULTIPLE orgIds (OrgMongoDBFactory exposes both get(orgId) and
                    // getByKey(shardKey) accessors), and the existing repository method
                    // (findAllByOrgIdAndEnabledAndEndDateAndRewardIdIn) takes a SINGLE orgId, not a list. The
                    // honest, achievable-today claim is "one bulk query PER ORG present in this shard" —
                    // grouping this shard's active meta docs by orgId first, then one IN-list query per org.
                    Map<Long, List<RewardCounterCycleMeta>> byOrg = activeMetaDocs.stream()
                        .collect(Collectors.groupingBy(RewardCounterCycleMeta::getOrgId));
                    Map<String, BigDecimal> limitsByRewardKpi = new HashMap<>(); // key: "rewardId:kpi"
                    for (Map.Entry<Long, List<RewardCounterCycleMeta>> orgEntry : byOrg.entrySet()) {
                        List<Long> rewardIds = orgEntry.getValue().stream().map(RewardCounterCycleMeta::getRewardId).distinct().collect(toList());
                        // ONE bulk query per ORG (not per meta doc) — reuses the EXISTING repository method:
                        List<RewardConstraint> constraints = rewardConstraintRepository
                            .findAllByOrgIdAndEnabledAndEndDateAndRewardIdIn(orgEntry.getKey(), true, today(), rewardIds);
                        for (RewardConstraint c : constraints) {
                            String key = c.getRewardId() + ":" + c.getKpi();
                            limitsByRewardKpi.merge(key, BigDecimal.valueOf(c.getLimitValue()), BigDecimal::min); // MIN across co-resident constraints
                        }
                    }
                    for (RewardCounterCycleMeta meta : activeMetaDocs) {
                        List<BigDecimal> perSlotConsumed = readSlotConsumedValues(meta);
                        String key = meta.getRewardId() + ":" + meta.getKpi();
                        cycleMetaService.refreshLimitIfChanged(meta, () -> limitsByRewardKpi.get(key), perSlotConsumed); // reads the PRE-FETCHED map, zero extra queries per doc
                        if (!"ETERNAL".equals(meta.getWindowCycleKey()) && cycleHasRolledOver(meta)) {
                            markExpired(meta); // never pre-creates the next cycle's doc — stays lazy
                        }
                    }
                } catch (Exception e) {
                    log.error("Rebalance sweep failed for org-shard {}", shardKey, e);
                    metricsService.incrementCounter("reward_counter_cron_org_shard_sweep_failure_count", "job", "rebalance");
                } finally {
                    MDC.remove(OrgMongoDBFactory.SHARD_KEY); // reset before next shard — no context bleed
                }
            }
        } finally {
            lock.get().unlock();
            long sweepDurationMs = System.currentTimeMillis() - sweepStart;
            metricsService.recordGauge("reward_counter_rebalance_sweep_duration_ms", sweepDurationMs);
            if (sweepDurationMs > rebalanceFixedDelayMs) {
                log.warn("Rebalance sweep ({} ms) exceeded its own cadence ({} ms) — propagation bound may be stretched",
                    sweepDurationMs, rebalanceFixedDelayMs);
            }
        }
    }
}

@Component
public class RewardCounterMySqlSyncJob {
    @Autowired private OrgMongoDataSourceManager orgMongoDataSourceManager;
    @Autowired private RedisLockService redisLockService;
    @Autowired @Qualifier("rewardCounterMySqlSyncLockRegistry") private RedisLockRegistry mySqlSyncLockRegistry;
    @Autowired private RewardConstraintFacade rewardConstraintFacade; // exposes resolveOrgZoneId(orgId), the SAME resolver the live write path already uses
    @Autowired private RewardCounterCycleMetaService cycleMetaService;

    @Scheduled(fixedDelayString = "${reward.counter.mysqlSync.fixedDelayMs:45000}")
    public void run() {
        Optional<Lock> lock = redisLockService.tryAcquireNonBlocking("reward-counter-cron-leader:mysqlsync", mySqlSyncLockRegistry);
        if (lock.isEmpty()) return;
        try {
            for (String shardKey : orgMongoDataSourceManager.getAllShardKeys()) {
                MDC.put(OrgMongoDBFactory.SHARD_KEY, shardKey);
                try {
                    Date staleThreshold = new Date(System.currentTimeMillis() - syncStalenessThresholdMs);
                    Map<Long, ZoneId> zoneCache = new HashMap<>(); // resolve org-zone ONCE per org per tick,
                                                                     // not once per meta doc — a shard can host many
                                                                     // meta docs for the same org
                    for (RewardCounterCycleMeta meta : findMetaDocsOverdueForSync(staleThreshold)) { // only overdue docs, not all
                        try {
                            List<BigDecimal> perSlotConsumed = readSlotConsumedValues(meta);
                            BigDecimal cumulativeTotal = perSlotConsumed.stream().reduce(BigDecimal.ZERO, BigDecimal::add);
                            if ("ETERNAL".equals(meta.getWindowCycleKey())) {
                                ZoneId orgZoneId = zoneCache.computeIfAbsent(meta.getOrgId(), rewardConstraintFacade::resolveOrgZoneId);
                                cycleMetaService.overwriteEternalMySqlSummary(meta, cumulativeTotal, orgZoneId); // §B.8's corrected method (Part 2)
                            } else {
                                overwriteMySqlSummary(meta, perSlotConsumed); // windowed path — unchanged, 1:1 row mapping
                            }
                            markSynced(meta); // sets lastSyncedAt = now (lastSyncedCumulativeTotal is set INSIDE overwriteEternalMySqlSummary itself)
                            metricsService.incrementCounter("reward_counter_mysql_sync_docs_synced_count");
                        } catch (ExternalServiceException e) {
                            // Caught PER META DOC, not per shard — one org's intouch zone-resolution
                            // failure must not abort sync for every OTHER org sharing this Mongo org-shard.
                            log.error("MySQL sync failed for org={} reward={} (org-zone resolution or write failure)",
                                meta.getOrgId(), meta.getRewardId(), e);
                            metricsService.incrementCounter("reward_counter_mysql_sync_doc_failure_count");
                        }
                    }
                    sweepOrphanedPendingDedupeMarkers(); // paired with rollback's "forced final sweep", §B.12
                } catch (Exception e) {
                    log.error("MySQL sync sweep failed for org-shard {}", shardKey, e);
                    metricsService.incrementCounter("reward_counter_cron_org_shard_sweep_failure_count", "job", "mysqlsync");
                } finally {
                    MDC.remove(OrgMongoDBFactory.SHARD_KEY);
                }
            }
        } finally {
            lock.get().unlock();
        }
    }
}
```

### B.9 `RewardCounterSlotService` — the write/compensation logic per KPI

Provides `increment(orgId, rewardId, kpi, windowCycleKey, delta, requestId)` — the atomic conditional write against a chosen slot (bounded random retry across `slot.maxRetryAttempts` on local exhaustion), and `compensate(...)` — the KPI-aware reversal logic invoked on downstream failure or explicit revoke, dispatching per KPI type (§B.3):

- **QUANTITY / POINTS**: straightforward proportional reversal of the originally-applied delta.
- **REDEMPTION_VALUE**: requires the `CompensationReason` discriminator — a `REVOKE` MUST supply a freshly DB-fetched per-unit value (never the stale in-memory delta); throws `IllegalStateException` if a `REVOKE` arrives with a null `originalDelta`, closing a real silent-miscount risk.
- **TRANSACTION_COUNT**: the reversal is conditional — `+1` reversal ONLY if `failedQty == totalQty` (a full failure), else `0` — a naive proportional reversal would be WRONG for this KPI specifically (mirrors `RewardConstraintFacade.java:754-755` exactly).

The slot write is a single atomic conditional `findOneAndUpdate` (no read-then-write, no application lock) — `consumed += delta` only if `consumed <= subBudget - delta`. On local slot exhaustion, retries against a bounded, randomly-chosen alternate slot (not the full slot set) — self-balances skew across many rewards without needing a separate rebalancing signal on the hot path itself (the cron's rebalance, §B.6, handles the slower-moving redistribution).

### B.10 Multi-counter atomicity — `incrementAllOrRollback`

**The problem this solves**: a single issuance can touch MULTIPLE distinct counters simultaneously (§B.2 — e.g. a reward with both a "max 100/day" AND a "max 1000/month" constraint on the same KPI). If the daily counter's increment SUCCEEDS but the monthly counter's increment then DENIES (over its own limit), the daily increment must be ROLLED BACK — otherwise that reward's daily counter silently over-counts relative to what was actually issued (the request as a whole was denied).

**Why not a Mongo multi-document ACID transaction instead?** `MongoTransactionManager` IS wired in this codebase's config (`MongoClientConfiguration.java:84-87`) but has ZERO actual callers anywhere — confirmed via grep. Introducing the FIRST real usage of Mongo transactions specifically for this narrow multi-counter case would add meaningful operational risk (transactions require a replica set, have their own retry/timeout semantics, and this codebase has no precedent for handling their failure modes) for a case that's actually rare in practice (most rewards have exactly ONE active reward-level constraint). **Chosen approach (a) — pre-check + ordered-increment + explicit-rollback-on-denial**, not (b) a real ACID transaction:

```java
public IncrementAllResult incrementAllOrRollback(List<CounterIncrementRequest> requests) {
    // Pre-check: cheap READ of all counters' current consumed/budget BEFORE attempting any write — catches
    // the common "one of them is already exhausted" case without any write, avoiding needless rollback churn.
    for (CounterIncrementRequest req : requests) {
        if (wouldExceedBudget(req)) return IncrementAllResult.denied(req.getWindowCycleKey());
    }
    // Ordered increment: attempt each counter's real atomic conditional write, in a FIXED order (by
    // windowCycleKey, deterministic) — if any DENIES (a race narrowed the pre-check's snapshot), roll back
    // every counter that had ALREADY succeeded in this same call, in REVERSE order, via compensate().
    List<CounterIncrementRequest> succeeded = new ArrayList<>();
    for (CounterIncrementRequest req : requests.stream().sorted(comparing(CounterIncrementRequest::getWindowCycleKey)).collect(toList())) {
        Optional<Integer> landedSlot = slotService.increment(req.getOrgId(), req.getRewardId(), req.getKpi(),
            req.getWindowCycleKey(), req.getDelta(), req.getRequestId());
        if (landedSlot.isEmpty()) {
            Collections.reverse(succeeded);
            for (CounterIncrementRequest done : succeeded) {
                slotService.compensate(done.getOrgId(), done.getRewardId(), done.getKpi(), done.getWindowCycleKey(),
                    done.getLandedSlot(), done.getDelta(), done.getDelta(), CompensationReason.DOWNSTREAM_FAILURE);
            }
            return IncrementAllResult.denied(req.getWindowCycleKey());
        }
        req.setLandedSlot(landedSlot.get());
        succeeded.add(req);
    }
    return IncrementAllResult.success(succeeded);
}
```

**Residual, honestly-flagged risk**: there is a narrow transient-read window between the pre-check and the ordered-increment phase where the pre-check's snapshot could be stale (another concurrent request changed a counter's state in between) — this is bounded and self-correcting (the ordered-increment phase's own atomic conditional writes are the REAL correctness gate; the pre-check is purely an optimization to short-circuit the common case cheaply) — not eliminated, judged acceptable (Risk §D.8).

### B.11 Metrics — including a critical gap that had to be fixed

⚠️ **Bug found [CRITICAL]**: the real `MetricsService` (`utils/metrics/MetricsService.java:9-19`) declares ONLY `addCustomParameter`/`addCustomEvent`/`publishPageInfo` — none of the counter/gauge methods this design's metrics have been assuming (`incrementCounter`, `recordGauge`) actually exist. **Required fix — an interface extension, a prerequisite for implementing ANY of this design's metrics:**
```java
public interface MetricsService {
    // ... existing methods unchanged ...
    void incrementCounter(String metricName, String... tagKeyValuePairs); // NEW
    void recordGauge(String metricName, double value, String... tagKeyValuePairs); // NEW
}
```

**Full metrics list (all depend on the extension above existing first):**
- `reward_counter_mongo_write_latency_ms` / `_failure_count`
- `reward_counter_slot_retry_count`, `reward_counter_slot_deny_count` (the sole authoritative "at limit" signal)
- `reward_counter_cron_rebalance_duration_ms`, `reward_counter_cron_sweep_docs_aggregated_count`
- `reward_counter_compensation_failure_count`, `reward_counter_dedupe_hit_count`
- `reward_counter_mongo_enabled` (gauge, per org), `reward_counter_legacy_reward_lock_usage_count` (should trend to zero once fully rolled out)
- `reward_counter_multi_constraint_and_gate_denial_count`, `reward_counter_redemption_value_null_skip_count`, `reward_counter_transaction_count_partial_failure_no_reversal_count`
- `reward_counter_limit_value_refresh_count{scope=windowed|eternal}`, `reward_counter_limit_refresh_lag_ms`
- `reward_counter_mysql_sync_docs_synced_count` / `_docs_skipped_stale_count`, `reward_counter_mysql_sync_overdue_backlog_gauge`
- `reward_counter_all_slots_exhausted_fast_fail_count` (optimization-effectiveness only, paired with the real deny metric to show the flag is advisory)
- `reward_counter_rebalance_job_leader_lock_gauge` / `reward_counter_mysql_sync_job_leader_lock_gauge` (one per split job)
- `reward_counter_ttl_expired_docs_count{collection}` (catches a regressed null-for-eternal rule)
- `reward_counter_rebalance_sweep_duration_ms` (gauge — tracks whether a sweep is exceeding its own cadence)
- `reward_counter_cron_org_shard_sweep_failure_count{job}` — per-org-shard sweep failures without aborting the whole tick
- `reward_counter_effective_slot_count_gauge{rewardId,kpi,windowCycleKey}` — the actual `computeEffectiveSlotCount()` output per meta-doc creation; the primary observability signal for the adaptive-sizing formula, and for spotting an ETERNAL counter (always `configuredMaxSlotCount`, should never drift) or a windowed counter whose sizing looks off relative to its real traffic.

### B.12 Migration / Cutover Plan

- Feature-flagged (`reward.counter.mongo.enabled`, default false).
- Sequencing: (1) ship code with flag OFF (safe no-op deploy); (2) TTL index provisioning (§B.7) must complete BEFORE the flag is turned on for ANY org; (3) `spring.task.scheduling.pool.size=2` must be deployed and confirmed BEFORE the split-cron jobs are relied upon; (4) seed Mongo slot/meta docs from the current MySQL `CONSUMED` value using the SAME `$setOnInsert` upsert path as live lazy-creation (never a separate direct write — a race between the seed script and a live in-flight request at cutover could otherwise produce two different seeding semantics for the same doc); seeding must populate ALL new lifecycle fields (`createdOn`, `windowStartAt`/`windowEndAt`, `allSlotsExhausted=false`, `lastSyncedAt=null`, `lastSyncedCumulativeTotal=alreadyConsumed` — CORRECTED from an earlier "starts null" claim: seeding this to `null` for a nonzero-`alreadyConsumed` ETERNAL doc would make the FIRST sync tick's delta equal the FULL cumulative total, double-counting the already-untouched historical rows on top of themselves — this seeding step MUST use the exact SAME value `getOrCreate()` itself seeds, since both paths need to agree; for a genuinely brand-new reward with zero prior consumption, `alreadyConsumed` is naturally `0`, the correct starting ledger value, not a special-cased null — `expireAt` with the identical null-for-eternal rule (the one seeding step where a mistake silently creates a self-deleting "eternal" counter)); (5) flip flag ON for a single canary org during a low-traffic window; (6) monitor for several cron-cadence multiples before widening org-by-org or globally.
- **The ETERNAL-counter migration-consolidation step from earlier design iterations is REMOVED entirely.** There is nothing to consolidate: legacy's pre-existing historical daily rows are left completely untouched (they're already correctly included in legacy's own unfiltered all-time SUM), and the sync path (§B.8) only ever ADDS new rows/increments for NEW consumption after cutover — using the exact same daily bucket-key legacy's own write path already computes. No separate migration script, no ordering-vs-cron-race concern, no cross-writer date-agreement problem.
- **Required pre-cutover step (0), before step 1 above**: verify `TBL_REWARD_CONSTRAINT` has an index covering `(ORG_ID, REWARD_ID)` in the PRODUCTION database via `SHOW INDEX FROM TBL_REWARD_CONSTRAINT`. Verified: `TBL_REWARD_ISSUE_SUMMARY` DOES have the needed composite index (`idx_rewardIssue_summary` on `ORG_ID,REWARD_ID,CONSTRAINT_LEVEL,KPI,ISSUE_DATE`), but `TBL_REWARD_CONSTRAINT`'s entity has NO `@Index` annotations and no confirmed secondary index in the available DDL — Hibernate's `hibernate.hbm2ddl.auto=none` means indexes here are DB-managed, not code-managed, so this cannot be verified from the repo alone. This matters directly for §B.6/§B.8's batched `LIMIT_VALUE`-check — without this index, that query risks a table scan at high reward-count. If missing, this is a DBA action item, not a code change — but it MUST be confirmed before relying on the batched-check's assumed performance at ~100K-meta-doc scale.
- **Required Mongo indexes** (beyond the `expireAt` TTL index from §B.7) — without these, several of the cron's own queries will full-collection-scan at ~100K docs:
  - `rewardCounterCycleMeta`: compound index `{status: 1, windowCycleKey: 1}` (backs `findActiveOrEternalMetaDocs()`), and a PARTIAL index `{lastSyncedAt: 1}` filtered to `{status: "ACTIVE"}` (backs the MySQL-sync job's staleness filter) — a partial index keeps this small even as the eternal/expired population grows.
  - `rewardCounterDedupe`: compound index `{status: 1, createdOn: 1}` (backs the orphaned-`PENDING`-marker GC sweep).
  - These must be declared via `@Indexed`/`@CompoundIndex` on the entity classes AND actually provisioned per the same mechanism (and same "additive-only, brand-new-collection" caveat) as §B.7's `expireAt` index.
- **JSVT-org exclusion is code-enforced**, not a rollout-runbook step: the flag gate at `processWrapper` entry ANDs with "this org is not currently JSVT-mirrored," reading the same org-state the live JSVT mirror/compare flow itself consults.
- **Rollback**: flag OFF reverts to lock-based enforcement instantly. The "forced final sweep" runbook step means: force one `RewardCounterMySqlSyncJob` tick to completion for ALL active docs (bypassing the staleness filter for this one forced run), which ALSO runs the dedupe-marker GC in the same pass — this single forced run handles both "catch MySQL up" and "clean up dedupe markers" for a clean rollback.
- Cutover verification should include one live `LIMIT_VALUE` edit on the canary org's constraint, confirming `reward_counter_limit_value_refresh_count` increments and `reward_counter_limit_refresh_lag_ms` stays bounded — proving §B.6's fix works against the real org-sharded topology, not just in unit tests.

*(Continued in Part 4 of 4 — Parts C (scaling), D (risks), E (code-paths + criticality tables), and F (the complete 118-test strategy + full version changelog).)*
