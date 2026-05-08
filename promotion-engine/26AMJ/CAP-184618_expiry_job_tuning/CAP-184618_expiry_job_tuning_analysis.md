# Investigation: Expiry Date Change Job — MongoDB Load Reduction
**Date:** 2026-04-02
**Investigator:** Saravanan Kesavan
**Status:** Implementation Ready (Tech Detail Complete — 2026-04-28)
**Confidence:** HIGH

---

## Problem Statement

The `ExpiryDateChangeJobListener` processes expiry date change jobs for three collection types
(`customerIssuedPromotion`, `customerEarnedPromotion`, `codeBasedPromotion`). When a brand
updates 300 promotion expiry dates in a single batch, up to 600 RMQ messages are pushed to
`PROMOTION_EXPIRY_UPDATE_QUEUE`. All 6 pods simultaneously pull and execute bulk MongoDB
update loops, causing sustained CPU and IOPS spikes on MongoDB. Two orthogonal optimisations
are in scope:

1. **Bulk update query inefficiency** — the inner batch loop uses an `_id` range predicate on
   `updateMulti` which can scan documents beyond the target promotion's set.
2. **Uncontrolled cross-pod job concurrency** — no mechanism limits how many jobs run in
   parallel across the 6 pods; the value is effectively hardcoded at `pods × concurrentConsumers`.

---

## System Map (Affected Path)

**Call chain:**
```
RabbitMQ (PROMOTION_EXPIRY_UPDATE_QUEUE)
  → ExpiryDateChangeJobListener.expiryReminderJobListener()
    → ExpiryDateChangeJobFacade.changeExpiryForPromotion()
      → ExpiryDateChangeFactory.getExpiryDateChangerForEntity(jobType)
        → [ISSUED]         CustomerIssuedCollectionProcessor
                           → CustomerIssuedCollectionStaticExpiryDateChangeDao
        → [EARNED-static]  CustomerEarnedPromotionCollectionProcessor
                           → CustomerEarnedCollectionStaticExpiryDateChangeDao
        → [EARNED-dynamic] CustomerEarnedPromotionCollectionProcessor
                           → CustomerEarnedCollectionDynamicExpiryDateChangeDao  ← overrides updateValidTillDate()
        → [CODE]           CodePromotionCollectionExpiryChangeProcessor
                           → CodeBasedPromotionCollectionStaticExpiryDateChangeDao
      ↑ all static DAOs extend AbstractExpiryDateChangeDao
```

**Tenant isolation points:**
- `orgId` is injected into every aggregation `$match` and every `updateMulti` criteria.
- `ExpiryDateChangeJob` documents are scoped by `orgId + promotionId`.
- The `$in` optimisation drops `orgId` from the update predicate but this is safe — the
  ID list is already filtered by `orgId + promotionId` in the aggregation phase.

**Upstreams called:** MongoDB (3 collections), RabbitMQ

**Downstreams affected:** None — this is a pure write path on already-issued documents.

---

## Optimisation 1 — Replace Range Query with `$in` on Primary Key

### Root Cause

`AbstractExpiryDateChangeDao.updateValidTill()` runs a while-loop with two phases per
iteration:

**Phase 1 — `getAggregation()` (`AbstractExpiryDateChangeDao.java:106–118`)**
```
match(_id > startId, promotionId = X, orgId = Y)
→ sort(_id ASC)
→ limit(batchSize)
→ group().last(_id) as END_ID       ← returns only the last _id of the batch
→ project END_ID
```

**Phase 2 — `updateValidTillDate()` (`AbstractExpiryDateChangeDao.java:83–101`)**
```java
Criteria.where(ID_COLUMN).gt(expiryDateChangeJob.getStartId())
Criteria.where(ID_COLUMN).lte(endId)          // <-- range on _id
Criteria.where(ORG_ID_COLUMN).is(...)
Criteria.where(PROMOTION_ID_COLUMN).is(...)
updateMulti(criteria, update, EntityClass)
```

`endId` is the ObjectId of the batchSize-th document **for this org+promotion** in the
compound index. Because ObjectIds are globally sequential across ALL tenants, the range
`(startId, endId]` on `_id` spans time during which other orgs' documents were also created.
When MongoDB executes the range `updateMulti`:
- If the planner uses the primary `_id` index → scans ALL documents globally in the range,
  filters by `orgId + promotionId` — potentially thousands of extra index entries per batch.
- Even via the compound index `{orgId, promotionId, _id}` → the range traversal still
  processes every `_id` value between `startId` and `endId` for this org+promotion, including
  any gaps, and loads matching documents into WiredTiger cache.

**Same pattern repeated in `CustomerEarnedCollectionDynamicExpiryDateChangeDao.java:55–59`**
which overrides `updateValidTillDate()` but uses identical range criteria.

### Why the IN Clause Is Strictly Better

`$in [list of precise _ids]` uses the primary `_id` index for **point lookups** — O(batchSize)
leaf I/O with zero wasted scans. The `_id` list is produced by the aggregation which already
filters by `orgId + promotionId`, so the update touches exactly the right documents.

**MongoDB internals clarifications (validated during investigation):**
- The compound index `{orgId:1, promotionId:1, _id:1}` stores `_id` in sorted order for a
  given `(orgId, promotionId)` prefix. The `$sort(_id ASC)` in the aggregation is therefore
  redundant — MongoDB's aggregation planner eliminates it as a covered index scan. It can be
  removed after this change.
- The aggregation pipeline (`match + sort + limit + group(push _id)`) is a **covered query**
  — all data needed is in the compound index leaf nodes. Zero collection (heap) pages are
  loaded into WiredTiger cache during Phase 1.
- `_id` must be declared explicitly as the third key in `{orgId, promotionId, _id}`. The
  implicit tiebreaker suffix that MongoDB appends to non-unique index entries is NOT a
  declared range key and cannot be used for `_id > startId` range scans.
- Adding `orgId` to the `$in` update predicate (as a defensive filter) adds zero I/O —
  the `orgId` check is in-memory after the document is loaded for the update, which must
  happen regardless.

### Compound Index Confirmation

All three affected collections have `{orgId:1, promotionId:1, _id:1}`:
- `CustomerIssuedPromotion.java:23`
- `CustomerEarnedPromotion.java:29`
- `CodeBasedPromotion.java:32`

### Solution

**Phase 1 — change aggregation to collect all IDs:**
```
// BEFORE  (AbstractExpiryDateChangeDao.java:113)
GroupOperation maxAs = Aggregation.group().last(ID_COLUMN).as(END_ID);

// AFTER
public static final String BATCH_IDS = "ids";
GroupOperation collectIds = Aggregation.group().push(ID_COLUMN).as(BATCH_IDS);
// END_ID constant can be removed; sort stage can be removed (index already sorted)
```

**Phase 2 — update via `$in`:**
```java
// BEFORE  (AbstractExpiryDateChangeDao.java:85–89)
Criteria criteria = new Criteria().andOperator(
    Criteria.where(ORG_ID_COLUMN).is(expiryDateChangeJob.getOrgId()),
    Criteria.where(PROMOTION_ID_COLUMN).is(promotionMeta.getId().toString()),
    Criteria.where(ID_COLUMN).gt(expiryDateChangeJob.getStartId()),
    Criteria.where(ID_COLUMN).lte(endId));

// AFTER
Criteria criteria = Criteria.where(ID_COLUMN).in(batchIds);
// orgId can optionally be retained as a defensive in-memory check — zero I/O cost
```

