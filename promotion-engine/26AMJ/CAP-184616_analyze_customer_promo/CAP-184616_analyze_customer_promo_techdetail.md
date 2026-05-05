# Tech Detail: GetCustomerPromotions Latency Optimization — Analysis & Investigation
**Date:** 2026-05-05
**Author:** Saravanan Kesavan
**Status:** Draft — Analysis Story
**Jira:** CAP-184616
**Confidence:** MEDIUM (root cause identified; optimization path requires profiling to confirm split)

---

## 1. Problem Statement

`GET /promotions/customer/{customerId}` is breaching SLA at production load. Two
request profiles are affected:

| SLO | RPM | Page size | Current p95 | Target |
|-----|-----|-----------|-------------|--------|
| SLO 1 & 2 | ~900 RPM | ≤ 50 items | 260 ms | ≤ 150 ms |
| SLO 3 | ~225 RPM | 50–100 items | ~400 ms | < 150 ms |

Concurrent KPI query load at these RPMs: ~1k RPM total (customer-level: ~13.5k, earn-level: ~3.9k, promo-level: ~240). As orgs configure more promotions with more restriction levels, query count grows linearly, making SLA degradation a scalability problem, not just a tuning problem.

---

## 2. Confirmed Call Chain

```
GET /promotions/customer/{customerId}
  → PromotionEarningResource.getAllForCustomer()          [controller/PromotionEarningResource.java:90]
    → CustomerPromotionFacade.getAllPromotionsForCustomer() [service/impl/CustomerPromotionFacade.java:127]
      → [Processor pipeline — executed sequentially in order]
         1. EarnedPromotionProcessor          — MongoDB: find all valid earned promotions for customer
         2. IssuedPromotionProcessor          — MongoDB: find all issued/active promotions
         3. EarnedPromotionMetaProcessor      — PromotionMeta cache lookup per promotion
         4. SupplementaryProgramProcessor     — supplementary program membership lookup
         5. CustomerLinkedCodePromotionProcessor — code-linked promotions
         6. StoreFilterProcessor              — filter by store scope
         7. CustomerPreferenceStatusProcessor — preference filtering
         8. ModeFilterProcessor               — earn/redeem mode filter
         9. DateFilterTypeProcessor           — validity window filter
        10. PaginationProcessor               — sort + skip/limit (CUTS LIST HERE)
        11. RedemptionProcessor               — KPI queries (fires AFTER pagination)
          → PromotionRestrictionDetailsService.getRestrictionDetails()  [per promotion]
            → per restriction level (Customer, Promotion, Earn, Code, Cart):
                PersistedRestrictionKPISummaryHandler.getCurrentKpis()
                  → RestrictionLevelRedemptionSummaryCustomDao.getKPIDetails()  [1 query per KPI per level]
                  (up to 3 queries: DISCOUNT, TRANSACTION, REDEMPTION)
                PersistedRestrictionKPISummaryHandler.getCurrentKpisForFixedWindow()
                  → FixedWindowRedemptionSummaryCustomDao.getKPISumForDateRange()  [1 query per KPI per FW restriction]
```

**Key structural fact**: Pagination (step 10) runs before KPI queries (step 11). This bounds the KPI query count to the page size (50 or 100). The query explosion is within that page, not across the full customer promotion list.

---

## 3. Bottleneck Identification

### 3.1 KPI Query Explosion (Primary Bottleneck)

For a single request with page size N, restriction levels L per promotion, and K KPI types (max 3):

```
total queries = N × L × K    (for sliding-window restrictions)
              + N × L × FW   (for fixed-window restrictions per FW config)
```

Example at page 50, 3 levels, 3 KPIs: **450 MongoDB aggregations per request** — all sequential.

At 900 RPM this is **405,000 aggregation queries/min → 6,750 agg/sec** to MongoDB. The observed ~13.5k/min customer + ~3.9k/min earn + ~240/min promo correlates directly to the level-weighted distribution across orgs.

### 3.2 Index Coverage Gap on `RestrictionLevelRedemptionSummary`

Existing compound indexes:
```java
@CompoundIndex(name = "orgId_promotionId_level_customerId",
    def = "{'orgId':1, 'promotionId':1, 'level':1, 'customerId':1}")
@CompoundIndex(name = "org_promotion_id_promo_code_level",
    def = "{'orgId':1, 'promotionId':1, 'promoCode':1, 'level':1}")
```

