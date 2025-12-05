# Promotion Expiry and Archival Strategy

## Overview
This document details how expiry is maintained at different levels (Meta, Issue, Earn, Code) and provides strategies for archiving expired promotions (>3 years) without impacting database performance.

## Current Data Volume
- **Issued Promotions**: 1.2B documents (~50GB)
- **Earned Promotions**: 1.2B documents (~65GB)
- **Cart Evaluations**: 591M documents (~1.79TB)

**Goal**: Archive promotions (issued, earned, code) expired > 3 years.

---

## 1. Expiry Maintenance at Different Levels

This section explains how expiry is maintained at four levels: **Meta** (campaign definition), **Issue** (promotion issued to customer), **Earn** (promotion earned by customer), and **Code** (promo code assigned). Understanding these relationships is critical for archival strategy.

### 1.1 Meta Level (PromotionMeta)

**Collection**: `PromotionMeta`

**What Expiry Means at This Level**:
- The **campaign-level expiry** defines when the promotion campaign itself expires
- This is the **source of truth** for expiry at the meta level
- When `endDate` passes, the promotion campaign is considered expired, but individual issued/earned instances may still be valid (see relationships below)

**Expiry-Related Fields**:
- `endDate` (Date) - **Primary expiry field**: Static expiry date for the promotion campaign. This is the absolute latest date any instance of this promotion can be valid.
- `loyaltyEarningBasedValidity.loyaltyEarningExpiryDays` (Integer) - **Dynamic validity configuration**: Number of days after earning that a promotion should expire. Used only for **earned promotions** (not issued). If set, allows earned promotions to expire X days after earning rather than on the fixed `endDate`.
- `earningCriteria.earningValidTillDays` (Integer) - **Earning window expiry**: For earning promotions with specific audience (`earnLimitedToSpecificAudience=true`), this defines how long after issuance a customer can still earn the promotion. Used to validate if earning is allowed, not to set `validTill` on earned promotions.

**Key Methods**:
```java
// Check if promotion has expired (always uses static endDate)
public boolean hasExpired() {
    Date date = new Date();
    return date.after(this.getEndDate());
}

// Check if promotion uses dynamic validity
public boolean isDynamicValidity() {
    return getLoyaltyEarningBasedValidity() != null &&
           getLoyaltyEarningBasedValidity().getLoyaltyEarningExpiryDays() != null;
}

// Get valid till date considering dynamic validity (used for EARNED promotions only)
public LocalDateTime getValidTillForPromotion(TimeZone timeZone) {
    LocalDateTime validTill;
    if (loyaltyEarningBasedValidity == null || 
        loyaltyEarningBasedValidity.getLoyaltyEarningExpiryDays() == null) {
        // Static expiry: use endDate
        validTill = LocalDateTime.ofInstant(Instant.ofEpochMilli(endDate.getTime()), 
                                            ZoneId.systemDefault());
    } else {
        // Dynamic expiry: current time + expiry days (in organization's timezone)
        ZonedDateTime zonedDateTime = 
            ZonedDateTime.now(timeZone.toZoneId())
                .plusDays(loyaltyEarningBasedValidity.getLoyaltyEarningExpiryDays());
        zonedDateTime = zonedDateTime.with(LocalTime.MAX); // Set to end of day
        
        // Convert from organization timezone to system timezone
        validTill = LocalDateTime.ofInstant(zonedDateTime.toInstant(), 
                                           TimeZone.getDefault().toZoneId());
    }
    return validTill;
}
```

**Indexes**:
- `{'orgId' : 1, 'endDate': 1}` - Supports querying by expiry date
- `{'orgId' : 1, 'lastUpdatedOn' : 1}` - Supports archival queries

**How Expiry is Computed**:

1. **Static Expiry** (Default):
    - **Computation**: `hasExpired() = currentDate > endDate`
    - **When Used**: When `loyaltyEarningBasedValidity` is `null` or `loyaltyEarningExpiryDays` is `null`
    - **Behavior**: The `endDate` is a fixed calendar date that applies to all instances of the promotion
    - **Impact on Instances**: All issued and earned promotions inherit this `endDate` as their maximum validity

