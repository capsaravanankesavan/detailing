---
title: "Rewards ↔ Audience Manager: Consolidated Integration Design"
subtitle: "Requirements → use cases → 1-* problems → data-modeling journey → AM architect discussion → final 1-to-1 design"
---

## 1. Background & Requirement

Rewards wants to let brand admins restrict a reward's visibility in the Catalogue to users who belong to a specific Audience Group (or groups) managed in Audience Manager (AM). This requires:
- A way to **link** a reward to one or more audience groups.
- A way, during **List Catalogue**, to check whether the current user belongs to any of the reward's linked groups, and show/hide the reward accordingly.
- Checking membership requires AM's **lookup** mechanism, which in turn requires a **subscription** (`POST /v1/audience/subscribe`, LOOKUP scope) to be created and asynchronously materialized (via RBS) before it can be queried.

From a discussion with Product, two explicit requirements emerged:

> **R1 (hassle-free launch):** A brand admin should be able to set up a reward's audience linkage such that the reward is **launched only when all its linked audience groups are `READY`** — not launched half-configured.

> **R2 (no mixed state on edit):** When an admin edits a reward's linked audience groups (e.g. `R1: [AG1, AG2, AG3]` → `[AG1, AG4, AG5]`), the reward must **never** serve an in-between mixed combination — e.g. never `[AG1, AG4]` while AG5 is still materializing, never `[AG1, AG2, AG5]` with a stale AG2 left over. It is either the **complete old set** or the **complete new set**, atomically — nothing in between.

This document walks through the full design journey: the use cases, why a naive 1-to-many (`Reward → *AudienceGroup`) relationship creates hard problems for R1/R2, the data-modeling path we explored (culminating in a proposed `reward_audiences`/`audience_group_set` entity), the discussion with the AM architect that reshaped this, and the final design.

## 2. Use Cases

| UC | Description |
|---|---|
| UC1 | Brand admin **creates** a reward and links it to one or more audience groups. Reward should not go live/visible until all linked groups are ready. |
| UC2 | Brand admin **edits** an existing (already-live) reward's linked audience-group list — adding, removing, or swapping groups. The reward must keep serving its **current, complete** group list until the **new, complete** group list is fully ready, then atomically swap. |
| UC3 | **Catalogue read:** for a given user, determine which of their eligible rewards to show, by checking group membership for each reward's linked group(s). |
| UC4 | **Health monitoring:** detect when a linked (or pending) audience group's lookup gets stuck in `ERROR`/`PROCESSING` beyond an acceptable window, and alert/ticket rather than blindly retry. |
| UC5 | Brand admin needs **visibility** into the status of a reward's audience linkage (per-group status, overall readiness) without needing to check AM directly. |

## 3. First approach: `Reward(1) --(*)--> AudienceGroup` — and why it breaks down

The most literal reading of the requirement is a plain many-to-many / one-to-many: a reward links to a list of audience-group IDs directly.

### 3.1 Problems this creates for **create** (UC1 / R1)

- AM's subscription/lookup lifecycle is **per group**, entirely independent across groups (verified: `AudienceGroupLookupMapping` has no cross-group relationship, no batch/group-of-groups status). If a reward links to `[AG1, AG2, AG3]`, Rewards must subscribe to all three, and each materializes on its own async timeline. Some might already be `READY` (previously used elsewhere), others `PROCESSING` for minutes.
- To satisfy R1 ("launch only when ALL are ready"), Rewards needs to **track a composite state** — not "is AG_i ready" (that's just one AM subscription's status) but "**are all N of them ready simultaneously**" — a fact that doesn't exist anywhere in AM. AM has zero notion of "this group of groups is jointly ready." Rewards would have to build and maintain that aggregate state itself, including watching for whichever group finishes last.
- This immediately implies a **background polling loop** that watches every group in every pending reward's list until the *slowest* one catches up.

### 3.2 Problems this creates for **edit** (UC2 / R2) — the harder one

This is where a flat list truly breaks:
- If `reward.linked_groups` is literally the mutable list `[AG1, AG2, AG3]`, then editing it to `[AG1, AG4, AG5]` means either:
  - **(a) Mutate the same rows in place** — remove AG2/AG3, add AG4/AG5. But AG4/AG5 aren't `READY` yet the instant they're added. If Catalogue reads `reward.linked_groups` live during this window, it sees a **half-updated list** — exactly the forbidden mixed state R2 rules out. There is no atomic way to "add 2, remove 2" in a way that's invisible to concurrent readers unless the whole list swap is versioned.
  - **(b) Stage the new list separately, cut over only when ready** — but this requires the reward to simultaneously "remember" its old, currently-serving list AND its new, pending list, and only flip which one is authoritative once the *entire* new list is confirmed ready. A flat list field cannot represent "two versions of the same list coexisting, one live, one pending" — you need a **new structural concept**, not just a mutable column.