The aggregation filter (in `getKPISumAfterDateFor`) includes:
- `orgId`, `promotionId`, `level`, `customerId` ✓ (covered by index 1)
- `redemptionKPI` ✗ (not in any index → post-index filter)
- `validTill > afterDate` ✗ (not in any index → post-index filter)
- `periodStart >= afterDate` ✗ (not in any index → post-index filter)

MongoDB hits the index, fetches all docs matching (orgId, promotionId, level, customerId), then applies the KPI and date predicates in-memory. For promotions with many redemptions, this causes full scan of the index bucket.

### 3.3 Sequential Execution Across Promotions and Levels

`RedemptionProcessor.processCustomerPromotions()` uses `.forEach()` → sequential.
`PromotionRestrictionDetailsService.getRestrictionDetails()` uses `.forEach()` on levels → sequential.

No parallelism at either level. Total latency ≈ sum of all agg query round trips.

### 3.4 Aggregation vs Bulk Fetch

Each aggregation is an independent MongoDB round trip. For (orgId, customerId, dateRange), all
queried promotions share the same customer/org context. A single aggregation grouping by
`promotionId + level + redemptionKPI` could replace N×L×K individual aggregations.

---

## 4. What to Analyze / Investigate

This section is the core of the analysis story. Each item is a discrete investigation
unit with a specific question to answer before choosing an optimization path.

### Investigation A — Query Volume Attribution

**Question**: Of the ~1k RPM of KPI queries, how are they distributed across orgs, restriction-level types, and page sizes? Is the explosion concentrated in a few heavy orgs (long tail optimization opportunity) or spread uniformly?

**How to investigate**:
- New Relic `QueryMetricsService` already emits `promotionId`, `level`, `duration` per query (see `addQueryMetrics()` in `PersistedRestrictionKPISummaryHandler:285`).
- Pull a distribution: queries per org, queries per restriction level, queries per promotion. Identify if P95 latency is driven by a small set of high-restriction-count orgs.
- Check `restriction.<level>` custom params added by `addMetricsForRestrictions()` in `CustomerPromotionFacade:155`.

**Expected output**: Heatmap of (org, level) → query count per request. Identifies whether the fix must be universal or targeted.

---

### Investigation B — Index Effectiveness for RestrictionLevelRedemptionSummary

**Question**: After the compound index hit on `{orgId, promotionId, level, customerId}`, how many documents does MongoDB scan per aggregation before applying the KPI and date predicates?

**How to investigate**:
- Run `explain("executionStats")` on representative `getKPISumAfterDateFor` queries in the dev/staging MongoDB.
- Check `executionStats.totalDocsExamined` vs `executionStats.nReturned`. High ratio means index is not selective enough.
- Determine: for a typical customer with a typical promotion, how many `RestrictionLevelRedemptionSummary` documents match (orgId, promotionId, level, customerId) before the date/KPI filter?

**Optimization candidate**: Add `redemptionKPI` and a date field to the compound index:
```
{orgId: 1, promotionId: 1, level: 1, customerId: 1, redemptionKPI: 1, periodStart: 1}
```
This would make the agg query nearly a point lookup + $sum on a tiny result set.

**Risk**: Adding this index increases write amplification on every redemption write. Must measure write throughput impact.

---

### Investigation C — Batched Aggregation (Single Query for All Promotions)

**Question**: Is a single MongoDB aggregation grouping by `{promotionId, level, redemptionKPI}` feasible given the query's date-predicate structure?

**Current query** (per-promotion per-level per-KPI):
```
$match: {orgId, promotionId, level, customerId, redemptionKPI, validTill > date, periodStart >= date}
$group: {_id: null, value: {$sum: kpiValue}}
```

**Proposed batch query** (all promotions in one shot for a given customer/level/KPI):
```
$match: {orgId, customerId, promotionId: {$in: [id1, id2, ...]}, level, redemptionKPI,
         validTill > date, periodStart >= date}
$group: {_id: {promotionId: "$promotionId", kpi: "$redemptionKPI"}, value: {$sum: "$kpiValue"}}
```

