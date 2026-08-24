---
title: "CAP-190208 — 2K POS Promotion Optimization: Design Plan"
---

# Design: SKU/Item-Based Shortlisting Optimization for `POST /v1/promotions/evaluate`

## Problem Statement

**Goal:** Reduce `POST /v1/promotions/evaluate` p50 latency from 9.5 s → ≤ 300 ms and p95 → ≤ 500 ms for a 10-item Pharma cart at 1,000 RPM.

**Root Cause (confirmed, APM trace crm-staging-new-promotion-engine, Jun 18 2026, 5:10–5:20 PM GMT+5:30):**
300 promotions survive the 8 existing shortlisting filters and reach `PromotionEvaluatorService.apply()`, where ~298 are rejected because their `productSelectionCriteriaList` requires SKUs not in the cart. SKU filtering only happens deep inside the evaluator loop — zero shortlisting services inspect cart contents. This produces 552 evaluator calls/txn at 16.35% of transaction time.

| Metric | Observed (staging) | SLO Target |
|--------|--------------------|------------|
| Avg response time | 9.5 s | ≤ 300 ms |
| p50 | 9.09 s | ≤ 300 ms |
| p95 | 23.8 s | ≤ 500 ms |
| p99 | 27.8 s | — |
| Apdex | 0.16 | ≥ 0.85 |
| Error rate | 27.29% | ≤ 0.5% |
| Throughput | 464 RPM | ≥ 1,000 RPM |

**Acceptance Criteria:**
- ≤ 50 evaluator calls/txn (only promotions with ≥ 1 required SKU present in the cart)
- p50 ≤ 300 ms, p95 ≤ 500 ms for a 10-item Pharma cart
- Promotions with no SKU criteria (brand/category/cart/no-condition) MUST pass through unchanged
- Feature-flag controlled per org, defaulting off until validated

* * *

## Architecture Orientation

The evaluation pipeline has two phases in `PromotionRedemptionFacade.evaluateCart()`:

1. **Shortlisting phase** (`getShortlistedPromotionsForLoyalUser` / `getShortlistedPromotionsForAnonymousUser`) — chain-of-responsibility via `ShortListingServiceProvider` + `getShortListedPromotions()` — runs against promotion metadata only, zero cart awareness today.
2. **Evaluation phase** (`getEvaluatedCartRO`) — `PromotionEvaluatorService.apply()` is called per shortlisted promotion against the full cart.

The `cartRO` is available in `evaluateCart()` at lines 250–266 before both shortlisting calls. The `PromotionContext` is a `@Builder` / `@Getter` / `@Setter` POJO already carrying per-request state — the cleanest injection point for cart SKU data without changing the `ShortListingService` interface.

* * *

## Fix 1 — New `ItemBasedShortlistingService` (Primary, highest impact)

### 1a. Condition Hierarchy — SKU Criteria Extraction

| Class | Has `productSelectionCriteriaList` | Accessor |
|---|---|---|
| `ProductCondition` | ✅ direct field | `condition.getProductSelectionCriteriaList()` — `ProductCondition.java:L13` |
| `ComboProductCondition extends ComplexCondition` | ✅ via `ComplexCondition.getProductConditions()` | `ComplexCondition.java:L17` |
| `CartCondition` | ❌ cart-KPI only (amount/qty), no product criteria | `CartCondition.java` |
| `ComplexCondition` | ✅ recursive via `getProductConditions()` — handles AND/OR trees | `ComplexCondition.java:L17` |
| `TenderCondition`, `PaymentModeCondition`, etc. | ❌ no product criteria | — |
| `null` | ❌ some promotions have no condition | must null-guard |

**SKU extraction (pure helper):**
```java
List<ProductSelectionCriteria> extractSkuInCriteria(Condition condition):
  if condition is null → return emptyList
  if condition instanceof ProductCondition → return condition.getProductSelectionCriteriaList()
                                              filtered to (productEntity == SKU AND operator == IN)
  if condition instanceof ComplexCondition → flatMap all getProductConditions()
                                              collect their productSelectionCriteriaLists
                                              filtered to (productEntity == SKU AND operator == IN)
  else → return emptyList   // CartCondition, TenderCondition, etc. — no SKU constraint
```

