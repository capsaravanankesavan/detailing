# Tech Detail: Reward Catalogue Listing API
**Date:** 2026-05-14
**Author:** Claude (tech-detailer skill)
**Status:** Draft
**Investigation Doc:** `arch-investigation.md`
**Confidence:** HIGH

---

## 1. Problem Statement

`sol-rewards-core` needs a Reward Catalogue Listing API (`GET /core/v1/rewards/catalogue`) with filter support for active/enabled rewards. Enterprise orgs have up to 2000 concurrent active rewards with high monthly churn. MySQL's normalised schema (4-5 JOINs, no array-filter support) cannot serve listing at < 150ms p95. A CQRS read model is built using MongoDB (`active_rewards` TTL collection + `rewards_catalogue` full catalogue), fed asynchronously from all reward mutation paths via a RabbitMQ batch listener.

---

## 2. Root Cause (Confirmed)

No efficient read model exists for reward catalogue listing — the MySQL schema is optimised for writes and single-reward lookups, not for filtered listing across 2000 active rewards per org. The new MongoDB CQRS read side addresses this directly.

### Contributing Factors
1. MySQL `TBL_REWARD` has no compound index covering `(ORG_ID, IS_ENABLED, START_DATE, END_DATE)` in a way that also supports array filtering on groups or labels.
2. Group and label associations are in separate join tables (`TBL_REWARD_GROUP_MAPPING`, `REF_REWARD_LABELS_MAPPING`), requiring joins for every listing query.
3. As expired rewards accumulate (2000/month × 12 months = 24,000 docs), any date-based filter query degrades over time without a separate active set.

---

## 3. Scope of Change

### In Scope
- New `GET /core/v1/rewards/catalogue` API endpoint
- MongoDB `active_rewards` TTL collection + `rewards_catalogue` full collection
- RMQ `rewards.catalogue.exchange` + `rewards-catalogue-sync-queue` + DLQ
- Batch RMQ listener `RewardCatalogueSyncListener`
- `afterCommit` publishing from reward create and update paths
- Direct publish from reward enable/disable toggle path (non-transactional JDBC)
- Redis 30s TTL cache for listing results
- Index initializer bean for both MongoDB collections
- New DTOs: request, response, event

### Out of Scope (Explicit)
- MySQL schema changes — no new tables or columns
- Kafka consumer — not in scope per pre-analysis decision §3.2
- `rewards_catalogue` TTL — history retained by design
- Reward delete API — does not exist in the codebase; no DELETE event needed
- B2C user-reward listing — separate `UserRewardResource` flow
- Re-enabling `MongoDbInitializer.createIndexes()` globally — scoped to catalogue only

### Deferred
- Kafka migration path (retire RMQ if Kafka events gain `orgId` + `operation`)
- Real-time index-based alerting on DLQ depth
- `rewards_catalogue` archival job when history exceeds storage threshold

---

## 4. Assumptions

| # | Assumption | Breaks if... | Owner to validate |
|---|-----------|-------------|------------------|
| 1 | `Reward.intouchPoints` (Integer) is the `pointsValue` for catalogue filtering | Rewards use a different points field for catalogue display | Saravanan |
| 2 | `Label.labelId` (Long) identifies the label dimension for filtering | Labels are identified by a different field | Confirmed from `Label.java:40` |
| 3 | `RewardGroupMapping.IS_ACTIVE = true` filters active group associations | Inactive group mappings should still appear in catalogue | Confirmed — `IS_ACTIVE` column exists |
| 4 | Eventual consistency lag of seconds is acceptable for catalogue listing | Consumer reports stale catalogue data as a defect | Pre-analysis decision §3.2 |
| 5 | Redis Sentinel SCAN supports key prefix patterns under expected load | SCAN is blocked or times out on Sentinel under peak load | Low risk — catalogue keys < 200/org |
| 6 | `MongoClientConfiguration` `@Profile("production")` means test profile uses embedded/mock Mongo | Tests fail with Mongo connection in test profile | Confirmed from existing test patterns |
| 7 | `updateStatus()` in `RewardFacade` auto-commits via JDBC (no Spring TX) | JDBC data source is wrapped in a JTA transaction manager | Confirmed — JDBC auto-commit pattern is standard in this codebase |

---

## 5. Design Validation

### Gaps vs Existing Patterns — CAUGHT

**GAP [HIGH]: enable/disable toggle path missing from afterCommit design**
- `RewardFacade.updateStatus()` at line 798 is NOT `@Transactional`. It calls `rewardJdbcRepository.updateStatus()` which auto-commits via JDBC.
- The handoff doc only covered create (`RewardFacade.create()`) and update (`RewardService.save()`) for afterCommit registration.
- **Fix:** Publish catalogue sync event directly from `RewardFacade` line ~1651 inside `if (updateCount > 0)` block. Since JDBC has already committed at that point, direct publish (not afterCommit) is correct and safe.

**GAP [MED]: TopicExchange not yet used in service**
- All existing exchanges in `SpringAmqpConfig` are `DirectExchange`. Pre-analysis wants `TopicExchange` for `rewards.catalogue.exchange`.
- **Resolution:** `TopicExchange` is valid Spring AMQP and works identically to `DirectExchange` for the single routing key pattern used here. However, to be consistent with existing codebase patterns, a `DirectExchange` with routing key `rewards.catalogue.sync` achieves identical behavior and is lower friction for ops. Adopting `DirectExchange` for the catalogue exchange.

**GAP [MED]: Batch listener groups events by orgId before MySQL IN queries**
- The pre-analysis batch sketch does `events.stream().map(RewardCatalogueEvent::getRewardId).toList()` — assumes single orgId per batch. In practice, a batch may contain events for multiple orgs.
- **Fix:** Group events by `orgId`, then execute one MySQL IN query per org group, and one MongoDB bulkWrite per org group. This is critical for correct tenant isolation.

**GAP [LOW]: `RewardCatalogueEvent` message serialisation format**
- Existing RMQ publishers use `Gson().toJson(object)` as String body. The batch listener receives `List<Message>` or `List<RewardCatalogueEvent>`. If using `SimpleMessageConverter`, the batch deserialisation needs the event class in the allowed list.
- **Resolution:** Serialize `RewardCatalogueEvent` as JSON String body (matching existing Gson pattern). Batch listener receives `List<String>` and deserializes each entry via Gson.

### MADR Compliance
- **MADR-0001 (Write via JDBC, Read via Hibernate):** New catalogue write path (listener → MongoDB) does NOT touch `rewards_utf8_db` MySQL write layer — COMPLIANT.
- **MADR-0002 (DateTime ISO 8601):** `startDate` and `endDate` in API response must be formatted as `yyyy-MM-dd'T'HH:mm:ss'Z'` using `DateTimeService.getFormattedDateInGMT()`. MongoDB stores as native BSON Date (`java.util.Date`) per infra.md guardrail — COMPLIANT.

### Guardrail Compliance

| Rule | Status |
|---|---|
| GC: pre-size collections in batch processor | MUST apply — `new ArrayList<>(events.size())` |
| GC: no object allocation in tight loops | MUST apply — build response DTOs once per result set |
| Caching: new 30s TTL bucket added to BOTH `RedisCacheUtil` and `RedisCacheKeys` together | REQUIRED |
| Caching: no new ad-hoc `RedisCacheConfiguration` outside `getRedisCacheConfig()` | COMPLIANT — adding to `getRedisCacheConfig()` |
| JDBC: orgId in every write — N/A for MongoDB writes | N/A |
| JPA: every query includes orgId predicate | COMPLIANT — all new JPA queries scoped by orgId |
| Logging: API entry INFO log + response time | REQUIRED in `RewardCatalogueResource` |
| Logging: `db.loadCount` and `response.count` for listing API | REQUIRED |
| Observability: New Relic `ORG_ID`, `REQUEST_ID` attributes | REQUIRED |
| Date-time: MongoDB fields use BSON Date (`java.util.Date`) | COMPLIANT |
| Date-time: response DTO date fields as ISO String via `DateTimeService` | REQUIRED |
| Code: no magic strings/numbers | REQUIRED — all constants via `Constants` / `RedisCacheKeys` |
| Collections: `isEmpty()` not `size() == 0` | REQUIRED |
| RMQ: `RMQMessageTrackerAspect` via `@RmqMessageTracker` on new listener | REQUIRED |