2. **Dynamic Expiry** (via `loyaltyEarningBasedValidity`):
    - **Purpose**: Allows **earned promotions only** to expire a fixed number of days after they are earned, rather than on a fixed calendar date
    - **Configuration**: Set via `loyaltyEarningBasedValidity.loyaltyEarningExpiryDays` (Integer)
    - **Computation**:
      ```java
      // Used in getValidTillForPromotion() - called when earning promotions
      if (loyaltyEarningBasedValidity != null && loyaltyEarningExpiryDays != null) {
          validTill = currentTime + loyaltyEarningExpiryDays  // in organization timezone
      } else {
          validTill = endDate  // fallback to static
      }
      ```
    - **Time Handling**:
        - Uses organization's timezone (`timeZone`) for calculation
        - Sets time to end of day (`LocalTime.MAX`) for the calculated date
        - Converts result back to system timezone for storage
    - **Use Case**: Enables "expire X days after earning" behavior instead of "expire on specific date"
    - **Important**: This only affects **earned promotions**. Issued promotions always use static `endDate`

**Important Notes on `getValidTillForPromotion()`**:

- **Usage**: This method is used **only for EARNED promotions** (`CustomerEarnedPromotion`), NOT for issued promotions
- **Where Used**:
    - `EarnedPromotionService.earnBulkPromotion()` - Sets `validTill` when earning promotions
    - `SaveEarnedPromotionProcessor.process()` - Sets `validTill` when saving earned promotions
- **For Issued Promotions**: The `getValidTillForPromotion()` method is **NOT used**. Issued promotions always use static `endDate` directly (see Section 1.2)
- **Timezone Handling**:
    - When dynamic validity is present, the organization's timezone is retrieved via `OrganizationService.getTimeZoneTillOrSystemDefault()`
    - The calculation uses "now" in that timezone, not the earning/issuance time
    - This means the expiry date is calculated at the time the promotion is earned, based on the current moment in the organization's timezone
- **Dynamic Validity Check**: Use `isDynamicValidity()` to determine if a promotion uses dynamic validity before calling `getValidTillForPromotion()`

**Example Scenarios**:

1. **Static Expiry**:
    - `endDate` = Dec 31, 2024
    - `loyaltyEarningBasedValidity` = `null`
    - Result: All instances expire on Dec 31, 2024, regardless of when earned

2. **Dynamic Expiry**:
    - `endDate` = Dec 31, 2024 (campaign end)
    - `loyaltyEarningBasedValidity.loyaltyEarningExpiryDays` = 30
    - Organization timezone = IST (UTC+5:30)
    - If earned on Nov 15, 2024 at 10:00 AM IST:
        - Calculation: Nov 15, 2024 23:59:59 IST + 30 days = Dec 15, 2024 23:59:59 IST
        - Result: `validTill` = Dec 15, 2024 23:59:59 (converted to system timezone)
    - Note: The actual expiry is the minimum of (calculated dynamic expiry, campaign endDate) - this is handled at the earned promotion level

**Relationship to Other Levels**:
- **To Issued Promotions**: `CustomerIssuedPromotion.validTill` is always set to `PromotionMeta.endDate` (direct copy, no calculation)
- **To Earned Promotions**: `CustomerEarnedPromotion.validTill` may use dynamic calculation if `loyaltyEarningBasedValidity` is set, but will never exceed `endDate`
- **Can Instances Be Valid After Meta Expires?**: **NO** - When `endDate` passes, the meta is expired. However, individual earned promotions with dynamic validity may have `validTill` dates that were calculated before meta expiry, but they cannot exceed `endDate`. In practice, all instances should be expired when meta expires, but there may be edge cases where `validTill` was set incorrectly.

**Note**: While `PromotionMeta` has methods like `getValidTillForPromotion()` that handle dynamic validity, these are **NOT used for issued promotions**. Issued promotions always use the static `endDate` regardless of `loyaltyEarningBasedValidity` settings.

---

### 1.2 Issue Level (CustomerIssuedPromotion)

**Collection**: `CustomerIssuedPromotion`

**What Expiry Means at This Level**:
- The **instance-level expiry** for a promotion that has been issued to a specific customer
- Represents when this particular issued promotion instance becomes invalid
- This is a **locked value** set at issuance time and updated only when the meta `endDate` changes

**Expiry-Related Fields**:
- `validTill` (Date) - **Expiry date for this issued promotion instance**. This is the date after which this issued promotion is no longer valid.
- `issuedTime` (Date) - When the promotion was issued to the customer (used for reference, not for expiry calculation)
- `active` (Boolean) - Whether the promotion is still active (soft delete flag)

**Key Methods**:
```java
// Query active issued promotions with validTill > date
findAllByOrgIdAndCustomerIdAndValidTillGreaterThanAndActiveIsTrue(
    Long orgId, Long customerId, Date date)
```

**Indexes**:
- `{'orgId' : 1, 'customerId': 1, 'promotionId': 1, 'validTill': 1}` - Supports customer queries
- `{'orgId' : 1, 'promotionId': 1, '_id': 1}` - Supports promotion-based queries
- `{'orgId' : 1, 'lastUpdatedOn' : 1}` - Supports archival queries

