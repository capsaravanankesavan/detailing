# Tech Detail: Expiry Date Change Job — MongoDB Load Reduction
**Date:** 2026-04-28
**Author:** Saravanan Kesavan
**Status:** Final
**Investigation Doc:** `CAP-184618_expiry_job_tuning_analysis.md`
**Confidence:** HIGH (inherited and validated against worktree source)

---

## 1. Problem Statement

The `ExpiryDateChangeJobListener` processes bulk expiry-date updates for three MongoDB
collections (`customerIssuedPromotion`, `customerEarnedPromotion`, `codeBasedPromotion`).
When a brand updates 300 promotion end dates in a single batch, up to 600 RMQ messages land
on `PROMOTION_EXPIRY_UPDATE_QUEUE`. All 6 pods concurrently pull and execute multi-iteration
bulk update loops, producing sustained CPU and IOPS spikes on MongoDB.

Two orthogonal optimisations are in scope:
1. **Query inefficiency** — `AbstractExpiryDateChangeDao.updateValidTill()` uses an `_id`
   range predicate on `updateMulti`, which can span documents from other tenants.
2. **Uncontrolled cross-pod concurrency** — no mechanism limits how many jobs run in
   parallel across pods; the value is `pods × consumers-per-queue` and cannot be tuned per
   queue without impacting all other listeners.

---

## 2. Root Cause (Confirmed)

**Optimisation 1:** `AbstractExpiryDateChangeDao.getAggregation()` returns only the last
`_id` of the batch. The subsequent `updateMulti` uses `_id > startId AND _id <= endId` as
a range, where `endId` is an ObjectId whose value spans other tenants' documents in global
time-ordered ID space, causing unnecessary index traversal.

**Optimisation 2:** `PROMOTION_EXPIRY_UPDATE_QUEUE`'s listener uses the global Spring AMQP
factory — configured via `spring.rabbitmq.listener.simple.*` — which applies uniformly to
all queues. No per-queue concurrency control is possible without a dedicated factory.

### Contributing Factors

1. `group().last(_id)` discards the full list of matching IDs, forcing the update to re-derive
   the set via range instead of targeting exact documents.
2. The global `consumers-per-queue=1` / `prefetch=1` setting cannot be decoupled per queue
   without a named container factory.
3. No distributed semaphore to cap total concurrent jobs across all pods.

---

## 3. Scope of Change

### In Scope
- Replace range-based `updateMulti` with `$in` on primary key in `AbstractExpiryDateChangeDao`.
- Same fix in `CustomerEarnedCollectionDynamicExpiryDateChangeDao` (override).
- Dedicated `SimpleRabbitListenerContainerFactory` for `PROMOTION_EXPIRY_UPDATE_QUEUE`.
- Redis atomic semaphore in `ExpiryDateChangeJobFacade` to cap cross-pod concurrency.
- Semaphore counter reconciliation in `ExpiryDateChangeHelper` (post-shard-loop).
- New `countByStatus` query on `ExpiryDateChangeJobDao`.
- One new env-var-backed property in `application.properties` (`EXPIRY_MAX_GLOBAL_CONCURRENT_JOBS`, default 3).
- **WAITING status** (`BatchStatus.WAITING`) — new enum value. On semaphore rejection, job transitions to WAITING (updates `lastUpdatedOn`), preventing premature EXPIRED marking during the retry window. `shouldProceed()` allows WAITING. `markAllJobAsExpiredIfNotRunning()` expires WAITING orphans after 2h. `isJobRunningForPromotion()` includes WAITING to prevent duplicate jobs during retry.
- **`SemaphoreRejectedException`** — new `RuntimeException` subclass. Thrown on semaphore limit hit. Allows `RMQMessageTrackerAspect` to distinguish throttle rejections from real failures.
- **`RMQMessageTrackerAspect` update** — skip `markAsFailedRequest()` for `SemaphoreRejectedException` to eliminate NR metric noise during normal throttle behaviour.
- **`MessageRecoverer` (Fix-B)** — wired to `expiryDateChangeContainerFactory`. Fires exactly once when all retry attempts are exhausted; logs ERROR and emits `EXPIRY_JOB_RETRY_EXHAUSTED_EVENT` NR custom event.

### Out of Scope (Explicit)
- `CustomerIssuedCollectionStaticExpiryDateChangeDao` — inherits abstract class; zero changes.
- `CustomerEarnedCollectionStaticExpiryDateChangeDao` — same.
- `CodeBasedPromotionCollectionStaticExpiryDateChangeDao` — same.
- All `ExpiryDateChangeProcessor` implementations — interface and impls unchanged.
- All MongoDB collection indexes — no changes.
- RMQ queue/exchange/binding declarations — no changes.
- `PromotionFacade.updatePromotion()` duplicate-job protection — already updated via WAITING addition to `isJobRunningForPromotion()`.
- Pre-existing `ExpiryDateChangeJobFacadeTest` missing `@Mock ExpiryDateChangeJobDao` — not fixed here.

### Deferred
- Admin API endpoint for manual job replay (reset job to OPEN, re-publish to queue) — separate PR.
- Observability dashboard for semaphore counter.
- Automated NR alert on `EXPIRY_JOB_RETRY_EXHAUSTED_EVENT` — configure post-deploy.

---

## 4. Assumptions

| # | Assumption | Breaks if… | Owner to validate |
|---|-----------|-------------|------------------|
| 1 | `EXPIRY_DATE_CHANGE_BATCH_SIZE=2000` in prod | Larger value → $in list exceeds memory budget or query plan changes | Saravanan — confirm prod env var |
| 2 | `"cacheEvictionRedisTemplate"` bean is always present when app starts | Bean removed or renamed → NPE on injection | Implementer — verified in `RedisCacheUtil.java:204` ✅ |
| 3 | `RabbitListenerContainerFactoryConfigurer` is auto-configured by Spring Boot AMQP starter | Starter version doesn't provide it → must manually wire retry | Implementer — verify on Spring AMQP version in pom.xml |
| 4 | Redis Sentinel is reachable from all pods | Redis outage → semaphore acquire fails; fail-open behaviour required | Implementer — confirm with infra team |
| 5 | In USHC, orgs are distributed across multiple MongoDB shards | Single-shard assumption in reconciliation → wrong counter | Confirmed multi-shard in USHC per `.context/infra.md` |
| 6 | `@Scheduled(fixedRate=1h)` in `ExpiryDateChangeHelper` runs on all pods | Scheduler disabled on some pods → reconciliation not triggered | Implementer — verify `@EnableScheduling` scope |

---

## 5. Design Validation

### Gaps vs Existing Patterns

**GAP 1 [BLOCKER] — Redis bean qualifier**
- Proposed: `@Autowired RedisTemplate<String, String> redisTemplate` (no qualifier)
- Required: `@Autowired @Qualifier("cacheEvictionRedisTemplate") RedisTemplate<String, String> redisTemplate`
- Evidence: `RedisCacheUtil.java:204` — only one `RedisTemplate<String,String>` bean exists, named `"cacheEvictionRedisTemplate"`. `CacheEvictionPublisher.java:28-29` shows the correct pattern.
- Resolution: Add `@Qualifier("cacheEvictionRedisTemplate")` on the injected field in `ExpiryDateChangeJobFacade`.

**GAP 2 [HIGH] — Shard-aware semaphore reconciliation**
- Proposed: Reconcile counter inside `ExpiryDateChangeJobService.markAllJobAsExpiredIfNotRunning()`
- Problem: `ExpiryDateChangeHelper.updateExpiryDateChangeJobs()` calls this method inside a per-shard loop (`ExpiryDateChangeHelper.java:37-44`) with MDC shard key set. `countByStatus(RUNNING)` inside the method executes against only the current shard. Counter is reset to last shard's count — incorrect in multi-shard deployments (USHC).
- Resolution: Move reconciliation to `ExpiryDateChangeHelper.updateExpiryDateChangeJobs()` AFTER the shard loop. Sum `countByStatus(RUNNING)` across all shards, then call a new `ExpiryDateChangeJobFacade.reconcileSemaphoreCounter(long totalRunning)` method that sets the Redis counter. This keeps Redis dependency out of the helper.

