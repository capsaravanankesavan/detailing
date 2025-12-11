# Expiry Filtering Cache Optimization

## Problem Statement

The `filterExpiredCoupons` method in `CouponRuntimeServiceImpl` was causing significant cache pressure and performance issues:

1. **Heavy Cache Usage**: For each coupon entity, the method was loading the full `CouponSeriesConfig` object, which is expensive and includes many fields not needed for expiry filtering.

2. **Unnecessary Data Loading**: The `prepareCouponsFromEntity` method was calling `m_couponSeriesConfigManager.getCouponConfig()` for every coupon, even when multiple coupons belonged to the same series ID.

3. **Premature Code-Level Redemption Loading**: `setCodeLevelRedemptions()` was being called for ALL coupons before filtering, when it should only be called for filtered coupons that pass the expiry check.

4. **Expensive Joins**: The `CouponSeriesEntityService.get()` method was calling `stitchJoinField()`, which loads data from multiple tables (OwnerInfo, AudienceGroups, ConfigKeyValues, RedemptionConfigs, etc.) when only 3 fields are needed for expiry calculation.

## Solution Overview

We implemented a **lightweight cache** specifically for expiry filtering using an **interface-based design** that:

1. Creates an `ExpiryInfo` interface to abstract expiry-related methods
2. Caches lightweight `CouponSeriesEntityForExpiry` objects (only 5 fields) instead of full `CouponSeriesEntity`
3. Uses an overloaded `getExpiryDate(ExpiryInfo)` method in `CouponImpl` - no field storage needed
4. Skips expensive database joins by loading only base entity fields + `OwnerInfoEntity` (needed for `getEffectiveExpiryDate()` fallback)
5. Moves `setCodeLevelRedemptions()` to after filtering, so it's only called for filtered coupons
6. Loads full `CouponSeriesEntity` from main cache only when needed for `setCodeLevelRedemptions()`
7. Uses the same cache pattern as `CouponSeriesConfigManagerImpl` for consistency

## Implementation Details

### 1. Interface-Based Design

#### 1.1 ExpiryInfo Interface
**Location**: `src/main/java/com/capillary/luci/api/ExpiryInfo.java`

- Interface that abstracts expiry-related methods
- Implemented by both `CouponSeriesEntity` and `CouponSeriesEntityForExpiry`
- Methods:
    - `getEffectiveExpiryDate()` - Base expiry date (from validTillDate or ownerInfoEntity fallback)
    - `getExpiryStrategyType()` - Strategy type (DAYS, MONTHS, MONTHS_END, SERIES_EXPIRY)
    - `getExpiryStrategyValue()` - Strategy value (e.g., number of days/months)

**Benefits**:
- Clean separation of concerns
- No inheritance overhead
- Works with both full and lightweight entities

#### 1.2 CouponSeriesEntityForExpiry
**Location**: `src/main/java/com/capillary/luci/impl/config/CouponSeriesEntityForExpiry.java`

- Lightweight data class that implements `ExpiryInfo`
- **Does NOT extend `CouponSeriesEntity`** - standalone class with only 5 fields:
    - `id` (int)
    - `orgId` (int)
    - `effectiveExpiryDate` (Date) - Pre-computed from validTillDate or ownerInfoEntity
    - `expiryStrategyType` (ExpiryStrategyType)
    - `expiryStrategyValue` (int)

- Factory method `from(CouponSeriesEntity)` extracts only needed fields
- Significantly reduces memory footprint compared to full `CouponSeriesEntity`

#### 1.3 CouponSeriesEntityForExpiryLoader
**Location**: `src/main/java/com/capillary/luci/impl/config/CouponSeriesEntityForExpiryLoader.java`

- Extends `CacheLoader<CouponSeriesKey, CouponSeriesEntityForExpiry>`
- Loads base `CouponSeriesEntity` with `OwnerInfoEntity` (needed for `getEffectiveExpiryDate()` fallback)
- Converts to lightweight `CouponSeriesEntityForExpiry` using factory method
- Uses lightweight methods from `CouponSeriesEntityService`:
    - `getForExpiryFiltering()` - Single entity load
    - `getAllForExpiryFiltering()` - Batch load

