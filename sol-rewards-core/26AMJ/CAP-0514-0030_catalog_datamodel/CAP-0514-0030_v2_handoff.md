# Arch-Investigator Handoff: v2 MongoDB Routing for getAllForBrand
**Feature:** CAP-0514-0030-V2 — `?v2=true` query-param toggle routing `getAllForBrand` to MongoDB  
**Branch:** CAP-0514-0030_catalog_datamodel  
**Date:** 2026-05-14  
**Author:** arch-investigator (claude-sonnet-4-6)

---

## 1. Problem Statement

The existing `getAllForBrand` endpoint (`GET /api_gateway/v1/user/reward/brand/{brandName}`) executes a Hibernate Criteria query against MySQL (`rewards_utf8_db`) with up to 12 join conditions: activation status, date range, group membership, label membership, category membership, geography (area/city/country), redemption type, vendor, card series, tier, label field, and points/cash range. For large orgs with thousands of active rewards and complex filter combinations, this query causes MySQL AAS (Average Active Sessions) spikes, degrading throughput for unrelated writes on the same DB host.

A MongoDB CQRS read model (`active_rewards` + `rewards_catalogue` collections) was introduced in Phase 1 of this ticket. The goal of this phase is to wire a production A/B test that routes qualified requests to MongoDB, measuring latency and result accuracy against the MySQL baseline — with zero contract change for callers.

---

## 2. Motivation (Root Cause)

- MySQL query for `getAllForBrand` joins up to 6 tables (`reward`, `reward_details`, `reward_group_mapping`, `reward_category`, `vendor_redemption`, `label`) plus optional subqueries for card series.
- The Criteria query is built dynamically in `RewardFacade.getRewardSummary()` (file:line `RewardFacade.java:809`), calling `getTotalElements()` for a separate COUNT query, then the main SELECT with pagination — two round-trips per request.
- High-traffic orgs doing `page=0 size=500` on the `/brand/{brandName}` endpoint generate sustained read load that blocks schema migrations and write-heavy reward issue flows.
- The MongoDB model was built precisely to serve this use case via pre-materialised denormalized docs.

---

## 3. Solution Summary

Add `?v2=true` query parameter to the existing `getAllForBrand` endpoint. When `v2=true` AND the org is in the `CATALOGUE_MONGO_ENABLED_ORGS` env-var allowlist, the request is routed to a new `UserRewardMongoService` which queries MongoDB. All other requests fall back silently to the existing MySQL path. Response shape is identical. The toggle is invisible to callers.

**Not chosen:**
- Separate endpoint: rejected — requirement is same URL, same contract.
- Feature flag per org in DB: rejected — adds schema dependency and DB read on every request. Env var is simpler and restartless-togglable via pod env.
- Cursor-based pagination in v2: deferred — offset pagination covers the real traffic pattern (skip rarely exceeds 500); cursor pagination is documented as a future migration path.

---

## 4. Exact Call Chain — Existing `getAllForBrand` (MySQL path)

```
UserRewardResource.getAllForBrand()          UserRewardResource.java:46-48
  └─ rewardFacade.getAllForBrand()           RewardFacade.java:1686
       ├─ authDetailsProvider.get().getOrgId()   RewardFacade.java:1693
       ├─ brandService.get(brandName, orgId)      RewardFacade.java:1694
       ├─ getUserRewardSummaryDTOs()              RewardFacade.java:1719
       │    └─ getRewardSummaries()              RewardFacade.java:1772
       │         ├─ labelService.getCustomerLabelIds()   RewardFacade.java:1823
       │         ├─ cardSeriesService.getCustomerCardSeriesIds()  RewardFacade.java:1827
       │         └─ getAllRewardDetails()         RewardFacade.java:1831
       │              └─ getRewardSummary()       RewardFacade.java:679, 809
       │                   ├─ getTotalElements()  RewardFacade.java:816 (COUNT query)
       │                   ├─ createQuery()       RewardFacade.java:820 (main SELECT)
       │                   │    Predicates applied:
       │                   │    ├─ orgId = ?                      RewardFacade.java:847
       │                   │    ├─ activationPredicate()          RewardSpecificationService.java:151
       │                   │    ├─ dateBasedFilters()             RewardSpecificationService.java:117
       │                   │    ├─ geographyService.filterPredicates()  GeographyService.java:344
       │                   │    ├─ categoryListPredicate()        RewardSpecificationService.java:44
       │                   │    ├─ rewardTypeListPredicate()      RewardSpecificationService.java:72
       │                   │    ├─ vendorListPredicate()          RewardSpecificationService.java:80
       │                   │    ├─ cardSeriesIds subquery         RewardFacade.java:878-916
       │                   │    ├─ customerLabelIds join          RewardFacade.java:920-930
       │                   │    ├─ labelGroupTierTypeFilter()     RewardFacade.java:1183
       │                   │    ├─ filter() [filterBy param]      RewardFacade.java:1298
       │                   │    └─ filterOnPointsAndCash()        RewardFacade.java:1040
       │                   └─ getRewardsBasedOnPagination()       RewardFacade.java:1210
       │                        (applies skip = page*size, limit = size)
       │                   └─ rewardSummaryJdbcRepository.setCategories()  RewardFacade.java:828
       ├─ rewardSegmentService.getRewardsAfterSegmentValidation() RewardFacade.java:1835
       ├─ getUserRewardsProcessors (processor chain)              RewardFacade.java:1842
       └─ userRewardHelper.dbToJson()                             RewardFacade.java:1798
            (converts RewardSummary list → UserRewardGetDto list)
  └─ Response wraps UserRewardGetAllResponse{status, rewardList, pagingDto, orgLevelRestrictions}
                                                                   RewardFacade.java:1733-1740
```

**Response type:** `UserRewardGetAllResponse` (`UserRewardGetAllResponse.java`)  
Contains: `StatusDto status`, `List<UserRewardGetDto> rewardList`, `PagingDto pagingDto`, `OrgLevelConstraintsSummaryResponse orgLevelRestrictions`

---

## 5. RewardFilter Gap Analysis (Phase 2)

All `RewardFilter` fields are listed below. The MongoDB v2 path must handle each.