**GAP 3 [MEDIUM] — Dedicated factory retry interceptor**
- Proposed: Custom factory inherits global retry policy
- Problem: `grep -rn RabbitListenerContainerFactoryConfigurer` returns no results in the codebase. Spring Boot wires retry interceptor only to the auto-configured default factory. Custom `@Bean("expiryDateChangeContainerFactory")` gets no retry by default.
- Resolution: Inject `RabbitListenerContainerFactoryConfigurer configurer` into `SpringAmqpConfig.expiryDateChangeContainerFactory(...)` and call `configurer.configure(factory, connectionFactory)` first, then set custom overrides (`setConcurrentConsumers`, `setMaxConcurrentConsumers`, `setPrefetchCount`). This is the Spring Boot-idiomatic approach.

**GAP 4 [MEDIUM] — Redis failure resilience**
- Proposed: `tryAcquireSemaphore` throws if Redis unavailable
- Problem: Redis outage → all expiry jobs fail retry loop → messages dead-lettered after 10 attempts (~17 min window). 600 queued messages would be permanently lost.
- Resolution: Wrap Redis operations in `tryAcquireSemaphore` with `try/catch`. On exception, log WARN and return `true` (fail-open). Jobs process normally; MongoDB concurrency control is temporarily lost, but no data loss.

**GAP 5 [LOW] — Stale `ObjectId endId` variable**
- `AbstractExpiryDateChangeDao.java:60`: `ObjectId endId;` is declared before the loop.
- Must be removed in the new implementation. `batchIds` is extracted inline from `mappedResults`.

### MADR Compliance
MADR 0001 (PromotionMeta two-tier cache) — not affected. This change does not touch
`PromotionMetaCacheService`, `CacheEvictionPublisher`, or any cache keys. ✅

### Guardrail Compliance (`.context/code.md`)

| Guardrail | Check | Result |
|---|---|---|
| **GC Health — pre-size collections** | `List<ObjectId> batchIds` cast from MongoDB `BasicDBList` (already sized) | PASS |
| **GC Health — no unbounded static maps** | Semaphore uses a single Redis counter key, not a local collection | PASS |
| **Caching — Redis via `@Qualifier`** | `"cacheEvictionRedisTemplate"` required | FAIL → fixed by Gap 1 |
| **Observability — include `orgId` in logs** | `log.info("Semaphore acquired jobId={}")` lacks orgId | FAIL → add `orgId` to semaphore log lines |
| **MongoDB — `lastUpdatedOn` must be set on every write** | `update.set(LAST_UPDATED_ON, now)` unchanged in both DAOs | PASS |
| **MongoDB — cross-org queries never issued** | `$in(batchIds)` uses pre-filtered IDs; `orgId` added as defensive filter | PASS |
| **Logging — no INFO in per-request hot paths** | Semaphore logs are per-job (not per-request); INFO is acceptable | PASS |

---

## 6. Alternative Designs Considered

| Alternative | Pros | Cons | Decision |
|------------|------|------|----------|
| **Keep `group().last()`, extract all IDs with a separate projection query** | Familiar aggregation shape | Two aggregation calls per batch = 2× index reads | Rejected — more expensive |
| **No `group()`, use `match+sort+limit+project(_id)` returning 1000 slim docs** | No in-memory accumulation; slightly lower memory | Java must iterate 1000 result maps to collect IDs; slightly more code change | Deferred — valid but adds more diff without meaningful gain |
| **Redisson RSemaphore instead of INCR/DECR** | Built-in fairness, TTL, lease renewal | New dependency (`redisson`), more complex config | Rejected — `spring-data-redis` already present; INCR/DECR is sufficient |
| **MongoDB-based semaphore (count RUNNING docs before processing)** | No Redis dependency | MongoDB I/O per job start; cross-shard coordination complex; not atomic | Rejected — Redis is faster and already available |

---

## 7. Use Cases

### B2B (Brand Admin Flows)

| ID | Flow | Today | After Fix | Risk | Test Type |
|----|------|-------|-----------|------|-----------|
| UC-1 | Brand updates 1 promotion end date → 2 jobs (ISSUED + EARNED) queued → processed → `impactedCount` matches actual documents | Range scan on `_id`, extra index traversal | `$in` point lookups; same correctness, lower IOPS | Wrong `impactedCount` if batchIds extraction fails | Integration |
| UC-2 | Brand updates 300 promotions → 600 messages → all 6 pods process concurrently | All 6 pods run simultaneously → MongoDB spike | Semaphore caps active jobs to `maxGlobalConcurrentJobs`; others retry | Deadlock if maxJobs=0 (misconfiguration) | Manual / load test |
| UC-3 | Brand updates same promotion end date twice while first job is OPEN | Second update blocked with `UPDATE_EXPIRY_DATE_NOT_ALLOWED` | Behaviour unchanged | None | Unit (existing) |
| UC-4 | Pod crashes mid-job (semaphore counter inflated by 1) | N/A | Counter healed by `ExpiryDateChangeHelper` reconciliation after ≤1 hour | If reconciliation doesn't run (scheduler off), counter stays inflated | Integration |
| UC-5 | `maxGlobalConcurrentJobs` slots full; new job arrives | N/A | `SemaphoreRejectedException` thrown; job transitions OPEN→WAITING (updates `lastUpdatedOn`); RMQ retry with exponential backoff; eventually acquires slot and processes | If max-attempts exhausted before slot opens → MessageRecoverer fires once → ERROR log + NR event; message lost | Unit |
| UC-6 | Promotion has 0 documents in the collection (new promotion with no issuances) | Loop exits immediately (empty `mappedResults`) | Same — `CollectionUtils.isEmpty(batchIds)` exits loop | None | Unit |
| UC-7 | Multi-shard env (USHC): pods across shards running jobs | N/A | Semaphore counter is global (Redis is shared); cross-shard job count correctly limited | If Redis connection routing differs per shard — verify same Redis instance | Manual in USHC env |
| UC-10 | Job stuck in WAITING for > 2h (pod died mid-retry; retries never resume) | N/A | `markAllJobAsExpiredIfNotRunning()` scheduler (hourly) expires WAITING jobs with `lastUpdatedOn > 2h ago` | If threshold is too short → active retries falsely expired. 2h >> 17-min retry window; false positive impossible. | Unit |
| UC-11 | Second `updatePromotion()` API call for same promotion while first job is WAITING (retrying) | API allowed: duplicate job created | API blocked: `isJobRunningForPromotion()` finds existing WAITING job → `UPDATE_EXPIRY_DATE_NOT_ALLOWED` | None — WAITING added to guard list | Unit |
| UC-12 | All 10 retry attempts exhausted for a job (semaphore always full) | N/A | `MessageRecoverer.recover()` fires once; ERROR logged with jobId; NR `EXPIRY_JOB_RETRY_EXHAUSTED_EVENT` emitted | If NR alert not configured → ops miss it until next batch review | AT-DEV (manual replay) |

### B2C (End Customer Flows)

| ID | Flow | Today | After Fix | Risk | Test Type |
|----|------|-------|-----------|------|-----------|
| UC-8 | Customer queries active promotions during job execution → sees old `validTill` for documents not yet processed in the current batch | Batch in progress; some docs updated, some not | Same eventual consistency window; no change | Customer sees stale expiry until batch reaches their document | N/A (acceptable) |
| UC-9 | Customer queries active promotions AFTER job completes → sees updated `validTill` | Correct | Correct — `impactedCount` on job equals number of documents updated | Wrong if `$in` criteria is mis-formed | Integration |

---

## 8. Low-Level Design

### 8a. `AbstractExpiryDateChangeDao.java`

