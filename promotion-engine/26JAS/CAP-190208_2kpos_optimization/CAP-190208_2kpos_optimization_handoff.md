# Arch-Investigator Handoff: CAP-190208 — 2K POS Promotion Optimization

**Ticket:** CAP-190208  
**Investigated:** 2026-07-02  
**Investigator:** Saravanan Kesavan  
**Handoff to:** Tech-Detailer  
**Confidence:** HIGH — root cause confirmed with APM data + code trace + domain context

---

## 1. Problem in One Sentence

`POST /v1/promotions/evaluate` evaluates all 300 shortlisted POS promotions sequentially for every cart request, including ~298 promotions whose required SKUs are not present in the cart, burning 552 evaluator calls per transaction and producing 9.5s average latency at 464 RPM.

---

## 2. SLO Target

| Metric | Current (staging, 1K POS) | Target (2K POS, 1K RPM) |
|--------|--------------------------|--------------------------|
| p50 latency | 9.09s | ≤ 300ms |
| p95 latency | 23.8s | ≤ 500ms |
| Error rate | 27.29% | ≤ 0.5% |
| Throughput | 464 RPM (at capacity) | 1,000 RPM sustained |

---

## 3. Confirmed Root Cause

### 3.1 Evaluation funnel

```
1,000 POS promotions (org)
  → [8 shortlisting services]          — 700 filtered, 300 survive
  → [evaluation loop, sequential]      — 552 PromotionEvaluatorService.apply() calls
  → [saveAppliedPromotions]            — ~1–2 promotions actually applied to cart
```

**298 of 300 shortlisted promotions are evaluated and rejected.** Shortlisting is promotion-context-aware only (is the promotion running? is its budget available?). It is not cart-context-aware (does this cart contain the SKUs this promotion requires?). For a Pharma org where each POS promotion is tied to a specific SKU, cart-SKU shortlisting would eliminate ~98% of promotions before the evaluation loop.

### 3.2 Binary search multiplier

`ApplicationService.apply()` ([ApplicationService.java:175–207](../../../../../../promotion-engine/src/main/java/com/capillary/promotionengine/service/impl/ApplicationService.java)) contains a binary search that finds the maximum number of times a promotion can be applied to the cart. For the 20% of promotions that allow multiple redemptions, this calls `PromotionEvaluatorService.apply()` an average of ~5.2 times. For the 80% with `maxRedemptions = 1`, `canMultiplyPromotion()` exits the loop immediately (1 evaluator call).

APM confirms: 552 evaluator calls / 300 shortlisted promotions = **1.84× average multiplier**.

For 2K POS: 600 shortlisted × 1.84 = **1,104 sequential evaluator calls per transaction at 2.9ms each = 3.2s in the evaluator alone** — impossible to reach 300ms SLO.

### 3.3 Cascade: DB connection pool saturation

At 464 RPM with 3.73s average in sequential evaluation, ~73 requests are in-flight at any moment. The MongoDB connection pool exhausts. Fast queries (24ms MongoDB aggregate) wait 900ms+ for a connection. This explains two APM anomalies:

- `getKPISumAfterDateFor` (Java wrapper): 935ms/call vs 24ms actual MongoDB time → **911ms is connection pool wait**
- `getActivePromotionIdsByType`: 1.72s/call → same pool contention, compounded by 62% cache miss falling back to MongoDB

### 3.4 Why capping shortlisting is not the DB bottleneck

`PromotionCappingShortlistingService.shortList()` ([PromotionCappingShortlistingService.java:36–46](../../../../../../promotion-engine/src/main/java/com/capillary/promotionengine/service/impl/shortlist/PromotionCappingShortlistingService.java)) guards DB calls:

```java
if (hasPromotionRestriction || hasFixedPromotionRestriction) {
    // DB call only here
} else {
    list.add(promotion);  // auto-pass, no DB call
}
```

