# Arch Investigation: Reward Catalogue Listing API
**Date:** 2026-05-14
**Investigator:** Claude (arch-investigator skill)
**Status:** Ready for Tech Detail
**Confidence:** HIGH

---

## 1. Problem Statement

`sol-rewards-core` needs a Reward Catalogue Listing API with multi-filter support and < 150ms p95 SLO for enterprise orgs running up to 2000 concurrent active rewards. MySQL's normalised reward schema (multi-join, no native array filtering) cannot serve as the read model. A CQRS read side using MongoDB (`active_rewards` + `rewards_catalogue`) fed via a RabbitMQ async sync listener is the chosen architecture.

---

## 2. Codebase Map

### 2.1 Reward Write Path (MySQL — Source of Truth)

| Layer | Class | File (relative to src/main/java) |
|---|---|---|
| Controller / Resource | `RewardResource` | `resource/RewardResource.java` |
| Facade (create) | `RewardFacade.create()` | `service/impl/RewardFacade.java:1330` |
| Facade (update) | `RewardFacade.update()` | `service/impl/RewardFacade.java:1362` |
| Service (create) | `RewardService.saveNewReward()` | `service/impl/RewardService.java:88` |
| Service (update) | `RewardService.save()` | `service/impl/RewardService.java:112` |
| JDBC write | `RewardJdbcRepository` | `jdbc/RewardJdbcRepository.java` |
| JPA read | `RewardRepository` | `repository/RewardRepository.java` |

**`@Transactional` placement:**
- `RewardFacade.create()` — annotated `@Transactional(rollbackFor = {Exception.class})` at line 1329. This is the MySQL commit boundary for reward creation.
- `RewardService.save()` — annotated `@Transactional(rollbackFor = {Exception.class})` at line 112. This is the MySQL commit boundary for reward updates.
- `RewardFacade.update()` (line 1362) has **no** `@Transactional` annotation at facade level — it delegates to `rewardService.save()` which is transactional.

**afterCommit hook placement:**
- For create: register synchronization inside `RewardFacade.create()` (after `rewardService.saveNewReward()` call, still inside the `@Transactional` boundary).
- For update: register synchronization inside `RewardService.save()` (already `@Transactional`).

### 2.2 MySQL Schema — Reward-Related Tables

| Table | Entity Class | Key Columns |
|---|---|---|
| `TBL_REWARD` | `Reward.java` | `ID, ORG_ID, START_DATE, END_DATE, IS_ENABLED, INTOUCH_POINTS (= pointsValue), REDEMPTION_TYPE` |
| `REF_REWARD_DETAILS` | `RewardDetails.java` | `REWARD_ID, ORG_ID, LANGUAGE_CODE, NAME, DESCRIPTION, IMAGE_URI` |
| `TBL_REWARD_GROUP_MAPPING` | `RewardGroupMapping.java` | `ID, ORG_ID, REWARD_ID, GROUP_ID, IS_ACTIVE` |
| `REF_REWARD_LABELS_MAPPING` | `Label.java` | `ID, ORG_ID, REWARD_ID, LABEL_ID, IS_ENABLED` |
| `TBL_USER_CUSTOM_FIELD_MAPPING` | `CustomFieldMapping.java` | custom field values |

**pointsValue mapping:** `TBL_REWARD.INTOUCH_POINTS` (Integer) — already used in `RewardRepository` queries. Maps to `pointsValue` in MongoDB documents.

**groupId source:** `TBL_REWARD_GROUP_MAPPING.GROUP_ID` (Long). The `Group` entity maps to group objects; `GROUP_ID` is what goes into `groupIds[]` in the catalogue.

**label source:** `REF_REWARD_LABELS_MAPPING.LABEL_ID` (Long). The label identifier. In the MongoDB catalogue, `labels` array will contain label IDs (as Strings to match the pre-analysis contract).

### 2.3 MongoDB — Existing Setup

| Item | Detail |
|---|---|
| Database name | `sol-rewards-core` (constant `Constants.MONGO_DATABASE_NAME`) |
| Document package | `com.capillary.solutions.rewards.db.mongo` (constant `Constants.MONGO_BO_PACKAGE`) |
| Migration package | `com.capillary.solutions.rewards.migration` (constant `Constants.MONGO_MIGRATION_PACKAGE`) |
| Config class | `MongoClientConfiguration` — `@Profile("production")` |
| Index/collection init | `MongoDbInitializer.execute()` — scans document package, calls `createCollections()`. `createIndexes()` is implemented but **commented out** in `execute()` — indexes are NOT auto-created on startup. |

