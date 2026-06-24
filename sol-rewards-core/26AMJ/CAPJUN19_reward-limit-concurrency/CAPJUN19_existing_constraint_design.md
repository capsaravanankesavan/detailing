# Existing Reward Constraint Design — sol-rewards-core
**Date:** 2026-06-22
**Purpose:** Reference document for the CAPJUN19 concurrency fix. Captures the existing constraint model, evaluation flow, summary persistence logic, and the `buildUniqueSummaries()` date-matching behaviour in full detail.

---

## 1. Domain Model

### `TBL_REWARD_CONSTRAINT` — constraint configuration

```
RewardConstraint
├── id, orgId
├── rewardId               → specific reward ID, OR -1L for org-wide constraints
├── kpi                    → QUANTITY | POINTS | REDEMPTION_VALUE | TRANSACTION_COUNT (deprecated)
├── constraintLevel        → see Level taxonomy below
├── constraintAttribute    → optional: tier ID / program ID / label / segment (org-level only)
├── windowType             → FIXED | ROLLING
├── repeatFrequencyType    → DAYS | WEEKS | MONTHS | NO_LIMIT
├── intervalValue          → e.g. 2 (means "every 2 weeks")
├── limitValue             → hard cap (Integer)
├── cycleStartDate         → anchor date for FIXED windows (START_OF_CYCLE column)
├── weekStartDay           → for WEEKS FIXED windows
├── endDate                → constraint expiry; null = permanent
└── enabled                → soft-delete flag
```

**Constraint is active if:** `enabled = true` AND (`endDate > today` OR `endDate IS NULL`).

---

### `TBL_REWARD_ISSUE_SUMMARY` — running consumption ledger

```
RewardIssueSummary
├── id, orgId
├── rewardId               → mirrors constraint's rewardId (-1L for org-level)
├── kpi                    → mirrors constraint's kpi
├── level                  → mirrors constraint's constraintLevel
├── constraintAttribute    → mirrors constraint's constraintAttribute (nullable)
├── userId                 → customerId (null for REWARD/TRANSACTION level)
├── consumed               → BigDecimal (precision 13, scale 4) — cumulative KPI consumed
├── issueDate              → date this summary row covers (time stripped, org-TZ adjusted)
└── lastUpdatedOn          → used by Databricks delta ETL
```

**No unique constraint** on the natural key columns. Duplicate rows for the same `(orgId, rewardId, kpi, level, attribute, userId, issueDate)` are possible and do occur in production.

---

## 2. Constraint-Level Taxonomy

Two distinct groupings drive different code paths throughout the system:

| Group | Levels | `rewardId` in summary | `userId` in summary |
|---|---|---|---|
| **Reward-level** (`REWARD_CONSTRAINT_ALLOWED_LEVELS`) | `REWARD`, `CUSTOMER`, `TRANSACTION` | specific rewardId | null (REWARD/TRANSACTION) / customerId (CUSTOMER) |
| **Org-level** (`ORG_CONSTRAINT_ALLOWED_LEVELS`) | `CUSTOMER_REDEMPTION_TYPE`, `CUSTOMER_TIER`, `CUSTOMER_LOYALTY_PROGRAM`, `CUSTOMER_SUPPL_PROGRAM`, `CUSTOMER_LABEL`, `CUSTOMER_SEGMENT` | **-1L** (sentinel) | customerId |

The `-1L` sentinel for `rewardId` is defined at `Constants.ORG_LEVEL_REWARD_ID` and is used in every query that distinguishes org-level from reward-level summaries.

**`TRANSACTION` level** is deprecated. `LevelService.evaluate()` always returns `true` for it — no actual limit enforced.

---

## 2a. Configuration Rules (invariants enforced by the system)