Most of the 300 shortlisted promotions have no capping restrictions → auto-pass → explains the low 2.31 DB calls/txn for restriction aggregates.

---

## 4. Domain Context

The first org to hit this limit is a **Pharma brand** with the following promotion patterns:

- **Per-SKU promotions:** One POS promotion per product SKU. A cart with 10 line items should match at most 10 promotions — not 300.
- **Combo promotions:** Some promotions require two specific SKUs to both be present (SKU_A + SKU_B). These use `ComboProductCondition`.
- **Scale target:** Pharma orgs may have 2,000+ active POS promotions (one per drug/SKU in their formulary).

---

## 5. Condition Model — Fully Structured, No Evaluator Needed

`PromotionMeta.condition` ([PromotionMeta.java:78](../../../../../../promotion-engine/src/main/java/com/capillary/promotionengine/bo/PromotionMeta.java)) is a polymorphic field that carries all condition data needed for cart-context pre-filtering:

### 5.1 Single-SKU promotion

```
ProductCondition
  └─ productSelectionCriteriaList: [
       ProductSelectionCriteria(
         productEntity: SKU,
         operator: IN,
         values: {"SKU_123"}
       )
     ]
```

**Shortlisting check:** `disjoint(cartSkuIds, criteria.values)` → if true, promotion cannot apply.

### 5.2 Combo promotion (SKU_A AND SKU_B)

```
ComboProductCondition
  ├─ condition1: ProductCondition(SKU IN {SKU_A})
  └─ condition2: ProductCondition(SKU IN {SKU_B})
```

`ComboProductCondition.getProductConditions()` flattens nested trees into a flat `List<ProductCondition>`.

**Shortlisting check:** ALL leaf `ProductCondition`s must be satisfiable → if ANY leaf's required SKUs are absent from the cart, promotion cannot apply.

### 5.3 Category-level promotion

```
ProductCondition
  └─ productSelectionCriteriaList: [
       ProductSelectionCriteria(productEntity: CATEGORY, operator: IN, values: {"CAT_X"})
     ]
```

**Shortlisting check:** `disjoint(cartCategoryIds, criteria.values)` → if true, filter out.

### 5.4 Cart-value promotion

```
CartCondition(kpi: SUBTOTAL, operator: GREATER_THAN_OR_EQUAL, value: 1000)
```

**Shortlisting check:** `cart.total < 1000` → filter out.

### 5.5 Conservative pass-through (do NOT filter)

The following must pass through without filtering — let the evaluator decide:
- `ProductSelectionCriteria.operator = NOT_IN` (exclusion logic is not safe to pre-filter)
- `productEntity = BRAND` or `ATTRIBUTE` (not reliably available on cart line items at shortlisting time)
- `TenderCondition`, `PaymentModeCondition`, and variants (no cart-level tender info at shortlisting time)
- `ConditionType.COMPLEX` with `LogicalOperator.OR` (either branch might apply — cannot pre-reject)
- `null` condition (no condition = always applicable)

### 5.6 Relevant classes

| Class | File | Purpose |
|-------|------|---------|
| `PromotionMeta` | `bo/PromotionMeta.java:78` | Holds `Condition condition` |
| `Condition` | `bo/conditions/Condition.java` | Interface: `getConditionType()`, `hasCartAttributes()` |
| `ConditionType` | `bo/conditions/ConditionType.java` | `CART`, `PRODUCT`, `COMBO_PRODUCT`, `COMPLEX`, `TENDER`, `PAYMENT_MODE`, … |
| `ProductCondition` | `bo/conditions/ProductCondition.java:22` | `List<ProductSelectionCriteria> productSelectionCriteriaList` |
| `ProductSelectionCriteria` | `bo/conditions/ProductSelectionCriteria.java` | `productEntity` (SKU/CATEGORY/BRAND/ATTRIBUTE), `operator` (IN/NOT_IN), `values` (Set\<String\>) |
| `ComboProductCondition` | `bo/conditions/ComboProductCondition.java` | `getProductConditions()` flattens nested AND-tree |
| `ComplexCondition` | `bo/conditions/ComplexCondition.java` | `condition1`, `condition2`, `LogicalOperator` (AND/OR) |
| `CartCondition` | `bo/conditions/CartCondition.java` | `kpi` (SUBTOTAL/ITEMCOUNT), `operator`, `value` |

