# Multi-Tenant Analysis: ID vs (ID, ORG_ID) Lookups

## Table of Contents

- [Purpose](#purpose)
- [1. Database Schema: Composite PK (ID, ORG_ID)](#1-database-schema-composite-pk-id-org_id)
- [2. Java JPA Layer: Single @Id vs Composite](#2-java-jpa-layer-single-id-vs-composite)
    - [2.1 Entity PK Mapping](#21-entity-pk-mapping)
    - [2.2 Inherited JpaRepository Methods (Risk)](#22-inherited-jparepository-methods-risk)
    - [2.3 @JoinColumn: Lazy Load and Joins Use Only FK Column](#23-joincolumn-lazy-load-and-joins-use-only-fk-column)
- [3. ID-Only Lookups and Cross-Org Risk](#3-id-only-lookups-and-cross-org-risk)
    - [3.1 Confirmed ID-Only Usage (Must Fix)](#31-confirmed-id-only-usage-must-fix)
    - [3.2 Correct Pattern (orgId + id)](#32-correct-pattern-orgid--id)
    - [3.3 JPA Repositories Extending JpaRepository](#33-jpa-repositories-extending-jparepositoryentity-long)
- [4. JDBC / Native Queries](#4-jdbc--native-queries)
- [5. Duplicate Data Copy (UAT → Prod Org): Checklist](#5-duplicate-data-copy-uat--prod-org-checklist)
- [6. Action Items: Prevent Cross-Org Lookup and Lookup/Stream/Map Errors](#6-action-items-prevent-cross-org-lookup-and-lookupstreammap-errors)
    - [6.1 Code Fixes (Must Do Before Org Copy)](#61-code-fixes-must-do-before-org-copy)
    - [6.2 Lookup Patterns (Prevent Wrong Row / Exceptions)](#62-lookup-patterns-prevent-wrong-row--exceptions)
    - [6.3 Stream / Map: Avoid Lazy Load and ID-Only Resolution](#63-stream--map-avoid-lazy-load-and-id-only-resolution)
    - [6.4 Exceptions / Errors to Prevent](#64-exceptions--errors-to-prevent)
    - [6.5 New Code and Review Checklist](#65-new-code-and-review-checklist)
    - [6.6 Optional: Static / Automation](#66-optional-static--automation)
    - [6.7 JPA Repositories: Do the Action Items Apply?](#67-jpa-repositories-do-the-action-items-apply)
- [7. Summary](#7-summary)

---

## Purpose

To support **duplicating UAT org data into a new prod org** (copy all rows and update `ORG_ID`), we must ensure:

1. **No duplicate key errors** in DB or Java when the same business IDs exist under a new org.
2. **No cross-org lookups** — every query and join must be scoped by `ORG_ID` so that after copy, the new org never reads or joins to another org’s data.

The DB uses **composite primary keys `(ID, ORG_ID)`** on tenant-scoped tables. The Java layer uses **single `@Id` (ID only)** and adds `orgId` in WHERE clauses. This document captures the gap and required checks.

---

## 1. Database Schema: Composite PK (ID, ORG_ID)

Tables in `rewards_utf8_db` that have **PRIMARY KEY (`ID`,`ORG_ID`)** (or equivalent composite including `ORG_ID`):

| Table | Primary Key | Notes |
|-------|-------------|--------|
| TBL_REWARD | (ID, ORG_ID) | Core reward entity |
| TBL_BRAND | (ID, ORG_ID) | |
| TBL_VENDOR | (ID, ORG_ID) | |
| TBL_VENDOR_REDEMPTION | (ID, ORG_ID) | |
| TBL_VENDOR_REDEMPTION_ACTION | (ID, ORG_ID) | |
| TBL_VENDOR_DETAILS | (ID, ORG_ID) | |
| TBL_VENDOR_DETAILS_EXTN_PROPERTY_VALUES | (ID, ORG_ID) | |
| TBL_USER_REWARD | (ID, ORG_ID) | |
| TBL_USER_REWARD_LINK | (ID, ORG_ID) | |
| TBL_USER_REDEMPTION_DETAILS | (ID, ORG_ID) | |
| TBL_USER_PAYMENT_CONFIG | (ID, ORG_ID) | |
| TBL_USER_PAYMENT_CONFIG_CURRENCY | (ID, ORG_ID) | |
| TBL_USER_TENDER_DETAILS | (ID, ORG_ID) | |
| TBL_USER_TENDER_ITEMS | (ID, ORG_ID) | |
| TBL_USER_CUSTOM_FIELD_MAPPING | (ID, ORG_ID) | |
| TBL_TRANSACTION | (ID, ORG_ID) | |
| TBL_TRANSACTION_ACTION_LOG | (ID, ORG_ID) | |
| TBL_REWARD_CONSTRAINT | (ID, ORG_ID) | |
| TBL_REWARD_GROUP_MAPPING | (ID, ORG_ID) | |
| TBL_REWARD_OWNER | (ID, ORG_ID) | |
| TBL_REWARD_ISSUE_SUMMARY | (ID, ORG_ID) | |
| TBL_REWARD_EXPIRY_REMINDER | (ID, ORG_ID) | |
| TBL_REWARD_REVENUE | (ID, ORG_ID) | |
| TBL_REWARD_DETAILS_EXTN_PROPERTY_VALUES | (ID, ORG_ID) | |
| TBL_PAYMENT_CONFIG | (ID, ORG_ID) | |
| TBL_PAYMENT_CONFIG_CURRENCY | (ID, ORG_ID) | |
| TBL_LINK_REWARD_CATALOG_PROMOTION | (ID, ORG_ID) | |
| TBL_CATALOG_PROMOTION | (ID, ORG_ID) | |
| TBL_RICH_CONTENT_META | (ID, ORG_ID) | |
| TBL_REWARD_UPLOAD | (id, org_id) | |
| TBL_FULFILLMENT_STATUS | (ID, ORG_ID) | |
| TBL_FULFILLMENT_DETAILS | (ID, ORG_ID) | |
| TBL_CUSTOM_FIELD_MAPPING | (ID, ORG_ID) | |
| TBL_CUSTOM_FIELDS | (ID, ORG_ID) | |
| TBL_CUSTOM_FIELD_ENUMS | (ID, ORG_ID) | |
| TBL_CONFIG_VALUES | (ID, ORG_ID) | |
| TBL_COUNTRY | (ID, ORG_ID) | |
| TBL_CITY | (ID, ORG_ID) | |
| TBL_CATEGORY | (ID, ORG_ID) | |
| TBL_COMMUNICATION | (ID, ORG_ID) | |
| TBL_AREA | (ID, ORG_ID) | |
| TBL_GROUP | (ID, ORG_ID) | |
| TBL_ISSUE_REVENUE | (ID, ORG_ID) | |
| TBL_ISSUE_REVENUE_DETAIL | (ID, ORG_ID) | |
| REF_* (reward ref tables) | Various composite (e.g. REWARD_ID, ORG_ID, …) | REF tables use natural/composite keys including ORG_ID |

**Tables with single-column PK (no ORG_ID in PK):**  
TBL_REWARD_REDEMPTION_TYPES, TBL_REWARD_DETAILS_EXTN_PROPERTY, TBL_VENDOR_DETAILS_EXTN_PROPERTY, TBL_CONFIG_KEYS, TBL_DETAILS_SCOPE, TBL_MIGRATE_TRANSACTIONS, REVINFO — typically reference/lookup tables, not tenant data.

**Why this matters for copy:**  
When copying UAT → prod org, new rows get the same `ID` values and a new `ORG_ID`. The composite PK `(ID, ORG_ID)` keeps rows unique per org and avoids PK violations. Any logic that assumes “ID is globally unique” is wrong in this model.

---

## 2. Java JPA Layer: Single @Id vs Composite

### 2.1 Entity PK Mapping

- **DB:** Composite PK `(ID, ORG_ID)` on tenant tables.
- **Java:** Most entities use **single `@Id`** on `id` only; `orgId` is a separate column, not part of the JPA identifier.

Example (`Reward.java`):

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
@Column(name = "ID")
private Long id;

@Column(name = "ORG_ID")
private Long orgId;
```

So:

- **Persistence identity** in JPA is only `id`. Hibernate’s internal identity map and `EntityManager.find(EntityClass, id)` use **ID only**.
- **Repository** methods must **always** use `(orgId, id)` in custom queries; inherited `findById(Long)` must not be used for tenant-scoped entities.

### 2.2 Inherited JpaRepository Methods (Risk)

Repositories extend `JpaRepository<Entity, Long>` and thus expose:

- `findById(Long id)` — **ID-only lookup; can return wrong org or wrong row after copy.**
- `getById(Long id)` — same.
- `getOne(Long id)` — same.

**Rule:** For any entity whose table has composite PK `(ID, ORG_ID)`, these methods must **not** be used. Use custom methods that take `(orgId, id)` instead.

### 2.3 @JoinColumn: Lazy Load and Joins Use Only FK Column

Many associations use a single column for the FK, e.g.:

- `@JoinColumn(name = "REWARD_ID")` → Hibernate joins only on `REWARD_ID`.
- Parent table is `TBL_REWARD(ID, ORG_ID)`. The logical FK is `(REWARD_ID, ORG_ID)`, but the mapping only has `REWARD_ID`.

So when Hibernate:

- Resolves a lazy `Reward` from a child (e.g. `RewardConstraint.reward`), or
- Generates a join in JPQL/criteria,

it typically produces **`WHERE ID = ?`** (or join on `REWARD_ID` only). It does **not** add `AND ORG_ID = ?` unless the query is written that way.

**Implication:**  
If a child row is loaded in context of org A and then `reward` is accessed, Hibernate may load `TBL_REWARD` by `ID` only. After UAT→prod copy, the same `ID` exists in both orgs; without `ORG_ID` in the join/lookup, the loaded `Reward` could be from the wrong org.

**Entities with @JoinColumn to Reward (or other composite-PK parent) using single column:**

- RewardConstraint → `REWARD_ID`
- RewardOwner, RewardGroupMapping, RewardRevenue, RewardPaymentConfig, RewardProgramMapping → `REWARD_ID`
- RewardWithCatalogPromotion → `REWARD_ID`, `PROMOTION_ID`
- RewardCategoryKey, RewardDetailsKey, RewardGeographyKey, RewardCommunicationKey → `REWARD_ID` (and others)
- UserReward → `REWARD_ID`, `BRAND_ID`
- UserRewardRedemptionDetails → `REWARD_ID`, `TRANSACTION_ID`
- UserRewardPaymentConfig → `REWARD_ID`
- Label, CardSeries → `REWARD_ID`
- VendorRedemption → `VENDOR_ID`
- VendorRedemptionAction → `VENDOR_REDEMPTION_ID`
- etc.

**Recommendation:**
- Prefer loading parent and child in the same org-scoped query (e.g. `findByOrgIdAndId(orgId, id)` and explicit child fetches with `orgId`), and avoid relying on lazy load of these associations across context.
- Long-term: consider composite join columns or dedicated queries that join on `(ID, ORG_ID)` where the schema supports it.

---

## 3. ID-Only Lookups and Cross-Org Risk

### 3.1 Confirmed ID-Only Usage (Must Fix)

| Location | Code | Risk | Status |
|----------|------|------|--------|
| `BrandFacade.java` | `brandRepository.findById(request.getId())` | Loads Brand by ID only. After org copy, same brand ID can exist in multiple orgs; can return wrong org or wrong row. | **Action:** Replace with `brandRepository.findByIdAndOrgId(request.getId(), authDetailsProvider.get().getOrgId())` before org copy. Implementation planned later. |

**No other ID-only lookups found** in the codebase (search: `findById(`, `getById(`, `getOne(`, `EntityManager.find`). Pre-copy action: this is the only code change required; to be implemented before org copy.

### 3.2 Correct Pattern (orgId + id)

These already scope by org and are safe for copy and multi-tenancy:

- `rewardRepository.findByOrgIdAndId(orgId, rewardId)` — used in RewardFacade.
- `transactionRepository.findByIdAndOrgId(transactionId, orgId)`, `findByIdAndOrgIdAndUserId`, etc.
- `brandRepository.findByIdAndOrgId(id, orgId)` — used in BrandService.
- `categoryRepository.findByIdAndOrgId(id, orgId)`.
- `communicationService.getByIdAndOrgId(id, orgId)`.
- `fulfillmentStatusRepository.findByIdAndOrgIdAndIsEnabledTrue(id, orgId)`.
- `vendorRedemptionRepository.getByIdVendorAndOrgId(orgId, id, vendorId)`.
- RewardRepository custom queries: `findByIdRewardsIsEnabledOrExpired`, `findByIdRewardsIsEnabledAndNotExpired` — all take `orgId` and use `reward.orgId=:orgId AND reward.id=:rewardId`.

### 3.3 JPA Repositories Extending JpaRepository&lt;Entity, Long&gt;

For any entity whose table has composite PK `(ID, ORG_ID)`:

- **Do not use** inherited `findById(Long)`, `getById(Long)`, or `getOne(Long)`.
- **Do use** custom methods that take `orgId` first, then `id`, e.g. `findByOrgIdAndId(Long orgId, Long id)`.

Repositories that extend `JpaRepository<*, Long>` and map to composite-PK tables include (non-exhaustive): RewardRepository, UserRewardRepository, TransactionRepository, BrandRepository, VendorRepository, VendorRedemptionRepository, RewardConstraintRepository, RewardOwnerRepository, RewardPaymentConfigRepository, RewardProgramMappingRepository, CommunicationRepository, CategoryRepository, CatalogPromotionRepository, and others for the tables listed in Section 1.

**Recommendation:**  
Add a code rule or review checklist: “For entities with composite PK (ID, ORG_ID), do not call findById/getById/getOne; use org-scoped finders only.”

---

## 4. JDBC / Native Queries

Spot-checked usage:

- **RewardSummaryJdbcRepository:** Builds `WHERE r.org_id = :orgId` and adds `AND r.id = :rewardId` when filtering by reward; joins use `r.id = ... AND r.org_id = ...` for child tables. Safe for org scope.
- **RewardWithCatalogPromotionJdbcRepository:** `WHERE ID = :ID AND ORG_ID=:ORG_ID`. Safe.
- **CatalogPromotionJdbcRepository:** `WHERE ID = :ID AND ORG_ID=:ORG_ID`. Safe.

**Recommendation:**  
For any new or existing native/JDBC query on tenant tables, require that:

- Every table in the FROM/JOIN that is tenant-scoped is filtered by `ORG_ID` (e.g. `r.org_id = :orgId`), and
- Joins between tenant tables use both ID and ORG_ID where the schema has composite PK (e.g. `ON r.id = rd.reward_id AND r.org_id = rd.org_id`).

---

## 5. Duplicate Data Copy (UAT → Prod Org): Checklist

When duplicating all data from one org to another and updating `ORG_ID`:

1. **DB**
    - All target tables use composite PK `(ID, ORG_ID)` (or equivalent) so inserting copied rows with new `ORG_ID` does not cause duplicate key errors.
    - No unique constraint on `ID` alone for tenant data.

2. **Java**
    - No use of `findById(id)` / `getById(id)` for these entities; all lookups use `(orgId, id)`.
    - No join or lazy load that resolves a composite-PK parent by ID only (prefer org-scoped queries and explicit fetches with `orgId`).

3. **Fix before copy**
    - Replace `brandRepository.findById(request.getId())` in BrandFacade with org-scoped lookup (e.g. `findByIdAndOrgId` with `orgId` from context).
    - Audit all repositories for any other `findById`/`getById`/`getOne` on tenant entities and replace with org-scoped alternatives.

4. **Ongoing**
    - Enforce “no ID-only lookup or join for tenant entities” in review and static checks where possible.

---

## 6. Action Items: Prevent Cross-Org Lookup and Lookup/Stream/Map Errors

Use this section as a checklist. Completing these items ensures no cross-org data access and no exceptions or wrong data during lookup, stream, or map operations.

### 6.1 Code Fixes (Must Do Before Org Copy)

| # | Action | Location / Scope | Why |
|---|--------|------------------|-----|
| 1 | **Replace ID-only Brand lookup** | `BrandFacade.java` (~line 94) | `brandRepository.findById(request.getId())` → replace with `findByIdAndOrgId(request.getId(), authDetailsProvider.get().getOrgId())` before org copy. (Impl later.) |
| 1a | Use org-scoped lookup and pass `orgId` from context. | Same | Ensures the brand is always for the current org; avoids cross-org and NPE when brand doesn't exist in that org. |
| 2 | **Audit all repositories** for `findById(`, `getById(`, `getOne(` on tenant entities. | All repositories extending `JpaRepository<Entity, Long>` for entities in Section 1 | Any ID-only call can return wrong org or throw after copy. Replace with `findByOrgIdAndId(orgId, id)` (or equivalent). |
| 3 | **Ensure `orgId` is always available** where tenant lookups happen. | Request handlers, facades, services that perform lookups | If `orgId` is missing, developers may fall back to `findById(id)`; enforce passing `orgId` from auth/context. |

### 6.2 Lookup Patterns (Prevent Wrong Row / Exceptions)

| # | Action | Details |
|---|--------|---------|
| 4 | **Never use inherited `findById(id)` / `getById(id)` / `getOne(id)`** for entities whose table has composite PK `(ID, ORG_ID)`. | Use only custom methods that take `(orgId, id)`, e.g. `findByOrgIdAndId(orgId, id)` or `findByIdAndOrgId(id, orgId)`. |
| 5 | **When loading a parent by ID (e.g. Reward, Brand), always pass `orgId`.** | Prevents loading another org's row and avoids "entity not found" in the correct org when the same ID exists elsewhere. |
| 6 | **When loading child collections (e.g. constraints, owners, details), always use org-scoped repository methods** that take `(orgId, parentId)`. | Avoids ID-only joins and keeps all data in the same org. |
| 7 | **Handle "not found" explicitly** when using org-scoped finders. | e.g. `Optional.ofNullable(repo.findByOrgIdAndId(orgId, id)).orElseThrow(...)` so callers get a clear exception instead of NPE or wrong data. |

### 6.3 Stream / Map: Avoid Lazy Load and ID-Only Resolution

During **stream**, **map**, or **forEach** over collections of entities, accessing a lazy association (e.g. `entity.getReward()`, `entity.getBrand()`) can:

- Trigger an **ID-only** load (wrong org after copy).
- Throw **LazyInitializationException** if the session is closed.
- Cause **NPE** if the association was never loaded or returns null.

| # | Action | Details |
|---|--------|---------|
| 8 | **Prefer loading associations in the same org-scoped query** (e.g. JOIN FETCH or explicit batch load with `orgId`). | When you need `reward` or `brand` in a list, load the list with associations in one query that filters by `orgId` so no lazy load by ID happens later. |
| 9 | **In stream/map, avoid calling getters that trigger lazy load** of composite-PK parents (e.g. `Reward`, `Brand`, `VendorRedemption`) unless that association was already loaded in an org-scoped way. | If you must touch these inside a stream, ensure they were eagerly loaded or batch-fetched with org scope; otherwise use a separate org-scoped query to load the parent by `(orgId, id)`. |
| 10 | **When mapping entity → DTO inside a stream**, use only fields already loaded or populated (e.g. `id`, `orgId`, in-memory collections). For parent references, prefer passing a pre-loaded map `(orgId, id) → entity` built with org-scoped queries. | Avoids lazy load and ID-only resolution during map. |
| 11 | **Use DTOs or read-only projections** for list APIs where you don't need full entity graph. | Reduces chance of lazy load and wrong-org resolution when streaming over results. |

### 6.4 Exceptions / Errors to Prevent

| # | Risk | Mitigation |
|---|------|------------|
| 12 | **Duplicate key / constraint violation** on insert or update | DB already uses composite PK `(ID, ORG_ID)`; ensure no unique index on `ID` alone. In Java, never assume ID is globally unique when building keys or merging. |
| 13 | **Wrong org returned (cross-org)** | All lookups and joins must include `orgId`; no `findById(id)` for tenant entities (see 6.2 items 4–7). |
| 14 | **NPE or "entity not found"** when entity exists in another org | Use org-scoped finders and handle Optional/empty explicitly; don't rely on `findById(id)` (see 6.2 items 5, 7). |
| 15 | **LazyInitializationException** during stream/map | Load associations in org-scoped queries or within an open transaction/session; avoid lazy load after leaving the service layer (see 6.3 items 8–10). |
| 16 | **Map/collection key collisions** when using only `id` as key | Use composite key `(orgId, id)` (e.g. in caches or in-memory maps) so data from different orgs doesn't overwrite. |

### 6.5 New Code and Review Checklist

| # | Check | When |
|---|--------|------|
| 17 | Every new repository method for tenant entities takes `orgId` (or equivalent context) and uses it in the WHERE clause. | New code / PR |
| 18 | No new use of `findById`, `getById`, or `getOne` for entities listed in Section 1. | New code / PR |
| 19 | New JPQL/native queries on tenant tables include `org_id` (or `entity.orgId`) for every tenant table in FROM/JOIN. | New code / PR |
| 20 | New stream/map over entities that reference Reward, Brand, Vendor, etc. does not rely on lazy load of those references; they are loaded with org scope or via a pre-built map. | New code / PR |

### 6.6 Optional: Static / Automation

| # | Action | Notes |
|---|--------|--------|
| 21 | Add a custom rule or script to flag `\.findById(`, `\.getById(`, `\.getOne(` on tenant entity repositories. | E.g. in code review or a simple static check. |
| 22 | Document "org-scoped lookup only" in team guidelines or `.cursor/rules` / RULE.md. | So all future code follows the same pattern. |

### 6.7 JPA Repositories: Do the Action Items Apply?

**Yes.** The action items apply to **every JPA repository whose entity is stored in a table with composite PK `(ID, ORG_ID)`**.

Because these repositories extend `JpaRepository<Entity, Long>`, they **inherit** `findById(Long)`, `getById(Long)`, and `getOne(Long)`. Those methods perform **ID-only** lookup. For tenant tables, that can return the wrong org or fail after org copy. So:

- **Do not use** inherited `findById` / `getById` / `getOne` for any of these repositories.
- **Do use** only custom methods that take `orgId` (and use it in the query).

Below: applicability for the repositories in scope and for all such repositories in the codebase.

#### Repositories in scope (sample)

| Repository | Entity | Table | PK | Action items apply? | Current status |
|------------|--------|-------|-----|---------------------|----------------|
| **RewardDetailsExtendedPropertyValueRepository** | RewardDetailsExtendedPropertyValue | TBL_REWARD_DETAILS_EXTN_PROPERTY_VALUES | (ID, ORG_ID) | **Yes** | Only org-scoped methods declared (`findByOrgIdAndRewardId...`). **Do not use** inherited `findById(id)`. |
| **DownstreamFailureLogRepository** | DownstreamFailureLog | LOG_DOWNSTREAM_FAILURE | (ID, ORG_ID) | **Yes** | No custom methods; only inherits `findById`. **Must not use** `findById(id)`; add org-scoped finders (e.g. `findByOrgIdAndId`) if lookup by id is needed. |
| **CustomFieldsMetaRepository** | CustomFieldMeta | TBL_CUSTOM_FIELDS | (ID, ORG_ID) | **Yes** | Has `findByOrgIdAndId`, `findByOrgIdAndIdIn`. **Do not use** inherited `findById(id)`; use `findByOrgIdAndId(orgId, id)` instead. |
| **CustomFieldRepository** | CustomFieldMapping | TBL_CUSTOM_FIELD_MAPPING | (ID, ORG_ID) | **Yes** | Only org-scoped methods (`findByOrgIdAndEntityIdInAndScope`, etc.). **Do not use** inherited `findById(id)`. |
| **ConfigValueRepository** | ConfigValue | TBL_CONFIG_VALUES | (ID, ORG_ID) | **Yes** | Has `findByOrgIdAndConfigKeyId`. **Do not use** inherited `findById(id)` for entity lookup; use org-scoped method if you need by-id (e.g. add `findByOrgIdAndId`). |

#### Rule for all JPA repositories in this codebase

| Repository extends | Entity table PK | Action items apply? |
|--------------------|------------------|----------------------|
| `JpaRepository<Entity, Long>` | Composite `(ID, ORG_ID)` (see Section 1) | **Yes** — never use `findById(id)` / `getById(id)` / `getOne(id)`; use only org-scoped finders. |
| `JpaRepository<Entity, Long>` | Single-column PK (e.g. TBL_CONFIG_KEYS, TBL_DETAILS_SCOPE, REVINFO) | No — ID is globally unique; `findById(id)` is acceptable. |
| `JpaRepository<Entity, CompositeKey>` (e.g. RewardDetailsRepository) | Composite (e.g. natural key) | Depends on whether the key includes orgId; if it does, same rules as above. |

So for **RewardDetailsExtendedPropertyValueRepository**, **DownstreamFailureLogRepository**, **CustomFieldsMetaRepository**, **CustomFieldRepository**, and **ConfigValueRepository**: the action items **do** hold. Always pass `orgId` and use only org-scoped methods; never use the inherited ID-only finders for these tenant entities.

---

## 7. Summary

| Area | Finding | Action |
|------|---------|--------|
| DB schema | Tenant tables use composite PK `(ID, ORG_ID)`; copy by updating ORG_ID is consistent with schema. | Ensure no unique constraint on `ID` alone. |
| JPA entities | Use single `@Id` (ID); `orgId` in WHERE only. | Keep adding `orgId` to all custom queries; avoid composite @Id unless we refactor. |
| Repositories | Inherited `findById(Long)` exists for all JpaRepository<*, Long>. | Do not use for tenant entities; use only org-scoped finders. |
| BrandFacade | Uses `brandRepository.findById(request.getId())`. | Replace with `findByIdAndOrgId(id, orgId)` (org from context) before org copy. (Impl later.) |
| @JoinColumn | Many associations use single column (e.g. REWARD_ID). | Avoid relying on lazy load of parent; load via org-scoped queries. |
| JDBC/native | Sampled queries use `org_id` and (id, org_id) joins. | Keep pattern; audit any new SQL. |

Ensuring **no lookup or join by ID alone** for tenant-scoped entities (in both JPA and JDBC) keeps the app safe for org duplication and prevents cross-org data access.
