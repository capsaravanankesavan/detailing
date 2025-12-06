# Promotion Expiry and Archival Strategy

## Table of Contents

- [Overview](#overview)
- [Current Data Volume](#current-data-volume)
    - [Primary Collections (High Volume)](#primary-collections-high-volume)
    - [Secondary Collections (Moderate Volume)](#secondary-collections-moderate-volume)
    - [Low Volume Collections](#low-volume-collections)
    - [Summary Statistics](#summary-statistics)
- [1. Expiry Maintenance at Different Levels](#1-expiry-maintenance-at-different-levels)
    - [1.1 Meta Level (PromotionMeta)](#11-meta-level-promotionmeta)
    - [1.2 Issue Level (CustomerIssuedPromotion)](#12-issue-level-customerissuedpromotion)
    - [1.3 Earn Level (CustomerEarnedPromotion)](#13-earn-level-customerearnedpromotion)
    - [1.4 Code Level (CodeBasedPromotion)](#14-code-level-codebasedpromotion)
    - [1.5 Expiry Relationships Summary](#15-expiry-relationships-summary)
    - [1.6 Collections Referencing PromotionMeta](#16-collections-referencing-promotionmeta)
        - [Primary Collections (Already Documented)](#primary-collections-already-documented)
        - [Secondary Collections (Require Archival Consideration)](#secondary-collections-require-archival-consideration)
        - [Archival Priority Summary](#archival-priority-summary)
        - [Archival Query Patterns](#archival-query-patterns)
- [2. Querying Expired Promotions](#2-querying-expired-promotions)
    - [2.1 Query Patterns](#21-query-patterns)
    - [2.2 Efficient Query Strategies](#22-efficient-query-strategies)
- [3. Archival Strategies Without DB Load](#3-archival-strategies-without-db-load)
    - [3.1 Batch Processing with Cursor-Based Pagination](#31-batch-processing-with-cursor-based-pagination)
    - [3.2 Strategy 1: Move to Archive Collection](#32-strategy-1-move-to-archive-collection)
    - [3.3 Strategy 2: Soft Delete with Archive Flag](#33-strategy-2-soft-delete-with-archive-flag)
    - [3.4 Strategy 3: TTL Index with Delayed Archive](#34-strategy-3-ttl-index-with-delayed-archive)
    - [3.5 Strategy 4: Partitioned Collections by Year](#35-strategy-4-partitioned-collections-by-year)
    - [3.6 Strategy 5: External Archive Storage](#36-strategy-5-external-archive-storage)
- [4. Recommended Archival Approach](#4-recommended-archival-approach)
    - [4.1 Hybrid Strategy](#41-hybrid-strategy)
    - [4.2 Implementation Checklist](#42-implementation-checklist)
    - [4.3 Performance Optimizations](#43-performance-optimizations)
        - [4.3.1 Lock Contention and MongoDB Locking Behavior](#431-lock-contention-and-mongodb-locking-behavior)
    - [4.4 Sample Implementation Structure](#44-sample-implementation-structure)
- [5. Query Modifications for Archived Data](#5-query-modifications-for-archived-data)
    - [5.1 Application Query Updates](#51-application-query-updates)
    - [5.2 Index Updates](#52-index-updates)
- [6. Rollback Strategy](#6-rollback-strategy)
    - [6.1 If Using Archive Collection](#61-if-using-archive-collection)
    - [6.2 If Using External Storage](#62-if-using-external-storage)
- [7. Monitoring and Metrics](#7-monitoring-and-metrics)
    - [7.1 Key Metrics to Track](#71-key-metrics-to-track)
    - [7.2 Alerts](#72-alerts)
- [8. Estimated Timeline](#8-estimated-timeline)
- [9. Validation Logic and Archival Impact](#9-validation-logic-and-archival-impact)
    - [9.1 Promotion Name Uniqueness Validation](#91-promotion-name-uniqueness-validation)
    - [9.2 Other Validation Considerations](#92-other-validation-considerations)
- [10. References](#10-references)

---

## Overview
This document details how expiry is maintained at different levels (Meta, Issue, Earn, Code) and provides strategies for archiving expired promotions (>3 years) without impacting database performance.

## Current Data Volume

This section provides the current data volume statistics from the asiacrm MongoDB cluster (emf) as of the last analysis. These numbers help prioritize archival efforts and estimate storage requirements.

### Primary Collections (High Volume)

| Collection | Documents | Storage Size | Avg Doc Size | Index Size | Total Size (Data + Index) |
|------------|-----------|-------------|--------------|------------|---------------------------|
| **cartEvaluation** | 630M | 1.95 TB | 12.70 kB | 57.68 GB | **~2.01 TB** |
| **customerEarnedPromotion** | 1.4B | 71.87 GB | 451 B | 161.66 GB | **~233.53 GB** |
| **customerIssuedPromotion** | 1.3B | 55.39 GB | 370 B | 151.47 GB | **~206.86 GB** |
| **promotionRedemption** | 138M | 36.28 GB | 1.11 kB | 16.80 GB | **~53.08 GB** |
| **restrictionLevelRedemptionSummary** | 89M | 4.15 GB | 266 B | 3.48 GB | **~7.63 GB** |

### Secondary Collections (Moderate Volume)

| Collection | Documents | Storage Size | Avg Doc Size | Index Size | Total Size (Data + Index) |
|------------|-----------|-------------|--------------|------------|---------------------------|
| **codeBasedPromotion** | 11M | 665.13 MB | 368 B | 1.09 GB | **~1.76 GB** |
| **customerPromotionPreference** | 16M | 984.78 MB | 323 B | 1.19 GB | **~2.17 GB** |
| **promotionMeta** | 993K | 167.02 MB | 969 B | 78.57 MB | **~245.59 MB** |
| **promotionSummary** | 764K | 42.52 MB | 203 B | 37.27 MB | **~79.79 MB** |
| **loyaltyTargetEvent** | 432K | 45.36 MB | 230 B | 48.50 MB | **~93.86 MB** |
| **promotionRedemptionFailureLog** | 176K | 55.03 MB | 1.36 kB | 13.64 MB | **~68.67 MB** |
| **codeBasedPromotionMeta** | 65K | 8.99 MB | 458 B | 3.25 MB | **~12.24 MB** |
| **cartEvaluationReservation** | 62 | 1.13 MB | 510 B | 1.24 MB | **~2.37 MB** |
| **jv_snapshots** | 4.1M | 739.45 MB | 1.25 kB | 336.24 MB | **~1.08 GB** |

### Low Volume Collections

| Collection | Documents | Storage Size | Notes |
|------------|-----------|--------------|-------|
| **promotionExpiryReminder** | 185 | 126.98 kB | Configuration data |
| **expiryReminderJob** | 3K | 184.32 kB | Job execution records |
| **expiryReminderBatchJob** | 293 | 81.92 kB | Batch job records |
| **expiryDateChangeJob** | 8.6K | 876.54 kB | Expiry date update jobs |
| **codeBasedPromotionErrorLog** | 91 | 45.06 kB | Error logs |

### Summary Statistics

**Total Collections Analyzed**: 26

**Top 5 Collections by Storage Size**:
1. **cartEvaluation** - 1.95 TB (largest, critical for archival)
2. **customerEarnedPromotion** - 71.87 GB
3. **customerIssuedPromotion** - 55.39 GB
4. **promotionRedemption** - 36.28 GB
5. **restrictionLevelRedemptionSummary** - 4.15 GB

**Top 5 Collections by Document Count**:
1. **customerEarnedPromotion** - 1.4B documents
2. **customerIssuedPromotion** - 1.3B documents
3. **cartEvaluation** - 630M documents
4. **promotionRedemption** - 138M documents
5. **restrictionLevelRedemptionSummary** - 89M documents

**Total Estimated Storage** (Primary + Secondary Collections):
- **Data**: ~2.16 TB
- **Indexes**: ~400 GB
- **Total**: **~2.56 TB**

**Goal**: Archive promotions (issued, earned, code) and related collections expired > 3 years.

**Archival Priority** (based on volume):
1. **cartEvaluation** - 630M documents, 1.95 TB (highest priority due to size)
2. **customerEarnedPromotion** - 1.4B documents, 71.87 GB
3. **customerIssuedPromotion** - 1.3B documents, 55.39 GB
4. **promotionRedemption** - 138M documents, 36.28 GB
5. **restrictionLevelRedemptionSummary** - 89M documents, 4.15 GB

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

## 1.6 Collections Referencing PromotionMeta

This section lists all collections that reference `PromotionMeta` (via `promotionId` or `promotionMetaId`) and their archival considerations.

### Primary Collections (Already Documented)

1. **PromotionMeta** - Campaign definition (Section 1.1)
2. **CustomerIssuedPromotion** - Issued promotions (Section 1.2)
3. **CustomerEarnedPromotion** - Earned promotions (Section 1.3)
4. **CodeBasedPromotion** - Promo codes (Section 1.4)

### Secondary Collections (Require Archival Consideration)

#### 5. PromotionRedemption
- **Collection**: `PromotionRedemption`
- **Reference Field**: `promotionId` (String)
- **Purpose**: Records of promotion redemptions/usage
- **Indexes**: `{'orgId': 1, 'promotionId': 1, 'customerId': 1, 'redemptionDate': -1}`
- **Archival Strategy**:
    - Archive based on `redemptionDate < archiveCutoffDate` (e.g., >3 years old)
    - Can also filter by `promotionId` if archiving specific expired promotions
    - **Volume**: High (one per redemption transaction)
    - **Retention**: Consider regulatory/compliance requirements for transaction history

#### 6. RestrictionLevelRedemptionSummary
- **Collection**: `RestrictionLevelRedemptionSummary`
- **Reference Field**: `promotionId` (String)
- **Purpose**: KPI summaries for promotion restrictions (cart/customer/earn level)
- **Indexes**: `{'orgId': 1, 'promotionId': 1, 'level': 1, 'customerId': 1}`
- **Archival Strategy**:
    - Archive based on `validTill < archiveCutoffDate` (expired restriction periods)
    - Can also filter by `promotionId` for expired promotions
    - **Volume**: Medium (summary data, less frequent)
    - **Retention**: May need for historical KPI analysis

#### 7. CustomerPromotionPreference
- **Collection**: `CustomerPromotionPreference`
- **Reference Field**: `promotionId` (String)
- **Purpose**: Customer preferences for promotions (e.g., opt-in/opt-out)
- **Indexes**: `{'orgId': 1, 'customerId': 1, 'promotionId': 1, 'earnId': 1}` (unique)
- **Archival Strategy**:
    - Archive when associated promotion is archived (via `promotionId`)
    - Can use `lastUpdatedOn < archiveCutoffDate` as secondary filter
    - **Volume**: Low to medium (one per customer-promotion preference)
    - **Retention**: May be needed for preference history

#### 8. CartEvaluation
- **Collection**: `CartEvaluation`
- **Reference Field**: `promotionId` (embedded in `appliedPromotions` map via `CartEvaluationPromotionDetails`)
- **Purpose**: Cart evaluation results with applied promotions
- **Indexes**: `{'orgId': 1, 'customerId': 1}`, `{'attribution.lastUpdatedOn': 1}`
- **Archival Strategy**:
    - Archive based on `attribution.lastUpdatedOn < archiveCutoffDate` or `validTillTime < archiveCutoffDate`
    - Filter by `promotionId` in nested `appliedPromotions` map if needed
    - **Volume**: Very High (591M documents, ~1.79TB) - **HIGHEST PRIORITY**
    - **Retention**: Consider shorter retention (e.g., 1-2 years) due to size
    - **Note**: Already mentioned in Current Data Volume section

#### 9. PromotionExpiryReminder
- **Collection**: `PromotionExpiryReminder`
- **Reference Field**: `promotionId` (String)
- **Purpose**: Configuration for sending expiry reminders for promotions
- **Indexes**: None specific to promotionId
- **Archival Strategy**:
    - Archive when associated promotion is archived (via `promotionId`)
    - Can also filter by `active = false` and old `attribution.lastUpdatedOn`
    - **Volume**: Low (one per promotion with reminder configured)
    - **Retention**: Can archive immediately when promotion expires

#### 10. CodeBasedPromotionMeta
- **Collection**: `CodeBasedPromotionMeta`
- **Reference Field**: `promotionId` (String), `metaId` (String - references CodeBasedPromotionMeta itself)
- **Purpose**: Metadata for code-based promotion creation (bulk code generation jobs)
- **Indexes**: `{'orgId': 1, 'promotionId': 1}`
- **Archival Strategy**:
    - Archive when associated promotion is archived (via `promotionId`)
    - Can filter by `attribution.lastUpdatedOn < archiveCutoffDate`
    - **Volume**: Low (one per code generation job)
    - **Retention**: Can archive after code generation is complete and promotion expires

#### 11. PromotionSummary
- **Collection**: `PromotionSummary`
- **Reference Field**: `promotionId` (String) - unique per promotion
- **Purpose**: Aggregated statistics for promotions (issued count, redeemed count, etc.)
- **Indexes**: `{'orgId': 1, 'promotionId': 1}` (unique)
- **Archival Strategy**:
    - Archive when associated promotion is archived (via `promotionId`)
    - Can use `lastRedeemed` or `lastEarned < archiveCutoffDate` as filters
    - **Volume**: Low (one per promotion)
    - **Retention**: May be needed for historical analytics - consider longer retention

#### 12. CodeBasedPromotionErrorLog
- **Collection**: `CodeBasedPromotionErrorLog`
- **Reference Field**: `promotionId` (String), `metaId` (String)
- **Purpose**: Error logs for code generation failures
- **Indexes**: `{'orgId': 1, 'metaId': 1}`, `{'orgId': 1, 'promoCode': 1}`
- **Archival Strategy**:
    - Archive when associated promotion is archived (via `promotionId`)
    - Can use creation date if available
    - **Volume**: Low to medium (errors during code generation)
    - **Retention**: Can archive after code generation is complete

#### 13. PromotionRedemptionFailureLog
- **Collection**: `PromotionRedemptionFailureLog`
- **Reference Field**: `promotionId` (embedded in `promotionRedemptions` list)
- **Purpose**: Logs of failed redemption attempts
- **Indexes**: `{'requestId': 1}`, `{'transactionIdentifier': 1}`
- **Archival Strategy**:
    - Archive based on `eventTime < archiveCutoffDate`
    - Filter by `promotionId` in nested `promotionRedemptions` if needed
    - **Volume**: Low to medium (only failed redemptions)
    - **Retention**: Can archive after investigation period (e.g., 1 year)

#### 14. ExpiryReminderJob
- **Collection**: `ExpiryReminderJob`
- **Reference Field**: `promotionId` (String)
- **Purpose**: Job execution records for expiry reminder processing
- **Indexes**: None specific
- **Archival Strategy**:
    - Archive based on `createdTime < archiveCutoffDate`
    - Can filter by `promotionId` for expired promotions
    - **Volume**: Low (job execution records)
    - **Retention**: Can archive after job completion (short retention)

#### 15. CartEvaluationReservation
- **Collection**: `CartEvaluationReservation`
- **Reference Field**: None directly, but related to `CartEvaluation` (which has promotionId)
- **Purpose**: Reservation/locking mechanism for cart evaluations
- **Indexes**: `{'orgId': 1, 'customerId': 1, 'sessionId': 1}`, TTL index on `validTillDateTime`
- **Archival Strategy**:
    - Uses TTL index - automatically expires based on `validTillDateTime`
    - No direct promotionId reference, but can archive based on `validTillDateTime < archiveCutoffDate`
    - **Volume**: Medium (one per cart evaluation session)
    - **Retention**: Short-term (TTL handles cleanup)

#### 16. LoyaltyTargetEvent
- **Collection**: `LoyaltyTargetEvent`
- **Reference Field**: None directly, but `targetEventId` may reference earned promotions
- **Purpose**: Tracks loyalty target events for earned promotions
- **Indexes**: `{'orgId': 1, 'customerId': 1, 'targetEventId': 1}` (unique), TTL on `createdOn`
- **Archival Strategy**:
    - Uses TTL index - automatically expires
    - No direct promotionId, but related to earned promotions
    - **Volume**: Low to medium
    - **Retention**: Short-term (TTL handles cleanup)

#### 17. jv_snapshots (JaVers Audit Logs)
- **Collection**: `jv_snapshots`
- **Reference Field**: Entity ID stored in `globalId` field (contains ObjectId of tracked entities)
- **Purpose**: **JaVers audit log** - Stores version history/snapshots of entity changes for audit trail
- **Volume**: 4.1M documents, 739.45 MB (data) + 336.24 MB (indexes) = **~1.08 GB total**
- **Indexes**: 7 indexes (JaVers internal structure)
- **Tracked Entities** (from `AuditLogEntityType`):
    - `PromotionMeta` - Promotion campaign definitions
    - `PromotionSettings` - Promotion settings
    - `PromotionExpiryReminder` - Expiry reminder configurations
    - `PromotionOrgConfiguration` - Organization-level promotion configurations

**What JaVers Does**:
- JaVers is a Java library that automatically tracks changes to annotated entities
- Creates snapshots on every create/update/delete operation
- Stores full entity state at each change point
- Enables audit trail and version history retrieval

**Archival Strategy**:

1. **Archive with PromotionMeta** (Recommended):
    - When archiving `PromotionMeta`, also archive related JaVers snapshots
    - Query snapshots by entity ID (promotionId) stored in `globalId` field
    - **Query Pattern**:
   ```javascript
   // MongoDB query to find snapshots for a specific PromotionMeta
   db.jv_snapshots.find({
     "globalId.entity": "com.capillary.promotionengine.bo.PromotionMeta",
     "globalId.cdoId": ObjectId("<promotionId>")
   })
   ```

2. **Date-Based Archival** (Alternative):
    - Archive snapshots older than retention period (e.g., >3 years)
    - Query by commit date stored in related `jv_commit` collection
    - **Consideration**: May archive snapshots for entities that are still active

3. **Retention Considerations**:
    - **Compliance/Legal**: Audit logs may have legal retention requirements (e.g., 7 years)
    - **Business Requirements**: May need audit trail for financial/compliance audits
    - **Recommendation**: Archive snapshots **only after** confirming retention requirements
    - **Alternative**: Keep audit logs longer than business data (e.g., archive promotions after 3 years, but keep audit logs for 7 years)

4. **JaVers Collection Structure**:
    - `jv_snapshots` - Entity snapshots (what we archive)
    - `jv_commit` - Commit metadata (dates, authors) - linked via `commit_fk`
    - `jv_global_id` - Entity references - linked via `globalId_fk`
    - `jv_commit_property` - Commit properties (e.g., orgId) - linked via `commit_fk`
    - `jv_head_id` - Head commit tracking (small, 1 document)

5. **Archival Implementation**:
   ```java
   // Archive JaVers snapshots for a specific PromotionMeta
   public void archiveJaversSnapshotsForPromotion(String promotionId) {
       // 1. Find all snapshots for this PromotionMeta
       Query snapshotQuery = new Query(
           Criteria.where("globalId.entity")
               .is("com.capillary.promotionengine.bo.PromotionMeta")
               .and("globalId.cdoId").is(new ObjectId(promotionId))
       );
       
       List<Document> snapshots = mongoTemplate.find(
           snapshotQuery, Document.class, "jv_snapshots");
       
       // 2. Get related commits
       Set<ObjectId> commitIds = snapshots.stream()
           .map(s -> (ObjectId) s.get("commit_fk"))
           .collect(Collectors.toSet());
       
       // 3. Archive snapshots, commits, and related data
       // Move to archive collection or external storage
   }
   ```

6. **Important Notes**:
    - **Do NOT archive if compliance requires longer retention** - Check legal/compliance requirements first
    - **Archive as a unit** - Archive all JaVers collections together (snapshots, commits, global_ids, commit_properties)
    - **Test restoration** - Ensure archived audit logs can be restored if needed for audits
    - **Consider separate retention** - Audit logs may need different retention than business data

**Archival Query Patterns**:

```java
// Query snapshots by PromotionMeta ID
Criteria.where("globalId.entity")
    .is("com.capillary.promotionengine.bo.PromotionMeta")
    .and("globalId.cdoId").in(promotionIds)

// Query by commit date (for date-based archival)
// Note: Requires join with jv_commit collection
Criteria.where("commit_fk").in(
    // Subquery: commits older than cutoff date
    commitIdsFromDateQuery
)
```

**Recommendation**:
- **Option 1 (Recommended)**: Archive JaVers snapshots **only when archiving PromotionMeta**, and only after confirming compliance requirements allow it
- **Option 2**: Keep audit logs longer (e.g., 7 years) while archiving business data (3 years)
- **Option 3**: Archive audit logs separately based on commit date, independent of promotion archival

### Archival Priority Summary

**High Priority** (Large Volume):
1. **CartEvaluation** - 591M documents (~1.79TB) - **CRITICAL**
2. **CustomerIssuedPromotion** - 1.2B documents (~50GB)
3. **CustomerEarnedPromotion** - 1.2B documents (~65GB)
4. **PromotionRedemption** - High volume (transactional data)

**Medium Priority** (Moderate Volume):
5. **CodeBasedPromotion** - Moderate volume
6. **RestrictionLevelRedemptionSummary** - Summary data
7. **CartEvaluationReservation** - Session data (TTL managed)

**Low Priority** (Small Volume, Can Archive with Promotion):
8. **PromotionMeta** - One per campaign
9. **PromotionSummary** - One per promotion
10. **CustomerPromotionPreference** - One per customer-promotion
11. **PromotionExpiryReminder** - One per promotion with reminder
12. **CodeBasedPromotionMeta** - One per code generation job
13. **CodeBasedPromotionErrorLog** - Error logs
14. **PromotionRedemptionFailureLog** - Failure logs
15. **ExpiryReminderJob** - Job records
16. **LoyaltyTargetEvent** - Event tracking (TTL managed)

**Special Consideration** (Compliance-Dependent):
17. **jv_snapshots** - 4.1M documents, ~1.08 GB - **Audit logs** - Archive only after confirming compliance/legal retention requirements. May need longer retention (e.g., 7 years) than business data (3 years).

### Archival Query Patterns

For collections with direct `promotionId` reference:
```java
// Archive by promotionId (when promotion is expired)
Criteria.where("orgId").is(orgId)
    .and("promotionId").in(expiredPromotionIds)
```

For collections with date-based archival:
```java
// Archive by date (e.g., redemptionDate, lastUpdatedOn)
Criteria.where("orgId").is(orgId)
    .and("redemptionDate").lt(archiveCutoffDate)
```

For nested promotionId (CartEvaluation):
```java
// Archive CartEvaluation with expired promotions
// Note: Requires checking nested appliedPromotions map
Criteria.where("orgId").is(orgId)
    .and("attribution.lastUpdatedOn").lt(archiveCutoffDate)
    // Additional filter needed for promotionId in nested structure
```

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

### 4.3.1 Lock Contention and MongoDB Locking Behavior

**MongoDB Locking Model**:
- **WiredTiger Storage Engine** (default in MongoDB 3.2+): Uses **document-level locks**, not collection-level locks
- **Lock Granularity**: Each document write operation acquires a lock only on the specific document being modified
- **Concurrent Operations**: Multiple operations can proceed simultaneously as long as they target different documents
- **Index Updates**: Index updates may require brief locks on index structures, but these are typically very short

**Lock Contention Scenarios During Archival**:

1. **Per-Promotion Processing (Smaller Chunks)**:
    - **Advantage**: More targeted operations, processes one promotion at a time
    - **Lock Behavior**:
        - Locks are held only on documents being archived for that specific promotion
        - Other promotions' documents remain unlocked and accessible
        - Minimal contention with application reads/writes on other promotions
    - **Risk**:
        - If a single promotion has millions of documents, long-running batch operations may hold locks on many documents
        - Application operations on the same promotion's documents may experience contention
    - **Mitigation**:
        - Process even within a promotion in smaller batches (1000-5000 documents)
        - Add delays between batches to release locks periodically
        - Use cursor-based pagination to process incrementally

2. **Per-Batch Processing (Across Multiple Promotions)**:
    - **Advantage**: Faster overall completion, processes multiple promotions in parallel
    - **Lock Behavior**:
        - Locks are held on documents across multiple promotions
        - Higher potential for contention if application operations target same documents
    - **Risk**:
        - More documents locked simultaneously
        - Higher chance of blocking application operations
    - **Mitigation**:
        - Limit batch size to reduce lock duration
        - Use write concern `{w: 1}` to avoid waiting for replication
        - Process during off-peak hours

**Best Practices to Minimize Lock Contention**:

1. **Batch Size Strategy**:
   ```java
   // Process in small batches with delays
   int batchSize = 1000; // Start small
   Thread.sleep(100); // 100ms delay between batches releases locks
   ```
    - **Smaller batches (500-1000)**: Shorter lock duration, less contention, more round trips
    - **Larger batches (5000+)**: Longer lock duration, higher contention risk, fewer round trips
    - **Recommendation**: Start with 1000, monitor contention, adjust based on DB load

2. **Write Concern Settings**:
   ```java
   // Use write concern to reduce lock time
   WriteConcern writeConcern = WriteConcern.ACKNOWLEDGED; // {w: 1}
   // For archive inserts (non-critical), can use {w: 0} to avoid waiting
   WriteConcern archiveWriteConcern = WriteConcern.UNACKNOWLEDGED; // {w: 0}
   ```
    - **`{w: 1}`**: Acknowledges write, doesn't wait for replication (recommended for archival)
    - **`{w: 0}`**: Fire-and-forget, minimal lock time (use only for archive collection inserts)
    - **`{w: "majority"}`**: Waits for majority replication (NOT recommended for archival - increases lock time)

3. **Processing Strategy - Per Promotion vs Per Batch**:

   **Option A: Process Per Promotion (Recommended for Large Promotions)**:
   ```java
   // Process one promotion at a time in batches
   for (String promotionId : expiredPromotionIds) {
       archivePromotionInBatches(orgId, promotionId, batchSize);
       Thread.sleep(200); // Delay between promotions
   }
   ```
    - **Pros**:
        - Isolated operations per promotion
        - Easier to track progress per promotion
        - Lower contention (only one promotion's documents locked at a time)
    - **Cons**:
        - Slower overall completion if many promotions
        - May take longer to archive all data

   **Option B: Process Per Batch (Recommended for Many Small Promotions)**:
   ```java
   // Process multiple promotions in each batch
   archiveBatchAcrossPromotions(orgId, expiredPromotionIds, batchSize);
   ```
    - **Pros**:
        - Faster overall completion
        - Better resource utilization
    - **Cons**:
        - More documents locked simultaneously
        - Higher contention potential
        - Harder to track per-promotion progress

4. **Index Considerations**:
    - **Ensure indexes exist** on query fields (`orgId`, `promotionId`, `validTill`, `_id`)
    - **Index locks**: Brief locks on index structures during writes
    - **Impact**: Minimal if indexes are well-designed
    - **Avoid**: Dropping/rebuilding indexes during archival (causes collection-level locks)

5. **Concurrent Processing Limits**:
   ```java
   // Limit concurrent archival operations
   int maxConcurrentArchivals = 2; // Process max 2 promotions concurrently
   Semaphore semaphore = new Semaphore(maxConcurrentArchivals);
   
   for (String promotionId : expiredPromotionIds) {
       semaphore.acquire();
       executorService.submit(() -> {
           try {
               archivePromotionInBatches(orgId, promotionId, batchSize);
           } finally {
               semaphore.release();
           }
       });
   }
   ```
    - **Recommendation**: Limit to 1-2 concurrent archival operations per collection
    - **Reason**: Prevents excessive lock contention across multiple promotions

6. **Monitoring Lock Contention**:
    - **MongoDB Metrics to Monitor**:
        - `globalLock.currentQueue.total` - Queued operations waiting for locks
        - `globalLock.activeClients.total` - Active client operations
        - `wiredTiger.concurrentTransactions.write.available` - Available write tickets
        - `wiredTiger.concurrentTransactions.write.out` - Write tickets in use
    - **Application Metrics**:
        - Average batch processing time
        - Documents archived per second
        - Application operation latency during archival

**Answer to Your Question**:

**Will there be lock contention when archival runs slowly in smaller chunks per promotion?**

**Short Answer**: **Minimal contention** if implemented correctly.

**Detailed Answer**:
1. **Document-Level Locks**: MongoDB WiredTiger uses document-level locks, so only documents being archived are locked, not the entire collection
2. **Smaller Chunks**: Processing in smaller chunks (1000 documents) per promotion means:
    - **Shorter lock duration** per batch
    - **Locks released frequently** (between batches)
    - **Minimal overlap** with application operations
3. **Per-Promotion Processing**: Processing one promotion at a time further reduces contention:
    - Only one promotion's documents are locked at a time
    - Application operations on other promotions proceed normally
    - Even if application operations target the same promotion, locks are brief
4. **Potential Contention Scenarios**:
    - **Same Document**: If application tries to update a document being archived (rare, as expired promotions shouldn't be updated)
    - **Index Updates**: Brief locks on index structures (minimal impact)
    - **Large Batches**: If batch size is too large (5000+), locks held longer
5. **Mitigation**:
    - Use batch size 1000-2000 documents
    - Add 100-200ms delays between batches
    - Process during off-peak hours
    - Use write concern `{w: 1}` (not `{w: "majority"}`)
    - Monitor lock metrics and adjust batch size if needed

**Recommendation**:
- **Process per promotion in batches of 1000-2000 documents**
- **Add 100-200ms delays between batches**
- **Limit to 1-2 concurrent archival operations per collection**
- **Monitor lock metrics and adjust batch size based on observed contention**

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

## 9. Validation Logic and Archival Impact

### 9.1 Promotion Name Uniqueness Validation

**Question**: Will archiving `PromotionMeta` break validation logic like unique promotion names?

**Answer**: **NO** - Archiving expired promotions will NOT break promotion name uniqueness validation.

#### How Promotion Name Validation Works

The validation logic in `PromotionMetaManagementService.validatePromotionName()` checks for duplicate promotion names using the following criteria:

```java
// Method: getActiveAndUnExpiredPromotionsWithPromotionNameAndNotId()
// Query conditions:
- orgId = <orgId>
- name = <promotionName>
- active = true
- startDate <= now
- endDate >= now  // KEY: Only checks UNEXPIRED promotions
- id != <promotionId>  // For updates, excludes current promotion
```

**Key Points**:
1. **Only Active Promotions**: Validation checks `active = true`
2. **Only Unexpired Promotions**: Validation checks `endDate >= now` (promotion must not be expired)
3. **Organization-Scoped**: Validation is per organization (`orgId`)
4. **Configurable**: Controlled by `PromotionOrgConfiguration.uniquePromotionNameConstraintEnabled` (default: `true`)

#### Why Archiving Won't Break Validation

1. **Expired Promotions Are Already Excluded**:
    - The validation query requires `endDate >= now`
    - Expired promotions (archived after >3 years) have `endDate < now`
    - Therefore, they are **already excluded** from the validation check

2. **Archive Collection Separation**:
    - If using archive collection strategy, archived promotions are in a separate collection
    - The validation query runs against the main `PromotionMeta` collection
    - Archived promotions won't be found by the validation query

3. **Soft Delete Strategy**:
    - If using soft delete with `archived = true` flag, add filter: `.and("archived").ne(true)` to validation query
    - This ensures archived promotions are excluded from validation

#### Validation Scenarios

**Scenario 1: Creating New Promotion**
- Checks for active, unexpired promotions with same name
- Archived/expired promotions are not considered
- **Result**: ✅ Safe to archive expired promotions

**Scenario 2: Updating Promotion Name**
- Same validation as creation
- Excludes current promotion being updated
- **Result**: ✅ Safe to archive expired promotions

**Scenario 3: Extending Expiry Date**
- When extending `endDate`, validation re-checks for duplicates
- Only checks active, unexpired promotions
- **Result**: ✅ Safe to archive expired promotions

**Scenario 4: Reactivating Promotion**
- When setting `active = true`, validation checks for duplicates
- Only checks other active, unexpired promotions
- **Result**: ✅ Safe to archive expired promotions

#### Important Considerations

1. **If Extending Expiry of Archived Promotion**:
    - If you restore an archived promotion and extend its expiry date, it will be checked against current active promotions
    - This is expected behavior and maintains data integrity

2. **Database Index**:
    - The index `{'orgId': 1, 'name': 1}` is **NOT unique** (see `PromotionMeta` class)
    - Uniqueness is enforced at application level, not database level
    - This allows multiple promotions with same name if they are expired/inactive

3. **Configuration Flag**:
    - `uniquePromotionNameConstraintEnabled` can be disabled per organization
    - If disabled, no validation is performed
    - Archival has no impact when validation is disabled

#### Recommended Approach

**For Archive Collection Strategy**:
- No changes needed - archived promotions are in separate collection
- Validation queries only run against main collection

**For Soft Delete Strategy**:
- Update validation query to exclude archived promotions:
  ```java
  // Add to getActiveAndUnExpiredPromotionsWithPromotionNameAndNotId()
  criteria.add(Criteria.where("archived").ne(true));
  ```
- This ensures archived promotions don't interfere with validation

**For External Storage Strategy**:
- No changes needed - archived promotions are removed from database
- Validation queries won't find them

### 9.2 Other Validation Considerations

**Max Active Promotions Limit**:
- `PromotionOrgConfiguration.maxActivePromotions` limits active promotions per org
- Archiving expired promotions **helps** by reducing active promotion count
- No negative impact expected

**Campaign ID Validation**:
- Campaign ID validation is separate from promotion name
- Archiving has no impact on campaign-related validations

---

## 10. References

- `AbstractExpiryDateChangeDao` - Batch processing pattern
- `ExpiryDateChangeJobService` - Job orchestration
- `PromotionMeta.hasExpired()` - Expiry check logic
- `PromotionMetaManagementService.validatePromotionName()` - Promotion name uniqueness validation
- `PromotionOrgConfiguration.uniquePromotionNameConstraintEnabled` - Configuration flag
- MongoDB TTL Indexes documentation
- Spring Data MongoDB batch operations

