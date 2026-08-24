# Design: 2K POS Promotion Optimization — `POST /v1/promotions/evaluate`
*(Revision 10 — CONSOLIDATED FINAL, post senior-review fidelity fixes)*

## Problem Statement

**Goal:** Reduce `POST /v1/promotions/evaluate` p50 latency from 9.5s → ≤300ms and p95 → ≤500ms for a 10-item Pharma cart at 1,000 RPM, supporting orgs with up to 2,000 active POS promotions.

**Root Cause (confirmed, APM trace crm-staging-new-promotion-engine, Jun 18 2026, 5:10–5:20 PM GMT+5:30):**
300 promotions survive the 8 existing shortlisting filters and reach `PromotionEvaluatorService.apply()`, where ~298 are rejected because their `productSelectionCriteriaList` requires SKUs not in the cart. 552 evaluator calls/txn at 16.35% of transaction time.

| Metric | Observed | SLO Target |
|--------|----------|------------|
| p50 | 9.09s | ≤300ms |
| p95 | 23.8s | ≤500ms |
| Apdex | 0.16 | ≥0.85 |
| Error rate | 27.29% | ≤0.5% |
| Throughput | 464 RPM | ≥1,000 RPM |

**Acceptance Criteria:** ≤50 evaluator calls/txn; p50 ≤300ms; p95 ≤500ms; SKU-criteria-free promotions unaffected; feature-flag controlled.

---

## FIX 1 — New `ItemBasedShortlistingService` (Primary)

*(Unchanged from Rev 3 — arbiter `VERDICT: PASS`.)*