**How Expiry is Computed**:

1. **Initial Issuance** (in `IssuedPromotionService.issueBulkPromotion()`):
   ```java
   CustomerIssuedPromotion.builder()
       .validTill(promotionMeta.getEndDate())  // Direct assignment from PromotionMeta
       .issuedTime(currentDate)
       // ... other fields
       .build()
   ```
    - **Computation**: `validTill = PromotionMeta.endDate` (direct copy, no calculation)
    - **No dynamic calculation**: Unlike earned promotions, issued promotions do NOT use `loyaltyEarningBasedValidity` or any time-based calculations
    - **Static value**: The `validTill` is set once at issuance and remains static regardless of when the promotion was issued
    - **Expiry check**: `isExpired = currentDate > validTill`

2. **Expiry Date Updates** (when `PromotionMeta.endDate` changes):
    - When the promotion's `endDate` is updated in `PromotionMeta`, all existing issued promotions for that promotion are updated via batch processing
    - Implementation: `AbstractExpiryDateChangeDao.updateValidTillDate()`
   ```java
   // Updates all issued promotions for a given promotionId
   update.set("validTill", promotionMeta.getEndDate());
   mongoOperation.updateMulti(query, update, CustomerIssuedPromotion.class);
   ```
    - Uses cursor-based pagination to process in batches (default: 1000 documents per batch)
    - Updates all `CustomerIssuedPromotion` documents where `promotionId` matches
    - The update is performed asynchronously via `ExpiryDateChangeJob`

**Relationship to Other Levels**:
- **From Meta Level**: `validTill` is always equal to `PromotionMeta.endDate` (direct relationship, no transformation)
- **To Earn Level**: When an issued promotion is converted to earned, the earned promotion may use different expiry calculation (see Earn Level)
- **Can Be Valid After Meta Expires?**: **NO** - Since `validTill = PromotionMeta.endDate`, when meta expires, all issued promotions are also expired. They are always in sync.

**Use Cases**:
- **Bulk Issuance**: Promotions issued to customers in bulk campaigns
- **Pre-issued Promotions**: Promotions given to customers before they meet earning criteria
- **Static Expiry**: All issued promotions expire on the same date (campaign end date)

**Critical Notes**:
- Issued promotions **never** use dynamic validity, even if `loyaltyEarningBasedValidity` is set in `PromotionMeta`
- The `issuedTime` field is for tracking purposes only and does not affect `validTill` calculation
- When querying for archival, use `validTill < archiveCutoffDate` to find expired issued promotions

---

### 1.3 Earn Level (CustomerEarnedPromotion)

**Collection**: `CustomerEarnedPromotion`

**What Expiry Means at This Level**:
- The **instance-level expiry** for a promotion that has been earned by a specific customer
- Represents when this particular earned promotion instance becomes invalid and can no longer be redeemed
- This value is **calculated at earning time** and may differ from the meta `endDate` if dynamic validity is configured

**Expiry-Related Fields**:
- `validTill` (LocalDateTime) - **Expiry date/time for this earned promotion instance**. This is calculated when the promotion is earned and represents the latest date/time this earned promotion can be used.
- `eventTime` (LocalDateTime) - When the promotion was earned (used for reference and some calculations)
- `active` (Boolean) - Whether the promotion is still active (soft delete flag)

**Key Methods**:
```java
// Query active earned promotions with validTill > date
findAllByOrgIdAndCustomerIdAndValidTillGreaterThanAndActiveIsTrue(
    Long orgId, Long customerId, Date date)
```

**Indexes**:
- `{'orgId' : 1, 'customerId': 1, 'validTill': 1}` - Supports customer queries
- `{'orgId' : 1, 'promotionId': 1, '_id': 1}` - Supports promotion-based queries
- `{'orgId' : 1, 'lastUpdatedOn' : 1}` - Supports archival queries

**How Expiry is Computed**:

The computation depends on whether the promotion uses dynamic validity (`loyaltyEarningBasedValidity`):

1. **With Dynamic Validity** (`loyaltyEarningBasedValidity.loyaltyEarningExpiryDays` is set):
   ```java
   // Called in EarnedPromotionService.earnBulkPromotion() and SaveEarnedPromotionProcessor
   LocalDateTime validTill = promotionMeta.getValidTillForPromotion(timeZone);
   // Where getValidTillForPromotion() calculates:
   // validTill = currentTime + loyaltyEarningExpiryDays (in org timezone)
   // Note: This is calculated at earning time, so it's based on "now" when earned
   ```
    - **Computation**: `validTill = earningTime + loyaltyEarningExpiryDays` (in organization timezone)
    - **Capped by endDate**: The calculated value should not exceed `PromotionMeta.endDate`, but the actual capping happens at the application level during redemption checks
    - **Expiry check**: `isExpired = currentDateTime > validTill`