---

## 6. Alternative Designs Considered

| Alternative | Pros | Cons | Decision |
|------------|------|------|----------|
| DirectExchange instead of TopicExchange | Consistent with all existing exchanges; simpler ops | No routing flexibility if multiple consumers added later | **Accepted** — consistency over flexibility |
| Single collection (no separate `active_rewards`) with compound date-range index | Simpler — one collection, no sync between two | Degrades over time as expired docs accumulate; no TTL purge | **Rejected** — violates pre-analysis §3.3 |
| Synchronous dual-write (MySQL + MongoDB in same request) | Zero lag | Violates pre-analysis §8 guardrail; adds MongoDB write latency to user-facing API | **Rejected** |
| `@Cacheable` instead of manual `RedisTemplate` | Less boilerplate | Cannot do prefix-based per-org invalidation with Spring Cache abstraction | **Rejected** — manual `RedisTemplate` required for `catalogue:{orgId}:*` SCAN |

---

## 7. Use Cases

### B2B (Brand Admin Flows)
| ID | Flow | Today | After Change | Risk | Test Type |
|----|------|-------|-----------|------|-----------|
| UC-1 | Admin creates 2000 rewards at month start → catalogue listing reflects new rewards | API returns MySQL data with join overhead | RMQ burst processed by batch listener; `active_rewards` populated within seconds | Stale listing during processing window | Unit (listener) + Integration |
| UC-2 | Admin updates reward pointsValue → listing reflects new value | Not available | afterCommit publishes UPSERT; listener re-syncs `active_rewards.pointsValue` | Stale value during sync lag | Unit (afterCommit) |
| UC-3 | Admin disables reward → it disappears from active listing | Not available | `updateStatus()` publishes UPSERT; listener sees `isEnabled=false`, removes from `active_rewards` | Reward stays visible until RMQ processed | Unit (toggle path) |
| UC-4 | Admin creates reward then immediately queries catalogue | Not available | Eventual consistency: reward may not appear for up to seconds | Caller sees stale empty result briefly | Integration |
| UC-5 | Two admins update the same reward concurrently | Not available | Both publish UPSERT; listener processes both; last MySQL fetch wins (safe — upsert semantics) | Transient inconsistency, resolved on 2nd message | Unit (idempotency) |
| UC-6 | Reward's `endDate` arrives — TTL purges from `active_rewards` | Not available | MongoDB TTL thread deletes within 60s of `endDate`; no app code needed | Reward appears in listing for up to 60s after expiry | Integration (TTL) |
| UC-7 | Admin calls GET with `pointsExact` AND `pointsMin` in same request | Not available | 400 Bad Request at controller layer | Silent filter confusion if not validated | Unit (validation) |
| UC-8 | Org A admin queries catalogue; Org B has 2000 rewards | Not available | Org A's `active_rewards` query is bounded by `{ orgId: A, ... }` — Org B data not touched | Cross-tenant data leak | Unit (orgId isolation) |
| UC-9 | RMQ listener fails to upsert MongoDB → message parks in DLQ | Not available | Manual replay (upsert semantics make it safe) | Stale catalogue until replayed | Unit (DLQ) |
| UC-10 | Reward does not exist in MySQL when listener processes UPSERT (deleted between publish and consume) | Not available | Listener fetches null from MySQL → logs WARN, acks message, returns | Stale `active_rewards` entry if reward was active | Unit |

### B2C (End Customer Flows)
| ID | Flow | Today | After Change | Risk | Test Type |
|----|------|-------|-----------|------|-----------|
| UC-11 | Customer app calls `GET /core/v1/rewards/catalogue?orgId=X` (no filters) | Not available | Returns up to 50 enabled active rewards, sorted by pointsValue ascending | Too many results without guidance | Unit (unfiltered guard) |
| UC-12 | Customer app calls with `groupIds=G1,G2` — customer eligible for groups G1 and G2 | Not available | `active_rewards` query with `groupIds: { $in: [G1, G2] }` → matching rewards returned | Wrong rewards if groupId type mismatch (Long vs String) | Unit (filter) |
| UC-13 | Customer app calls with `cursor` from previous page | Not available | Cursor decoded to `{ pointsValue, _id }` → `$or: [ { pointsValue: { $gt: V } }, { pointsValue: V, _id: { $gt: id } } ]` | Wrong page boundary if cursor tampered | Unit (cursor) |
| UC-14 | Customer calls catalogue; cache HIT | Not available | Redis lookup returns in < 10ms | Cache returning stale result beyond 30s window | Unit (cache) |
| UC-15 | Customer calls catalogue; cache MISS → MongoDB → Redis SET | Not available | Full path: `active_rewards` query + `rewards_catalogue` projection join + cache write | MongoDB query times out (`maxTimeMS=100`) → 500 | Unit (timeout handling) |
| UC-16 | Org has 0 active rewards | Not available | `active_rewards.find()` returns 0 docs; response: `{ items: [], nextCursor: null, total: 0 }` | Null pointer on empty result handling | Unit (empty org) |
| UC-17 | Unknown `orgId` provided | Not available | `active_rewards.find({ orgId: X })` returns 0 docs; same response as empty org | No 404 (correct — org may exist with no rewards) | Unit |
| UC-18 | Customer calls with `labels=L1,L2` — reward has label L1 | Not available | `$in: [L1, L2]` on `active_rewards.labels` → matches reward with L1 | Incorrect if labels stored as Long not String | Unit (type fidelity) |

---

## 8. Low-Level Design

### 8.1 New Classes

---

#### `RewardCatalogueDoc` — `db/mongo/RewardCatalogueDoc.java`

```java
@Document(collection = "rewards_catalogue")
@CompoundIndexes({
    @CompoundIndex(def = "{'orgId': 1, 'rewardId': 1}", unique = true, background = true),
    @CompoundIndex(def = "{'orgId': 1, 'isEnabled': 1, 'startDate': 1, 'endDate': 1}", background = true),
    @CompoundIndex(def = "{'orgId': 1, 'updatedAt': 1, '_id': 1}", background = true)
})
public class RewardCatalogueDoc {
    @Id  private ObjectId id;
    private Long orgId;
    private Long rewardId;          // business ID
    private boolean isEnabled;
    private Date startDate;         // BSON Date per infra.md
    private Date endDate;           // BSON Date per infra.md
    private List<Long> groupIds;    // Long, not String — matches DB type
    private List<Long> labelIds;    // Long — label IDs from REF_REWARD_LABELS_MAPPING
    private Integer pointsValue;    // from INTOUCH_POINTS
    private Map<String, String> langDetails;  // languageCode → name
    private String imageUrl;        // from REF_REWARD_DETAILS.IMAGE_URI (default lang)
    private Map<String, Object> customFields;
    private Date syncedAt;          // last synced from MySQL
    private Date updatedAt;
}
```

**Note:** `groupIds` and `labelIds` are stored as `Long` (matching MySQL column types `GROUP_ID Long`, `LABEL_ID Long`) — not String. The API response serialises them as strings if needed.

---

#### `ActiveRewardDoc` — `db/mongo/ActiveRewardDoc.java`

```java
@Document(collection = "active_rewards")
@CompoundIndexes({
    @CompoundIndex(def = "{'orgId': 1, 'rewardId': 1}", unique = true, background = true),
    @CompoundIndex(def = "{'orgId': 1, 'groupIds': 1, 'pointsValue': 1}", background = true),
    @CompoundIndex(def = "{'orgId': 1, 'labelIds': 1, 'pointsValue': 1}", background = true),
    @CompoundIndex(def = "{'orgId': 1, 'pointsValue': 1}", background = true),
    @CompoundIndex(def = "{'orgId': 1, 'isEnabled': 1}", background = true)
})
public class ActiveRewardDoc {
    @Id  private ObjectId id;
    private Long orgId;
    private Long rewardId;
    private Date expiresAt;         // TTL field = endDate
    private boolean isEnabled;
    private List<Long> groupIds;    // multikey indexed
    private List<Long> labelIds;    // multikey indexed
    private Integer pointsValue;
}
```