| RewardFilter field | MySQL mechanism | Current in ActiveRewardDoc? | Current in RewardCatalogueDoc? | Action needed |
|---|---|---|---|---|
| `activationStatus` (ENABLED/DISABLED/ALL) | `isEnabled = true/false` predicate on `Reward.enabled` | YES — `isEnabled` field | YES — `isEnabled` field | PRESENT — query on `active_rewards.isEnabled` |
| `status` (LIVE/UPCOMING/ENDED) | Date range: startDate < now < endDate etc. | PARTIAL — `expiresAt` (end); no startDate | NO — `startDate` and `endDate` present | ADD-TO-ACTIVE: add `startDate` (Date, BSON Date). Already in RewardCatalogueDoc. |
| `includeExpired` | Suppresses `endDate >= now` predicate if true | NO | NO | COMPUTED — apply skip of `expiresAt >= now` filter in query if `includeExpired=false`; if `includeExpired=true`, do not filter on `isEnabled/expiresAt`. Note: active_rewards TTL auto-removes expired docs, so expired docs will not be present unless includeExpired=true requires querying rewards_catalogue instead. See design note below. |
| `groupName` (multi-value, comma-sep group names) | JOIN to `reward_group_mapping`, resolve name→groupId | PARTIAL — `groupIds` (List<Long> by ID) | YES — `groupIds` | ADD-TO-ACTIVE: no change needed — groupIds already stored. But group name→id resolution must happen before query (call GroupService); store groupIds. PRESENT after resolution. |
| `group` (single group string, Reward.INTOUCH_GROUP) | Direct equality on `Reward.group` column | NO | NO | ADD-TO-ACTIVE: add `group` (String). ADD-TO-CATALOGUE: add `group` (String). |
| `label` (single label string, Reward.LABEL) | Direct equality on `Reward.label` column | NO | NO | ADD-TO-ACTIVE: add `label` (String). ADD-TO-CATALOGUE: add `label` (String). |
| `tier` (single tier string, Reward.INTOUCH_TIER) | Direct equality on `Reward.tier` column | NO | NO | ADD-TO-ACTIVE: add `tier` (String). ADD-TO-CATALOGUE: add `tier` (String). |
| `type` (Reward.TYPE) | Direct equality on `Reward.type` column | NO | NO | ADD-TO-ACTIVE: add `type` (String). ADD-TO-CATALOGUE: add `type` (String). |
| `redemptionType` (multi-value enum) | `redemptionType IN (...)` on Reward | NO | NO | ADD-TO-ACTIVE: add `redemptionType` (String). ADD-TO-CATALOGUE: add `redemptionType` (String). |
| `vendorId` (multi-value, vendor IDs) | JOIN to `vendor_redemption`, vendorId IN (...) | NO | NO | ADD-TO-ACTIVE: add `vendorId` (Long). ADD-TO-CATALOGUE: add `vendorId` (Long). Sourced from `Reward.vendorRedemptionId` → VendorRedemption.vendorId. |
| `minPoints` / `maxPoints` | `intouchPoints` range or PaymentConfig join | PARTIAL — `pointsValue` (Integer) | YES — `pointsValue` | PRESENT for simple intouchPoints range. PaymentConfig points (POINTS/POINTS_CASH modes) are NOT captured. Mark as PRESENT with known limitation: PaymentConfig-based points range not supported in v2 (rare use case). |
| `minCash` / `maxCash` | PaymentConfig JOIN for CASH/POINTS_CASH mode | NO | NO | NOT-SUPPORTED-IN-V2 — cash filtering requires PaymentConfig join; PaymentConfig data not in MongoDB model. Fallback to MySQL if minCash or maxCash present. |
| `category` (multi-value, category names) | JOIN to `reward_category`, resolve name→categoryId | NO | NO | ADD-TO-ACTIVE: add `categoryIds` (List<Long>). ADD-TO-CATALOGUE: add `categoryIds` (List<Long>). Sync must resolve category name→ID and store IDs. Query resolves name→ID via CategoryRepository then filters. |
| `area` (multi-value, area names) | JOIN to `reward_geography` + area lookup | NO | NO | NOT-SUPPORTED-IN-V2 — geography filtering requires separate geography join tables; data volume is high and geography is org-specific. Fallback to MySQL if area filter present. |
| `city` (multi-value, city names) | JOIN to `reward_geography` + city lookup | NO | NO | NOT-SUPPORTED-IN-V2 — same reason as area. Fallback to MySQL if city filter present. |
| `country` (multi-value, country names) | JOIN to `reward_geography` + country lookup | NO | NO | NOT-SUPPORTED-IN-V2 — same reason. Fallback to MySQL if country filter present. |
| `createdOnDaysAgo` | `createdOn <= now - N days` | NO | NO | ADD-TO-ACTIVE: add `createdOn` (Date, BSON Date). |
| `sortBy` | Dynamic expression on Reward fields | NO | NO | COMPUTED — support: `createdOn` (default), `priority`, `intouchPoints`/`pointsValue`, `rewardRank`. Unsupported sortBy values → fallback to `createdOn`. |
| `orderBy` | ASC/DESC on sort expression | N/A | N/A | COMPUTED — pass to MongoDB Sort direction. |
| `page` / `size` | `setFirstResult(page*size)`, `setMaxResults(size)` | N/A | N/A | COMPUTED — `skip(page * size).limit(size)`. Guard: if `skip > 1000` → WARN + fallback to MySQL. |
| `filterBy` (freeform field=value comparison) | Dynamic Criteria predicate on arbitrary field | NO | NO | NOT-SUPPORTED-IN-V2 — arbitrary field expressions cannot be safely translated. Fallback to MySQL if filterBy present. |
| `sortOnGroups` | Sort by rewardGroupRank (from group mapping join) | NO | NO | NOT-SUPPORTED-IN-V2 — requires group rank from mapping table, not stored in docs. Fallback to MySQL if sortOnGroups=true. |
| `userId` | Used to fetch customerLabelIds and cardSeriesIds | N/A | N/A | COMPUTED — userId triggers label/card-series lookup in MySQL even in v2 path. These are customer-specific restriction checks, not catalogue filters. In v2: if userId present, always fallback to MySQL (customer restriction chain uses processor pattern; too complex to replicate in v2 POC). |
| `language` | JOIN to reward_details for language-specific name | N/A | N/A | COMPUTED — `RewardCatalogueDoc.langDetails` Map<language, name>. Serve from map at display time. |
| `includeVendorId` | Controls whether vendorId is included in SELECT | N/A | N/A | COMPUTED — always include vendorId from stored doc; flag is for SELECT-level, not filter-level. Honour by including/excluding in response. |

**Summary of fallback triggers (any one → MySQL path regardless of v2=true):**
- `minCash` or `maxCash` present
- `area`, `city`, or `country` filter present
- `filterBy` present
- `sortOnGroups = true`
- `userId` present (customer-specific restrictions)
- `skip > 1000` (deep pagination guard)

