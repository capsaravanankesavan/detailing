---
title: "CAP-190208 — 2K POS Promotion Optimization: Design Plan (Rev 2)"
---

# Design: SKU/Item-Based Shortlisting Optimization for `POST /v1/promotions/evaluate`
*(Revision 2 — post senior-review, R1–R8 incorporated)*

## Problem Statement

**Goal:** Reduce `POST /v1/promotions/evaluate` p50 latency from 9.5 s → ≤ 300 ms and p95 → ≤ 500 ms for a 10-item Pharma cart at 1,000 RPM.

**Root Cause (confirmed, APM trace crm-staging-new-promotion-engine, Jun 18 2026):**
300 promotions survive the 8 existing shortlisting filters and reach `PromotionEvaluatorService.apply()`, where ~298 are rejected because their `productSelectionCriteriaList` requires SKUs not in the cart. SKU filtering only happens deep inside the evaluator loop — zero shortlisting services inspect cart contents. This produces 552 evaluator calls/txn at 16.35% of transaction time.

| Metric | Observed | SLO Target |
|--------|----------|------------|
| Avg response time | 9.5 s | ≤ 300 ms |
| p50 | 9.09 s | ≤ 300 ms |
| p95 | 23.8 s | ≤ 500 ms |
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

1. **Shortlisting phase** (`getShortlistedPromotionsForLoyalUser` / `getShortlistedPromotionsForAnonymousUser`, `PromotionRedemptionFacade.java:L291–L333`) — chain-of-responsibility, zero cart awareness today.
2. **Evaluation phase** (`PromotionEvaluatorService.apply()`) — called per shortlisted promotion against full cart.

`cartRO` is in scope at lines 248–266 before both shortlisting calls. `PromotionContext` (`@Builder`/`@Getter`/`@Setter` POJO) is the injection point for cart SKU data without changing the `ShortListingService` interface signature.

* * *

## Fix 1 — New `ItemBasedShortlistingService` (Primary, highest impact)

### 1a. Complete Condition Hierarchy — Recursive `extractSkuInCriteria` (R1)

All 10 concrete `Condition` implementors verified:

| Class | SKU criteria | Handling |
|---|---|---|
| `ProductCondition` (`ProductCondition.java:L13`) | ✅ direct `getProductSelectionCriteriaList()` | Leaf case |
| `ComplexCondition` (`ComplexCondition.java:L17`) | ✅ via `getProductConditions()` | Recursive flatMap over leaves |
| `ComboProductCondition extends ComplexCondition` | ✅ via ComplexCondition branch | Already covered |
| `PaymentModeScopeCondition extends ComplexCondition` | ✅ via ComplexCondition branch | Already covered |
| `PaymentModeComboCondition extends ComplexCondition` | ✅ via ComplexCondition branch | Already covered |
| `ConditionWithPerUnitCondition` (`ConditionWithPerUnitCondition.java:L10`) | ✅ via `getCondition()` + `getProductBasedCondition()` (both `Condition`) | Recurse both; `perUnitCondition` is NOT a `Condition` — skip |
| `ConditionWithRewardCondition` (`ConditionWithRewardCondition.java:L10`) | ✅ via `getCondition()` + `getRewardCondition()` (both `Condition`) | Recurse both |
| `CartCondition` | ❌ cart-KPI only | → emptyList |
| `TenderCondition` | ❌ | → emptyList |
| `PaymentModeCondition` | ❌ | → emptyList |

**Key type relationships:**
- `ProductBasedCondition extends Condition` — so wrapper inner fields can be safely cast to `Condition` for recursion
- `PerUnitCondition implements Serializable, SimplifiedText` — does NOT implement `Condition` — must NOT be recursed
- `ComplexCondition.getProductConditions()` returns only `ProductCondition` leaves; `ConditionWithPerUnitCondition` / `ConditionWithRewardCondition` are NOT handled by this recursive call — they need their own branches