**TTL index** added via `CatalogueIndexInitializer`, not `@Indexed` annotation (the `expireAfterSeconds` in `@Indexed` requires seconds at annotation time, which works for fixed TTLs, but `expireAfterSeconds=0` means "expire at `expiresAt` time" — this is best expressed programmatically).

---

#### `RewardCatalogueEvent` — `queue/RewardCatalogueEvent.java`

```java
public class RewardCatalogueEvent {
    private Long orgId;
    private Long rewardId;
    private String operation;       // "UPSERT" | "DELETE"
    private String triggeredAt;     // ISO 8601 String
}
```

Serialised/deserialised via Gson (matching existing RMQ pattern).

---

#### `CatalogueIndexInitializer` — `config/mongo/CatalogueIndexInitializer.java`

```java
@Component
@Profile("production")          // same guard as MongoClientConfiguration
@Slf4j
public class CatalogueIndexInitializer {

    @Autowired
    private MongoTemplate mongoTemplate;

    @PostConstruct
    public void ensureIndexes() {
        // TTL index on active_rewards.expiresAt
        mongoTemplate.indexOps(ActiveRewardDoc.class)
            .ensureIndex(new Index("expiresAt", Sort.Direction.ASC).expire(0));

        // All @CompoundIndexes on both doc classes are created by MongoDbInitializer
        // (which calls ensureIndex via createIndexes). Only the TTL index needs
        // special handling here.
        log.info("Catalogue MongoDB indexes ensured");
    }
}
```

**Note on compound index creation:** The `@CompoundIndexes` annotations on both document classes will be picked up by `MongoDbInitializer.createIndexes()` if we un-comment the `createIndexes()` call. However, to avoid touching existing collection indexes, a safer approach is to call `ensureIndex` for the catalogue-specific indexes directly in `CatalogueIndexInitializer`. The full list is given in section 9.

---

#### `RewardCatalogueEventPublisher` — `queue/RewardCatalogueEventPublisher.java`

```java
@Service
@Slf4j
public class RewardCatalogueEventPublisher {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void publishUpsert(Long orgId, Long rewardId) {
        publish(orgId, rewardId, "UPSERT");
    }

    private void publish(Long orgId, Long rewardId, String operation) {
        try {
            RewardCatalogueEvent event = new RewardCatalogueEvent();
            event.setOrgId(orgId);
            event.setRewardId(rewardId);
            event.setOperation(operation);
            event.setTriggeredAt(Instant.now().toString());
            rabbitTemplate.convertAndSend(
                SpringAmqpConfig.REWARDS_CATALOGUE_EXCHANGE,
                SpringAmqpConfig.REWARDS_CATALOGUE_ROUTING_KEY,
                new Gson().toJson(event)
            );
            log.info("Published catalogue sync event orgId={} rewardId={} op={}", orgId, rewardId, operation);
        } catch (Exception e) {
            log.error("Failed to publish catalogue sync event orgId={} rewardId={} op={}", orgId, rewardId, operation, e);
        }
    }
}
```

**Guardrail:** Exceptions are caught and logged — publish failure must NOT propagate back to the caller (reward write already succeeded). The catalogue will self-heal on the next mutation event.

---

#### Modifications to `RewardFacade.create()` — Line 1339

After `rewardService.saveNewReward(...)` succeeds (line 1339), register afterCommit:

```java
// Inside RewardFacade.create() — still within @Transactional boundary
reward = rewardService.saveNewReward(orgId, reward, rewardCreateRequest, authDetailsProvider.get());
final Long savedRewardId = reward.getId();
final Long finalOrgId = orgId;
TransactionSynchronizationManager.registerSynchronization(
    new TransactionSynchronizationAdapter() {
        @Override
        public void afterCommit() {
            catalogueEventPublisher.publishUpsert(finalOrgId, savedRewardId);
        }
    }
);
```

---

#### Modifications to `RewardService.save()` — Line 112

After `rewardJdbcRepository.update(reward)` (line 114):

```java
// Inside RewardService.save() — @Transactional
final Long rewardId = reward.getId();
final Long orgId = reward.getOrgId();
TransactionSynchronizationManager.registerSynchronization(
    new TransactionSynchronizationAdapter() {
        @Override
        public void afterCommit() {
            catalogueEventPublisher.publishUpsert(orgId, rewardId);
        }
    }
);
```

---

#### Modifications to `RewardFacade` — `updateStatus()` path (line ~1651)

```java
// Inside the toggle-status block — after updateCount > 0
if (updateCount > 0) {
    reward.setEnabled(status.getStatus());
    eventsService.pushRewardUpdatedEvent(reward);
    catalogueEventPublisher.publishUpsert(orgId, id);  // direct publish — JDBC already committed
    statusDto = new StatusDto(true, REWARD_STATUS_UPDATED.code(), REWARD_STATUS_UPDATED.msg());
}
```

---

#### `RewardCatalogueSyncListener` — `queue/RewardCatalogueSyncListener.java`