**Design note on includeExpired:** The `active_rewards` collection only holds currently-active docs (TTL removes expired). If `includeExpired=true`, the v2 path must query `rewards_catalogue` directly (no TTL) with `isEnabled=true` and no date filter. This adds complexity; for the POC, if `includeExpired=true` → fallback to MySQL.

---

## 6. Fields to Add to MongoDB Documents

### 6a. ActiveRewardDoc — fields to add

| New field | BSON type | Java type | Source (MySQL) | Purpose |
|---|---|---|---|---|
| `startDate` | Date | `java.util.Date` | `Reward.startDate` | status filter (LIVE/UPCOMING) |
| `group` | String | `String` | `Reward.group` (INTOUCH_GROUP) | group filter |
| `label` | String | `String` | `Reward.label` (LABEL) | label string filter |
| `tier` | String | `String` | `Reward.tier` (INTOUCH_TIER) | tier filter |
| `type` | String | `String` | `Reward.type` (TYPE) | type filter |
| `redemptionType` | String | `String` | `Reward.redemptionType.name()` | redemptionType filter |
| `vendorId` | Long | `Long` | `Reward.vendorRedemption.vendorId` | vendor filter |
| `categoryIds` | Array | `List<Long>` | `RewardCategory.categoryId` list | category filter |
| `priority` | Int32 | `Integer` | `Reward.priority` | sortBy=priority |
| `rewardRank` | Int32 | `Integer` | `Reward.rewardRank` | sortBy=rewardRank |
| `createdOn` | Date | `java.util.Date` | `Reward.createdOn` | createdOnDaysAgo filter + sortBy=createdOn |

### 6b. RewardCatalogueDoc — fields to add (display/response use)

| New field | BSON type | Java type | Source (MySQL) | Purpose |
|---|---|---|---|---|
| `group` | String | `String` | `Reward.group` | response: UserRewardGetDto.group |
| `label` | String | `String` | `Reward.label` | response: UserRewardGetDto.label |
| `tier` | String | `String` | `Reward.tier` | response: UserRewardGetDto.tier |
| `type` | String | `String` | `Reward.type` | response: not directly but type is stored |
| `redemptionType` | String | `String` | `Reward.redemptionType.name()` | response: UserRewardGetDto.redemptionType |
| `vendorId` | Long | `Long` | `VendorRedemption.vendorId` | response: UserRewardGetDto.vendorId |
| `categoryIds` | Array | `List<Long>` | from RewardCategory join | response: categoryList (requires reverse lookup for display names) |
| `priority` | Int32 | `Integer` | `Reward.priority` | response: UserRewardGetDto.priority |
| `rewardRank` | Int32 | `Integer` | `Reward.rewardRank` | response: UserRewardGetDto.rewardRank |
| `programId` | Long | `Long` | `Reward.programId` | response: UserRewardGetDto.programId |
| `thumbnailPath` | String | `String` | `RewardDetails.thumbnailPath` | response: UserRewardGetDto.thumbnailUrl |
| `thumbnailId` | String | `String` | `RewardDetails.thumbnailId` | response: UserRewardGetDto.thumbnailId |
| `termAndConditionsPath` | String | `String` | `RewardDetails.termAndConditionsPath` | response: UserRewardGetDto.termAndConditionsUrl |
| `termAndConditionsId` | String | `String` | `RewardDetails.termAndConditions` | response: UserRewardGetDto.termAndConditionsId |
| `description` | String | `String` | `RewardDetails.description` | response: UserRewardGetDto.description |
| `imageId` | String | `String` | `RewardDetails.imageId` | response: UserRewardGetDto.imageId (NOTE: this is IMAGE_URI, the reference ID) |

**Note:** Category display names (`CategoryDto.name`) require a reverse lookup from categoryId → name. This can be done via CategoryRepository in the service layer, or by denormalising the category names into the catalogue doc. For POC: store categoryIds in doc, fetch names via CategoryRepository in `UserRewardMongoService` (single batch call per page). If performance is acceptable, promote to stored names in a later iteration.

---

## 7. Index Design (Phase 3)

All indexes on `active_rewards`. `orgId` is always the first field (tenant isolation — non-negotiable).

| Pattern | Index definition |
|---|---|
| Base: enabled + live (default listing) | `{orgId:1, isEnabled:1, startDate:1, expiresAt:1}` |
| + groupIds filter | `{orgId:1, isEnabled:1, groupIds:1, expiresAt:1}` |
| + labelIds filter | `{orgId:1, isEnabled:1, labelIds:1, expiresAt:1}` |
| + label (string field) filter | `{orgId:1, isEnabled:1, label:1, createdOn:-1}` |
| + type filter | `{orgId:1, isEnabled:1, type:1, createdOn:-1}` |
| + tier filter | `{orgId:1, isEnabled:1, tier:1, createdOn:-1}` |
| + redemptionType filter | `{orgId:1, isEnabled:1, redemptionType:1, createdOn:-1}` |
| + categoryIds filter | `{orgId:1, isEnabled:1, categoryIds:1, createdOn:-1}` |
| + vendorId filter | `{orgId:1, isEnabled:1, vendorId:1, createdOn:-1}` |
| + pointsValue range | `{orgId:1, isEnabled:1, pointsValue:1}` (existing) |
| + sortBy=priority DESC | `{orgId:1, isEnabled:1, priority:-1, createdOn:-1}` |
| + sortBy=rewardRank | `{orgId:1, isEnabled:1, rewardRank:1, createdOn:-1}` |
| + createdOnDaysAgo filter | `{orgId:1, isEnabled:1, createdOn:1}` |
| Unique lookup (existing) | `{orgId:1, rewardId:1}` unique (existing) |
| TTL (existing) | `{expiresAt:1}` with expireAfterSeconds=0 (existing) |

All indexes created via `CatalogueIndexInitializer.ensureIndexes()` on `@PostConstruct` with `background: true` (async creation — does not block startup). Follow the existing pattern in `CatalogueIndexInitializer.java` (use `CompoundIndexDefinition` + `mongoTemplate.indexOps(...).ensureIndex(...)`).

---

## 8. New Component Designs (Phase 4)

### 8a. CatalogueMongoEnabledOrgs

**Package:** `com.capillary.solutions.rewards.config`  
**File:** `CatalogueMongoEnabledOrgs.java`

