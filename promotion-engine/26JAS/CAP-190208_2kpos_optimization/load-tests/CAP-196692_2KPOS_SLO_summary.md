# CAP-196692 (PR #866) — 2K POS Promotion evaluate() Latency: SLO Summary

**Service:** promotion-engine-a · **Cluster:** crm-staging-new · **Org:** 50223 (staging)
**PR:** https://github.com/Capillary/promotion-engine/pull/866 — SKU-based shortlisting to reduce evaluator calls at 2K POS promotion scale
**Date of testing:** 2026-08-11 to 2026-08-13

---

## Test conditions

| Parameter | Value | Source of confirmation |
|---|---|---|
| Infra | 1 pod, CPU limit 2 cores, memory limit 4GB | Grafana (Kubernetes Application Overview, `promotion-engine-a`); no horizontal scaling in staging (cost constraint) |
| Load | 100 RPM sustained, `evaluate` only | k6 (`mixed_loadtest.js`, `EVAL_RPM=100 GET_RPM=0`) |
| Promo cardinality | 2,500 active POS promos | Confirmed via `/v1/promotion-management/promotions/filters?active=true` (management API) and NewRelic `promotions.pos.count=2500` on every run |
| Hot SKUs | 2 (`HOT_SKU_01`, `HOT_SKU_02`), ~1,250 promos per hot SKU | `common.py::sku_block()` — `hot = hotSkus[i % 2]` over indices 1..2500 |
| Cart size | **4 items per cart** (1 guaranteed hot-SKU hit + 3 promo-specific SKUs) | NewRelic `cart.items.count` (avg 4.0 post-fix, 3.80 pre-fix); generator code `items_per_cart=4` |

**Note:** 8 is `skusPerPromo` — the SKU-IN condition list length on each *promotion*, not the cart size. Cart size is 4.

**Important scoping note:** 100 RPM was deliberately chosen as the comparison tier because it sits below the pod's CPU-throttling ceiling. At 200 RPM+ the single pod repeatedly hits its 2-core limit and gets throttled (visible directly in Grafana as CPU pinned at 2.00 with sharp drops), producing highly unstable latency (ranging 2s to 35s+ avg across repeated runs) — that instability is present in **both** pre-fix and post-fix builds equally, so it is an infra/capacity characteristic of the single-pod cost constraint, not something this fix addresses or regresses.

---

## SLO1 — evaluate() latency (server-side, NewRelic `duration`, app `crm-staging-new-promotion-engine`)

| | avg | p95 | Runs | Success rate |
|---|---|---|---|---|
| **SLO1-current** (pre-fix) | **456ms** | **491ms** | 3 × 201 req | 100% |
| **SLO1-withFix** (post-fix, PR #866) | **264ms** | **284ms** | 4 × 201 req | 100% |
| **Improvement** | **−42%** | **−42%** | | |

Both sets are tightly reproducible run-to-run:
- Pre-fix p95 range across 3 runs: 479–513ms
- Post-fix p95 range across 4 runs: 278–330ms

All measurements are **app-level NewRelic `duration`** (pure server-side compute, excludes client network/TLS/DNS overhead). Client-side (k6, from a laptop) showed a similar but smaller relative improvement (~25%, ~747ms→~558ms avg) because network overhead to staging is constant across both builds and dilutes the relative server-side gain.

---

## Confirmed mechanism — not inferred

NewRelic custom attribute `promotions.shortlisted.count` (introduced by PR #866's `ItemBasedShortlistingService`), averaged across every run:

| Build | `promotions.pos.count` | `promotions.shortlisted.count` | Reduction |
|---|---|---|---|
| Pre-fix (confirmed via revert to previous tag) | 2500 | 2500 (no reduction) | 0% |
| Post-fix | 2500 | **1251.5** | **~50%** |

This confirms the fix is doing exactly what it's designed to do — eliminating promotions whose SKU isn't in the cart before the expensive downstream evaluator/capping path runs — deterministically, on every run, not just correlating with the latency drop.

---

## CPU — directionally confirmed, precise % pending an isolated capture

From the attached before/after Grafana panel (`Container CPU Cores Used`, limit 2 cores per pod):

- **Pre-fix pod** (`...gtx4d`): avg 0.590 cores over the captured window — but this window includes 200 RPM runs where the pod visibly hits the 2.0-core ceiling and throttles. During 100-RPM-only stretches it oscillates roughly 0.5–1.0 core.
- **Post-fix pod** (`...g967x`): avg 0.535 cores — includes a brief cold-start spike to 2.0 right after deploy (JIT/cache warmup, expected), then settles to roughly 0.3–0.6 core during 100 RPM runs.

**Confirmed:** the post-fix pod runs measurably cooler at steady state — consistent with doing ~50% less promo-matching work per request.

**Not yet certified:** a precise "50% CPU reduction" or "1 core → 0.5 core" figure. The available capture mixes RPM tiers (pre-fix side) and includes a cold-start transient (post-fix side), so it isn't a clean 100-RPM-only, steady-state-only comparison. **Recommend one more isolated capture** — a 100-RPM-only pre-fix window and a 100-RPM-only post-fix window (post-warmup), each with no other RPM tier or deploy event in frame — before quoting an exact CPU percentage with the same confidence as the latency numbers.

---

## Next step: 50 hot SKUs (5 promos/SKU) instead of 2 hot SKUs (1,250 promos/SKU)

Hypothesis: a brand like SPF with far lower promo concentration per hot SKU should shortlist even more aggressively and further reduce latency, since `ItemBasedShortlistingService`'s benefit scales with how much the shortlisting step can eliminate.

**Correction to the original estimate:** the measured effect for this test's one shortlisting step (2500→1251 promos, a 50% cut) was **~207ms on p95** (491ms→284ms), not ~100ms. Using the corrected per-step rate changes any linear extrapolation.

**Caveat on extrapolating to <100ms:** this relationship is very unlikely to be linear — there is a fixed floor from network round-trip, serialization, and base request-handling overhead that shortlisting cannot shrink below. A responsible framing for the next test: *"we expect further latency reduction from a lower promo-per-hot-SKU ratio, with diminishing returns expected below the fixed request-overhead floor — to be confirmed by testing, not by extrapolation."*

---

## Evidence sources (for reference)
- k6 load-test logs: `auto_loadtests/k6-loadtests/scripts/promotionEngine/CAP-190801/logs/*100rpm*.log`
- NewRelic NRQL: `Transaction` events, `appName='crm-staging-new-promotion-engine'`, `name LIKE '%evaluate%'`, filtered to each run's UTC window
- Grafana: Kubernetes Application Overview dashboard, `promotion-engine-a`, CPU/memory panels
- Code: `cc-stack-crm/service/instances/promotion-engine-a.json` (CPU/memory limits, `autoscaling.min:1,max:1`), `auto_loadtests/.../common.py` (SKU/cart generation logic)