**Filter logic (must-pass-through rule):**
```java
boolean shouldExclude(Promotion p, Set<String> cartSkuSet):
  List<ProductSelectionCriteria> skuInCriteria = extractSkuInCriteria(p.getPromotionMeta().getCondition())
  if skuInCriteria.isEmpty() → PASS (no SKU IN constraint → brand/category/cart condition → pass through)
  // Exclude only if EVERY SKU IN group has zero intersection with the cart
  return skuInCriteria.stream()
           .noneMatch(c -> !Collections.disjoint(c.getValues(), cartSkuSet))
```

**NOT_IN criteria skipped:** A `NOT_IN` SKU criteria cannot be safely applied at shortlist time (false-negative risk). Only `IN` criteria are used for exclusion. A promotion with only `NOT_IN` SKU criteria has `skuInCriteria.isEmpty()` → passes shortlisting → evaluator handles it correctly.

**ComplexCondition AND/OR safety:** `getProductConditions()` collects all `ProductCondition` leaves regardless of AND/OR. At shortlist time we pass if **any** leaf's SKU set intersects the cart — conservative (never false-negative; may pass a promotion whose AND leg fails, but the evaluator catches it).

### 1b. Cart SKU Set — Exact Field Chain

`CartRO.cartItems: List<CartItemRO>` → `CartItemRO.sku: String` (`CartItemRO.java:L33`)

Built once in `PromotionRedemptionFacade.evaluateCart()` before shortlisting calls:
```java
Set<String> cartSkuSet = cartRO.getCartItems().stream()
    .map(CartItemRO::getSku)
    .filter(Objects::nonNull)
    .collect(Collectors.toSet());
promotionContext.setCartSkuSet(cartSkuSet);
```

### 1c. Files to Create/Modify

**New file:**
```
src/main/java/.../service/impl/shortlist/ItemBasedShortlistingService.java
```
- Implements `ShortListingService`
- `@Service @Order(2)` — runs after `PromotionValidityShortlistServiceImpl` (validity is the cheapest filter; item-check must run before capping which is the most expensive)
- Reads `promotionContext.getCartSkuSet()`; if empty (anonymous with no cart, or redemption-without-cart path) → passes all promotions through
- Calls `shouldExclude()` for each promotion; builds a failure log entry for excluded promotions

**Corresponding test:**
```
src/test/java/.../service/impl/shortlist/ItemBasedShortlistingServiceTest.java
```

**Modified files:**

| File | Change | Line range |
|------|--------|------------|
| `PromotionContext.java` | Add `private Set<String> cartSkuSet = Collections.emptySet()` with `@Builder.Default`, `@Getter`, `@Setter` | After existing fields |
| `PromotionRedemptionFacade.java` | Build `cartSkuSet` and call `promotionContext.setCartSkuSet(cartSkuSet)` before line 250 (shortlisting calls) | L248–L252 |
| `ShortListingServiceProvider.java` | Register `ItemBasedShortlistingService` in BOTH `getLoyalUserServices()` and `getAnonymousUserServices()` lists | L45–L80 |
| `OrgConfiguration.java` (or org config model) | Add `boolean itemBasedShortlistingEnabled` field (feature flag, default `false`) | New field |
| `PromotionRedemptionFacade.java` | Gate the `setCartSkuSet()` call behind `orgConfig.isItemBasedShortlistingEnabled()` | L248–L252 |

### 1d. Ordering in the Chain

Proposed `@Order` values after this change:

| Order | Service | Rationale |
|-------|---------|-----------|
| 1 | `PromotionValidityShortlistServiceImpl` | Cheapest: date check only |
| 2 | `ItemBasedShortlistingService` ← **NEW** | Early exit on SKU mismatch before expensive capping |
| 3 | `RedeemablePromotionShortListService` | Redeemable-from date |
| 4 | `TimeCriteriaShortlistServiceImpl` | Time recurrence |
| 5 | `StoreShortListServiceImpl` | Store hierarchy lookup |
| 6 | `SupplementaryCriteriaShortListServiceImpl` | Loyalty tier/membership |
| 7 | `CustomerPreferenceShortlistServiceImpl` | Customer opt-in |
| 8 | `PromotionCappingShortlistingService` | Most expensive: MongoDB KPI aggregation |

### 1e. Feature Flag

`OrgConfiguration.itemBasedShortlistingEnabled` (default: `false`).
- In `ItemBasedShortlistingService.shortList()`: if flag is false → return all promotions unchanged (pass-through).
- Rollout: enable per org in staging first, validate ≤ 50 evaluator calls/txn, then enable for Pharma org.

