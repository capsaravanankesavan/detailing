---
title: "CAP-190208 — 2K POS Promotion Optimization: Design Plan (Rev 3)"
---

# Design: SKU/Item-Based Shortlisting Optimization for `POST /v1/promotions/evaluate`
*(Revision 3 — R9: ComplexCondition branch fixed to full inline recursion)*

## Problem Statement

**Goal:** Reduce `POST /v1/promotions/evaluate` p50 latency from 9.5 s → ≤ 300 ms and p95 → ≤ 500 ms for a 10-item Pharma cart at 1,000 RPM.

**Root Cause (confirmed, APM trace crm-staging-new-promotion-engine, Jun 18 2026, 5:10–5:20 PM GMT+5:30):**
300 promotions survive the 8 existing shortlisting filters and reach `PromotionEvaluatorService.apply()`, where ~298 are rejected because their `productSelectionCriteriaList` requires SKUs not in the cart. SKU filtering only happens deep inside the evaluator loop — zero shortlisting services inspect cart contents. 552 evaluator calls/txn at 16.35% of transaction time.

| Metric | Observed | SLO Target |
|--------|----------|------------|
| Avg response time | 9.5 s | ≤ 300 ms |
| p50 | 9.09 s | ≤ 300 ms |
| p95 | 23.8 s | ≤ 500 ms |
| Apdex | 0.16 | ≥ 0.85 |
| Error rate | 27.29% | ≤ 0.5% |
| Throughput | 464 RPM | ≥ 1,000 RPM |

**Acceptance Criteria:**
- ≤ 50 evaluator calls/txn
- p50 ≤ 300 ms, p95 ≤ 500 ms for a 10-item Pharma cart
- Promotions with no SKU criteria MUST pass through unchanged
- Feature-flag controlled per org, defaulting off

* * *

## Fix 1 — New `ItemBasedShortlistingService` (Primary)

### 1a. Fully-Recursive `extractSkuInCriteria` (R1 + R9)

**Why `getProductConditions()` cannot be used for the ComplexCondition branch:**
`ComplexCondition.getProductConditions()` → private `getConditions()` at `ComplexCondition.java:L57–L63` only handles `ProductCondition` and `ComplexCondition` children — silently discards `ConditionWithPerUnitCondition`/`ConditionWithRewardCondition` children. Any wrapper type nested under a `ComplexCondition` node produces a false result. Fix: inline-recurse over `getCondition1()`/`getCondition2()` directly, applying the full dispatch table at every tree level.

All 10 Condition types verified:

| Class | Accessor | Handling |
|---|---|---|
| `ProductCondition` (`ProductCondition.java:L13`) | `getProductSelectionCriteriaList()` | Leaf — filter SKU+IN |
| `ComplexCondition` (`ComplexCondition.java:L9,15,16`) | `getCondition1()`, `getCondition2()` | **Inline recurse both children (R9)** |
| `ComboProductCondition extends ComplexCondition` | via ComplexCondition branch | Covered |
| `PaymentModeScopeCondition extends ComplexCondition` | via ComplexCondition branch | Covered |
| `PaymentModeComboCondition extends ComplexCondition` | via ComplexCondition branch | Covered |
| `ConditionWithPerUnitCondition` (`ConditionWithPerUnitCondition.java:L10`) | `getCondition()`, `getProductBasedCondition()` (both `Condition`) | Recurse both; `perUnitCondition: PerUnitCondition` — NOT a Condition, skip |
| `ConditionWithRewardCondition` (`ConditionWithRewardCondition.java:L10`) | `getCondition()`, `getRewardCondition()` (both `Condition`) | Recurse both |
| `CartCondition`, `TenderCondition`, `PaymentModeCondition` | — | → emptyList |
| `null` | — | → emptyList (null-guard) |

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
        // R9: inline recursion over condition1/condition2 — covers ConditionWithPerUnitCondition
        // and ConditionWithRewardCondition as children. getProductConditions() is NOT used here
        // because its private helper silently drops wrapper types (ComplexCondition.java:L57–63).
        ComplexCondition complex = (ComplexCondition) condition;
        List<ProductSelectionCriteria> result = new ArrayList<>();
        result.addAll(extractSkuInCriteria(complex.getCondition1()));
        result.addAll(extractSkuInCriteria(complex.getCondition2()));
        return result;
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
    return Collections.emptyList();
}
```

### 1b. Filter Logic

```java
private boolean shouldExclude(Promotion promotion, Set<String> cartSkuSet) {
    List<ProductSelectionCriteria> skuInCriteria = extractSkuInCriteria(
        promotion.getPromotionMeta().getCondition());
    if (skuInCriteria.isEmpty()) return false; // no SKU IN constraint — pass through
    return skuInCriteria.stream()
        .noneMatch(c -> !Collections.disjoint(c.getValues(), cartSkuSet));
}
```

**Conservative AND/OR posture:** Pass if any leaf's SKU set intersects the cart — never a false negative (may pass a promotion whose AND leg ultimately fails; evaluator corrects). NOT_IN criteria skipped. Case-sensitive — same as `OperatorEvaluationUtil.evaluate()` (`rhs.contains(lhs)`, `OperatorEvaluationUtil.java:L23–28`); no normalization applied.

### 1c. Cart SKU Set and `PromotionContext` Change

`CartRO.cartItems: List<CartItemRO>` → `CartItemRO.sku: String` (`CartItemRO.java:L33`)

```java
// PromotionContext.java — add field
@Builder.Default
private Set<String> cartSkuSet = Collections.emptySet();