**Existing collections:**
| Collection | Document Class | Purpose |
|---|---|---|
| `issuedTransaction` | `IssuedTransaction.java` | Idempotency tracking |
| `pointsRetrial` | `PointsRetrial.java` | Points retry records |
| `rmqFailure` | `RmqFailure.java` | RMQ failure log |
| `rewardExpiryReminderJob` | `RewardExpiryReminderJob.java` | Expiry reminder jobs |
| `sampleInstance` | `SampleInstance.java` | Sample/test |
| `richContent` | `RichContent.java` | Rich content data |

**Confirmed: `active_rewards` collection does NOT exist.**
**Confirmed: `rewards_catalogue` collection does NOT exist.**

**Index creation approach:** `MongoDbInitializer.createIndexes()` method exists but is commented out. New indexes for the catalogue collections must be created via `ensureIndex` calls within `MongoDbInitializer.execute()` or a new `@PostConstruct` bean. The safest approach is to re-enable `createIndexes()` for the two new document classes, or add explicit `ensureIndex` calls in `MongoDbInitializer`. See section 4.3 for recommendation.

**MongoRepository vs MongoTemplate:** Existing DAOs in `mongoDao/` use `MongoRepository` (Spring Data). For the catalogue listing with complex filter queries (multiple `$in`, `maxTimeMS`, cursor pagination), `MongoTemplate` with manual query construction is required — matching the pre-analysis spec.

### 2.4 Redis — Existing Cache Setup

| TTL Bucket | Constant in `RedisCacheKeys` | TTL |
|---|---|---|
| `ONE_MINUTE_CACHE` | `RedisCacheKeys.ONE_MINUTE_CACHE` | 1 minute |
| `FIVE_MINUTE_CACHE` | `RedisCacheKeys.FIVE_MINUTE_CACHE` | 5 minutes |
| `ONE_HOUR_CACHE` | `RedisCacheKeys.ONE_HOUR_CACHE` | 1 hour |
| `TWO_HOURS_CACHE` | `RedisCacheKeys.TWO_HOURS_CACHE` | 2 hours |
| `ONE_DAY_CACHE` | `RedisCacheKeys.ONE_DAY_CACHE` | 1 day |

**Gap:** No 30-second TTL bucket exists. The pre-analysis requires 30s TTL for catalogue listing cache. A new bucket `THIRTY_SECOND_CACHE` must be added to both `RedisCacheUtil.getRedisCacheConfig()` and `RedisCacheKeys`. Both files must be updated together per code guardrail.

**Redis key pattern for catalogue cache:** `catalogue:{orgId}:{filterHash}:{page}` — stored via `RedisTemplate<String, String>` (not `@Cacheable` annotation, as prefix-based invalidation requires manual `keys()` scan or `SCAN`+`DEL`). Existing code already injects `RedisTemplate<String, String>` (seen in `RewardService`).

**Cache invalidation approach:** `catalogue:{orgId}:*` pattern invalidation. Must use `RedisTemplate.execute(connection -> connection.scan(...))` with a SCAN pattern or store a set of active keys per org. Given Redis Sentinel setup, prefix-based pattern scan is the safest approach.

### 2.5 RabbitMQ — Existing Configuration

**Config class:** `SpringAmqpConfig.java` — `@Configuration`, defines all exchanges/queues/bindings.

**Existing exchanges (all DirectExchange):**
| Exchange | Purpose |
|---|---|
| `points.redemption.main.exchange` | Points redemption retry |
| `points.redemption.dlx` | DLX for points redemption |
| `REWARDS_BULK_EXCHANGE` | Bulk reward upload |
| `reward_scheduler_exchange` | Cron scheduler events |
| `REWARD_EXPIRY_REMINDER_INTERNAL_EXCHANGE` | Expiry reminder |
| `TRANSACTION_ACTION_LOGS_EXCHANGE` | Transaction action logs |
| `API_MIRROR_CALL_EXCHANGE` | API migration |