### 1f. Expected Impact (Fix 1 alone)

| Metric | Before | After Fix 1 |
|--------|--------|-------------|
| Evaluator calls/txn | 552 | ~50 (300 shortlisted → ~50 with matching SKUs) |
| `PromotionEvaluatorService/apply` time | 1.60 s/txn | ~0.15 s/txn |
| Capping queries (Fix 2 input) | 300 promotions × N | ~50 promotions × N |

* * *

## Fix 2 — Parallel Capping KPI Queries (Secondary)

### Problem

`PromotionCappingShortlistingService.shortList()` calls `getKPISumAfterDateFor()` sequentially for each promotion. APM shows 935 ms/call vs 24 ms actual MongoDB time — the gap is connection pool saturation from sequential blocking calls accumulating across 300 promotions.

After Fix 1 reduces promotions to ~50, the capping shortlist input shrinks ~6×. However, even 50 × 935 ms = 46.75 s at current pool contention — so parallel execution is still required to hit the ≤ 300 ms target.

### Design

**File to modify:** `PromotionCappingShortlistingService.java` (exact path confirmed in shortlist package)

**Approach:**
```java
// Existing sequential loop → replace with parallel CompletableFuture fan-out
List<CompletableFuture<Promotion>> futures = promotions.stream()
    .map(promotion -> CompletableFuture.supplyAsync(
        () -> evaluateCapping(orgId, promotion, promotionContext),
        cappingExecutor))  // bounded executor, see below
    .collect(Collectors.toList());

List<Promotion> shortlisted = futures.stream()
    .map(f -> f.join())  // collect results; exceptions caught per-future
    .filter(Objects::nonNull)
    .collect(Collectors.toList());
```

**Bounded executor:**
- New `@Bean ThreadPoolTaskExecutor cappingExecutor` in a Spring config class (e.g., `PromotionEngineConfig.java` or a new `CappingAsyncConfig.java`)
- Core pool: 10 threads; max: 20; queue: 50; rejection policy: CallerRunsPolicy (degrades gracefully)
- Named: `capping-shortlist-` (for thread dumps)

**Thread-safety of `PromotionContext` (CRITICAL):**
`getRestrictionsForShortlisting()` calls `context.setEarnedPromotionId()`, `context.setCurrentPromoCode()`, `context.setTriggerDate()` on the shared context (`PromotionRestrictionService.java:L249–L251`). These writes are promotion-specific and must NOT be shared across parallel tasks.

Solution: Pass each parallel task a **shallow copy** of `PromotionContext` with promotion-specific mutable fields blank (they are set by the task). The `cartSkuSet`, `orgId`, `customerId`, and `storeId` fields are read-only for this purpose and are safe to share in the copy.

```java
PromotionContext taskContext = PromotionContext.builder()
    .orgId(promotionContext.getOrgId())
    .customerId(promotionContext.getCustomerId())
    .cartSkuSet(promotionContext.getCartSkuSet())
    .storeId(promotionContext.getStoreId())
    // ... other read-only fields
    // earnedPromotionId, currentPromoCode, triggerDate: left blank — set by task
    .build();
```

Evaluation logs (`addEvaluationLog`) must be collected from each task result and merged back into the main context after all futures complete.

### 2a. Files to Modify

| File | Change |
|------|--------|
| `PromotionCappingShortlistingService.java` | Replace sequential loop with `CompletableFuture` fan-out; use shallow `PromotionContext` copy per task |
| `PromotionEngineConfig.java` (or new `CappingAsyncConfig.java`) | Define `cappingExecutor` `ThreadPoolTaskExecutor` bean |

### 2b. Expected Impact

| Metric | Before | After Fix 1+2 |
|--------|--------|---------------|
| Capping queries/txn | 300 seq. × 935 ms | ~50 parallel (effective: ~935 ms total for 50 promotions) |
| `PromotionCappingShortlistingService` contribution | ~280 s (sequential!) | ~1 s (parallel, bounded pool) |
| p50 e2e | 9.09 s | Target: ≤ 300 ms |

* * *

## Fix 3 — Cache Warming for `getActivePromotionIdsByType` (Tertiary)

### Problem

`getActivePromotionIdsByType` has 62% cache miss rate with 5-minute TTL. APM shows 1.72 s/call. This is not on the primary hot path after Fix 1+2, but contributes to the baseline cost.