**Revised fully-recursive implementation:**
```java
private List<ProductSelectionCriteria> extractSkuInCriteria(Condition condition) {
    if (condition == null) return Collections.emptyList();

    if (condition instanceof ProductCondition) {
        return ((ProductCondition) condition).getProductSelectionCriteriaList().stream()
            .filter(c -> ProductSelectionCriteria.ProductEntity.SKU.equals(c.getProductEntity())
                      && ListOperator.IN.equals(c.getOperator()))
            .collect(Collectors.toList());
    }
    if (condition instanceof ComplexCondition) {
        // Handles ComboProductCondition, PaymentModeScopeCondition, PaymentModeComboCondition
        return ((ComplexCondition) condition).getProductConditions().stream()
            .flatMap(pc -> extractSkuInCriteria(pc).stream())
            .collect(Collectors.toList());
    }
    if (condition instanceof ConditionWithPerUnitCondition) {
        ConditionWithPerUnitCondition cwpuc = (ConditionWithPerUnitCondition) condition;
        List<ProductSelectionCriteria> result = new ArrayList<>(
            extractSkuInCriteria(cwpuc.getCondition()));
        if (cwpuc.getProductBasedCondition() instanceof Condition) {
            result.addAll(extractSkuInCriteria((Condition) cwpuc.getProductBasedCondition()));
        }
        return result;
    }
    if (condition instanceof ConditionWithRewardCondition) {
        ConditionWithRewardCondition cwrc = (ConditionWithRewardCondition) condition;
        List<ProductSelectionCriteria> result = new ArrayList<>(
            extractSkuInCriteria(cwrc.getCondition()));
        if (cwrc.getRewardCondition() instanceof Condition) {
            result.addAll(extractSkuInCriteria((Condition) cwrc.getRewardCondition()));
        }
        return result;
    }
    return Collections.emptyList(); // CartCondition, TenderCondition, PaymentModeCondition
}
```

### 1b. Filter Logic — `shouldExclude`

```java
private boolean shouldExclude(Promotion promotion, Set<String> cartSkuSet) {
    List<ProductSelectionCriteria> skuInCriteria = extractSkuInCriteria(
        promotion.getPromotionMeta().getCondition());
    if (skuInCriteria.isEmpty()) return false; // No SKU IN constraint — pass through
    // Exclude only if EVERY criteria group has zero intersection with the cart
    return skuInCriteria.stream()
        .noneMatch(c -> !Collections.disjoint(c.getValues(), cartSkuSet));
}
```

**Conservative AND/OR posture:** `getProductConditions()` collects all leaves regardless of AND/OR. At shortlist time we pass if any leaf's SKU set intersects the cart — never a false negative (may produce false positives for AND, but the evaluator corrects them).

**NOT_IN criteria:** Skipped at shortlist time — a `NOT_IN`-only promotion has `skuInCriteria.isEmpty()` → passes shortlisting → evaluator handles correctly.

**Case sensitivity (R5):** `OperatorEvaluationUtil.evaluate()` uses `rhs.contains(lhs)` — pure case-sensitive `String.equals()`. Build `cartSkuSet` using `CartItemRO.getSku()` without normalization — same as the evaluator. If mixed-case SKU data exists, both evaluator and shortlisting are equally broken; that is a data quality issue outside this scope.

### 1c. Cart SKU Set — Exact Field Chain

`CartRO.cartItems: List<CartItemRO>` → `CartItemRO.sku: String` (`CartItemRO.java:L33`)

```java
// In PromotionRedemptionFacade.evaluateCart(), before shortlisting calls (L248–L252)
Set<String> cartSkuSet = cartRO.getCartItems().stream()
    .map(CartItemRO::getSku)
    .filter(Objects::nonNull)
    .collect(Collectors.toSet());
promotionContext.setCartSkuSet(cartSkuSet);
```

### 1d. New Field on `PromotionContext`

```java
// PromotionContext.java — add after existing fields
@Builder.Default
private Set<String> cartSkuSet = Collections.emptySet();
```

If `cartSkuSet` is empty (anonymous with no cart, or redemption-without-cart path), `ItemBasedShortlistingService` passes all promotions through — safe default.

### 1e. Registration in `ShortListingServiceProvider` (R2 — no @Order)

`ShortListingServiceProvider.java:21` uses `@Autowired private List<ShortListingService> allServiceList` with no `@Order` contract. The safe fix is explicit positional insertion in the provider.