```
Class: AbstractExpiryDateChangeDao  [dao/AbstractExpiryDateChangeDao.java]
Existing responsibility: Cursor-paginated bulk updateMulti loop for all ExpiryDateChange DAOs
Change: Aggregation returns all IDs in batch; updateMulti uses $in on primary key

Constants:
  REMOVE: public static final String END_ID = "last_id";
  ADD:    public static final String BATCH_IDS = "ids";

+ getAggregation(ExpiryDateChangeJob job): Aggregation
    Pipeline:
      match: _id > job.startId, promotionId = job.promotionId, orgId = job.orgId
      sort:  _id ASC  ← REMOVE (redundant; covered index already sorted)
      limit: batchSize
      group: push(_id) as "ids"            ← was last(_id) as "last_id"
      project: "ids"                       ← was "last_id"
    DB: IXSCAN on {orgId, promotionId, _id} — covered query, zero heap I/O
    Guardrail: PASS — index-covered, no WiredTiger doc pages loaded

+ updateValidTill(ExpiryDateChangeJob job, PromotionMeta meta): void
    Loop change at line 67:
      REMOVE: ObjectId endId;  [line 60 — stale declaration]
      REMOVE: endId = (ObjectId) mappedResults.get(0).get(END_ID);
      ADD:    List<ObjectId> batchIds = (List<ObjectId>) mappedResults.get(0).get(BATCH_IDS);
              if (CollectionUtils.isEmpty(batchIds)) break;
      REMOVE: Long count = updateValidTillDate(job, meta, endId, zoneOffset);
      ADD:    Long count = updateValidTillDate(job, meta, batchIds, zoneOffset);
      CHANGE: expiryDateChangeJob.setStartId(endId)
           → expiryDateChangeJob.setStartId(batchIds.get(batchIds.size() - 1));
    Guardrail: PASS

+ updateValidTillDate(ExpiryDateChangeJob job, PromotionMeta meta,
                       List<ObjectId> batchIds, ZoneOffset zoneOffset): Long  [was ObjectId endId]
    Criteria change:
      REMOVE: andOperator(orgId.is, promotionId.is, _id.gt(startId), _id.lte(endId))
      ADD:    Criteria.where(ID_COLUMN).in(batchIds)
              .and(ORG_ID_COLUMN).is(job.getOrgId())   ← defensive, zero I/O cost
    DB: updateMulti — _id $in (batchIds), orgId = Y
        Uses primary _id index; O(batchSize) point lookups
    Emits: log.info("Time taken to update {} documents is {}", count, timeTakenToUpdate)  [unchanged]
    Guardrail: PASS
```

### 8b. `CustomerEarnedCollectionDynamicExpiryDateChangeDao.java`

```
Class: CustomerEarnedCollectionDynamicExpiryDateChangeDao
Existing responsibility: Override updateValidTillDate() with AggregationUpdate for dynamic validity
Change: Signature change + Criteria change only; AggregationUpdate pipeline is UNCHANGED

+ updateValidTillDate(ExpiryDateChangeJob job, PromotionMeta meta,
                       List<ObjectId> batchIds, ZoneOffset zoneOffset): Long  [was ObjectId endId]
    Criteria change (lines 55-59):
      REMOVE: andOperator(orgId.is, promotionId.is, _id.gt(startId), _id.lte(endId))
      ADD:    Criteria.where(ID_COLUMN).in(batchIds)
              .and(ORG_ID_COLUMN).is(job.getOrgId())
    Rest of AggregationUpdate (DateFromParts + ArithmeticOperators.Add) is UNCHANGED
    DB: updateMulti with AggregationUpdate pipeline on _id $in (batchIds)
    Guardrail: PASS
```

### 8c. `SpringAmqpConfig.java`

```
Class: SpringAmqpConfig  [queue/SpringAmqpConfig.java]
Existing responsibility: Queue, exchange, binding bean declarations
Change: Add named container factory for PROMOTION_EXPIRY_UPDATE_QUEUE

Purpose: PROTECTIVE ISOLATION — not tuning. RabbitMQ is shared with redemption and other
flows whose future throughput demands may push the global prefetch/consumers settings higher.
This factory hard-pins the expiry queue at prefetch=1 / consumers=1 so it is never affected
by global broker config changes.

New @Value fields in SpringAmqpConfig (required for Fix-B custom retry interceptor):
  @Value("${spring.rabbitmq.listener.simple.retry.max-attempts:${RABBITMQ_RETRY_MAX_ATTEMPTS:10}}")
  private int retryMaxAttempts;
  @Value("${spring.rabbitmq.listener.simple.retry.initial-interval:${RABBITMQ_RETRY_INITIAL_INTERVAL_MS:2000}}")
  private long retryInitialInterval;
  @Value("${spring.rabbitmq.listener.simple.retry.multiplier:${RABBITMQ_RETRY_MULTIPLIER:2.0}}")
  private double retryMultiplier;
  @Value("${spring.rabbitmq.listener.simple.retry.max-interval:${RABBITMQ_RETRY_MAX_INTERVAL_MS:600000}}")
  private long retryMaxInterval;

+ expiryDateChangeContainerFactory(
      ConnectionFactory connectionFactory,
      RabbitListenerContainerFactoryConfigurer configurer
  ): SimpleRabbitListenerContainerFactory
    @Bean("expiryDateChangeContainerFactory")
    Steps:
      1. factory = new SimpleRabbitListenerContainerFactory()
      2. configurer.configure(factory, connectionFactory)
           ← MUST be called first — inherits message converter and error handler.
             No RabbitListenerContainerFactoryConfigurer usage existed anywhere in the codebase
             before this. Spring Boot 2.6.6 / Spring AMQP 2.4.x — configurer auto-configured ✅
      3. Build custom StatefulRetryOperationsInterceptor with Fix-B MessageRecoverer:
           retryInterceptor = RetryInterceptorBuilder.stateful()
               .maxAttempts(retryMaxAttempts)
               .backOffOptions(retryInitialInterval, retryMultiplier, retryMaxInterval)
               .recoverer((message, cause) -> {           ← Fix-B: fires ONCE on exhaustion
                   log.error("Expiry job exhausted all {} retry attempts — message discarded. "
                             + "cause={}", retryMaxAttempts, cause.getMessage(), cause);
                   metricsService.markAsFailedRequest(EXPIRY_JOB_RETRY_EXHAUSTED_NR_EVENT);
               })
               .build();
           factory.setAdviceChain(retryInterceptor);      ← replaces chain set by configurer;
                                                            message converter & error handler
                                                            from step 2 are unaffected
      4. factory.setConcurrentConsumers(1)    ← hard-pinned
      5. factory.setMaxConcurrentConsumers(1) ← prevents dynamic scale-up
      6. factory.setPrefetchCount(1)          ← hard-pinned
    New constant:
      private static final String EXPIRY_JOB_RETRY_EXHAUSTED_NR_EVENT =
          NewrelicConstants.EXPIRY_JOB_RETRY_EXHAUSTED_EVENT;  ← add to NewrelicConstants
    Guardrail: PASS
```

### 8d. `ExpiryDateChangeJobListener.java`

```
Class: ExpiryDateChangeJobListener  [queue/ExpiryDateChangeJobListener.java]
Change: One-line — add containerFactory to @RabbitListener

@RabbitListener(
    queues = {SpringAmqpConfig.PROMOTION_EXPIRY_UPDATE_QUEUE},
    containerFactory = "expiryDateChangeContainerFactory")   ← ADD
```

### 8e. `ExpiryDateChangeJobFacade.java`