**Pre-analysis requires TopicExchange** for `rewards.catalogue.exchange`. This is a new exchange type not yet used in this service but valid Spring AMQP.

**Publishing pattern (existing):** `RabbitTemplate.convertAndSend(exchange, routingKey, message)` — used in `UploadFacade`, `RewardExpiryReminderService`, `PointsRedeemProcessor`, `PointsRedemptionService`, `UserRewardUtils`. The message serialized via `new Gson().toJson(object)` (String body) in several places.

**Listener pattern (existing):** `@RabbitListener(queues = "queueName")` on `@Service` class methods. Decorated with `@Trace(dispatcher = true, metricName = ...)` for New Relic. MDC set from message properties. Uses `RMQMessageTrackerAspect` via `@RmqMessageTracker(name = ...)`.

**Batch listener:** Not yet used. `SpringAmqpConfig` has no `BatchListenerContainerFactory` bean. A new `@Bean batchListenerFactory()` must be added.

**DLQ pattern (existing):** `QueueBuilder.durable(...).withArgument("x-dead-letter-exchange", dlx)` — used for `points.redemption.main.queue` with explicit DLX + DLQ pair. Same pattern to apply for catalogue sync queue.

**SimpleMessageConverter allowed classes:** `converter()` bean at `SpringAmqpConfig:173` allows `com.capillary.solutions.rewards.*`, `java.util.*`, `java.lang.*`. The new `RewardCatalogueEvent` message class in `com.capillary.solutions.rewards.*` package will be automatically allowed.

### 2.6 No Existing afterCommit Patterns

Grep confirmed: **zero existing uses** of `TransactionSynchronizationManager.registerSynchronization()` or `afterCommit()` in the codebase. The implementation will introduce this pattern for the first time. The `create()` method in `RewardFacade` at line 1329 is `@Transactional` and is the correct attachment point for creation events.

---

## 3. Open Questions — Resolved

Per pre-analysis section 9, the following are resolved here for the tech-detailer:

1. **Cursor token format:** Base64-encoded JSON `{ "pointsValue": N, "_id": "hex" }` decoded server-side. Opaque to client. No Redis storage needed for cursor.

2. **Lang resolution:** Single locale resolved server-side using `Accept-Language` header (default `en` — matching `Constants.DEFAULT_LANGUAGE`). Only the resolved name is returned in `RewardSummary`. Full `langDetails` map is stored in `rewards_catalogue` but not exposed in listing API.

3. **Unfiltered query guard:** Silently apply `limit=50`. No 400 rejection. Log a WARN with `orgId` and `db.loadCount` for observability.

4. **Batch activation job:** No new scheduled job. The RMQ batch listener handles the month-start burst natively (2000 messages, batch consumer). No `@Scheduled` bean required.

5. **`pointsExact` vs range mutual exclusivity:** Validated at controller layer — return HTTP 400 if both `pointsExact` AND (`pointsMin` OR `pointsMax`) are provided.

6. **DLQ alerting:** No new alerting infrastructure needed. The existing shared intouch RMQ monitoring covers DLQ depth. Document the DLQ name `rewards-catalogue-sync-dlq` in the service wiki post-deploy.

---

## 4. Implementation Anchors for Tech Detailer

### 4.1 afterCommit Hook Placement

```
RewardFacade.create() [line 1329, @Transactional]
  └── rewardService.saveNewReward(orgId, reward, ...)
  └── [register TransactionSynchronization here — inside the @Transactional boundary]
       └── afterCommit() → publish UPSERT event

RewardService.save() [line 112, @Transactional]
  └── rewardJdbcRepository.update(reward)
  └── [register TransactionSynchronization here]
       └── afterCommit() → publish UPSERT event
```

For reward disable (isEnabled = false update): goes through the same `update()` path — the UPSERT event is published, listener fetches from MySQL and sees `isEnabled=false`, removes from `active_rewards`.

For reward delete: if a delete path exists, publish DELETE event. Search `RewardFacade` for delete method.

### 4.2 MongoDB Document Classes

New classes go in `com.capillary.solutions.rewards.db.mongo` (package already scanned by `MongoDbInitializer`).

```java
// RewardCatalogueDoc.java — @Document(collection = "rewards_catalogue")
// ActiveRewardDoc.java — @Document(collection = "active_rewards")
```