**Loop changes in `updateValidTill()` (`AbstractExpiryDateChangeDao.java:55–81`):**
```java
// BEFORE
endId = (ObjectId) mappedResults.get(0).get(END_ID);
Long count = updateValidTillDate(expiryDateChangeJob, promotionMeta, endId, zoneOffset);
expiryDateChangeJob.setStartId(endId);

// AFTER
List<ObjectId> batchIds = (List<ObjectId>) mappedResults.get(0).get(BATCH_IDS);
if (CollectionUtils.isEmpty(batchIds)) break;
Long count = updateValidTillDate(expiryDateChangeJob, promotionMeta, batchIds, zoneOffset);
expiryDateChangeJob.setStartId(batchIds.get(batchIds.size() - 1));
```

**Method signature change:**
```java
// BEFORE
protected Long updateValidTillDate(ExpiryDateChangeJob job, PromotionMeta meta,
                                   ObjectId endId, ZoneOffset zoneOffset)
// AFTER
protected Long updateValidTillDate(ExpiryDateChangeJob job, PromotionMeta meta,
                                   List<ObjectId> batchIds, ZoneOffset zoneOffset)
```

**`CustomerEarnedCollectionDynamicExpiryDateChangeDao.java:55–59` — same fix:**
```java
// BEFORE
Criteria criteria = new Criteria().andOperator(
    Criteria.where(ORG_ID_COLUMN).is(expiryDateChangeJob.getOrgId()),
    Criteria.where(PROMOTION_ID_COLUMN).is(promotionMeta.getId().toString()),
    Criteria.where(ID_COLUMN).gt(expiryDateChangeJob.getStartId()),
    Criteria.where(ID_COLUMN).lte(endId));

// AFTER
Criteria criteria = Criteria.where(ID_COLUMN).in(batchIds);
```

### Evidence Trail

| File | Line | Finding |
|------|------|---------|
| `AbstractExpiryDateChangeDao.java` | 107–109 | Aggregation match: `_id > startId, promotionId, orgId` |
| `AbstractExpiryDateChangeDao.java` | 113 | `group().last(_id)` — returns only endId, not full batch |
| `AbstractExpiryDateChangeDao.java` | 85–89 | Range predicate `_id > startId AND _id <= endId` on updateMulti |
| `CustomerEarnedCollectionDynamicExpiryDateChangeDao.java` | 55–59 | Same range pattern in override |
| `CustomerIssuedPromotion.java` | 23 | `{orgId, promotionId, _id}` compound index confirmed |
| `CustomerEarnedPromotion.java` | 29 | `{orgId, promotionId, _id}` compound index confirmed |
| `CodeBasedPromotion.java` | 32 | `{orgId, promotionId, _id}` compound index confirmed |
| `application.properties` | 46 | `expiry.date.change.batch.size=${EXPIRY_DATE_CHANGE_BATCH_SIZE:1000}` — prod override: 2000 |

### What Does NOT Change

- The outer while-loop structure in `updateValidTill()` is unchanged.
- `ExpiryDateChangeJob.startId` cursor mechanics are unchanged (last element of batchIds
  becomes the new startId — equivalent to the old endId).
- No change to `CustomerIssuedCollectionStaticExpiryDateChangeDao`,
  `CustomerEarnedCollectionStaticExpiryDateChangeDao`,
  `CodeBasedPromotionCollectionStaticExpiryDateChangeDao` — they delegate entirely to the
  abstract class, so the fix in the abstract class covers them automatically.
- No change to the `ExpiryDateChangeProcessor` interface or any processor.
- No collection schema changes. No index changes.

---

## Optimisation 2 — Controlled Cross-Pod Job Concurrency

### Root Cause

`application.properties` line 23–24:
```properties
spring.rabbitmq.listener.simple.prefetch=1
spring.rabbitmq.listener.simple.consumers-per-queue=1
```
These are **global** settings that apply to all `@RabbitListener` queues. With 6 pods:
- Max concurrent expiry jobs = 6 (1 per pod).
- This value cannot be tuned for the expiry queue alone without impacting redemption,
  code-processing, and scheduler queues.
- 300 promotions updated → 600 RMQ messages → 6 pods immediately pull and run 6 concurrent
  bulk update loops → sustained MongoDB pressure until all 600 messages drain.

There is no mechanism to limit concurrent jobs to fewer than `pods × consumers-per-queue`.

### Duplicate Job Protection (Existing — Confirmed Safe)

A second `updatePromotion()` call for the same promotion while a job is OPEN or RUNNING is
**blocked at the API layer**, not the consumer layer:

1. `@Lockable(key = "'promotion_management_' + #orgId")` acquires a distributed Redis lock
   (via `RedisLockRegistry`, `spring-integration-redis`) for the org.
2. Inside the lock: `ExpiryDateChangeJobService.isJobRunningForPromotion()` queries
   `ExpiryDateChangeJobDao` for any job in `{OPEN, RUNNING}` for `orgId + promotionId + jobType`.
3. If found: throws `PromotionEngineException(UPDATE_EXPIRY_DATE_NOT_ALLOWED)`.

**Therefore:** duplicate jobs for the same promotion cannot be spawned. Each promotion has
at most 1 active job per type at any time.

### Solution

#### Layer 1: Dedicated Container Factory (Protective Isolation)

RabbitMQ is shared with other flows — notably redemption — whose throughput requirements may
drive future increases to the global `spring.rabbitmq.listener.simple.prefetch` or
`consumers-per-queue` settings. Without a dedicated factory, the expiry queue would silently
inherit any such increase, allowing more bulk-update jobs to run concurrently than intended and
undoing the MongoDB load reduction this change targets.

A dedicated factory pins the expiry queue's concurrency at explicit, protected values independent
of global settings. The values themselves are unchanged from today (prefetch=1, consumers=1) — this
is purely about ensuring they remain stable regardless of what happens to the globals.

**New bean in `SpringAmqpConfig.java`:**
```java
@Bean("expiryDateChangeContainerFactory")
public SimpleRabbitListenerContainerFactory expiryDateChangeContainerFactory(
        ConnectionFactory connectionFactory,
        RabbitListenerContainerFactoryConfigurer configurer) {

    SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
    // Inherit global retry interceptor, message converter, error handler.
    // MUST be called first — no RabbitListenerContainerFactoryConfigurer usage existed
    // previously; omitting it silently drops the retry policy on this factory.
    configurer.configure(factory, connectionFactory);
    // Pin explicitly — protects this queue from future global prefetch/consumers changes
    // driven by redemption or other high-throughput flows on the same broker.
    factory.setConcurrentConsumers(1);
    factory.setMaxConcurrentConsumers(1);
    factory.setPrefetchCount(1);
    return factory;
}
```

**`ExpiryDateChangeJobListener.java` — add `containerFactory`:**
```java
@RabbitListener(
    queues = {SpringAmqpConfig.PROMOTION_EXPIRY_UPDATE_QUEUE},
    containerFactory = "expiryDateChangeContainerFactory")
```

#### Layer 2: Redis Atomic Semaphore (True Cross-Pod Throttle)

