# Investigation Plan: issueBulkReward Inconsistency — Points vs Benefits
**Date:** 2026-05-06
**Author:** saravanan.kesavan@capillarytech.com
**Ticket:** CAP-184648
**Story Type:** Analysis (this story produces confirmed root causes — solution implementation is a separate story)
**Status:** Not Started

---

## 1. Story Goal

Produce a **confirmed root cause list** for the two classes of inconsistency seen in `issueBulkRewardAPI`, with enough evidence from code, config, and downstream system behaviour to unambiguously specify fixes in the next story.

Additionally, assess whether a **distributed transaction management (DTM) pattern** — specifically Saga orchestration or TCC — should be adopted on top of the tactical root cause fixes, or whether it is infeasible given the constraints of downstream systems and vendor APIs.

**This story does NOT implement any fix.** Its output is a completed version of this document with every hypothesis marked CONFIRMED or RULED OUT, every open question answered, and a populated "Handoff to Tech Detail" section.

---

## 2. Observed Problem

`issueBulkRewardAPI` produces two classes of inconsistency in production:

| Class | Symptom | Error volume (from ticket) |
|-------|---------|--------------------------|
| **Brand loses** | Customer receives benefit (coupon / cart promotion) without points being deducted | USHC: ~1,250; other clusters: ~600 |
| **Customer loses** | Customer's points are deducted but the benefit is revoked | USHC: 0.714%, ASIA: 0.259%, others: 0.007–0.02% |

USHC shows the highest rate on both classes. The asymmetry across clusters is unexplained — the analysis must determine whether it is driven by reward-type mix (more vendor rewards in USHC), latency profile, or config differences.

---

## 3. System Map — Full Call Chain

> Read this before starting any analysis step. The processor order is the single most important structural fact about this system.

```
API Request
  └─► UserRewardUtils.issueReward()          [CustomerLockable: orgId_customerId, TTL=?]
        └─► BulkIssueService.issueBulk()
              ├─ [10] IsRedeemableProcessor
              │        └─► IntouchServiceImpl.isRedeemable()
              │                └─► HTTP: intouch /redemptions/isRedeemable
              │                        └─► (internal) emf.checkPoints (thrift)
              │
              ├─ [14] CouponIssueProcessor
              │        └─► IntouchServiceImpl.issueBulkIntouchReward()
              │                └─► HTTP: intouch /v2/coupon/bulk/issue    ←── TIMEOUT POINT A
              │                        └─► luci.issueCoupon (thrift)      ←── commit timing: unknown
              │
              ├─ [15] PromotionEarnProcessor
              │        └─► HTTP: PromotionEngine /earn                    ←── TIMEOUT POINT B
              │
              ├─ [16] VendorIssueProcessor    (awaitTermination: ?)       ←── LOCK EXPIRY WINDOW?
              │        └─► External vendor API calls (parallel threads)
              │
              ├─ [17] PointsRedeemProcessor
              │        └─► IntouchServiceImpl.redeemIntouchPoints()
              │                └─► HTTP: intouch /redemptions/redeem      ←── TIMEOUT POINT C
              │                        └─► emf.redeemPoints (thrift)      ←── commit timing: unknown
              │
              ├─ [19] CouponRevokeProcessor   (fires on: ?)
              │        └─► LuciService.revokeCoupon()
              │
              └─ [20] RevokeEarnProcessor     (fires on: ?)
                       └─► PromotionEngineService.revokeEarn()

Async retry path (RabbitMQ):
  PointsRedemptionListener.retryPointsRedeem()
    └─► PointsRedemptionFacade.retryPointsRedeem()
          ├─ checks getCustomerRedemptions(externalReferenceNumber)   ←── idempotency oracle: unverified
          └─► IntouchServiceImpl.redeemIntouchPoints()  (retry)
```

> "?" markers are unknowns to be filled in during analysis. Do not assume default values.

---

## 4. Hypotheses Going In

These are the **suspected** root causes based on an initial code read. Each must be validated — not assumed — during the analysis sequence.