```java
@Service
@Slf4j
public class RewardCatalogueSyncListener {

    @Autowired private RewardRepository rewardRepository;
    @Autowired private RewardGroupMappingMetaRepository groupMappingRepository;
    @Autowired private LabelRepository labelRepository;
    @Autowired private RewardRepository rewardDetailsRepository;  // same bean
    @Autowired private CustomFieldService customFieldService;
    @Autowired private ActiveRewardMongoRepository activeRewardRepo;
    @Autowired private RewardCatalogueMongoRepository catalogueRepo;
    @Autowired private CatalogueCache catalogueCache;

    @RabbitListener(
        queues = SpringAmqpConfig.REWARDS_CATALOGUE_SYNC_QUEUE,
        containerFactory = "batchListenerContainerFactory"
    )
    @Trace(dispatcher = true, metricName = "rewards.catalogue.sync")
    @RmqMessageTracker(name = "rewards.catalogue.sync")
    public void onBatch(List<String> messages) {
        List<RewardCatalogueEvent> events = messages.stream()
            .map(m -> new Gson().fromJson(m, RewardCatalogueEvent.class))
            .filter(e -> e != null && e.getOrgId() != null && e.getRewardId() != null)
            .collect(Collectors.toList());

        // Group by orgId — CRITICAL for tenant isolation in batch MySQL queries
        Map<Long, List<RewardCatalogueEvent>> byOrg = new HashMap<>(events.size());
        for (RewardCatalogueEvent e : events) {
            byOrg.computeIfAbsent(e.getOrgId(), k -> new ArrayList<>()).add(e);
        }

        for (Map.Entry<Long, List<RewardCatalogueEvent>> entry : byOrg.entrySet()) {
            processOrgBatch(entry.getKey(), entry.getValue());
        }
    }

    private void processOrgBatch(Long orgId, List<RewardCatalogueEvent> events) {
        try {
            MDC.put(Constants.X_CAP_API_AUTH_ORG_ID, orgId.toString());

            List<Long> upsertIds = new ArrayList<>();
            List<Long> deleteIds = new ArrayList<>();
            for (RewardCatalogueEvent e : events) {
                if ("UPSERT".equals(e.getOperation())) upsertIds.add(e.getRewardId());
                else deleteIds.add(e.getRewardId());
            }

            if (!upsertIds.isEmpty()) processUpserts(orgId, upsertIds);
            if (!deleteIds.isEmpty()) processDeletes(orgId, deleteIds);

            catalogueCache.invalidateOrg(orgId);
        } catch (Exception e) {
            log.error("Catalogue sync failed for orgId={}", orgId, e);
            throw e;  // re-throw → RMQ retries → DLQ after max retries
        } finally {
            MDC.remove(Constants.X_CAP_API_AUTH_ORG_ID);
        }
    }

    private void processUpserts(Long orgId, List<Long> rewardIds) {
        // Fetch all from MySQL (always fresh — never trust event payload)
        List<Reward> rewards = rewardRepository.findByOrgIdAndIdIn(orgId, rewardIds);
        if (rewards.isEmpty()) {
            log.warn("No rewards found in MySQL for orgId={} rewardIds={}", orgId, rewardIds);
            return;
        }

        List<Long> foundIds = rewards.stream().map(Reward::getId).collect(Collectors.toList());

        // Fetch associated data in bulk
        List<RewardGroupMapping> groupMappings = groupMappingRepository
            .findByOrgIdAndRewardIdInAndIsActiveTrue(orgId, foundIds);
        List<Label> labels = labelRepository
            .findByOrgIdAndRewardIdInAndIsEnabled(orgId, foundIds, true);
        List<RewardDetails> detailsList = rewardRepository.getRewardDetails(orgId, foundIds, Constants.DEFAULT_LANGUAGE);
        List<CustomFieldMapping> customFields = customFieldService
            .getAllActiveCustomFields(orgId, foundIds, CustomFieldMeta.CustomFieldScope.REWARD);

        // Index by rewardId for O(1) lookup during doc construction
        Map<Long, List<Long>> groupsByReward = groupMappings.stream()
            .collect(Collectors.groupingBy(RewardGroupMapping::getRewardId,
                Collectors.mapping(RewardGroupMapping::getGroupId, Collectors.toList())));
        Map<Long, List<Long>> labelsByReward = labels.stream()
            .collect(Collectors.groupingBy(Label::getRewardId,
                Collectors.mapping(Label::getLabelId, Collectors.toList())));
        Map<Long, RewardDetails> detailsByReward = detailsList.stream()
            .collect(Collectors.toMap(RewardDetails::getRewardId, d -> d, (a, b) -> a));
        // customFields grouping omitted for brevity — same pattern

        Date now = new Date();
        List<RewardCatalogueDoc> catalogueDocs = new ArrayList<>(rewards.size());
        List<ActiveRewardDoc> activeDocs = new ArrayList<>();
        List<Long> toRemoveFromActive = new ArrayList<>();

        for (Reward reward : rewards) {
            Long rid = reward.getId();
            RewardDetails details = detailsByReward.get(rid);

            RewardCatalogueDoc cat = buildCatalogueDoc(orgId, reward, details,
                groupsByReward.getOrDefault(rid, Collections.emptyList()),
                labelsByReward.getOrDefault(rid, Collections.emptyList()));
            catalogueDocs.add(cat);

            boolean isActive = reward.isEnabled()
                && reward.getStartDate() != null && reward.getStartDate().before(now)
                && reward.getEndDate() != null && reward.getEndDate().after(now);

            if (isActive) {
                activeDocs.add(buildActiveDoc(orgId, reward,
                    groupsByReward.getOrDefault(rid, Collections.emptyList()),
                    labelsByReward.getOrDefault(rid, Collections.emptyList())));
            } else {
                toRemoveFromActive.add(rid);
            }
        }

        catalogueRepo.bulkUpsert(orgId, catalogueDocs);
        if (!activeDocs.isEmpty()) activeRewardRepo.bulkUpsert(orgId, activeDocs);
        if (!toRemoveFromActive.isEmpty()) activeRewardRepo.deleteByOrgIdAndRewardIdIn(orgId, toRemoveFromActive);
    }

    private void processDeletes(Long orgId, List<Long> rewardIds) {
        catalogueRepo.deleteByOrgIdAndRewardIdIn(orgId, rewardIds);
        activeRewardRepo.deleteByOrgIdAndRewardIdIn(orgId, rewardIds);
    }
}
```

---

#### `ActiveRewardMongoRepository` — `mongoDao/ActiveRewardMongoRepository.java`

```java
@Repository
public class ActiveRewardMongoRepository {

    @Autowired
    private MongoTemplate mongoTemplate;

    private static final String COLLECTION = "active_rewards";

    public ActiveRewardQueryResult findFiltered(Long orgId, RewardCatalogueRequest request) {
        // orgId always first in criteria — non-negotiable
        Criteria criteria = Criteria.where("orgId").is(orgId)
                                    .and("isEnabled").is(true);

        if (!CollectionUtils.isEmpty(request.getGroupIds()))
            criteria = criteria.and("groupIds").in(request.getGroupIds());
        if (!CollectionUtils.isEmpty(request.getLabelIds()))
            criteria = criteria.and("labelIds").in(request.getLabelIds());

        if (request.getPointsExact() != null) {
            criteria = criteria.and("pointsValue").is(request.getPointsExact());
        } else {
            if (request.getPointsMin() != null)
                criteria = criteria.and("pointsValue").gte(request.getPointsMin());
            if (request.getPointsMax() != null)
                criteria = criteria.and("pointsValue").lte(request.getPointsMax());
        }

        // Cursor pagination: (pointsValue > V) OR (pointsValue = V AND _id > id)
        if (request.getCursor() != null) {
            CursorToken cursor = CursorToken.decode(request.getCursor());
            criteria = criteria.orOperator(
                Criteria.where("pointsValue").gt(cursor.getPointsValue()),
                Criteria.where("pointsValue").is(cursor.getPointsValue())
                        .and("_id").gt(new ObjectId(cursor.getId()))
            );
        }

        Query query = new Query(criteria)
            .with(Sort.by(Sort.Direction.ASC, "pointsValue", "_id"))
            .limit(request.getLimit() + 1)  // +1 to detect hasMore
            .maxTimeMsec(100);              // SLO guard

        // Count query — separate, also with maxTimeMS
        Query countQuery = new Query(Criteria.where("orgId").is(orgId).and("isEnabled").is(true));
        // apply same filters for count...
        countQuery.maxTimeMsec(100);
        long total = mongoTemplate.count(countQuery, COLLECTION);

        List<ActiveRewardDoc> docs = mongoTemplate.find(query, ActiveRewardDoc.class, COLLECTION);
        return new ActiveRewardQueryResult(docs, total);
    }

    public void bulkUpsert(Long orgId, List<ActiveRewardDoc> docs) {
        BulkOperations ops = mongoTemplate.bulkOps(BulkOperations.BulkMode.UNORDERED, COLLECTION);
        for (ActiveRewardDoc doc : docs) {
            Query q = new Query(Criteria.where("orgId").is(doc.getOrgId())
                                        .and("rewardId").is(doc.getRewardId()));
            Update u = new Update()
                .set("orgId", doc.getOrgId())
                .set("rewardId", doc.getRewardId())
                .set("expiresAt", doc.getExpiresAt())
                .set("isEnabled", doc.isEnabled())
                .set("groupIds", doc.getGroupIds())
                .set("labelIds", doc.getLabelIds())
                .set("pointsValue", doc.getPointsValue());
            ops.upsert(q, u);
        }
        ops.execute();
    }

    public void deleteByOrgIdAndRewardIdIn(Long orgId, List<Long> rewardIds) {
        mongoTemplate.remove(
            new Query(Criteria.where("orgId").is(orgId).and("rewardId").in(rewardIds)),
            COLLECTION
        );
    }
}
```

---

#### `RewardCatalogueMongoRepository` — `mongoDao/RewardCatalogueMongoRepository.java`

```java
@Repository
public class RewardCatalogueMongoRepository {

    @Autowired
    private MongoTemplate mongoTemplate;

    private static final String COLLECTION = "rewards_catalogue";

    public List<RewardCatalogueDoc> findByOrgIdAndRewardIdIn(Long orgId, List<Long> rewardIds) {
        return mongoTemplate.find(
            new Query(Criteria.where("orgId").is(orgId).and("rewardId").in(rewardIds))
                .maxTimeMsec(100),
            RewardCatalogueDoc.class,
            COLLECTION
        );
    }

    public void bulkUpsert(Long orgId, List<RewardCatalogueDoc> docs) {
        BulkOperations ops = mongoTemplate.bulkOps(BulkOperations.BulkMode.UNORDERED, COLLECTION);
        for (RewardCatalogueDoc doc : docs) {
            Query q = new Query(Criteria.where("orgId").is(doc.getOrgId())
                                        .and("rewardId").is(doc.getRewardId()));
            Update u = buildCatalogueUpdate(doc);
            ops.upsert(q, u);
        }
        ops.execute();
    }

    public void deleteByOrgIdAndRewardIdIn(Long orgId, List<Long> rewardIds) {
        mongoTemplate.remove(
            new Query(Criteria.where("orgId").is(orgId).and("rewardId").in(rewardIds)),
            COLLECTION
        );
    }
}
```

