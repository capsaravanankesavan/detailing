# Pre-Analysis: Reward Catalogue Listing with Filters
**Document Type:** Arch Investigation Input / Agent Orchestration Seed  
**Target Skill Pipeline:** `/arch-investigator` → `/tech-detailer` → `/tech-detail-reviewer` → `/test-plan-architect` → `/test-case-sheet` → `/tdd-developer`  
**Service:** `sol-rewards-core`  
**Status:** All design decisions finalised. Agent to proceed through skill pipeline without manual review gates.

---

## 1. Problem Statement

The rewards service (`sol-rewards-core`) needs a **Reward Catalogue Listing API** that allows consumers to query active rewards with multiple filter dimensions. The system is multi-tenant (100+ orgs), with enterprise orgs running up to **2000 concurrently active rewards** at any point in time. Rewards are **short-lived and high-churn** — a typical enterprise org creates a fresh batch of ~2000 rewards every month, each active for ~30 days, then replaced by a new batch.

The current MySQL-normalised schema (rewards + reward_groups + reward_labels + reward_lang + reward_custom_fields) is the source of truth but is **not suitable as the read model** for this listing API due to multi-join cost and lack of native array-filter support.

The service currently publishes Kafka events on reward create/update for notification consumers, but **does not consume Kafka**. MongoDB is already used in the service for idempotency. RabbitMQ is already present in the service.

---

## 2. Business Requirements

| Requirement | Detail |
|---|---|
| Filter: date window | Return rewards where `startDate <= now AND endDate >= now` |
| Filter: isEnabled | Boolean; only enabled rewards in default listing |
| Filter: groupId | Multi-value (1-to-many); match any of provided groupIds |
| Filter: label | Multi-value (1-to-many); match any of provided labels |
| Filter: pointsValue | Exact match OR range (`gte` / `lte`) |
| Pagination | 50 items per page; cursor-based |
| SLO | p95 response < 150ms |
| Tenancy | Per-org scoped; all queries must be bounded by `orgId` |
| Scale — standard orgs | 100 orgs, ~100 active rewards each |
| Scale — enterprise orgs | Up to 2000 concurrently active rewards per org |
| Churn pattern | ~2000 new rewards created per org per month; prior batch expires at month end |

---

## 3. Architectural Decisions (Finalised — Do Not Re-evaluate)

### 3.1 Dual Data Store: MySQL (write) + MongoDB (read)

**Decision:** MySQL remains the source of truth for all reward writes. A denormalised MongoDB read model (`rewards_catalogue` collection) is built and maintained as a CQRS read side.

**Rationale:**
- MySQL normalised schema requires 4–5 JOINs + GROUP_CONCAT for a single listing query
- Multi-value array filters (`groupIds IN [...]`, `labels IN [...]`) are not natively index-friendly in MySQL
- MongoDB multikey indexes and document model directly support array containment queries
- Service already uses MongoDB for idempotency — no new infrastructure dependency

---

### 3.2 RMQ Async Sync (MySQL → MongoDB)

**Decision:** MongoDB read model is kept in sync via a RabbitMQ listener. The reward write path publishes a lightweight event to a new RMQ exchange after MySQL commit. The listener fetches fresh state from MySQL and upserts into MongoDB.

**Rationale:**
- `sol-rewards-core` does not currently consume Kafka — adding Kafka consumer infra is out of scope
- RMQ is already present in the service — new exchange + queue only, no new infrastructure
- Decouples MongoDB write latency from the reward write API response time
- Natural buffer for month-start burst (2000 rewards created simultaneously)
- Eventual consistency (seconds lag) is acceptable for a catalogue listing use case
- Future migration path: if Kafka reward events are enriched with `orgId` + `operation`, retire RMQ queue and replace with Kafka consumer group

**Exchange / Queue setup:**
```
Exchange:  rewards.catalogue.exchange   (topic)
Queue:     rewards-catalogue-sync-queue (durable)
DLQ:       rewards-catalogue-sync-dlq   (durable)
Routing:   rewards.catalogue.sync
```