| # | Hypothesis | Failure class it would explain | Analysis step that tests it |
|---|-----------|-------------------------------|----------------------------|
| H-1 | Redis lock TTL is shorter than the maximum processor chain execution time (vendor path), allowing a concurrent request to pass `isRedeemable` before the first request deducts points | Both classes | Step 2 |
| H-2 | On `issueBulkCoupon` timeout (Timeout Point A), luci has already committed the coupon but rewards marks the wrapper as failed → points = 0 → no deduction | Brand loses | Step 3 + Step 6 |
| H-3 | On `redeemPoints` timeout (Timeout Point C), EMF has already committed the deduction but rewards triggers coupon revocation → customer loses benefit and points | Customer loses | Step 3 + Step 5 |
| H-4 | The revocation trigger (`shouldRevokeCoupons`) only fires on one exact status code — other points failure codes leave benefits un-revoked silently | Brand loses (silent) | Step 3 |
| H-5 | The async retry path does not check EMF state before revoking at max retry, causing revocation even when EMF already deducted | Customer loses | Step 4 + Step 5 |
| H-6 | USHC's higher error rate is explained by a higher proportion of vendor reward transactions (longer chain = more lock expiry exposure) | Asymmetry across clusters | Step 1 |
| H-7 | The current processor chain is conceptually a saga (issue benefit → deduct points → compensate on failure) but violates saga properties: no durable saga state, compensation trigger is a fragile code match, and compensating transactions are not retried on their own failures | Architecture | Step 9 |
| H-8 | A properly orchestrated saga is feasible for the intouch/luci/EMF/PE steps but NOT for vendor API steps (no compensation protocol exists on external vendor APIs), meaning vendor steps would remain a gap regardless of DTM pattern chosen | Architecture | Step 9 |

---

## 5. Analysis Sequence

Execute steps in order. Each step has a **gate**: if the gate condition is not met, do not proceed to the next step — raise a blocker.

---

### Step 1 — Quantify and classify production errors by cluster

**Goal:** Understand the true shape of the problem before touching code. Prevents misattribution of cause.

**What to do:**
- Pull from New Relic / Splunk: `issueBulkRewardAPI` failures in the last 30 days, segmented by cluster (USHC, ASIA, EU, others) and by error type (`REDEEM_POINTS_API_EXCEPTION`, coupon-related failures, timeouts)
- Identify which failure code maps to "brand loses" vs "customer loses" — they may not be the same code in all clusters
- Check if `isRetryPointsRedeemEnabled` is enabled per cluster (env var / config server) — this changes which failure path is active
- Pull the reward-type mix per cluster: what % of transactions in USHC are vendor rewards vs coupon-only?

**What to produce:**
- A table: cluster × error class × count × retry flag state × vendor reward %
- A judgement: does the USHC asymmetry correlate with vendor reward % (H-6) or with a different retry flag state?

**Gate:** Without this table, Steps 2–5 may be investigating the wrong path. Confirm which failure class is dominant in USHC before proceeding.

---

### Step 2 — Audit Redis lock TTL vs processor execution time (H-1)

**Goal:** Determine whether the lock can expire while the processor chain is still running.

**What to do:**
1. Read `application.properties` (and all cluster-specific overrides / env vars) for:
   - `redis.lock.ttl` (or `REDIS_LOCK_TTL`)
   - `redis.lock.maxWaitTime` (or `REDIS_LOCK_MAX_WAIT_TIME`)
2. Read `VendorIssueProcessor` for `awaitTermination` / `VENDOR_RESPONSE_WAIT_TIME` value
3. Read `RedisLockRegistry` (Spring Integration) to confirm: does it auto-renew a held lock, or does it expire at TTL regardless?
4. Read `CustomerLockManager` / `CustomerLockable` to confirm: does the lock wrap the full `issueReward` call, or only part of it?
5. Measure: what is the actual p95 / p99 latency of the full `issueBulk` call in USHC? (New Relic APM trace)

**What to produce:**
- Confirmed values: `lock.ttl` (per cluster), `VENDOR_RESPONSE_WAIT_TIME`, whether Spring Integration auto-renews
- A yes/no: can the lock expire before `VendorIssueProcessor` completes?
- A yes/no: can the lock expire before `PointsRedeemProcessor` completes (even without vendor, given intouch latency)?
- If YES to either: H-1 is confirmed; note the exact TTL gap