// PromotionRedemptionFacade.evaluateCart() — before shortlisting (L248–L252)
Set<String> cartSkuSet = cartRO.getCartItems().stream()
    .map(CartItemRO::getSku)
    .filter(Objects::nonNull)
    .collect(Collectors.toSet());
promotionContext.setCartSkuSet(cartSkuSet);
```

Empty `cartSkuSet` → pass-through all promotions (safe default).

### 1d. Registration in `ShortListingServiceProvider` (R2)

`ShortListingServiceProvider.java:21` uses `@Autowired List<ShortListingService>` with no `@Order` contract. Fix: inject by `@Qualifier`, explicitly insert before `PromotionCappingShortlistingService` in both list builders:

```java
@Autowired @Qualifier("itemBasedShortlistingService")
private ItemBasedShortlistingService itemBasedShortlistingService;

public List<ShortListingService> getServiceForLoyalUser() {
    List<ShortListingService> services = allServiceList.stream()
        .filter(s -> !ANONYMOUS_USER_ONLY_SERVICES.contains(ClassUtils.getUserClass(s.getClass())))
        .filter(s -> !(s instanceof ItemBasedShortlistingService)) // prevent double-add
        .collect(Collectors.toList());
    int cappingIdx = findCappingServiceIndex(services); // before PromotionCappingShortlistingService
    services.add(cappingIdx, itemBasedShortlistingService);
    return services;
}
// getServiceForAnonymousUser() mirrors the same pattern
```

**Ordering constraint:** Item-based shortlisting MUST run before capping. The `retainAll` loop passes fewer promotions to capping — this is the mechanism that eliminates ~250 expensive MongoDB aggregation calls.

### 1e. Feature Flag

Add to `PromotionOrgConfiguration.java` (existing boolean flag pattern: `isLockingEnabled`, `accumulationEnabled`):
```java
@Builder.Default
private Boolean itemBasedShortlistingEnabled = Boolean.FALSE;
```

`ItemBasedShortlistingService` injects `PromotionOrgConfigurationService` and checks the flag at the top of `shortList(orgId, ...)`.

### 1f. New/Modified Files

| File | Type | Change |
|------|------|--------|
| `src/main/java/.../service/impl/shortlist/ItemBasedShortlistingService.java` | NEW | Full implementation |
| `src/test/java/.../service/impl/shortlist/ItemBasedShortlistingServiceTest.java` | NEW | Unit tests |
| `PromotionContext.java` | MODIFY | Add `cartSkuSet: Set<String>` field |
| `PromotionRedemptionFacade.java:L248–L252` | MODIFY | Build + set `cartSkuSet` before shortlisting |
| `ShortListingServiceProvider.java` | MODIFY | Inject + positionally insert new service |
| `PromotionOrgConfiguration.java` | MODIFY | Add `itemBasedShortlistingEnabled` flag |

* * *

## Fix 2 — Parallel Capping KPI Queries (Secondary)

### Problem

`PromotionCappingShortlistingService.shortList()` calls `getKPISumAfterDateFor()` sequentially per promotion. APM: 935 ms/call vs 24 ms actual MongoDB time — connection pool saturation. After Fix 1 reduces input to ~50 promotions, sequential cost is still ~50 × 935 ms. Fix 2 parallelizes.

### 2a. `PromotionContext` Copy Contract (R3)

All mutable fields confirmed in `PromotionContext.java`. Per-task copy must have:

- **Read-only (copy by reference):** `orgId`, `customerId`, `tillId`, `sessionId`, `orgZoneId`, `cartSkuSet`, `reservationContext`
- **Mutable collections (fresh empty instances):** `promotionLogs: new ArrayList<>()`, `codeBasedEvaluationLogs: new ArrayList<>()`, `promotionMetaLevelCapping: new HashMap<>()`, `restrictionMetrics: new HashMap<>()`
- **Promotion-specific write fields (blank — set by task):** `earnedPromotionId`, `currentPromoCode`, `triggerDate`

**Post-join merge:**
```java
taskContexts.forEach(tc -> {
    mainCtx.getPromotionLogs().addAll(tc.getPromotionLogs());
    mainCtx.getCodeBasedEvaluationLogs().addAll(tc.getCodeBasedEvaluationLogs());
    mainCtx.getPromotionMetaLevelCapping().putAll(tc.getPromotionMetaLevelCapping());
    tc.getRestrictionMetrics().forEach((k, v) ->
        mainCtx.getRestrictionMetrics().merge(k, v, Integer::sum));
});
```

Note: `PromotionContext.equals()` uses `earnedPromotionId` — concurrent mutation of a shared context would corrupt equality checks; per-task copy eliminates this.

### 2b. Async Pattern (R4 — first async usage in service)

**Executor bean (new `CappingAsyncConfig.java`):**
```java
@Configuration
public class CappingAsyncConfig {
    @Bean(name = "cappingExecutor", destroyMethod = "shutdown")
    public ThreadPoolTaskExecutor cappingExecutor() {
        ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
        exec.setCorePoolSize(4);
        exec.setMaxPoolSize(8);
        exec.setQueueCapacity(50);
        exec.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        exec.setThreadNamePrefix("capping-kpi-");
        exec.initialize();
        return exec;
    }
}
```

**MDC propagation + error isolation:**
```java
Map<String, String> mdcContext = MDC.getCopyOfContextMap();
CompletableFuture<Promotion> future = CompletableFuture.supplyAsync(() -> {
    if (mdcContext != null) MDC.setContextMap(mdcContext);
    try { return evaluateCapping(orgId, promotion, taskContext); }
    finally { MDC.clear(); }
}, cappingExecutor)
.exceptionally(ex -> {
    log.warn("Capping check failed for promo {}: {}", promotion.getId(), ex.getMessage());
    return null;
});
```

**Lock TTL confirmed safe:** `DEFAULT_UNLOCK_TTL = 900 s` >> ~200 ms expected parallel capping time. Per-org override must stay > 5 s.

### 2c. Modified Files

| File | Change |
|------|--------|
| `PromotionCappingShortlistingService.java` | Replace sequential loop with `CompletableFuture` fan-out; per-task context copy; post-join merge |
| New `CappingAsyncConfig.java` | `cappingExecutor` bean |

* * *

## Fix 3 — Cache TTL for `getActivePromotionIdsByType` (Tertiary — on hot path, usually cached)

`getActivePromotionIdsByType` IS called every request via `getActivePosPromotions()`. Cache IS Redis-backed (`RedisCacheUtil.java`). 62% miss rate with 5-min TTL from pod restarts/staggered expiry.

**Fix:** Extend `FIVE_MINUTE_CACHE` → `FIFTEEN_MINUTE_CACHE` on `@Cacheable` AND all corresponding `@CacheEvict` annotations for `RedisCacheKeys.ACTIVE_PROMOTION_WITH_TYPE` — **must change atomically in one commit**. Divergence causes stale-entry leaks (evict fires against old cache name; new TTL cache retains stale entry).

| File | Change |
|------|--------|
| `PromotionMetaManagementService.java` | `@Cacheable` + all `@CacheEvict` for this key: `FIVE_MINUTE_CACHE` → `FIFTEEN_MINUTE_CACHE` |

* * *

## Criticality

| Fix | Risk | Notes |
|-----|------|-------|
| Fix 1 — `ItemBasedShortlistingService` | **Medium** | Changes shortlisted set; feature-flagged; false-negative prevention is the critical correctness requirement |
| Fix 2 — Parallel capping | **Medium** | First async pattern; PromotionContext copy correctness critical |
| Fix 3 — Cache TTL | **Low** | Atomic `@CacheEvict` update required |

* * *

## Test Strategy

### Fix 1 Unit Tests (`ItemBasedShortlistingServiceTest.java`)

| Test | cartSkuSet | Expected |
|------|-----------|----------|
| Null condition | {A,B} | PASS |
| CartCondition | {A,B} | PASS |
| ProductCondition SKU IN {A}, cart has A | {A,B} | PASS |
| ProductCondition SKU IN {X,Y}, cart has {A,B} | {A,B} | EXCLUDE |
| ProductCondition SKU NOT_IN {A} | {A,B} | PASS (NOT_IN skipped) |
| ComboProductCondition SKU IN {A} | {A,B} | PASS |
| ComboProductCondition SKU IN {X} | {A,B} | EXCLUDE |
| ConditionWithPerUnitCondition, inner condition SKU IN {A} | {A,B} | PASS |
| ConditionWithPerUnitCondition, inner condition SKU IN {X} | {A,B} | EXCLUDE |
| ConditionWithRewardCondition, reward SKU IN {A} | {A,B} | PASS |
| ConditionWithRewardCondition, null inner condition | {A,B} | PASS (null-guard) |
| **ComplexCondition(ConditionWithPerUnitCondition(SKU IN {A}), AND, CartCondition)** **(R9)** | {A} | **PASS** |
| **ComplexCondition(ConditionWithPerUnitCondition(SKU IN {X}), AND, CartCondition)** **(R9)** | {A,B} | **EXCLUDE** |
| ComplexCondition AND: SKU IN {A} AND SKU IN {X} | {A,B} | PASS (conservative) |
| Empty cartSkuSet | {} | PASS ALL |
| Feature flag disabled | any | PASS ALL |
| Mixed list: 5 SKU-match, 3 SKU-no-match, 2 no-condition | {A,B} | 7 returned |
| ShortListingServiceProvider: no double-add | both user types | 1 instance each |

### Fix 2 Unit Tests (extend `PromotionCappingShortlistingServiceTest.java`)

| Test | Expected |
|------|----------|
| All futures succeed | All results collected correctly |
| One future throws | Promotion excluded; others unaffected; warning logged |
| Context mutation isolation | `earnedPromotionId`/`currentPromoCode` in task context do not affect other tasks |
| Post-join merge: promotionLogs | All logs from all task contexts merged into main context |
| Pool saturation | CallerRunsPolicy: no deadlock |

### Fix 3 Unit Tests

| Test | Expected |
|------|----------|
| Cache evict after promo create | Entry removed; next call hits DB |
| `@CacheEvict` targets `FIFTEEN_MINUTE_CACHE` | Eviction is effective (no stale-entry leak) |

### Integration / Regression

- Staging: 1,000 POS promotions, 10-item Pharma cart → APM confirms ≤ 50 evaluator calls/txn
- Load test: 1,000 RPM, 1,000 POS promotions → p50 ≤ 300 ms, p95 ≤ 500 ms
- Regression: `CartCondition`-only org → all promotions pass shortlisting unchanged
- Regression: `ConditionWithPerUnitCondition` promotions with matching SKUs → none incorrectly excluded (R9 guard)
- Regression: flag disabled → identical to current

* * *

## Metrics / Observability

| Metric | Captures |
|--------|----------|
| `promotions.sku.shortlist.in/out/excluded` | Fix 1 effectiveness (New Relic custom attribute) |
| `capping.kpi.parallel.ms` | Fix 2 wall-time |
| `capping.kpi.timeout.fallback` | Alert if > 0 |
| `capping.kpi.task.failures` | Per-task failure count |
| `capping-kpi-` thread pool `QueueSize`/`ActiveCount` | Pool health via Spring Boot Actuator |

* * *

## Post-Fix Projected Metrics

| Metric | Current | After Fix 1+2 | SLO |
|--------|---------|---------------|-----|
| Evaluator calls/txn | 552 | ~50 | ≤ 50 |
| p50 e2e | 9.09 s | ≤ 300 ms | ≤ 300 ms |
| p95 e2e | 23.8 s | ≤ 500 ms | ≤ 500 ms |
| Cache miss rate | 62% | ~15% (after Fix 3) | N/A |

> **Both Fix 1 and Fix 2 are required to hit the ≤ 300 ms p50 target.**

* * *

## Open Questions / Risks

1. **`productBasedCondition` null safety:** `instanceof` check returns false for null — correctly handled. Verify with R9 unit tests.
2. **NOT_IN-only SKU promotions in Pharma org:** Confirm profile breakdown before finalising ≤ 50 calls/txn projection.
3. **`ShortListingServiceProvider` double-injection guard:** Add unit test with Spring AOP proxy to verify `instanceof ItemBasedShortlistingService` filter works post-proxy.
4. **Lock TTL:** `DEFAULT_UNLOCK_TTL = 900 s` >> 200 ms. If per-org `unlockTtl` is ever reduced below 5 s, re-evaluate.
5. **`@CacheEvict` atomicity for Fix 3:** Change cache name in `@Cacheable` and all `@CacheEvict` in one commit.
6. **`cartSkuSet` invariant:** Document in Javadoc: empty `cartSkuSet` → pass all promotions (safe default).