2. **Without Dynamic Validity** (static expiry):
   ```java
   // Same method, but returns endDate when loyaltyEarningBasedValidity is null
   LocalDateTime validTill = promotionMeta.getValidTillForPromotion(timeZone);
   // Returns: LocalDateTime.ofInstant(endDate.toInstant(), ZoneId.systemDefault())
   ```
    - **Computation**: `validTill = PromotionMeta.endDate` (converted to LocalDateTime)
    - **Expiry check**: `isExpired = currentDateTime > validTill`

3. **Special Case: Earning Valid Till Days** (`earningCriteria.earningValidTillDays`):
    - This field is used to validate **when a customer can earn** a promotion (not to set `validTill` on earned promotions)
    - Used in `PromotionMeta.getActiveEarningExpiryDate()`:
   ```java
   // Used to check if earning is still allowed (not for setting validTill)
   LocalDateTime promotionEndDate = promotionMeta.getEndDate();
   LocalDateTime earningValidTillDate = issuedTime.plusDays(earningValidTillDays);
   LocalDateTime activeEarningExpiryDate = min(promotionEndDate, earningValidTillDate);
   // If eventTime > activeEarningExpiryDate, earning is rejected
   ```
    - **Purpose**: Prevents customers from earning promotions too late (e.g., must earn within 7 days of issuance)
    - **Note**: This does NOT affect the `validTill` of earned promotions, only whether earning is allowed

**Relationship to Other Levels**:
- **From Meta Level**:
    - If dynamic validity is set: `validTill` is calculated as `earningTime + loyaltyEarningExpiryDays`, but should not exceed `PromotionMeta.endDate`
    - If static: `validTill = PromotionMeta.endDate`
- **From Issue Level**: When an issued promotion is converted to earned, the earned promotion uses its own expiry calculation (may differ from issued `validTill`)
- **Can Be Valid After Meta Expires?**: **POTENTIALLY YES (Edge Case)** - If a promotion with dynamic validity was earned close to the meta `endDate`, the calculated `validTill` might be set to a date after `endDate` due to the calculation using "now + days". However, redemption checks should validate against both `validTill` and meta `endDate`. For archival purposes, we should consider both conditions.

**Use Cases**:
- **Dynamic Expiry**: "Expire 30 days after earning" - allows customers who earn later to have valid promotions longer
- **Static Expiry**: All earned promotions expire on campaign end date regardless of when earned
- **Earning Window**: `earningValidTillDays` ensures customers can only earn within a certain window after issuance

**Critical Notes**:
- Earned promotions are the **only** level that can use dynamic validity calculations
- The `validTill` is calculated **once at earning time** and stored - it does not recalculate
- For archival, query using `validTill < archiveCutoffDate`, but also consider checking against `PromotionMeta.endDate` to catch edge cases
- When `PromotionMeta.endDate` changes, earned promotions are updated via `ExpiryDateChangeJob` (similar to issued promotions)

---

### 1.4 Code Level (CodeBasedPromotion)

**Collection**: `CodeBasedPromotion`

**What Expiry Means at This Level**:
- The **instance-level expiry** for a promo code that has been assigned to a specific customer
- Represents when this particular code becomes invalid and can no longer be used for redemption
- Similar to issued promotions, this is typically set to the meta `endDate`

**Expiry-Related Fields**:
- `validTillDate` (Date) - **Expiry date for this promo code instance**. This is the date after which this code can no longer be used.
- `active` (Boolean) - Whether the code is still active (soft delete flag)
- `promoCode` (String) - The actual code string (unique identifier)

**Key Methods**:
```java
// Query active codes with validTillDate > date
findByOrgIdAndCustomerIdAndValidTillDateGreaterThanAndActiveIsTrue(
    Long orgId, Long customerId, Date date)
```

**Indexes**:
- `{'orgId' : 1, 'customerId': 1, 'validTillDate' : 1}` - Supports customer queries
- `{'orgId' : 1, 'promotionId': 1, '_id': 1}` - Supports promotion-based queries
- `{'orgId' : 1, 'promoCode': 1}` - Unique index for code lookup
- `{'orgId' : 1, 'lastUpdatedOn' : 1}` - Supports archival queries

**How Expiry is Computed**:

1. **Code Assignment** (in `CodeBasedPromotionService`):
   ```java
   CodeBasedPromotion.builder()
       .validTillDate(promotionMeta.getEndDate())  // Typically set to PromotionMeta.endDate
       // ... other fields
       .build()
   ```
    - **Computation**: `validTillDate = PromotionMeta.endDate` (typically, direct copy)
    - **Expiry check**: `isExpired = currentDate > validTillDate`
    - **Validation during cart evaluation**:
   ```java
   if (new Date().after(codeBasedPromotion.getValidTillDate()))
       // Code expired - reject redemption
   ```

**Relationship to Other Levels**:
- **From Meta Level**: `validTillDate` is typically set to `PromotionMeta.endDate` (direct relationship)
- **Can Be Valid After Meta Expires?**: **NO** - Since `validTillDate` is typically equal to `PromotionMeta.endDate`, when meta expires, all codes are also expired. They are always in sync.

**Use Cases**:
- **Promo Codes**: Unique codes assigned to customers for redemption
- **Static Expiry**: All codes expire on the campaign end date
- **Code Validation**: Codes are validated during cart evaluation to ensure they haven't expired

**Critical Notes**:
- Code-based promotions follow the same static expiry pattern as issued promotions
- The `promoCode` field is unique and used for lookup during redemption
- For archival, use `validTillDate < archiveCutoffDate` to find expired codes

---

## 1.5 Expiry Relationships Summary

**Key Relationships Between Levels**:

1. **Meta → Issue**: `CustomerIssuedPromotion.validTill = PromotionMeta.endDate` (always, direct copy)
2. **Meta → Earn**:
    - Static: `CustomerEarnedPromotion.validTill = PromotionMeta.endDate`
    - Dynamic: `CustomerEarnedPromotion.validTill = earningTime + loyaltyEarningExpiryDays` (capped by `endDate` in practice)
3. **Meta → Code**: `CodeBasedPromotion.validTillDate = PromotionMeta.endDate` (typically, direct copy)

**Can Instances Be Valid After Meta Expires?**:

- **Issued Promotions**: **NO** - Always in sync with meta `endDate`
- **Earned Promotions**: **POTENTIALLY YES (Edge Case)** - If dynamic validity was used and calculated `validTill` exceeds `endDate` due to calculation timing. However, redemption should check both conditions. For archival, consider both `validTill` and meta `endDate`.
- **Code Promotions**: **NO** - Typically always in sync with meta `endDate`

**Archival Implications**:

- When archiving, query each level independently using their `validTill`/`validTillDate` fields
- For earned promotions, also consider checking `PromotionMeta.endDate` to catch edge cases
- Meta expiry (`endDate < archiveCutoffDate`) is a good indicator, but instance-level expiry is the definitive source
- All levels can be archived independently, but meta expiry is a strong signal for bulk archival

---

## 2. Querying Expired Promotions

### 2.1 Query Patterns

#### For PromotionMeta (Meta Level)
```java
// Query expired promotions
Criteria criteria = Criteria.where("orgId").is(orgId)
    .and("endDate").lt(threeYearsAgoDate)
    .and("active").is(true);

Query query = new Query(criteria);
List<PromotionMeta> expiredPromotions = mongoTemplate.find(query, PromotionMeta.class);
```

#### For CustomerIssuedPromotion (Issue Level)
```java
// Query expired issued promotions
Date threeYearsAgo = Date.from(Instant.now().minus(3, ChronoUnit.YEARS));

// Using DAO method
List<CustomerIssuedPromotion> expired = 
    customerIssuedPromotionDao.findAllByOrgIdAndCustomerIdAndValidTillGreaterThanAndActiveIsTrue(
        orgId, customerId, threeYearsAgo);

// Custom query for all expired
Criteria criteria = Criteria.where("orgId").is(orgId)
    .and("validTill").lt(threeYearsAgo)
    .and("active").is(true);
```

#### For CustomerEarnedPromotion (Earn Level)
```java
// Query expired earned promotions
Date threeYearsAgo = Date.from(Instant.now().minus(3, ChronoUnit.YEARS));
LocalDateTime threeYearsAgoLocal = LocalDateTime.ofInstant(
    threeYearsAgo.toInstant(), ZoneId.systemDefault());

// Using QueryDSL
BooleanExpression conditions = QCustomerEarnedPromotion.customerEarnedPromotion
    .orgId.eq(orgId)
    .and(QCustomerEarnedPromotion.customerEarnedPromotion.validTill.lt(threeYearsAgoLocal))
    .and(QCustomerEarnedPromotion.customerEarnedPromotion.active.isTrue());
```

#### For CodeBasedPromotion (Code Level)
```java
// Query expired codes
Date threeYearsAgo = Date.from(Instant.now().minus(3, ChronoUnit.YEARS));

Criteria criteria = Criteria.where("orgId").is(orgId)
    .and("validTillDate").lt(threeYearsAgo)
    .and("active").is(true);
```