**Gate:** If H-1 is confirmed and TTL values differ per cluster, record the per-cluster gap. The fix TTL value depends on actual measured latency, not assumptions.

---

### Step 3 — Trace timeout failure paths in rewards code (H-2, H-3, H-4)

**Goal:** Confirm exactly what state each timeout produces in `BulkRewardIssueContext` and what downstream effect follows.

#### 3a — Timeout Point A: `issueBulkCoupon` (H-2)

1. In `CouponIssueProcessor`: what does the `IOException` / `ExternalServiceException` catch block do to the wrapper? Does it set `successCount=0`?
2. In `BulkRewardIssueContext.getPointsToBeRedeemed()`: does filtering on `isAnySuccessful()` exclude this wrapper, causing `pointsToBeRedeemed = 0`?
3. In `PointsRedeemProcessor`: does `pointsToBeRedeemed == 0` cause an early return with no deduction?
4. Does any processor after the timeout attempt to revoke or reconcile the coupon that may have been issued?

**What to produce:** A confirmed yes/no for H-2 with the exact file:line evidence.

#### 3b — Timeout Point C: `redeemPoints` (H-3)

1. In `PointsRedeemProcessor`: what does the IOException catch block do?
2. When `isRetryPointsRedeemEnabled = false`: what status code is set? Does `shouldRevokeCoupons()` fire? Does `shouldRevokePromotions()` fire?
3. When `isRetryPointsRedeemEnabled = true`: what status code is set? Does revocation fire or is it suppressed?
4. In `PointsRedemptionFacade.retryPointsRedeem()`: at max retry, what happens? Is `getCustomerRedemptions` called before `revokeReward`? (H-5)

**What to produce:** A confirmed yes/no for H-3 and H-5 with exact file:line evidence.

#### 3c — Revocation trigger coverage (H-4)

1. In `BulkRewardIssueContext.shouldRevokeCoupons()` and `shouldRevokePromotions()`: what is the exact condition? Is it a single status code match or a broader condition?
2. List all status codes that `PointsRedeemProcessor` can set on failure. For each: does `shouldRevokeCoupons()` fire?
3. Are there any points failure codes that would NOT trigger revocation, leaving an un-revoked benefit?

**What to produce:** A list of status codes that bypass revocation (if any). A yes/no for H-4.

---

### Step 4 — Audit the async retry path (H-5)

**Goal:** Confirm whether the RabbitMQ retry path can produce revocation even when EMF already deducted points.

**What to do:**
1. Read `PointsRedemptionFacade.retryPointsRedeem()` end to end — specifically the max-retry branch
2. Is `getCustomerRedemptions(externalReferenceNumber)` called at every retry attempt, or only on success?
3. What is the condition that triggers `revokeReward()` at max retry? Does it check `getCustomerRedemptions` immediately before calling `revokeReward`?
4. What is the max retry count? Is it configurable per cluster?

**What to produce:** A flow diagram of the retry path showing every decision point and what state triggers revocation. A yes/no for H-5.

---

### Step 5 — Confirm EMF behaviour on timeout and idempotency (Downstream: EMF / intouch)

**Goal:** Determine whether "timeout on `redeemPoints` = EMF already committed" is actually the dominant mode, and whether the retry path is safe.

**What to ask EMF / intouch team:**

| Question | Why it determines the fix direction |
|----------|-----------------------------------|
| Does EMF commit the point deduction before intouch returns the HTTP response to rewards? Or does a timeout on the intouch side mean EMF never processed? | If EMF commits first → timeout = "points deducted but rewards unaware" → revocation is wrong. If EMF commits after → timeout = "EMF never ran" → revocation is correct. |
| Is `externalReferenceNumber` idempotent at EMF? If rewards sends the same value in a retry, does EMF return the existing redemption or process a second deduction? | If NOT idempotent → the entire retry path risks double deduction and must be redesigned. If idempotent → the retry path is architecturally safe; the gap is only the max-retry revocation. |
| Does `getCustomerRedemptions(externalReferenceNumber)` in intouch return results within the same request window as the EMF commit, or is there an eventual consistency lag? | If there is a lag → the pre-revocation check could return "not found" even when EMF committed → producing a false positive revocation. |
| What is the EMF thrift timeout? Is it shorter than the intouch HTTP timeout? | If EMF thrift timeout < intouch HTTP timeout → EMF has already finished (committed or errored) before intouch's HTTP layer times out. This is the "dominant mode" question. |