```
Class: ExpiryDateChangeJobFacade  [service/impl/ExpiryDateChange/ExpiryDateChangeJobFacade.java]
Existing responsibility: Orchestrate job lifecycle (find → RUNNING → process → COMPLETED/ERRORED)
Change: Add Redis semaphore acquire/release wrapping the processing block

New fields:
  private static final String SEMAPHORE_KEY = "promotion_engine:expiry_job:running_count";

  @Autowired
  @Qualifier("cacheEvictionRedisTemplate")          ← REQUIRED — see Gap 1
  private RedisTemplate<String, String> redisTemplate;

  @Value("${expiry.date.change.max.global.concurrent.jobs:3}")
  private int maxGlobalConcurrentJobs;

+ tryAcquireSemaphore(String jobId, Long orgId): boolean
    try {
        int effectiveMax = Math.max(1, maxGlobalConcurrentJobs);   ← EC-4 clamp
        if (effectiveMax != maxGlobalConcurrentJobs) {
            log.warn("expiry.date.change.max.global.concurrent.jobs={} is invalid (<1); "
                     + "clamped to 1 — correct EXPIRY_MAX_GLOBAL_CONCURRENT_JOBS env var",
                     maxGlobalConcurrentJobs);
        }
        Long current = redisTemplate.opsForValue().increment(SEMAPHORE_KEY);
        if (current != null && current <= effectiveMax) {
            log.info("Semaphore acquired orgId={} jobId={} count={}/{}", orgId, jobId, current, effectiveMax);
            return true;
        }
        redisTemplate.opsForValue().decrement(SEMAPHORE_KEY);
        log.info("Semaphore limit hit orgId={} jobId={} count={}/{}", orgId, jobId, current, effectiveMax);
        return false;
    } catch (Exception e) {
        log.warn("Semaphore Redis error orgId={} jobId={} — failing open", orgId, jobId, e);
        return true;   ← fail-open: jobs process normally if Redis unavailable
    }
    Throws: never — exceptions caught internally
    Rationale for clamp (not @PostConstruct throw): stopping the pod means zero expiry jobs
    run until env var is fixed and pod restarted — worse than running at effectiveMax=1.
    Clamp keeps the system functional at safest concurrency and surfaces via WARN log.
    Guardrail: PASS — orgId in log line per .context/code.md

+ releaseSemaphore(String jobId, Long orgId): void
    try {
        Long remaining = redisTemplate.opsForValue().decrement(SEMAPHORE_KEY);
        log.info("Semaphore released orgId={} jobId={} remaining={}", orgId, jobId, remaining);
    } catch (Exception e) {
        log.warn("Semaphore release Redis error orgId={} jobId={}", orgId, jobId, e);
    }
    Throws: never

+ reconcileSemaphoreCounter(long totalRunning): void         ← called by ExpiryDateChangeHelper
    try {
        long safe = Math.max(0, totalRunning);
        redisTemplate.opsForValue().set(SEMAPHORE_KEY, String.valueOf(safe));
        log.info("Semaphore reconciled to actual running count: {}", safe);
    } catch (Exception e) {
        log.warn("Semaphore reconcile Redis error count={}", totalRunning, e);
    }
    Throws: never

+ shouldProceed(ExpiryDateChangeJob job): boolean
    BEFORE:
      return job.getStatus().equals(BatchStatus.OPEN)
          || job.getStatus().equals(BatchStatus.ERRORED);
    AFTER:
      return job.getStatus().equals(BatchStatus.OPEN)
          || job.getStatus().equals(BatchStatus.ERRORED)
          || job.getStatus().equals(BatchStatus.WAITING);   ← NEW — allow retry from WAITING
    Rationale: Without this, a WAITING job is silently dropped on the next retry attempt
    (shouldProceed returns false → early return → message consumed, job never runs).

Modified changeExpiryForPromotion():
  ExpiryDateChangeJob job = expiryDateChangeJobService.findById(...)
  if (!shouldProceed(job)) return;                          ← now allows WAITING
  if (!tryAcquireSemaphore(jobId, job.getOrgId())) {
      // Transition to WAITING: updates lastUpdatedOn to NOW, resetting the 24h expiry clock.
      // On every retry that again hits the semaphore limit, WAITING is set again (clock refreshed).
      // A job actively retrying can NEVER be expired by markAllJobAsExpiredIfNotRunning().
      expiryDateChangeJobService.updateExpiryDateChangeJobStatus(job, BatchStatus.WAITING);
      throw new SemaphoreRejectedException(              ← Fix-A: not generic RuntimeException
          "Expiry job concurrency limit reached orgId=" + job.getOrgId()
          + " jobId=" + jobId);                          ← RMQ retry kicks in
  }
  try {
      ... existing logic unchanged ...
  } catch (Exception e) {
      ... existing catch unchanged ...
      throw e;
  } finally {
      releaseSemaphore(jobId, job.getOrgId());              ← always releases acquired slot
  }
  Guardrail: PASS
```

### 8f. `ExpiryDateChangeHelper.java`

```
Class: ExpiryDateChangeHelper  [util/ExpiryDateChangeHelper.java]
Existing responsibility: @Scheduled job to expire stale OPEN/RUNNING jobs (every 1 hour, all shards)
Change: After shard loop, sum RUNNING counts across shards and reconcile semaphore counter

New field:
  @Autowired
  private ExpiryDateChangeJobFacade expiryDateChangeJobFacade;

Modified updateExpiryDateChangeJobs():
  // Existing shard loop — UNCHANGED:
  for (String shardKey : allShardKeys) {
      MDC.put(SHARD_KEY, shardKey);
      try { expiryDateChangeJobService.markAllJobAsExpiredIfNotRunning(); }
      finally { MDC.remove(SHARD_KEY); }
  }
  // NEW — reconcile after all shards processed:
  long totalRunning = 0;
  for (String shardKey : allShardKeys) {
      MDC.put(SHARD_KEY, shardKey);
      try { totalRunning += expiryDateChangeJobDao.countByStatus(BatchStatus.RUNNING); }
      finally { MDC.remove(SHARD_KEY); }
  }
  expiryDateChangeJobFacade.reconcileSemaphoreCounter(totalRunning);

New field:
  @Autowired
  private ExpiryDateChangeJobDao expiryDateChangeJobDao;

Guardrail: PASS — orgId not available here (cross-org), reconcile log is sufficient
```

### 8g. `ExpiryDateChangeJobDao.java`

```
Interface: ExpiryDateChangeJobDao  [dao/ExpiryDateChangeJobDao.java]
Change: Add one derived query method

+ countByStatus(BatchStatus status): long
  Spring Data derived query — uses @CompoundIndex(def = "{'status' : 1}") on ExpiryDateChangeJob
  → index-covered count query, no heap I/O
  No custom @Query needed — Spring Data derives: db.expiryDateChangeJob.countDocuments({status: ?})
```

### 8h. `application.properties`

```
# Expiry date change job — cross-pod concurrency throttle
# Default 3 = half of 6 pods. Raises to 6 to disable throttle; lower to 2 only with
# retry window review (10 attempts × exponential backoff = ~17 min total window).
expiry.date.change.max.global.concurrent.jobs=${EXPIRY_MAX_GLOBAL_CONCURRENT_JOBS:3}

# NOTE: No per-queue prefetch/consumers properties.
# The dedicated factory hard-pins concurrentConsumers=1 and prefetchCount=1.
# These are fixed, not operator-tunable — the factory's purpose is protective isolation
# from global broker setting changes, not flexibility.
```

### 8i. `BatchStatus.java`

```
Enum: BatchStatus  [bo/reminder/BatchStatus.java]
Change: Add WAITING value

BEFORE: OPEN, RUNNING, ERRORED, COMPLETED, EXPIRED, STOPPED
AFTER:  OPEN, RUNNING, ERRORED, WAITING, COMPLETED, EXPIRED, STOPPED
                               ↑ NEW — placed between ERRORED and COMPLETED for logical flow

Purpose: WAITING signals that a job has been throttled by the semaphore and is
pending retry. It is NOT a terminal state — the job will resume when a semaphore
slot becomes available. Key invariants:
  - shouldProceed() allows WAITING (retry can proceed)
  - isJobRunningForPromotion() includes WAITING (prevents duplicate job creation)
  - markAllJobAsExpiredIfNotRunning() expires WAITING jobs older than 2h (orphan cleanup)
  - lastUpdatedOn is refreshed on EVERY transition to WAITING (resets the expiry clock)
```

