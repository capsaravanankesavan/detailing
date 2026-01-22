# Masan Voucher Series Data Summary

## Table of Contents

- [Executive Summary](#executive-summary)
    - [Overall Statistics (Excluding Automation Orgs)](#overall-statistics-excluding-automation-orgs)
    - [Masan Organizations Overview](#masan-organizations-overview)
    - [Market Position & Rankings](#market-position--rankings)
    - [Key Findings](#key-findings)
- [Organizations](#organizations)
- [Data Source](#data-source)
- [Voucher Series Distribution by Organization](#voucher-series-distribution-by-organization)
    - [Masan (2219)](#masan-2219)
    - [Masan Demo (2207)](#masan-demo-2207)
    - [Masan Win+ Demo (2301)](#masan-win-demo-2301)
    - [Organizations with No Data](#organizations-with-no-data)
- [Key Insights](#key-insights)
    - [Masan-Specific](#masan-specific)
    - [Series Type Distribution](#series-type-distribution)
    - [Comparison Context](#comparison-context)
- [Breakdown by Series Type](#breakdown-by-series-type)

---

## Executive Summary

### Overall Statistics (Excluding Automation Orgs)
- **Total Voucher Series**: 614,728
- **Automation Orgs Excluded**: 998, 2200, 1003, 1138, 2312, 1732, 1969

### Masan Organizations Overview
- **Total Masan Series**: 479,708
- **Market Share**: 78.0% (479,708 / 614,728)

### Market Position & Rankings
| Organization | Org ID | Series Count | Overall Rank | % of Masan Total |
|--------------|--------|--------------|--------------|-----------------|
| **Masan** | 2219 | 361,959 | **#1** (Top organization) | 75.5% |
| **Masan Demo** | 2207 | 117,710 | **#2** (Second largest) | 24.5% |
| Masan Win+ Demo | 2301 | 39 | #205 | <0.01% |
| Masan GT | 2346 | 0 | N/A | 0% |
| Masan GT(Demo) | 2345 | 0 | N/A | 0% |
| Masan (Win+) | 2300 | 0 | N/A | 0% |

### Key Findings
1. **Dominant Market Position**: Masan organizations hold the **top 2 positions** among non-automation organizations
2. **Significant Concentration**: Masan accounts for **78% of all voucher series** (excluding automation orgs)
3. **Primary Driver**: Masan (2219) alone represents **58.9% of all series** (361,959 / 614,728)
4. **Scale Comparison**: The third-largest organization (org 1492) has 65,500 series - **1.8x smaller** than Masan Demo and **5.5x smaller** than Masan
5. **Active Organizations**: Only 3 of 6 Masan organizations have active series data

---

## Organizations

| Organization Name | Org ID |
|------------------|--------|
| Masan | 2219 |
| Masan Demo | 2207 |
| Masan GT | 2346 |
| Masan GT(Demo) | 2345 |
| Masan (Win+) | 2300 |
| Masan Win+ Demo | 2301 |

## Data Source

**Query:**
```sql
select org_id, series_type, discount_type, count(*) 
from source_delta.luci__voucher_series 
where org_id in (2219, 2207, 2346, 2345, 2300, 2301) 
group by org_id, series_type, discount_type
```

## Voucher Series Distribution by Organization

### Masan (2219)
| Series Type | Discount Type | Count |
|------------|---------------|-------|
| CAMPAIGN | ABS | 316 |
| JOURNEY | ABS | 15 |
| UNDEFINED | PERC | 361,499 |
| CAMPAIGN | PERC | 30 |
| UNDEFINED | ABS | 94 |
| JOURNEY | PERC | 2 |
| LOYALTY | PERC | 1 |
| GOODWILL | ABS | 1 |
| DVS | PERC | 1 |
| **Total** | | **361,959** |

### Masan Demo (2207)
| Series Type | Discount Type | Count |
|------------|---------------|-------|
| GOODWILL | ABS | 2 |
| LOYALTY | PERC | 118 |
| UNDEFINED | PERC | 117,281 |
| REFERRAL | PERC | 21 |
| CAMPAIGN | ABS | 19 |
| CAMPAIGN | PERC | 19 |
| JOURNEY | PERC | 3 |
| UNDEFINED | ABS | 148 |
| LOYALTY | ABS | 50 |
| JOURNEY | ABS | 48 |
| DVS | PERC | 1 |
| **Total** | | **117,710** |

### Masan Win+ Demo (2301)
| Series Type | Discount Type | Count |
|------------|---------------|-------|
| UNDEFINED | PERC | 12 |
| UNDEFINED | ABS | 3 |
| LOYALTY | PERC | 23 |
| LOYALTY | ABS | 1 |
| **Total** | | **39** |

### Organizations with No Data
- **Masan GT (2346)** - No records found
- **Masan GT(Demo) (2345)** - No records found
- **Masan (Win+) (2300)** - No records found

## Key Insights

### Masan-Specific
1. **Total Masan Records**: 479,708 voucher series across all Masan organizations
2. **Largest Masan Organization**: Masan (2219) with 361,959 records (75.5% of Masan total)
3. **Masan Market Dominance**:
    - Masan (2219) is the #1 organization (excluding automation orgs)
    - Masan Demo (2207) is the #2 organization (excluding automation orgs)
    - Combined, they represent 78% of all voucher series

### Series Type Distribution
4. **Dominant Series Type**: UNDEFINED accounts for 478,791 records (99.8% of Masan total)
5. **Discount Type Distribution**:
    - **PERC (Percentage)**: 478,857 records (99.8%)
    - **ABS (Absolute)**: 851 records (0.2%)
6. **Series Types Present**:
    - UNDEFINED (most common)
    - LOYALTY
    - CAMPAIGN
    - JOURNEY
    - GOODWILL
    - REFERRAL (only in 2207)
    - DVS

### Comparison Context
7. **Scale Perspective**:
    - Masan (2219) has **5.5x more** series than the 3rd largest organization
    - Masan Demo (2207) has **1.8x more** series than the 3rd largest organization

## Breakdown by Series Type

| Series Type | Total Count | Percentage |
|------------|-------------|------------|
| UNDEFINED | 478,791 | 99.8% |
| LOYALTY | 193 | 0.04% |
| CAMPAIGN | 384 | 0.08% |
| JOURNEY | 68 | 0.01% |
| GOODWILL | 3 | <0.01% |
| REFERRAL | 21 | <0.01% |
| DVS | 2 | <0.01% |