#### 1.4 CouponSeriesEntityForExpiryCacheManager
**Location**: `src/main/java/com/capillary/luci/impl/config/CouponSeriesEntityForExpiryCacheManager.java`

- Manages the lightweight cache using Guava's `LoadingCache`
- Same cache configuration as `CouponSeriesConfigManagerImpl`:
    - `expireAfterAccess` - Based on `appConfig.getConfigCacheExpiryAfterAccessTime()`
    - `maximumSize` - Based on `appConfig.getConfigCacheMaximumSize()`
    - `refreshAfterWrite` - Based on `appConfig.getConfigCacheExpiryTime()`
    - `recordStats()` - For monitoring cache performance

**Key Methods**:
- `getCouponSeriesEntityForExpiry(int orgId, int couponSeriesId)` - Get single `CouponSeriesEntityForExpiry`
- `getCouponSeriesEntitiesForExpiry(List<CouponSeriesKey> keys)` - Batch get
- `invalidateCouponSeriesEntityForExpiry(CouponSeriesKey)` - Invalidate cache entry

### 2. Lightweight Service Methods

#### 2.1 CouponSeriesEntityService Enhancements
**Location**: `src/main/java/com/capillary/luci/impl/config/CouponSeriesEntityService.java`

Added two new lightweight methods that load minimal data:

**`getForExpiryFiltering(CouponSeriesKey key)`**:
- Loads base `CouponSeriesEntity` from database
- **Explicitly loads `OwnerInfoEntity`** - Required because `getEffectiveExpiryDate()` falls back to `ownerInfoEntity.getExpiryDate()` when `validTillDate` is null
- **Skips** other expensive joins from `stitchJoinField()`:
    - ❌ `CouponSeriesAudienceGroupEntityList` - Not needed for expiry
    - ❌ `CouponConfigKeyValuesEntity` (keyValueList) - Not needed for expiry (loaded separately for `setCodeLevelRedemptions()`)
    - ❌ `CouponConfigKeyValuesEntity` (redemptionConfigKeyValueList) - Not needed for expiry (loaded separately for `setCodeLevelRedemptions()`)
    - ❌ `CouponSeriesRedemptionOrgEntityList` - Not needed for expiry
    - ❌ `CouponSeriesCustomPropertyKeyValueEntities` - Not needed for expiry
- Returns entity with only base table fields + `OwnerInfoEntity` needed for expiry:
    - `getEffectiveExpiryDate()` - Uses validTillDate or ownerInfoEntity.getExpiryDate() as fallback
    - `getExpiryStrategyType()`
    - `getExpiryStrategyValue()`

**`getAllForExpiryFiltering(Iterable<CouponSeriesKey> keys)`**:
- Batch version that loads multiple entities
- Loads `OwnerInfoEntity` for each series (needed for `getEffectiveExpiryDate()` fallback)
- **Skips** `batchStitchJoinFields()` call for other joined fields
- Groups by orgId for efficient batch loading
- Returns map of entities with only base fields + `OwnerInfoEntity`

**Why OwnerInfoEntity is Loaded**:
- `getEffectiveExpiryDate()` in `CouponSeriesEntity` has this logic:
  ```java
  if (this.validTillDate != null)
      validTillDate = this.validTillDate;
  else {
      validTillDate = ownerInfoEntity == null ? null : ownerInfoEntity.getExpiryDate();
  }
  ```
- To ensure correct expiry calculation, we need `OwnerInfoEntity` as a fallback when `validTillDate` is null

**Note**: While `setCodeLevelRedemptions()` needs `keyToCouponConfigKeyValueEntityMap` and `redemptionConfigEntityIdToRedemptionConfigMap`, these are loaded when the full `CouponSeriesEntity` is accessed from the main cache after filtering. The lightweight cache is only used for the initial expiry filtering step.

### 3. Modified filterExpiredCoupons Method