### Design

**Step 1 — Determine cache backend:**
`FIVE_MINUTE_CACHE` TTL config must be verified: if backed by Caffeine (in-process per-pod), a Redis-backed cache is needed for consistent TTL across pods. Check `RedisCacheUtil.java` / Spring cache config.

**Step 2 — Cache warming via RabbitMQ promotion lifecycle events:**
If promotion create/update events exist on RabbitMQ (confirmed: 14 RabbitMQ call sites in the service), add a consumer that evicts + pre-warms the `getActivePromotionIdsByType` cache on promotion state change.

**Fallback:** Extend TTL from 5 min → 30 min (safe if promotions change infrequently at POS orgs; must confirm update cadence with SPF brand team).

### 3a. Files to Modify

| File | Change |
|------|--------|
| `RedisCacheUtil.java` (or Spring cache config) | Add/verify Redis-backed cache region for `getActivePromotionIdsByType`; adjust TTL to 30 min |
| New `PromotionCacheWarmupConsumer.java` | RabbitMQ consumer for promotion lifecycle events; evicts + re-warms cache |

* * *

## Criticality Assessment

| Fix | Schema/Migration | Auth/Security | Public API | Shared Contract | Risk |
|-----|-----------------|---------------|------------|-----------------|------|
| Fix 1 — `ItemBasedShortlistingService` | ❌ | ❌ | ❌ | Medium: changes which promotions reach the evaluator | **Medium** — must not produce false negatives |
| Fix 2 — Parallel capping | ❌ | ❌ | ❌ | Low: internal service change; `PromotionContext` mutation is scoped | **Medium** — thread-safety of context copy is critical |
| Fix 3 — Cache warming | ❌ | ❌ | ❌ | Low: internal cache config | **Low** |

* * *

## Test Strategy

### Fix 1 — `ItemBasedShortlistingService`

**Unit tests (ItemBasedShortlistingServiceTest.java):**

| Test case | Input | Expected |
|-----------|-------|----------|
| Promotion with no condition (null) | cartSkuSet={A,B}, condition=null | PASS — included in result |
| Promotion with CartCondition only | cartSkuSet={A,B}, condition=CartCondition | PASS — no SKU criteria |
| Promotion with ProductCondition, SKU IN {A}, cart has A | cartSkuSet={A,B} | PASS — intersects |
| Promotion with ProductCondition, SKU IN {X,Y}, cart has {A,B} | cartSkuSet={A,B} | EXCLUDE — no intersection |
| Promotion with ProductCondition, SKU NOT_IN {A}, cart has {A,B} | cartSkuSet={A,B} | PASS — NOT_IN skipped at shortlist |
| Promotion with ComboProductCondition (SKU IN {A}), cart has A | cartSkuSet={A,B} | PASS |
| Promotion with ComboProductCondition (SKU IN {X}), cart has {A,B} | cartSkuSet={A,B} | EXCLUDE |
| Promotion with ComplexCondition AND: SKU IN {A} AND SKU IN {X} | cartSkuSet={A,B} | PASS — conservative: A intersects |
| Promotion with ComplexCondition OR: SKU IN {A} OR SKU IN {X} | cartSkuSet={A,B} | PASS — A intersects |
| Empty cartSkuSet (anonymous with no cart) | cartSkuSet={} | PASS ALL — pass-through |
| Feature flag disabled | orgConfig.itemBasedShortlistingEnabled=false | PASS ALL — pass-through |
| Mixed list: 5 SKU-match, 3 SKU-no-match, 2 no-condition | cartSkuSet={A,B} | 7 returned, 3 excluded |

### Fix 2 — Parallel Capping

**Unit tests (PromotionCappingShortlistingServiceTest.java — extend existing):**

| Test case | Expected |
|-----------|----------|
| All capping queries succeed in parallel | All results collected correctly |
| One capping query throws exception | That promotion excluded; others unaffected; exception logged |
| Context mutations (earnedPromotionId, promoCode) do not leak across tasks | Each task context is independent |
| Pool saturation (> queue capacity) | CallerRunsPolicy degrades gracefully; no deadlock |

### Fix 3 — Cache Warming

**Unit tests (PromotionCacheWarmupConsumerTest.java):**