### 2.2 Efficient Query Strategies

1. **Use Indexed Fields**: Always query using indexed fields (`orgId`, `validTill`, `endDate`)
2. **Batch Processing**: Process in batches using cursor-based pagination
3. **Date Range Queries**: Use `$lt` (less than) for expiry queries
4. **Compound Queries**: Combine `orgId` + `validTill` for optimal index usage

---

## 3. Archival Strategies Without DB Load

### 3.1 Batch Processing with Cursor-Based Pagination

**Pattern**: Similar to `AbstractExpiryDateChangeDao` used for expiry date updates.

**Key Principles**:
- Process documents in small batches (1000-5000 per batch)
- Use `_id` cursor for pagination (ensures no duplicates/misses)
- Process during off-peak hours
- Use aggregation pipeline to identify batch boundaries

**Implementation Pattern**:
```java
public void archiveExpiredPromotions(Long orgId, Date archiveCutoffDate) {
    ObjectId startId = new ObjectId("000000000000000000000000");
    int batchSize = 1000;
    int maxIterations = 100000;
    int iterationCount = 0;
    
    while (iterationCount < maxIterations) {
        // 1. Find batch boundary using aggregation
        Aggregation aggregation = Aggregation.newAggregation(
            Aggregation.match(
                Criteria.where("orgId").is(orgId)
                    .and("validTill").lt(archiveCutoffDate)
                    .and("_id").gt(startId)
            ),
            Aggregation.sort(Sort.by(Sort.Direction.ASC, "_id")),
            Aggregation.limit(batchSize),
            Aggregation.group().last("_id").as("lastId")
        );
        
        AggregationResults<Map> results = mongoTemplate.aggregate(
            aggregation, "CustomerIssuedPromotion", Map.class);
        
        if (results.getMappedResults().isEmpty()) {
            break; // No more documents
        }
        
        ObjectId endId = (ObjectId) results.getMappedResults().get(0).get("lastId");
        
        // 2. Archive batch
        archiveBatch(orgId, startId, endId, archiveCutoffDate);
        
        // 3. Update cursor
        startId = endId;
        iterationCount++;
        
        // 4. Rate limiting - sleep between batches
        Thread.sleep(100); // 100ms delay between batches
    }
}
```

### 3.2 Strategy 1: Move to Archive Collection

**Approach**: Copy documents to archive collection, then delete from main collection.

**Advantages**:
- Data preserved for audit/compliance
- Can restore if needed
- Minimal impact on main collection queries

**Implementation**:
```java
private void archiveBatch(Long orgId, ObjectId startId, ObjectId endId, Date cutoffDate) {
    // 1. Find documents to archive
    Query query = new Query(
        Criteria.where("orgId").is(orgId)
            .and("validTill").lt(cutoffDate)
            .and("_id").gt(startId)
            .and("_id").lte(endId)
    );
    
    List<CustomerIssuedPromotion> toArchive = 
        mongoTemplate.find(query, CustomerIssuedPromotion.class);
    
    if (toArchive.isEmpty()) return;
    
    // 2. Insert into archive collection
    mongoTemplate.insertAll(toArchive, "CustomerIssuedPromotion_Archive");
    
    // 3. Delete from main collection (in batches)
    mongoTemplate.remove(query, CustomerIssuedPromotion.class);
    
    log.info("Archived {} documents from {} to {}", 
        toArchive.size(), startId, endId);
}
```

**Considerations**:
- Use `insertMany` with `ordered: false` for better performance
- Delete after successful archive
- Monitor disk space for archive collection

### 3.3 Strategy 2: Soft Delete with Archive Flag

**Approach**: Mark documents as archived instead of deleting.

**Advantages**:
- Faster (no data movement)
- Can query archived data if needed
- Reversible

**Implementation**:
```java
private void archiveBatchSoft(Long orgId, ObjectId startId, ObjectId endId, Date cutoffDate) {
    Query query = new Query(
        Criteria.where("orgId").is(orgId)
            .and("validTill").lt(cutoffDate)
            .and("_id").gt(startId)
            .and("_id").lte(endId)
            .and("archived").ne(true) // Only process non-archived
    );
    
    Update update = new Update()
        .set("archived", true)
        .set("archivedAt", new Date())
        .set("lastUpdatedOn", new Date());
    
    UpdateResult result = mongoTemplate.updateMulti(query, update, CustomerIssuedPromotion.class);
    
    log.info("Soft archived {} documents from {} to {}", 
        result.getModifiedCount(), startId, endId);
}
```

