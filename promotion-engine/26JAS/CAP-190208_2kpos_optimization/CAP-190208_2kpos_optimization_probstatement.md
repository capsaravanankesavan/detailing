# Problem Statement: CAP-190208 — 2K POS Promotion Optimization

## Symptom

`POST /v1/promotions/evaluate` latency is unacceptable at POS-heavy org scale.

| Metric | Observed (staging, Jun 18 2026) | SLO Target |
|--------|---------------------------------|------------|
| Average response time | 9.5s | ≤ 300ms |
| Median response time | 9.09s | ≤ 300ms |
| 95th percentile | 23.8s | ≤ 500ms |
| 99th percentile | 27.8s | — |
| Apdex score | 0.16 (very poor) | ≥ 0.85 |
| Error rate | 27.29% | ≤ 0.5% |
| Throughput | 464 RPM | Must sustain ≥ 1,000 RPM at 2K POS scale |

## Scope

- **Affected flow:** `POST /v1/promotions/evaluate` (cart evaluation for POS orgs)
- **Trigger condition:** Org has ≥ 1,000 active POS promotions where each promotion is tied to specific SKU(s) (Pharma-sector pattern: one promotion per SKU, plus combo SKU promotions)
- **Scale target:** Must support orgs with up to 2,000 active POS promotions at 1,000 RPM
- **Environment confirmed:** `crm-staging-new-promotion-engine` with 1,000 active POS promotions

## Entry Point

- **Service:** `promotion-engine`
- **Endpoint:** `POST /v1/promotions/evaluate`
- **Controller:** `PromotionRedemptionResource.evaluateCart()` → `PromotionRedemptionFacade.evaluateCart()`

## Expected vs Actual Behaviour

**Expected:** Cart evaluation for a customer with a 10-item Pharma cart completes in ≤ 300ms regardless of how many POS promotions are active in the org. Only promotions whose required SKUs are present in the cart should be evaluated.

**Actual:** All 1,000 active POS promotions pass shortlisting and are evaluated sequentially. With 300 promotions surviving shortlisting, 552 sequential calls to `PromotionEvaluatorService.apply()` are made per request. Of those 552 evaluations, ~298 are for promotions whose required SKUs are not in the cart at all — they are rejected by the full evaluator after the fact.

## Evidence

- New Relic APM trace: `crm-staging-new-promotion-engine`, Jun 18 2026, 5:10–5:20 PM GMT+5:30
- Top segments by time:
  - `ApplicationService/apply`: 552 calls/txn, 2.13s total (21.67%)
  - `PromotionEvaluatorService/apply`: 551 calls/txn, 1.60s total (16.35%)
  - `getKPISumAfterDateFor`: 1.55 calls/txn, 1.45s total — 935ms/call vs 24ms actual MongoDB time (connection pool saturation signal)
  - `getActivePromotionIdsByType`: 0.622 calls/txn, 1.07s total — 1.72s/call (62% cache miss rate)
- Staging data: 1,000 active POS promotions, 300 passing all shortlisting phases
- `saveAppliedPromotions` called ~1.2 times/txn → the vast majority of 300 shortlisted promotions are evaluated and rejected