### 8j. `SemaphoreRejectedException.java`

```
Class: SemaphoreRejectedException  [NEW — service/impl/ExpiryDateChange/SemaphoreRejectedException.java]
Extends: RuntimeException
Purpose: Thrown by ExpiryDateChangeJobFacade.changeExpiryForPromotion() when the semaphore
         limit is hit. Distinct from processing exceptions so RMQMessageTrackerAspect can
         skip counting throttle events as NR failures.

public class SemaphoreRejectedException extends RuntimeException {
    public SemaphoreRejectedException(String message) {
        super(message);
    }
}

Guardrail: PASS — no PII; message contains only orgId + jobId (operational identifiers)
```

### 8k. `RMQMessageTrackerAspect.java`

```
Class: RMQMessageTrackerAspect  [aspect/RMQMessageTrackerAspect.java]
Change: Skip markAsFailedRequest() for SemaphoreRejectedException (Fix-A)

Problem: Aspect fires markAsFailedRequest() on ANY exception (lines 77-81).
With max=3 concurrent jobs and 600 queued messages, each semaphore rejection
fires this → up to 3 × 10 = 30 spurious NR failure events per batch. This drowns
real failures in NR dashboards and inflates failure-rate alerts.

Fix: Before calling markAsFailedRequest(), check for SemaphoreRejectedException.
Spring AMQP wraps listener exceptions in ListenerExecutionFailedException, so
both the direct exception and its cause must be checked:

ADD before the existing markAsFailedRequest() call:
  Throwable root = throwable.getCause() != null ? throwable.getCause() : throwable;
  if (root instanceof SemaphoreRejectedException) {
      log.debug("Semaphore rejection — not counted as NR failure: {}", root.getMessage());
      return;
  }
  // existing: metricsService.markAsFailedRequest(...)

Guardrail: PASS — no behavioural change for real failures; only throttle events are filtered
```

### 8l. `ExpiryDateChangeJobService.java`

```
Class: ExpiryDateChangeJobService  [service/impl/ExpiryDateChange/ExpiryDateChangeJobService.java]
Change 1: markAllJobAsExpiredIfNotRunning() — add 2h WAITING orphan expiry
Change 2: isJobRunningForPromotion() — add WAITING to status guard

Change 1 — markAllJobAsExpiredIfNotRunning():
  Existing loop (lines 113-127) is UNCHANGED — expires OPEN/RUNNING jobs after 24h.
  ADD after existing loop:

  // Expire orphaned WAITING jobs.
  // WAITING.lastUpdatedOn is refreshed on every semaphore retry attempt.
  // hoursDiff > 2 means no retry for 2h → pod died → orphaned → safe to expire.
  // 2h >> 17-min retry window; no active retry can trigger this threshold.
  List<ExpiryDateChangeJob> waitingJobs =
      expiryDateChangeJobDao.findByStatusIn(Lists.newArrayList(BatchStatus.WAITING));
  waitingJobs.forEach(job -> {
      Date lastUpdatedOn = job.getAttribution().getLastUpdatedOn();
      Date now = new Date();
      long hoursDiff = TimeUnit.HOURS.convert(
          Math.abs(now.getTime() - lastUpdatedOn.getTime()), TimeUnit.MILLISECONDS);
      if (hoursDiff > 2) {
          job.setStatus(BatchStatus.EXPIRED);
          job.getAttribution().setLastUpdatedOn(now);
          expiryDateChangeJobDao.save(job);
          log.info("Expired orphaned WAITING job orgId={} jobId={} lastUpdatedOn={}",
                   job.getOrgId(), job.getId(), lastUpdatedOn);
      }
  });

Change 2 — isJobRunningForPromotion() (lines 96-102):
  BEFORE:
    Lists.newArrayList(BatchStatus.OPEN, BatchStatus.RUNNING)
  AFTER:
    Lists.newArrayList(BatchStatus.OPEN, BatchStatus.RUNNING, BatchStatus.WAITING)

  Rationale: A job in WAITING is actively retrying (or orphaned). In either case,
  a second updatePromotion() call for the same promotion must be blocked to prevent
  a duplicate job. Without this, the WAITING job is invisible to the API guard and
  a second job is created, causing double processing of the same documents.

Guardrail: PASS — existing {status:1} index on ExpiryDateChangeJob covers both queries
```

---

## 9. Data Model Changes

No changes to any MongoDB collection schema, documents, or indexes.

`ExpiryDateChangeJob.java` BO unchanged. `ExpiryDateChangeJobDao.java` gets a new derived
method (`countByStatus`) — this is query-only, no schema change.

---

## 10. API Changes

No API changes. This is a pure internal background job processing path.

---

## 11. Security

No new attack surface is introduced. This change is entirely internal to background job
processing — no new API endpoints, no new user input surfaces, no auth/role check changes.

| Check | Finding |
|-------|---------|
| New API endpoints requiring auth/role checks | **None** |
| PII in new log lines | **None** — `orgId` and `jobId` are operational identifiers (not personal data); semaphore count is a plain integer |
| New fields storing PII | **None** — Redis key stores a plain integer counter; `batchIds` are MongoDB internal ObjectIds |
| Token / credential handling | **None** — Redis connection uses existing Lettuce client configuration; no new credentials introduced |
| Tenant data leak risk | **Low / Acceptable** — semaphore counter is intentionally global (not per-org); it limits total concurrency, not per-tenant throughput. `$in(batchIds)` IDs are pre-filtered by `orgId` in the aggregation phase — no cross-tenant document access is possible |
| User-supplied input in new code paths | **None** — `batchIds` are produced internally by the aggregation pipeline; `maxGlobalConcurrentJobs` is a server-side env var, not user input |

---

## 12. DB & Infra Considerations

```
CONSIDERATION [INFRA] [ACTION]
Finding: Dedicated container factory adds one additional AMQP connection channel per pod.
Impact: Negligible — shared RabbitMQ server; 6 pods × 1 channel = 6 additional channels.
Action: None required.
Owner: tech-detailer

CONSIDERATION [INFRA] [ACTION]
Finding: Redis semaphore key "promotion_engine:expiry_job:running_count" has no TTL.
  A Redis flush or expiry would reset it to 0 — this is the DESIRED behaviour (counter resets).
  Key persists across pod restarts since Redis is a shared Sentinel cluster.
Impact: None. Counter drift is bounded to pod crash window; reconciled every 1 hour.
Action: Do NOT set a TTL on this key. Document in runbook.
Owner: tech-detailer

CONSIDERATION [INFRA] [NOTE]
Finding: reconcileSemaphoreCounter runs on ALL pods (scheduler on all pods).
  Multiple pods may call reconcile concurrently every hour.
  Race: pod A reads countByStatus=2 and writes 2; pod B reads countByStatus=2 and writes 2.
  Since both read the same actual count and both write the same value, the race is benign —
  no corruption, no inflated value.
Impact: None (idempotent write of the same value).
Action: No mitigation needed.
Owner: tech-detailer

CONSIDERATION [DB] [NOTE]
Finding: countByStatus(RUNNING) is a new query issued once per shard per hour.
  Uses the {status:1} compound index on ExpiryDateChangeJob — index-covered count.
  Zero document heap I/O. Negligible MongoDB load.
Impact: None.
Owner: tech-detailer
```

---

## 13. Internal Architecture Changes

**New dependency introduced:** `ExpiryDateChangeHelper` now depends on `ExpiryDateChangeJobFacade`.
Check for circular dependency: `ExpiryDateChangeHelper → ExpiryDateChangeJobFacade → ExpiryDateChangeFactory → processors → DAOs → ExpiryDateChangeJobDao`. No cycle back to `ExpiryDateChangeHelper`. ✅