Both will be auto-discovered by `MongoDbInitializer.getMongoDocumentClasses()` on startup, ensuring collection creation.

### 4.3 Index Creation Strategy

`MongoDbInitializer.execute()` currently calls `createCollections()` but NOT `createIndexes()`. For the two new collections, indexes must be created explicitly. **Recommended approach:** Add a new dedicated `CatalogueIndexInitializer` `@Component` with `@PostConstruct` that calls `mongoTemplate.indexOps(ActiveRewardDoc.class).ensureIndex(...)` for each required index. This isolates catalogue index management and avoids re-enabling the commented-out `createIndexes()` call (which would affect all existing collections).

TTL index for `active_rewards.expiresAt` requires `expireAfterSeconds: 0` — use:
```java
mongoTemplate.indexOps(ActiveRewardDoc.class)
    .ensureIndex(new Index("expiresAt", Sort.Direction.ASC).expire(0));
```

### 4.4 Redis Cache — New Bucket Required

Add to `RedisCacheKeys`:
```java
public static final String THIRTY_SECOND_CACHE = "THIRTY_SECOND_CACHE";
public static final String CATALOGUE_CACHE = "'catalogue:'";
```

Add to `RedisCacheUtil.getRedisCacheConfig()`:
```java
RedisCacheConfiguration thirtySecondCacheConfig = baseConfig.entryTtl(Duration.ofSeconds(30));
configurationMap.put(RedisCacheKeys.THIRTY_SECOND_CACHE, thirtySecondCacheConfig);
```

**Cache key pattern:** `catalogue:{orgId}:{filterHash}:{page}` — manual `RedisTemplate` operations (not `@Cacheable`) because invalidation requires `SCAN` on prefix `catalogue:{orgId}:*`.

### 4.5 RMQ New Exchange/Queue/Binding

Add to `SpringAmqpConfig.java`:
```
Exchange: rewards.catalogue.exchange  (TopicExchange, durable)
Queue:    rewards-catalogue-sync-queue (durable, x-dead-letter-exchange → rewards.catalogue.dlx)
DLX:      rewards.catalogue.dlx (DirectExchange)
DLQ:      rewards-catalogue-sync-dlq (durable)
Routing:  rewards.catalogue.sync
```

Batch listener factory bean:
```java
@Bean
public SimpleRabbitListenerContainerFactory batchListenerFactory(
    ConnectionFactory connectionFactory, SimpleMessageConverter converter) {
    SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
    factory.setConnectionFactory(connectionFactory);
    factory.setMessageConverter(converter);
    factory.setBatchListener(true);
    factory.setConsumerBatchEnabled(true);
    factory.setBatchSize(200);
    return factory;
}
```

### 4.6 Reward Fetch for Listener (MySQL → MongoDB sync)

The listener receives `{ orgId, rewardId, operation }` and fetches fresh state. Use existing `RewardRepository.findByOrgIdAndId(orgId, rewardId)` for single reward fetch. For batch: `RewardRepository.findByOrgIdAndIdIn(orgId, rewardIds)`.

For group IDs: `RewardGroupMappingMetaRepository` or `rewardGroupMappingJdbcRepository` — check for fetch method by `orgId + rewardId`.

For labels: `LabelService.getLabelsForReward(orgId, rewardId)` — verify this exists, or query `Label` entities via JPA repository.

For lang details (for `rewards_catalogue`): `RewardRepository.getRewardDetails(orgId, rewardId)` returns `List<RewardDetails>` — contains all language variants.

For custom fields: `CustomFieldService.getCustomFields(orgId, rewardId, ...)` — verify signature.

### 4.7 API Path

Per existing `URLPrefix`:
```
GET /core/v1/rewards/catalogue
```
Path constant to add: `public static final String V1_REWARDS_CATALOGUE = V1 + "/rewards/catalogue";`

### 4.8 Test Infrastructure

- Tests use JUnit 5 + Mockito only. No embedded MongoDB or RabbitMQ in existing tests.
- New tests for listener and repositories: use Mockito to mock `MongoTemplate`, `RabbitTemplate`, `RewardRepository`.
- Integration tests that require real MongoDB: use `@ExtendWith(MockitoExtension.class)` with mocked MongoTemplate for unit tests of service/repository logic.
- Test pattern: `@ExtendWith(MockitoExtension.class)` class, `@InjectMocks` on the class under test, `@Mock` on dependencies.