---

## 6. Current Shortlisting Services

Eight `ShortListingService` implementations, injected via Spring `List<ShortListingService>` with **no explicit `@Order`** ([ShortListingServiceProvider.java:21](../../../../../../promotion-engine/src/main/java/com/capillary/promotionengine/service/impl/shortlist/ShortListingServiceProvider.java)):

| Service | Type | DB call? | Notes |
|---------|------|----------|-------|
| `PromotionValidityShortlistServiceImpl` | Loyal + Anon | No | Checks running status in-memory |
| `RedeemablePromotionShortListService` | Loyal + Anon | No | Checks redeemability date range |
| `StoreShortListServiceImpl` | Loyal + Anon | No | Checks `storeBasedCriteria` in-memory |
| `TimeCriteriaShortlistServiceImpl` | Loyal + Anon | No | Checks `timeBasedCriteria` in-memory |
| `PromotionCappingShortlistingService` | Loyal + Anon | Conditional | DB only if `hasPromotionRestriction` |
| `SupplementaryCriteriaShortListServiceImpl` | Loyal only | Yes | Tier/subscription check |
| `CustomerPreferenceShortlistServiceImpl` | Loyal only | Yes | Customer preference check |
| `AnonymousUserPromotionShortListService` | Anon only | No | Filters out loyalty-only promotions |

The order of execution is Spring's component-scan order — **undefined and not guaranteed**.

---

## 7. Proposed Solution — Three Changes

### Change 1 (P0): New `CartItemSkuShortlistServiceImpl`

**A new shortlisting service that checks cart-context conditions against the structured `PromotionMeta.condition` before the evaluation loop.**

Pseudocode:

```
CartItemSkuShortlistServiceImpl.shortList(promotions, promotionContext):
  cartSkuIds      = extract all SKU IDs from cart line items
  cartCategoryIds = extract all category IDs from cart line items
  cartTotal       = cart subtotal

  for each promotion:
    condition = promotion.getPromotionMeta().getCondition()

    match condition type:

      PRODUCT (operator=IN, entity=SKU):
        if disjoint(cartSkuIds, condition.requiredSkuIds) → REMOVE

      PRODUCT (operator=IN, entity=CATEGORY):
        if disjoint(cartCategoryIds, condition.requiredCategoryIds) → REMOVE

      COMBO_PRODUCT:
        leafConditions = ComboProductCondition.getProductConditions()
        for each leaf (ProductCondition):
          apply same SKU/CATEGORY check as above
          if ANY leaf fails → REMOVE (AND semantics)

      CART (kpi=SUBTOTAL):
        if cartTotal < condition.minimumValue → REMOVE

      PRODUCT (operator=NOT_IN), BRAND, ATTRIBUTE, TENDER, PAYMENT_*, COMPLEX(OR), null:
        → KEEP (conservative pass-through)
```

**Correctness invariant:** The filter is strictly one-directional — it only removes promotions where the required items are provably absent from the cart. It never removes a promotion that could legitimately apply. When in doubt, keep.

**Placement:** Must run **first** in the shortlisting pipeline, before all other shortlisters. Assign `@Order(1)`.

**No new DB calls:** All data comes from `PromotionMeta.condition` (already in-memory from the promotion meta cache) and the cart in `PromotionContext`.

**Expected impact (Pharma org, cart with 10 SKUs, 1,000 POS per-SKU promotions):**

```
1,000 promotions
  → CartItemSkuShortlistServiceImpl   →  ~10 survive (one per cart SKU)
  → remaining 7 shortlisters          →  ~8–10 survive
  → evaluation loop                   →  8 × 1.84 = ~15 evaluator calls
  → total evaluation time             →  ~43ms  (vs 3.73s today)
```