**Location**: `src/main/java/com/capillary/luci/impl/CouponRuntimeServiceImpl.java`

#### Before:
```java
private List<Coupon> filterExpiredCoupons(int orgId, Date minExpiryDate, List<CouponEntity> couponEntities) {
    List<Coupon> coupons = prepareCouponsFromEntity(orgId, couponEntities);  // Loads full CouponSeriesConfig
    couponExpiryService.updateBulkExpiryDates(coupons);
    List<Coupon> filteredCoupons = coupons.stream()
            .filter((coupon) -> coupon.getExpiryDate().after(minExpiryDate))
            .collect(Collectors.toList());
    // setCodeLevelRedemptions() called for ALL coupons before filtering
    // ... sorting logic
}
```

#### After:
```java
private List<Coupon> filterExpiredCoupons(int orgId, Date minExpiryDate, List<CouponEntity> couponEntities) {
    // Use lightweight cache to create coupons for filtering
    List<Coupon> coupons = prepareCouponsFromEntityForExpiryFilter(orgId, couponEntities);
    couponExpiryService.updateBulkExpiryDates(coupons);
    List<Coupon> filteredCoupons = coupons.stream()
            .filter((coupon) -> coupon.getExpiryDate().after(minExpiryDate))
            .collect(Collectors.toList());

    // Sort filtered coupons before setting code level redemptions
    if (filteredCoupons != null && filteredCoupons.size() > 1) {
        filteredCoupons = filteredCoupons.stream()
                .sorted(Comparator.comparing(Coupon::getExpiryDate))
                .collect(Collectors.toList());
    }

    // Set code level redemptions ONLY for filtered and sorted coupons (not all coupons)
    // Note: setCodeLevelRedemptions needs full CouponSeriesEntity (with config key values),
    // so we load it from the main cache only for filtered coupons
    if (filteredCoupons != null && !filteredCoupons.isEmpty()) {
        for (Coupon coupon : filteredCoupons) {
            try {
                CouponImpl couponImpl = (CouponImpl) coupon;
                // Load full CouponSeriesEntity from main cache (needed for setCodeLevelRedemptions)
                CouponSeriesEntity couponSeriesEntity = m_couponSeriesConfigManager.getCouponConfig(
                        couponImpl.getOrgId(), couponImpl.getCouponEntity().getCouponSeriesId())
                        .getCouponSeriesEntity();
                if (couponSeriesEntity != null) {
                    m_couponService.setCodeLevelRedemptions(couponSeriesEntity, 
                            couponImpl.getCouponEntity(), couponImpl);
                }
            } catch (Exception e) {
                logger.debug("could not set code level redemptions for coupon id {}", 
                        coupon.getId(), e);
            }
        }
    }
    return filteredCoupons;
}
```

**Key Changes**:
1. Uses `prepareCouponsFromEntityForExpiryFilter()` instead of `prepareCouponsFromEntity()`
2. Sorting happens **before** `setCodeLevelRedemptions()` - processes expensive operations only on final set
3. `setCodeLevelRedemptions()` called **only for filtered coupons** (not all coupons)
4. Full `CouponSeriesEntity` loaded from main cache **only when needed** for `setCodeLevelRedemptions()`

### 4. New prepareCouponsFromEntityForExpiryFilter Method

**Location**: `src/main/java/com/capillary/luci/impl/CouponRuntimeServiceImpl.java`

```java
private List<Coupon> prepareCouponsFromEntityForExpiryFilter(int orgId, List<CouponEntity> couponEntities) {
    List<Coupon> coupons = new ArrayList<>();
    if (couponEntities != null && !couponEntities.isEmpty()) {
        for (CouponEntity currentEntity : couponEntities) {
            try {
                // Use lightweight cache - returns CouponSeriesEntityForExpiry (implements ExpiryInfo)
                CouponSeriesEntityForExpiry expiryInfo =
                        m_couponSeriesEntityForExpiryCacheManager.getCouponSeriesEntityForExpiry(
                                orgId, currentEntity.getCouponSeriesId());
                // Create coupon with couponSeriesEntity = null to avoid storing full entity
                CouponImpl coupon = new CouponImpl(currentEntity, (CouponSeriesEntity) null);
                // Calculate and cache expiry date using lightweight ExpiryInfo
                coupon.getExpiryDate(expiryInfo);
                coupons.add(coupon);
            } catch (Exception e) {
                logger.debug("couponseries entity could not be loaded for expiry filtering, coupon series id {}",
                        currentEntity.getCouponSeriesId(), e);
            }
        }
    }
    return coupons;
}
```