**Message payload — identity only, no reward data:**
```json
{
  "orgId":       "ORG-42",
  "rewardId":    "RWD-2001",
  "operation":   "UPSERT | DELETE",
  "triggeredAt": "2026-05-13T14:00:00Z"
}
```

**Why no payload?** Listener always fetches fresh state from MySQL. Avoids stale snapshot problem when messages are delayed or retried out of order.

**Publish point — afterCommit() only:**
```java
@Transactional
public void updateReward(UpdateRewardRequest request) {
    // MySQL writes inside transaction
    rewardRepository.save(reward);
    rewardGroupRepository.saveAll(groups);

    // Publish only after MySQL commit succeeds
    TransactionSynchronizationManager.registerSynchronization(
        new TransactionSynchronizationAdapter() {
            @Override
            public void afterCommit() {
                catalogueEventPublisher.publish(
                    new RewardCatalogueEvent(orgId, rewardId, UPSERT)
                );
            }
        }
    );
}
```

**Critical constraint:** Never publish inside the transaction body. If MySQL rolls back, message must not reach the listener.

**Listener behaviour:**
```
On UPSERT:
  1. Fetch full reward from MySQL (rewards + groups + labels + lang + customFields)
  2. If not found (deleted between publish and consume) — log warn, ack, return
  3. Upsert into rewards_catalogue
  4. If isEnabled=true AND startDate <= now AND endDate >= now:
       Upsert into active_rewards (expiresAt = endDate)
     Else:
       Delete from active_rewards if present
  5. Invalidate Redis cache: catalogue:{orgId}:*

On DELETE:
  1. Delete from rewards_catalogue by { orgId, rewardId }
  2. Delete from active_rewards by { orgId, rewardId }
  3. Invalidate Redis cache: catalogue:{orgId}:*

On failure:
  Throw exception → RMQ retries → parks in DLQ after max retries
  Alert on DLQ depth > 0
  Manual replay supported — upsert semantics make replay safe
```

**Batch handling (month-start burst — 2000 rewards):**
```java
// Batch consumer — one MySQL IN query + one MongoDB bulkWrite
@RabbitListener(
    queues = "rewards-catalogue-sync-queue",
    containerFactory = "batchListenerFactory"
)
public void onBatch(List<RewardCatalogueEvent> events) {
    List<String> rewardIds = events.stream()
        .map(RewardCatalogueEvent::getRewardId).toList();

    List<RewardDetail> details = rewardReadService.fetchBatch(orgId, rewardIds);
    rewardsCatalogueRepo.bulkUpsert(details);
    activeRewardsRepo.bulkUpsertOrRemove(details);
    catalogueCache.invalidateOrg(orgId);
}
```

---

### 3.3 TTL Collection for Active Set (`active_rewards`)

**Decision:** Maintain a separate thin MongoDB collection `active_rewards` with a TTL index on `expiresAt = endDate`. This is the **primary target for listing queries** — not `rewards_catalogue`.

**Rationale:**
- Without TTL collection, `rewards_catalogue` accumulates expired rewards: 2000/month = 24,000 docs after 12 months per org
- Date range filter (`startDate <= now AND endDate >= now`) degrades as expired:active ratio grows (1:1 at month 1, 11:1 at month 12)
- `active_rewards` collection size stays constant at ~2000 per org regardless of how many months pass
- All indexes on `active_rewards` permanently fit in WiredTiger cache — query time stays flat forever
- TTL thread auto-purges expired rewards within 60s of `endDate` — no manual archival job
- Start boundary: application inserts on activation; up to 60s lag acceptable for catalogue

**TTL index:**
```js
db.active_rewards.createIndex(
  { expiresAt: 1 },
  { expireAfterSeconds: 0 }
)
```

---

### 3.4 Filter Dimensions Denormalised into `active_rewards`

**Decision:** `active_rewards` stores filter-dimension fields (`groupIds`, `labels`, `pointsValue`, `isEnabled`) alongside identity fields. Listing queries hit `active_rewards` only for filtering and pagination. Display fields (`name`, `imageUrl`, `langDetails`) are fetched from `rewards_catalogue` by `rewardId` for the final result set (post-filter, typically 50 docs).