**New pattern introduced:**
- Redis atomic counter semaphore (`INCR/DECR` on a named key) for cross-pod rate control.
- Fail-open Redis resilience: catch Redis exceptions → return `true` to avoid blocking on Redis unavailability.
- Factory configurer pattern (`RabbitListenerContainerFactoryConfigurer.configure()`) for inheriting Spring Boot AMQP defaults on custom factories. **This should be the repo convention** for any future custom RabbitMQ listener factories.

---

## 14. Upstream / Downstream Impact

### Upstream
- RabbitMQ `PROMOTION_EXPIRY_UPDATE_QUEUE` — no schema change. Queue name unchanged. Message structure unchanged.

### Downstream
- MongoDB `customerIssuedPromotion`, `customerEarnedPromotion`, `codeBasedPromotion` — same documents updated, same fields set. `impactedCount` on `ExpiryDateChangeJob` is unchanged in meaning.
- Redis `incentives` cluster — new key `"promotion_engine:expiry_job:running_count"` written by all pods. Low-frequency writes (1 INCR per job start, 1 DECR per job end, 1 SET per shard-loop reconciliation per hour).

---

## 15. SLA Impact

**Optimisation 1 (IN clause):** Lower MongoDB IOPS and CPU per job run. Bulk update wall-clock time should be same or lower. `impactedCount` is unchanged.

**Optimisation 2 (concurrency throttle):** Default `maxGlobalConcurrentJobs=3` (half of 6 pods) throttles from day 1 — peak MongoDB pressure is halved relative to today. A 300-promotion batch (600 messages) drains at 3 concurrent jobs; all messages complete within the ~17-minute retry window. Operator can raise `EXPIRY_MAX_GLOBAL_CONCURRENT_JOBS` to 6 to disable throttling if MongoDB headroom allows.

**Degraded mode:** If Redis is unavailable, semaphore fails open — all 6 pods process concurrently, same as today. No regression.

---

## 16. Observability

### New log lines
| Location | Level | Content |
|---|---|---|
| `ExpiryDateChangeJobFacade.tryAcquireSemaphore` | INFO | `"Semaphore acquired orgId={} jobId={} count={}/{}"` |
| `ExpiryDateChangeJobFacade.tryAcquireSemaphore` | INFO | `"Semaphore limit hit orgId={} jobId={} count={}/{}"` |
| `ExpiryDateChangeJobFacade.tryAcquireSemaphore` | WARN | `"Semaphore Redis error orgId={} jobId={} — failing open"` |
| `ExpiryDateChangeJobFacade.releaseSemaphore` | INFO | `"Semaphore released orgId={} jobId={} remaining={}"` |
| `ExpiryDateChangeJobFacade.reconcileSemaphoreCounter` | INFO | `"Semaphore reconciled to actual running count: {}"` |
| `ExpiryDateChangeJobFacade.changeExpiryForPromotion` | INFO | `"Job transitioned to WAITING orgId={} jobId={} — semaphore limit reached, will retry"` |
| `SpringAmqpConfig` MessageRecoverer | ERROR | `"Expiry job exhausted all {} retry attempts — message discarded. cause={}"` |
| `ExpiryDateChangeJobService.markAllJobAsExpiredIfNotRunning` | INFO | `"Expired orphaned WAITING job orgId={} jobId={} lastUpdatedOn={}"` |

### New Relic
- No new `@Trace` annotations required — the facade method is already traced via `@Profiled`.
- Recommended: add `metricsService.addCustomParameter("semaphoreCount", current)` in `tryAcquireSemaphore` to surface semaphore level in New Relic transaction traces.
- **New custom event:** `EXPIRY_JOB_RETRY_EXHAUSTED_EVENT` emitted by MessageRecoverer on retry exhaustion. Add `NewrelicConstants.EXPIRY_JOB_RETRY_EXHAUSTED_EVENT = "expiry_job_retry_exhausted"`. Configure a New Relic alert on this event post-deploy — if it fires, ops investigates and manually replays the job.

### Redis monitoring
- Key to watch: `promotion_engine:expiry_job:running_count`
- Expected: value stays ≤ `maxGlobalConcurrentJobs` during a batch run; returns to 0 after all jobs complete.

---

## 17. Rollout Plan

1. **No feature flag required** — both changes are internal; no API behaviour change.
2. **Deploy order:** Standard rolling deploy — no migration scripts, no schema changes.
3. **Env vars to set at deploy:**
   - `EXPIRY_MAX_GLOBAL_CONCURRENT_JOBS=3` (default; throttles from day 1)
   - No other new env vars — factory pins `prefetch=1` and `consumers=1` in code; not operator-configurable by design.
4. **Post-deploy validation:**
   - Watch Redis key `promotion_engine:expiry_job:running_count` during next brand batch update.
   - Confirm value stays ≤ 3 and returns to 0 after completion.
   - Confirm MongoDB `opcounters.update` per second is lower during job runs vs pre-deploy baseline.
   - Confirm `ExpiryDateChangeJob.impactedCount` on a sample job matches expected doc count.
5. **Throttle tuning (if MongoDB headroom is comfortable):** raise `EXPIRY_MAX_GLOBAL_CONCURRENT_JOBS` to 4–6 to increase throughput. Do not lower below 2 without reviewing the retry window — at limit=1 with a 5-minute job, the 10-attempt retry window (~17 min) may not be sufficient for queued messages.
6. **Rollback:** revert single commit. No data migration to undo. Redis counter drifts to 0 within 1 hour via reconciliation even without rollback-specific cleanup.

---

## 18. Risks

| # | Description | Likelihood | Mitigation |
|---|------------|-----------|-----------|
| 1 | `RabbitListenerContainerFactoryConfigurer` not available in project's Spring AMQP version → compile error | Low | Check `spring-amqp` version in `pom.xml`; fallback: manually configure `StatefulRetryOperationsInterceptor` |
| 2 | `configurer.configure(factory, connectionFactory)` sets `concurrentConsumers=1` from global config, then our override sets it again — order matters | Low | Our overrides are applied AFTER `configure()` — correct order per Spring docs |
| 3 | Circular dependency: `ExpiryDateChangeHelper` → `ExpiryDateChangeJobFacade` | Low | Verified: no cycle exists in the dependency graph |
| 4 | `countByStatus` Spring Data derived method name conflicts with existing method | Low | Check `ExpiryDateChangeJobDao` interface — no collision found |
| 5 | Semaphore counter drifts negative if `releaseSemaphore` called without prior acquire | Very Low | `finally` is only reached after `tryAcquireSemaphore` returns `true`; no acquire = no finally block |
| 6 | USHC multi-shard: reconciliation runs concurrently on multiple pods — count is summed before write, not atomic | Very Low | Race is benign (idempotent — both pods write same value derived from same actual count) |
| 8 | **EC-4: `maxGlobalConcurrentJobs=0` or negative — permanent deadlock without a guard** | **Low likelihood, HIGH impact** — a typo (`EXPIRY_MAX_GLOBAL_CONCURRENT_JOBS=0`) causes all jobs rejected forever; reconciliation writes 0 hourly, preserving the deadlock. | **Resolved by design:** `int effectiveMax = Math.max(1, maxGlobalConcurrentJobs)` inside `tryAcquireSemaphore`. Invalid config silently becomes 1 (safest concurrency). WARN logged so ops can correct the env var. Pod continues running — zero job processing (fail-fast) is explicitly worse than single-job processing (clamp). Test: UT-EC4 (see test plan). |
| 9 | **WAITING + `isJobRunningForPromotion()` gap — duplicate jobs during retry window** | **Medium likelihood, HIGH impact** — without WAITING in the status guard, a second `updatePromotion()` API call while the first job is WAITING creates a duplicate job, causing double document updates. | **Resolved by design:** WAITING added to `Lists.newArrayList(BatchStatus.OPEN, BatchStatus.RUNNING, BatchStatus.WAITING)` in `isJobRunningForPromotion()`. Test: UT-W3 (see test plan). |
| 10 | **Fix-B `setAdviceChain()` replaces retry chain set by `configurer.configure()`** | **Low likelihood** — `factory.setAdviceChain(retryInterceptor)` replaces the retry advice. If retry `@Value` props are wrong or missing in `SpringAmqpConfig`, the custom interceptor uses different retry settings than the rest of the app (silently). | **Required:** Inject all retry props via `@Value` in `SpringAmqpConfig` matching the exact property keys used in `application.properties`. Verify max-attempts and backoff values match global config on first deploy. |
| 11 | **WAITING job transitions WAITING → RUNNING while `updateExpiryDateChangeJobStatus(WAITING)` is in-flight** | **Very Low** — `updateExpiryDateChangeJobStatus(WAITING)` is called before throwing, and `releaseSemaphore()` is NOT called (no acquire was issued). If MongoDB save fails mid-throw, the job status is unchanged and the next retry proceeds from OPEN/ERRORED — correct. | No code change needed — the transition is a best-effort status update; failure here is safe. |
| 7 | **Redis Sentinel restart with no persistence (AOF/RDB disabled) — counter key lost mid-batch** | **Medium** — Confirmed with infra team that persistence status is unknown (see §19 Open Questions). If Redis restarts without AOF/RDB, the `running_count` key resets to 0. In-flight jobs (which already incremented) call `releaseSemaphore` (DECR) against a fresh key, driving it negative. Interleaved with new acquisitions, up to all 6 pods can run concurrently until the next hourly reconciliation — same load profile as today before this fix. Self-heals within ≤1 hour. | **(a) Confirm AOF status with infra team (§19).** If AOF is enabled, risk is negligible — key survives restart. **(b) Optional mitigation — seed counter on pod startup:** add `@EventListener(ApplicationReadyEvent.class)` that calls `reconcileSemaphoreCounter(countByStatus(RUNNING))` on each pod startup. Eliminates drift window regardless of AOF status. **(c) Accept as bounded fail-open** if infra confirms AOF is on all Sentinel nodes — behaviour is identical to the existing Redis-unavailable fail-open path. |