At 2K POS: ~20 shortlisted → 37 evaluator calls × 2.9ms = **107ms**. SLO is met.

**Future optimization (beyond this ticket):** At cache-warm time, build an inverted index `Map<SkuId, Set<PromotionId>>` inside `PromotionMetaCacheService`. At request time, the shortlister becomes O(cartLines) instead of O(promotions): union `skuIndexMap.get(sku)` for each cart SKU. This is the Talon.One-equivalent structural optimization — see §13 for full context. Not in scope here; in-memory scan of 1K–2K conditions is fast enough (~1–2ms). Becomes necessary at 10K+ promotions.

### Change 2 (P1): Formalize shortlisting order with `@Order`

Add `@Order` annotations to all shortlisting services to guarantee execution sequence:

| Order | Service | Rationale |
|-------|---------|-----------|
| 1 | `CartItemSkuShortlistServiceImpl` (NEW) | Most effective filter; zero DB cost |
| 2 | `PromotionValidityShortlistServiceImpl` | Fast in-memory status check |
| 3 | `RedeemablePromotionShortListService` | Fast date range check |
| 4 | `TimeCriteriaShortlistServiceImpl` | Fast time window check |
| 5 | `StoreShortListServiceImpl` | Fast store criteria check |
| 6 | `AnonymousUserPromotionShortListService` | Anon-only; fast |
| 7 | `SupplementaryCriteriaShortListServiceImpl` | Loyal-only; may have DB |
| 8 | `CustomerPreferenceShortlistServiceImpl` | Loyal-only; may have DB |
| 9 | `PromotionCappingShortlistingService` | DB-touching; must run last on smallest list |

**Why this matters:** Without explicit ordering, a future Spring upgrade or bean registration change could silently reorder these services, causing `PromotionCappingShortlistingService` (DB-touching) to run on 1,000 promotions instead of the 10 that survive SKU shortlisting.

### Change 3 (P2): Fix `getActivePromotionIdsByType` cache miss rate

APM shows `PromotionMetaManagementService.getActivePromotionIdsByType()` is called in 62% of transactions with a 1.72s per-call cost (vs sub-100ms when served from cache). This is a separate but significant bottleneck (11% of transaction time).

**Investigation needed:** Determine why `PromotionMetaCacheService` is missing 62% of the time for this org:
- Is the cache TTL too short relative to the cache load interval?
- Is the promotion meta cache being evicted under write pressure (new/updated promotions)?
- Is the cache key structure causing unnecessary misses (e.g., per-org vs per-org-per-type partitioning)?

**Scope:** This is an independent investigation. It does not block Change 1 or Change 2.

---

## 8. Open Questions for Tech-Detailer

These must be answered before writing the final low-level design:

1. **`MAX_ACTIVE_POS_PROMOTION = 300` in `PromotionEngineConstants.java:53`**
   - Where is this constant applied? Is it enforced as a hard cap on the shortlist output?
   - If yes: does the cap run before or after the new SKU shortlister? If before, the SKU shortlister has no effect when there are ≥ 300 promotions.
   - Resolution: the cap must either be removed (if it was a workaround for this exact performance problem) or repositioned to run after the SKU shortlister.

2. **`CONDITION_WITH_REWARD_CONDITION` and `CONDITION_WITH_PER_UNIT_CONDITION` types**
   - The `ConditionType` enum includes these compound types. Do they wrap a `ProductCondition` as an inner condition? If so, can the inner product condition be extracted for SKU shortlisting?
   - If yes: extend `CartItemSkuShortlistServiceImpl` to unwrap these. If no: conservative pass-through.

