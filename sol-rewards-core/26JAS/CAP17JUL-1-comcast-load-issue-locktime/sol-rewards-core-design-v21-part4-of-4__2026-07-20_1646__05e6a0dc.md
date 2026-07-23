---
title: "issueReward Lock Removal — Complete Design v21 (Part 4 of 4, FINAL)"
subtitle: "Scaling, Risks, Code-Paths + Criticality Tables, Complete Test Strategy, Full Changelog"
---

* * *
title: "issueReward Lock Removal — Complete Design v21 (Part 4 of 4)"
subtitle: "Parts C (scaling), D (risks), E (code-paths + criticality table), F (complete 118-test strategy)"
---

# sol-rewards-core: Removing the issueReward Redis Lock — Complete Design Reference (v21, self-contained) — PART 4 of 4 (FINAL)

**⚠️ NOTE ON THIS SPLIT:** This is the final part, continuing from Part 3 (§B.12 Migration/Cutover). Covers Part C (5K rpm scaling), Part D (honest risk list), Part E (verified code-paths table + criticality table), and Part F (the complete 118-test strategy plus the full version changelog v11→v21).

* * *

## PART C — SCALING TO 5,000 RPM

| Setting | Value | Reasoning |
|---|---|---|
| `reward.counter.slot.count` | **16** | 83.3 req/sec ÷ 16 ≈ 5.2 req/sec/slot — comfortably sub-contention even under 3x skew (~15.6 req/sec on the hottest slot). More slots (e.g. 32) increase cron aggregation cost with diminishing returns once per-slot rate is already this low. |
| `reward.counter.slot.maxRetryAttempts` | **4** | Retries expected near-zero in steady state (cron rebalancing keeps skew low); worst case adds ≈4×5-10ms ≈ 20-40ms tail latency — still vastly better than the eliminated 91-101s lock-wait. |
| `reward.counter.rebalance.fixedDelayMs` | **5000** | ~26 requests/slot/tick at this rate — small enough for 5s cadence to keep skew self-correcting. |
| `reward.counter.mysqlSync.fixedDelayMs` | **45000** | GET-API staleness tolerance is much looser than enforcement timing; paired with the `lastSyncedAt` staleness filter so idle counters aren't needlessly re-synced. |
| Mongo write concern | **`w:majority, j:true`** | Correctness-critical counter — `w:1`/`j:false` risks a lost write on failover (silent under-count, undermining strict enforcement). Extra cost ≈ single-digit ms — negligible vs. the eliminated lock-wait. |
| `reward.counter.slot.minUnitsPerSlot` | **10** | See §B.5's adaptive slot-count formula — ensures each slot's sub-budget is meaningfully sized relative to a typical issuance delta. |

**What actually gates latency**: a single-document conditional `findOneAndUpdate` is storage-engine-cheap; at this write rate, the dominant cost is the write-concern acknowledgment round trip (journal commit + replica-set ack), not document contention.

⚠️ **This table's `slot.count=16` describes a HIGH-traffic reward at the configured MAX slot count.** `computeEffectiveSlotCount()` (§B.5) only reaches 16 for a reward whose `limitValue` is at least `16 × minUnitsPerSlot` (=160 at the proposed defaults). A LOW-limit reward (e.g. `limitValue=100`) uses FEWER slots (10, per the formula) — this does not change the 5K-rpm analysis above, since that analysis is scoped, by construction, to a reward large/popular enough to need the max slot count in the first place: a `limitValue=100` reward sustaining 5,000 rpm would exhaust its own budget in well under a minute regardless of slot count — its OWN limit, not slot contention, is the binding constraint at that point. The adaptive formula optimizes the other end of the spectrum: a low-limit reward under modest, sub-limit traffic, where fewer, better-utilized slots mean less retry-to-next-slot overhead than a wastefully-fine split would cause.

These are concrete starting defaults with reasoning shown, not guesses — but should still be validated against a real load test before go-live.

* * *

## PART D — RISKS / OPEN QUESTIONS (honest, not swept under the rug)