**What to assess**:
- Each promotion may have a different `restrictionDateBeforeFor` (varies by `frequency` and `minTimeBetweenRepeat`). This means the `validTill > date` predicate is promotion-specific. A single batched query can't apply per-promotion date predicates.
- **Verify**: do all promotions in a typical request use the same `frequency`? If yes (e.g., all MONTHLY), the batch query works. If frequencies differ across promotions in the same response, batching is only valid per-frequency group.
- **Sub-investigation**: Group promotions by `(frequency, minTimeBetweenRepeat)` and fire one aggregation per group. How many distinct frequency groups exist in typical requests?

---

### Investigation D — Parallelization Safety

**Question**: Can KPI queries for different promotions be executed in parallel without violating tenant isolation or correctness?

**What to verify**:
- Each query is read-only (no writes during `getAllForCustomer`). Safe to parallelize reads.
- `PromotionContext` and `RedemptionContext` are request-local. No shared mutable state between promotion processing paths.
- MongoDB Lettuce connection pool: check `spring.data.mongodb.uri` pool size. Parallelizing 50 promotions × 3 queries may exhaust the pool if sized for sequential workload.
- The `ReservationContext` is shared across promotions in `RedemptionProcessor` (via `customerPromotionContext.getReservationContext()`). Verify it is read-only after `loadReservationsFromPendingCarts()` completes before the forEach.

**If safe**: Replace sequential `forEach` in `RedemptionProcessor` with `parallelStream()` or `CompletableFuture.allOf()`. Expected gain: latency ≈ max(single promotion KPI time) instead of sum.

**Risk**: Parallel MongoDB requests may cause connection pool saturation at 900 RPM. Need to model concurrent requests × parallel queries per request vs pool capacity.

---

### Investigation E — Promotion-Level vs Customer-Level KPI Query Symmetry

**Question**: The KPI query rate breakdown (customer: ~13.5k, earn: ~3.9k, promo: ~240) suggests customer-level restrictions dominate. Are customer-level KPI queries hitting a well-indexed path, or is there a per-customer cardinality issue?

**What to assess**:
- For `CustomerRestrictionKPIHandler` (`shouldUseCustomerIdContext = true`), the query includes `customerId`. This means documents are already partitioned per-customer in the index.
- For `EarnRestrictionKPIHandler` (`shouldUseCustomerIdContext = true, shouldUseEarnedPromotionIdContext = true`), both customerId and earnedPromotionId are in the query — highly selective, likely fast.
- For `PromotionRestrictionKPIHandler` (`shouldUseCustomerIdContext = false`), customerId is null in the query. The index `{orgId, promotionId, level, customerId}` with `customerId = null` hits ALL customer documents for that promotion at promotion level. For popular promotions with high redemption volume, this could be a hot partition.
- **Verify**: For promo-level restrictions, how many documents match the index prefix before date/KPI filtering? Run `explain()` on a popular promotion.

---

### Investigation F — PromotionMeta Cache Hit Rate in RedemptionProcessor

**Question**: In `RedemptionProcessor`, each promotion calls `promotionMetaCacheService.getPromotionMeta(orgId, promotionId)`. What is the L1 (Caffeine) vs L2 (Redis) vs miss rate for these lookups during the request?

**What to assess**:
- `EarnedPromotionProcessor` already resolved PromotionMeta during step 1 for the customer's earned promotions. Steps 3 (`EarnedPromotionMetaProcessor`) also accesses the cache. By the time `RedemptionProcessor` runs at step 11, all promotion metas for this customer should already be in L1 cache.
- **Verify via New Relic**: Check `Custom/PromotionMetaCache/RedisEviction` and Micrometer cache stats for `promotionMetaCache`. If L1 eviction is non-zero during SLO-breach periods, the cache is undersized for the concurrent request load and is causing Redis round trips inside the hot path.

---

### Investigation G — External Service Calls in RedemptionProcessor

**Question**: `organizationService.getOrgZoneIdOrSystemDefault(orgId)` and `promotionOrgConfigurationService.getOrgConfiguration(orgId)` are called once per request in `RedemptionProcessor`. Are these cached or blocking?

**What to assess**:
- If `OrganizationService.getOrgZoneIdOrSystemDefault()` makes an outbound HTTP call (to Capillary CRM / org service) without caching, it adds RTT latency to every request.
- If `PromotionOrgConfigurationService.getOrgConfiguration()` is Redis/MongoDB-backed and not request-scoped, it adds a cache lookup per request.
- Check if these return quickly (<5 ms each) or are potential non-KPI contributors to the latency budget.

---