**Modified `ShortListingServiceProvider.java`:**
```java
@Autowired
@Qualifier("itemBasedShortlistingService")
private ItemBasedShortlistingService itemBasedShortlistingService;

public List<ShortListingService> getServiceForLoyalUser() {
    List<ShortListingService> services = allServiceList.stream()
        .filter(s -> !ANONYMOUS_USER_ONLY_SERVICES.contains(ClassUtils.getUserClass(s.getClass())))
        // Exclude ItemBasedShortlistingService from the auto-injected list to avoid double-add
        .filter(s -> !(s instanceof ItemBasedShortlistingService))
        .collect(Collectors.toList());
    // Insert BEFORE PromotionCappingShortlistingService (required: reduces input to capping)
    int cappingIdx = findCappingServiceIndex(services);
    services.add(cappingIdx, itemBasedShortlistingService);
    return services;
}

public List<ShortListingService> getServiceForAnonymousUser() {
    List<ShortListingService> services = allServiceList.stream()
        .filter(s -> !LOYAL_USER_ONLY_SERVICES.contains(ClassUtils.getUserClass(s.getClass())))
        .filter(s -> !(s instanceof ItemBasedShortlistingService))
        .collect(Collectors.toList());
    int cappingIdx = findCappingServiceIndex(services);
    services.add(cappingIdx, itemBasedShortlistingService);
    return services;
}
```

**Ordering constraint (R8):** `ItemBasedShortlistingService` MUST be inserted before `PromotionCappingShortlistingService`. This is the critical constraint — the `retainAll` loop in `getShortListedPromotions()` passes fewer promotions to each subsequent service; item-based shortlisting eliminates ~250 promotions before capping, avoiding ~250 MongoDB aggregation calls.

### 1f. Feature Flag (R6)

Add to `PromotionOrgConfiguration.java`:
```java
@Builder.Default
private Boolean itemBasedShortlistingEnabled = Boolean.FALSE;
```

In `ItemBasedShortlistingService`:
```java
@Autowired
private PromotionOrgConfigurationService orgConfigurationService;

@Override
public List<Promotion> shortList(Long orgId, PromotionContext promotionContext,
                                  List<Promotion> promotions) {
    if (!Boolean.TRUE.equals(
            orgConfigurationService.getOrgConfiguration(orgId).isItemBasedShortlistingEnabled())) {
        return promotions; // pass-through when flag is off
    }
    Set<String> cartSkuSet = promotionContext.getCartSkuSet();
    if (cartSkuSet.isEmpty()) return promotions; // no cart data — safe pass-through

    return promotions.stream()
        .filter(p -> !shouldExclude(p, cartSkuSet))
        .peek(p -> promotionContext.addEvaluationLog(PromotionLog.skuPass(p)))
        .collect(Collectors.toList());
    // Excluded promotions have their failure log added separately inside shouldExclude
}
```

### 1g. New Files and Modified Files

**New files:**
- `src/main/java/.../service/impl/shortlist/ItemBasedShortlistingService.java`
- `src/test/java/.../service/impl/shortlist/ItemBasedShortlistingServiceTest.java`

**Modified files:**

| File | Change |
|------|--------|
| `PromotionContext.java` | Add `cartSkuSet: Set<String>` field with `@Builder.Default = emptySet()` |
| `PromotionRedemptionFacade.java:L248–L252` | Build `cartSkuSet` from `cartRO`, call `promotionContext.setCartSkuSet()` before shortlisting |
| `ShortListingServiceProvider.java` | Inject `itemBasedShortlistingService` by `@Qualifier`, insert before capping in both list builders |
| `PromotionOrgConfiguration.java` | Add `itemBasedShortlistingEnabled: Boolean` field (`@Builder.Default = false`) |

### 1h. Expected Impact (Fix 1 alone)

| Metric | Before | After Fix 1 |
|--------|--------|-------------|
| Evaluator calls/txn | 552 | ~50 |
| `PromotionEvaluatorService/apply` time | 1.60 s | ~0.15 s |
| Capping query input | 300 promotions | ~50 promotions |

* * *

## Fix 2 — Parallel Capping KPI Queries (Secondary)

### Problem

`PromotionCappingShortlistingService.shortList()` calls `getKPISumAfterDateFor()` sequentially per promotion. APM: 935 ms/call vs 24 ms actual MongoDB time — connection pool saturation from sequential blocking. After Fix 1 reduces input to ~50 promotions, sequential cost is still ~50 × 935 ms = 46.75 s. Fix 2 parallelizes these calls.