**Key Differences from `prepareCouponsFromEntity()`**:
- ✅ Uses `CouponSeriesEntityForExpiryCacheManager` instead of `CouponSeriesConfigManager`
- ✅ Uses lightweight `CouponSeriesEntityForExpiry` (implements `ExpiryInfo`) instead of full `CouponSeriesConfig`
- ✅ Creates `CouponImpl` with `couponSeriesEntity = null` - avoids storing full entity
- ✅ Calls `getExpiryDate(expiryInfo)` to calculate and cache expiry date using lightweight `ExpiryInfo`
- ✅ **Does NOT** call `setCodeLevelRedemptions()` here (moved to after filtering)

### 5. CouponImpl Enhancements

**Location**: `src/main/java/com/capillary/luci/impl/CouponImpl.java`

#### 5.1 Overloaded getExpiryDate Method

Added a public overloaded method that accepts `ExpiryInfo`:

```java
/**
 * Calculate expiry date from ExpiryInfo interface.
 * This method works with both CouponSeriesEntity and CouponSeriesEntityForExpiry.
 * Used for expiry filtering to avoid loading full CouponSeriesEntity.
 * The calculated expiry date is cached in the expiryDate field.
 * 
 * @param expiryInfo the expiry information (can be CouponSeriesEntity or CouponSeriesEntityForExpiry)
 * @return calculated expiry date (also caches it in expiryDate field)
 */
public Date getExpiryDate(ExpiryInfo expiryInfo) {
    if (expiryDate != null) {
        return expiryDate;  // Return cached value if already calculated
    }
    
    if (expiryInfo == null) {
        //Default to already expired
        expiryDate = new DateTime().minusDays(1).toDate();
        return expiryDate;
    }

    // Calculate expiry date using ExpiryInfo interface methods
    Date validTillDate = DateUtil.getDateWithEndOfDayTime(expiryInfo.getEffectiveExpiryDate());
    Date issuedDate = getIssuedDate();
    if(issuedDate == null){
        expiryDate = validTillDate;
        return expiryDate;
    }
    ExpiryStrategyType expiryStrategyType = expiryInfo.getExpiryStrategyType();

    Calendar cal = Calendar.getInstance();
    cal.setTime(issuedDate);
    switch (expiryStrategyType) {
        case DAYS:
            cal.add(Calendar.DATE, expiryInfo.getExpiryStrategyValue());
            break;
        case MONTHS:
            cal.add(Calendar.MONTH, expiryInfo.getExpiryStrategyValue());
            break;
        case MONTHS_END:
            cal.add(Calendar.MONTH, expiryInfo.getExpiryStrategyValue());
            int maxDayOfMonth = cal.getActualMaximum(Calendar.DATE);
            int currentDayOfMonth = cal.get(Calendar.DAY_OF_MONTH);
            cal.add(Calendar.DATE, maxDayOfMonth - currentDayOfMonth);
            break;
        case SERIES_EXPIRY:
            cal.setTime(validTillDate);
            break;
    }
    Date cpnExpiryDate = cal.getTime();
    cpnExpiryDate = cpnExpiryDate.after(validTillDate) ? validTillDate : cpnExpiryDate;
    cpnExpiryDate = DateUtil.getDateWithEndOfDayTime(cpnExpiryDate);
    expiryDate = cpnExpiryDate;  // Cache the calculated value
    return expiryDate;
}
```