### Investigation H — Page-Before-KPI Ordering (Already Correct — Confirm No Regression Risk)

**Question**: Pagination (step 10) runs before KPI queries (step 11). Is there any path where KPI queries run on the pre-pagination list?

**What to verify**:
- The `customerPromotionProcessors` order in `CustomerPromotionFacade.initializeProcessors()` is `ImmutableList` — order is fixed and correct.
- Confirm no processor between steps 1–9 calls `PromotionRestrictionDetailsService` or any `PersistedRestrictionKPISummaryHandler` method.
- **Verify**: grep for `getRestrictionDetails\|getCurrentKpis` callers — should only be in `RedemptionProcessor`.

---

### Investigation I — Fixed Window Query Volume

**Question**: `getCurrentKpisForFixedWindow()` fires separate aggregations against `FixedWindowRedemptionSummaryCustomDao` and `PromotionLevelFixedWindowRedemptionSummaryCustomDao`. How many promotions in the response have fixed-window restrictions? What fraction of the total query load comes from fixed-window paths?

**What to assess**:
- `FixedWindowRedemptionSummary` index: `{orgId, promotionId, customerId, earnedPromotionId, promoCode, redemptionKPI, redemptionDate}` — well-covered for the date-range query.
- `PromotionLevelFixedWindowRedemptionSummaryCustomDao` — check its compound index.
- If fixed-window promotions are rare (<10% of orgs), their query load is not the priority. Confirm from the New Relic `QueryMetricsService` data (`getCurrentKpisForFixedWindow` service name).

---

## 5. Optimization Candidates (Ranked by Expected Impact)

To be validated by investigations above. Do not implement without confirming the
investigation finding first.

| # | Optimization | Pre-condition to validate | Expected gain | Risk |
|---|---|---|---|---|
| O1 | Batch KPI aggregation — group all promotionIds in one `$group by promotionId+level+kpi` per frequency group | Investigation C: confirm per-promotion date predicates are groupable | Reduces queries from N×L×K to (freq-groups × L × K) | Index shape may change; correctness of per-promo date boundary grouping |
| O2 | Extend compound index on `RestrictionLevelRedemptionSummary` to include `redemptionKPI + periodStart` | Investigation B: confirm high docs-examined ratio | Eliminates post-index scan; fastest single-query improvement | Write amplification on every redemption |
| O3 | Parallel KPI execution per promotion using `CompletableFuture` with bounded executor | Investigation D: confirm pool capacity and shared-state safety | Latency ≈ max(single promotion) instead of sum | MongoDB connection pool saturation under concurrent requests |
| O4 | Request-scoped bulk pre-load: fetch all `RestrictionLevelRedemptionSummary` docs for (orgId, customerId, {promotionIds}, dateRange) in one `find()` then group in-app | Investigation C: confirm fixed date predicate for customer-level | Eliminates all individual agg queries for customer-level; minimal index change | Memory: loading docs for 50+ promotions in-process |
| O5 | Short-lived Redis cache for KPI results (5 min TTL, keyed by orgId+promotionId+level+customerId+periodStart) | Investigation A: confirm same customer hits same promo KPI multiple times within 5 min | Eliminates repeat DB queries for high-RPM customers | Cache invalidation complexity; must follow `.context/code.md` Step 0 → Step 1 → Step 2 hierarchy |

---

## 6. Scope

### In Scope
- Analysis of KPI query patterns, query plans, and index coverage
- Profiling `RedemptionProcessor` and `PromotionRestrictionDetailsService` under production-like load
- Identifying top-N orgs driving the query volume
- Evaluation of batching, parallelism, and index changes
- Producing a ranked optimization shortlist with evidence for each option

### Out of Scope
- Changes to the 11-processor pipeline order
- Changes to `PromotionMeta` cache (L1/L2) — separate MADR governs this
- Changes to how earned promotions, issued promotions, or pagination work
- Any write-path changes (redemption recording, KPI update)

### Deferred
- Redis caching of KPI query results (requires `.context/code.md` Step 0 design-pattern analysis first)
- Cross-shard query optimization (relevant only for USHC cluster multi-shard setup)

---

## 7. Assumptions