**Query Modifications Required**:
- All queries need to filter: `.and("archived").ne(true)` or `.and("archived").is(false)`
- Update indexes to include `archived` field

### 3.4 Strategy 3: TTL Index with Delayed Archive

**Approach**: Use MongoDB TTL index to automatically delete after archive period.

**Advantages**:
- Automatic cleanup
- No application code needed
- MongoDB handles deletion efficiently

**Implementation**:
```java
// Create TTL index (runs every 60 seconds)
@PostConstruct
public void createTTLIndex() {
    mongoTemplate.getCollection("CustomerIssuedPromotion_Archive")
        .createIndex(
            new Document("archivedAt", 1),
            new IndexOptions().expireAfterSeconds(31536000) // 1 year in archive
        );
}
```

**Limitations**:
- TTL only works on date fields
- Deletion happens automatically (not reversible)
- Need to move to archive collection first

### 3.5 Strategy 4: Partitioned Collections by Year

**Approach**: Create separate collections per year (e.g., `CustomerIssuedPromotion_2021`).

**Advantages**:
- Easy to drop old collections
- Better query performance (smaller collections)
- Can archive entire collections

**Implementation**:
```java
// Route writes to year-based collection
public String getCollectionName(Date validTill) {
    int year = LocalDate.ofInstant(validTill.toInstant(), 
                                   ZoneId.systemDefault()).getYear();
    return "CustomerIssuedPromotion_" + year;
}

// Archive old collections
public void archiveOldPartitions(int yearsToKeep) {
    int currentYear = LocalDate.now().getYear();
    int archiveYear = currentYear - yearsToKeep;
    
    for (int year = 2020; year < archiveYear; year++) {
        String collectionName = "CustomerIssuedPromotion_" + year;
        // Export to archive storage (S3, etc.)
        exportCollectionToArchive(collectionName);
        // Drop collection
        mongoTemplate.getCollection(collectionName).drop();
    }
}
```

### 3.6 Strategy 5: External Archive Storage

**Approach**: Export to external storage (S3, HDFS) then delete from MongoDB.

**Advantages**:
- Cost-effective (cheaper storage)
- No MongoDB storage impact
- Can restore if needed

**Implementation**:
```java
private void archiveToExternalStorage(Long orgId, ObjectId startId, ObjectId endId, Date cutoffDate) {
    // 1. Export batch to JSON/Parquet
    Query query = new Query(/* criteria */);
    List<CustomerIssuedPromotion> batch = mongoTemplate.find(query, CustomerIssuedPromotion.class);
    
    // 2. Upload to S3/HDFS
    String archiveKey = String.format("archive/%d/%s-%s.json", 
        orgId, startId, endId);
    s3Client.putObject(bucketName, archiveKey, convertToJson(batch));
    
    // 3. Verify upload
    if (s3Client.doesObjectExist(bucketName, archiveKey)) {
        // 4. Delete from MongoDB
        mongoTemplate.remove(query, CustomerIssuedPromotion.class);
        log.info("Archived and deleted batch: {}", archiveKey);
    }
}
```

---

## 4. Recommended Archival Approach

### 4.1 Hybrid Strategy

**Recommended**: Combine Strategy 1 (Move to Archive) + Strategy 5 (External Storage)

**Phase 1: Initial Archive (Months 1-3)**
1. Move expired (>3 years) documents to archive collections in MongoDB
2. Process in batches of 1000-5000 documents
3. Run during off-peak hours (2 AM - 6 AM)
4. Process one collection at a time (Issued → Earned → Code)

**Phase 2: External Archive (Months 4-6)**
1. Export archive collections to S3/HDFS in Parquet format
2. Verify data integrity
3. Delete archive collections from MongoDB
4. Keep metadata index in MongoDB for lookup

**Phase 3: Ongoing Maintenance**
1. Monthly job to archive documents expired >3 years
2. Quarterly export to external storage
3. Monitor and adjust batch sizes based on performance

### 4.2 Implementation Checklist

- [ ] Create archive collections with same schema
- [ ] Implement batch processing service
- [ ] Add monitoring/metrics for archival progress
- [ ] Schedule jobs for off-peak hours
- [ ] Set up external storage (S3 bucket)
- [ ] Create data export utilities
- [ ] Implement verification/validation
- [ ] Update application queries to exclude archived data
- [ ] Document rollback procedures
- [ ] Test with small dataset first

### 4.3 Performance Optimizations

1. **Batch Size Tuning**:
    - Start with 1000 documents per batch
    - Monitor DB load and adjust (500-5000 range)
    - Larger batches = fewer round trips but more lock time