**Key Points**:
- **No field storage**: `ExpiryInfo` is passed as parameter, not stored in `CouponImpl`
- **Caching**: Calculated expiry date is cached in `expiryDate` field
- **Interface-based**: Works with both `CouponSeriesEntity` and `CouponSeriesEntityForExpiry`
- **Memory efficient**: No need to store full `CouponSeriesEntity` in `CouponImpl` for expiry filtering

#### 5.2 CouponSeriesEntity Implements ExpiryInfo

**Location**: `src/main/java/com/capillary/luci/data/entity/CouponSeriesEntity.java`

- `CouponSeriesEntity` now implements `ExpiryInfo` interface
- This allows it to be used with the overloaded `getExpiryDate(ExpiryInfo)` method
- Maintains backward compatibility - existing code using `CouponSeriesEntity` continues to work

### 6. Cache Invalidation Integration

**Location**: `src/main/java/com/capillary/luci/impl/config/CouponSeriesConfigManagerImpl.java`

The lightweight cache is automatically invalidated when the main config cache is invalidated:

```java
@Autowired(required = false)
private CouponSeriesEntityForExpiryCacheManager m_couponSeriesEntityForExpiryCacheManager;

public void invalidateCouponSeriesConfig(CouponSeriesKey couponSeriesKey) {
    this.m_couponSeriesConfigMap.invalidate(couponSeriesKey);
    // Also invalidate the lightweight cache for expiry filtering
    if (m_couponSeriesEntityForExpiryCacheManager != null) {
        m_couponSeriesEntityForExpiryCacheManager.invalidateCouponSeriesEntityForExpiry(couponSeriesKey);
    }
}
```

Invalidation happens in:
- `reloadConfig(int orgId, int couponSeriesId)`
- `saveCouponSeriesConfig()` - When config is saved/updated
- `invalidateCouponSeriesConfig(CouponSeriesKey)`

## Fields Needed for Expiry Calculation

From `CouponImpl.getExpiryDate(ExpiryInfo)` analysis, only these fields are required:

1. **`getEffectiveExpiryDate()`** - Base expiry date from series (from `validTillDate` or `ownerInfoEntity.getExpiryDate()` as fallback)
2. **`getExpiryStrategyType()`** - Strategy (DAYS, MONTHS, MONTHS_END, SERIES_EXPIRY)
3. **`getExpiryStrategyValue()`** - Value for the strategy (e.g., number of days/months)

**Why `OwnerInfoEntity` is Needed**:
- `getEffectiveExpiryDate()` in `CouponSeriesEntity` has fallback logic:
  ```java
  if (this.validTillDate != null)
      validTillDate = this.validTillDate;
  else {
      validTillDate = ownerInfoEntity == null ? null : ownerInfoEntity.getExpiryDate();
  }
  ```
- When `validTillDate` is null, the method falls back to `ownerInfoEntity.getExpiryDate()`
- Therefore, `OwnerInfoEntity` must be loaded to ensure correct expiry calculation

**Fields NOT Needed for Expiry Filtering**:
- ❌ `CouponSeriesAudienceGroupEntityList` - Not used in expiry calculation
- ❌ `CouponConfigKeyValuesEntity` (keyValueList) - Not needed for expiry (but needed for `setCodeLevelRedemptions()` - loaded separately)
- ❌ `CouponConfigKeyValuesEntity` (redemptionConfigKeyValueList) - Not needed for expiry (but needed for `setCodeLevelRedemptions()` - loaded separately)
- ❌ `CouponSeriesRedemptionOrgEntityList` - Not needed for expiry
- ❌ `CouponSeriesCustomPropertyKeyValueEntities` - Not needed for expiry

**Note on `setCodeLevelRedemptions()`**: This method is called AFTER filtering and uses:
- `keyToCouponConfigKeyValueEntityMap` (via `isCodeLevelLimitEnabled()`)
- `redemptionConfigEntityIdToRedemptionConfigMap` (via `isEntityLevelRedemptionConfigEnabled()` and `isCodeLevelLimitEnabledForOuId()`)