| Test case | Expected |
|-----------|----------|
| Promotion created event received | Cache evicted + re-warmed |
| Promotion updated event received | Cache evicted + re-warmed |
| Cache warm-up fails (Mongo unavailable) | Error logged, no exception propagation |

### Integration / Regression

- Staging smoke test: org with 1,000 POS promotions, 10-item Pharma cart → verify ≤ 50 evaluator calls/txn in APM
- Staging load test: 1,000 RPM, 1,000 active POS promotions → verify p50 ≤ 300 ms, p95 ≤ 500 ms
- Regression: org with 0 POS promotions (no SKU criteria) → all promotions pass shortlisting unchanged
- Regression: org with flag disabled → behavior identical to current

* * *

## Metrics / Observability

- Add metric: `promotions.shortlisted.by.sku` — count of promotions excluded by `ItemBasedShortlistingService` per request (New Relic custom attribute via `metricsService.addListSizeAsAttribute`)
- Add metric: `promotions.evaluator.calls` — count of promotions reaching `PromotionEvaluatorService.apply()` (already partially tracked via `ApplicationService/apply` segment; add explicit attribute)
- Existing metric `promotions.pos` already tracks the pre-shortlist POS promotion count — keep; add new `promotions.pos.after.sku.shortlist` for Fix 1's output size
- Monitor: `capping-shortlist-` thread pool queue depth (via Spring Boot Actuator / JMX) to detect pool saturation after Fix 2

* * *

## Post-Fix Projected Metrics

| Metric | Current | After Fix 1 | After Fix 1+2 | After Fix 1+2+3 |
|--------|---------|-------------|---------------|-----------------|
| Evaluator calls/txn | 552 | ~50 | ~50 | ~50 |
| `PromotionEvaluatorService/apply` time | 1.60 s | ~0.15 s | ~0.15 s | ~0.15 s |
| `PromotionCappingShortlistingService` time | ~280 s (seq) | ~46.75 s (50×seq) | ~1 s (parallel) | ~1 s |
| p50 e2e | 9.09 s | ~4–5 s | **≤300 ms** | ≤300 ms |
| p95 e2e | 23.8 s | ~9–12 s | **≤500 ms** | ≤500 ms |
| `getActivePromotionIdsByType` miss rate | 62% | 62% | 62% | ~10% |

> **Both Fix 1 and Fix 2 are required to hit the ≤ 300 ms p50 target.** Fix 1 alone reduces evaluator overhead but the sequential capping queries (still 50 × 935 ms = 46.75 s) still dominate. Fix 2 parallelizes those and brings the total under the SLO.

* * *

## Risks / Open Questions

1. **`PromotionContext` thread-safety in capping parallel path (BLOCKER for Fix 2):** `getRestrictionsForShortlisting()` mutates `context.setEarnedPromotionId()`, `context.setCurrentPromoCode()`, `context.setTriggerDate()` (`PromotionRestrictionService.java:L249–L251`). A shallow copy per parallel task is required. The exact read-only vs. write fields must be confirmed before merging Fix 2.

2. **`NOT_IN` SKU criteria safety:** Design conservatively skips `NOT_IN` at shortlist time. Confirm with product team whether `NOT_IN`-only SKU promotions exist in the Pharma org set.

3. **`ComplexCondition` AND/OR conservative posture:** `getProductConditions()` collects all leaves. Fix 1 passes a promotion if any leaf intersects — correct for OR, conservative (never false-negative) for AND. Acceptable trade-off.

4. **`@Order` stability:** Adding `@Order` annotations to existing shortlisting services for the first time changes the injection order contract. `ShortListingServiceProvider` filtering logic is order-independent, so the change is safe — but requires staging smoke test.

5. **`getActivePromotionIdsByType` cache backend:** 62% miss rate may indicate Caffeine (in-process per-pod) not Redis. Verify `RedisCacheUtil.java` / Spring cache config before implementing Fix 3.

6. **Pharma org promotion profile:** Design assumes ~298/300 promotions are SKU-based (`SKU IN`). If a significant fraction are `CATEGORY IN` or `CartCondition`, Fix 1 reduction will be less. Confirm breakdown from APM trace before committing to ≤ 300 ms p50 projection.

7. **`cartSkuSet` when `isRedemptionWithoutCartEvaluation = true`:** If this path does not feed `getShortListedPromotions()`, `cartSkuSet` defaults to `emptySet()` → all promotions pass (correct safe default). Verify path does not bypass `evaluateCart()`.
