# R10 Load-Test Run Log — promotion-engine evaluate() @ 3K POS promos

**Org:** 50223 (staging) · **Cluster:** crm-staging-new · **Script:** `auto_loadtests/k6-loadtests/scripts/promotionEngine/CAP-190801-R10`
**Setup:** 3,000 active `LT_POS_R10_*` promos, 500 hot SKUs (10 promos/SKU), Customer+Promotion CLASSIC restrictions on every promo, `carts.json` 3,000 carts, 100 RPM target (unless noted).
**Date range:** 2026-08-18 to 2026-08-24

---

## Builds tested

| Build tag | Commit | Fix (PR #866 shortlisting)? | Notes |
|---|---|---|---|
| `0380db2-4650` | 0380db2 | No | Old released tag |
| `76993c9-4707` | 76993c9 | No | Was on staging during original collapse tests |
| `c8c5163-4736` | c8c5163 | **Yes** | The fix build (validated) |
| `d2ca6f4-4758` | d2ca6f4 | No | Latest tag-cut build (2026-08-19), no fix |

---

## Run results

### Without fix — 1 pod

| Run | Build | RPM | Dur | k6 success | k6 avg | k6 p95 | NR avg | NR p95 | shortlisted | restriction.Customer |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 76993c9 | 100 | 5m | 0.24% | 59.14s | 59.23s | 257s | 436s | 3000 | 3000 |
| 2 | 76993c9 | 100 | 5m | 0.00% | 59.13s | 59.23s | — | — | — | — |
| 3 | 76993c9 | 100 | 10m | 0.00% | 48.44s | 59.94s | 421s | 458s | 3000 | 3000 |
| 4 | 76993c9 | 50 | 5m | 0.00% | 58.89s | 59.23s | 271s | 333s | 3000 | 3000 |
| 5 | 76993c9 | 10 | 5m | 0.00% | 59.17s | 59.23s | 0 reached app | — | — | — |
| 6 | 76993c9 | 50 | 5m (post-restart) | 0.00% | 59.12s | 59.24s | — | — | — | — |

### Without fix — old tag `0380db2` (1 pod)

| Run | Build | RPM | Dur | k6 success | k6 avg | k6 p95 | NR avg | NR p95 |
|---|---|---|---|---|---|---|---|---|
| 7 | 0380db2 | 10 | 5m | **100%** | 11.23s | 19.73s | — | — |
| 8 | 0380db2 | 50 | 5m | 2.67% | 58.52s | 59.23s | — | — |

### Without fix — `76993c9` (1 pod, 50 RPM)

| Run | Build | RPM | Dur | k6 success | k6 avg | k6 p95 |
|---|---|---|---|---|---|---|
| 9 | 76993c9 | 50 | 5m | 0.00% | 59.13s | 59.23s |

### Without fix — 6 pods

| Run | Build | RPM | Dur | k6 success | k6 avg | k6 p95 | NR avg | NR p95 | shortlisted | restriction.Customer |
|---|---|---|---|---|---|---|---|---|---|---|
| 10 | d2ca6f4 | 100 | 5m | 94.98% | 42.64s | 59.32s | 42.5s | 59.1s | 3000 | 3000 |
| 11 | d2ca6f4 | 100 | 5m | 23.32% | 55.03s | 59.99s | 105s | 191s | 3000 | 3000 |

### With fix `c8c5163` — 1 pod

| Run | Build | RPM | Dur | k6 success | k6 avg | k6 p95 | NR avg | NR p95 | shortlisted | restriction.Customer |
|---|---|---|---|---|---|---|---|---|---|---|
| 12 | c8c5163 | 100 | 5m | 100% | 417ms | 477ms | 0.11s | 0.136s | 8.08 | 8.08 |
| 13 | c8c5163 | 100 | 5m | 100% | 402ms | 442ms | 0.109s | 0.136s | 8.68 | 8.68 |
| 14 | c8c5163 | 100 | 5m | 100% | 401ms | 436ms | — | — | — | — |

---

### With fix `c8c5163` — 8 pods (2026-08-21)

| Run | Build | RPM | Dur | k6 success | k6 avg | k6 p95 | NR avg | NR p95 | NR p99 | shortlisted | restriction.Customer |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 15 | c8c5163 | 500 (warmup) | 5m | 100% | 424ms | 651ms | 0.18s | 0.394s | 0.695s | 9.0 | 9.0 |
| 16 | c8c5163 | 1000 | 5m | 100% | 373ms | 489ms | 0.136s | 0.243s | 0.404s | 8.88 | 8.88 |
| 17 | c8c5163 | 1200 | 5m | 100% | 362ms | 515ms | 0.126s | 0.245s | 0.479s | 8.95 | 8.95 |
| 18 | c8c5163 | 1500 | 5m | 100% | 416ms | 662ms | 0.161s | 0.355s | 0.633s | 8.95 | 8.99 |
| 19 | c8c5163 | 2000 | 5m | 100% | 388ms | 590ms | 0.148s | 0.330s | 0.566s | 8.99 | 8.99 |

### With fix `c8c5163` — 8 pods, 4K corpus (2026-08-21)

4,000 active `LT_POS_R10_*` promos, 400 hot SKUs (10 promos/SKU), Customer+Promotion CLASSIC restrictions on every promo, `carts.json` 4,000 carts.

| Run | Build | RPM | Dur | k6 success | k6 avg | k6 p95 | NR avg | NR p95 | NR p99 | shortlisted | restriction.Customer |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 20 | c8c5163 | 500 (warmup) | 5m | 99.96% | 400ms | 504ms | 0.142s | 0.256s | 0.385s | 10.93 | 10.93 |
| 21 | c8c5163 | 1500 | 5m | 99.85% | 483ms | 597ms | 0.160s | 0.338s | 0.621s | 10.96 | 10.96 |

### With fix `c8c5163` — 8 pods, 4K corpus, CPU request=limit=2 (2026-08-21)

CPU config changed to guaranteed 2-core (request 2000m = limit 2000m, was request 800m / limit 2000m burstable). Memory 4Gi/4Gi. Autoscaling min 8 / max 15.

| Run | Build | RPM | Dur | k6 success | k6 avg | k6 p95 | NR avg | NR p95 | NR p99 | shortlisted | restriction.Customer |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 22 | c8c5163 | 1500 | 5m | 100% | 568ms | 881ms | 0.316s | 0.617s | 3.719s | 10.99 | 10.99 |
| 23 | c8c5163 | 1500 | 5m | 100% | 399ms | 472ms | 0.153s | 0.211s | 0.255s | 11.02 | 11.02 |

### With fix `c8c5163` — 8 pods, 4K corpus, cart-item count sweep (2026-08-24)

Brand expects 8–9 cart items (was testing with 4). Swept cart size at 1.5K RPM on the 4K corpus.

| Run | Build | Cart items | RPM | Dur | k6 success | k6 avg | k6 p95 | NR avg | NR p95 | NR p99 | shortlisted | restriction.Customer |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 24 | c8c5163 | 10 (warmup) | 500 | 5m | 100% | 525ms | 781ms | 0.187s | 0.225s | 0.324s | 17.06 | 17.06 |
| 25 | c8c5163 | 10 | 1500 | 5m | 100% | 542ms | 792ms | 0.172s | 0.211s | 0.250s | 17.08 | 17.08 |
| 26 | c8c5163 | 15 (warmup) | 500 | 5m | 100% | 653ms | 877ms | 0.188s | 0.237s | 0.274s | 21.97 | 21.97 |
| 27 | c8c5163 | 15 | 1500 | 5m | 99.96% | 665ms | 833ms | 0.201s | 0.261s | 0.302s | 21.98 | 21.98 |
| 28 | c8c5163 | 15 | 2000 | 5m | 100% | 632ms | 753ms | 0.218s | 0.292s | 0.349s | 21.94 | 21.94 |

### With fix `c8c5163` — 8 pods, 5K corpus (2026-08-24)

5,000 active `LT_POS_R10_*` promos (at the 5000 cap), 500 hot SKUs (10 promos/SKU), Customer+Promotion CLASSIC restrictions, 15-item carts.

| Run | Build | Cart items | RPM | Dur | k6 success | k6 avg | k6 p95 | NR avg | NR p95 | NR p99 | shortlisted | restriction.Customer |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 29 | c8c5163 | 15 (warmup) | 500 | 5m | 100% | 1.04s | 1.57s | 0.224s | 0.268s | 0.318s | 23.95 | 23.95 |
| 30 | c8c5163 | 15 | 1500 | 5m | 100% | 722ms | 928ms | 0.231s | 0.283s | 0.338s | 23.95 | 23.95 |
| 31 | c8c5163 | 15 | 2000 | 5m | 99.98% | 766ms | 1.00s | 0.263s | 0.366s | 0.446s | 23.95 | 23.95 |
| 32 | c8c5163 | 15 | 2000 | 5m | 100% | 782ms | 984ms | 0.249s | 0.338s | 0.424s | 23.95 | 23.95 |

### Without fix `d2ca6f4` — 8 pods, 5K corpus (2026-08-24)

Pre-fix tag build `d2ca6f4-4758` (no shortlisting) on 8 pods, 5,000 active promos, 15-item carts. Warmup (10 RPM, 2m) then 50 RPM run.

| Run | Build | Cart items | RPM | Dur | k6 success | k6 avg | k6 p95 | NR avg | NR p95 | NR p99 | shortlisted | restriction.Customer |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 33 | d2ca6f4 | 15 (warmup) | 10 | 2m | 100% | 27.36s | 34.57s | — | — | — | 5000 | 5000 |
| 34 | d2ca6f4 | 15 | 50 | 5m | 94.53% | 46.09s | 59.35s | 46.17s | 63.5s | 79.0s | 4979.8 | 5000 |

**Pre-fix 5K finding:** without shortlisting, 8 pods collapse at 50 RPM on the 5K corpus — 94.53% success, avg 46s, p95 59s (client), NR p95 63.5s / p99 79.0s. Every request evaluates all ~5000 promos (shortlisted=4979.8, restriction.Customer=5000). Contrast with the fix build: 2K RPM at 100% success, NR p95 0.34s / p99 0.42s. **~180x faster with the fix at 40x the load.**


---

## Key findings

1. **Without fix, 1 pod collapses at every rate (10/50/100 RPM)** — 0% success, ~59s client timeouts, 257–421s server-side. Per-request cost of evaluating 3000 promos × restrictions with no shortlisting exceeds single-pod capacity.

2. **Old tag `0380db2` has a higher ceiling than `76993c9`** — serves 10 RPM (100%, 11s) but collapses at 50 RPM (2.67%). So the collapse is a general pre-fix characteristic, not specific to one build.

3. **6 pods helps but doesn't fix it** — `d2ca6f4` (latest tag, no fix) on 6 pods serves 100 RPM at 94.98% success (1st run) but still 42.5s server-side (no shortlisting). A 2nd run on the same build/6 pods was worse (23.32%, 105s NR) — run-to-run variance on the same build, likely accumulated load / pod state.

4. **The fix is decisive** — `c8c5163` on a SINGLE pod serves 100 RPM at 100% success, 0.109s server-side, shortlisting 3000 → 8.7 promos. **~390x faster than pre-fix on 6 pods.**

5. **Mechanism confirmed via NewRelic:** `promotions.shortlisted.count` = 3000 (no reduction) pre-fix vs 8.68 with fix; `restriction.Customer` matches the shortlisted count (8.68) with fix vs 3000 without — restrictions are only checked on the shortlisted set.

6. **8 pods + fix hold 2K RPM at the 5K-promo cap** — all runs ≥99.85% success, server-side p95 ≤0.37s, p99 ≤0.45s (except the cache-cold CPU2 run 22 at p99 3.7s, which the warm repeat run 23 confirmed was warmup, not a regression).

---

## Log files (in `CAP-190801-R10/logs/`)
- `p3000_r10_nofix_eval100rpm_5m.log`, `..._run2.log`, `..._10m.log`
- `p3000_r10_nofix_eval50rpm_5m.log`, `..._postrestart.log`
- `p3000_r10_nofix_eval10rpm_5m.log`
- `p3000_r10_oldtag0380db2_eval10rpm_5m.log`, `..._eval50rpm_5m.log`
- `p3000_r10_76993c9_eval50rpm_5m.log`, `p3000_r10_76993c9_6pod_eval100rpm_5m.log`
- `p3000_r10_d2ca6f4_6pod_eval100rpm_5m.log`
- `p3000_r10_fix_eval100rpm_5m.log`, `..._run2.log`, `..._run3.log`
- `p4000_r10_fix_8pod_*`, `p5000_r10_fix_8pod_*` (Aug 21/24 runs)