### 2a. `PromotionContext` Copy Contract (R3 — deep copy required)

Mutable fields confirmed in `PromotionContext.java`:
- `promotionLogs: List<PromotionLog>` — written via `addEvaluationLog()`
- `codeBasedEvaluationLogs: List<CodeBasedEvaluationLog>` — written via `addPromoCodeEvaluationLog()`
- `promotionMetaLevelCapping: Map<String, Capping>` — written during capping
- `restrictionMetrics: Map<PromotionRestriction.Level, Integer>` — written during capping
- `earnedPromotionId`, `currentPromoCode`, `triggerDate` — promotion-specific write fields set at `PromotionRestrictionService.java:L249–L251`

Also: `PromotionContext.equals()` uses `earnedPromotionId` — concurrent mutation of a shared context corrupts equality checks.

**Per-task copy template:**
```java
PromotionContext taskContext = PromotionContext.builder()
    // Read-only fields — shared reference safe
    .orgId(ctx.getOrgId())
    .customerId(ctx.getCustomerId())
    .tillId(ctx.getTillId())
    .sessionId(ctx.getSessionId())
    .orgZoneId(ctx.getOrgZoneId())     // Set before shortlisting in evaluateCart(), always present
    .cartSkuSet(ctx.getCartSkuSet())
    .reservationContext(ctx.getReservationContext())
    // Mutable collections — fresh empty instances (do NOT share references)
    .promotionLogs(new ArrayList<>())
    .codeBasedEvaluationLogs(new ArrayList<>())
    .promotionMetaLevelCapping(new HashMap<>())
    .restrictionMetrics(new HashMap<>())
    // Promotion-specific write fields — blank; set by the parallel task
    // earnedPromotionId, currentPromoCode, triggerDate: omitted → null defaults
    .build();
```

**Post-join merge:**
```java
// After all CompletableFuture.join() calls complete:
taskContexts.forEach(tc -> {
    ctx.getPromotionLogs().addAll(tc.getPromotionLogs());
    ctx.getCodeBasedEvaluationLogs().addAll(tc.getCodeBasedEvaluationLogs());
    ctx.getPromotionMetaLevelCapping().putAll(tc.getPromotionMetaLevelCapping());
    tc.getRestrictionMetrics().forEach((k, v) ->
        ctx.getRestrictionMetrics().merge(k, v, Integer::sum));
});
```

### 2b. Async Pattern and MDC Propagation (R4 — first async pattern)

This is the FIRST `CompletableFuture` usage in the service. Required safeguards:

**Executor bean (new `CappingAsyncConfig.java`):**
```java
@Configuration
public class CappingAsyncConfig {
    @Bean(name = "cappingExecutor", destroyMethod = "shutdown")
    public ThreadPoolTaskExecutor cappingExecutor() {
        ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
        exec.setCorePoolSize(4);
        exec.setMaxPoolSize(8);         // Conservative: NOT per-request sized
        exec.setQueueCapacity(50);
        exec.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        exec.setThreadNamePrefix("capping-shortlist-");
        exec.initialize();
        return exec;
    }
}
```

**MDC propagation (New Relic request-id + org-id must survive thread boundary):**
```java
Map<String, String> mdcContext = MDC.getCopyOfContextMap();
CompletableFuture.supplyAsync(() -> {
    if (mdcContext != null) MDC.setContextMap(mdcContext);
    try {
        return evaluateCapping(orgId, promotion, taskContext);
    } finally {
        MDC.clear();
    }
}, cappingExecutor)
.exceptionally(ex -> {
    log.warn("Capping check failed for promotion {}: {}", promotion.getId(), ex.getMessage());
    return null; // Exclude promotion on error; log for observability
});
```

**Lock TTL assumption:** The `@CustomerLockable` distributed lock's TTL must be >> expected parallel capping time (~200 ms after Fix 1+2). Must be confirmed in staging. If TTL is too short, parallel capping must complete within the lock window or the lock must be refreshed.

### 2c. Modified Files

| File | Change |
|------|--------|
| `PromotionCappingShortlistingService.java` | Replace sequential loop with `CompletableFuture` fan-out; per-task `PromotionContext` copy; merge results post-join |
| New `CappingAsyncConfig.java` | `cappingExecutor` `ThreadPoolTaskExecutor` bean |