**What to produce:**
- A definitive answer to: does timeout at rewards = EMF committed or EMF never ran?
- A definitive answer to: is `externalReferenceNumber` idempotent?
- Record whether `getCustomerRedemptions` is strongly consistent or eventually consistent (with measured lag if applicable)

**Gate:** Steps 5 and 6 are the two highest-stakes downstream questions. If neither EMF team nor intouch team can confirm idempotency within the sprint, escalate to the engineering manager — the fix direction for H-3/H-5 is not determinable without this answer.

---

### Step 6 — Confirm luci commit timing and coupon query capability (Downstream: luci / intouch)

**Goal:** Determine whether Timeout Point A consistently leaves a committed coupon in luci, and whether rewards can reconcile state post-timeout.

**What to ask luci / intouch team:**

| Question | Why it determines the fix direction |
|----------|-----------------------------------|
| Does luci commit `issueCoupon` before intouch returns the HTTP response? Or can the intouch HTTP layer time out before luci has committed? | If luci commits first → every timeout at Point A = coupon in luci, no deduction. If sometimes luci hasn't committed yet → the inconsistency rate is lower than the timeout rate. |
| Is luci's `issueCoupon` idempotent per (customerId, seriesId, requestId)? | If idempotent → rewards can safely retry after timeout and get back the already-issued coupon. If not → retry risks issuing a duplicate coupon. |
| Does `POST /v2/coupon/bulk/issue` support an idempotency key header (e.g. `X-CAP-IDEMPOTENCY-KEY`)? | If yes → timeout recovery is clean: retry with same key, get existing coupon back. If no → recovery requires a query-based reconciliation. |
| Is there a query API on intouch/luci to check if a coupon was issued for (customerId, seriesId) in a given time window? | Required for the fallback recovery path if idempotency key is not supported. |

**What to produce:**
- A definitive answer on whether luci commits before the HTTP timeout window
- A yes/no on idempotency key support
- If no idempotency key: document the available query API (endpoint, params, response shape) for the fallback path

---

### Step 7 — Confirm Promotion Engine behaviour (Downstream: PE team)

**Goal:** Determine whether Timeout Point B has the same commit-before-response pattern as Timeout Points A and C.

**What to ask PE team:**

| Question |
|----------|
| Does PE commit the earn before returning the HTTP response? |
| Is `earnPromotion` idempotent per request ID? |
| Is there a query API to check earn status by request ID? |
| Does `revokeEarn` confirm earn existence before revoking, or is it a blind delete? |

**What to produce:** Same shape as Steps 5 and 6 — a yes/no for each question.

---

### Step 9 — Assess distributed transaction management feasibility

**Goal:** Determine whether formalising the transaction lifecycle into a recognised DTM pattern (Saga or TCC) is viable given downstream system capabilities, and whether it provides meaningful residual-risk reduction beyond the tactical root cause fixes.

**Important framing before starting this step:**
> The current code is already an attempted saga — issue benefit → deduct points → compensate on failure. The question is not "should we use a saga?" but "should we formalise the existing broken saga into one that has durable state, reliable compensation, and proper failure handling?" Step 9 assesses whether the downstream systems can actually support that formalisation.

**Note on sequencing:** Step 9 reuses findings from Steps 5, 6, and 7. Do not start it until those steps are complete — the downstream idempotency and compensation answers are direct inputs here.

---

#### 9a — Rule out 2PC immediately

Two-phase commit requires all participants to support a prepare/commit protocol. Answer one question:

> Do ALL participants in the chain — intouch, luci, EMF, Promotion Engine, AND external vendor APIs — expose a prepare/commit API?

External vendor APIs are a black box; no protocol negotiation is possible. **If any vendor reward type is in scope → 2PC is ruled out without further analysis.** Record this as a one-line finding and move on.

---

#### 9b — Assess TCC (Try-Confirm/Cancel) viability

TCC requires each participant to support three phases: reserve the resource, confirm it, or cancel the reservation. This is stronger than idempotency — it requires a "hold" phase that doesn't actually commit until a confirm arrives.