| # | Assumption | Breaks if... |
|---|-----------|-------------|
| 1 | Pagination (step 10) always runs before KPI queries (step 11) | Processor order in `ImmutableList` is changed |
| 2 | All KPI queries are read-only during `getAllForCustomer` | Any processor before step 11 triggers a KPI write |
| 3 | The query volume attributed to customer-level KPIs dominates (13.5k vs 3.9k earn vs 240 promo) | Org growth shifts the distribution to promo-level (null customerId) which has different cardinality |
| 4 | Fixed-window restrictions are less common than sliding-window across the org base | Major orgs migrate to fixed-window configuration |

---

## 8. Data Model (Read-Only for This Analysis)

Collections queried in the hot path:

| Collection | BO Class | Index gaps to evaluate |
|---|---|---|
| `restrictionLevelRedemptionSummary` | `RestrictionLevelRedemptionSummary` | `redemptionKPI`, `periodStart`, `validTill` not in compound index |
| `fixedWindowRedemptionSummary` | `FixedWindowRedemptionSummary` | Fully indexed: `{orgId, promotionId, customerId, earnedPromotionId, promoCode, redemptionKPI, redemptionDate}` |
| `promotionLevelFixedWindowRedemptionSummary` | `PromotionLevelFixedWindowRedemptionSummary` | Verify index includes `redemptionKPI` and `redemptionDate` |

No schema changes in this analysis story.

---

## 9. API

No API changes. This is a latency optimization on the existing `GET /promotions/customer/{customerId}` endpoint.

---

## 10. Security

| Check | Result |
|-------|--------|
| New API endpoints requiring auth/role checks | N/A — no new endpoints |
| PII in new log lines | N/A — analysis only |
| New fields storing PII | N/A |
| Token / credential handling | N/A |
| Tenant data leak risk (new shared/global state) | N/A — analysis only; any caching optimization must use orgId-scoped cache keys |
| User-supplied input in new code paths | N/A |

> Note: If Investigation E confirms that promo-level KPI queries return data for NULL customerId
> across all customers of a promotion, confirm that the API response correctly scopes returned
> restriction data to the requesting customer only (no cross-customer data exposure). This is a
> read-isolation check, not a write-isolation check.

---

## 11. DB & Infra Considerations

```
CONSIDERATION [DB] [ACTION]
Finding: RestrictionLevelRedemptionSummary compound index does not include redemptionKPI or date fields.
         Every aggregation scans all docs matching (orgId, promotionId, level, customerId) before
         applying KPI and date predicates.
Impact if ignored: Query latency grows with redemption volume per promotion (unbounded growth).
Recommended action: Run explain() in staging; if docs-examined >> n-returned, add extended index
                    as O2 above. Measure write amplification before applying to prod.
Owner: tech-detailer / DBA

CONSIDERATION [INFRA] [NOTE]
Finding: Parallelizing KPI queries (O3) multiplies MongoDB connection usage by concurrent-query factor.
         Lettuce pool sizing must be evaluated before enabling parallelism at 900 RPM.
Impact if ignored: Connection pool exhaustion → timeouts → SLA breach.
Recommended action: Model peak connections = RPM/60 × page_size × parallel_factor × avg_query_ms/1000.
                    Verify against current pool max-size in application config.
Owner: tech-detailer / infra team
```

---

## 12. Observability — Required for Analysis

The following signals must be captured before any optimization can be sized:

| Signal | Where to get it | What to measure |
|---|---|---|
| KPI query duration distribution | New Relic custom events (`QUERY_DURATION` attribute in `QueryMetricsService`) | p50, p95, p99 per level (Customer, Earn, Promo, Code) |
| KPI query count per request | `restriction.<level>` custom params in `CustomerPromotionFacade` | Avg and max KPI queries per request, by org |
| MongoDB `explain()` on `getKPISumAfterDateFor` | MongoDB shell / Compass on staging | `totalDocsExamined` vs `nReturned` |
| PromotionMeta L1 cache eviction rate | Micrometer `promotionMetaCache` metrics | Steady-state eviction count (non-zero = undersized) |
| `organizationService` and `promotionOrgConfigurationService` latency | New Relic transaction traces | p95 call time for both in `RedemptionProcessor` |
| Fixed-window vs sliding-window query split | `addQueryMetrics` serviceName = `"getCurrentKpisForFixedWindow"` vs others | % of queries from fixed-window path |

---

## 13. SLA Impact