**Rationale:** Avoids two-hop query (fetch active IDs → $in on rewards_catalogue). Single collection scan on ~2000 docs per org is sufficient and fast.

---

### 3.5 `orgId` as First Field in Every Index

**Decision:** `orgId` must be the leading field in every compound index on both collections without exception.

**Rationale:** Without `orgId` first, a query for one org scans all orgs' documents. At 100 orgs × 2000 active rewards = 200,000 docs, missing `orgId` prefix is a full collection scan.

---

### 3.6 Redis Cache Layer

**Decision:** Cache listing query results in Redis keyed by `catalogue:{orgId}:{hash(filterParams)}:{page}`.

**TTL:** 30s flat. Invalidated on any reward mutation event for the org (triggered by RMQ listener after successful MongoDB write).

**Rationale:** Monthly churn means filter combinations within an org are bounded and repeat frequently within the 30s window. Cache hit rate will be high. Cached path targets < 10ms.

---

## 4. Data Models

### 4.1 `rewards_catalogue` (Full Read Model)

```js
{
  _id:          ObjectId,
  orgId:        String,           // tenant key
  rewardId:     String,           // business ID, unique within org
  isEnabled:    Boolean,
  startDate:    ISODate,
  endDate:      ISODate,
  groupIds:     [String],
  labels:       [String],
  pointsValue:  Number,           // scaled integer (x100 if decimals needed)
  langDetails:  { en: String, fr: String },
  imageUrl:     String,
  customFields: Object,
  syncedAt:     ISODate,          // last synced from MySQL
  updatedAt:    ISODate
}

// Indexes
{ orgId: 1, rewardId: 1 }                              // unique — upsert key
{ orgId: 1, isEnabled: 1, startDate: 1, endDate: 1 }  // history / admin queries
{ orgId: 1, updatedAt: 1, _id: 1 }                    // cursor pagination
```

### 4.2 `active_rewards` (TTL Collection — Primary Listing Query Target)

```js
{
  _id:          ObjectId,
  orgId:        String,
  rewardId:     String,
  expiresAt:    ISODate,          // TTL field = endDate

  // Filter dimensions (denormalised from rewards_catalogue)
  isEnabled:    Boolean,
  groupIds:     [String],         // multikey indexed
  labels:       [String],         // multikey indexed
  pointsValue:  Number
}

// Indexes
{ expiresAt: 1 }                               // TTL — expireAfterSeconds: 0
{ orgId: 1, rewardId: 1 }                      // unique — upsert key
{ orgId: 1, groupIds: 1, pointsValue: 1 }      // group-filtered listing
{ orgId: 1, labels: 1,   pointsValue: 1 }      // label-filtered listing
{ orgId: 1, pointsValue: 1 }                   // points-only listing
{ orgId: 1, isEnabled: 1 }                     // enabled filter
```

---

## 5. Query Flow

```
Request
  │
  ▼
Redis Cache (key: catalogue:{orgId}:{filterHash}:{page})
  │
  ├── HIT  → return 50 items                                [< 10ms]
  │
  └── MISS
        │
        ▼
        active_rewards.find({
          orgId:        <orgId>,
          isEnabled:    true,
          groupIds:     { $in: [...] },        // if provided
          labels:       { $in: [...] },        // if provided
          pointsValue:  { $gte: x, $lte: y }  // if provided
        })
        .maxTimeMS(100)
        .sort({ pointsValue: 1, _id: 1 })
        .limit(50)
              │
              ▼
        rewards_catalogue.find({
          orgId:    <orgId>,
          rewardId: { $in: resultRewardIds }   // fetch display fields only
        })
        .projection({ name, imageUrl, langDetails, ... })
              │
              ▼
        merge + build response
              │
              ▼
        write to Redis cache (TTL 30s)
              │
              ▼
        return to caller
```

**Expected latency (uncached):**

| Step | Expected |
|---|---|
| `active_rewards` index scan (~2000 docs) | 5–15ms |
| `rewards_catalogue` $in fetch (50 docs) | 5–10ms |
| Serialisation + network | 10–20ms |
| **Total** | **~30–45ms** — comfortable against 150ms SLO |