For each participant, answer: **does it support a hold/reserve operation that can be later confirmed or cancelled?**

| Participant | Question to answer | Source |
|------------|-------------------|--------|
| Intouch / luci (`issueBulkCoupon`) | Is there a "reserve coupon" API that holds but does not issue until confirmed? | Ask luci team |
| EMF / intouch (`redeemPoints`) | Is there a "reserve points" (escrow) API that holds points without deducting until confirmed? | Ask EMF team |
| Promotion Engine (`earnPromotion`) | Is there a "reserve earn" API? | Ask PE team |
| Vendor APIs | Any hold/cancel protocol? | Vendor API docs |

**What to produce:**
- A table: each participant → TCC supported (YES / NO / PARTIAL)
- If ANY participant is NO → full TCC is not viable; note whether partial TCC is worth pursuing for the participants that do support it
- If all core participants (intouch, EMF, PE) say YES but vendor says NO → TCC viable for non-vendor reward paths only; vendor path stays on tactical fix

---

#### 9c — Assess Saga (Orchestration-based) viability

Saga orchestration is the most realistic candidate. It does not require downstream hold phases — each step either succeeds or triggers a compensating transaction. Assess on five dimensions:

**Dimension 1 — Compensation completeness**

For each step in the chain, a reliable compensating transaction must exist. Reuse Step 5/6/7 findings:

| Forward step | Compensating transaction | Already exists in code? | Idempotent? | Reliable? |
|-------------|--------------------------|------------------------|-------------|-----------|
| `issueBulkCoupon` (luci) | `revokeCoupon` (luci) | Yes — `CouponRevokeProcessor` | Confirm with luci | Confirm with luci |
| `earnPromotion` (PE) | `revokeEarn` (PE) | Yes — `RevokeEarnProcessor` | Confirm with PE | Confirm with PE |
| `redeemPoints` (EMF) | `reversePoints` / none? | No reversal exists currently | N/A | N/A |
| Vendor API call | Cancel/refund API | Varies per vendor | No — black box | Likely NO |

> Key gaps to confirm: Does EMF have a `reversePoints` API? If the deduction succeeds but a later step fails, can EMF reverse it? Currently rewards relies on *not deducting* (via revocation before deduction) rather than *reversing* a completed deduction. A saga model reverses, not prevents.

**Dimension 2 — Pivot transaction identification**

In a saga, the "pivot transaction" is the last step that cannot be compensated. Everything before it is compensatable; everything after must be retried until it succeeds (cannot be rolled back).

Identify the pivot: **vendor API call is the likely pivot** because external vendors have no compensation protocol.

This has a structural implication: if the vendor step is the pivot, all compensatable steps (coupon issue, promotion earn) must come **before** vendor in the chain — which is NOT the current order. The current order is: Coupon [14] → Promotion [15] → Vendor [16] → Points [17]. Points deduction happens after vendor and is therefore on the "must-retry" side.

Confirm or challenge this by checking: does any vendor API support a cancel operation? If yes for some vendors, the pivot position may shift for those reward types.

**Dimension 3 — Saga state store**

A saga orchestrator needs durable, queryable saga state: which steps completed, which failed, which compensations were executed. Assess:

- MongoDB is already used in rewards (`TransactionActionLogs`). Is its schema and operational setup suitable for saga state (frequent small writes, query by sagaId, TTL cleanup)?
- Is `TBL_TRANSACTION` (MySQL) a better fit for saga state given it is already the authoritative transaction record?
- What is the maximum saga lifetime (from first step to final compensation or confirmation)? This drives TTL and storage sizing.

**Dimension 4 — Compensation failure handling**

A saga must define what happens when a compensating transaction itself fails (e.g., `revokeCoupon` times out). In the current code this is unhandled — `DownstreamFailureLog` captures it but nothing acts on it.

Assess: is there infrastructure for a "dead-letter saga" — a saga that is stuck and needs manual intervention or escalation? This is an operational requirement, not just a code requirement.

**Dimension 5 — Framework vs custom state machine**

Two implementation paths:

| Option | What it is | Operational cost |
|--------|-----------|-----------------|
| Custom state machine in rewards service | Saga state persisted to MongoDB; processor chain drives transitions; RabbitMQ for async steps | Low infra overhead; high implementation effort to get right |
| Axon Framework / Eventuate Tram | Purpose-built saga orchestration; handles compensation retry, idempotency, state store | Adds a framework dependency; team must learn and operate it |

For this assessment: confirm whether the team has appetite for a framework dependency, or whether a custom state machine is preferred. This is a product/engineering decision, not a technical one — flag it as a decision point.

**What to produce from 9c:**
- Dimension 1: compensation completeness table filled in — is there a gap (e.g., no `reversePoints` in EMF)?
- Dimension 2: pivot transaction identified; current processor order assessed against saga ordering constraint
- Dimension 3: saga state store recommendation (MongoDB vs MySQL vs Redis)
- Dimension 4: dead-letter saga operational plan — yes it exists / no gap / needs design
- Dimension 5: framework vs custom — flag as open decision for the team

---

#### 9d — Residual risk comparison

**Goal:** Determine whether Saga adds enough residual risk reduction over the tactical root cause fixes to justify the complexity.

After completing 9a–9c, fill in this table:

| Failure scenario | Resolved by tactical root cause fix? | Resolved by Saga? | Residual risk if Saga NOT adopted |
|-----------------|--------------------------------------|-------------------|----------------------------------|
| Lock expiry during vendor execution | Yes (fix lock TTL) | Yes (saga state survives pod restart) | None if lock TTL fix is correct |
| Coupon timeout → free benefit | Partial (depends on downstream idempotency) | Yes (saga queries state and compensates reliably) | Remains if intouch has no idempotency key |
| Points timeout → revocation | Partial (pre-revocation check helps) | Yes (saga retries deduction until confirmed) | Remains if EMF not idempotent |
| Compensation failure (revoke itself times out) | No (DownstreamFailureLog is passive) | Yes (saga retries compensation) | Permanent inconsistency path |
| Vendor API — no compensation | No | No | Gap in both approaches |

**Decision gate:** If the residual risk column shows significant gaps even after tactical fixes — specifically if intouch has no idempotency key AND EMF is not idempotent — the case for Saga strengthens significantly. If both idempotency questions are confirmed YES, tactical fixes alone close most gaps and Saga becomes an optional architectural improvement rather than a necessity.

---

#### 9e — Recommendation output

Produce one of four recommendations:

| Recommendation | Condition |
|---------------|-----------|
| **Saga — proceed to design** | Compensation is complete for all non-vendor steps; downstream confirms idempotent compensations; EMF has a `reversePoints` API or equivalent; pivot transaction is correctly positioned |
| **Saga — proceed with scope limit** | Compensation is complete for intouch/EMF/PE steps but vendor APIs cannot be covered; recommend Saga for non-vendor reward paths and tactical lock fix for vendor path |
| **Tactical fixes sufficient** | Downstream confirms idempotency on all paths; pre-revocation checks close the timeout gap; no compensation gaps; Saga complexity not justified |
| **Saga — blocked pending downstream** | Compensation table has critical gaps (e.g., no `reversePoints` in EMF); cannot proceed until EMF exposes a reversal API |

---

### Step 8 — Synthesise: map confirmed hypotheses to observed error classes

**Goal:** Produce a clean root cause → observed error mapping that directly drives the scope of the next story.

**What to do:**
- For each hypothesis (H-1 through H-6): mark CONFIRMED, PARTIAL, or RULED OUT based on evidence from Steps 1–7
- For each CONFIRMED hypothesis: state which failure class it produces (brand loses / customer loses / both)
- For each CONFIRMED hypothesis: note whether the fix is in rewards code only, requires downstream change, or requires both
- Produce the final error volume attribution: what % of USHC's 0.714% error rate is explained by each confirmed root cause?

---

## 6. Open Questions

> These must be answered during the analysis, not after.