---

## 5. Evidence Trail

| File | Location | Finding |
|---|---|---|
| `service/impl/RewardFacade.java` | Line 1329 | `create()` is `@Transactional` — afterCommit registration point |
| `service/impl/RewardService.java` | Line 112 | `save()` is `@Transactional` — afterCommit registration point for updates |
| `queue/SpringAmqpConfig.java` | Line 13 | All exchanges are DirectExchange; no TopicExchange yet; new catalogue exchange must be added |
| `config/mongo/MongoDbInitializer.java` | Line 47 | `createIndexes()` is commented out in `execute()` — new indexes need separate bean |
| `config/mongo/MongoClientConfiguration.java` | Line 51 | Database name is `sol-rewards-core` |
| `domain/Constants.java` | Line 219,226 | `MONGO_DATABASE_NAME`, `MONGO_BO_PACKAGE` — new doc classes go in `db.mongo` |
| `utils/RedisCacheKeys.java` | Line 7-45 | No 30s TTL bucket — must add `THIRTY_SECOND_CACHE` |
| `utils/RedisCacheUtil.java` | Line 161-186 | `getRedisCacheConfig()` — add 30s entry |
| `db/Reward.java` | Line 63 (IS_ENABLED), 54 (START_DATE), 58 (END_DATE), 70 (INTOUCH_POINTS) | Core filter fields map to `isEnabled`, `startDate`, `endDate`, `pointsValue` |
| `db/RewardGroupMapping.java` | Line 37 | `GROUP_ID` (Long) → `groupIds[]` in catalogue |
| `db/Label.java` | Line 40 | `LABEL_ID` (Long) → `labels[]` in catalogue |
| `db/RewardDetails.java` | Line 38 | `NAME`, `IMAGE_URI` → `name`, `imageUrl` in catalogue |
| `mongoDao/IssuedTransactionDao.java` | Line 9 | MongoRepository pattern — NOT for catalogue (MongoTemplate needed for complex queries) |

---

## 6. System Map (Affected Path)

**Write path (augmented):**
```
RewardResource → RewardFacade.create() [@Transactional]
                  └── RewardService.saveNewReward() → RewardJdbcRepository → MySQL TBL_REWARD
                  └── [afterCommit] → CatalogueEventPublisher → rewards.catalogue.exchange
                                           → rewards-catalogue-sync-queue
                                           → RewardCatalogueSyncListener
                                                → RewardRepository (MySQL fetch)
                                                → RewardCatalogueMongoRepo (upsert rewards_catalogue)
                                                → ActiveRewardMongoRepo (upsert/delete active_rewards)
                                                → CatalogueCache.invalidate(orgId)
```

**Read path (new):**
```
GET /core/v1/rewards/catalogue
  → RewardCatalogueResource
  → RewardCatalogueService
  → CatalogueCache.get(orgId, filterHash, page)
      └── HIT  → return cached response
      └── MISS → ActiveRewardMongoRepo.findFiltered(orgId, filters, cursor, limit) [maxTimeMS=100]
               → RewardCatalogueMongoRepo.findByIds(orgId, rewardIds) [projection]
               → merge + build RewardCatalogueResponse
               → CatalogueCache.put(orgId, filterHash, page, response, 30s TTL)
```

**Tenant isolation points:**
- Every MongoDB query has `orgId` as first filter field
- Every Redis key prefixed with `orgId`
- RMQ message carries `orgId` — listener scopes all MySQL and MongoDB ops to that `orgId`

**Upstreams called:** MySQL (RewardRepository), MongoDB (two collections), Redis

**Downstreams affected:** None — read-only API path

---

## 7. Risks for Implementer