1. **5K-rpm defaults need real load-test validation** before go-live.
2. **Multi-constraint AND-gate (§B.2) is genuinely new design intent**, not verified regression parity — needs explicit product/business sign-off that "most-restrictive-wins across co-resident constraints" is the intended behavior.
3. **§B.6 residual staleness window**: a `LIMIT_VALUE` decrease has a small, bounded (not zero) over-issuance window, bounded by the rebalance cadence (~5s default) — must be communicated to product as "propagates within ~1 tick," not instant parity with legacy.
4. **Independent-NO_LIMIT eternal counters** — a limit change now propagates via §B.6 even for `ETERNAL` docs (bounded by rebalance cadence). What remains open: a *structural* reclassification (dependent↔independent, e.g. a sibling constraint added/removed) is a different transition, unaffected by this fix (covered by test #48).
5. **Dedupe `pendingTimeoutMs` sizing** must be validated against the real `IdempotencyCheckIssueProcessor`/`IssuedTransaction` retry-detection window.
6. **ROLLING compensation-drift**: a late compensation landing on an already-closed prior day causes minor, bounded, self-correcting drift in a later day's frozen headroom (bounded by the single compensation's size, not cumulative) — accepted, not eliminated, per explicit user direction not to over-engineer this edge case.
7. **JSVT migration-variant interaction** — whether `sol-rewards-core-jsvt-a` runs an equivalent enforcement path that must stay in lockstep is still open for its owning team to confirm; the exclusion mechanism itself is code-enforced regardless of the answer.
8. **Multi-counter atomicity's transient-read window** (§B.10) — narrow, judged acceptable, not eliminated.
9. **`OrgMongoDataSourceManager.getAllShardKeys()`'s refresh semantics** (does it pick up a newly-provisioned org-shard without an app restart?) live in an external base class, not verified from this repo — confirm with the owning team before relying on it for cron completeness. Applies to BOTH split jobs.
10. **First-ever in-process `@Scheduled` job in this service** — new infrastructure surface (Spring's scheduling thread pool, behavior under app restart/redeploy mid-tick) with no precedent in this codebase. `spring.task.scheduling.pool.size=2` (§B.8) resolves the specific starvation risk found; recommend the implementation phase pay particular attention to integration-testing the cron jobs end-to-end early, given this exact area (locking/scheduling/metrics plumbing) has proven to be where the real bugs hid across two consecutive review rounds.
11. **TTL null-for-eternal is a silent-failure risk class** (§B.7) — a regression quietly deletes a live eternal counter with no application error. Verify Mongo's TTL monitor correctly skips null/absent-field docs (standard behavior, not verified against this repo's specific driver/version) before relying on it.
12. **Two independent leader-lock keys** must stay clearly, distinctly named (`reward-counter-cron-leader:rebalance` vs `:mysqlsync`) — a copy-paste error reusing one key for both jobs would silently starve one across the whole fleet. Covered by test #60.
13. **FUTURE-CHANGE CAVEAT — TTL index-definition changes after initial rollout will not self-reconcile.** §B.7's re-enabled `createIndexes()` (or `auto-index-creation=true`) is sufficient for THIS design's initial go-live (brand-new collections, no pre-existing index to reconcile). But the underlying limitation — Spring Data MongoDB's annotation-driven index creator being additive-only, unable to drop a stale index after a future definition change — is a **pre-existing gap in this service's Mongo index-management story generally**, affecting every collection that uses `@Indexed`/`@CompoundIndex` (not just the two new ones this design adds). **Explicitly OUT OF SCOPE for this design to solve** — it should be addressed once, platform-wide, in a separate initiative if/when it matters, not re-solved narrowly here.
14. **Adaptive-slot-count `minUnitsPerSlot` sizing** — the proposed default of 10 is a reasoned starting point, not load-tested; if a reward's typical single-issuance delta for a given KPI (e.g. a POINTS delta routinely in the hundreds) is much larger than `minUnitsPerSlot`, the formula could still under-allocate slots relative to what's ideal for THAT reward's actual issuance size. This is a TUNING risk, not a correctness risk: `computeEffectiveSlotCount()` always floors at 1 and caps at `configuredMaxSlotCount`, so it can never produce zero slots or exceed the existing global ceiling.
15. **[RESOLVED, was a real gap] Eternal-counter slot-count freeze on a later LIMIT_VALUE increase** — a windowed (FIXED/ROLLING) counter that's sized small at creation self-heals automatically the next time its cycle rolls over — but an independent-NO_LIMIT/`ETERNAL` counter's meta doc lives for the reward's entire lifetime with **no such rollover boundary**. **Fixed**: `computeEffectiveSlotCount()` takes `windowCycleKey` and short-circuits to the full `configuredMaxSlotCount` for `"ETERNAL"` — adaptive sizing applies ONLY to windowed counters. No longer open; listed for visibility.
16. **[NEW, this version] `RewardIssueSummaryJdbcRepository.bulkAtomicIncrement`/`save` DuplicateKeyException-catch reconciliation pattern** (§B.8) is a NEWLY-INTRODUCED defense-in-depth guard, not previously exercised anywhere in this codebase for this exact race shape — recommend explicit integration testing of this catch-and-reconcile branch early in implementation, since it's the newest, least-precedented piece of logic in the whole design.
17. **[NEW, this version] Multi-counter atomicity's chosen approach (pre-check + ordered-increment + rollback, §B.10)** deliberately avoids Mongo's wired-but-unused `MongoTransactionManager` — if a future requirement needs STRONGER cross-counter atomicity guarantees than "eventually rolled back on denial," revisit whether a real transaction is now justified (the current approach is judged sufficient for the documented multi-counter case, which is expected to be rare in practice).