3. **Cart line item structure at shortlisting time**
   - Does `PromotionContext.getCart()` at the point where shortlisting runs carry SKU IDs and category IDs for each line item?
   - Specifically: are product details loaded before or after shortlisting? (See `PromotionRedemptionFacade.evaluateCart()` around line 243: "All active promotions loaded into cache" — is product enrichment in the same pre-shortlisting phase?)
   - If cart line items do not carry category IDs at shortlisting time: the category filter must be skipped (conservative pass-through).

4. **`ComboProductCondition` nesting for 3+ SKU combos**
   - Confirm that `getProductConditions()` correctly flattens arbitrary-depth nesting (e.g., `Combo(A, Combo(B, C))`).
   - If it only handles 2-level nesting, a recursive extractor is needed.

5. **Stacking semantics — read-only evaluation at shortlisting time**
   - The evaluation loop is sequential because each promotion sees the cart-after-previous-promotions (stacking). Confirm that shortlisting operates on the original unmodified cart (not the stacking-mutated cart). If shortlisting sees the unmodified cart, the SKU filter is always safe. If shortlisting somehow sees a partially-stacked cart state, the filter logic needs to account for items already consumed by prior promotions.

---

## 9. Key File Inventory

| File | Relevance |
|------|-----------|
| `service/impl/PromotionRedemptionFacade.java:196–282` | Main `evaluateCart()` flow; shortlisting and evaluation entry |
| `service/impl/PromotionRedemptionFacade.java:336–348` | `getShortListedPromotions()` — loops all shortlisters |
| `service/impl/shortlist/ShortListingServiceProvider.java` | Spring-injected service list; no ordering today |
| `service/impl/shortlist/PromotionCappingShortlistingService.java:36–46` | Capping shortlist; DB guard already present |
| `service/impl/ApplicationService.java:50–232` | Evaluation loop + binary search |
| `service/impl/ApplicationService.java:175–207` | Binary search loop; `canMultiplyPromotion()` check |
| `promoeval/PromotionEvaluatorService.java:54–74` | Full condition + discount evaluation per promotion |
| `bo/PromotionMeta.java:62,78` | `PromotionType` field; `Condition condition` field |
| `bo/conditions/Condition.java` | Condition interface |
| `bo/conditions/ConditionType.java` | Condition type enum |
| `bo/conditions/ProductCondition.java:22` | `List<ProductSelectionCriteria>` |
| `bo/conditions/ProductSelectionCriteria.java` | `productEntity`, `operator`, `values` (Set\<String\>) |
| `bo/conditions/ComboProductCondition.java` | `getProductConditions()` — flattens AND-tree |
| `bo/conditions/ComplexCondition.java` | `condition1`, `condition2`, `LogicalOperator` |
| `bo/conditions/CartCondition.java` | `kpi`, `operator`, `value` |
| `PromotionEngineConstants.java:53` | `MAX_ACTIVE_POS_PROMOTION = 300` — **must investigate** |
| `service/impl/PromotionMetaManagementService.java` | `getActivePromotionIdsByType()` — cache miss path |

---

## 10. Out of Scope for This Ticket

| Area | Reason |
|------|--------|
| Parallelizing the evaluation loop | Stacking semantics require sequential evaluation; correctness analysis is a separate workstream |
| Binary search optimization (20% multiplied promotions) | Already guarded by `canMultiplyPromotion()`; not the primary bottleneck |
| MongoDB connection pool sizing | Ops concern; resolves automatically once evaluator call volume drops |
| Changing the binary search upper bound (`Integer.MAX_VALUE`) | Separate correctness/cap topic; not related to this latency fix |
| `PromotionCappingShortlistingService` batch DB fetch | 2.31 DB calls/txn already minimal; not a bottleneck at current or projected scale |

---

## 11. Acceptance Criteria

The implementation is complete when ALL of the following hold in a load test with 1,000 POS promotions (Pharma pattern: per-SKU + combo) at 1,000 RPM:

- [ ] p50 latency ≤ 300ms
- [ ] p95 latency ≤ 500ms
- [ ] Error rate ≤ 0.5%
- [ ] `ApplicationService/apply` avg calls/txn ≤ 30 (down from 552)
- [ ] `PromotionEvaluatorService/apply` avg calls/txn ≤ 30 (down from 551)
- [ ] Discount amounts on cart are identical to pre-fix results (no regression in applied promotions)
- [ ] All existing unit and integration tests pass

At 2,000 POS promotions:
- [ ] p50 latency ≤ 300ms at 1,000 RPM sustained

---

## 12. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| SKU shortlister incorrectly removes an applicable promotion | Low — filter is one-directional (conservative) | High — missed discount | Unit test every `ConditionType` variant with assertions that no eligible promotion is removed |
| `MAX_ACTIVE_POS_PROMOTION = 300` cap runs before new shortlister, negating its effect | Medium — location unknown | High — zero latency improvement | Tech-detailer must trace constant usage before implementation |
| Cart line items lack category IDs at shortlisting time | Medium — unknown without tracing `PromotionContext` loading | Low — category filter skipped, falls back to conservative | Verify in code; if absent, disable category pre-filter and document |
| `getProductConditions()` doesn't handle 3+ SKU nested combos | Low — Pharma combos are likely 2-SKU max | Medium — combo promotions not filtered | Write explicit test with depth-3 nesting |
| Spring `@Order` interaction with existing `@Primary` or `@Qualifier` on shortlisters | Low | Medium — ordering silently broken | Verify no existing ordering annotations conflict |

---

## 13. Industry Reference — Talon.One Architecture Learning

**Source:** Emma Sleep engineering blog, Feb 2026 — *"Behind the Build: How Emma Sleep overhauled its online checkout"* (Talon.One blog).

### The parallel problem

Emma Sleep's stated pain point before migrating to Talon.One:

> *"Inefficient evaluations. Due to global promotions, all promotion rules were evaluated for each cart."*

This is the same root cause as CAP-190208. Their fix was not algorithmic — it was architectural.

### How Talon.One achieves sub-100ms evaluation

**Primary mechanism: Application scoping.**

Each deployment context (market, channel, store type) is a separate Talon.One "Application". Campaigns belong to exactly one Application. Every evaluation API request targets one specific Application. The rule engine only evaluates that Application's campaigns — never the global pool.

Emma Sleep: 22 markets × ~45 campaigns/market = 45 evaluations per cart. Before: 1 global pool × ~1,000 campaigns = 1,000 evaluations per cart. **The 95% reduction came from structural scoping, not rule engine optimization.**

**Secondary mechanism: stateless, in-memory evaluation.**

The evaluation API call carries the full cart context. The rule engine evaluates entirely in-memory against the pre-loaded Application campaign set. No database lookups occur during the evaluation hot path.

### Architectural equivalence

| Talon.One | Our solution (this ticket) | Next architectural step |
|-----------|---------------------------|------------------------|
| Application scoping — structural, set at config time, zero per-request cost | `CartItemSkuShortlistServiceImpl` — runtime SKU intersection, O(N promotions) scan | Pre-built `Map<SkuId, Set<PromotionId>>` at cache-warm time → O(cartLines) per request |
| Campaigns indexed to an Application at creation time | Condition tree scanned per request | Inverted index built once when promotions load into `PromotionMetaCacheService` |
| In-memory rule evaluation, no DB hot path | In-memory evaluator (already true); shortlisting becomes in-memory after this fix | No change needed |

### Key learning for this ticket

The `CartItemSkuShortlistServiceImpl` is the runtime equivalent of Application scoping: it scopes the promotion set to cart-relevant promotions before evaluation. The architectural end-state (inverted SKU index) makes this scoping as cheap as Talon.One's Application lookup — a set membership check, not a linear scan.

**The principle is not "make the evaluator faster." It is "shrink the set that reaches the evaluator."** This ticket implements that principle at request time. The inverted index optimization (§7 Change 1 — Future Optimization) implements it at cache time.