---

#### `CatalogueCache` — `service/CatalogueCache.java`

```java
@Service
@Slf4j
public class CatalogueCache {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    private static final String KEY_PREFIX = "catalogue:";
    private static final long TTL_SECONDS = 30L;

    public Optional<String> get(Long orgId, String filterHash, int page) {
        String key = buildKey(orgId, filterHash, page);
        try {
            String value = redisTemplate.opsForValue().get(key);
            return Optional.ofNullable(value);
        } catch (Exception e) {
            log.warn("Cache get failed for key {}: {}", key, e.getMessage());
            return Optional.empty();
        }
    }

    public void put(Long orgId, String filterHash, int page, String json) {
        String key = buildKey(orgId, filterHash, page);
        try {
            redisTemplate.opsForValue().set(key, json, Duration.ofSeconds(TTL_SECONDS));
        } catch (Exception e) {
            log.warn("Cache put failed for key {}: {}", key, e.getMessage());
        }
    }

    public void invalidateOrg(Long orgId) {
        String pattern = KEY_PREFIX + orgId + ":*";
        try {
            Set<String> keys = redisTemplate.keys(pattern);
            if (!CollectionUtils.isEmpty(keys)) {
                redisTemplate.delete(keys);
                log.info("Invalidated {} catalogue cache keys for orgId={}", keys.size(), orgId);
            }
        } catch (Exception e) {
            log.warn("Cache invalidation failed for orgId={}: {}", orgId, e.getMessage());
        }
    }

    private String buildKey(Long orgId, String filterHash, int page) {
        return KEY_PREFIX + orgId + ":" + filterHash + ":" + page;
    }
}
```

**Note on SCAN vs `keys()`:** In development, `keys()` is acceptable. For production (Redis Sentinel), `keys()` is a blocking command that can cause latency spikes on large keyspaces. Since catalogue cache keys TTL at 30s and are few per org, `keys()` is acceptable here. If the Redis keyspace grows, replace with a `SCAN`-based iteration. Document this trade-off.

---

#### `RewardCatalogueService` — `service/RewardCatalogueService.java`

```java
@Service
@Slf4j
public class RewardCatalogueService {

    @Autowired private ActiveRewardMongoRepository activeRewardRepo;
    @Autowired private RewardCatalogueMongoRepository catalogueRepo;
    @Autowired private CatalogueCache cache;

    public RewardCatalogueResponse list(Long orgId, RewardCatalogueRequest request) {
        String filterHash = request.toFilterHash();
        int page = request.getPageNumber();

        Optional<String> cached = cache.get(orgId, filterHash, page);
        if (cached.isPresent()) {
            // deserialize and return
            return deserialize(cached.get());
        }

        long startMs = System.currentTimeMillis();
        ActiveRewardQueryResult queryResult = activeRewardRepo.findFiltered(orgId, request);
        List<ActiveRewardDoc> activeDocs = queryResult.getDocs();

        boolean hasMore = activeDocs.size() > request.getLimit();
        if (hasMore) activeDocs = activeDocs.subList(0, request.getLimit());

        String nextCursor = hasMore
            ? CursorToken.encode(activeDocs.get(activeDocs.size() - 1))
            : null;

        List<Long> rewardIds = activeDocs.stream().map(ActiveRewardDoc::getRewardId).collect(Collectors.toList());
        List<RewardCatalogueDoc> catalogueDocs = rewardIds.isEmpty()
            ? Collections.emptyList()
            : catalogueRepo.findByOrgIdAndRewardIdIn(orgId, rewardIds);

        Map<Long, RewardCatalogueDoc> catalogueById = catalogueDocs.stream()
            .collect(Collectors.toMap(RewardCatalogueDoc::getRewardId, d -> d));

        // Accept-Language defaulting to "en"
        String lang = request.getLocale() != null ? request.getLocale() : Constants.DEFAULT_LANGUAGE;

        List<RewardSummaryDto> items = new ArrayList<>(activeDocs.size());
        for (ActiveRewardDoc active : activeDocs) {
            RewardCatalogueDoc cat = catalogueById.get(active.getRewardId());
            items.add(buildSummary(active, cat, lang));
        }

        long elapsed = System.currentTimeMillis() - startMs;
        log.info("Catalogue listing orgId={} filters={} dbLoadCount={} responseCount={} elapsedMs={}",
            orgId, filterHash, activeDocs.size(), items.size(), elapsed);

        RewardCatalogueResponse response = RewardCatalogueResponse.builder()
            .items(items)
            .nextCursor(nextCursor)
            .total((int) queryResult.getTotal())
            .build();

        cache.put(orgId, filterHash, page, serialize(response));
        return response;
    }
}
```

---

#### `RewardCatalogueResource` — `resource/RewardCatalogueResource.java`

```java
@Path(URLPrefix.V1_REWARDS_CATALOGUE)
@Consumes(MediaType.APPLICATION_JSON)
@Produces(MediaType.APPLICATION_JSON)
@Service
@Slf4j
public class RewardCatalogueResource {

    @Autowired private RewardCatalogueService catalogueService;
    @Autowired private Provider<AuthDetails> authDetailsProvider;
    @Autowired private MetricsService metricsService;

    @GET
    @Trace(dispatcher = true, metricName = "GET /core/v1/rewards/catalogue")
    public Response list(@Context UriInfo uriInfo, @Context HttpHeaders headers) {
        long start = System.currentTimeMillis();
        Long orgId = authDetailsProvider.get().getOrgId();
        log.info("Catalogue listing request orgId={}", orgId);
        metricsService.addCustomParameter(NewRelicConstants.ORG_ID, orgId);
        metricsService.addCustomParameter(NewRelicConstants.REQUEST_ID,
            MDC.get(Constants.REQUEST_ID_MDC));

        try {
            RewardCatalogueRequest request = RewardCatalogueRequest.fromUriInfo(uriInfo, headers);

            // Validate mutual exclusivity of pointsExact vs range
            if (request.getPointsExact() != null
                    && (request.getPointsMin() != null || request.getPointsMax() != null)) {
                return Response.status(400).entity(
                    new StatusDto(false, "INVALID_REQUEST",
                        "pointsExact and pointsMin/pointsMax are mutually exclusive")
                ).build();
            }

            RewardCatalogueResponse result = catalogueService.list(orgId, request);
            metricsService.addCustomParameter("response.count", result.getItems().size());
            long elapsed = System.currentTimeMillis() - start;
            log.info("Catalogue listing response orgId={} count={} elapsedMs={}", orgId,
                result.getItems().size(), elapsed);
            return Response.ok(result).build();

        } catch (Exception e) {
            log.error("Catalogue listing failed orgId={}", orgId, e);
            return Response.serverError().build();
        }
    }
}
```

---

#### `SpringAmqpConfig` additions