---

## 6. Reward Lifecycle Operations

### Create / Update
```
1. Write to MySQL (source of truth) inside @Transactional
2. afterCommit() → publish to rewards.catalogue.exchange
     { orgId, rewardId, operation: UPSERT, triggeredAt }
3. RMQ listener:
     a. Fetch full reward from MySQL
     b. Upsert rewards_catalogue
     c. If currently active → upsert active_rewards (expiresAt = endDate)
        Else                → delete from active_rewards
     d. Invalidate Redis: catalogue:{orgId}:*
```

### Disable
```
1. Update MySQL isEnabled = false
2. afterCommit() → publish { operation: UPSERT }
3. RMQ listener:
     a. Fetch reward — sees isEnabled=false
     b. Update rewards_catalogue isEnabled=false
     c. Delete from active_rewards
     d. Invalidate Redis
```

### Expiry (endDate arrives)
```
MongoDB TTL thread auto-deletes from active_rewards within 60s
No application action required
rewards_catalogue retains full history
```

### Monthly Batch Activation (enterprise orgs — 2000 rewards)
```
At batch startDate (month start):
  afterCommit() fires per reward → 2000 messages on RMQ
  Batch listener consumes in groups:
    One MySQL IN query per batch
    One MongoDB bulkWrite per batch
  active_rewards populated for new month
  Prior month already purged by TTL
```

---

## 7. API Contract

```
GET /core/v1/rewards/catalogue

Query Parameters:
  orgId         String    required
  groupIds      String[]  optional  — comma-separated or repeated param
  labels        String[]  optional  — comma-separated or repeated param
  pointsMin     Integer   optional
  pointsMax     Integer   optional
  pointsExact   Integer   optional  — mutually exclusive with pointsMin/pointsMax
  isEnabled     Boolean   optional  — default: true
  cursor        String    optional  — base64 encoded { pointsValue, _id }
  limit         Integer   optional  — default: 50, max: 50

Response:
{
  "items":      [ RewardSummary ],
  "nextCursor": "base64token | null",
  "total":      Integer              // count of matching active rewards
}

RewardSummary:
{
  "rewardId":    String,
  "name":        String,             // lang-resolved from langDetails
  "imageUrl":    String,
  "pointsValue": Integer,
  "groupIds":    String[],
  "labels":      String[],
  "startDate":   ISO8601,
  "endDate":     ISO8601,
  "isEnabled":   Boolean
}
```

---

## 8. Constraints and Guardrails for Agent

| Constraint | Detail |
|---|---|
| Do not re-evaluate MySQL vs MongoDB | Decision is final |
| Do not add `isActive` materialized field | Use `active_rewards` TTL collection instead |
| Do not use `skip()` for deep pagination | Cursor on `(pointsValue, _id)` only |
| `orgId` must be first field in every query and index | Non-negotiable tenant boundary |
| Apply `maxTimeMS(100)` on all MongoDB queries | Fail fast — SLO guard |
| Never publish RMQ message inside transaction body | Use `afterCommit()` hook only |
| Listener always fetches fresh state from MySQL | Never trust message payload as reward data |
| Do not over-index array fields | Only `groupIds` and `labels` as specified |
| Sync is RMQ async — not synchronous dual-write | No direct MongoDB write in reward write path |
| `active_rewards` is not the source of truth | Never read display fields from it |
| Cache invalidation is per-org prefix | `catalogue:{orgId}:*` on every mutation |
| Upsert semantics on both collections | Ensures listener replay is always safe |

---

## 9. Open Questions for Agent to Resolve During Tech Detailing