- Compounding this: unsubscribing from a dropped group (AG2/AG3) **must not happen** until the cutover is confirmed — if the swap fails partway and rewards falls back to the old list, but AG2/AG3 were already unsubscribed, the "old" list is no longer actually functional. So teardown timing is coupled to the cutover success, which a flat list has no way to express.

### 3.3 State-management problems, summarized

The flat 1–* model conflates two different lifecycles that need to be tracked separately, and has no natural home for either:
1. **Per-group materialization state** (from AM: PROCESSING/ACTIVE/ERROR) — fine, AM gives you this per group.
2. **Per-edit-generation composite state** ("is this WHOLE set of groups, as a unit, ready to become the live set") — AM gives you nothing for this; it has to be modeled and tracked entirely in Rewards, and a flat mutable list gives you no place to put "the pending generation" separately from "the live generation."

Without solving #2, you cannot honor R2. This pushed us toward inventing a new structural entity.

## 4. The `reward_audiences` (a.k.a. proposed `audience_group_set`) concept — our data-modeling path

To solve the state-management gap in §3.3, we designed a new, admin-invisible internal entity representing **one versioned, atomic set of audience groups**:

```
reward_audiences (
  id,
  reward_id,
  status,            -- PENDING_ACTIVATION | ACTIVE | ARCHIVED
  created_at,
  activated_at
)
reward_audience_group (
  reward_audiences_id,
  audience_group_id,
  last_known_status  -- cache of AM's subscription status for this pairing
)
```

Relationship: `Reward(1) --(*)--> RewardAudiences`, and `RewardAudiences(1) --(*)--> AudienceGroup` (via the join table above). Key idea: **every edit mints a brand-new `reward_audiences` row** (a new "generation") rather than mutating the live one in place.