2. **Index Management**:
    - Ensure indexes exist on `orgId`, `validTill`, `_id`
    - Drop unused indexes before archival
    - Rebuild indexes after archival if needed

3. **Connection Pooling**:
    - Use dedicated connection pool for archival jobs
    - Limit concurrent archival jobs per shard

4. **Rate Limiting**:
    - Add delays between batches (50-200ms)
    - Use MongoDB write concern `{w: 1}` (acknowledge, not wait for replication)
    - Consider `{w: 0}` for archive inserts (fire and forget)

5. **Monitoring**:
    - Track documents archived per hour
    - Monitor DB CPU, memory, I/O
    - Alert on errors or performance degradation
    - Track estimated completion time

### 4.4 Sample Implementation Structure

```
ArchiveService
├── PromotionArchiveService (main orchestrator)
├── IssuedPromotionArchiveService
├── EarnedPromotionArchiveService
├── CodePromotionArchiveService
└── ArchiveStorageService (S3/HDFS integration)

ArchiveJob (Scheduled)
├── ArchiveExpiredPromotionsJob
└── ExportToExternalStorageJob
```

---

## 5. Query Modifications for Archived Data

### 5.1 Application Query Updates

After implementing archival, update all queries to exclude archived data:

```java
// Before
findAllByOrgIdAndCustomerIdAndValidTillGreaterThanAndActiveIsTrue(orgId, customerId, date);

// After (if using soft delete)
findAllByOrgIdAndCustomerIdAndValidTillGreaterThanAndActiveIsTrueAndArchivedIsFalse(
    orgId, customerId, date);

// Or update query builder
BooleanExpression conditions = QCustomerIssuedPromotion.customerIssuedPromotion
    .orgId.eq(orgId)
    .and(QCustomerIssuedPromotion.customerIssuedPromotion.active.isTrue())
    .and(QCustomerIssuedPromotion.customerIssuedPromotion.archived.isFalse());
```

### 5.2 Index Updates

Add compound indexes including `archived` field:
```java
@CompoundIndex(def = "{'orgId' : 1, 'customerId': 1, 'validTill': 1, 'archived': 1}")
```

---

## 6. Rollback Strategy

### 6.1 If Using Archive Collection

```java
// Restore from archive
public void restoreFromArchive(Long orgId, Date restoreDate) {
    Query archiveQuery = new Query(
        Criteria.where("orgId").is(orgId)
            .and("archivedAt").gte(restoreDate)
    );
    
    List<CustomerIssuedPromotion> toRestore = 
        mongoTemplate.find(archiveQuery, CustomerIssuedPromotion.class, 
                          "CustomerIssuedPromotion_Archive");
    
    mongoTemplate.insertAll(toRestore, "CustomerIssuedPromotion");
    log.info("Restored {} documents from archive", toRestore.size());
}
```

### 6.2 If Using External Storage

```java
// Restore from S3
public void restoreFromS3(String archiveKey) {
    // 1. Download from S3
    S3Object s3Object = s3Client.getObject(bucketName, archiveKey);
    List<CustomerIssuedPromotion> documents = parseJson(s3Object.getObjectContent());
    
    // 2. Insert into MongoDB
    mongoTemplate.insertAll(documents, "CustomerIssuedPromotion");
}
```

---

## 7. Monitoring and Metrics

### 7.1 Key Metrics to Track

- Documents archived per hour/day
- Archive job duration
- Database load (CPU, memory, I/O)
- Error rates
- Storage space freed
- Query performance impact

### 7.2 Alerts

- Archive job failures
- Database load > threshold
- Archive rate < expected
- Storage space issues

---

## 8. Estimated Timeline

For 1.2B issued + 1.2B earned + codes:

**Assumptions**:
- Batch size: 2000 documents
- Processing rate: 10,000 documents/minute (with delays)
- Daily processing window: 4 hours (off-peak)

**Calculation**:
- Total documents: ~2.4B (issued + earned, excluding codes)
- Documents per day: 10,000 * 60 * 4 = 2.4M
- Days to complete: 2.4B / 2.4M = **~1000 days** (2.7 years)

**Optimization**:
- Increase batch size to 5000: **~400 days**
- Process 24/7: **~167 days**
- Parallel processing across shards: **~50-80 days** (depending on shard count)

**Recommendation**: Start with parallel processing during off-peak hours, targeting completion in 3-6 months.

---

## 9. References

- `AbstractExpiryDateChangeDao` - Batch processing pattern
- `ExpiryDateChangeJobService` - Job orchestration
- `PromotionMeta.hasExpired()` - Expiry check logic
- MongoDB TTL Indexes documentation
- Spring Data MongoDB batch operations