These maps are populated when the full `CouponSeriesEntity` is loaded from the main cache (`m_couponSeriesConfigManager.getCouponConfig()`) after filtering. The lightweight cache is only used for the initial expiry filtering step.

## Performance Benefits

### 1. Reduced Database Queries
**Before**: For each coupon entity:
- 1 query to load `CouponSeriesConfig` (which internally loads full entity with joins)
- `stitchJoinField()` makes 6+ additional queries:
    - OwnerInfo query
    - AudienceGroups query
    - ConfigKeyValues query (twice - regular and redemption)
    - RedemptionOrgEntities query
    - CustomPropertyKeyValues query

**After**: For each coupon entity:
- 1 query to load base `CouponSeriesEntity` (no joins)
- Cache hit on subsequent requests for same series ID

### 2. Reduced Memory Usage
- **Before**: Full `CouponSeriesConfig` object with all joined collections stored in `CouponImpl`
- **After**:
    - Lightweight `CouponSeriesEntityForExpiry` objects in cache (only 5 fields)
    - No `ExpiryInfo` field stored in `CouponImpl` - passed as parameter only when needed
    - `CouponImpl` stores `couponSeriesEntity = null` during filtering, avoiding full entity storage
    - Full `CouponSeriesEntity` loaded only when needed for `setCodeLevelRedemptions()` after filtering

### 3. Optimized Code-Level Redemption Loading
- **Before**: `setCodeLevelRedemptions()` called for ALL coupons before filtering
- **After**: `setCodeLevelRedemptions()` called ONLY for filtered coupons that pass expiry check

### 4. Better Cache Efficiency
- Lightweight cache entries (`CouponSeriesEntityForExpiry`) take significantly less memory (only 5 fields)
- More entries can fit in the same cache size
- Faster cache loading due to fewer database queries (only base entity + OwnerInfoEntity)
- O(1) cache lookup performance with Guava cache

### 5. Interface-Based Design Benefits
- Clean separation of concerns with `ExpiryInfo` interface
- No inheritance overhead - `CouponSeriesEntityForExpiry` does not extend `CouponSeriesEntity`
- Polymorphic behavior - both full and lightweight entities work with same `getExpiryDate(ExpiryInfo)` method
- No object conversion needed - interface abstraction handles both types

## Code Changes Summary

### New Files Created
1. **`ExpiryInfo.java`** - Interface defining common expiry-related methods
    - Location: `src/main/java/com/capillary/luci/api/ExpiryInfo.java`
    - Implemented by `CouponSeriesEntity` and `CouponSeriesEntityForExpiry`

2. **`CouponSeriesEntityForExpiry.java`** - Lightweight data class for expiry filtering
    - Location: `src/main/java/com/capillary/luci/impl/config/CouponSeriesEntityForExpiry.java`
    - Implements `ExpiryInfo` interface
    - Contains only 5 fields needed for expiry calculation

3. **`CouponSeriesEntityForExpiryLoader.java`** - Cache loader for lightweight entities
    - Location: `src/main/java/com/capillary/luci/impl/config/CouponSeriesEntityForExpiryLoader.java`
    - Extends `CacheLoader<CouponSeriesKey, CouponSeriesEntityForExpiry>`

4. **`CouponSeriesEntityForExpiryCacheManager.java`** - Cache manager
    - Location: `src/main/java/com/capillary/luci/impl/config/CouponSeriesEntityForExpiryCacheManager.java`
    - Manages Guava `LoadingCache` for lightweight entities

### Modified Files
1. **`CouponImpl.java`**:
    - Added public overloaded method `getExpiryDate(ExpiryInfo expiryInfo)`
    - No `expiryInfo` field stored - passed as parameter only when needed
    - Calculated expiry date is cached in `expiryDate` field

2. **`CouponRuntimeServiceImpl.java`**:
    - Modified `filterExpiredCoupons()` to use lightweight cache
    - Added `prepareCouponsFromEntityForExpiryFilter()` method
    - Moved `setCodeLevelRedemptions()` to after filtering and sorting
    - Loads full `CouponSeriesEntity` from main cache only for filtered coupons when needed