| Rule | Detail |
|---|---|
| **FIXED and ROLLING cannot coexist at the same level** | For a given `(rewardId, constraintLevel, kpi)` tuple, only ONE of FIXED / ROLLING / NO_LIMIT is allowed. Configuring both FIXED and ROLLING at the same level+KPI is not supported. |
| **FIXED + ROLLING can coexist across levels** | Example: `CUSTOMER`-level FIXED window + `REWARD`-level ROLLING window on the same reward is valid because the levels differ. |
| **Org-level constraint count limits** | Max 50 org-level constraints total per org; max 15 for KPI=POINTS. |
| **NO_LIMIT independence** | A `NO_LIMIT` constraint at `(Level, KPI)` is "independent" only if no other windowed (FIXED/ROLLING) constraint exists at the same `(Level, KPI)`. Independent ones get their own summary row; dependent ones are skipped (the windowed constraint's row is sufficient). |

---

## 2b. KPI Types — What Each Tracks

| KPI | What it counts | Supported constraint levels | Notes |
|---|---|---|---|
| `QUANTITY` | Number of reward issuances | REWARD, CUSTOMER, org-level | Most common. `kpiValue = requestedQuantity` |
| `POINTS` | Intouch points redeemed at issuance | REWARD, CUSTOMER, org-level | `kpiValue = pointsPerUnit × quantity` |
| `REDEMPTION_VALUE` | Monetary value converted at issuance (CONV_RATIO rewards only) | REWARD, CUSTOMER **only** | Not supported at org-level — see below |
| `TRANSACTION_COUNT` | Count of transactions | (deprecated) | `evaluate()` always returns `true`; no limit enforced |

---

### `REDEMPTION_VALUE` — detailed behaviour

`REDEMPTION_VALUE` tracks the **monetary/cash value consumed** when a reward is fulfilled. It is only meaningful for rewards configured with `CONV_RATIO` payment mode, where the customer's Intouch points are converted into a redemption value (e.g. ₹50) that is passed to the vendor.

**How `redemptionValue` is derived — `PaymentConfigIssueProcessor` (runs before constraint check):**

```
CONV_RATIO payment mode — three derivation paths:

  Caller passes points:
    redemptionValue = requestedPoints × conversionRatio
    e.g. 500 pts × 0.10 = ₹50

  Caller passes redemptionValue directly:
    redemptionValue = requestedPaymentConfig.redemptionValue  (honoured as-is)

  Caller passes neither:
    redemptionValue = requestedQuantity  (quantity treated as monetary amount)
    quantity overridden to 1
```

`redemptionValue` is set on `RewardIssueWrapper` before `RewardConstraintProcessor` runs (processor chain position ~5 vs ~12).

**KPI value used in constraint check:**
```java
kpiValue = redemptionValue × quantity
// e.g. 2 vouchers each worth ₹50 → kpiValue = ₹100
```

**What the limit means:** "Total monetary value redeemed via this reward must not exceed `limitValue` in the configured window."
Example: CUSTOMER-level, MONTHLY FIXED, `limitValue = 5000` → customer can redeem at most ₹5000 through this reward per month.

**Org-level constraints do NOT support `REDEMPTION_VALUE`:**

`RewardConstraintProcessor.validateForOrgLevelConstraints()` explicitly passes `null` for `redemptionValue`:
```java
// ORG level constraint doesn't support REDEMPTION_VALUE KPI
BigDecimal kpiValue = context.getValidKPIValue(quantity, points, null, ctx);
```
This makes `kpiValue = null`, which causes `evaluate()` to return `null`, which triggers a hard exception.

**Config error — reward has no `CONV_RATIO` payment mode:**
If a `REDEMPTION_VALUE` constraint is configured on a reward that uses `POINTS` or `POINTS_CASH` payment mode, `rewardIssueWrapper.redemptionValue` is `null` at constraint check time. Flow:
```
getValidKPIValue() → returns null
setIsValidForRedemptionValue(null) → isValid = null  (tri-state error)
evaluate() → returns null
RewardConstraintProcessor → throws REDEMPTION_VALUE_KPI_RESTRICTION_WITHOUT_CONV_RATIO_PAYMENT_CONFIG
```
There is no upfront validation at constraint creation time — this config error surfaces only at issuance time.

---

## 3. Window Types and Cycle Calculation

### ROLLING window
```
startDate = today - (intervalValue × repeatFrequencyType.daysEquivalent)
endDate   = today
```
Query filters: `issueDate BETWEEN startDate AND today`

### FIXED window
Three concrete cycle calculators, each returning `FixedWindowCycle(startDate, endDate)` adjusted for org timezone:

| RepeatFrequencyType | Calculator | Cycle logic |
|---|---|---|
| `DAYS` | `DailyFixedWindowCycleCalculator` | Cycles every N days from `cycleStartDate`; current cycle = `cycleStartDate + N × floor(daysSinceStart / N)` |
| `WEEKS` | `WeeklyFixedWindowCycleCalculator` | Aligns to `weekStartDay` via `TemporalAdjusters.previousOrSame()`; 7-day cycle |
| `MONTHS` | `MonthlyFixedWindowCycleCalculator` | First of current month to first of next month |

All cycles trimmed by `constraint.endDate` via `Utils.adjustCycleForConstraintEndDate()`.

Query filters: `issueDate BETWEEN cycleStart AND cycleEnd`

### NO_LIMIT
No date filter. Queries lifetime consumption: `SUM(consumed) WHERE rewardId=? AND kpi=? AND level=?`

---

## 4. Evaluation Flow at Issuance Time

```
RewardConstraintProcessor.process(BulkRewardIssueContext)
│
├── validateForOrgLevelConstraints()
│     ├── fetch org constraints: rewardId = -1L, enabled, not expired
│     ├── for each constraint × reward combination:
│     │     1. levelService.loadKPI(userId, constraint)    → build RewardIssueSummaryContext
│     │     2. context.setOrgZoneId(orgZoneId)
│     │     3. levelService.refreshSummary(context)        → SUM(consumed) live MySQL read → context.consumedFromDB
│     │     4. kpiValue = context.getValidKPIValue()       → quantity / points / redemptionValue×qty
│     │     5. levelService.evaluate(context, kpiValue)    → kpiValue + context.consumed <= limitValue ?
│     │        PASS → context.consumed += kpiValue  (in-memory accumulator)
│     │        FAIL → mark ALL reward wrappers FAILED  ← org-level is all-or-nothing
│
└── validateForNonOrgLevelConstraints()
      for each reward in context:
        ├── fetch reward constraints: specific rewardId, enabled, not expired
        ├── for each constraint:
        │     1–5. same load → refresh → kpiValue → evaluate flow
        │        PASS → add constraint to rewardIssueWrapper.validRewardConstraintList
        │        FAIL → set StatusDto on that reward wrapper only  ← per-reward, not global
```

**Key asymmetry:** Org-level failure rejects the entire request batch. Reward-level failure rejects only that reward.

---

## 5. `RewardIssueSummaryContext` — per-constraint runtime state

```java
consumed        // starts as consumedFromDB; accumulates KPI across this request's constraints
consumedFromDB  // fetched once from DB via refreshSummary(), cached — never re-fetched within same request
limitValue      // from constraint config
isValid         // tri-state: true | false | null (null = error/unknown)
windowType      // ROLLING | FIXED
cycleStartDate  // anchor for FIXED window queries
```

Validation methods:
- `setIsValidForQuantity(qty)` → `qty + consumed <= limitValue`
- `setIsValidForPoints(pts)` → `pts + consumed <= limitValue`
- `setIsValidForRedemptionValue(val)` → `val + consumed <= limitValue`
- `setIsValidForTransaction()` → always `true` (deprecated level)

---

## 6. `updateSummaries()` — post-issuance persistence

Called after `issueBulk()` succeeds (all external calls completed). Entry point at `RewardConstraintFacade.java:328`.

### Step 1 — Compute dates

```java
trimmedIssualDate = Utils.getDateWithoutTimestampInSpecifiedZone(now(), orgZoneId)
trimmedEventDate  = rewardIssueWrapper.getEventDate() == null
                      ? trimmedIssualDate
                      : Utils.getDateWithoutTimestampInSpecifiedZone(eventDate, orgZoneId)
```

`eventDate` is the transaction event timestamp from the request. It can be in the past (backfill flows) or equal to `now()` for live issuances.

### Step 2 — Fetch existing summary rows

```java
// Org-level rows: rewardId = -1L, date = trimmedIssualDate
orgLevelSummaries = getExistingRewardIssueSummariesForOrgLevel(orgId, userId, trimmedIssualDate)

// Reward/customer-level rows: specific rewardId, date depends on FIXED window presence
isFixedWindowPresent = validConstraints.stream()
    .anyMatch(c -> REWARD_CONSTRAINT_ALLOWED_LEVELS.contains(c.level)
               && c.windowType == FIXED
               && c.repeatFrequencyType != NO_LIMIT)

fetchDate = isFixedWindowPresent ? trimmedEventDate : trimmedIssualDate
nonOrgLevelSummaries = getExistingRewardIssueSummariesForNonOrgLevel(orgId, rewardId, userId, fetchDate)
```

**Important:** The fetch date for non-org summaries switches to `trimmedEventDate` if ANY FIXED window constraint exists for this reward. This means ALL non-org summary rows for this request are fetched by event date when any constraint is FIXED — even rolling-window constraints on the same reward.

### Step 3 — `buildUniqueSummaries()` — merge existing + new

Input:
- `validRewardConstraints` — constraints that passed evaluation
- `rewardIssueSummaries` — existing DB rows (org + non-org, combined)
- `userId`, `trimmedIssualDate`, `trimmedEventDate`, `isFixedWindowPresent`

#### Step 3a — Prune stale org-level rows

```java
uniqueSet.removeIf(summary ->
    summary.rewardId == -1L &&
    validConstraints.noneMatch(c ->
        c.constraintLevel == summary.level &&
        c.constraintAttribute.equals(summary.constraintAttribute)
    )
)
```

Org-level summary rows that have no matching active constraint are dropped from the working set. This prevents accumulating orphaned rows when org-level constraints are disabled or deleted.

#### Step 3b — Identify independent NO_LIMIT constraints

`getIndependentRewardCustomerNoLimitRestrictions()` groups constraints by `(Level, KPI)`. A `NO_LIMIT` constraint is **independent** if it is the only constraint in its `(Level, KPI)` group — i.e., no sibling ROLLING or FIXED constraint with the same level + KPI exists.

| Scenario | Treatment |
|---|---|
| `(CUSTOMER, QUANTITY, NO_LIMIT)` — alone for that level+KPI | **Independent** — gets its own summary row |
| `(CUSTOMER, QUANTITY, NO_LIMIT)` + `(CUSTOMER, QUANTITY, DAYS)` | **Dependent** — NO_LIMIT row is SKIPPED; the DAYS/FIXED row covers it |

Rationale: if a FIXED or ROLLING constraint already tracks the running total for `(Level, KPI)`, writing a separate NO_LIMIT row would double-count.

#### Step 3c — Date selection per constraint

For each constraint in `validRewardConstraints`:

```
if DEPENDENT NO_LIMIT (reward/customer level):
    → SKIP (continue)

if INDEPENDENT NO_LIMIT (reward/customer level):
    expectedDate = isFixedWindowPresent ? trimmedEventDate : trimmedIssualDate

else if FIXED window (reward/customer level):
    expectedDate = trimmedEventDate

else (ROLLING, or org-level anything):
    expectedDate = trimmedIssualDate
```

**Decision matrix:**

| Constraint type | Window | expectedDate |
|---|---|---|
| Reward/Customer level | FIXED (DAYS/WEEKS/MONTHS) | `trimmedEventDate` |
| Reward/Customer level | ROLLING | `trimmedIssualDate` |
| Reward/Customer level | NO_LIMIT (independent) | `trimmedEventDate` if any FIXED present, else `trimmedIssualDate` |
| Reward/Customer level | NO_LIMIT (dependent) | SKIPPED |
| Org-level (any window) | any | `trimmedIssualDate` |

#### Step 3d — Row matching (triple-filter)

For each constraint, find an existing summary row that matches all three conditions:

```java
existingRow = uniqueSet.stream()
    .filter(rewardConstraint::isEquivalentToConstraint)  // filter 1: structural match
    .filter(summary -> summary.isValidUserId(userId))     // filter 2: userId match
    .filter(summary -> summary.issueDate == expectedDate) // filter 3: date match
    .findFirst()
```

**Filter 1 — `isEquivalentToConstraint(summary)`** (on `RewardConstraint.java:136`):

```
For REWARD_CONSTRAINT_ALLOWED_LEVELS:
  match if summary.level == constraint.level AND summary.kpi == constraint.kpi

For ORG_CONSTRAINT_ALLOWED_LEVELS (non-CUSTOMER levels):
  match if summary.level == constraint.level
       AND summary.kpi == constraint.kpi
       AND summary.constraintAttribute == constraint.constraintAttribute

For ORG CUSTOMER level:
  match if summary.level == constraint.level AND summary.kpi == constraint.kpi
```

Note: `rewardId` is NOT in the match — it's implicitly correct because the working set was fetched for the specific rewardId.

**Filter 2 — `isValidUserId(userId)`** (on `RewardIssueSummary.java:70`):

```
if level is in NON_CONSUMER_LIST (REWARD, TRANSACTION):
    → always true (no userId match needed)
if summary.userId is null OR userId is null:
    → false
else:
    → summary.userId == userId
```

**Filter 3 — date match:**
Exact `Date.compareTo()` equality against `expectedDate`.

#### Step 3e — Create new row if no match

If no existing row survives all three filters:
```java
newSummary = levelService.buildSummary(userId, constraint)  // creates entity with id=null
newSummary.setIssueDate(expectedDate)
uniqueSet.add(newSummary)
```

`id = null` tells `bulkSaveOrUpdate()` to INSERT rather than UPDATE.

### Step 4 — Set consumed and persist

```java
for each summary in uniqueSet:
    if kpi == QUANTITY:
        summary.setConsumed(BigDecimal.valueOf(successCount))
    else if kpi == POINTS:
        summary.setConsumed(intouchPoints × successCount)
    else if kpi == REDEMPTION_VALUE:
        summary.setConsumed(redemptionValue × successCount)

bulkSaveOrUpdate(summaries)
  → id == null  → bulkSave()   → plain INSERT
  → id != null  → bulkUpdate() → UPDATE SET CONSUMED = :computedValue WHERE ID = :id
```

`RewardIssueSummary.setConsumed(delta)` **adds** `delta` to the entity's existing `consumed` field (or sets it if null). So the value written to DB = `consumedAtFetch + delta`. This is the blind overwrite — `delta` was computed from the in-flight request's KPI value; `consumedAtFetch` was read at the JPA fetch step above with no row lock.

---

## 7. JPA Named Queries for Consumed Value

Eight queries covering every combination of `userId`, date filtering, and `constraintAttribute`:

| Query | userId | Date range | constraintAttribute | Used for |
|---|---|---|---|---|
| `fetchKPIConsumedValueWithUserIdNull` | IS NULL | yes | no | REWARD/TRANSACTION level, ROLLING/FIXED |
| `fetchKPIConsumedValueWithUserIdNotNull` | = :userId | yes | no | CUSTOMER level, ROLLING/FIXED |
| `fetchKPIConsumedValueWithUserIdNotNullAndConstraintAttribute` | = :userId | yes | yes | Org-level with attribute, ROLLING/FIXED |
| `fetchKPIConsumedValueForRewardId` | IS NULL | no | no | REWARD level, NO_LIMIT |
| `fetchKPIConsumedValueForRewardIdAndUser` | = :userId | no | no | CUSTOMER level, NO_LIMIT |
| `fetchKPIConsumedValueForRewardIdAndUserAndConstraintAttribute` | = :userId | no | yes | Org-level with attribute, NO_LIMIT |
| `fetchKPIConsumedValueForRewardLevelFixedWindow` | IS NULL | FIXED cycle dates | no | REWARD level, FIXED |
| `fetchKPIConsumedValueForCustomerLevelFixedWindow` | = :userId | FIXED cycle dates | no | CUSTOMER level, FIXED |

All return `SUM(consumed)` — they aggregate across any duplicate rows. This means duplicate rows in the summary table don't cause double-rejection at the constraint check stage (the SUM is still correct), but they do bloat the table and break the UPDATE path (since UPDATE targets a single `ID`, only one of the duplicate rows gets updated, the other drifts stale).

---

## 8. Complete Data Flow Diagram

```
issueReward API call
│
├── [1] CONSTRAINT LOAD
│     RewardConstraintRepository.findAllByOrgId...
│     → org-level (rewardId=-1L) + reward-level constraints
│
├── [2] EVALUATION (RewardConstraintProcessor)
│     For each constraint:
│       refreshSummary() → live MySQL SUM(consumed)  [no row lock]
│       evaluate()       → consumed + kpiValue <= limitValue ?
│       PASS → in-memory ctx.consumed += kpiValue
│       FAIL → reject (all rewards for org-level; this reward for reward-level)
│
├── [3] ISSUANCE (issueBulk)
│     CouponIssueProcessor / VendorIssueProcessor / PointsRedeemProcessor
│     → external calls (Intouch, vendors, points system)
│
└── [4] SUMMARY UPDATE (updateSummaries)
      fetch existing rows from TBL_REWARD_ISSUE_SUMMARY [JPA, no lock]
      buildUniqueSummaries()
        → prune stale org rows
        → identify independent NO_LIMIT constraints
        → for each constraint: pick date, find/create summary row
      set consumed = existingConsumed + thisDeltaKPI  [not atomic]
      bulkSaveOrUpdate()
        → id=null  → INSERT (plain, no ON DUPLICATE KEY)
        → id!=null → UPDATE SET CONSUMED=:value  (blind overwrite)
```

---

## 9. Known Design Gaps (targeted by CAPJUN19)

### Gap A — Wrong lock granularity
- Lock key: `orgId + "_" + customerId`
- Two requests from **different customers** for the **same reward** are not serialized
- Both enter step [2] simultaneously, both read the same `SUM(consumed)`, both pass

### Gap B — Concurrent INSERT (no DB unique constraint)
- Two threads reach `buildUniqueSummaries()` simultaneously, both find no existing row
- Both create a new entity with `id = null`
- Both execute INSERT → two identical rows with `consumed = delta` each
- Subsequent SUM reads are correct (2 × delta), but UPDATE path targets only one row

### Gap C — Blind UPDATE (lost update)
- Two threads both fetch the same existing row (consumed = 98)
- Both compute `new_consumed = 98 + 2 = 100`
- Both execute `UPDATE SET CONSUMED = 100 WHERE ID = :id`
- True consumed should be 102; second write silently overwrites first

### Gap D — `HashSet` provides no deduplication (`RewardIssueSummary` has no `@EqualsAndHashCode`)

`RewardIssueSummary` is annotated `@Getter @Setter @Builder` — no `@Data`, no `@EqualsAndHashCode`. Java falls back to `Object` reference identity for `equals()` and `hashCode()`.

`new HashSet<>(rewardIssueSummaries)` therefore deduplicates by object reference, not by field values. Every JPA-returned instance is a distinct heap object, so all rows survive into the working set — including pre-existing duplicate rows from Gap B.

**Downstream consequence of pre-existing duplicates:**
When Gap B has already produced two rows (`id=10`, `id=11`) for the same `(rewardId, level, kpi, userId, issueDate)`:
- Both objects enter `uniqueRewardIssueSummariesSet`
- The triple-filter `.findFirst()` picks `id=10` (stream order) → `continue`, no new entity created
- `id=11` remains in the set, unchecked
- After the loop, `updateSummaries()` iterates the entire set and calls `setConsumed(delta)` on ALL members — both `id=10` and `id=11` get updated
- Both go into `bulkUpdate()`: each is independently written as `consumed = consumedAtFetch + delta`
- Since both were fetched with the same prior value, both write the same value — the true total is under-reported by one row's worth on every subsequent issuance

### Why the SUM query masks Gap B at CHECK time but not UPDATE time
`refreshSummary()` queries `SUM(consumed)` — so duplicate rows do not cause incorrect REJECTION at constraint check time (the sum is still right). But the blind UPDATE writes a computed value to each row independently. As rows accumulate different `consumed` values over time (due to which row `.findFirst()` happens to pick), SUM reads drift from reality.

---

## 10. `buildUniqueSummaries()` Date-Matching — Summary

The function answers one question per constraint: **"is there already a row in the DB that is this constraint's bucket for today's (or this event's) date?"**

If yes → reuse it (UPDATE). If no → create a new one (INSERT).

---

### The summary table is a date-series of daily deltas, not a running total

`buildUniqueSummaries()` has **no concept of cycles**. It only does date equality:

```java
.filter(summary -> summary.issueDate.compareTo(trimmedEventDate) == 0)
```

The cycle window (e.g., "Jun 1–Jun 30 monthly") is **invisible at write time**. It appears only at **read time** in `LevelService.getConsumedValueFromDB()`:

```sql
SUM(consumed) WHERE issueDate BETWEEN :cycleStart AND :cycleEnd
```

So the table holds one row per distinct `(constraint-identity, userId, date)`. Each row is one day's accumulator. The SUM at check time spans all rows whose `issueDate` falls inside the current window.

**Example — monthly FIXED window, limit=10:**
```
Jun 5  issuance → no Jun 5 row → INSERT (consumed=1)
Jun 5  issuance → Jun 5 row exists → UPDATE (consumed=2)
Jun 10 issuance → no Jun 10 row → INSERT (consumed=1)
Jun 10 issuance → Jun 10 row exists → UPDATE (consumed=2)

Constraint check on Jun 15:
  SUM WHERE issueDate BETWEEN Jun 1 AND Jun 30 = 4   ✓ under limit

Jul 3 issuance:
  new cycle: SUM WHERE issueDate BETWEEN Jul 1 AND Jul 31 = 0
  no Jul 3 row → INSERT (consumed=1)
  (Jun rows still exist but are outside the Jul 1–31 window → don't count)
```

The cycle boundary doesn't trigger any DELETE or reset. Old rows simply fall outside the new cycle's `BETWEEN` filter and stop contributing.

---

### When is a new row inserted?

A new row is inserted when the date-equality filter finds no match — meaning no existing row has `issueDate == expectedDate` for this constraint+userId combination. This happens in two practical situations:

**First-ever issuance for this constraint** — no history at all, no row exists.

**Different date from all existing rows** — each calendar day produces its own row.
For ROLLING and NO_LIMIT constraints: `trimmedIssualDate` = today. A new day = no existing row for today = INSERT.
For FIXED window constraints: `trimmedEventDate` = the event's date (time-stripped, org-TZ). A different event date = no row for that date = INSERT.

**There is no "window boundary" detection at write time.** The function does not know or care which cycle a date belongs to. It only asks: "is there a row for this exact date?"

---

### Date selection per constraint type

| Constraint | `expectedDate` used for matching | Why |
|---|---|---|
| FIXED window, reward/customer level | `trimmedEventDate` | Event anchors the row to the correct backfill cycle |
| ROLLING window | `trimmedIssualDate` | Window is always relative to today; event date not used |
| Org-level (any window) | `trimmedIssualDate` | Org-level constraints always use today |
| Independent NO_LIMIT | `trimmedEventDate` if any FIXED constraint present on this reward, else `trimmedIssualDate` | Follows FIXED context when mixed levels are present |
| Dependent NO_LIMIT | SKIPPED | Windowed sibling at same (Level, KPI) already covers it |

**Cross-level FIXED bleed:** `isFixedWindowPresent = true` when ANY reward/customer-level FIXED constraint exists on the reward. This causes ALL non-org summary rows (including ROLLING constraints at a different level) to be fetched and matched by `trimmedEventDate`. This only occurs in the cross-level scenario (e.g., `CUSTOMER`-level FIXED + `REWARD`-level ROLLING) — FIXED and ROLLING cannot coexist at the same level.

---

### Triple-filter for row matching

```java
existingRow = uniqueSet.stream()
    .filter(rewardConstraint::isEquivalentToConstraint)   // level + kpi (+ attribute for org-level)
    .filter(summary -> summary.isValidUserId(userId))      // userId scope
    .filter(summary -> summary.issueDate == expectedDate)  // exact date match
    .findFirst()
```

All three must pass. A row that matches on level+kpi+userId but has a different date is NOT reused — a new row is created for the new date.

---

### Decision tree

```
For each constraint in validRewardConstraintList:
│
├── Dependent NO_LIMIT? → SKIP
│
├── Compute expectedDate:
│     FIXED window (reward/customer)  → trimmedEventDate
│     ROLLING / org-level             → trimmedIssualDate
│     Independent NO_LIMIT            → trimmedEventDate if isFixedWindowPresent, else trimmedIssualDate
│
├── Find existing row (triple-filter: level+kpi+attr, userId, date)
│     Found → reuse entity (will UPDATE consumed += delta)
│     Not found → create new entity with id=null (will INSERT consumed = delta)
│
└── After full loop: setConsumed(delta) on all entities → bulkSaveOrUpdate()
```

---

**Edge case: `eventDate` ≠ `issueDate` (backfill)**
If `eventDate = Jun 15` and `issueDate = Jun 19` (today):
- FIXED window constraint → row anchored to Jun 15 (falls in the Jun 1–30 cycle, correctly counted there)
- ROLLING constraint → row anchored to Jun 19 (today), unless cross-level FIXED bleed applies
- Both rows coexist; the SUM query for each constraint uses its own date range

**Edge case: day boundary (midnight to 01:00 AM)**
`trimmedIssualDate` = `Utils.getDateWithoutTimestampInSpecifiedZone(now(), orgZoneId)`. A request at 00:30 AM on Jun 20 gets `issueDate = Jun 20`. The new day's bucket starts immediately at midnight; no overlap with Jun 19 rows.