```java
@Component
@Slf4j
public class CatalogueMongoEnabledOrgs {

    private static final String ENV_VAR = "CATALOGUE_MONGO_ENABLED_ORGS";
    private final Set<Long> enabledOrgIds;

    public CatalogueMongoEnabledOrgs(@Value("${catalogue.mongo.enabled.orgs:}") String raw) {
        // Property bound from env var via spring.config or direct env
        Set<Long> ids = new HashSet<>();
        if (StringUtils.hasLength(raw)) {
            for (String part : raw.split(",")) {
                String trimmed = part.trim();
                if (!trimmed.isEmpty()) {
                    try {
                        ids.add(Long.parseLong(trimmed));
                    } catch (NumberFormatException e) {
                        log.warn("Invalid orgId in CATALOGUE_MONGO_ENABLED_ORGS: '{}'", trimmed);
                    }
                }
            }
        }
        this.enabledOrgIds = Collections.unmodifiableSet(ids);
        log.info("CatalogueMongoEnabledOrgs: {} orgs enabled for MongoDB path: {}", ids.size(), ids);
    }

    public boolean isEnabled(Long orgId) {
        return orgId != null && enabledOrgIds.contains(orgId);
    }
}
```

Spring property binding: `catalogue.mongo.enabled.orgs` backed by env var `CATALOGUE_MONGO_ENABLED_ORGS`. In application.properties: `catalogue.mongo.enabled.orgs=${CATALOGUE_MONGO_ENABLED_ORGS:}`.

**Startup log (mandatory):** logs which orgIds are enabled so ops can validate immediately after deploy.

---

### 8b. UserRewardMongoService

**Package:** `com.capillary.solutions.rewards.service`  
**File:** `UserRewardMongoService.java`

```java
@Service
@Slf4j
public class UserRewardMongoService {

    private static final int SKIP_GUARD = 1000;
    private static final String BACKEND_MONGO = "mongo";

    @Autowired private MongoTemplate mongoTemplate;
    @Autowired private RewardCatalogueMongoRepository catalogueRepo;
    @Autowired private CategoryRepository categoryRepository;
    @Autowired private GroupService groupService;
    @Autowired private MetricsService metricsService;
    @Autowired private DateTimeService dateTimeService;

    /**
     * Returns null if skip guard triggers (caller must fall back to MySQL).
     * Returns empty UserRewardGetAllResponse with pagingDto if no results.
     */
    public UserRewardGetAllResponse listForOrg(
            Long orgId, RewardFilter rewardFilter, Long customerId)
            throws MarvelException {

        int page = rewardFilter.getPage() != null ? rewardFilter.getPage() : 0;
        int size = rewardFilter.getSize() != null ? rewardFilter.getSize() : Integer.MAX_VALUE;
        int skip = page * size;

        if (skip > SKIP_GUARD) {
            log.warn("v2 skip guard triggered: orgId={} page={} size={} skip={} > {}. Falling back to MySQL.",
                orgId, page, size, skip, SKIP_GUARD);
            metricsService.addCustomParameter("catalogue.v2.skipGuardTriggered", Boolean.TRUE);
            return null; // signals fallback
        }

        long startMs = System.currentTimeMillis();
        String filterHash = buildFilterHash(orgId, rewardFilter);

        metricsService.addCustomParameter("catalogue.backend", BACKEND_MONGO);
        metricsService.addCustomParameter(NewRelicConstants.ORG_ID, orgId);
        metricsService.addCustomParameter("catalogue.filterHash", filterHash);
        metricsService.addCustomParameter("catalogue.page", page);
        metricsService.addCustomParameter("catalogue.size", size);

        log.info("v2 MongoDB path: orgId={} filterHash={} page={} size={} skip={}",
            orgId, filterHash, page, size, skip);

        Criteria criteria = buildCriteria(orgId, rewardFilter);
        Query countQuery = new Query(criteria).maxTimeMsec(100);
        long total = mongoTemplate.count(countQuery, "active_rewards");

        Query query = new Query(criteria)
            .with(buildSort(rewardFilter))
            .skip(skip)
            .limit(size)
            .maxTimeMsec(200);

        List<ActiveRewardDoc> activeDocs = mongoTemplate.find(query, ActiveRewardDoc.class, "active_rewards");
        int dbLoadCount = activeDocs.size();

        List<Long> rewardIds = new ArrayList<>(dbLoadCount);
        for (ActiveRewardDoc doc : activeDocs) {
            rewardIds.add(doc.getRewardId());
        }

        List<RewardCatalogueDoc> catalogueDocs = rewardIds.isEmpty()
            ? Collections.emptyList()
            : catalogueRepo.findByOrgIdAndRewardIdIn(orgId, rewardIds);

        Map<Long, RewardCatalogueDoc> catalogueById = new HashMap<>(catalogueDocs.size() * 2);
        for (RewardCatalogueDoc doc : catalogueDocs) {
            catalogueById.put(doc.getRewardId(), doc);
        }

        String lang = rewardFilter.getLanguage();

        List<UserRewardGetDto> dtos = buildDtos(activeDocs, catalogueById, lang, orgId);

        long elapsed = System.currentTimeMillis() - startMs;
        metricsService.addCustomParameter("catalogue.elapsedMs", elapsed);
        metricsService.addCustomParameter("catalogue.resultCount", dtos.size());
        metricsService.addCustomParameter("db.loadCount", dbLoadCount);
        metricsService.addCustomParameter("response.count", dtos.size());

        log.info("v2 MongoDB exit: orgId={} dbLoadCount={} responseCount={} elapsedMs={}",
            orgId, dbLoadCount, dtos.size(), elapsed);

        PagingDto pagingDto = buildPagingDto(page, size, total, dtos.size());

        return UserRewardGetAllResponse.builder()
            .status(new StatusDto(true, USER_REWARD_FETCHED.code(), USER_REWARD_FETCHED.msg()))
            .rewardList(dtos.isEmpty() ? null : dtos)
            .pagingDto(pagingDto)
            .build();
    }
    // ... buildCriteria(), buildSort(), buildDtos(), buildPagingDto() methods below
}
```

**buildCriteria() logic (method signature and predicate table):**