For limiting total concurrent jobs to fewer than pod count (e.g., 3 out of 6 pods active),
a distributed Redis semaphore using atomic `INCR/DECR` is the correct mechanism.

**No new dependency required.** `spring-data-redis` (Lettuce) and `RedisTemplate` are
already configured in `RedisCacheUtil`. `spring-integration-redis` is already in `pom.xml`.

**Redis key:** `"promotion_engine:expiry_job:running_count"`

**Logic added to `ExpiryDateChangeJobFacade.java`:**

```java
// New fields
private static final String SEMAPHORE_KEY = "promotion_engine:expiry_job:running_count";

// Qualifier confirmed: only named RedisTemplate bean in codebase is "cacheEvictionRedisTemplate"
// (RedisCacheUtil.java:204). Reference pattern: CacheEvictionPublisher.java:28-29.
@Autowired
@Qualifier("cacheEvictionRedisTemplate")
private RedisTemplate<String, String> redisTemplate;

@Value("${expiry.date.change.max.global.concurrent.jobs:3}")
private int maxGlobalConcurrentJobs;

// Acquire: Redis INCR is atomic — no race condition between pods.
// Fail-open: if Redis is unavailable, let the job proceed rather than dead-letter 600 messages.
private boolean tryAcquireSemaphore(long orgId, String jobId) {
    try {
        Long current = redisTemplate.opsForValue().increment(SEMAPHORE_KEY);
        if (current != null && current <= maxGlobalConcurrentJobs) {
            log.info("Semaphore acquired orgId={} jobId={} count={}/{}", orgId, jobId, current, maxGlobalConcurrentJobs);
            return true;
        }
        redisTemplate.opsForValue().decrement(SEMAPHORE_KEY); // give back the slot
        log.info("Semaphore limit hit orgId={} jobId={} count={}/{}", orgId, jobId, current, maxGlobalConcurrentJobs);
        return false;
    } catch (Exception e) {
        log.warn("Redis unavailable, failing open for semaphore orgId={} jobId={}", orgId, jobId, e);
        return true; // fail-open
    }
}

private void releaseSemaphore(long orgId, String jobId) {
    try {
        Long remaining = redisTemplate.opsForValue().decrement(SEMAPHORE_KEY);
        log.info("Semaphore released orgId={} jobId={} remaining={}", orgId, jobId, remaining);
    } catch (Exception e) {
        log.warn("Redis unavailable during semaphore release orgId={} jobId={}", orgId, jobId, e);
    }
}

// Called by ExpiryDateChangeHelper post-shard-loop (see Counter Leak Mitigation section).
public void reconcileSemaphoreCounter(long actualRunning) {
    try {
        redisTemplate.opsForValue().set(SEMAPHORE_KEY, String.valueOf(Math.max(0, actualRunning)));
        log.info("Reconciled expiry semaphore counter to actual running count: {}", actualRunning);
    } catch (Exception e) {
        log.warn("Redis unavailable during semaphore reconciliation", e);
    }
}

// Modified changeExpiryForPromotion():
public void changeExpiryForPromotion(String expiryDateChangeJobId) {
    ExpiryDateChangeJob expiryDateChangeJob =
            expiryDateChangeJobService.findById(expiryDateChangeJobId);
    if (!shouldProceed(expiryDateChangeJob)) {
        return;
    }
    long orgId = expiryDateChangeJob.getOrgId();
    if (!tryAcquireSemaphore(orgId, expiryDateChangeJobId)) {
        // Throws → existing retry policy kicks in:
        // initial-interval=2000ms, multiplier=2, max-attempts=10
        throw new RuntimeException("Expiry job concurrency limit reached, will retry");
    }
    try {
        expiryDateChangeJob = expiryDateChangeJobService
                .updateExpiryDateChangeJobStatus(expiryDateChangeJob, BatchStatus.RUNNING);
        PromotionMeta promotionMeta = promotionMetaManagementService
                .getPromotionMeta(orgId, expiryDateChangeJob.getPromotionId());
        ExpiryDateChangeProcessor processor =
                expiryDateChangeFactory.getExpiryDateChangerForEntity(expiryDateChangeJob.getJobType());
        processor.process(expiryDateChangeJob, promotionMeta);
        expiryDateChangeJob = expiryDateChangeJobService
                .updateExpiryDateChangeJobStatus(expiryDateChangeJob, BatchStatus.COMPLETED);
        pushToMetrics(expiryDateChangeJob);
    } catch (Exception e) {
        log.error("Error while updating expiry date orgId={}: ", orgId, e);
        expiryDateChangeJob = expiryDateChangeJobService
                .updateExpiryDateChangeJobStatus(expiryDateChangeJob, BatchStatus.ERRORED);
        expiryDateChangeJobDao.save(expiryDateChangeJob);
        pushToMetrics(expiryDateChangeJob);
        throw e;
    } finally {
        releaseSemaphore(orgId, expiryDateChangeJobId); // always release
    }
}
```

**`application.properties`:**
```properties
expiry.date.change.max.global.concurrent.jobs=${EXPIRY_MAX_GLOBAL_CONCURRENT_JOBS:3}
```

Default is **3** — half the pod count. This halves MongoDB pressure from peak-burst scenarios
(300-promotion batch → 600 messages) while ensuring all messages complete within the retry window.
Raise to 6 to disable throttling, lower to 1–2 only with careful retry window sizing.

**Retry behaviour with semaphore rejection (default config):**
```
Retry 1 : wait  2,000 ms
Retry 2 : wait  4,000 ms
Retry 3 : wait  8,000 ms
Retry 4 : wait 16,000 ms
...
Retry 10: wait ~512,000 ms
Total window: ~17 minutes for a message to exhaust all retries
```
With `maxGlobalConcurrentJobs=3` and 600 queued messages: 3 pods run jobs while 3 pods hold
their messages in retry backoff. As each job completes, a waiting pod acquires the semaphore on
its next retry. All 600 messages drain in batches of 3, well within the 17-minute retry window
for any reasonably-sized job. No messages are dead-lettered under normal conditions.

#### Counter Leak Mitigation (Pod Crash)


If a pod is killed between `INCR` and the `finally` block, the counter stays inflated.

**Reconciliation placement — CRITICAL (multi-shard correctness):**
`ExpiryDateChangeJobService.markAllJobAsExpiredIfNotRunning()` runs **per-shard** — it is called
inside the shard loop in `ExpiryDateChangeHelper.updateExpiryDateChangeJobs()`. Placing
reconciliation inside that method would reset the counter to only the last shard's RUNNING count,
not the cross-shard sum. The reconciliation **must** go in `ExpiryDateChangeHelper` AFTER the
shard loop completes, once the full cross-shard total is known.

**Changes to `ExpiryDateChangeHelper.updateExpiryDateChangeJobs()`:**
```java
// Inject ExpiryDateChangeJobFacade (new dependency — no cycle, confirmed during tech detail)
@Autowired
private ExpiryDateChangeJobFacade expiryDateChangeJobFacade;

// Inside updateExpiryDateChangeJobs(), modified shard loop:
long totalRunning = 0;
for (String shardKey : allShardKeys) {
    // ... existing MDC setup and per-shard logic ...
    expiryDateChangeJobService.markAllJobAsExpiredIfNotRunning();
    totalRunning += expiryDateChangeJobDao.countByStatus(BatchStatus.RUNNING); // per-shard context
}
// AFTER loop — reconcile with global sum across all shards:
expiryDateChangeJobFacade.reconcileSemaphoreCounter(totalRunning);
```