### Fully-Recursive `extractSkuInCriteria`

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
        ComplexCondition complex = (ComplexCondition) condition;
        List<ProductSelectionCriteria> result = new ArrayList<>();
        result.addAll(extractSkuInCriteria(complex.getCondition1()));
        result.addAll(extractSkuInCriteria(complex.getCondition2()));
        return result;
    }
    if (condition instanceof ConditionWithPerUnitCondition) {
        ConditionWithPerUnitCondition cwpuc = (ConditionWithPerUnitCondition) condition;
        List<ProductSelectionCriteria> result = new ArrayList<>(extractSkuInCriteria(cwpuc.getCondition()));
        if (cwpuc.getProductBasedCondition() instanceof Condition) {
            result.addAll(extractSkuInCriteria((Condition) cwpuc.getProductBasedCondition()));
        }
        return result;
    }
    if (condition instanceof ConditionWithRewardCondition) {
        ConditionWithRewardCondition cwrc = (ConditionWithRewardCondition) condition;
        List<ProductSelectionCriteria> result = new ArrayList<>(extractSkuInCriteria(cwrc.getCondition()));
        if (cwrc.getRewardCondition() instanceof Condition) {
            result.addAll(extractSkuInCriteria((Condition) cwrc.getRewardCondition()));
        }
        return result;
    }
    return Collections.emptyList();
}
```

**Why NOT `ComplexCondition.getProductConditions()`:** its private helper only pattern-matches `ProductCondition`/`ComplexCondition`, silently dropping `ConditionWithPerUnitCondition`/`ConditionWithRewardCondition` children. Inline recursion over `getCondition1()`/`getCondition2()` fixes this at every tree level.

### Recursion Safety & Condition-Type Coverage

**Recursion safety (no infinite loop / no stack overflow):**
- **No infinite loop:** every recursive call strictly descends into a *child* node of the condition tree (`condition1`/`condition2`/`getCondition()`/`getProductBasedCondition()`/`getRewardCondition()`). The `Condition` objects form a pure tree — a child can never reference its own ancestor (they are built bottom-up from leaf conditions) — so there is no cycle and the recursion always terminates.
- **No stack overflow in practice:** recursion depth equals the condition tree's depth. Real promotion conditions are shallow — the deepest realistic case is a `ComboProductCondition` chain (a list of product conditions folded into a left-nested AND chain), and even a 20-product combo is ~20 frames deep. Java's default stack handles thousands of frames; a tree would need to be hundreds of levels deep to risk overflow, which no real promotion config produces (and the API cannot even construct a raw `ComplexCondition` — the request-side `ConditionType` enum has no `COMPLEX` value and the `@EnumNamePattern` regex excludes it).
- **Known limitation (accepted, not a bug):** the recursion is not explicitly depth-bounded in code. A hand-constructed pathological 10,000-deep tree in a test could overflow, but that is not reachable from any real promotion configuration. A depth guard could be added as belt-and-suspenders if desired; it was deliberately omitted to keep the logic simple.

**Condition-type coverage — all 7 concrete `Condition` implementations are handled:**

| # | Condition type | Handled? | Why |
|---|---|---|---|
| 1 | `ProductCondition` | ✅ (leaf) | The leaf that carries SKU criteria — the whole point of the filter. Extracts its `productSelectionCriteriaList`. |
| 2 | `ComplexCondition` | ✅ (tree node) | Has `condition1`/`condition2` children — must recurse to reach SKU leaves. **Also catches its 3 subclasses** (`ComboProductCondition`, `PaymentModeComboCondition`, `PaymentModeScopeCondition`) via `instanceof` subclass-matching. |
| 3 | `ConditionWithPerUnitCondition` | ✅ (wrapper) | Holds a nested `condition` + a `productBasedCondition` — both can contain SKU criteria. Must recurse into both, or SKU criteria inside them would be silently missed. |
| 4 | `ConditionWithRewardCondition` | ✅ (wrapper) | Holds a buy-side `condition` + a reward-side `rewardCondition`. Must recurse into both — this is the **accumulation-promotion safety mechanism** (reward SKU must be in cart even if buy is historical). |
| 5 | `TenderCondition` | ✅ (wrapper) | Holds a nested `condition` (can be a `ProductCondition`). Must recurse, or tender-scoped SKU promos ("10% off SKU Y when paid by card X") would be silently unfiltered. |
| 6 | `CartCondition` | ❌ (correct terminal) | A leaf with NO product/SKU criteria — cart-level KPI (amount/qty threshold). Nothing to extract, no nested condition. Correctly returns empty. |
| 7 | `PaymentModeCondition` | ❌ (correct terminal) | A leaf with NO product/SKU criteria — payment-mode-only. Nothing to extract, no nested condition. Correctly returns empty. |

**Summary:** the 5 explicitly-handled types are the ones that can *contain* SKU criteria (directly as a leaf, or by wrapping/recurring into children that do). The 2 "unhandled" types (`CartCondition`, `PaymentModeCondition`) are pure leaves with no SKU criteria and no nested condition — they correctly fall through to `return Collections.emptyList()`. The 3 `ComplexCondition` subclasses are not separate unhandled types — they are caught by the single `instanceof ComplexCondition` branch via Java subclass-matching. **Effective coverage: all 7 concrete types are handled** — 5 explicitly, 2 as correct empty-leaf terminals, and 3 subclasses folded into the `ComplexCondition` branch.

### Filter Logic
```java
private boolean shouldExclude(Promotion promotion, Set<String> cartSkuSet) {
    List<ProductSelectionCriteria> skuInCriteria = extractSkuInCriteria(promotion.getPromotionMeta().getCondition());
    if (skuInCriteria.isEmpty()) return false;
    return skuInCriteria.stream().noneMatch(c -> !Collections.disjoint(c.getValues(), cartSkuSet));
}
```
Conservative AND/OR posture (never false-negative). NOT_IN skipped. Case-sensitive, matching `OperatorEvaluationUtil.evaluate()`.

### Cart SKU Set, Registration, Feature Flag
`cartSkuSet` built from `cartRO.getCartItems().stream().map(CartItemRO::getSku)`, set on `PromotionContext` before shortlisting. Registered via explicit `@Qualifier` injection in `ShortListingServiceProvider`, positioned before `PromotionCappingShortlistingService` (no `@Order` — no ordering contract on `@Autowired List`). Feature-flagged via `PromotionOrgConfiguration.itemBasedShortlistingEnabled` (default false).

### New/Modified Files
**NEW:** `ItemBasedShortlistingService.java`, `ItemBasedShortlistingServiceTest.java`. **MODIFY:** `PromotionContext.java`, `PromotionRedemptionFacade.java:248-252`, `ShortListingServiceProvider.java`, `PromotionOrgConfiguration.java`.

---

### Post-Rev-3 Review Findings (Fix 1) — all CONFIRM the design is correct, ZERO code changes

**Finding 1 — Accumulation promotions:**
Investigated whether accumulation-enabled promos (est. 70-80% of the target 2K POS promotions), whose "buy" condition may be satisfied by historical purchases rather than the current cart, could be false-negative excluded.

- **Verified (load-bearing fact):** the reward/benefit condition (`ConditionWithRewardCondition.getRewardCondition()`) is evaluated via the same cart-filter dispatch as the buy condition (`ConditionEvaluatorFactoryService.java:530-536`, confirmed) — the reward SKU must already be in the current cart for the promotion to apply.
- **Supporting observation (narrower scope than initially stated):** within the conditions/evaluator code path examined (`promoeval/` package), no cart-item-insertion logic was found for `FREE_PRODUCT` rewards. Reward-application/redemption code outside this path was not exhaustively searched, so this is a supporting observation, not an independently exhaustive proof — the LOAD-BEARING conclusion rests on the verified evaluator dispatch fact above, which does not depend on this observation.
- **Conclusion:** the symmetric extraction already in `extractSkuInCriteria` — recursing into BOTH `getCondition()` (buy) AND `getRewardCondition()` (reward) — is the correct mechanism. For an accumulation promo where the buy-SKU is historical, reward-SKU extraction still correctly signals "this SKU must be in the cart," regardless of accumulation status.
- **Rejected alternative (session-internal decision, not present in any reviewed baseline artifact):** during design iteration, a blanket `accumulationConfig.enabled → pass through unconditionally` guard was drafted and then rejected before being incorporated into any reviewed design revision — it would have defeated ~70-80% of Fix 1's benefit. The symmetric extraction (above) handles this correctly without such a guard. This alternative was never part of the Rev 3 or Rev 8 baselines; it is recorded here purely as design-history context.
- **Remaining edge case (safely handled, no new code):** an accumulation promo where BOTH buy and reward use CATEGORY-based (non-SKU) conditions returns empty from `extractSkuInCriteria` → passes through via the existing default.

**Finding 2 — Caching `extractSkuInCriteria` per promotion:**
- **Verified:** tree-walk cost is negligible (~1,800 `instanceof` checks total per request for 300 promotions), vs. the 552 MongoDB evaluator calls being eliminated (orders of magnitude more expensive). No `@Version` field exists on `PromotionMeta` (only a weak `lastUpdatedOn`); `PromotionMetaCacheService` has no derived-value extension point.
- **Conclusion: premature optimization — explicitly rejected.** Per-request recomputation stands as designed.

**Finding 3 — Category-based promotion shortlisting:**
- **Verified:** `categoryList`/`brandList`/`attributePairs` exist on `CartItemUnrolled`, but are only populated at shortlisting time if the client sends `categoryHierarchySentInPayload=true`; otherwise fetched from an external `ProductService` call AFTER shortlisting runs. Categories are multi-valued and hierarchical — a promotion targeting a parent category could be false-negative excluded if only leaf categories are sent.
- **Conclusion: NOT implemented in this PR** — real ordering/hierarchy risk for likely modest gain (test-fixture proxy: CATEGORY ~10%, BRAND ~6%, ATTRIBUTE ~4% vs SKU ~80% — not production-confirmed). Existing safe default (pass-through for non-SKU entities) is already correct, just not optimized for that slice. **Flagged as a follow-up**, gated on confirming real-client payload behavior and production category-promotion fraction.

---

## FIX 2 — Parallel Capping KPI Queries (Secondary)

*(Unchanged from Rev 3 — arbiter `VERDICT: PASS`.)*

### Problem
`PromotionCappingShortlistingService.shortList()` calls `getKPISumAfterDateFor()` sequentially per promotion. APM: 935ms/call vs 24ms actual MongoDB time — connection pool saturation. Fix 2 parallelizes the ~50 calls remaining after Fix 1.

### `PromotionContext` Copy Contract
Mutable fields (`promotionLogs`, `codeBasedEvaluationLogs`, `promotionMetaLevelCapping`, `restrictionMetrics`) get fresh empty instances per parallel task. `earnedPromotionId`/`currentPromoCode`/`triggerDate` left blank, set per task. Post-join merge: `addAll` for logs, `putAll`/`merge(Integer::sum)` for maps.

### Async Pattern (first in this service)

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

- Executor: `@Bean(name="cappingExecutor", destroyMethod="shutdown")` — core=4, max=8, queue=50, `CallerRunsPolicy`
- MDC propagation is required — `OrgMongoDBFactory` shard routing depends on MDC; a `ThreadPoolTaskExecutor` thread does not inherit it automatically
- `exceptionally()` handles a FAILED future (isolates one bad promotion from the batch) — this is the mechanism specified in Rev 3
- **Clarification added at this consolidation (not present in the Rev 3 baseline as a concrete snippet):** for a genuinely SLOW (not failed) future, `CompletableFuture.allOf(futures).get(2000, TimeUnit.MILLISECONDS)` with a `TimeoutException` catch that falls back to the existing sequential loop is the intended safety net, consistent with Rev 3's prose ("Lock TTL... 900s >> ~200ms expected parallel time" implies a bounded expectation) but the exact timeout code was not spelled out as a snippet in Rev 3. This should be implemented as specified in the original Rev 3 prose during Phase C.
- Lock TTL confirmed safe: `DEFAULT_UNLOCK_TTL=900s` >> ~200ms expected parallel time

### Metrics / Observability (Fix 2) — restored from Rev 3 baseline

| Metric | Where | Captures |
|---|---|---|
| `promotions.sku.shortlist.in` | `ItemBasedShortlistingService` before loop | Input set size to SKU filter |
| `promotions.sku.shortlist.out` | After loop | Output size — primary fix validation metric |
| `promotions.sku.shortlist.excluded` | After loop | Drop count |
| `capping.kpi.parallel.ms` | `PromotionCappingShortlistingService` | Parallel block wall time |
| `capping.kpi.timeout.fallback` | Timeout branch | Alert if > 0 |
| `capping.kpi.task.failures` | `exceptionally` branches | Per-task failure count |

Monitor `capping-kpi-`/`capping-shortlist-` thread pool (`QueueSize`, `ActiveCount`, `PoolSize`) via Spring Boot Actuator. Alert if `QueueSize` approaches 50 under normal load. Existing `metricsService.addListSizeAsAttribute("promotions.shortlisted", ...)` retained as the end-state metric.

### Modified Files
`PromotionCappingShortlistingService.java` (sequential→parallel), new `CappingAsyncConfig.java` (executor bean).

### Expected Impact (Fix 1+2)
| Metric | Before | After Fix 1+2 |
|--------|--------|---------------|
| Evaluator calls/txn | 552 | ~50 |
| Capping time/txn | ~280s (300 seq) | ~1s (parallel, 50) |
| p50 e2e | 9.09s | **≤300ms** |
| p95 e2e | 23.8s | **≤500ms** |

---

## FIX 3 — Extract `PromotionMetaCustomDao` (Secondary)

*(Finalized per Rev 8, post senior-review refinements — unchanged in this consolidation.)*

### Root Cause
`PromotionMetaManagementService.getActivePromotionByType()` (`:356-357`) self-invokes `getActivePromotionIdsByType()` (`:376`, `@Cacheable`) — bare `this.` call within the same `@Service` bean. Default CGLIB proxy AOP (no AspectJ weaving) — self-invocation bypasses the proxy, `@Cacheable` never fires on the production hot path. The APM "62% miss rate" is misleading — on the real hot path caching is a complete no-op (0% effective).

### Option B — DAO Extraction, Full Scope (all 4 sibling methods)

| # | Method | Line | Caller | Bug today? |
|---|--------|------|--------|-------------|
| 1 | `getActivePromotionIdsByType` | `:376` | `getActivePromotionByType():357` (same bean) | **YES — the bug** |
| 2 | `getActivePromotionsForAnOrg` | `:397` | `PromotionRedemptionFacade:243` (cross-bean) | No — already works |
| 3 | `getEarningPromotionIdsForGeneral` | `:247` | `EarnedPromotionMetaProcessor:66` (cross-bean) | No — already works |
| 4 | `getPosPromotionIdsWithSupplementaryCriteria` | `:283` | `SupplementaryProgramProcessor:87` (cross-bean) | No — already works |

```java
@Service
@Profiled
@Slf4j
public class PromotionMetaCustomDao {
    @Autowired
    private MongoTemplate mongoTemplate;  // matches source field type — minimizes diff on verbatim move
    private final QPromotionMeta qPromotionMeta = QPromotionMeta.promotionMeta;
    // 4 methods moved verbatim with @Cacheable annotations
}
```

### Caller Changes

| Caller | Change |
|--------|--------|
| `PromotionMetaManagementService` | Inject DAO; repoint `:357`. `@CacheEvict` on `save()`/`update()` STAYS (name+key eviction, single shared `RedisCacheManager`) |
| `PromotionRedemptionFacade` | Inject DAO; KEEPS `PromotionMetaManagementService` (2 other live calls) |
| `EarnedPromotionMetaProcessor` | Inject DAO; DROPS `PromotionMetaManagementService` (grep-confirmed sole use) |
| `SupplementaryProgramProcessor` | Inject DAO; DROPS `PromotionMetaManagementService` (grep-confirmed sole use) |

### Mandatory Regression Test
Spring-context test (not Mockito-only) verifying a repeat call is served from cache; dedicated/randomized orgId per test to avoid Testcontainers-Redis cross-test flakiness.

### New/Modified Files
**NEW:** `PromotionMetaCustomDao.java`, `PromotionMetaCustomDaoTest.java`. **MODIFY:** `PromotionMetaManagementService.java`, `PromotionRedemptionFacade.java`, `EarnedPromotionMetaProcessor.java`, `SupplementaryProgramProcessor.java` + 4 test files.

### Priority
Secondary — NOT required for the SLO (Fix 1+2 alone deliver it). Genuine correctness fix + architectural cleanup, low-risk (mechanical verbatim-body move).

---

## Criticality Summary

| Fix | Risk | Reason |
|-----|------|--------|
| Fix 1 | Medium | Changes shortlisted set; feature-flagged; validated against accumulation + category edge cases |
| Fix 2 | Medium | First async pattern; context copy + MDC propagation correctness critical |
| Fix 3 | Low-Medium | Mechanical DAO extraction; genuine bug fix; verbatim body moves |

## Test Strategy Summary

**Fix 1:** ~17 unit tests (all 10 Condition types, accumulation reward-SKU scenarios, NOT_IN skip, null/empty guards, feature flag, provider ordering).
**Fix 2:** parallel execution, timeout/fallback (per the Rev 3 prose + this consolidation's clarification), context isolation, MDC propagation, pool saturation.
**Fix 3:** 4 migrated unit tests + mandatory Spring-context cache-firing test (isolated orgId) + eviction tests + repointed-caller tests.
**Integration/Regression:** staging smoke (1,000 POS promotions → ≤50 evaluator calls/txn), load test (1,000 RPM → p50≤300ms/p95≤500ms), flag-off regression, accumulation-promotion regression.

## Post-Fix Projected Metrics

| Metric | Current | After Fix 1+2 | After Fix 1+2+3 | SLO |
|--------|---------|----------------|-------------------|-----|
| Evaluator calls/txn | 552 | ~50 | ~50 | ≤50 |
| p50 e2e | 9.09s | **≤300ms** | ≤300ms | ≤300ms |
| p95 e2e | 23.8s | **≤500ms** | ≤500ms | ≤500ms |
| `getActivePromotionIdsByType` cache-hit rate | ~0% | ~0% | Near-100% on repeat orgId+type within TTL | N/A |

## Open Questions

1. NOT_IN-only SKU promotions in Pharma org — confirm profile breakdown.
2. Accumulation promo SKU-vs-CATEGORY reward mix — confirm via manual MongoDB query.
3. Category-based shortlisting — follow-up candidate, gated on client payload behavior + production data confirmation.
4. `ShortListingServiceProvider` double-injection guard — unit test for proxy unwrapping.
5. Lock TTL — re-evaluate only if any org's custom `unlockTtl` drops below ~5s.
6. `cartSkuSet` invariant — document: empty set → pass all promotions through.