```java
private Criteria buildCriteria(Long orgId, RewardFilter rewardFilter) throws MarvelException {
    // orgId always first — non-negotiable for tenant isolation
    Criteria c = Criteria.where("orgId").is(orgId);

    // Activation status
    if (rewardFilter.activationCheckNeeded()) {
        c = c.and("isEnabled").is(rewardFilter.fetchActivationFlag());
    }

    // Status filter (LIVE / UPCOMING / ENDED)
    Date now = dateTimeService.getCurrentDate();
    if (!rewardFilter.statusParamNotPassed()) {
        List<Criteria> statusCriterias = new ArrayList<>();
        if (rewardFilter.liveStatus()) {
            statusCriterias.add(new Criteria().andOperator(
                Criteria.where("startDate").lte(now),
                Criteria.where("expiresAt").gte(now)));
        }
        if (rewardFilter.upcomingStatus()) {
            statusCriterias.add(Criteria.where("startDate").gt(now));
        }
        if (rewardFilter.expiredStatus()) {
            // expired docs may not be in active_rewards (TTL removes them)
            // best-effort: query with expiresAt < now
            statusCriterias.add(Criteria.where("expiresAt").lt(now));
        }
        if (!statusCriterias.isEmpty()) {
            c = c.orOperator(statusCriterias.toArray(new Criteria[0]));
        }
    }

    // groupIds (from groupName param — resolve name → ID)
    if (!rewardFilter.groupList().isEmpty()) {
        List<Long> groupIds = resolveGroupIds(orgId, rewardFilter.groupList());
        c = c.and("groupIds").in(groupIds);
    }

    // labelIds (from filter.label param on Reward.LABEL field — string direct match)
    if (StringUtils.hasLength(rewardFilter.getLabel())) {
        c = c.and("label").is(rewardFilter.getLabel());
    }

    // tier
    if (StringUtils.hasLength(rewardFilter.getTier())) {
        c = c.and("tier").is(rewardFilter.getTier());
    }

    // type
    if (StringUtils.hasLength(rewardFilter.getType())) {
        c = c.and("type").is(rewardFilter.getType());
    }

    // redemptionType
    if (!rewardFilter.rewardTypeList().isEmpty()) {
        List<String> redemptionTypeNames = rewardFilter.rewardTypeList().stream()
            .map(Enum::name).collect(Collectors.toList());
        c = c.and("redemptionType").in(redemptionTypeNames);
    }

    // vendorId
    if (!rewardFilter.vendorList().isEmpty()) {
        c = c.and("vendorId").in(rewardFilter.vendorList());
    }

    // categoryIds (resolve name → IDs)
    if (!rewardFilter.categoryList().isEmpty()) {
        List<Long> catIds = resolveCategoryIds(orgId, rewardFilter.categoryList());
        c = c.and("categoryIds").in(catIds);
    }

    // pointsValue range (minPoints/maxPoints)
    if (rewardFilter.getMinPoints() != null) {
        c = c.and("pointsValue").gte(rewardFilter.getMinPoints().intValue());
    }
    if (rewardFilter.getMaxPoints() != null) {
        c = c.and("pointsValue").lte(rewardFilter.getMaxPoints().intValue());
    }

    // createdOnDaysAgo
    if (rewardFilter.getCreatedOnDaysAgo() != null) {
        Date cutoff = dateTimeService.getDateNDaysAgo(rewardFilter.getCreatedOnDaysAgo());
        c = c.and("createdOn").lte(cutoff);
    }

    return c;
}
```

**buildSort() logic:**

```java
private Sort buildSort(RewardFilter rewardFilter) {
    Sort.Direction dir = "ASC".equalsIgnoreCase(rewardFilter.orderBy())
        ? Sort.Direction.ASC : Sort.Direction.DESC;
    String sortBy = rewardFilter.sortBy(); // returns "createdOn" by default
    switch (sortBy) {
        case "priority":   return Sort.by(dir, "priority").and(Sort.by(Sort.Direction.DESC, "createdOn"));
        case "rewardRank": return Sort.by(dir, "rewardRank").and(Sort.by(Sort.Direction.DESC, "createdOn"));
        case "intouchPoints":
        case "points":     return Sort.by(dir, "pointsValue");
        default:           return Sort.by(dir, "createdOn");
    }
}
```

**buildDtos() approach:**

For each `ActiveRewardDoc`, look up the paired `RewardCatalogueDoc` by rewardId. Build `UserRewardGetDto` from both docs:
- `id` ← `activeDoc.rewardId`
- `name` ← `catalogueDoc.langDetails.get(lang)` or default language fallback
- `description` ← `catalogueDoc.description`
- `imageId` ← `catalogueDoc.imageId`
- `imageUrl` ← `catalogueDoc.imageUrl` (already the IMAGE_PATH value)
- `thumbnailId` ← `catalogueDoc.thumbnailId`
- `thumbnailUrl` ← `catalogueDoc.thumbnailPath`
- `termAndConditionsId` ← `catalogueDoc.termAndConditionsId`
- `termAndConditionsUrl` ← `catalogueDoc.termAndConditionsPath`
- `tier` ← `activeDoc.tier`
- `label` ← `activeDoc.label`
- `priority` ← `activeDoc.priority`
- `intouchPoints` ← `activeDoc.pointsValue`
- `group` ← `activeDoc.group`
- `startTime` ← `SimpleDateFormat(DATE_FORMAT).format(catalogueDoc.startDate)` (use same format as UserRewardHelper)
- `endTime` ← `SimpleDateFormat(DATE_FORMAT).format(activeDoc.expiresAt)`
- `startDateTime` ← `Utils.getDateInFormatInUTC(catalogueDoc.startDate)`
- `endDateTime` ← `Utils.getDateInFormatInUTC(activeDoc.expiresAt)`
- `expired` ← `now.after(activeDoc.expiresAt)`
- `started` ← `now.after(catalogueDoc.startDate)`
- `enabled` ← `activeDoc.isEnabled`
- `programId` ← `catalogueDoc.programId`
- `customFields` ← `catalogueDoc.customFields` (already Map<String,Object> → convert to Map<String,String>)
- `redemptionType` ← `RewardRedemptionType.valueOf(activeDoc.redemptionType)` (null-safe)
- `vendorId` ← `activeDoc.vendorId`
- `status` ← `Utils.getRewardStatus(startDate, expiresAt)` (computed, same as MySQL path)
- `categoryList` ← fetch category names from CategoryRepository by categoryIds (single batch call per page)
- `rewardRank` ← `activeDoc.rewardRank`
- fields NOT available in v2 (null in response): `loyaltyProgramCriteria`, `appliedPromotions`, `groups` (GroupingRO), `images`, `videos`, `rewardRevenueDetails`, `paymentConfigs`, `cardSeries`, `labels` (label IDs), `richContentRO`