---

## 19. Open Questions

| Question | Owner | Due | Status |
|---------|-------|-----|--------|
| ~~Spring AMQP version — does it include `RabbitListenerContainerFactoryConfigurer`?~~ | — | — | **Resolved** — Spring Boot 2.6.6 → AMQP 2.4.x. Configurer is auto-configured since AMQP 2.0. ✅ |
| Should `sort` stage be explicitly removed from `getAggregation()` after change, or left as a no-op? | Saravanan | Before build | **Open** — recommend removing with an explanatory comment |
| ~~Initial production value for `EXPIRY_MAX_GLOBAL_CONCURRENT_JOBS`?~~ | — | — | **Resolved** — default set to **3**. Raise post-deploy if headroom allows. |
| Does the `incentives` Redis cluster key namespace collide with `"promotion_engine:expiry_job:running_count"`? | Infra team | Before deploy | **Open** |
| Is AOF (append-only file) or RDB persistence enabled on the `incentives` Redis Sentinel cluster? If not, a Redis restart loses the `running_count` key and causes a temporary fail-open until hourly reconciliation heals it (Risk #7). If yes, risk is negligible. | Infra team | Before deploy | **Open** |
| Should the WAITING job expiry threshold (2h) be made configurable via env var `EXPIRY_WAITING_JOB_EXPIRY_HOURS`? Currently hardcoded as `if (hoursDiff > 2)` in `markAllJobAsExpiredIfNotRunning()`. The value is derived from the retry window (~17 min); configurable only if per-environment fine-tuning is expected. | Tech lead | Before build | **Open** |
| NR alert on `EXPIRY_JOB_RETRY_EXHAUSTED_EVENT` — which team/channel should receive it, and what threshold (first occurrence = alert)? | Saravanan | Post-deploy | **Open** |

---

## 20. Questions for arch-investigator — Resolved

**Q1 — Semaphore reconciliation placement** ✅ Resolved
Reconciliation moved to `ExpiryDateChangeHelper.updateExpiryDateChangeJobs()` post-shard-loop
with cross-shard sum. `reconcileSemaphoreCounter(long totalRunning)` exposed on Facade.
Confirmed acceptable; design updated accordingly.

**Q2 — `"cacheEvictionRedisTemplate"` bean uniqueness** ✅ Resolved
Confirmed sole `RedisTemplate<String,String>` bean in codebase at `RedisCacheUtil.java:204`.
No unnamed bean exists. `@Qualifier("cacheEvictionRedisTemplate")` is the correct and only
injection pattern. No ambiguity risk.

---

## 21. Task Breakdown

> **Phased delivery:** Three independent PRs in recommended deploy order.
> PR-1 ships first for immediate MongoDB load relief. PR-2 and PR-3 can follow in any order.
> All three PRs are zero-dependency on each other — no file overlaps, no shared methods.

---

### PR-1 — Semaphore + WAITING + Fix-A
**Goal:** Immediate cross-pod concurrency throttle. Reduces peak concurrent MongoDB `updateMulti`
operations from 6 to 3. WAITING status prevents premature job expiry during the retry window.
Fix-A eliminates NR metric noise from semaphore throttle events.

**Known gap until PR-2 lands:** Retry exhaustion (all 10 attempts used) is still silently lost —
no NR event, no ERROR log. Acceptable if PR-2 follows quickly; document in PR description.

#### Backend — PR-1

| # | Task | File | Size | Depends on |
|---|------|------|------|-----------|
| B9 | Add `countByStatus(BatchStatus status): long` derived query | `ExpiryDateChangeJobDao.java` | S | — |
| B10 | Add env-var-backed property `expiry.date.change.max.global.concurrent.jobs=${EXPIRY_MAX_GLOBAL_CONCURRENT_JOBS:3}` | `application.properties` | S | — |
| B11 | Add `WAITING` enum value between `ERRORED` and `COMPLETED` | `BatchStatus.java` | S | — |
| B12 | Create `SemaphoreRejectedException extends RuntimeException` | `SemaphoreRejectedException.java` (new file) | S | — |
| B7 | Add `tryAcquireSemaphore()`, `releaseSemaphore()`, `reconcileSemaphoreCounter()` + `@Qualifier("cacheEvictionRedisTemplate") RedisTemplate` + `@Value` max jobs; wrap `changeExpiryForPromotion()` in try-finally | `ExpiryDateChangeJobFacade.java` | M | B9 |
| B13 | Update `shouldProceed()` to allow `BatchStatus.WAITING`; update `changeExpiryForPromotion()` to transition job → WAITING and throw `SemaphoreRejectedException` on semaphore rejection | `ExpiryDateChangeJobFacade.java` | S | B7, B11, B12 |
| B14 | Update `isJobRunningForPromotion()` to include `BatchStatus.WAITING` — prevents duplicate job creation during retry window | `ExpiryDateChangeJobService.java` | S | B11 |
| B15 | Add second loop to `markAllJobAsExpiredIfNotRunning()` expiring WAITING jobs with `lastUpdatedOn > 2h` | `ExpiryDateChangeJobService.java` | S | B11 |
| B16 | Skip `markAsFailedRequest()` in `RMQMessageTrackerAspect` when exception is (or wraps) `SemaphoreRejectedException` | `RMQMessageTrackerAspect.java` | S | B12 |
| B8 | Sum `countByStatus(RUNNING)` across all shards after shard loop; call `reconcileSemaphoreCounter(totalRunning)`; inject `ExpiryDateChangeJobFacade` + `ExpiryDateChangeJobDao` | `ExpiryDateChangeHelper.java` | S | B7, B9 |

#### Testing — PR-1

| # | Task | File | Size | Depends on |
|---|------|------|------|-----------|
| T2 | Unit tests for semaphore: acquire/release/limit-hit/fail-open; add `@Mock RedisTemplate<String,String>` | `ExpiryDateChangeJobFacadeTest.java` | M | B7 |
| T6 | Unit tests for WAITING state machine: `shouldProceed(WAITING)=true`, transition to WAITING on rejection, `SemaphoreRejectedException` thrown | `ExpiryDateChangeJobFacadeTest.java` | S | B13 |
| T3 | Unit test: `countByStatus` stub; verify reconcile NOT triggered inside `markAllJobAsExpiredIfNotRunning` (reconcile lives in Helper) | `ExpiryDateChangeJobServiceTest.java` | S | B9 |
| T7 | Unit tests: `isJobRunningForPromotion(WAITING)=true`; WAITING expired after 2h; NOT expired under 2h; OPEN NOT expired under 24h | `ExpiryDateChangeJobServiceTest.java` | S | B14, B15 |
| T8 | Unit tests: aspect skips NR metric for `SemaphoreRejectedException` (direct + wrapped); counts real exceptions | `RMQMessageTrackerAspectTest.java` | S | B16 |
| T4 | Unit test: `updateExpiryDateChangeJobs` calls `reconcileSemaphoreCounter` with correct cross-shard sum after loop | `ExpiryDateChangeHelperTest.java` | S | B8 |

---

### PR-2 — Container Factory + Fix-B
**Goal:** Protective isolation of `PROMOTION_EXPIRY_UPDATE_QUEUE` from future global broker
config changes. Fix-B adds retry-exhaustion observability (ERROR log + NR event).

**Why separate from PR-1:** The factory change has no effect on current behaviour (prefetch=1 and
consumers=1 are already the global defaults). Shipping it independently keeps PR-1 minimal and
reduces blast radius. The semaphore in PR-1 works correctly with or without the dedicated factory.

#### Backend — PR-2

| # | Task | File | Size | Depends on |
|---|------|------|------|-----------|
| B5 | Add `expiryDateChangeContainerFactory` bean: call `configurer.configure()` first, then hard-pin `concurrentConsumers=1`, `maxConcurrentConsumers=1`, `prefetchCount=1` | `SpringAmqpConfig.java` | S | — |
| B17 | Inject retry `@Value` props; build `StatefulRetryOperationsInterceptor` with Fix-B `MessageRecoverer` (ERROR log + `EXPIRY_JOB_RETRY_EXHAUSTED_EVENT`); call `factory.setAdviceChain()`; add constant to `NewrelicConstants` | `SpringAmqpConfig.java`, `NewrelicConstants.java` | M | B5 |
| B6 | Add `containerFactory = "expiryDateChangeContainerFactory"` to `@RabbitListener` | `ExpiryDateChangeJobListener.java` | S | B5, B17 |

#### Testing — PR-2

| # | Task | File | Size | Depends on |
|---|------|------|------|-----------|
| T9 | Unit tests: `configurer.configure()` called first (InOrder); factory hard-pins prefetch/consumers; `MessageRecoverer.recover()` calls `metricsService.markAsFailedRequest(EXPIRY_JOB_RETRY_EXHAUSTED_EVENT)` and logs ERROR | `SpringAmqpConfigTest.java` | S | B17 |

---

### PR-3 — $in Query Fix (Optimisation 1)
**Goal:** Replace `_id` range scan in `updateMulti` with `$in` point lookups — eliminates
cross-tenant index traversal, reduces `docsExamined` to exactly `batchSize` per iteration.

**Why last:** Provides per-query efficiency on top of the concurrency throttle already in place
from PR-1. Fully independent — no shared files with PR-1 or PR-2.

#### Backend — PR-3

| # | Task | File | Size | Depends on |
|---|------|------|------|-----------|
| B1 | Change `getAggregation()` to `group().push(_id)`, remove `sort` stage, update `BATCH_IDS` constant | `AbstractExpiryDateChangeDao.java` | S | — |
| B2 | Change `updateValidTill()` loop: extract `List<ObjectId> batchIds`, update `setStartId`, remove stale `ObjectId endId` declaration (line 60) | `AbstractExpiryDateChangeDao.java` | S | B1 |
| B3 | Change `updateValidTillDate()` signature → `List<ObjectId> batchIds`; replace range criteria with `Criteria.where(ID_COLUMN).in(batchIds).and(ORG_ID_COLUMN).is(...)` | `AbstractExpiryDateChangeDao.java` | S | B2 |
| B4 | Update `updateValidTillDate()` override: same signature + criteria change; `AggregationUpdate` pipeline unchanged | `CustomerEarnedCollectionDynamicExpiryDateChangeDao.java` | S | B3 |

#### Testing — PR-3

| # | Task | File | Size | Depends on |
|---|------|------|------|-----------|
| T1 | Update `shouldUpdateValidTillDate` test: replace `ObjectId endId` param with `List<ObjectId>` | `CustomerEarnedCollectionDynamicExpiryDateChangeDaoTest.java` | S | B4 |
| T5 | Verify existing `PromotionExpiryDateChangeIntegrationTest` suite passes unchanged after `$in` change | — | S | B1–B4 |

---

## Handoff Notes for test-plan-architect

**Tech detail doc location:**
`CAP-184618_expiry_job_tuning/CAP-184618_expiry_job_tuning_techdetail.md`

### Critical B2B Flows to Test
- UC-1: Single promotion end-date update → 2 jobs → correct `impactedCount` and `validTill` → Integration
- UC-2: 300 promotions batch update → semaphore limits active jobs → MongoDB load controlled → Load test
- UC-4: Pod crash scenario → counter healed after reconciliation → Integration
- UC-5: Semaphore full → job retried → eventually processed → Unit

### Critical B2C Flows to Test
- UC-9: After job completes, customer sees updated `validTill` → Integration (via `PromotionExpiryDateChangeIntegrationTest`)

### Regression Risks (must not break)
- `impactedCount` on `ExpiryDateChangeJob` must equal the number of documents updated per promotion — verify in existing integration tests.
- EARNED-dynamic path (`CustomerEarnedCollectionDynamicExpiryDateChangeDao`) uses `AggregationUpdate` (not simple `Update`) — the `$in` criteria change must be validated to work with `AggregationUpdate.updateMulti()`, not just simple `Update.updateMulti()`.
- `ExpiryDateChangeJobListener` must still consume from `PROMOTION_EXPIRY_UPDATE_QUEUE` after `containerFactory` switch — verify no message routing breakage.
- `shouldProceed()` must still reject `COMPLETED`, `EXPIRED`, `STOPPED`, `RUNNING` — only OPEN, ERRORED, WAITING should be allowed. Regression test: verify shouldProceed(COMPLETED) = false.

### Tenant Isolation Tests
- After processing job for orgId=A, verify no documents for orgId=B are modified.
- Verify Redis semaphore key is GLOBAL (not org-scoped) — all orgs compete for the same `maxGlobalConcurrentJobs` slots.

### Suggested Test Emphasis
- **Unit:** `AbstractExpiryDateChangeDao` via `CustomerIssuedCollectionStaticExpiryDateChangeDao` — mock `mongoOperation.aggregate()` returning a result with `ids` array; verify `updateMulti` called with `$in` criteria.
- **Unit:** `ExpiryDateChangeJobFacade` — semaphore scenarios (acquire/release/limit-hit/fail-open) + WAITING transition on rejection + SemaphoreRejectedException thrown.
- **Unit:** `ExpiryDateChangeHelper` — verify `reconcileSemaphoreCounter` called with correct cross-shard sum.
- **Unit:** `ExpiryDateChangeJobService` — WAITING state machine: isJobRunningForPromotion(WAITING)=true; markExpired after 2h; NOT marked before 2h.
- **Unit:** `RMQMessageTrackerAspect` — SemaphoreRejectedException NOT counted; real exceptions ARE counted.
- **Unit:** `SpringAmqpConfig` — MessageRecoverer fires on retry exhaustion; configurer called first.
- **Integration:** `PromotionExpiryDateChangeIntegrationTest` — existing tests must pass without modification.
- **Integration:** Verify EARNED-dynamic path works end-to-end with `AggregationUpdate` + `$in` criteria.