| Risk | Likelihood | Mitigation |
|---|---|---|
| `MongoDbInitializer` doesn't create indexes for new collections | HIGH (indexes commented out) | Add `CatalogueIndexInitializer @PostConstruct` bean |
| afterCommit fires after transaction but RabbitMQ is down | LOW | RabbitTemplate has retry config; message lost → DLQ alert needed |
| `rewards_catalogue` grows unboundedly (historical rewards accumulate) | MEDIUM | TTL is only on `active_rewards`; `rewards_catalogue` retains history by design |
| Pattern-scan Redis invalidation (`catalogue:{orgId}:*`) slow on large keyspace | LOW | Catalogue keys are few per org; 30s TTL limits accumulation |
| Batch listener deserializing mixed-org events in single batch | MEDIUM | Batch processor must group by orgId before MySQL IN queries |
| `isActive` field on `RewardGroupMapping` — must filter only active group mappings | MEDIUM | Query `WHERE IS_ACTIVE = true` when fetching group IDs for catalogue upsert |

---

## 8. Handoff Note for tech-detailer

### What to build (classes to create)

**New classes — code layer:**
1. `RewardCatalogueResource` (`resource/` package) — GET /core/v1/rewards/catalogue
2. `RewardCatalogueService` (`service/` package) — orchestrates cache + MongoDB queries
3. `ActiveRewardMongoRepository` (`mongoDao/` or new `catalogue/` sub-package) — MongoTemplate queries on `active_rewards`
4. `RewardCatalogueMongoRepository` (`mongoDao/`) — MongoTemplate queries on `rewards_catalogue`
5. `RewardCatalogueSyncListener` (`queue/`) — batch RMQ listener
6. `RewardCatalogueEventPublisher` (`service/` or `queue/`) — afterCommit publisher
7. `CatalogueCache` (`service/` or `utils/`) — Redis get/put/invalidate

**New classes — data model:**
8. `RewardCatalogueDoc` (`db/mongo/`) — MongoDB document, `@Document(collection = "rewards_catalogue")`
9. `ActiveRewardDoc` (`db/mongo/`) — MongoDB document, `@Document(collection = "active_rewards")`
10. `RewardCatalogueEvent` — message payload class for RMQ

**New classes — DTOs:**
11. `RewardCatalogueRequest` — query param DTO
12. `RewardCatalogueResponse` — response wrapper (items, nextCursor, total)
13. `RewardSummaryDto` — per-item response

**Modifications to existing classes:**
- `RewardFacade.create()` — add afterCommit synchronization registration
- `RewardService.save()` — add afterCommit synchronization registration
- `SpringAmqpConfig` — add catalogue exchange/queue/DLQ/binding beans + batchListenerFactory bean
- `RedisCacheKeys` — add `THIRTY_SECOND_CACHE` + `CATALOGUE_CACHE` constants
- `RedisCacheUtil.getRedisCacheConfig()` — add 30s TTL entry
- `URLPrefix` — add `V1_REWARDS_CATALOGUE` constant

**New infrastructure bean:**
- `CatalogueIndexInitializer` — `@Component` with `@PostConstruct` to `ensureIndex` all 7 indexes on `active_rewards` and 3 indexes on `rewards_catalogue`

### Key contracts to validate
- `RewardRepository.findByOrgIdAndId(orgId, rewardId)` — confirm signature and return type `Reward`
- `RewardRepository.findByOrgIdAndIdIn(orgId, rewardIds)` — confirm signature for batch fetch
- Fetch group IDs for a reward: check `RewardGroupMappingMetaRepository` or `rewardGroupMappingJdbcRepository` for `findByOrgIdAndRewardId` method
- Fetch labels for a reward: check `Label` JPA entity and repository for `findByOrgIdAndRewardId(orgId, rewardId)` — note `IS_ACTIVE = true` filter
- `RedisTemplate<String, String>` bean — already declared in `RewardService`; inject directly

### Suggested test emphasis
- **Unit — afterCommit:** verify publish NOT called when transaction rolls back; verify UPSERT event published on successful create
- **Unit — listener:** verify listener calls MySQL fetch (not trusts payload), handles not-found gracefully (reward deleted between publish and consume)
- **Unit — ActiveRewardMongoRepository:** verify `orgId` always first in query filter, `maxTimeMS(100)` applied, cursor pagination produces correct `$gt` predicate
- **Unit — CatalogueCache:** verify invalidation pattern `catalogue:{orgId}:*` scoped to org, not cross-org
- **Unit — filter combinations:** isEnabled only, groupId filter, label filter, pointsExact, pointsMin+pointsMax, all filters simultaneously, no filters (unfiltered guard)