**IMPORTANT:** Fields set to null are rendered as absent in JSON due to `@JsonInclude(NON_NULL)` on those fields — no contract break. Tech-detailer must verify each omitted field's `@JsonInclude` annotation in `UserRewardGetDto.java` before implementation.

---

### 8c. UserRewardResource.getAllForBrand Modification

**File:** `UserRewardResource.java:46-48`

```java
@Trace(dispatcher = true)
@Path("reward/brand/{brandName}")
@GET
@Produces(value = MediaType.APPLICATION_JSON_VALUE)
public Response getAllForBrand(
        @DefaultValue("false") @QueryParam("includeExpired") boolean includeExpired,
        @PathParam("brandName") String brandName,
        @DefaultValue("false") @QueryParam("v2") boolean v2,
        @Context UriInfo uriInfo,
        @BeanParam @Valid RewardFilter rewardFilter) {

    if (v2) {
        Long orgId = authDetailsProvider.get().getOrgId(); // reads X-CAP-API-AUTH-ORG-ID header
        if (mongoEnabledOrgs.isEnabled(orgId) && isV2Eligible(rewardFilter)) {
            Response mongoResponse = rewardFacade.getAllForBrandV2(orgId, rewardFilter);
            if (mongoResponse != null) {
                return mongoResponse;
            }
            // null = skip guard triggered → fall through to MySQL
        }
    }
    metricsService.addCustomParameter("catalogue.backend", "mysql");
    return rewardFacade.getAllForBrand(brandName, uriInfo, includeExpired, rewardFilter);
}

private boolean isV2Eligible(RewardFilter rewardFilter) {
    // Fallback to MySQL if any unsupported filter is active
    if (rewardFilter.getMinCash() != null || rewardFilter.getMaxCash() != null) return false;
    if (!rewardFilter.areaList().isEmpty()) return false;
    if (!rewardFilter.cityList().isEmpty()) return false;
    if (!rewardFilter.countryList().isEmpty()) return false;
    if (rewardFilter.filterBy() != null) return false;
    if (rewardFilter.isSortOnGroups()) return false;
    if (rewardFilter.getUserId() != null) return false;
    if (rewardFilter.isIncludeExpired()) return false;
    return true;
}
```

`@Autowired CatalogueMongoEnabledOrgs mongoEnabledOrgs` added to `UserRewardResource`.

The `rewardFacade.getAllForBrandV2(orgId, rewardFilter)` delegates to `UserRewardMongoService.listForOrg()`. If it returns null (skip guard), `getAllForBrandV2` returns null, and the resource falls back to MySQL.

---

### 8d. Data Correctness Sampling

When v2=true for an enabled org, optionally also run MySQL query async and compare result counts. Controlled by an additional env var or property: `catalogue.mongo.sampling.rate` (0.0–1.0). Default 0.0 (disabled). When enabled, the comparison is fire-and-forget on a dedicated `ThreadPoolTaskExecutor` (same pattern as `compareRewardsSummaryExecutor` in `RewardFacade.java:686`).

```java
// In rewardFacade.getAllForBrandV2() — after mongo result:
if (samplingEnabled && ThreadLocalRandom.current().nextDouble() < samplingRate) {
    Long finalOrgId = orgId;
    int mongoCount = dtos.size();
    compareExecutor.execute(() -> {
        try {
            List<RewardSummary> mysqlResult = getRewardSummary(null, finalOrgId, new PagingDto(),
                false, rewardFilter, null, null, false);
            if (mysqlResult.size() != mongoCount) {
                log.warn("catalogue.countMismatch: orgId={} filterHash={} mongoCount={} mysqlCount={}",
                    finalOrgId, filterHash, mongoCount, mysqlResult.size());
                metricsService.addCustomEvent("CatalogueCountMismatch", Map.of(
                    "orgId", finalOrgId,
                    "filterHash", filterHash,
                    "mongoCount", mongoCount,
                    "mysqlCount", mysqlResult.size()
                ));
            }
        } catch (Exception e) {
            log.warn("Correctness sampling failed for orgId={}", finalOrgId, e);
        }
    });
}
```

---

## 9. Sync Listener Expansion (Phase 5)

**File:** `RewardCatalogueSyncListener.java`

The `processUpserts()` method (lines 100–159) must be expanded to populate the new fields. Currently it fetches:
- `Reward` list (all basic fields)
- `RewardGroupMapping` (groupIds)
- `Label` (labelIds)
- `RewardDetails` (default lang name, imagePath)
- `CustomFieldMapping` (customFields map)

**Additional data needed for new fields:**

| New field(s) | Source | Already fetched? | Action |
|---|---|---|---|
| `group`, `label`, `tier`, `type`, `redemptionType`, `priority`, `rewardRank`, `programId`, `createdOn` | `Reward` entity fields | YES — `rewards` list already fetched | Just read from `Reward` object in `buildActiveDoc()` and `buildCatalogueDoc()` |
| `vendorId` | `Reward.vendorRedemptionId` → `VendorRedemption.vendorId` | NO | Add: `List<VendorRedemption> vendorRedemptions = vendorRedemptionRepository.findByOrgIdAndIdIn(orgId, vendorRedemptionIds)`. Then build `Map<Long, Long> vendorIdByVendorRedemptionId`. In build methods: `vendorId = vendorIdByRedemptionId.get(reward.getVendorRedemptionId())`. |
| `categoryIds` | `RewardCategory.categoryId` list for each reward | NO | Add: `List<RewardCategory> rewardCategories = rewardCategoryRepository.findByOrgIdAndRewardIdIn(orgId, foundIds)`. Build `Map<Long, List<Long>> categoryIdsByReward`. |
| `startDate` | `Reward.startDate` | YES (in Reward entity) | Just read in build methods |
| `createdOn` | `Reward.createdOn` | YES (in Reward entity) | Just read in build methods |
| `description`, `thumbnailPath`, `thumbnailId`, `termAndConditionsPath`, `termAndConditionsId` | `RewardDetails` fields | PARTIAL — only default lang | `getRewardDetails()` already fetches default lang `RewardDetails`; those fields are present in `RewardDetails` entity. Just read them. |

**New repository injections needed in `RewardCatalogueSyncListener`:**
- `@Autowired VendorRedemptionRepository vendorRedemptionRepository` (read vendor ID)
- `@Autowired RewardCategoryRepository rewardCategoryRepository` (read categoryIds)