3. **`CouponSeriesEntityService.java`**:
    - Added `getForExpiryFiltering()` method - loads base entity + OwnerInfoEntity
    - Added `getAllForExpiryFiltering()` method - batch version

4. **`CouponSeriesEntity.java`**:
    - Implements `ExpiryInfo` interface

5. **`CouponSeriesConfigManagerImpl.java`**:
    - Added cache invalidation for lightweight cache
    - Integrated with existing invalidation methods

## Testing Considerations

1. **Cache Hit/Miss Rates**: Monitor cache statistics to ensure good hit rates
    - Use `CouponSeriesEntityForExpiryCacheManager.getCacheStats()` to monitor performance

2. **Performance Metrics**: Measure improvement in `filterExpiredCoupons` execution time
    - Compare before/after execution times
    - Monitor database query reduction

3. **Memory Usage**: Monitor memory footprint reduction
    - Compare memory usage of `CouponSeriesEntityForExpiry` vs full `CouponSeriesConfig`
    - Monitor cache size and memory footprint

4. **Correctness**: Verify expiry filtering still works correctly with lightweight entities
    - Test with coupons that have `validTillDate` set
    - Test with coupons that rely on `ownerInfoEntity.getExpiryDate()` fallback
    - Test all expiry strategy types (DAYS, MONTHS, MONTHS_END, SERIES_EXPIRY)

5. **Code-Level Redemptions**: Ensure `setCodeLevelRedemptions()` works correctly after filtering
    - Verify full `CouponSeriesEntity` is loaded correctly from main cache
    - Test with coupons that have code-level redemption limits enabled
    - Test with entity-level redemption configs

6. **Interface Compatibility**: Verify `ExpiryInfo` interface works correctly
    - Test that both `CouponSeriesEntity` and `CouponSeriesEntityForExpiry` work with `getExpiryDate(ExpiryInfo)`
    - Verify no casting or conversion issues

## Architecture Benefits

### Interface-Based Design
- **Polymorphism**: Both `CouponSeriesEntity` and `CouponSeriesEntityForExpiry` implement `ExpiryInfo`, allowing them to be used interchangeably
- **No Inheritance Overhead**: `CouponSeriesEntityForExpiry` does not extend `CouponSeriesEntity`, avoiding unnecessary inheritance chain
- **Clean Separation**: Expiry-related logic is abstracted through interface, making code more maintainable

### Memory Optimization
- **No Field Storage**: `ExpiryInfo` is not stored in `CouponImpl` - passed as parameter only when needed
- **Lightweight Cache**: Cache stores only 5 fields instead of full entity with all joined collections
- **Lazy Loading**: Full `CouponSeriesEntity` loaded only when needed for `setCodeLevelRedemptions()` after filtering

### Performance Optimization
- **O(1) Cache Lookup**: Guava cache provides constant-time access
- **Reduced Database Queries**: Only base entity + OwnerInfoEntity loaded, skipping 5+ other joins
- **Optimized Order**: Sorting happens before `setCodeLevelRedemptions()`, so expensive operations only run on filtered set

## Future Optimizations

1. **Batch Loading**: Further optimize by batching coupon series entity loads in `prepareCouponsFromEntityForExpiryFilter()`
2. **Selective Field Loading**: Consider loading only the 5 required fields directly from database if possible
3. **Cache Warming**: Pre-warm cache for frequently accessed series
4. **Metrics**: Add detailed metrics for cache performance monitoring
5. **Parallel Processing**: Consider parallel processing of expiry filtering for large coupon lists

## Related Documentation

- `getCouponDetails_Performance_CouponSeriesConfig_Fields.md` - Original performance analysis
- `getCouponSeriesEntity_Fields_Accessed_Line1356.md` - Fields accessed analysis
- `getCouponDetails_CouponSeriesConfig_Usage.md` - Config usage patterns
- `pb.txt` - Original problem statement