```java
// New constants
public static final String REWARDS_CATALOGUE_EXCHANGE = "rewards.catalogue.exchange";
public static final String REWARDS_CATALOGUE_SYNC_QUEUE = "rewards-catalogue-sync-queue";
public static final String REWARDS_CATALOGUE_ROUTING_KEY = "rewards.catalogue.sync";
public static final String REWARDS_CATALOGUE_DLX = "rewards.catalogue.dlx";
public static final String REWARDS_CATALOGUE_DLQ = "rewards-catalogue-sync-dlq";

// Beans
@Bean
DirectExchange rewardsCatalogueExchange() {
    return new DirectExchange(REWARDS_CATALOGUE_EXCHANGE, true, false);
}

@Bean
DirectExchange rewardsCatalogueDlx() {
    return new DirectExchange(REWARDS_CATALOGUE_DLX);
}

@Bean
Queue rewardsCatalogueSyncQueue() {
    return QueueBuilder.durable(REWARDS_CATALOGUE_SYNC_QUEUE)
        .withArgument("x-dead-letter-exchange", REWARDS_CATALOGUE_DLX)
        .withArgument("x-dead-letter-routing-key", REWARDS_CATALOGUE_ROUTING_KEY)
        .build();
}

@Bean
Queue rewardsCatalogueDlq() {
    return QueueBuilder.durable(REWARDS_CATALOGUE_DLQ).build();
}

@Bean
Binding rewardsCatalogueSyncBinding() {
    return BindingBuilder.bind(rewardsCatalogueSyncQueue())
        .to(rewardsCatalogueExchange())
        .with(REWARDS_CATALOGUE_ROUTING_KEY);
}

@Bean
Binding rewardsCatalogueDlqBinding() {
    return BindingBuilder.bind(rewardsCatalogueDlq())
        .to(rewardsCatalogueDlx())
        .with(REWARDS_CATALOGUE_ROUTING_KEY);
}

@Bean("batchListenerContainerFactory")
SimpleRabbitListenerContainerFactory batchListenerContainerFactory(
        ConnectionFactory connectionFactory) {
    SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
    factory.setConnectionFactory(connectionFactory);
    factory.setMessageConverter(converter());
    factory.setBatchListener(true);
    factory.setConsumerBatchEnabled(true);
    factory.setBatchSize(200);
    return factory;
}
```

---

#### `CursorToken` — `dto/CursorToken.java`

```java
public class CursorToken {
    private Integer pointsValue;
    private String id;  // ObjectId hex string

    public static String encode(ActiveRewardDoc last) {
        String json = new Gson().toJson(
            Map.of("pointsValue", last.getPointsValue(), "id", last.getId().toHexString())
        );
        return Base64.getUrlEncoder().withoutPadding().encodeToString(json.getBytes(StandardCharsets.UTF_8));
    }

    public static CursorToken decode(String cursor) {
        byte[] bytes = Base64.getUrlDecoder().decode(cursor);
        return new Gson().fromJson(new String(bytes, StandardCharsets.UTF_8), CursorToken.class);
    }
}
```

---

## 9. Data Model Changes

### MongoDB Collections

**`rewards_catalogue`** — New collection, no MySQL schema change.
Compound indexes to `ensureIndex` on startup:
1. `{ orgId: 1, rewardId: 1 }` — unique
2. `{ orgId: 1, isEnabled: 1, startDate: 1, endDate: 1 }` — history/admin
3. `{ orgId: 1, updatedAt: 1, _id: 1 }` — cursor pagination

**`active_rewards`** — New collection, no MySQL schema change.
Compound indexes to `ensureIndex` on startup:
1. `{ expiresAt: 1 }` — TTL (`expireAfterSeconds: 0`)
2. `{ orgId: 1, rewardId: 1 }` — unique
3. `{ orgId: 1, groupIds: 1, pointsValue: 1 }` — group-filtered listing
4. `{ orgId: 1, labelIds: 1, pointsValue: 1 }` — label-filtered listing
5. `{ orgId: 1, pointsValue: 1 }` — points-only listing
6. `{ orgId: 1, isEnabled: 1 }` — enabled filter

All indexes created by `CatalogueIndexInitializer.ensureIndexes()` on startup (idempotent — `ensureIndex` is safe to re-run).

### MySQL Schema Changes

**None.** No new tables, columns, or indexes.

### Redis TTL Bucket

Add `THIRTY_SECOND_CACHE` to `RedisCacheKeys` + `RedisCacheUtil.getRedisCacheConfig()`.
Note: `CatalogueCache` uses manual `RedisTemplate` operations — `THIRTY_SECOND_CACHE` bucket is only needed if any `@Cacheable` annotations are added later. The 30s TTL is hardcoded in `CatalogueCache.TTL_SECONDS`.

---

## 10. API Changes

### New Endpoint

**`GET /core/v1/rewards/catalogue`**

Request parameters:
| Param | Type | Required | Default | Notes |
|-------|------|----------|---------|-------|
| `groupIds` | Long[] | No | — | Comma-separated or repeated param |
| `labelIds` | Long[] | No | — | Comma-separated or repeated param |
| `pointsMin` | Integer | No | — | Mutually exclusive with `pointsExact` |
| `pointsMax` | Integer | No | — | Mutually exclusive with `pointsExact` |
| `pointsExact` | Integer | No | — | Mutually exclusive with range |
| `isEnabled` | Boolean | No | `true` | Default enabled only |
| `cursor` | String | No | — | Opaque base64 cursor |
| `limit` | Integer | No | 50 | Max 50 |

Response:
```json
{
  "items": [ RewardSummaryDto ],
  "nextCursor": "base64token | null",
  "total": 178
}
```

`RewardSummaryDto`:
```json
{
  "rewardId": 12345,
  "name": "Gold Reward",
  "imageUrl": "https://...",
  "pointsValue": 500,
  "groupIds": [1, 2],
  "labelIds": [10, 11],
  "startDate": "2026-05-01T00:00:00Z",
  "endDate": "2026-05-31T23:59:59Z",
  "isEnabled": true
}
```

Dates formatted via `DateTimeService.getFormattedDateInGMT()` per MADR-0002.

**Breaking change:** No — new endpoint only.

---

## 11. Security

| Check | Result | Notes |
|-------|--------|-------|
| New API endpoint requiring auth/role checks | PASS | Auth is handled by existing `RequestFilterV1` / Jersey filters; orgId extracted from `AuthDetails` (not from query param) |
| Query param `orgId` not trusted | PASS | `orgId` always from `authDetailsProvider.get().getOrgId()` — client-supplied `orgId` is ignored |
| PII in new log lines | PASS | No customer data in catalogue listing; only `orgId`, `rewardId`, counts logged |
| New fields storing PII | N/A | `rewards_catalogue` stores reward metadata only — no customer PII |
| Token / credential handling | N/A | No new credentials |
| Tenant data leak risk | PASS | `orgId` is first filter in every MongoDB query; Redis key prefixed with `orgId`; RMQ event carries `orgId` scoped to batch |
| User-supplied input in MongoDB queries | PASS | `groupIds`, `labelIds`, `pointsExact`, `pointsMin`, `pointsMax` are typed integers — no string injection risk; cursor is base64-decoded JSON with typed fields |
| Cursor tampering | NOTE | A tampered cursor (wrong `_id`) returns results from an arbitrary page offset — not a security issue (no data leak across orgs, since `orgId` filter is always applied). No HMAC needed. |
| Cross-tenant isolation in batch listener | PASS | Events grouped by `orgId` before MySQL queries; no shared mutable state in listener |

---

## 12. DB & Infra Considerations

**CONSIDERATION [INFRA] [ACTION]**
Finding: `CatalogueCache.invalidateOrg()` uses `RedisTemplate.keys(pattern)` — blocking command on Redis Sentinel.
Impact if ignored: Latency spike on Redis primary during peak invalidation (e.g., 2000 reward bulk create → 2000 invalidation calls, though batched by orgId).
Recommended action: Accept for initial release (keys per org are few, 30s TTL self-heals). Add SCAN-based iteration as a follow-up if Redis monitoring shows `keys` command latency.
Owner: tech-detailer

**CONSIDERATION [INFRA] [ACTION]**
Finding: `CatalogueIndexInitializer` annotated `@Profile("production")`. Test profile will NOT run index creation, so integration tests need either a separate test initializer or mock Mongo.
Impact if ignored: Test suite passes without indexes, masking performance regressions.
Recommended action: Create a `@Profile("test")` variant of the initializer or use an embedded Mongo with the same index setup in integration tests.
Owner: tech-detailer

**CONSIDERATION [DB] [NOTE]**
Finding: `rewards_catalogue` collection grows unboundedly (no TTL, no archival job).
Impact if ignored: Collection size grows 2000 docs/org/month = 24,000 after 12 months for enterprise orgs. At 100 orgs this is 2.4M docs, which is manageable for MongoDB but should be monitored.
Recommended action: Add a `rewards_catalogue` TTL index on `syncedAt` or `endDate` (e.g., 18 months) as a follow-up. Not blocking for initial release.
Owner: Deferred