**Updated `buildActiveDoc()` signature and additional fields:**
```java
private ActiveRewardDoc buildActiveDoc(Long orgId, Reward reward,
        List<Long> groupIds, List<Long> labelIds,
        Long vendorId, List<Long> categoryIds) {
    return ActiveRewardDoc.builder()
        .orgId(orgId)
        .rewardId(reward.getId())
        .expiresAt(reward.getEndDate())
        .startDate(reward.getStartDate())          // NEW
        .isEnabled(reward.isEnabled())
        .groupIds(groupIds)
        .labelIds(labelIds)
        .pointsValue(reward.getIntouchPoints())
        .group(reward.getGroup())                  // NEW
        .label(reward.getLabel())                  // NEW
        .tier(reward.getTier())                    // NEW
        .type(reward.getType())                    // NEW
        .redemptionType(reward.getRedemptionType() != null
            ? reward.getRedemptionType().name() : null)  // NEW
        .vendorId(vendorId)                        // NEW
        .categoryIds(categoryIds)                  // NEW
        .priority(reward.getPriority())            // NEW
        .rewardRank(reward.getRewardRank())        // NEW
        .createdOn(reward.getCreatedOn())          // NEW
        .build();
}
```

**Updated `buildCatalogueDoc()` similarly** adds: `group`, `label`, `tier`, `type`, `redemptionType`, `vendorId`, `categoryIds`, `priority`, `rewardRank`, `programId`, `description`, `thumbnailPath`, `thumbnailId`, `termAndConditionsPath`, `termAndConditionsId`, `imageId`.

---

## 10. Observability Design (Phase 6)

### New Relic Custom Attributes (added to `NewRelicConstants.java`)

```java
public static final String CATALOGUE_BACKEND       = "catalogue.backend";        // "mongo" or "mysql"
public static final String CATALOGUE_FILTER_HASH   = "catalogue.filterHash";
public static final String CATALOGUE_ELAPSED_MS    = "catalogue.elapsedMs";
public static final String CATALOGUE_RESULT_COUNT  = "catalogue.resultCount";
public static final String CATALOGUE_SKIP_GUARD    = "catalogue.v2.skipGuardTriggered";
public static final String CATALOGUE_V2_ELIGIBLE   = "catalogue.v2.eligible";
```

### Attribute Emission Points

| Where | Attributes emitted |
|---|---|
| `UserRewardResource.getAllForBrand()` — MySQL path | `catalogue.backend=mysql` |
| `UserRewardResource.getAllForBrand()` — v2 eligible check fails | `catalogue.backend=mysql`, `catalogue.v2.eligible=false` |
| `UserRewardMongoService.listForOrg()` — entry | `catalogue.backend=mongo`, `org_id`, `catalogue.filterHash`, `catalogue.page`, `catalogue.size` |
| `UserRewardMongoService.listForOrg()` — exit | `catalogue.elapsedMs`, `catalogue.resultCount`, `db.loadCount`, `response.count` |
| `UserRewardMongoService.listForOrg()` — skip guard | `catalogue.v2.skipGuardTriggered=true` |
| Correctness sampling | Custom event `CatalogueCountMismatch` with orgId, filterHash, mongoCount, mysqlCount |

### Log Lines

**Entry (INFO):**
```
v2 MongoDB path: orgId={} filterHash={} page={} size={} skip={}
```

**Exit (INFO):**
```
v2 MongoDB exit: orgId={} dbLoadCount={} responseCount={} elapsedMs={}
```

**Skip guard fallback (WARN):**
```
v2 skip guard triggered: orgId={} page={} size={} skip={} > {}. Falling back to MySQL.
```

**Count mismatch (WARN):**
```
catalogue.countMismatch: orgId={} filterHash={} mongoCount={} mysqlCount={}
```

**Ineligible filter fallback (INFO):**
```
v2 ineligible: orgId={} reason={unsupported filter: minCash/area/city/country/filterBy/sortOnGroups/userId/includeExpired}. Using MySQL.
```

### Dashboard Metric Design

- **`catalogue.backend` distribution per org** — query NR for `catalogue.backend` attribute, group by `org_id`. Shows mongo vs mysql split per org over time.
- **`catalogue.elapsedMs` histogram** — p50/p95/p99 per backend per org.
- **`CatalogueCountMismatch` event count** — alert if > 0 in any 5-minute window (data correctness alarm).
- **`catalogue.v2.skipGuardTriggered` count** — alert if spikes (indicates traffic pattern shifted to deep pagination).

### Response Header

Add `X-CAP-Data-Freshness: eventual` on all v2=mongo responses (indicates async-sync data, not real-time MySQL). This is informational; clients are not expected to act on it.

---

## 11. Risk Assessment (Phase 7)

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Deep skip degrades MongoDB performance | Medium | High — O(N) scan for large N | Guard: skip > 1000 → WARN + fallback to MySQL. Document O(N) degradation. |
| New fields missing from sync → wrong filter results | High | High — silent wrong results | Integration test: seed known reward with all filter fields set, verify each filter returns only expected rewards. |
| Sync lag → stale results during high write period | Medium | Medium — acceptable for POC | Document in `X-CAP-Data-Freshness: eventual` response header. DLQ alerts are existing mechanism for sync failure. |
| Response shape mismatch breaks clients | High | Critical — silent breakage | Integration test: compare JSON field-by-field against MySQL response for same org + same filter (data correctness sampling). |
| Null fields in v2 response where MySQL populated them | High | Medium — degraded response | Tech-detailer must enumerate which fields are null in v2 and verify all have `@JsonInclude(NON_NULL)` in `UserRewardGetDto`. |
| `brandName` still in URL path (legacy callers) | Low | Low | v2 path ignores brandName and uses orgId from header. MySQL path unchanged. |
| `CATALOGUE_MONGO_ENABLED_ORGS` env var misconfigured | Low | Low | Logged on startup: `CatalogueMongoEnabledOrgs: N orgs enabled: [ids]`. |
| Processor chain not run on v2 path | High | Medium — missing segment/restriction data | v2 path bypasses `GetUserRewardsProcessor` chain. For POC this is accepted. If any processor is critical, fallback to MySQL. Tech-detailer to validate processor list and classify each as critical vs nice-to-have. |
| CategoryRepository call per v2 page adds latency | Medium | Low | Single batch call per page (not per reward). Monitor via `catalogue.elapsedMs`. |
| VendorRedemption fetch in sync adds latency | Low | Low | Batch fetch via `findByOrgIdAndIdIn`. Negligible for typical sync batch sizes (1–50 rewards). |
| SimpleDateFormat is not thread-safe | High | Critical | Do NOT use in service singleton. Use `Utils.getDateInFormatInUTC()` for ISO strings; instantiate `SimpleDateFormat` per call or use `DateTimeFormatter` (thread-safe). Guardrail from `code.md`: "Never instantiate new SimpleDateFormat(...) in service/processor/DAO code". |