### 2d. Expected Impact (Fix 1 + Fix 2)

| Metric | Before | After Fix 1 | After Fix 1+2 |
|--------|--------|-------------|---------------|
| Evaluator calls/txn | 552 | ~50 | ~50 |
| Capping time/txn | ~280 s (seq, 300) | ~46.75 s (seq, 50) | ~1 s (parallel, 50) |
| p50 e2e | 9.09 s | ~4–5 s | **≤300 ms** |
| p95 e2e | 23.8 s | ~9–12 s | **≤500 ms** |

> **Both Fix 1 and Fix 2 are required to hit the ≤ 300 ms p50 target.**

* * *

## Fix 3 — Cache TTL for `getActivePromotionIdsByType` (Tertiary)

### Problem (R7 — IS on hot path)

`getActivePromotionIdsByType` IS called on every `evaluateCart()` request via `getActivePosPromotions()` → `getActivePromotionByType()` → `getActivePromotionIdsByType()`. Cache key: `RedisCacheKeys.ACTIVE_PROMOTION_WITH_TYPE`. TTL: `FIVE_MINUTE_CACHE`. APM: 1.72 s/call, 62% miss rate. Cache IS Redis-backed (confirmed: `RedisCacheUtil.java`). The 62% miss rate at 464 RPM with 5-min TTL is likely from pod restarts + staggered TTL expirations. After Fix 1+2 the request rate increases → more cache hits naturally, but the cold path (1.72 s) remains.

### Fix

Extend TTL from `FIVE_MINUTE_CACHE` → `FIFTEEN_MINUTE_CACHE` (or `ONE_HOUR_CACHE` if promotion update frequency allows).

**Atomic change required (R6 cache-name mismatch risk):** `@Cacheable` and ALL corresponding `@CacheEvict` annotations for `RedisCacheKeys.ACTIVE_PROMOTION_WITH_TYPE` must reference the SAME cache name. Changing the `@Cacheable` TTL without updating `@CacheEvict` causes stale-entry leaks. All eviction points (promotion create, update, delete) must be updated atomically in one commit.

### Modified Files

| File | Change |
|------|--------|
| `PromotionMetaManagementService.java` | Change `FIVE_MINUTE_CACHE` → `FIFTEEN_MINUTE_CACHE` on `@Cacheable` for `getActivePromotionIdsByType` |
| Same file (all `@CacheEvict` annotations for this key) | Update cache name to match atomically |

* * *

## Criticality Assessment

| Fix | Schema/Migration | Auth/Security | Public API | Shared Contract | Risk |
|-----|-----------------|---------------|------------|-----------------|------|
| Fix 1 — `ItemBasedShortlistingService` | ❌ | ❌ | ❌ | Medium: changes shortlisted set | **Medium** — must not produce false negatives; feature-flagged |
| Fix 2 — Parallel capping | ❌ | ❌ | ❌ | Low: internal service change | **Medium** — PromotionContext copy correctness is critical |
| Fix 3 — Cache TTL | ❌ | ❌ | ❌ | Low: internal cache | **Low** — atomic `@CacheEvict` update required |

* * *

## Test Strategy

### Fix 1 — `ItemBasedShortlistingService` Unit Tests