**CONSIDERATION [INFRA] [ACTION]**
Finding: Batch listener `setBatchSize(200)` — with 2000 rewards per org per month-start burst, this means ~10 batches per org. Each batch does 1 MySQL IN query + 1 MongoDB bulkWrite per org sub-group. This is within normal RabbitMQ consumer throughput.
Impact if ignored: None — sizing is adequate.
Recommended action: Monitor RMQ queue depth during first month-start after deploy.
Owner: Observability

**CONSIDERATION [DB] [ACTION]**
Finding: `bulkUpsert` on `active_rewards` uses `BulkOperations.BulkMode.UNORDERED` — failures on individual operations do not abort the batch.
Impact if ignored: A single malformed doc does not block the rest of the batch. Individual failures logged at WARN.
Recommended action: Log failed operations count from `BulkWriteResult`. Already good practice.
Owner: tech-detailer

---

## 13. Internal Architecture Changes

### New Packages / Classes
- `db/mongo/RewardCatalogueDoc` — new MongoDB doc
- `db/mongo/ActiveRewardDoc` — new MongoDB doc
- `queue/RewardCatalogueEvent` — RMQ message POJO
- `queue/RewardCatalogueSyncListener` — batch RMQ consumer
- `queue/RewardCatalogueEventPublisher` — afterCommit + direct publisher
- `mongoDao/ActiveRewardMongoRepository` — MongoTemplate DAO
- `mongoDao/RewardCatalogueMongoRepository` — MongoTemplate DAO
- `service/RewardCatalogueService` — listing orchestration
- `service/CatalogueCache` — Redis read/write/invalidate
- `resource/RewardCatalogueResource` — JAX-RS endpoint
- `config/mongo/CatalogueIndexInitializer` — @PostConstruct index setup
- `dto/RewardCatalogueRequest` — query param DTO
- `dto/RewardCatalogueResponse` — response DTO
- `dto/RewardSummaryDto` — per-item DTO
- `dto/CursorToken` — cursor encode/decode

### Modified Classes
- `service/impl/RewardFacade` — afterCommit registration in `create()`, direct publish in `updateStatus()` path
- `service/impl/RewardService` — afterCommit registration in `save()`
- `queue/SpringAmqpConfig` — new exchange/queue/DLQ/binding + batchListenerContainerFactory beans
- `utils/RedisCacheKeys` — new `THIRTY_SECOND_CACHE` constant (may not be needed if CatalogueCache uses hardcoded TTL)
- `utils/RedisCacheUtil` — new 30s TTL config entry
- `resource/URLPrefix` — `V1_REWARDS_CATALOGUE` constant

### New Patterns Introduced
- `TransactionSynchronizationAdapter.afterCommit()` — first use in this codebase
- `BulkOperations` for MongoDB — first use in this codebase
- Manual `RedisTemplate` key SCAN-based cache invalidation

---

## 14. Upstream / Downstream Impact

### Upstream
- **Reward write clients** (brand admin API, bulk upload): No change to contract. New behaviour: RMQ message is published after each successful reward write. No new API fields.

### Downstream
- **MongoDB `sol-rewards-core` database**: Two new collections added. No impact on existing collections.
- **RabbitMQ `intouch` server**: New exchange `rewards.catalogue.exchange` + two queues. Must be pre-declared before first deploy — Spring AMQP auto-declares on startup.
- **Redis `incentives-redis-sentinel`**: New cache keys with `catalogue:` prefix. No impact on existing keys.

### Coordination Required
- **RabbitMQ pre-declaration**: Spring AMQP will auto-declare exchange/queue on application startup. No manual ops step needed.
- **MongoDB index creation**: `CatalogueIndexInitializer.ensureIndexes()` runs on startup — idempotent. No manual MongoDB script needed.

---

## 15. SLA Impact

**GET /core/v1/rewards/catalogue:**
- Cache HIT: < 10ms (Redis GET)
- Cache MISS: 30–45ms expected (5–15ms `active_rewards` scan + 5–10ms `rewards_catalogue` $in + 10–20ms merge + network)
- `maxTimeMS(100)` applied to all MongoDB queries — request fails fast if MongoDB is slow
- **p95 target < 150ms** — comfortable with 45ms expected + 105ms headroom

**Reward write path (create/update):**
- `afterCommit()` hook adds ~0ms to user-facing latency (fires after MySQL commit, not in the HTTP response path)
- `updateStatus()` path: `catalogueEventPublisher.publishUpsert()` is called inline — adds ~1ms for RabbitMQ publish. Catch block ensures no impact on failure.

**RMQ listener (sync latency):**
- Single reward sync: ~20–50ms (MySQL fetch + MongoDB upsert)
- Batch of 200 rewards: ~100–500ms (batch MySQL IN + MongoDB bulkWrite)
- Month-start burst (2000 rewards, 10 batches): ~1–5 seconds total sync time per org

---

## 16. Observability

### New Log Lines
| Location | Level | Content |
|----------|-------|---------|
| `RewardCatalogueResource.list()` entry | INFO | `orgId`, request params |
| `RewardCatalogueResource.list()` exit | INFO | `response.count`, `elapsedMs` |
| `RewardCatalogueService.list()` | INFO | `db.loadCount`, `response.count`, filter hash, elapsed |
| `RewardCatalogueEventPublisher.publish()` | INFO | `orgId`, `rewardId`, `operation` |
| `RewardCatalogueEventPublisher.publish()` failure | ERROR | exception details |
| `RewardCatalogueSyncListener.processOrgBatch()` failure | ERROR | `orgId`, exception |
| `RewardCatalogueSyncListener` reward not found in MySQL | WARN | `orgId`, `rewardIds` |
| `CatalogueCache.invalidateOrg()` | INFO | `orgId`, keys invalidated count |
| `CatalogueCache` get/put failures | WARN | key, exception message |

### New Relic Attributes (mandatory)
- `ORG_ID` (all transactions)
- `REQUEST_ID` (all transactions)
- `db.loadCount` — `active_rewards` docs fetched
- `response.count` — items in response
- `catalogue.cacheHit` — `true` / `false` per request

### Alerts
- RMQ DLQ `rewards-catalogue-sync-dlq` depth > 0: add to existing RMQ monitoring.
- MongoDB `maxTimeMS` exceeded: surfaced via `MongoExecutionTimeoutException` in application logs → New Relic error rate alert.

---

## 17. Rollout Plan

1. **No feature flag needed** — new endpoint is additive; existing endpoints unchanged.
2. **Deploy order:**
   - Deploy application: Spring AMQP auto-declares RMQ exchange + queues on startup.
   - `CatalogueIndexInitializer` creates MongoDB indexes on startup (idempotent).
   - No MongoDB schema migration script needed.
3. **Initial MongoDB population:**
   - After first deploy, `active_rewards` and `rewards_catalogue` are empty.
   - All existing rewards will be synced progressively as they are mutated.
   - Optionally, run a one-time back-fill job (not in scope) or accept eventual fill-up over days.
   - For the load test environment, manually trigger UPSERT events for all active rewards.
4. **Go/no-go criteria:**
   - Existing reward create/update APIs working (smoke test)
   - `GET /core/v1/rewards/catalogue` returns data for at least one org with active rewards
   - MongoDB indexes confirmed via `db.active_rewards.getIndexes()`
   - RMQ `rewards-catalogue-sync-queue` depth declining after burst
5. **Rollback:**
   - Remove `CatalogueIndexInitializer` bean (or `@Profile` guard it)
   - Remove `RewardCatalogueResource` from Jersey config
   - The afterCommit + direct publish additions are benign even if catalogue is disabled — RMQ messages accumulate in queue but don't affect reward write API

---

## 18. Risks