`ExpiryDateChangeJobDao` needs one new derived query (index-covered by existing `{status:1}`):
```java
long countByStatus(BatchStatus status);
```

---

#### Semaphore Counter Edge Cases

> **For tech-detailer and test-plan-architect:** All known ways the counter can drift, which
> self-heal, and which require explicit code. EC-4 is the only case that requires a code fix.

| EC | Trigger | Counter direction | Jobs blocked? | Self-recovery | Action required |
|----|---------|------------------|--------------|--------------|-----------------|
| EC-1A | Redis restart without AOF/RDB persistence — key lost | Goes negative (DECRs land on fresh 0) | **No** — negative value ≤ max, fail-open | Yes — hourly reconciliation | Confirm AOF with infra; see §Open Questions |
| EC-1B | Fail-open acquire (Redis down, INCR never sent) + successful release (Redis back up, DECR sent) | Drifts −1 per occurrence | **No** — fail-open | Yes — hourly reconciliation | Accept; bounded drift |
| EC-2A | Pod crash between INCR and `finally` block | Over-counts by 1 per crash | Partially — effective max = max − leaked slots | Yes — hourly reconciliation | Existing design; no change |
| EC-2B | `releaseSemaphore` DECR throws permanently | Over-counts by 1 | Partially | Yes — hourly reconciliation | Existing catch in `releaseSemaphore`; no change |
| EC-3 | Rejection race: Pod A INCRs to 4 (>max), Pod B INCRs to 5, both DECR back to 3 | Transient spike to 4–5 | **No** — both pods correctly rejected | Yes — same call | By design; no action needed |
| **EC-4** | `maxGlobalConcurrentJobs = 0` (misconfiguration or typo in env var) | Stuck at 0 | **YES — hard deadlock on all pods** | **NO — reconciliation writes 0, deadlock persists** | **@PostConstruct validation required (see below)** |
| EC-5 | `@EnableScheduling` off on all pods — reconciliation never runs | Drift accumulates | Eventually yes (over-count path) | No | Verify `@EnableScheduling` scope (Assumption 6) |

**Why negative counter does not block jobs (EC-1A, EC-1B):**
The check is `current != null && current <= maxGlobalConcurrentJobs`. A negative value always
satisfies `≤ 3`, so acquisitions are allowed freely — the semaphore becomes fail-open until
enough INCRs push the counter back above zero. This is the correct safe-default behaviour.

**Why EC-4 is the only case with no self-recovery:**
`reconcileSemaphoreCounter(totalRunning)` calls `SET counter = countByStatus(RUNNING)`.
During an EC-4 deadlock, no jobs are RUNNING (all are being rejected), so `totalRunning = 0`
and `SET counter = 0`. The counter stays at 0. The next INCR produces 1, which is `> 0` (max),
so the pod DECRs back to 0 and rejects. The deadlock is perpetual until operator intervention.

#### EC-4 Fix — Clamp to 1 in `tryAcquireSemaphore` (Required)

Implementer **must** apply `Math.max(1, maxGlobalConcurrentJobs)` as the effective limit inside
`tryAcquireSemaphore`. A misconfigured value of 0 (or negative) silently becomes 1 — the pod
continues to function, processes jobs one at a time, and logs a WARN so ops can see and correct
the env var without any service disruption:

```java
private boolean tryAcquireSemaphore(String jobId, Long orgId) {
    try {
        int effectiveMax = Math.max(1, maxGlobalConcurrentJobs);   // EC-4: clamp; never deadlock
        if (effectiveMax != maxGlobalConcurrentJobs) {
            log.warn("expiry.date.change.max.global.concurrent.jobs={} is invalid; "
                     + "clamped to 1", maxGlobalConcurrentJobs);
        }
        Long current = redisTemplate.opsForValue().increment(SEMAPHORE_KEY);
        if (current != null && current <= effectiveMax) { ... }
        ...
    }
}
```

**Why clamp instead of fail-fast (`@PostConstruct` throwing):**
Stopping the pod on a misconfiguration means zero expiry jobs run until the env var is corrected
and pods are restarted — a worse outcome than running at throughput=1. Clamping to 1 keeps the
system functional, limits MongoDB load to the safest possible concurrency (1 job at a time),
and surfaces the problem via a WARN log rather than a crash.

#### EC-1B Detail — Fail-open Acquire + Successful Release (Under-count Drift)

```
t=0  Counter = 2 (2 jobs running)
t=1  Redis briefly unavailable
t=2  Pod 4: tryAcquireSemaphore → INCR throws → catch → return true
     ← NO INCR sent to Redis
t=3  Pod 4 runs the job
t=4  Redis recovers
t=5  Pod 4 finishes → releaseSemaphore → DECR → counter = 1
Net: 3 jobs ran, counter says 1. Under-count by 1 per occurrence.
```

Behaviour: semaphore allows slightly more jobs than intended (opposite of pod-crash over-count).
Recovery: hourly reconciliation corrects. No code change needed — documented here for
test-plan-architect awareness (test: stub increment to throw, verify DECR still called in finally).

---

## Alternatives Considered for Layer 2 (Cross-Pod Throttle)

Five approaches were evaluated. Option A was chosen.

### Option A — Redis INCR/DECR Atomic Semaphore ✅ CHOSEN

A single Redis key `"promotion_engine:expiry_job:running_count"` holds the global running count.
`INCR` is atomic; no race between pods. **No TTL on the counter key** — the counter never expires
while jobs are in flight. Drift from pod crash is corrected by the hourly reconciliation scheduler.

**Why chosen:** Only approach with no TTL expiry risk for variable-duration jobs. The counter is a
plain integer — no lock ownership semantics, no TTL pressure, no cross-concern side-effects.

---

### Option B — RedisLockRegistry Slot Pool ⚠️ UNSAFE FOR LONG-RUNNING JOBS

Reuse the existing `RedisLockRegistry` bean (`RedisLockRegistryConfig.java:26`). Create N named lock
keys (`expiry:slot:0` … `expiry:slot:N-1`). Each pod calls `tryLock(0)` on each slot in sequence;
first successful `tryLock` allows the job to proceed.

**Why this looks attractive:** `RedisLockRegistry` bean already exists. `tryLock` semantics map
naturally to slot acquisition.

**Why it fails for this use case — verified via bytecode inspection of 5.3.6.RELEASE:**

`obtainLock()` inside `RedisLock` calls the Lua acquisition script once, passing `expireAfter`
(= `redis.lock.expiry.ms = 60000 ms`) as the Redis key TTL. There is **no renewal method, no
heartbeat thread, and no background scheduler** anywhere in `RedisLockRegistry` or `RedisLock` in
5.3.6.RELEASE. Automatic lock renewal was introduced in Spring Integration 5.5+/6.x — not present
here. `redis.lock.expiry.ms=60000` is therefore a **hard wall-clock TTL** set once at acquisition.

A bulk-update job processing 2 M documents routinely runs for minutes. Once 60 s elapses the Redis
key expires, the slot becomes available, another pod acquires it, and the global concurrency limit
is silently violated.