* * *

## PART E — CODE PATHS TABLE (verified file:line)

| # | File:line / New | Change |
|---|---|---|
| 1 | `NonOrgSummaryWriteProcessor.java:53` | Remove `REWARD_LOCK_KEY_PREFIX` |
| 2 | `NonOrgSummaryWriteProcessor.java:100-134` | Replace lock/reEvaluate/writeAtomically block with the flow in §B.5-B.10 |
| 3 | `NonOrgSummaryWriteProcessor.java:152-183` | No longer called for REWARD under flag; kept for flag-OFF legacy path |
| 4 | `RedisLockService.java:76-102` (`acquireRewardLock`) | Caller removed for REWARD; method kept for flag-OFF fallback |
| 5 | `RedisLockService.java:104-128` (`acquireLock`) | Superseded by `tryAcquireNonBlocking` for the new crons; original method untouched for other existing callers |
| 6 | `RewardConstraintFacade.java:617-702` (`writeNonOrgSummariesAtomically`) | Zero invocations for REWARD when flag ON; customer-tier call structurally untouched |
| 7 | `RewardConstraintFacade.java:326-355` (`getIndependentRewardCustomerNoLimitRestrictions`) | Grouping logic mirrored in the new flow's classification step |
| 8 | `RewardConstraintFacade.java:461-465` (`fetchPerUnitRedemptionValue`) | Called BEFORE Mongo compensation on a REDEMPTION_VALUE revoke; required by `compensate(..., REVOKE)`'s null-guard |
| 9 | `RewardConstraintFacade.java:754-755` | TRANSACTION_COUNT conditional compensation, mirrored exactly |
| 10 | `RewardConstraint.java:136-166` (`isEquivalentToConstraint`) | Grounds the `(level,kpi)`-only key-identity fix (§B.2) |
| 11 | `RewardLevelService.java:80-91` | Per-KPI evaluation semantics mirrored (TRANSACTION_COUNT hardcoded 1) |
| 12 | New: `db/mongo/RewardCounterSlot.java`, `RewardCounterCycleMeta.java`, `RewardCounterDedupe.java` | New entities, corrected key shape (§B.5) |
| 13 | New: `service/RewardCounterSlotService.java`, `RewardCounterCycleMetaService.java` | Per §B.9/B.6 |
| 14 | New: `service/impl/RewardCounterRebalanceJob.java`, `RewardCounterMySqlSyncJob.java` | Per §B.8 — split, org-shard-aware, corrected leader-election |
| 15 | `dto/wrapper/BulkRewardIssueContext.java:87-114` | Carry `{landedSlot, windowCycleKey, kpi}` per counter for REWARD entries |
| 16 | `RewardsApplicationConfiguration.java` | `mongo.enabled`, `slot.count=16`, `slot.minUnitsPerSlot=10`, `slot.maxRetryAttempts=4`, `rebalance.fixedDelayMs=5000`, `mysqlSync.fixedDelayMs=45000`, `mysqlSync.syncStalenessThresholdMs=45000`, `dedupe.pendingTimeoutMs` |
| 17 | `NewRelicConstants.java:41-44` | Deprecate 4 reward-lock metrics for this path; add new (§B.11) |
| 18 | `Constants.java:281` (`JSVT_MIRROR_ENABLED`) | Explicit AND at processor-entry gate (§B.12) |
| 19 | `MongoClientConfiguration.java:84-87` (`mongoTransactionManager`) | Confirmed wired but unused; grounds the "approach (a) vs (b)" justification (§B.10) |
| 20 | `OrgMongoDBFactory.java:30-38,16` | Grounds the org-shard-iteration fix (§B.8) |
| 21 | `OrgMongoDataSourceManager.java:16` (`getAllShardKeys()`) | Cron's shard-enumeration source |
| 22 | `MongoDbInitializer.java:38-39,46` | Precedent referenced; its `createIndexes()` is dead code needing re-enabling (§B.7) |
| 23 | New: `RewardCounterSlotService.CounterIncrementRequest` (DTO) | Input to `incrementAllOrRollback` |
| 24 | New: `RewardCounterCycleMetaService.refreshLimitIfChanged()` | §B.6 |
| 25 | `RewardCounterCycleMetaService.getOrCreate()` | Extended with `windowStartAt`/`windowEndAt` params + new field seeding (§B.5); `slotCount` param REMOVED, replaced by internal `computeEffectiveSlotCount(limitValue)` call |
| 28 | New: `MetricsService.incrementCounter()`, `.recordGauge()` | **Required prerequisite** — nothing in this design's metrics can be implemented without it (§B.11) |
| 29 | New: `RedisLockService.tryAcquireNonBlocking()` | Non-throwing leader-election helper (§B.8, Bug 1) |
| 30 | New: `rewardCounterRebalanceLockRegistry` / `rewardCounterMySqlSyncLockRegistry` beans | Dedicated TTL per job (§B.8, Bug 2) |
| 31 | `RewardsApplicationConfiguration.java` | `spring.task.scheduling.pool.size=2` — REQUIRED (§B.8, Bug 3) |
| 32 | `MongoDbInitializer.java` (re-enable `createIndexes`) OR ops runbook step | TTL index provisioning — explicit, required (§B.7) |
| 33 | `RewardCounterCycleMetaService.refreshLimitIfChanged()` | Combined into ONE atomic `updateFirst` (§B.6 fix) |
| 34 | New: `RewardCounterCycleMetaService.computeEffectiveSlotCount(BigDecimal, String)` | `min(configuredMaxSlotCount, max(1, floor(limitValue / minUnitsPerSlot)))`, called once inside `getOrCreate()` (§B.5) |
| 35 | `RewardConstraintValidation.java:68-69` (`validateEachLevel`) | No code change — grounds §B.3's factual correction: unconditionally blocks `KPI.POINTS` at creation for every level |
| 36 | `RewardIssueSummaryContext.java:80-91` (`setIsValidForPoints`) | No code change — grounds §B.3: the evaluator's real, functional REWARD+POINTS limit check |
| 37 | New: `RewardCounterCycleMetaService.overwriteEternalMySqlSummary()` | §B.8 — the daily-row-mirroring ETERNAL sync mechanism (v19-v21's corrected version) |
| 38 | New: `RewardCounterCycleMeta.lastSyncedCumulativeTotal` field | §B.5/B.8 — the delta-computation ledger, seeded to `alreadyConsumed` at creation, never null |
| 39 | `RewardConstraintFacade.java:581-585` (`resolveOrgZoneId`) | Reused by the cron's org-timezone resolution (§B.8 fix #2) |
| 40 | `RewardIssueSummaryJdbcRepository.java` — `findExistingForNonOrgLevel`, `bulkAtomicIncrement`, `save`, `decrementConsumedFloorZero` | Reused (not reinvented) by the ETERNAL sync's find-or-create + negative-delta paths (§B.8) |
| 41 | `RewardConstraintFacade.java:412-414` (`bulkSaveOrUpdate`'s insert-then-reconcile pattern) | Reused shape for the ETERNAL sync's `DuplicateKeyException` catch-and-reconcile (§B.8, v21 fix) |

## Criticality Table

| Component | Criticality | Why |
|---|---|---|
| Key-identity fix (§B.2) | High | Wrong key creates duplicate/orphaned counters mismatched to the code's real dedup semantics |
| Multi-constraint-same-KPI resolution (§B.2) | High | Wrong resolution under/over-enforces across co-resident constraints |
| Multi-counter atomicity / `incrementAllOrRollback` (§B.10) | **High** | Without it, partial application on denial silently over-counts a counter forever |
| Cron org-shard iteration (§B.8) | **High** | Without it, cron throws or silently sweeps zero/one slot — reporting/rebalancing breaks org-wide |
| REDEMPTION_VALUE null-skip + DB-fetch-on-revoke, with `CompensationReason` (§B.3, B.9) | High | Silent miscounting or a crash on revoke if ordering is violated |
| POINTS defensive write/compensate support (§B.3) | Low-Medium | Currently unreachable via the supported creation path — kept for forward-compatibility |
| TRANSACTION_COUNT conditional compensation (§B.3) | Medium-High | Naive decrement incorrectly reverses partial failures |
| Window/frequency matrix, ROLLING-MONTHS 30-day math (§B.4) | High | Silent limit miscalculation if calendar-month math substituted |
| LIMIT_VALUE live-refresh (§B.6) | **High** | Without it, this design regresses vs. legacy's effectively-instant limit-edit propagation |
| Split cron / two schedules (§B.8) | Medium-High | First-ever in-process scheduled job; two independent leader-lock keys must stay correctly distinct |
| Leader-election API fix (§B.8, Bug 1) | **Critical** | As originally drafted, doesn't compile / would throw every tick on every non-leader instance |
| Dedicated lock-registry TTLs (§B.8, Bug 2) | **High** | Without it, a crashed leader strands the lock 10 minutes, breaking the propagation-bound promise |
| Scheduler pool sizing (§B.8, Bug 3) | **High** | Without it, splitting the cron achieves nothing — the two jobs would still serialize on one thread |
| TTL index provisioning, initial go-live (§B.7) | **High** | Without it, the entire TTL-hygiene mechanism silently provisions nothing on day one |
| TTL index provisioning, future-change caveat (§B.7) | Medium | Not a day-one blocker; becomes High the moment someone changes the TTL definition without adding the reconciliation step |
| `MetricsService` API extension (§B.11) | **High** | Without it, this design's entire observability story is unimplementable as specified |
| `refreshLimitIfChanged` atomicity (§B.6 fix) | Medium | Removes a narrow but real inconsistent-intermediate-state window |
| TTL `expireAt` null-for-eternal correctness | Medium-High | Silent-failure risk class — a regression quietly deletes a live eternal counter with no application error |
| Adaptive slot-count formula (§B.5) | Medium | Not correctness-bearing — a practical-efficiency fix; a wrong formula wastes retries, it does not cause a breach |
| 5K-rpm scaling defaults (§C) | Medium | Tuning, validated by load test before go-live |
| Core slot/rebalance/dedupe mechanism | High | Foundational — everything else builds on this |
| **ETERNAL-counter daily-row-mirroring sync mechanism (§B.8, this version)** | **High** | The most-revised piece of this entire design (5 fix rounds v15→v21) — the find-or-create sequence, org-timezone resolution, write ordering, first-tick double-count avoidance, and negative-delta routing are ALL individually correctness-bearing |
| Customer-level path | None | Untouched, structurally leak-proof |

* * *

## PART F — HOW TO USE THIS DOCUMENT

This is the FINAL part (4 of 4) of a design that went through 21 iterations. To read the complete design in order:
1. **Part 1** — Parts A (problem/root cause/acceptance criteria) and B.1–B.5 (core mechanism: key identity, per-KPI handling, window/frequency matrix, the three Mongo collections).
2. **Part 2** — continuation of §B.5 (the full `getOrCreate()` seeding logic with all its historical fixes), §B.6 (LIMIT_VALUE live-refresh), §B.7 (TTL storage hygiene), §B.8 (the cron jobs' ETERNAL-sync mechanism — the single most-revised piece of this whole design).
3. **Part 3** — the three real bugs found in the cron leader-election/scheduling plumbing + their fixes, the complete corrected job class bodies, §B.9 (`RewardCounterSlotService`), §B.10 (multi-counter atomicity via `incrementAllOrRollback`), §B.11 (metrics — including the critical `MetricsService` API gap), §B.12 (the full migration/cutover plan).
4. **Part 4 (this part)** — Parts C (5K rpm scaling numbers), D (the honest risk list), E (the verified code-paths + criticality tables), and F (this section, plus the complete 118-test strategy below).

### Complete Test Strategy (118 tests total)

**Foundational tests (1–28)**: slot `increment()` boundary/fresh/existing-slot tests; strict-no-breach concurrency test; retry-to-next-slot exhaustion/randomization/cap tests; meta-doc lazy-creation race test; cron rebalancing saturation/skewed-consumption regression cases; ROLLING per-day + drift worked example; FIXED rollover; compensation-targets-actual-landed-slot; flag ON/OFF regression tests; customer-level isolation; cron leader election; cron/Redis-outage-degrades-reporting-only test; migration seeding sum-check; slot-count=1 degenerate-config guard; Decimal128 precision; EXPIRED-meta fail-closed; dedupe-TTL sizing; ROLLING-reads-MySQL-not-Mongo; compensation-failure-queued test; metrics-diff test; duplicate-constraintId dedup test.

**Key-identity / multi-constraint / KPI / window-matrix tests (29–51)**: key-identity + multi-constraint AND-gate tests; degenerate same-windowCycleKey sharing; dependent/independent NO_LIMIT tests; per-KPI delta/compensation tests (QUANTITY, POINTS, REDEMPTION_VALUE null-skip + DB-fetch-ordering, TRANSACTION_COUNT full-vs-partial); ROLLING-MONTHS 30-day math; 5K-rpm load/soak test (+ CI-fast substitute); MIN-limit test; rollback-on-partial-denial test; dedupe PENDING/COMMITTED tests; `computeDelta`/`compensate` fail-closed tests; full-KPI end-to-end tests; flag-OFF regression; ROLLING-doesn't-double-sum; NO_LIMIT reclassification; enforcement-during-cron-outage; deterministic concurrency unit test; cron org-shard-iteration test.

**Backdated-date / mid-cycle-change tests (52–53)**: ROLLING-vs-FIXED backdated-date handling; mid-cycle frequency/interval-change test.

**LIMIT_VALUE live-refresh / doc-lifecycle tests (54–62)**: increase/decrease live-refresh; multi-constraint live-refresh; `allSlotsExhausted` advisory test; hybrid sync staleness-filter test; TTL null-for-eternal + computed-correctly tests; split-cron leader-lock isolation; `createdOn` presence.

**Plumbing-fix tests (63–75)**: leader-election non-throwing test; dedicated lock-registry TTL test; scheduler pool capacity test; TTL index existence integration test; `refreshLimitIfChanged` atomicity test; `MetricsService` contract test; dedupe-GC-vs-forced-sweep; limit-decrease concurrency stress; flag-OFF disables both jobs; advisory-flag processor-level test; limit-increase edge cases; `createdOn` immutability; timezone-boundary test.

**Adaptive-slot-count tests (76–89)**: POINTS-for-REWARD defensive-support; adaptive slot-count basic/clamp/floor tests; ETERNAL-always-max-slots test; multi-constraint MIN-limit-drives-slot-count test; negative/zero limitValue defensive floor; `slotCount` immutability under refresh; fractional limitValue; exact-boundary tests; config-boundary parameterization; new-metric-emission test; low-slot-count concurrency stress; `RoundingMode.FLOOR` precision boundary.

**Q&A-round fixes (90–96)**: `evenSplit` exact-sum test; non-upserting/floor-at-zero compensation tests (97a, 97b); existing-reward-with-consumption seeding test (92, verifying actual slot docs); eternal-doc-loss recovery test (93); over-limit-at-seed-time test (97c); RMQ-vs-cron-backstop idempotency test (97d); `evenSplitBudgets` hotspot-distribution + multi-case exact-sum tests (97e, 97f); slot-seed-before-meta-publish ordering test (97g); event-driven LIMIT_VALUE propagation test (94); batched cron-check scaling test (95); `incrementAllOrRollback` no-op-overhead test (96).

**ETERNAL-sync mechanism tests (98–101, spanning v15→v21's 5 fix rounds)**:
- **98**: daily-increment correctness — historical rows untouched, one row per calendar day, all-time SUM matches Mongo total.
- **98b**: zero-delta sync skip (no wasted writes on idle counters).
- **98c**: same-day multiple-tick accumulation (one row per day, correctly accumulated).
- **98d**: day-rollover new-row test (correct bucket-boundary behavior).
- **98e**: concurrent legacy-write-and-sync-tick same-day test (correct scoping, no interference).
- **98f**: find-or-create correctness (the real `bulkAtomicIncrement` signature only UPDATEs, never INSERTs).
- **98f-2**: concurrent find-or-create safety (two racing writers never produce duplicate un-consolidated rows).
- **98f-3**: same-org legacy-write-races-sync test (the `DuplicateKeyException` catch-and-reconcile closes this cleanly either write-order).
- **98g**: org-timezone resolution correctness (non-UTC zone boundary, per-org caching, per-doc exception isolation).
- **98h / 98h-2**: crash-between-writes ordering + explicit 3-tick end-to-end self-heal sequence.
- **98i**: first-tick double-count regression test (the exact bug QA caught in v19/v20).
- **98j**: negative-delta (revoke/compensation) routing test.
- **98k**: sentinel-row-gone permanent regression guard.
- **99**: `TBL_REWARD_CONSTRAINT` index-assumption query-plan test.
- **100 / 100b**: new Mongo index existence + partial-index filter-selectivity tests.
- **101**: compensation-retry backoff-shape consistency test (reworded to NOT claim shared RabbitMQ machinery).

### Full version changelog summary (v11 → v21)

- **v11**: POINTS-for-REWARD factual correction; ROLLING-drift clarification; TTL additive-only sharpened fix; adaptive slot-count mechanism introduced.
- **v12**: fixed ETERNAL counters being wrongly included in adaptive sizing (they now always get max slots); added the missing metric; 10 new tests closing senior+QA gaps in the adaptive-sizing mechanism.
- **v13**: shard→slot terminology rename (avoiding confusion with MongoDB's own native sharding); `getOrCreate()` proportional-seeding fix; several clarifications (customer-lock vs dedupe, PointsRetrial orthogonality, idempotency-processor interaction, TTL scoping); LIMIT_VALUE cron-scaling fix + event-driven propagation; `evenSplit` exact-sum specification; non-upserting compensation fix.
- **v14 / v14.1 / v14.2**: fixed 5 real bugs where v13's claimed fixes were narrative-only (proportional seeding never actually created slot docs, `evenSplit` truncated fractional remainders, cron batching was narrative-only, compensation conflated two distinct failure cases); v14.1 fixed a sequential-round-trip performance issue in the v14 fix itself; v14.2 (arbiter-caught) fixed a real, bounded over-issuance race in the v14.1 fix, reordering slot-seed-before-meta-publish to close it entirely. **Arbiter PASS at v14.2.**
- **v15–v18**: a user-directed e2e review surfaced the ETERNAL-counter MySQL-sync gap (among other findings) — v15-v18 tried a "sentinel row" mechanism for this, which went through 3 consecutive arbiter REFINE cycles fixing: a nonexistent repository method assumption, a migration-vs-cron race, a factually-wrong `ON DUPLICATE KEY UPDATE` claim against the real DDL (the table has no unique key on the relevant columns), and a cross-writer sentinel-date-agreement problem.
- **v19**: a user-driven investigation (tracing `Utils.getDateWithoutTimestampInSpecifiedZone`'s callers) confirmed `TBL_REWARD_ISSUE_SUMMARY` is a genuinely daily-bucketed table for ETERNAL constraints — **replaced the entire sentinel-row mechanism** with a simpler approach that mirrors legacy's own daily bucketing, eliminating the migration-consolidation step and the cross-writer date problem entirely.
- **v20**: fixed 5 real bugs senior+QA review found in v19's pseudocode (wrong repository method signature assumed — needed an explicit find-or-create; org-timezone was never resolved in the cron's background context; the two-store write had unsafe ordering — fixed by writing the Mongo ledger first; a first-tick double-count bug from a null-defaulted ledger; unspecified negative-delta handling). Also fixed a corrupted job body from an editing mishap.
- **v21 (this version)**: fixed 2 more real bugs QA found in v20 (the find-or-create sequence itself had an unaddressed concurrent-writer race — fixed via a duplicate-key-catch-and-reconcile pattern reusing an existing repository convention; a migration-seed documentation inconsistency that could have silently reintroduced the v20 first-tick double-count fix).

**Test count: 118 total.** **Not yet re-arbitrated** — v15 onward are real design changes since the last arbiter PASS (v14.2); a fresh senior/QA review + arbiter cycle is required on this version before it can be treated as approved for implementation.