| # | Description | Likelihood | Mitigation |
|---|------------|-----------|-----------|
| 1 | `active_rewards` empty on first deploy — catalogue returns 0 results until rewards are mutated | HIGH (expected) | Back-fill script or pre-load via admin tool; document for load test |
| 2 | `keys()` Redis command causes latency spike during bulk invalidation | LOW | 30s TTL self-heals; keys per org bounded; replace with SCAN if monitoring shows impact |
| 3 | Batch listener re-uses orgId grouping incorrectly (bug in grouping logic) | MEDIUM | Unit test with mixed-org batch input |
| 4 | afterCommit not firing — Spring version or proxy issue | LOW | Unit test that verifies event published after successful transaction |
| 5 | MongoDB TTL thread lag (up to 60s) causes expired rewards to appear briefly | LOW | Document as known behaviour; not a correctness issue |
| 6 | `BulkOperations` partial failure (malformed doc) — silent drop | MEDIUM | Log `BulkWriteResult.getModifiedCount()` + exceptions; alert on error rate |
| 7 | `maxTimeMS(100)` too aggressive for MongoDB under heavy write load | LOW | Set as config property `catalogue.mongo.maxTimeMs` for tuning; 100ms is conservative for index scans on 2000 docs |
| 8 | New 30s Redis TTL bucket collides with another cache using that TTL | LOW | Only `CatalogueCache` uses the bucket; manual `RedisTemplate` with explicit duration avoids collision |

---

## 19. Open Questions

| Question | Owner | Due |
|---------|-------|-----|
| Back-fill strategy for existing active rewards on first deploy | Saravanan | Before load test |
| Redis `keys()` vs SCAN acceptable for current Redis Sentinel keyspace size | Infra | Before production deploy |
| Should `rewards_catalogue` have a TTL/archival job for historical rewards? | Saravanan | Deferred — not blocking |
| `pointsValue` = `intouchPoints` (Integer) — confirm no decimal rewards exist in load test org | Saravanan | Before load test |

---

## 20. Questions for arch-investigator

**[BLOCKER — resolved in this doc]**
Q1: `updateStatus()` in `RewardFacade` (line 798) is NOT `@Transactional`. The handoff doc only covered `create()` and `save()` for afterCommit.
Resolution: Publish directly from `updateStatus()` block after confirmed JDBC success (not via afterCommit). Added to low-level design §8.

**[CLARIFICATION]**
Q2: `groupIds` and `labelIds` in MongoDB — stored as `Long` (matching MySQL DB types) or `String` (matching API contract)? Resolved as Long in storage, serialised to Long in API response. If clients expect String, add `@JsonSerialize` or use String in DTO only.

---

## 21. Task Breakdown

### Backend
| Task | Size | Deps |
|------|------|------|
| `RewardCatalogueDoc` + `ActiveRewardDoc` MongoDB document classes | S | — |
| `RewardCatalogueEvent` POJO | S | — |
| `CatalogueIndexInitializer` — @PostConstruct ensureIndex for all 7+3 indexes | S | Doc classes |
| `SpringAmqpConfig` additions — exchange/queue/DLQ/binding/batchListenerFactory | S | — |
| `RewardCatalogueEventPublisher` — afterCommit + direct publish | S | SpringAmqpConfig |
| Modify `RewardFacade.create()` — register afterCommit | S | Publisher |
| Modify `RewardService.save()` — register afterCommit | S | Publisher |
| Modify `RewardFacade.updateStatus()` path — direct publish | S | Publisher |
| `ActiveRewardMongoRepository` — MongoTemplate queries (find, bulkUpsert, delete) | M | Doc classes |
| `RewardCatalogueMongoRepository` — MongoTemplate queries (findByIds, bulkUpsert, delete) | M | Doc classes |
| `RewardCatalogueSyncListener` — batch consumer with orgId grouping | M | All repos, Publisher |
| `CatalogueCache` — Redis get/put/invalidate | S | RedisCacheKeys update |
| `RedisCacheKeys` + `RedisCacheUtil` — add THIRTY_SECOND_CACHE | S | — |
| `RewardCatalogueRequest` DTO + `fromUriInfo()` parsing | S | — |
| `RewardCatalogueResponse` + `RewardSummaryDto` DTOs | S | — |
| `CursorToken` encode/decode | S | — |
| `RewardCatalogueService` — orchestration | M | All repos, cache, DTOs |
| `URLPrefix` — add `V1_REWARDS_CATALOGUE` | S | — |
| `RewardCatalogueResource` — JAX-RS endpoint | S | Service, DTOs |
| Register `RewardCatalogueResource` in Jersey config | S | Resource |

### Testing
| Task | Size | Deps |
|------|------|------|
| Unit: `RewardCatalogueEventPublisher` — afterCommit fires on success, not on rollback | S | Publisher |
| Unit: `RewardCatalogueSyncListener` — UPSERT + DELETE flows, mixed-org batch grouping, reward-not-found WARN | M | Listener |
| Unit: `ActiveRewardMongoRepository` — orgId first in all queries, maxTimeMS applied, cursor predicate correct | M | Repo |
| Unit: `RewardCatalogueService` — cache HIT returns early, cache MISS does full query, empty org returns empty response | M | Service |
| Unit: `RewardCatalogueResource` — 400 on pointsExact+range conflict, 200 on valid request | S | Resource |
| Unit: `CatalogueCache` — invalidation scoped to orgId, does NOT touch other org keys | S | Cache |
| Unit: filter combinations — isEnabled only, groupId, label, points range, all together, no filters | M | Service |
| Unit: `CursorToken` — encode/decode round-trip | S | CursorToken |
| Unit: listener idempotency — same UPSERT twice results in single doc (upsert semantics) | S | Listener |

### Infra / DB
| Task | Size | Deps |
|------|------|------|
| Verify MongoDB indexes created on startup (manual check post-deploy in test env) | S | Initializer |
| Verify RMQ exchange+queue declared on startup | S | SpringAmqpConfig |

### Observability
| Task | Size | Deps |
|------|------|------|
| Add New Relic attributes in `RewardCatalogueResource` (orgId, requestId, response.count, db.loadCount) | S | Resource |
| Add DLQ monitoring note to service wiki | S | — |

---

## Handoff Notes for test-plan-architect

**Tech detail doc location:** `arch-investigation.md` + `tech-detail.md`

### Critical B2B Flows to Test
- UC-1: Month-start 2000 reward batch → batch listener syncs all to `active_rewards` → test type: unit (listener batch grouping) + integration
- UC-3: Reward disable toggle → removed from `active_rewards` → test type: unit
- UC-5: Concurrent updates → idempotent upsert resolves correctly → test type: unit
- UC-9: RMQ listener failure → DLQ parking → manual replay → test type: unit

### Critical B2C Flows to Test
- UC-12: groupIds filter → correct `$in` on `active_rewards.groupIds` → unit
- UC-13: Cursor pagination → correct `$or` predicate on (pointsValue, _id) → unit
- UC-14/15: Cache HIT/MISS paths → unit
- UC-16: Empty org → `{ items: [], total: 0, nextCursor: null }` → unit

### Regression Risks (must not break)
- Reward create API — afterCommit registration must not change behaviour on MySQL rollback (no event published)
- Reward update API — same
- Reward toggle API — direct publish must not affect HTTP response if RMQ is down

### Tenant Isolation Tests
- `active_rewards` query with `orgId=A` must return 0 docs when all active rewards belong to `orgId=B`
- Batch listener with events from orgId=A and orgId=B must group separately and not cross-contaminate MySQL queries
- Redis cache key for orgId=A must not be invalidated when orgId=B reward is mutated

### Contract Tests
- `RewardRepository.findByOrgIdAndIdIn(orgId, rewardIds)` returns `List<Reward>` with `labels` lazy-loaded — listener must explicitly call `LabelRepository` for labels (NOT rely on lazy-loaded `Reward.labels`)
- `LabelRepository.findByOrgIdAndRewardIdInAndIsEnabled(orgId, ids, true)` — verify returns only enabled labels

### Suggested Test Emphasis
- **Unit:** filter combination matrix (6 filter dimensions × enabled/disabled combinations)
- **Unit:** afterCommit — verify TransactionSynchronizationManager is called inside @Transactional boundary, not called on exception
- **Integration:** batch listener processes mixed-org batch correctly
- **Tenant isolation:** orgId boundary on every MongoDB query and every Redis key operation