---

## 12. Handoff Notes for Tech-Detailer

### What to validate before writing code

1. **`@JsonInclude` audit on `UserRewardGetDto`** — verify every field that v2 omits (loyaltyProgramCriteria, appliedPromotions, groups-as-GroupingRO, images, videos, rewardRevenueDetails, paymentConfigs, cardSeries, richContentRO) has `@JsonInclude(Include.NON_NULL)`. If any does NOT, clients will receive `null` in JSON which is a contract break. Those fields must be backfilled or the response shape must be documented as diverging.

2. **Processor chain audit** — list all processors registered in `getAllForOrgUserRewardsProcessors` (set at `RewardFacade.java:237`). For each, classify: (a) is it a filter that removes rewards? (b) is it a data enricher? Processors that remove rewards (e.g. segment validation at line 1835) are NOT run in v2 path. If segment filtering is critical for a given org, that org should NOT be enabled. Document which processors are bypassed.

3. **`authDetailsProvider` in `UserRewardResource`** — verify `authDetailsProvider` is accessible at the resource layer (it is injected into `RewardFacade`; check if it's also injectable into the resource, or if the orgId must be passed differently). Current pattern: `rewardFacade.getAllForBrand()` calls `authDetailsProvider.get().getOrgId()` internally. For v2, resource needs orgId before calling facade. Two options: (a) inject `authDetailsProvider` into resource (safe, it's request-scoped), or (b) pass brandName to a facade method that resolves orgId internally (cleaner boundary). Option (a) is recommended — same pattern as other facade methods.

4. **CategoryRepository batch lookup** — confirm `categoryRepository.getId(orgId, categoryName)` returns a `List<Long>` (it does, per `RewardSpecificationService.java:52`). Use this for category name→ID resolution in both sync listener and query service.

5. **VendorRedemptionRepository** — confirm this repository exists and has `findByOrgIdAndIdIn()`. Check `com.capillary.solutions.rewards.repository.*`.

6. **Date format consistency** — `UserRewardHelper.buildUserRewardGetDto()` uses `SimpleDateFormat(DATE_FORMAT)` for `startTime`/`endTime` and `Utils.getDateInFormatInUTC()` for `startDateTime`/`endDateTime`. The v2 builder must use the same formatting. Never use `SimpleDateFormat` in a Spring singleton — instantiate per request or use `DateTimeFormatter`. MADR-0002 + `code.md` date-time guardrails apply.

6. **Custom fields response** — `catalogueDoc.customFields` is `Map<String, Object>`. `UserRewardGetDto.customFields` is `Map<String, String>`. Convert: for each entry, call `.toString()` on value, null-safe.

### What to flesh out

- `buildPagingDto()` in `UserRewardMongoService`: replicate the logic in `RewardFacade.getRewardsBasedOnPagination()` (lines 1210–1244). Same fields: `totalElements`, `size`, `number`, `totalPages`, `first`, `last`, `numberOfElements`.
- Error handling: what if MongoDB is unreachable during a v2 request? Catch `DataAccessException` / `MongoException` and fall back to MySQL with a WARN log.
- `maxTimeMsec(200)` on main query — validate with actual org data sizes. If orgs have > 10k docs, 200ms may be too tight for sort+skip scenarios. Start with 500ms for the POC.

### Tests to emphasise

- **V2 filter coverage IT**: for each supported filter (tier, label, group, type, redemptionType, vendorId, category, minPoints, maxPoints, createdOnDaysAgo), seed two rewards — one matching, one not. Verify v2 returns only the matching one.
- **Fallback IT**: for each unsupported filter (area, city, country, filterBy, sortOnGroups, userId, includeExpired, minCash, maxCash), verify request falls back to MySQL (check `catalogue.backend=mysql` NR attribute or response header).
- **Skip guard IT**: page=2, size=501 (skip=1002) — verify fallback to MySQL and WARN log.
- **Response shape IT**: same org, same filter, compare v2 and MySQL response JSON field-by-field. Focus on fields that v2 omits (must be absent, not null).
- **Data correctness IT**: seed known reward with all new fields (tier, label, type, vendorId, categoryId). Sync via RabbitMQ event. Verify MongoDB doc has all fields. Then query v2 with each filter; verify correct reward returned.
- **Disabled org IT**: v2=true for org NOT in CATALOGUE_MONGO_ENABLED_ORGS → MySQL path used.
- **Org enabled but ineligible filter IT**: org in allowlist, but v2=true with area filter → MySQL path.
- **SimpleDateFormat thread safety**: verify no instance of `SimpleDateFormat` created in any service/repository singleton. Use `DateTimeFormatter` (thread-safe) in the new service.

---

## 13. Files to Create / Modify

### New files

| File | Package | Description |
|---|---|---|
| `CatalogueMongoEnabledOrgs.java` | `config` | Env var reader for enabled orgs |
| `UserRewardMongoService.java` | `service` | MongoDB query service for v2 path |

### Modified files

| File | Change |
|---|---|
| `UserRewardResource.java` | Add `@QueryParam("v2")`, routing logic, `isV2Eligible()`, autowire `CatalogueMongoEnabledOrgs` |
| `RewardFacade.java` | Add `getAllForBrandV2()` method, correctness sampling executor |
| `ActiveRewardDoc.java` | Add 11 new fields (startDate, group, label, tier, type, redemptionType, vendorId, categoryIds, priority, rewardRank, createdOn) |
| `RewardCatalogueDoc.java` | Add 16 new display fields |
| `CatalogueIndexInitializer.java` | Add 10 new compound indexes for new filter fields |
| `RewardCatalogueSyncListener.java` | Expand `processUpserts()` to populate new fields; inject VendorRedemptionRepository, RewardCategoryRepository |
| `NewRelicConstants.java` | Add catalogue.backend, catalogue.filterHash, catalogue.elapsedMs, catalogue.resultCount, catalogue.v2.skipGuardTriggered constants |
| `application.properties` | Add `catalogue.mongo.enabled.orgs=${CATALOGUE_MONGO_ENABLED_ORGS:}` and `catalogue.mongo.sampling.rate=${CATALOGUE_MONGO_SAMPLING_RATE:0.0}` |