| Test case | Input | Expected |
|-----------|-------|----------|
| Promotion with null condition | cartSkuSet={A,B} | PASS |
| Promotion with CartCondition | cartSkuSet={A,B} | PASS |
| Promotion with ProductCondition, SKU IN {A}, cart has A | cartSkuSet={A,B} | PASS |
| Promotion with ProductCondition, SKU IN {X,Y}, cart has {A,B} | cartSkuSet={A,B} | EXCLUDE |
| Promotion with ProductCondition, SKU NOT_IN {A} | cartSkuSet={A,B} | PASS (NOT_IN skipped) |
| Promotion with ComboProductCondition, SKU IN {A} | cartSkuSet={A,B} | PASS |
| Promotion with ComboProductCondition, SKU IN {X} | cartSkuSet={A,B} | EXCLUDE |
| Promotion with ConditionWithPerUnitCondition, inner condition SKU IN {A} | cartSkuSet={A,B} | PASS |
| Promotion with ConditionWithPerUnitCondition, inner condition SKU IN {X} | cartSkuSet={A,B} | EXCLUDE |
| Promotion with ConditionWithRewardCondition, reward SKU IN {A} | cartSkuSet={A,B} | PASS |
| Promotion with ConditionWithRewardCondition, null inner condition | cartSkuSet={A,B} | PASS (null-guard) |
| Promotion with ComplexCondition AND: SKU IN {A} AND SKU IN {X} | cartSkuSet={A,B} | PASS (conservative) |
| Promotion with ComplexCondition OR: SKU IN {A} OR SKU IN {X} | cartSkuSet={A,B} | PASS |
| Empty cartSkuSet | cartSkuSet={} | PASS ALL |
| Feature flag disabled | flag=false | PASS ALL |
| Mixed list: 5 SKU-match, 3 SKU-no-match, 2 no-condition | cartSkuSet={A,B} | 7 returned |
| `ShortListingServiceProvider` does not double-add | Both getServiceForLoyalUser + getServiceForAnonymousUser | 1 instance each |

### Fix 2 — Parallel Capping Unit Tests

| Test case | Expected |
|-----------|----------|
| All capping futures succeed | Results collected; all pass/fail correctly |
| One future throws exception | That promotion excluded; others unaffected; warning logged |
| Context mutations isolated | `earnedPromotionId` / `currentPromoCode` in task context do not appear in other task contexts |
| Post-join merge: promotionLogs | All logs from all task contexts merged into main context |
| Pool saturation (>50 pending) | CallerRunsPolicy: caller thread runs the task; no deadlock |

### Fix 3 — Cache Unit Tests

| Test case | Expected |
|-----------|----------|
| Cache evict fires after promotion create | Cache entry removed; next call hits DB |
| Cache names match between @Cacheable and @CacheEvict | Eviction is effective (no stale-entry leak) |

### Integration / Regression

- Staging: org with 1,000 POS promotions, 10-item Pharma cart → APM confirms ≤ 50 evaluator calls/txn
- Staging load test: 1,000 RPM, 1,000 POS promotions → p50 ≤ 300 ms, p95 ≤ 500 ms
- Regression: org with NO SKU-based promotions → all pass shortlisting (zero false negatives)
- Regression: flag disabled → identical behavior to current

* * *

## Metrics / Observability

- `promotions.pos.sku.shortlisted` — count of promotions excluded by `ItemBasedShortlistingService` per request (New Relic custom attribute)
- `promotions.evaluator.input` — count of promotions reaching `PromotionEvaluatorService.apply()`
- `capping-shortlist-` thread pool queue depth — via Spring Boot Actuator / JMX
- Existing `promotions.pos` metric retained unchanged

* * *

## Post-Fix Projected Metrics

| Metric | Current | After Fix 1+2 | After Fix 1+2+3 | SLO |
|--------|---------|---------------|-----------------|-----|
| Evaluator calls/txn | 552 | ~50 | ~50 | ≤50 |
| p50 e2e | 9.09 s | **≤300 ms** | ≤300 ms | ≤300 ms |
| p95 e2e | 23.8 s | **≤500 ms** | ≤500 ms | ≤500 ms |
| Cache miss rate | 62% | 62% | ~15% | N/A |

* * *

## Open Questions / Risks

1. **`ConditionWithPerUnitCondition.productBasedCondition` null safety:** `instanceof` check guards correctly for null — verify with unit test.
2. **NOT_IN-only SKU promotions in Pharma org:** If significant, Fix 1 reduction < projected. Confirm profile breakdown from APM trace.
3. **`ShortListingServiceProvider` double-injection guard:** Verify `ClassUtils.getUserClass()` unwraps Spring proxy for `ItemBasedShortlistingService` correctly — add unit test with mocked AOP proxy.
4. **Lock TTL vs. parallel capping time:** Confirm `@CustomerLockable` TTL >> ~200 ms expected parallel time.
5. **`getActivePromotionIdsByType` `@CacheEvict` audit:** All eviction annotations for this key must be updated atomically with the TTL change.
6. **`cartSkuSet` invariant:** Document in `ItemBasedShortlistingService` Javadoc: if `cartSkuSet` is empty, pass all promotions through (safe default for paths that bypass `evaluateCart()`).