**Secondary concern:** `redis.lock.expiry.ms` is a global setting consumed by all `@Lockable`
usages in the codebase (e.g., `@Lockable(key = "'promotion_management_' + #orgId")` in
`PromotionFacade.updatePromotion()`). Increasing it to cover long expiry jobs (e.g., 30 min) also
widens org-level lock windows, creating potential deadlock windows elsewhere.

**On `cleanupUnusedLocks` (a common concern):**
`RedisLockRegistryConfig.cleanupUnusedLocks()` calls `redisLockRegistry.expireUnusedOlderThan(ms)`.
Bytecode inspection confirms this method executes `locks.entrySet().removeIf(predicate)` — where
`locks` is the local pod's `ConcurrentHashMap<String, RedisLock>` in JVM heap. **No Redis `DEL`
command is issued.** No remote operation is performed. Other pods each hold their own independent
`ConcurrentHashMap` — this pod's cleanup scheduler has zero visibility into them.

The predicate also guards active locks: an entry is only removed if
`(now − lockedAt) > unusedDuration AND !isAcquiredInThisProcess()`. Any lock currently held by this
pod returns `isAcquiredInThisProcess() == true` and is skipped. In short: `cleanupUnusedLocks` is a
local memory leak prevention mechanism and will not release locks of any pod, including its own
active ones.

**Bottom line:** Option B is safe as a general distributed lock (it is what backs `@Lockable`), but
it is not suitable as a long-running job slot pool without a TTL renewal mechanism that does not
exist in 5.3.6.RELEASE.

---

### Option C — Redis SET NX EX Slot Pool ❌ UNSAFE — DROPPED

Same hard-TTL problem as Option B. `SET key value NX EX <seconds>` writes the TTL once at
acquisition time; Redis counts down regardless of whether the job is still running. If a job exceeds
the configured TTL the key expires, the slot is freed, and another pod can acquire it — violating
the concurrency limit.

Unlike Option B, Option C would not share a TTL config with `@Lockable`, making the TTL tunable
per slot. But the fundamental safety gap remains: **there is no reliable way to set TTL > max job
duration when job duration depends on document volume, which varies per promotion.** Setting a
very large TTL (e.g., 60 min) extends crash recovery windows equally. Dropped in favour of Option A.

---

### Option D — RabbitMQ-Level Throttle via `prefetch + consumers-per-queue`

Global `prefetch=1` and `consumers-per-queue=1` currently provide the right per-pod behaviour.
However, RabbitMQ is shared with other flows (notably redemption); future throughput demands on
those flows could push the global settings higher. A dedicated factory for the expiry queue
protects against this by pinning `prefetch=1` and `concurrentConsumers=1` independently.