### 4.1 How this solves create (UC1)
- On reward create, a `reward_audiences` row is created `PENDING_ACTIVATION`, subscribe() is called for every linked group (using `reward_audiences.id` — **not** the reward's own ID — as the AM subscription's `entityId`, so this generation's subscriptions are independently trackable from any other generation).
- A background job polls `/v1/audience/subscription` for every group under this `reward_audiences` row until **all** report `ACTIVE`. Only then does the row flip to `ACTIVE` and the reward becomes visible in Catalogue.

### 4.2 How this solves edit (UC2 / R2)
- Editing creates a **new** `reward_audiences` row (`PENDING_ACTIVATION`) with the new group list, subscribing each group using the **new** row's id as `entityId` — a completely independent AM subscription lineage from the old (currently `ACTIVE`) row.
- The **old** `reward_audiences` row keeps serving Catalogue reads, unchanged, for the entire materialization window.
- Once **all** groups under the new row report `ACTIVE`, the swap is atomic: new row → `ACTIVE`, old row → `ARCHIVED` (which triggers unsubscribing groups no longer used — reference-counted, since a group might be shared across multiple rewards).
- **Never a window with a mixed set** — Catalogue always reads from exactly one `reward_audiences` row's group list at a time, and that row is always either fully-old or fully-new.
- One efficiency nicety we verified: AM's lookup materialization is per **group**, not per **(group, entityId)** — so a group common to both old and new lists (e.g. `AG1` in `[AG1,AG2,AG3]→[AG1,AG4,AG5]`) is already `READY` from the old generation's subscription; subscribing it again under the new generation's id short-circuits immediately. Only the genuinely new groups (`AG4`, `AG5`) gate the cutover.

### 4.3 Why `entityId = reward_audiences.id` (not `reward_id`) is the key design choice
AM's subscription uniqueness key is `(orgId, groupId, entityName, entityId)`. Using `rewardId` as `entityId` would only ever allow **one** subscription per (reward, group) pair — no way to represent "the reward's OLD generation's subscription" and "the reward's NEW generation's subscription" coexisting during the transition. Minting a fresh id per edit gives each generation an independent AM subscription lineage, which is exactly the mechanism that makes "old keeps serving while new materializes" work.

### 4.4 Remaining open edge cases in this model (identified, not yet resolved)
- **Concurrent edit while a previous edit is still pending** — does a second edit block, or abandon (immediately unsubscribe) the first pending generation?
- **Cancel/rollback of a pending edit** — same abandon-and-unsubscribe path, needs an explicit trigger.
- **Permanently-stuck group** — if a new group never reaches `ACTIVE` (a Databricks-class permanent failure per AM's team), the edit silently never completes; needs to surface via the alert/threshold mechanism (UC4) and be visible to the admin (UC5).
- **Archive-unsubscribe ordering** — must happen strictly after confirmed cutover, never before/concurrently.

### 4.5 Whether polling is really needed here
We explicitly revisited "can the brand admin just manually confirm readiness instead of Rewards polling automatically?" AM has **no webhook/push mechanism** — the only way to learn a group's status is to ask `/v1/audience/subscription`. Making this manual (admin watches a status page, clicks "activate" when ready) trades an automated cutover for admin babysitting, and directly conflicts with R1's "hassle-free" ask — it also doesn't cleanly satisfy R2 for edits, since nothing forces the admin to ever complete the edit if they walk away. **Conclusion: a small, bounded, scoped background poll** (only while `PENDING_ACTIVATION`, stops once `ACTIVE`) is the right mechanism — not the sprawling "poll everything forever" of a naive design, but not eliminable either.

## 5. Discussion with the AM Architect — and its outcome

While the `reward_audiences` model above technically works, we brought it to the AM architect to sanity-check whether "a set of audience groups treated as one unit" was really Rewards-domain logic, or whether it more naturally belonged inside AM. This surfaced several important facts that reshaped the design:

### 5.1 AM already has a "Parent/Child Audience" concept — but it doesn't fit
AM has a **Parent Audience / Child Audience** hierarchy: a child audience has exactly one parent, is tightly validated (org match, type match, depth ceiling, cycle detection — confirmed in `HierarchyValidator`), and is synced by combining the parent's and child's filter rules via an external NFS service. We verified:
- **Single parent only** — `AudienceGroup.parentGroupId` is a single field, not a list; a child cannot belong to multiple parents.
- **Combine semantics are AM-external** — AM delegates the actual filter-combine to NFS; AM's own code describes it as "concatenating filter target-blocks," not confirmed as strict AND/intersection at the AM layer.
- **Sync is event-driven, not a fixed daily cron** — a parent edit cascades to descendants via a job queue; there's no periodic re-evaluation cron.
- **Conclusion: doesn't fit.** This is a rigid, validated AND-composition tree for a completely different use case (nested audience refinement) — not a fit for "an arbitrary, admin-chosen set of independent groups per reward."

### 5.2 AM has no live OR/union primitive
We asked specifically: could a brand ever want "AG1 **OR** AG2" (not AND) as a single reward's eligibility rule? AM does have a `MERGE` operation (`Operator.UNION/INTERSECT/SUBTRACT` via `DerivedGroupExpression`) that combines independent groups — **but it produces a static, materialized user-list snapshot** (a batch file merge), not a live filter that a `subscribe()`/lookup call resolves dynamically at query time. `MERGE` groups are explicitly excluded from the hierarchy, and cannot be evaluated live the way `FILTER_BASED`/`REALTIME_FILTER` groups can.
- **Implication:** even if Rewards built its own `reward_audiences` set concept, it could only ever express AND-style "must match this exact set of live groups" semantics for the membership check itself (via intersecting `listAllGroups` results) — it **could not** solve a brand's "AG1 OR AG2" ask, because that fundamentally requires a *live*, filter-level OR primitive that doesn't exist anywhere yet, in AM or Rewards.

### 5.3 One AudienceGroup = one rule, always
We confirmed `AudienceGroup.uuid` is a single field — never a collection of rules. Multiple rules are always modeled as **multiple groups**, composed externally (hierarchy AND, or MERGE snapshot) — never as multiple rule definitions attached to one group.

### 5.4 The architect's and our joint conclusion
Given §5.1–§5.3, we agreed:
1. **A "set of audience groups treated as one live-evaluatable unit" is conceptually an Audience Manager–domain entity, not a Rewards-domain one.** We're calling it (working name) **`AudienceGroupSet`** on the AM side — the natural home for solving "a group of groups, evaluated live, potentially with OR semantics" is inside AM, next to where `AudienceGroup`, the hierarchy, and MERGE already live — not reinvented inside Rewards as `reward_audiences`.
2. **This entity does not exist yet in AM.** Building it (with real live-OR support) is future AM roadmap work, not something to block this design on.
3. **A background poll/cron would still be needed even if this entity lived in AM** — someone still has to call `unsubscribe`/manage lifecycle on the *set* as a unit, and AM itself has no push mechanism today (fact §2.1) — so moving the set concept into AM doesn't eliminate the polling problem, it just relocates where the set's *membership rules* live. The operational polling concern is orthogonal to which domain owns the entity.
4. **Until `AudienceGroupSet` exists in AM, restrict Rewards↔Audience to a strict 1-to-1 relationship** — `Reward(1) --(1)--> AudienceGroup`. This is deliberately narrower than what Product originally asked for (a set of groups), but:
   - It matches AM's own data-model grain (one group, one rule — §5.3).
   - It **structurally eliminates** the R2 "never a mix" problem: with only one group, there's no combination to mix. An edit becomes a single atomic pointer swap in Rewards' own DB.
   - It **structurally eliminates** the runtime "wait for all N to be ready" polling-driven cutover gate — R1 is satisfied by moving the readiness check to **before** linking is allowed, rather than after.
   - **Trade-off, explicitly accepted:** the brand admin must fully set up the audience group in AM (lookup enabled, reaches `READY`) **before** Rewards will let them link it to a reward. This is real friction, and a genuine change from "link now, becomes active automatically" — flagged to Product for sign-off (§8).

## 6. Final Design: 1-to-1 `Reward ↔ AudienceGroup` (with a documented path to `AudienceGroupSet`)

### 6.1 Data model

```
reward(
  id,
  audience_group_id,              -- single FK, nullable if unlinked
  audience_group_linked_at,
  audience_group_last_status,     -- READY | PROCESSING | ERROR | unknown (cached)
  audience_group_last_checked_at,
  audience_group_alert_raised_at  -- null until threshold alert/ticket fires; suppresses duplicate tickets
)
```

Deliberately simple — no per-generation/versioning complexity, because there's exactly one group per reward, and AM's own group-level subscription dedup (verified: `LookupService.enableLookup()` is idempotent per group; multiple entities subscribing to the same group don't create duplicate RBS rules, and unsubscribing one doesn't tear down lookup for others still using it — reference-counted via `hasActiveLookupSubscriptions`) already prevents duplication if multiple rewards happen to share a group.

**Forward-compatible note:** when `AudienceGroupSet` eventually exists in AM, this same `audience_group_id` column can be widened to optionally reference a set instead of a single group (or a parallel `audience_group_set_id` column added) — the 1-to-1 shape of the relationship (one reward → one linkable "audience thing," whatever that thing is) is preserved either way; only what's on the other side of that FK changes.

### 6.2 Create flow (UC1) — brand admin experience

1. Brand admin goes to **Audience Manager UI** first, creates/configures the audience group, enables lookup, and **waits for AM to report it `READY`**. (This step is entirely on AM's UI/side — outside Rewards.)
2. Brand admin goes to **Rewards UI**, creates a reward, and picks an audience group to link.
3. **Rewards UI queries the group's current status** (via a cached read of `/v1/audience/subscription`, or Rewards' own last-known-status cache) and only allows the "Link" action to succeed if the group is `READY`. If not `READY`, the UI shows a clear blocking message (e.g. *"This audience isn't ready yet — finish setting it up in Audience Manager and try again once it's active"*) rather than silently accepting a not-yet-ready link.
4. On successful link, Rewards calls `subscribe()` (`entityName="REWARD"`, `entityId=<rewardId>`) — this is effectively a confirmation/no-op materialization call at this point (fact §5, group-level idempotency), since the admin's AM-side setup already triggered and completed materialization.
5. Reward is immediately eligible to show in Catalogue for matching users — **no waiting, no polling, no intermediate state** on the Rewards side, because readiness was already confirmed before the link was permitted.

### 6.3 Edit flow (UC2) — brand admin experience

1. Brand admin opens an existing, live reward and chooses a **different** audience group to link (or unlinks).
2. Same gate as create: the **new** target group must already report `READY` in AM before Rewards allows the swap.
3. Until the admin picks a ready replacement, **the reward keeps serving against its current, unchanged group** — there is no in-between state, because the swap is a single field update (`reward.audience_group_id = new_group_id`), executed only once the gate passes. There is nothing to "roll forward" or "roll back" mid-transition, because the transition itself never begins until the target is already valid.
4. On a confirmed swap, Rewards subscribes to the new group (again, typically a no-op confirmation per group-level idempotency) and — after the swap is durably committed — unsubscribes from the old group **only if no other reward still references it** (reference-counted at the Rewards level, mirroring AM's own reference-counting for lookup teardown).

### 6.4 Catalogue read flow (UC3) — unchanged core mechanism
- Rewards calls `GET /v1/lookup/user/{userId}/groups` (AM's "list all groups this user belongs to" endpoint) once per user, cached 5–15 minutes, and checks whether the reward's single linked `audience_group_id` is in that list.
- This endpoint is chosen specifically because it **never fails the whole request** over one bad/pending group (unlike AM's per-group bulk-status endpoint, which throws HTTP 400 for the entire call if any one requested group isn't `READY`) — so a reward whose linked group later regresses degrades gracefully to "not shown," never breaking the catalogue page.
- A genuinely-empty/non-matching result → reward not visible (fail-closed). An **exception** from the call (AM/RBS outage) is handled separately — via a circuit breaker + last-known-good cache fallback — never conflated with "user not a member."

### 6.5 Post-link health monitoring (UC4) — the one thing that still polls
Even under 1-to-1 with a pre-link gate, an **already-linked** group's lookup can regress after linking (e.g., the admin edits the group's filter on AM's side later, and the new materialization errors). A lightweight background job periodically checks status for currently-linked groups and, if one is stuck `ERROR`/`PROCESSING` beyond a threshold (recommended ≥2 hours, aligned to AM's own hourly stuck-lookup reconciliation cron), raises an internal alert and a support ticket to the AM team — mirroring AM's own stated model of "we retry transient failures; permanent ones need a brand-raised ticket." This is unrelated to, and much lighter than, the create/edit-time gating logic — it never blocks any UI action, purely observability.

### 6.6 Brand admin visibility (UC5)
The reward's audience-linkage status (current group, its cached AM status, last-checked time, whether an alert is active) should be visible in the reward-management UI, so an admin can see "this reward's audience health" without needing access to AM directly — especially useful for catching a post-link regression (§6.5) before a customer notices the reward disappeared.

## 7. Non-goals (of this design, as scoped today)
- Modeling a set-of-groups or OR/union semantics inside Rewards — explicitly deferred to a future AM-side `AudienceGroupSet` (§5.4, §8).
- Any Rewards-side retry loop against `subscribe()` beyond a single bounded re-check — AM already retries transient failures; permanent failures need a ticket, not a retry loop.
- Any atomic multi-group cutover mechanism inside Rewards — moot under the 1-to-1 restriction.
- Changing AM's parent/child hierarchy or MERGE operation semantics.

## 8. Open items

**For Product (needs explicit sign-off):**
- The friction trade-off: admins must fully set up and reach `READY` in AM **before** Rewards allows linking — replacing the "link now, becomes active automatically" model originally described as the "hassle-free" goal (R1). Confirm this reframing is acceptable.
- Fail-closed-by-omission as the default when a linked group's status regresses post-link.
- ≥2-hour alert/ticket threshold for a post-link regression.
- Whether brand admins should see a lookup-health indicator in reward management proactively (UC5), ahead of a customer-facing complaint.

**For the AM team (forward-looking, not blocking this design):**
- Track `AudienceGroupSet` (a live-evaluatable set of audience groups, ideally with OR/union support at the filter level) as a future AM roadmap item — this is the durable home for "multiple groups per reward" once a brand genuinely needs it, rather than Rewards reinventing set semantics on top of 1-to-1 links.

## 9. Test Strategy (categories — full specs belong in the implementation plan)
- **Link/edit gate:** cannot link or swap to a group that isn't `READY`; can once it is; a swap is a single atomic field update with no observable intermediate state.
- **Catalogue visibility:** reward visible iff user is in its linked group per `listAllGroups`; hidden if absent/unresolvable.
- **Empty-vs-exception distinction:** genuinely-not-a-member vs. an AM/RBS exception handled and metriced separately.
- **Cache behavior:** hit/miss/TTL expiry; concurrent same-user requests coalesced; circuit-breaker fallback to last-known-good on AM failure.
- **Group-level dedup:** two rewards linking to the same group create no duplicate RBS rule; unsubscribing one reward doesn't disable lookup for another reward still referencing the group.
- **Post-link regression:** background job detects a `READY→ERROR` regression on an already-linked group; alert/ticket fires once per stuck link, not per poll.
- **Non-blocking guarantees:** reward create/update/link/edit succeeds even if a confirmation `subscribe()` call throws or times out (since readiness was already gated pre-action).