| # | Question | Step | Owner | Blocker? |
|---|---------|------|-------|---------|
| Q1 | What is `REDIS_LOCK_TTL` in USHC / ASIA clusters (env var may override default)? | Step 2 | Rewards infra | Yes — determines if H-1 is active |
| Q2 | Is `isRetryPointsRedeemEnabled` enabled in USHC / ASIA? | Step 1 + 3 | Rewards team | Yes — determines which H-3 path is active |
| Q3 | Does EMF commit before intouch HTTP responds? Is `externalReferenceNumber` idempotent? | Step 5 | EMF / intouch team | Yes — determines if H-3/H-5 fix is safe |
| Q4 | Does `getCustomerRedemptions` have eventual consistency lag? | Step 5 | Intouch / EMF team | Yes — determines if pre-revocation check is reliable |
| Q5 | Does luci commit `issueCoupon` before intouch HTTP responds? | Step 6 | Luci / intouch team | Yes — determines if H-2 is consistently reproducible |
| Q6 | Does `POST /v2/coupon/bulk/issue` support idempotency key? | Step 6 | Intouch team | Yes — determines recovery mechanism for H-2 |
| Q7 | What are the actual intouch HTTP timeouts and luci/EMF thrift timeouts? Are they aligned? | Step 5 + 6 | Intouch infra | No — needed for sizing but not a fix blocker |
| Q8 | What is the USHC reward-type mix (vendor % vs coupon %)? | Step 1 | Rewards / analytics | No — needed to confirm H-6 |
| Q9 | Does EMF expose a `reversePoints` (deduction reversal) API? If not, is one on the roadmap? | Step 9c | EMF team | Yes — without this, Saga compensation for points deduction is not possible and Saga scope is limited |
| Q10 | Are compensating transactions (`revokeCoupon`, `revokeEarn`) idempotent? If called twice for the same transaction, does the second call error or silently succeed? | Step 9c | Luci / PE teams | Yes — non-idempotent compensations make Saga unsafe (double compensation = double revoke) |
| Q11 | Do any vendor APIs support a cancel / refund operation for a previously issued reward? | Step 9c | Vendor API docs / vendor account managers | No — informational; determines pivot position but vendor path is expected to have no compensation |
| Q12 | Does the team have appetite for an Axon / Eventuate framework dependency, or is a custom saga state machine in MongoDB preferred? | Step 9e | Engineering / product decision | No — implementation approach only; does not change feasibility assessment |

---

## 7. Exit Criteria — Analysis Complete

This story is complete when **all** of the following are true:

- [ ] Steps 1–9 executed and documented
- [ ] Every hypothesis (H-1 through H-8) marked CONFIRMED, PARTIAL, or RULED OUT with evidence
- [ ] Q1–Q11 answered with documented responses from relevant teams (Q12 flagged as a team decision)
- [ ] Root cause → error class mapping table populated (Step 8)
- [ ] DTM recommendation produced (one of the four options in Step 9e) with justification
- [ ] "Handoff to Tech Detail" section (§8) filled in with confirmed scope, covering both tactical fixes and DTM direction

---

## 8. Handoff to Tech Detail Story

> To be filled in at the end of this analysis. Stub only until then.

### 8a. Confirmed Root Causes
*[Populate after Step 8 — list only CONFIRMED hypotheses with their evidence pointers]*

### 8b. Fix Scope per Root Cause
*[Populate after Step 8 — for each confirmed RC: what system owns the fix, what is the constraint]*

### 8c. Downstream Contracts Needed
*[Populate after Steps 5–7 — list any API / contract changes required from intouch, luci, EMF, PE]*

### 8d. Constraints on the Solution
*[Populate after analysis — e.g. "EMF is NOT idempotent → retry path must not re-send redeemPoints; must reconcile via query only"]*

### 8e. Known Non-Goals for the Fix Story
*[Populate after analysis — e.g. "Promotion Engine timeout recovery deferred — PE does not support idempotency key"]*

### 8f. DTM Recommendation and Scope
*[Populate after Step 9 — record the recommendation (Saga proceed / Saga scoped / Tactical only / Saga blocked), the key condition that drove it, and the implementation path (framework vs custom state machine)]*

**Compensation gap summary:**
*[Populate after Step 9c — list any steps that do NOT have a viable compensating transaction, e.g. "EMF has no reversePoints API — Saga cannot cover the points deduction step"]*

**Pivot transaction:**
*[Populate after Step 9c — identify which step is the pivot and whether the current processor order needs to change to accommodate Saga ordering constraints]*

**Saga state store:**
*[Populate after Step 9c — MongoDB or MySQL, with rationale]*