**Role in final design:** Included — not as a throttle mechanism (that is Option A's job), but as
**protective isolation** ensuring the expiry queue's concurrency characteristics remain stable
regardless of global broker configuration changes. It cannot by itself cap total concurrent jobs
below pod count; that remains Option A's responsibility.

---

### Option E — Database-Level Advisory Lock (MongoDB `findAndModify` counter)

Maintain a running count in a MongoDB document instead of Redis. Atomic `findAndModify` with
`$inc` + conditional on max threshold.

**Design equivalence to Option A:** This is the **same counter semaphore pattern as Option A**
— atomic increment, compare-against-max, allow or decrement-and-reject, decrement on release.
The mechanics are identical. The only difference is the storage tier: MongoDB (Option E) vs
Redis (Option A).

**Why not chosen:** Tier mismatch — MongoDB is the resource under pressure that this change is
specifically trying to protect. Adding a `findAndModify` round-trip on every job start and
every job end adds to the MongoDB load we are reducing. Redis is already present, has sub-
millisecond latency for counter operations, and is the correct tier for lightweight coordination
primitives. Same design, wrong storage choice.

---

### Summary Table

| Option | TTL Risk for Long Jobs | Cross-Pod | Existing Infra | Crash Recovery | Verdict |
|--------|----------------------|-----------|---------------|----------------|---------|
| A — INCR/DECR counter | None (no TTL on counter) | ✅ True global | ✅ Yes | Hourly reconciliation | **Chosen** |
| B — RedisLockRegistry slot | ❌ Hard 60 s TTL (no renewal in 5.3.6) | ✅ | ✅ Yes | Auto (TTL expiry) | Unsafe for jobs > 60 s |
| C — SET NX EX slot | ❌ Hard TTL at acquisition | ✅ | ✅ Yes | Auto (TTL expiry) | Unsafe — dropped |
| D — dedicated factory (isolation) | None | ❌ Per-pod only | ✅ Yes | N/A | Included — protective isolation from global setting changes, not a throttle |
| E — MongoDB advisory | None | ✅ | ❌ Extra load | N/A | Same counter pattern as A — wrong tier (adds load to the resource being protected) |

---

## What Changes — Full File List

| File | Change |
|------|--------|
| `AbstractExpiryDateChangeDao.java` | Phase 1: `group().push(_id)` instead of `group().last(_id)`. Phase 2: `$in(batchIds)` criteria. Loop: extract List, setStartId to last element. Method signature: `ObjectId endId` → `List<ObjectId> batchIds`. Remove `END_ID` constant, add `BATCH_IDS` constant. Remove redundant `sort` stage (index already sorted). Remove stale `ObjectId endId` field declaration (line 60). |
| `CustomerEarnedCollectionDynamicExpiryDateChangeDao.java` | `updateValidTillDate()` override: same signature change, replace range criteria with `Criteria.where(ID_COLUMN).in(batchIds)`. AggregationUpdate pipeline itself unchanged. |
| `SpringAmqpConfig.java` | Add `expiryDateChangeContainerFactory` bean. Inject `RabbitListenerContainerFactoryConfigurer`, call `configurer.configure()` first (inherits message converter + error handler), then add custom `StatefulRetryOperationsInterceptor` with Fix-B `MessageRecoverer`, then pin `concurrentConsumers=1`, `maxConcurrentConsumers=1`, `prefetchCount=1`. Purpose: protective isolation + retry exhaustion observability. |
| `ExpiryDateChangeJobListener.java` | Add `containerFactory = "expiryDateChangeContainerFactory"` to `@RabbitListener`. |
| `ExpiryDateChangeJobFacade.java` | Inject `@Qualifier("cacheEvictionRedisTemplate") RedisTemplate<String,String>`. Add `tryAcquireSemaphore(orgId, jobId)` with fail-open catch. Add `releaseSemaphore(orgId, jobId)` with fail-safe catch. Add `reconcileSemaphoreCounter(actualRunning)` (called by Helper post-loop). Wrap `changeExpiryForPromotion()` in try-finally with semaphore acquire/release. On rejection: transition job to `WAITING` (updates `lastUpdatedOn`), throw `SemaphoreRejectedException`. Update `shouldProceed()` to also allow `WAITING` status. |
| `ExpiryDateChangeJobService.java` | **`markAllJobAsExpiredIfNotRunning()`:** Add second query for `WAITING` jobs with **2-hour** expiry threshold (separate from 24h for OPEN/RUNNING; see Operational Robustness section). **`isJobRunningForPromotion()`:** Add `WAITING` to the status list so a duplicate job is not created while an existing job is throttle-retrying. |
| `ExpiryDateChangeHelper.java` | Inject `ExpiryDateChangeJobFacade` (new dependency — no cycle confirmed). Accumulate `countByStatus(RUNNING)` per shard during existing loop. Call `expiryDateChangeJobFacade.reconcileSemaphoreCounter(totalRunning)` after loop. |
| `ExpiryDateChangeJobDao.java` | Add `long countByStatus(BatchStatus status)` — derived query, covered by existing `{status:1}` index on `ExpiryDateChangeJob`. |
| `BatchStatus.java` | Add `WAITING` enum value between `ERRORED` and `COMPLETED`. |
| `SemaphoreRejectedException.java` | **NEW** — `extends RuntimeException`. Thrown by `ExpiryDateChangeJobFacade` when semaphore limit is hit. Allows `RMQMessageTrackerAspect` to distinguish throttle rejections from real failures. |
| `RMQMessageTrackerAspect.java` | Add check: skip `metricsService.markAsFailedRequest()` when the caught exception is (or is caused by) `SemaphoreRejectedException`. Prevents NR failure counter from inflating on every retry attempt during normal throttle behaviour. |
| `application.properties` | Add `expiry.date.change.max.global.concurrent.jobs=${EXPIRY_MAX_GLOBAL_CONCURRENT_JOBS:3}`. Add retry param properties used by the custom retry interceptor in the dedicated factory. |

## What Does NOT Change

- `CustomerIssuedCollectionStaticExpiryDateChangeDao` — inherits abstract class, no change.
- `CustomerEarnedCollectionStaticExpiryDateChangeDao` — inherits abstract class, no change.
- `CodeBasedPromotionCollectionStaticExpiryDateChangeDao` — inherits abstract class, no change.
- All three `ExpiryDateChangeProcessor` implementations — no change.
- All collection indexes — no change.
- RMQ queue/exchange/binding declarations — no change.
- Duplicate job prevention logic in `PromotionFacade.updatePromotion()` — no change (already correct).

---

## Assumptions — Resolution Status

1. `EXPIRY_DATE_CHANGE_BATCH_SIZE=2000` in prod — **confirmed by user**. In-memory List of 2000
   ObjectIds ≈ 24 KB per batch iteration — acceptable.
2. ~~`RedisTemplate<String, String>` bean is available — verify qualifier.~~ **Resolved.**
   Only named `RedisTemplate<String,String>` bean is `"cacheEvictionRedisTemplate"` declared at
   `RedisCacheUtil.java:204`. Canonical injection pattern: `CacheEvictionPublisher.java:28-29`.
   Always use `@Qualifier("cacheEvictionRedisTemplate")`.
3. ~~Custom factory may not inherit global retry interceptor — verify.~~ **Resolved.**
   No `RabbitListenerContainerFactoryConfigurer` usage existed anywhere in the codebase. Custom
   factory silently has no retry policy without it. Fix: inject `RabbitListenerContainerFactoryConfigurer`
   and call `configurer.configure(factory, connectionFactory)` before any custom overrides.
4. `markAllJobAsExpiredIfNotRunning()` runs per-shard — **confirmed**. Called inside the shard
   loop in `ExpiryDateChangeHelper.updateExpiryDateChangeJobs()` (`ExpiryDateChangeHelper.java:37–44`).
   Reconciliation must therefore be placed AFTER the loop, not inside this method.
5. `ExpiryDateChangeJob` has `@CompoundIndex(def = "{'status' : 1}")` — **confirmed** at
   `ExpiryDateChangeJob.java:20`. The `countByStatus` derived query is index-covered.

---

---

## Operational Robustness — WAITING Status, Fix-A, Fix-B

Three additional changes are agreed to correct NR metric noise, ensure jobs are not prematurely expired
during the retry window, and provide a clean signal when retries are exhausted.

---

### WAITING Status — Prevent Premature EXPIRED During Throttle Retry

**Problem:** When `tryAcquireSemaphore()` returns `false`, `changeExpiryForPromotion()` throws
a `RuntimeException` and Spring AMQP schedules a retry. During the retry window (~17 min), the
job sits in its previous status (`OPEN` or `ERRORED`) with its `lastUpdatedOn` unchanged.
If the job was already close to the 24-hour expiry threshold (e.g., last updated 23h ago due to
slow processing or a prior ERRORED state), the hourly `markAllJobAsExpiredIfNotRunning()` scheduler
could expire the job before all retry attempts are exhausted — permanently losing it.

**Solution:** On semaphore rejection, transition the job to `WAITING` status (via
`updateExpiryDateChangeJobStatus(job, WAITING)`) BEFORE throwing `SemaphoreRejectedException`.
This updates `lastUpdatedOn` to NOW, resetting the 24-hour clock. On every subsequent retry
that again hits the semaphore limit, the job transitions back to `WAITING` and `lastUpdatedOn`
is refreshed again. The 24-hour expiry clock can never fire while a job is actively retrying.

**State machine changes:**
```
OPEN  ──(semaphore acquired)──→ RUNNING ──(success)──→ COMPLETED
OPEN  ──(semaphore rejected)──→ WAITING ──(next retry, acquired)──→ RUNNING
WAITING ──(next retry, rejected again)──→ WAITING  [loop, lastUpdatedOn refreshed]
WAITING ──(retries exhausted)── [WAITING, orphaned] ──(2h expiry)──→ EXPIRED
ERRORED ──(semaphore acquired on retry)──→ RUNNING
```

**`shouldProceed()` must allow WAITING:**
```java
// BEFORE
return job.getStatus().equals(BatchStatus.OPEN)
    || job.getStatus().equals(BatchStatus.ERRORED);

// AFTER
return job.getStatus().equals(BatchStatus.OPEN)
    || job.getStatus().equals(BatchStatus.ERRORED)
    || job.getStatus().equals(BatchStatus.WAITING);
```

Without this change, a job in WAITING status would be silently dropped on the next retry attempt
(the `shouldProceed` check returns `false` → early return → message consumed but job never runs).

**`markAllJobAsExpiredIfNotRunning()` expiry threshold for WAITING:**
WAITING jobs require a SHORTER orphan threshold than OPEN/RUNNING jobs because:
- Active retrying: `lastUpdatedOn` refreshed every retry attempt (seconds to minutes) → never hits any threshold
- Orphaned WAITING (pod died mid-retry): no more updates → threshold signals it is dead

The retry window is ~17 minutes maximum. A 2-hour threshold is conservative (7× the window)
while still being far shorter than 24h, allowing orphaned WAITING jobs to be detected and
expired the same day rather than lingering for up to 48h (24h idle + 24h scheduler lag):

```java
// NEW second loop in markAllJobAsExpiredIfNotRunning():
List<ExpiryDateChangeJob> waitingJobs = expiryDateChangeJobDao.findByStatusIn(
        Lists.newArrayList(BatchStatus.WAITING));
waitingJobs.forEach(job -> {
    long hoursDiff = TimeUnit.HOURS.convert(
        Math.abs(new Date().getTime() - job.getAttribution().getLastUpdatedOn().getTime()),
        TimeUnit.MILLISECONDS);
    if (hoursDiff > 2) {    // 2h >> 17-min retry window; orphaned if exceeded
        job.setStatus(BatchStatus.EXPIRED);
        job.getAttribution().setLastUpdatedOn(new Date());
        expiryDateChangeJobDao.save(job);
    }
});
```

**`isJobRunningForPromotion()` must include WAITING:**
This method guards the API layer against creating duplicate jobs for the same promotion.
Currently queries `OPEN + RUNNING`. If a job transitions to `WAITING`, the API layer no longer
sees it as "running" and could create a second job for the same promotion during the retry window,
producing two concurrent jobs that both process the same documents.

```java
// BEFORE
Lists.newArrayList(BatchStatus.OPEN, BatchStatus.RUNNING)

// AFTER
Lists.newArrayList(BatchStatus.OPEN, BatchStatus.RUNNING, BatchStatus.WAITING)
```

---

### Fix-A — SemaphoreRejectedException (NR Metric Noise)

**Problem:** `RMQMessageTrackerAspect` fires `metricsService.markAsFailedRequest()` on ANY
exception thrown by the listener. With `maxGlobalConcurrentJobs=3` and 600 queued messages,
up to 3 pods × 10 retry attempts = 30 `markAsFailedRequest()` calls per promotion batch update
from semaphore throttling alone — pure noise that drowns real failures in NR dashboards and alerts.

**Solution:** Create `SemaphoreRejectedException extends RuntimeException`. Throw it instead of
a generic `RuntimeException` when `tryAcquireSemaphore()` returns `false`. Update
`RMQMessageTrackerAspect` to skip `markAsFailedRequest()` when the root cause is
`SemaphoreRejectedException`:

```java
// RMQMessageTrackerAspect — add before markAsFailedRequest():
if (throwable instanceof SemaphoreRejectedException
        || (throwable.getCause() instanceof SemaphoreRejectedException)) {
    log.debug("Semaphore rejection — not a failure metric: {}", throwable.getMessage());
    return;
}
metricsService.markAsFailedRequest(...);
```

Spring AMQP wraps listener exceptions in a `ListenerExecutionFailedException`, so both
`throwable instanceof SemaphoreRejectedException` and `throwable.getCause()` checks are needed.

---

### Fix-B — MessageRecoverer (Retry Exhaustion Observability)

**Problem:** When all 10 retry attempts are exhausted (after ~17 min), the message is silently
discarded (no DLQ configured). There is no log, no NR event, and no alert. Ops have no way to
know a job has permanently failed.

**Solution:** Configure a `MessageRecoverer` on `expiryDateChangeContainerFactory`. This hook
is called exactly once by Spring AMQP when the retry policy is exhausted. It logs an `ERROR`
and emits a NR custom event so the ops team is notified:

```java
// In expiryDateChangeContainerFactory() — after configurer.configure():
StatefulRetryOperationsInterceptor retryInterceptor = RetryInterceptorBuilder.stateful()
    .maxAttempts(retryMaxAttempts)                     // @Value from existing retry config
    .backOffOptions(retryInitialInterval, retryMultiplier, retryMaxInterval)
    .recoverer((message, cause) -> {
        log.error("Expiry job exhausted all {} retry attempts — message discarded. cause={}",
                  retryMaxAttempts, cause.getMessage(), cause);
        metricsService.markAsFailedRequest(EXPIRY_JOB_RETRY_EXHAUSTED_NR_EVENT);
    })
    .build();
factory.setAdviceChain(retryInterceptor);
// NOTE: setAdviceChain() replaces the chain set by configurer.configure().
// Message converter and error handler (set by configurer) are unaffected.
// Retry config values must be injected via @Value and must match global retry props.
```

**New NR event name constant:** `NewrelicConstants.EXPIRY_JOB_RETRY_EXHAUSTED_EVENT`
= `"expiry_job_retry_exhausted"` — enables a NR alert: if this event fires, ops can investigate
and replay via the Admin API (Option 2 — not in scope of this change but documented below).

**Option 2 — DLQ Replay via Admin API (documented, not in this PR's scope):**
Once Fix-B emits the NR event and ops identify the affected jobId from the ERROR log, the
Admin API can reset the job:
1. Read the job by ID from MongoDB
2. Reset `status → OPEN`, `startId → new ObjectId("000000000000000000000000")`
3. Re-publish to `PROMOTION_EXPIRY_UPDATE_QUEUE`

This is a manual-replay mechanism. No DLQ queue is required. The Admin API endpoint is deferred
to a separate PR/ticket.

---

## Risks for Implementer

| Risk | Likelihood | Mitigation |
|------|-----------|-----------|
| `$in` with 2000 ObjectIds hits MongoDB document size limit | Low — 2000 × 12 bytes = 24 KB, well within 16 MB BSON limit | Verify batchSize upper bound in prod config |
| Redis semaphore counter drifts negative if `releaseSemaphore` called without matching acquire | Low — `finally` block always releases; only mis-path is `tryAcquireSemaphore` returns false (no acquire issued, no release needed) | Review all exception paths in `changeExpiryForPromotion` |
| Stateless RMQ retry exhausted (10 attempts) before semaphore frees — message lost | Low at default `max.global.concurrent.jobs=3`; total retry window ~17 min covers all realistic job durations. Becomes a real risk only if limit is dropped to 1–2 with very long-running jobs. | Do not set `maxGlobalConcurrentJobs` below 2 without also increasing `max-attempts` or `initial-interval`. |
| Redis unavailable — `INCR/DECR` throws, all expiry jobs fail their retry window | Low — Redis `incentives` cluster is HA; resolved by fail-open pattern in `tryAcquireSemaphore` | Catch all exceptions in semaphore acquire/release and return true (fail-open). Already in final design. |
| Redis Sentinel restart with no persistence (AOF/RDB disabled) — counter key lost mid-batch | **Medium** — If Redis restarts without persistence, `running_count` key is lost (resets to 0). New jobs see a fresh counter and acquire normally (correct). But in-flight jobs that already incremented the counter will call `releaseSemaphore` (DECR) on a zero-baseline key, driving the counter negative. Interleaved with new acquisitions, up to all 6 pods can run concurrently for the period between restart and the next hourly reconciliation — same as today before this fix. | **(1) Acknowledge as bounded fail-open:** behaviour is identical to the existing Redis-unavailable fail-open path; MongoDB load temporarily reverts to today's baseline, self-corrects within ≤1 hour via reconciliation. **(2) Optional mitigation — seed counter on startup:** add a call to `reconcileSemaphoreCounter(countByStatus(RUNNING))` in an `@EventListener(ApplicationReadyEvent.class)` on each pod startup. On restart, each pod reads actual RUNNING jobs from MongoDB and sets the counter before processing begins, eliminating the drift window. Confirm with infra team whether AOF is enabled; if so this risk is negligible. |
| Dedicated container factory silently loses retry policy | **Resolved** — `RabbitListenerContainerFactoryConfigurer.configure()` called first in factory bean. No prior configurer usage existed in codebase; omitting it silently drops the retry policy. Pattern is specified in the Layer 1 snippet above. |
| `CustomerEarnedCollectionDynamicExpiryDateChangeDao.updateValidTillDate()` method visibility | Low — currently `protected`; signature change must maintain visibility | Keep `protected`, update test accordingly |
| Redis key namespace collision on shared `incentives` cluster | Low — `"promotion_engine:expiry_job:running_count"` is specific; no TTL required | Verify namespace with infra team before deploy |
| **EC-4: `maxGlobalConcurrentJobs=0` or negative misconfiguration — permanent deadlock without a guard** | **Low likelihood, HIGH impact** — a typo (`EXPIRY_MAX_GLOBAL_CONCURRENT_JOBS=0`) causes all jobs to be rejected forever; reconciliation writes 0 each hour, preserving the deadlock. | **Required fix:** `int effectiveMax = Math.max(1, maxGlobalConcurrentJobs)` inside `tryAcquireSemaphore`. Invalid value silently becomes 1 (safest concurrency). Log WARN so ops can see and correct the env var without service disruption. Stopping the pod is explicitly rejected — zero job processing is worse than single-job processing. |
| **WAITING + `isJobRunningForPromotion()` gap — duplicate jobs created during retry window** | **Medium likelihood, HIGH impact** — If `isJobRunningForPromotion()` does not include `WAITING` in its status list, a second `updatePromotion()` API call for the same promotion while the first job is WAITING (retrying) creates a duplicate job. Both jobs process the same promotion documents, producing double updates. | **Required fix:** Add `BatchStatus.WAITING` to `Lists.newArrayList(BatchStatus.OPEN, BatchStatus.RUNNING, BatchStatus.WAITING)` in `ExpiryDateChangeJobService.isJobRunningForPromotion()`. See Operational Robustness section. |
| **Fix-B `setAdviceChain()` replaces `configurer.configure()` retry chain** | **Low likelihood** — calling `factory.setAdviceChain(retryInterceptor)` after `configurer.configure()` replaces the retry advice set by the configurer. If retry `@Value` params are not injected in `SpringAmqpConfig`, the custom interceptor silently uses different retry settings than the rest of the app. | **Required:** Inject all retry props (`RABBITMQ_RETRY_MAX_ATTEMPTS`, `RABBITMQ_RETRY_INITIAL_INTERVAL_MS`, `RABBITMQ_RETRY_MULTIPLIER`, `RABBITMQ_RETRY_MAX_INTERVAL_MS`) via `@Value` into `SpringAmqpConfig` and wire them into the custom `StatefulRetryOperationsInterceptor`. |

---

## Open Questions (Must Resolve Before Build)

- [x] ~~Does the dedicated factory need explicit retry interceptor configuration?~~
  **Resolved:** Must use `RabbitListenerContainerFactoryConfigurer.configure()` first. No prior
  usage existed in codebase — without it the factory has no retry policy.
- [x] ~~What is the `RedisTemplate` bean qualifier?~~
  **Resolved:** `@Qualifier("cacheEvictionRedisTemplate")` — sole named bean at `RedisCacheUtil.java:204`.
- [ ] Should the `sort` stage be removed from `getAggregation()` after the `$in` change?
  It is redundant (compound index is pre-sorted) but harmless. **Owner:** Tech lead — remove for
  cleanliness or retain for readability. Recommended: remove with an explaining comment.
- [x] ~~Target value for `EXPIRY_MAX_GLOBAL_CONCURRENT_JOBS` in prod.~~
  **Resolved:** Default set to **3** (half of 6 pods). Raise to 6 to disable throttling post-deploy
  if MongoDB headroom permits; lower to 2 only with retry window review.
- [ ] Should the WAITING job expiry threshold (2h) be configurable via `EXPIRY_WAITING_JOB_EXPIRY_HOURS`?
  **Owner:** Tech lead — hardcoded constant is acceptable (derived from retry window ≈ 17 min);
  make configurable only if per-environment fine-tuning is expected.
- [ ] Should `NewrelicConstants.EXPIRY_JOB_RETRY_EXHAUSTED_EVENT` trigger an automated NR alert?
  **Owner:** Saravanan — configure alert threshold and notification channel post-deploy.

---

## What to Watch Post-Deploy

**MongoDB metrics (during next 300-promotion batch update):**
- `opcounters.update` per second — should be same or lower peak, smoother curve
- WiredTiger cache eviction rate — should drop below 50 pages/sec during job runs
- `getmore` operations — should remain near zero (not using streaming cursor)
- Index usage on `{orgId, promotionId, _id}` — confirm via `$indexStats` that the compound
  index is being used and not the primary `_id` index

**Application metrics:**
- New Relic: `ExpiryDateChangeJobFacade` method duration — should be unchanged or faster
- Redis key `promotion_engine:expiry_job:running_count` — monitor value during batch runs;
  should stay ≤ `maxGlobalConcurrentJobs` and return to 0 after all jobs complete
- RMQ queue depth for `PROMOTION_EXPIRY_UPDATE_QUEUE` — should drain at a controlled rate

**Tenant isolation validation:**
- After a 300-promotion batch run, verify `impactedCount` on each `ExpiryDateChangeJob`
  matches the count of documents for that specific `orgId + promotionId` combination.
  No cross-tenant contamination is expected, but verify explicitly.

---

## Implementation Notes (Post Tech Detail)

Tech detail completed 2026-04-28. All design decisions are final. Notes for the implementer:

**Optimisation 1 — DAO changes**
- Bounded to `AbstractExpiryDateChangeDao` and `CustomerEarnedCollectionDynamicExpiryDateChangeDao`.
- Three static DAO subclasses inherit the abstract class with no override — they get the fix for free.
- `countByStatus` on `ExpiryDateChangeJobDao` is a plain Spring Data derived method — no `@Query` needed.
- The existing `ExpiryDateChangeJobFacadeTest` uses `@InjectMocks` without a `@Mock` for
  `ExpiryDateChangeJobDao`; the `dao.save()` in the catch block NPEs but is swallowed by the outer
  try-catch. Pre-existing — do not fix in this PR; document when adding new mocks.

**Optimisation 2 — Concurrency control**
- `ExpiryDateChangeHelper` has no existing dependency on `ExpiryDateChangeJobFacade`. The new
  injection is safe (no cycle confirmed). Document this dependency clearly for future developers.
- The Redis `incentives` cluster is shared across all orgs. The semaphore key
  `"promotion_engine:expiry_job:running_count"` must have **no TTL** — counter semantics require it
  to persist across job lifecycles. Verify namespace with infra team to avoid collision with other
  services on the same cluster.

**Test coverage required**
- **Unit — `AbstractExpiryDateChangeDao`:** verify aggregation produces `ids` array (not `endId`);
  verify `updateMulti` is called with `$in` criteria (not range predicates).
- **Unit — `CustomerEarnedCollectionDynamicExpiryDateChangeDao`:** update existing
  `shouldUpdateValidTillDate` test for `List<ObjectId>` signature.
- **Unit — `ExpiryDateChangeJobFacade`:** mock `@Qualifier("cacheEvictionRedisTemplate") RedisTemplate`;
  verify acquire/release on success path; verify release in `finally` on exception path; verify
  `RuntimeException` thrown when limit exceeded; verify fail-open (returns true) when Redis throws.
- **Unit — `ExpiryDateChangeHelper`:** verify `reconcileSemaphoreCounter` is called after the shard
  loop with the summed RUNNING count, not the per-shard count.
- **Integration — `PromotionExpiryDateChangeIntegrationTest`:** existing tests must pass unchanged
  (`$in` change is transparent to callers). Verify Redis counter returns to 0 after job completes.
- **Unit — `SpringAmqpConfig`:** verify `expiryDateChangeContainerFactory` bean sets
  `concurrentConsumers=1`, `maxConcurrentConsumers=1`, `prefetchCount=1` and calls
  `configurer.configure()` before overrides.