1. **Cursor token format** — encode as base64 JSON `{ pointsValue, _id }` client-side, or opaque server-side cursor stored in Redis?
2. **Lang resolution** — does listing API resolve `langDetails` to single locale via `Accept-Language` header, or return full map and let client resolve?
3. **Unfiltered query guard** — should API reject or warn when no filters beyond `orgId` are provided for orgs with 2000 active rewards, or silently apply `limit=50`?
4. **Batch activation job** — is the month-start activation a new `@Scheduled` bean in `sol-rewards-core`, or a separate CronJob? Confirm with service ownership constraints.
5. **`pointsExact` vs range mutual exclusivity** — validate at controller layer (400 response) or service layer?
6. **DLQ alerting** — confirm New Relic NRQL alert or existing RMQ monitoring already covers DLQ depth threshold.

---

## 10. Skill Pipeline Execution Notes

```
/arch-investigator
  Input : this document
  Goal  : map existing sol-rewards-core codebase.
          Find: existing reward fetch patterns, MySQL reward tables and
          relationships, MongoDB config + existing collections,
          Redis config + existing cache patterns, RMQ config +
          existing exchange/queue setup.
          Confirm active_rewards collection does not already exist.
          Confirm rewards_catalogue collection does not already exist.
          Identify existing @Transactional patterns for afterCommit hook placement.
          Output: arch-investigation.md

/tech-detailer
  Input : arch-investigation.md + this document
  Goal  : full technical spec covering —
          API contract (resolve open questions from section 9),
          Controller + Service + Repository layer design,
          MongoTemplate query implementations for active_rewards,
          RewardCatalogueSyncService (RMQ listener) design,
          RMQ exchange/queue/DLQ bean configuration,
          afterCommit publisher implementation,
          Batch listener factory configuration,
          Redis cache key strategy + invalidation,
          MongoDB index DDL (ensureIndex on startup or @Document),
          TTL index setup for active_rewards.
          Output: tech-detail.md

/tech-detail-reviewer
  Input : tech-detail.md
  Goal  : challenge design against constraints in section 8.
          Verify: orgId-first on every index, cursor pagination correct,
          maxTimeMS on every query, afterCommit not inside transaction,
          listener fetches MySQL not trusting payload,
          cache invalidation covers all mutation paths,
          upsert semantics preserved on both collections.
          Write feedback to tech-detailer learnings file if gaps found.
          Output: tech-detail-reviewed.md

/test-plan-architect
  Input : tech-detail-reviewed.md
  Goal  : test plan covering —
          Filter combinations (all filters, subset, none beyond orgId),
          TTL expiry behaviour (reward removed from active_rewards post-endDate),
          Start boundary activation (reward appears after startDate),
          RMQ listener: UPSERT, DELETE, idempotent replay, DLQ on failure,
          afterCommit: no publish on MySQL rollback,
          Batch consume: 2000 reward burst,
          Cache: hit path, miss path, invalidation on mutation,
          Multi-tenant isolation (orgId boundary),
          SLO: p95 < 150ms under load,
          Edge cases: no active rewards for org, all filters simultaneously,
          unknown orgId, pointsExact + range conflict.
          Output: test-plan.md

/test-case-sheet
  Input : test-plan.md
  Goal  : detailed test cases with given/when/then.
          Each case specifies: MongoDB query shape expected,
          Redis interaction expected, RMQ message shape,
          response body + status code.
          Output: test-cases.md

/tdd-developer
  Input : tech-detail-reviewed.md + test-cases.md
  Goal  : implement in sol-rewards-core. Write failing tests first.
          Classes to create:
            RewardCatalogueController
            RewardCatalogueService
            ActiveRewardRepository        (MongoTemplate)
            RewardCatalogueRepository     (MongoTemplate)
            RewardCatalogueSyncListener   (RMQ batch listener)
            RewardCatalogueEventPublisher (afterCommit RMQ publish)
            CatalogueCache                (Redis)
            RewardCatalogueDoc            (MongoDB document)
            ActiveRewardDoc               (MongoDB document)
            RewardCatalogueEvent          (RMQ message)
          Index setup via ensureIndex on application startup or
          @Document annotation — confirm with arch-investigation findings.
          Output: working implementation + passing tests.
```

---

*All design decisions in sections 3–6 are final and validated through architectural review. Agent treats section 8 as hard guardrails, section 9 as the only open decision surface, and section 10 as the execution contract for skill handoffs.*