| Scenario | Today | Target | Gap to close |
|---|---|---|---|
| SLO 1 & 2 (~900 RPM, 50 items) | 260 ms p95 | ≤ 150 ms | 110 ms |
| SLO 3 (~225 RPM, 50–100 items) | ~400 ms p95 | < 150 ms | 250 ms |

The larger gap in SLO 3 (100 items) is consistent with the linear scaling of KPI queries with page size: 100 items produce ~2× the queries of 50 items and ~2× the latency. This confirms the query count is the primary contributor, not network or serialization overhead.

---

## 14. Risks

| # | Description | Likelihood | Mitigation |
|---|------------|-----------|-----------|
| R1 | Investigation C reveals per-promotion date predicates cannot be batched (different frequencies per promotion) | Medium | Fall back to O3 (parallelism) or O4 (bulk find + in-app group) instead of O1 |
| R2 | Adding extended index (O2) causes write latency regression on high-redemption orgs | Medium | Measure with and without index in staging under redemption load; A/B before prod |
| R3 | Parallelizing queries (O3) exhausts MongoDB connection pool at 900 RPM | Medium | Model and test under load; add connection pool monitoring alert before enabling |
| R4 | Some SLO breach is driven by outlier orgs with 200+ active promotions per customer — optimization doesn't help the tail | Low | Identify heavy orgs in Investigation A; add org-level rate limiting as a safety valve |

---

## 15. Open Questions

| Question | Owner | Due |
|---------|-------|-----|
| Q1: What is the max number of distinct `(frequency, minTimeBetweenRepeat)` values observed across promotions in a single `getAllForCustomer` response? (batching feasibility) | Saravanan | Before optimization sprint |
| Q2: What is the MongoDB Lettuce connection pool max size in production? (`spring.data.mongodb.options.max-connection-pool-size` or equivalent) | Infra team | Before parallelism decision |
| Q3: Does `OrganizationService.getOrgZoneIdOrSystemDefault()` make a network call or is it already cached within the service? | Saravanan (code check) | Investigation G |
| Q4: For promo-level KPI queries (customerId = null), what is the typical doc count returned before date filtering? Is this a hot partition risk? | Saravanan (explain()) | Investigation E |

---

## 16. Task Breakdown (Analysis Story)

**Analysis tasks — no implementation in this story**

| # | Task | Size | Depends on |
|---|------|------|-----------|
| T1 | Pull New Relic query distribution: (org, level, duration) heatmap using `QueryMetricsService` events | M | — |
| T2 | Run `explain("executionStats")` on `getKPISumAfterDateFor` for top-5 heavy orgs in staging | M | T1 (identify orgs) |
| T3 | Run `explain()` on promo-level (customerId = null) query for popular promotion | S | T1 |
| T4 | Assess per-promotion date predicate variability: sample 20 requests, count distinct `(frequency, minTimeBetweenRepeat)` groups | S | — |
| T5 | Check `OrganizationService` and `PromotionOrgConfigurationService` for caching / network call | S | — |
| T6 | Model MongoDB connection pool usage for parallelism scenario at 900 RPM | S | — |
| T7 | Check PromotionMeta L1 cache eviction rate from Micrometer dashboards | S | — |
| T8 | Produce ranked shortlist of optimizations with evidence from T1–T7 | M | T1–T7 |

---

## Handoff Notes for Implementation Story

After T1–T8 complete, a follow-up implementation story should be created. Likely candidates for implementation (to be confirmed by analysis):

### Expected Critical Path
1. **Index extension** (O2) — highest confidence, lowest blast radius; apply first to reduce per-query cost
2. **Parallelism** (O3) — apply second to eliminate sequential wait; requires pool sizing validation
3. **Batched aggregation** (O1) — only if T4 confirms frequency groups are homogeneous in typical requests

### Regression Risks
- Processor pipeline order (`ImmutableList` in `CustomerPromotionFacade:98`) must not change
- Any index addition must be `background: true` and applied to staging under load before prod
- `RedemptionKPI` enum values must not be reordered (used as stored strings in MongoDB)

### Tenant Isolation Tests
- All KPI query paths scope by `orgId` at the first criteria position — verify no cross-org leakage in any batched query alternative
- Promo-level (null customerId) queries: confirm response only exposes aggregate, not per-customer data

### Contract Tests
- `RestrictionLevelRedemptionSummaryCustomDao.getKPISumAfterDateFor()` must return identical results before and after any index or batching change — integration test with known dataset is the regression gate
