# XTS Order Transaction Data Dictionary

This document covers the XTS (eXtended Transaction System) order transaction data, which provides granular order-level detail for qualitative business analysis. XTS is the primary order management system for SCS and Markets operating groups.

**Key Use Cases:**
- Production country mix analysis
- Supplier/vendor concentration and performance
- Product category breakdown
- Customer channel analysis
- Business nature classification (Agency vs Principal)

**Relationship to Other Data Sources:**
- **FINANCIALS.md**: XTS feeds into financial actuals but with ~1-5% variance due to timing, adjustments, and currency
- **ORDERBOOK.md**: XTS is a source for the forward shipment snapshot (`SOURCE = 'XTS'`)

---

## Agent Notes

XTS queries require `run_sql` - there is no specialized tool for XTS data.

**Key table:** `lft.bronze.dds_ssbi_xts_order_transaction_tab`

**Common query patterns:**
- Latest data: Filter `extraction_date = (SELECT MAX(extraction_date) FROM ...)`
- Country mix: Group by `PRODUCTION_COUNTRY_CODE` or `PRODUCTION_COUNTRY_DESC`
- Supplier analysis: Group by `VENDOR_CODE`, `VENDOR_NAME`
- Agency vs Principal: Use `BUSINESS_NATURE` column ('A' = Agency, 'P' = Principal)

**Value scaling:** XTS values are in actual USD (no scaling needed)

**OG Code mapping:**
- `SCS1` = Apparel, `SCS2` = Home And Accessories
- `LFMU` = LF Markets USA, `LFEU` = LF Europe, `FWK` = Firework

---

## 1. Core Table

### 1.1. Primary Table: `lft.bronze.dds_ssbi_xts_order_transaction_tab`

- **Content:** Order line-level transactions from XTS system
- **Granularity:** One row per order line item (SO/Case/Item level)
- **Update Frequency:** Daily extraction
- **Partition Column:** `extraction_date`

---

## 2. Key Columns

### 2.1. Time Dimensions

| Column | Type | Description | Example |
|--------|------|-------------|---------|
| `ORDER_YEAR` | varchar(4) | Year order was placed | '2025' |
| `ORDER_MONTH` | varchar(2) | Month order was placed | '03' |
| `SHIP_YEAR` | varchar(4) | Year scheduled to ship | '2025' |
| `SHIP_MONTH` | varchar(2) | Month scheduled to ship | '06' |
| `SHIP_WEEK` | varchar(2) | Week scheduled to ship | '24' |
| `extraction_date` | date | Data extraction date | 2025-11-28 |

### 2.2. Order Identifiers

| Column | Type | Description |
|--------|------|-------------|
| `SO_NO` | varchar(78) | Sales Order number |
| `CASE_NO` | varchar(50) | Case number (groups items) |
| `PO_NUMBER` | varchar(3000) | Customer PO number |
| `INVOICE_NO` | varchar(50) | Invoice number |
| `ITEM_NUMBER` | varchar(100) | Item/SKU identifier |

### 2.3. Order Status

| Column | Type | Values | Description |
|--------|------|--------|-------------|
| `ORDER_STATUS` | varchar(6) | 'OPEN', 'CLOSED' | Order lifecycle status |
| `PR_STATUS_CODE` | varchar(2) | Various | Production Request status code |
| `PR_STATUS_DESC` | varchar(9) | 'Confirmed', 'Potential' | Production Request status |
| `INVOICE_STATUS_CODE` | varchar(2) | Various | Invoice status code |
| `INVOICE_STATUS_DESC` | varchar(8) | 'Posted', 'Active', 'Released' | Invoice status |

### 2.4. Entity Hierarchy

| Column | Type | Description | Example |
|--------|------|-------------|---------|
| `OPERATING_GROUP_CODE` | varchar(50) | OG code | 'SCS1', 'LFMU' |
| `OPERATING_GROUP_DESC` | varchar(50) | OG name | 'APPAREL', 'LF MARKETS USA' |
| `STREAM_CODE` | varchar(12) | Stream code | 'DEN', 'DSS' |
| `STREAM_DESC` | varchar(160) | Stream name | 'DENIM LIFESTYLE' |
| `PRODUCT_GROUP_CODE` | varchar(40) | Product Group code | 'AEO' |
| `PRODUCT_GROUP_DESC` | varchar(160) | Product Group name | 'AEO' |
| `DIVISION_CODE` | varchar(40) | Division code | 'AE1', 'AE2' |
| `DIVISION_DESC` | varchar(160) | Division name | 'AE1 DIVISION' |

**OG Code Mapping:**

| Code | Description | Segment |
|------|-------------|---------|
| `SCS1` | Apparel | SCS |
| `SCS2` | Home And Accessories | SCS |
| `LFMU` | LF Markets USA | Markets |
| `LFEU` | LF Europe | Markets |
| `FWK` | Firework | Markets |
| `LFAD` | LF Asia Direct | Markets |

### 2.5. Customer Hierarchy

| Column | Type | Description | Example |
|--------|------|-------------|---------|
| `CORPORATE_CUSTOMER_CODE` | varchar(15) | Corporate customer code | 'AEOI' |
| `CORPORATE_CUSTOMER_NAME` | varchar(150) | Corporate customer name | 'AMERICAN EAGLE OUTFITTERS INC.' |
| `CUSTOMER_CODE` | varchar(15) | Legal entity customer code | 'AEOI' |
| `CUSTOMER_NAME` | varchar(150) | Legal entity customer name | 'AMERICAN EAGLE OUTFITTERS INC.' |
| `ULTIMATE_CUSTOMER` | varchar(80) | Final destination/channel | 'STORE', 'WEB', 'CANADA' |
| `ULTIMATE_CORP_CUST_CODE` | varchar(150) | Ultimate corporate customer code | |
| `ULTIMATE_CORP_CUST_FULL_NAME` | varchar(150) | Ultimate corporate customer name | |

**Customer Hierarchy Logic:**
- `CORPORATE_CUSTOMER_NAME` = Parent company (use for customer-level analysis)
- `CUSTOMER_NAME` = Legal entity (may have multiple per corporate customer)
- `ULTIMATE_CUSTOMER` = Final channel/destination (STORE, WEB, regional markets)

### 2.6. Supplier/Vendor Hierarchy

| Column | Type | Description | Example |
|--------|------|-------------|---------|
| `SUPPLIER_CODE` | varchar(18) | Supplier code | |
| `SUPPLIER_NAME` | varchar(240) | Supplier name | 'SESHIN APPAREL CO., LTD.' |
| `SUPPLIER_COUNTRY_CODE` | varchar(18) | Supplier registered country code | 'KR' |
| `SUPPLIER_COUNTRY_NAME` | varchar(240) | Supplier registered country | 'REPUBLIC OF KOREA' |
| `PARENT_SUPPLIER_CODE` | varchar(18) | Parent supplier code | |
| `PARENT_SUPPLIER_NAME` | varchar(240) | Parent supplier name | |

**Note:** `SUPPLIER_COUNTRY_NAME` is where the supplier is registered/headquartered, which may differ from `PRODUCTION_COUNTRY_NAME` where goods are actually made.

### 2.7. Factory & Production Geography

| Column | Type | Description | Example |
|--------|------|-------------|---------|
| `FACTORY_CODE` | varchar(18) | Factory identifier | |
| `FACTORY_NAME` | varchar(242) | Factory name | |
| `FACTORY_RATING` | varchar(603) | Factory compliance rating | |
| `PRODUCTION_COUNTRY_CODE` | varchar(9) | Production country code | 'BD' |
| `PRODUCTION_COUNTRY_NAME` | varchar(120) | Production country | 'BANGLADESH' |
| `PRODUCTION_HUB_CODE` | varchar(4) | Production hub code | |
| `PRODUCTION_HUB_NAME` | varchar(50) | Production hub name | |
| `SOURCING_COUNTRY_CODE` | varchar(4) | Sourcing country code | |
| `SOURCING_COUNTRY_NAME` | varchar(30) | Sourcing country | |
| `SOURCING_OFFICE_CODE` | varchar(9) | Sourcing office code | |
| `SOURCING_OFFICE_NAME` | varchar(90) | Sourcing office name | |

### 2.8. Product Hierarchy

| Column | Type | Description | Example |
|--------|------|-------------|---------|
| `PRODUCT_FAMILY_CODE` | varchar(5) | Product family code | |
| `PRODUCT_FAMILY_DESC` | varchar(100) | Product family | 'Apparel', 'Accessories', 'Hardgoods' |
| `PRODUCT_CAT_GROUP_CODE` | varchar(5) | Category group code | |
| `PRODUCT_CAT_GROUP_DESC` | varchar(100) | Category group | 'Bottoms', 'Tops', 'Jacket / Coat' |
| `PRODUCT_CATEGORY_CODE` | varchar(12) | Category code | |
| `PRODUCT_CATEGORY_DESC` | varchar(65) | Category | |
| `PRODUCT_FABR_CODE` | varchar(5) | Fabric code | |
| `PRODUCT_FABR_DESC` | varchar(100) | Fabric type | |
| `PRODUCT_GENDER_CODE` | varchar(5) | Gender code | |
| `PRODUCT_GENDER_DESC` | varchar(50) | Gender | 'Men', 'Women', 'Unisex' |

### 2.9. Value Metrics

| Column | Type | Description | Scale |
|--------|------|-------------|-------|
| `COGS_USD` | decimal(38,10) | **Primary metric** - Cost of goods sold | Actual USD |
| `FOB_USD` | decimal(38,10) | FOB value (often equals COGS) | Actual USD |
| `COM_USD` | decimal(38,10) | Commission amount | Actual USD |
| `CST_USD` | decimal(38,10) | Cost (same as COGS) | Actual USD |
| `ACTUAL_FOB_USD` | decimal(38,10) | Actual FOB value | Actual USD |
| `QTY` | decimal(38,10) | Quantity shipped/ordered | Units |
| `SCHEDULE_QTY` | decimal(38,10) | Scheduled quantity | Units |

**Note:** Values are in actual USD (NOT thousands like FINANCIALS.md). Use `COGS_USD` as the primary value metric for analysis.

### 2.10. Business Nature & Intercompany

| Column | Type | Values | Description |
|--------|------|--------|-------------|
| `BUSINESS_NATURE` | varchar(1) | 'A', 'P', NULL | A=Agency, P=Principal |
| `DWCPMS_INTER_CO` | varchar(3) | 'Y', 'N' | Intercompany flag |
| `DWCPMS_INTCO_COD` | varchar(15) | Various | Intercompany entity code |

**Business Nature Logic:**

| BUSINESS_NATURE | DWCPMS_INTER_CO | Effective Nature (Group Level) |
|-----------------|-----------------|--------------------------------|
| A (Agency) | N | **Agency** - Direct to external customer |
| A (Agency) | Y | **Imputed Principal** - Interco takes ownership |
| P (Principal) | N or Y | **Principal** - LF takes ownership |

**SQL for Effective Business Nature:**
```sql
CASE 
    WHEN BUSINESS_NATURE = 'A' AND DWCPMS_INTER_CO = 'Y' THEN 'Imputed Principal'
    WHEN BUSINESS_NATURE = 'A' THEN 'Agency'
    WHEN BUSINESS_NATURE = 'P' THEN 'Principal'
    ELSE 'Unknown'
END AS Effective_Business_Nature
```

**Intercompany Prevalence by OG (2025):**
- **Apparel (SCS1):** ~0% intercompany - almost all direct agency
- **H&A (SCS2):** ~0% intercompany - almost all direct agency
- **LF Europe:** ~45% intercompany agency (Imputed Principal)
- **LF Markets USA:** ~35% intercompany agency (Imputed Principal)
- **Promocean:** ~30% intercompany agency (Imputed Principal)

### 2.11. Commission & Pricing

| Column | Type | Description |
|--------|------|-------------|
| `COMMISSION_RATE` | decimal(38,10) | Commission rate (decimal) |
| `PLI_RATE` | decimal(38,10) | Product Liability Insurance rate |
| `PLI_AMOUNT` | decimal(38,10) | PLI amount |
| `ITEM_PRICE` | decimal(38,10) | Item selling price |
| `ITEM_COST` | decimal(38,10) | Item cost |

### 2.12. Key Dates

| Column | Type | Description |
|--------|------|-------------|
| `SO_CREATION_DATE` | timestamp | Sales Order creation date |
| `PO_CREATION_DATE` | timestamp | Purchase Order creation date |
| `PR_CREATION_DATE` | timestamp | Production Request creation date |
| `SCHEDULE_DATE` | timestamp | Original scheduled ship date |
| `RESCHEDULE_DATE` | timestamp | Rescheduled ship date |
| `SHIPMENT_DATE` | timestamp | Actual shipment date |
| `INVOICE_CREATION_DATE` | timestamp | Invoice creation date |
| `EX_FACTORY_DATE` | timestamp | Ex-factory date |

---

## 3. Standard Query Patterns

### 3.1. Basic Aggregation

```sql
-- Total COGS by Stream for a ship year
SELECT 
    STREAM_DESC,
    ROUND(CAST(SUM(COGS_USD) AS DECIMAL(38,10)) / 1000000, 2) AS COGS_M
FROM lft.bronze.dds_ssbi_xts_order_transaction_tab
WHERE SHIP_YEAR = '2025'
  AND OPERATING_GROUP_CODE = 'SCS1'
GROUP BY STREAM_DESC
ORDER BY COGS_M DESC;
```

### 3.2. Production Country Mix

```sql
SELECT 
    PRODUCTION_COUNTRY_NAME,
    ROUND(CAST(SUM(COGS_USD) AS DECIMAL(38,10)) / 1000000, 1) AS COGS_M,
    ROUND(SUM(COGS_USD) / SUM(SUM(COGS_USD)) OVER () * 100, 1) AS Pct
FROM lft.bronze.dds_ssbi_xts_order_transaction_tab
WHERE SHIP_YEAR = '2025'
  AND STREAM_CODE = 'DEN'
GROUP BY PRODUCTION_COUNTRY_NAME
ORDER BY COGS_M DESC;
```

### 3.3. Top Suppliers/Vendors

```sql
SELECT 
    SUPPLIER_NAME,
    SUPPLIER_COUNTRY_NAME,
    ROUND(CAST(SUM(COGS_USD) AS DECIMAL(38,10)) / 1000000, 1) AS COGS_M,
    COUNT(DISTINCT CASE_NO) AS Order_Count
FROM lft.bronze.dds_ssbi_xts_order_transaction_tab
WHERE SHIP_YEAR = '2025'
  AND STREAM_CODE = 'DEN'
GROUP BY SUPPLIER_NAME, SUPPLIER_COUNTRY_NAME
ORDER BY COGS_M DESC
LIMIT 15;
```

### 3.4. Product Category Breakdown

```sql
SELECT 
    PRODUCT_FAMILY_DESC,
    PRODUCT_CAT_GROUP_DESC,
    ROUND(CAST(SUM(COGS_USD) AS DECIMAL(38,10)) / 1000000, 1) AS COGS_M,
    ROUND(SUM(COGS_USD) / SUM(SUM(COGS_USD)) OVER () * 100, 1) AS Pct
FROM lft.bronze.dds_ssbi_xts_order_transaction_tab
WHERE SHIP_YEAR = '2025'
  AND STREAM_CODE = 'DEN'
GROUP BY PRODUCT_FAMILY_DESC, PRODUCT_CAT_GROUP_DESC
ORDER BY COGS_M DESC;
```

### 3.5. Customer Channel Analysis

```sql
SELECT 
    CORPORATE_CUSTOMER_NAME,
    CUSTOMER_NAME,
    ULTIMATE_CUSTOMER,
    ROUND(CAST(SUM(COGS_USD) AS DECIMAL(38,10)) / 1000000, 1) AS COGS_M
FROM lft.bronze.dds_ssbi_xts_order_transaction_tab
WHERE SHIP_YEAR = '2025'
  AND STREAM_CODE = 'DEN'
GROUP BY CORPORATE_CUSTOMER_NAME, CUSTOMER_NAME, ULTIMATE_CUSTOMER
ORDER BY COGS_M DESC
LIMIT 15;
```

### 3.6. Business Nature with Intercompany Logic

```sql
SELECT 
    OPERATING_GROUP_DESC,
    BUSINESS_NATURE,
    DWCPMS_INTER_CO AS Interco_Flag,
    CASE 
        WHEN BUSINESS_NATURE = 'A' AND DWCPMS_INTER_CO = 'Y' THEN 'Imputed Principal'
        WHEN BUSINESS_NATURE = 'A' THEN 'Agency'
        WHEN BUSINESS_NATURE = 'P' THEN 'Principal'
        ELSE 'Unknown'
    END AS Effective_Nature,
    ROUND(CAST(SUM(COGS_USD) AS DECIMAL(38,10)) / 1000000, 1) AS COGS_M
FROM lft.bronze.dds_ssbi_xts_order_transaction_tab
WHERE SHIP_YEAR = '2025'
GROUP BY OPERATING_GROUP_DESC, BUSINESS_NATURE, DWCPMS_INTER_CO
ORDER BY COGS_M DESC;
```

### 3.7. Monthly Trend

```sql
SELECT 
    SHIP_MONTH,
    ROUND(CAST(SUM(COGS_USD) AS DECIMAL(38,10)) / 1000000, 1) AS COGS_M
FROM lft.bronze.dds_ssbi_xts_order_transaction_tab
WHERE SHIP_YEAR = '2025'
  AND STREAM_CODE = 'DEN'
GROUP BY SHIP_MONTH
ORDER BY SHIP_MONTH;
```

---

## 4. Cross-Reference to Other Data Sources

### 4.1. XTS vs Forward Shipment Snapshot (ORDERBOOK.md)

XTS transactions feed into the forward shipment snapshot. To validate:

```sql
-- XTS Total
SELECT CAST(SUM(COGS_USD) AS DECIMAL(38,10)) AS XTS_Total
FROM lft.bronze.dds_ssbi_xts_order_transaction_tab
WHERE SHIP_YEAR = '2025' AND STREAM_CODE = 'DEN';

-- Snapshot Total (same scope)
SELECT CAST(SUM(RPT_FOBC) AS DECIMAL(38,10)) AS Snapshot_Total
FROM lft.bronze.dds_ssbi_frwshp_hist_snapshot
WHERE WEEK_NO = '47'
  AND CAST(SUBSTR(CAST(SNAPSHOT_DATE AS STRING), 1, 4) AS INT) = 2025
  AND SHIP_YEAR = '2025'
  AND REPORT_TYPE = 'C'
  AND TRANS_STATUS = 'A'
  AND REPORT_STREAM_CODE = 'DEN'
  AND SOURCE = 'XTS';
```

**Expected variance:** 1-5% due to timing differences and snapshot point-in-time capture.

### 4.2. XTS vs Financials (FINANCIALS.md)

XTS COGS should approximate financial Turnover, but expect variance due to:
- Timing differences (order date vs recognition date)
- Currency translation
- Adjustments and allocations in Plan/CC cube
- Different granularity

**Mapping:**

| XTS Column | Financials Equivalent |
|------------|----------------------|
| `OPERATING_GROUP_DESC` | `ou.ent3name` |
| `STREAM_CODE` | `ou.ent4` (with `ST:` prefix and `_FC` suffix) |
| `DIVISION_CODE` | `ou.ent6` (with `DIV:` prefix and `_FC` suffix) |
| `CORPORATE_CUSTOMER_CODE` | `cust.element` (with `PCUST:` prefix) |
| `COGS_USD` (sum) | Turnover (`ac.level9 = 'P651000MT'`) x 1000 |

### 4.3. Entity Code Mapping (Simple Cases)

For **some** XTS codes, a direct string transformation works:

| XTS | Financials | Example |
|-----|------------|---------|
| `STREAM_CODE = 'DEN'` | `ou.ent4 = 'ST:DEN_FC'` | Denim Lifestyle |
| `DIVISION_CODE = 'AE1'` | `ou.ent6 = 'DIV:AE1_FC'` | AE1 Division |
| `CORPORATE_CUSTOMER_CODE = 'AEOI'` | `cust.element = 'PCUST:AEOI'` | American Eagle |

### 4.4. Division Mapping to Entity Dimension (Conformed Dimension)

**CRITICAL:** XTS `DIVISION_CODE` does **not** always match Financials `ent6` by simple string transformation. The Entity dimension (`jedoxviewer_entity`) is the **conformed dimension** for management accounting structures. XTS division codes must be mapped to Entity codes via the Forward Shipment Snapshot's `REPORT_DIVISION_CODE`.

**Why this matters:**
- XTS is an operational system with its own division codes
- Financials uses Entity dimension codes for management reporting
- Many XTS divisions consolidate into single Entity divisions
- Some XTS divisions are renamed when mapped to Entity
- New divisions may only have valid mappings in recent snapshot years

**Mapping Patterns Found:**

| Pattern | Example XTS → Entity |
|---------|---------------------|
| **1:1 Match** | `AE1` → `DIV:AE1_FC` |
| **Many:1 Consolidation** | `AC1, AC3` → `DIV:ABADIV_FC` |
| **Many:1 Consolidation** | `AKI, ALA, AME, ASP` → `DIV:APLAD_FC` |
| **Rename** | `COMET` → `DIV:EONCM_FC` |
| **Rename** | `BCUKA` → `DIV:EONBC_FC` |

**Building the Division Mapping Table:**

Use the Forward Shipment Snapshot (`dds_ssbi_frwshp_hist_snapshot`) which contains both XTS codes and their corresponding Entity codes:

```sql
-- Division mapping from XTS to Entity dimension
CREATE OR REPLACE TEMP VIEW div_map AS 
WITH ranked AS (
  SELECT
    DIVISION_CODE,
    REPORT_DIVISION_CODE,
    ROW_NUMBER() OVER (
      PARTITION BY DIVISION_CODE
      ORDER BY SNAPSHOT_DATE DESC
    ) AS rn
  FROM
    lft.bronze.dds_ssbi_frwshp_hist_snapshot
  WHERE
    WEEK_NO <> 99
    AND REPORT_DIVISION_CODE <> 'LFCCOR'  -- Exclude corporate catch-all
    AND REPORT_DIVISION_CODE IS NOT NULL
    AND DIVISION_CODE IS NOT NULL
)
SELECT
  DIVISION_CODE AS XTS_DIV,
  REPORT_DIVISION_CODE AS JEDOX_DIV,
  CONCAT('DIV:', REPORT_DIVISION_CODE, '_FC') AS ENT6_CODE
FROM ranked
WHERE rn = 1;
```

**Important:** Do NOT filter by historical years only (e.g., `2021-2024`). New divisions like `AES` (AEO Accessories) first appear in 2025 and will be excluded if year-filtered.

**Using the Mapping:**

```sql
-- Join XTS data to Entity dimension via mapping
SELECT 
    ou.ent6name AS Division,
    ou.ent3name AS OG,
    ROUND(SUM(x.COGS_USD) / 1000000, 2) AS COGS_M
FROM lft.bronze.dds_ssbi_xts_order_transaction_tab x
JOIN div_map dm ON x.DIVISION_CODE = dm.XTS_DIV
JOIN fna.fna_gold.jedoxviewer_entity ou 
    ON dm.ENT6_CODE = ou.ent6 
    AND ou.ent1 = 'OG_Forecast'
WHERE x.SHIP_YEAR = '2025'
GROUP BY ou.ent6name, ou.ent3name
ORDER BY COGS_M DESC;
```

**Coverage Statistics (as of Nov 2025):**
- ~179 XTS divisions have direct Entity matches
- ~125 XTS divisions require mapping via snapshot
- ~23 divisions only have valid mappings starting in 2025

---

## 5. Data Quality Notes

### 5.1. Known Issues

- **Supplier country vs Production country:** These can differ - Korean suppliers often produce in Vietnam/Bangladesh
- **NULL business nature:** Some records have NULL `BUSINESS_NATURE` - treat as unknown
- **Multiple extraction dates:** Use latest `extraction_date` for current state analysis
- **Division code mismatch:** XTS `DIVISION_CODE` ≠ Financials `ent6` in many cases. Always use the snapshot-based mapping (see Section 4.4). Direct string transformation (`CONCAT('DIV:', DIVISION_CODE, '_FC')`) will miss consolidations and renames.

### 5.2. Recommended Filters

```sql
-- For current state analysis
WHERE extraction_date = (SELECT MAX(extraction_date) FROM lft.bronze.dds_ssbi_xts_order_transaction_tab)

-- For confirmed orders only (exclude potential)
AND PR_STATUS_DESC = 'Confirmed'

-- For shipped orders only
AND ORDER_STATUS = 'CLOSED'
```

---

## 6. Glossary

| Term | Definition |
|------|------------|
| **XTS** | eXtended Transaction System - primary order management system |
| **COGS** | Cost of Goods Sold - primary value metric |
| **FOB** | Free on Board - shipping/pricing term |
| **SO** | Sales Order |
| **PO** | Purchase Order |
| **PR** | Production Request |
| **Agency** | Business model where LF acts on behalf of buyer |
| **Principal** | Business model where LF takes ownership of goods |
| **Intercompany** | Transactions between LF entities |
| **Imputed Principal** | Agency transactions to intercompany buyers (treated as Principal at group level) |

# Orderbook Data Dictionary

Forward-looking order data for customer orders scheduled for future shipment. Complements historical actuals in [FINANCIALS.md](./FINANCIALS.md).

**Key Context:** Average lead time ~100 days for SCS. Orderbook is a leading indicator for turnover 3-4 months ahead.

---

## Critical: WEEK_NO = 99 Trap

**NEVER use `MAX(SNAPSHOT_DATE)` alone!** Each snapshot contains BOTH regular weekly data (`WEEK_NO = '01'-'53'`) AND cumulative aggregation (`WEEK_NO = '99'`). Using `MAX(SNAPSHOT_DATE)` without filtering doubles results.

**ALWAYS filter:** `WEEK_NO = 'XX'` (specific week) OR `WEEK_NO < '99'` (exclude aggregation)

**Value scaling:** Orderbook values in actual USD (no scaling). Plan/CC cube in thousands.

---

## 1. Core Concepts

### Order on Hand (OOH)

Cumulative dollar value of confirmed orders scheduled for shipment within a calendar year.

**Critical - OOH Includes Shipped Orders:** OOH represents **full year cumulative** - includes both shipped and not-yet-shipped. If OOH *decreases* week-over-week, it's due to:
1. **Cancellations** - orders removed
2. **Slippage** - orders re-dated to future year

When analyzing late-year OOH declines, check if next-year orders (`REPORT_TYPE = 'N'`) are increasing correspondingly.

### Report Types

| Code | Name | SHIP_YEAR relative to Snapshot | Description |
|------|------|--------------------------------|-------------|
| `C` | Current | Same year | Orders to ship same year as snapshot |
| `N` | Next | +1 year | Orders to ship year after snapshot |
| `L` | Last | -1 year | Prior year shipped (for YoY) |
| `X` | Other | Various | Special (e.g., Firework seasonal) |

### Key Metrics

| Metric | Definition |
|--------|------------|
| **Filled Rate** | `(OOH at Week X) / (Target Turnover) * 100%` |
| **Weekly Orders** | Net new orders in a week |
| **Rolling 4-Week** | Sum of net new orders over 4 weeks |

---

## 2. Data Source

**Primary Table:** `lft.bronze.dds_ssbi_frwshp_hist_snapshot`

For fill rate denominator, use Plan/CC cube turnover from [FINANCIALS.md](./FINANCIALS.md) (ensures consistency).

### Schema - Key Columns

| Column | Description |
|--------|-------------|
| `SNAPSHOT_DATE` | Date of snapshot (YYYYMMDD string) |
| `WEEK_NO` | Week number (01-53, 99 = special) |
| `SHIP_YEAR` | Year order scheduled to ship |
| `SHIP_MONTH` | Month order scheduled to ship |
| `REPORT_TYPE` | 'C', 'N', 'L', 'X' |
| `TRANS_STATUS` | 'A' = Active, 'I' = Inactive |
| `SOURCE` | Data source: XTS, OnShore, OBADJ |
| `REPORT_OPERATING_GROUP_CODE` | OG code (SCS1, SCS2, LFMU, etc.) |
| `REPORT_OPERATING_GROUP_DESC` | OG name |
| `REPORT_STREAM_CODE/DESC` | Stream |
| `REPORT_DIVISION_CODE/DESC` | Division |
| `REPORT_CUSTOMER_NAME` | Customer (individual entity) |
| `REPORT_CORPORATE_CUSTOMER_CODE/NAME` | Corporate customer (parent grouping) |
| `RPT_FOBC` | **Primary metric** - FOB value (actual USD) |

---

## 3. Standard Filters

### Exclusions (Always Apply)

```sql
REPORT_OPERATING_GROUP_CODE NOT IN (
    'COB', 'FG', 'LFL', 'LFX', 'SCSSC', 'GCADJ', 'LFOSG', 'LFF', 'LFBT', 'LFP'
)
AND SOURCE != 'LFB'
AND TRANS_STATUS = 'A'
```

### Week Filter

```sql
-- Specific week
WEEK_NO = 'NN'
-- Or exclude week 99
WEEK_NO < '99'
```

---

## 4. Source Systems

| SOURCE | OGs | Notes |
|--------|-----|-------|
| **XTS** | Apparel, H&A, LFEU, LFMU, Promocean, LFAD | Primary - use as "latest week" indicator |
| **OnShore** | Firework, LFEU, LFMU, Promocean | Markets/onshore system |
| **OBADJ** | LF Asia Direct | Adjustments |

**Latest complete week:** Use `MAX(WEEK_NO) WHERE SOURCE = 'XTS' AND WEEK_NO < '99'`

---

## 5. Business OGs

| Code | Description | Segment |
|------|-------------|---------|
| `SCS1` | Apparel | SCS |
| `SCS2` | Home And Accessories | SCS |
| `LFMU` | LF Markets USA | Markets |
| `LFFA` | LF Europe | Markets |
| `LFEU` | Promocean and EU Others | Markets |
| `FWK` | Firework | Markets |
| `LFAD` | LF Asia Direct | Markets |

**Note:** `LFFA` (LF Europe) and `LFEU` (Promocean) are separate OGs.

---

## 6. Common Query Patterns

### Find Latest Complete Week

```sql
SELECT MAX(WEEK_NO) AS latest_week
FROM lft.bronze.dds_ssbi_frwshp_hist_snapshot
WHERE SOURCE = 'XTS' AND WEEK_NO < '99'
  AND CAST(SUBSTR(CAST(SNAPSHOT_DATE AS STRING), 1, 4) AS INT) = 2025
  AND TRANS_STATUS = 'A';
```

### Current Year OOH by OG

```sql
SELECT REPORT_OPERATING_GROUP_DESC, CAST(SUM(RPT_FOBC) AS DECIMAL(38,10)) AS OOH
FROM lft.bronze.dds_ssbi_frwshp_hist_snapshot
WHERE WEEK_NO = 'NN'
  AND CAST(SUBSTR(CAST(SNAPSHOT_DATE AS STRING), 1, 4) AS INT) = 2025
  AND SHIP_YEAR = '2025' AND REPORT_TYPE = 'C' AND TRANS_STATUS = 'A'
  AND REPORT_OPERATING_GROUP_CODE NOT IN ('COB','FG','LFL','LFX','SCSSC','GCADJ','LFOSG','LFF','LFBT','LFP')
  AND SOURCE != 'LFB'
GROUP BY REPORT_OPERATING_GROUP_DESC ORDER BY OOH DESC;
```

### YoY Comparison

```sql
SELECT 
    CAST(SUM(CASE WHEN snap_year = 2025 THEN RPT_FOBC ELSE 0 END) AS DECIMAL(38,10)) AS TY,
    CAST(SUM(CASE WHEN snap_year = 2024 THEN RPT_FOBC ELSE 0 END) AS DECIMAL(38,10)) AS LY
FROM (
    SELECT *, CAST(SUBSTR(CAST(SNAPSHOT_DATE AS STRING), 1, 4) AS INT) AS snap_year
    FROM lft.bronze.dds_ssbi_frwshp_hist_snapshot
    WHERE WEEK_NO = 'NN' AND TRANS_STATUS = 'A' AND REPORT_TYPE = 'C'
      AND REPORT_OPERATING_GROUP_CODE NOT IN ('COB','FG','LFL','LFX','SCSSC','GCADJ','LFOSG','LFF','LFBT','LFP')
      AND SOURCE != 'LFB'
) sub
WHERE (snap_year = 2025 AND SHIP_YEAR = '2025') OR (snap_year = 2024 AND SHIP_YEAR = '2024');
```

### Matching Report Numbers to Weeks

When user provides report numbers, query multiple recent weeks to find which matches:

```sql
SELECT WEEK_NO,
    CAST(SUM(CASE WHEN snap_year = 2025 THEN RPT_FOBC ELSE 0 END) AS DECIMAL(38,2)) AS TY,
    CAST(SUM(CASE WHEN snap_year = 2024 THEN RPT_FOBC ELSE 0 END) AS DECIMAL(38,2)) AS LY
FROM (
    SELECT *, CAST(SUBSTR(CAST(SNAPSHOT_DATE AS STRING), 1, 4) AS INT) AS snap_year
    FROM lft.bronze.dds_ssbi_frwshp_hist_snapshot
    WHERE WEEK_NO IN ('40', '41', '42') AND TRANS_STATUS = 'A' AND REPORT_TYPE = 'C'
      AND SOURCE != 'LFB'
) sub
WHERE (snap_year = 2025 AND SHIP_YEAR = '2025') OR (snap_year = 2024 AND SHIP_YEAR = '2024')
GROUP BY WEEK_NO ORDER BY WEEK_NO;
```

---

## 7. Fill Rate Calculation

```
Fill Rate % = (OOH in actual USD) / (Target Turnover * 1000) * 100
```

**CRITICAL:** OOH is actual USD. Plan/CC cube is thousands. Scale target by 1000.

**Steps:**
1. Get OOH from snapshot (actual USD)
2. Get target from Plan/CC cube (thousands) - see [FINANCIALS.md](./FINANCIALS.md)
3. Multiply target by 1000, divide OOH by result

---

## 8. Cross-Year Queries

Orders for next year use different REPORT_TYPE depending on snapshot year:
- 2025 snapshot, 2026 orders → `REPORT_TYPE = 'N'`
- 2026 snapshot, 2026 orders → `REPORT_TYPE = 'C'`

```sql
-- 2026 orders from 2025 snapshots (Q4)
SELECT WEEK_NO, CAST(SUM(RPT_FOBC) AS DECIMAL(38,10)) AS OOH
FROM lft.bronze.dds_ssbi_frwshp_hist_snapshot
WHERE CAST(SUBSTR(CAST(SNAPSHOT_DATE AS STRING), 1, 4) AS INT) = 2025
  AND WEEK_NO >= '40' AND WEEK_NO < '99'
  AND SHIP_YEAR = '2026' AND REPORT_TYPE = 'N' AND TRANS_STATUS = 'A'
  AND REPORT_OPERATING_GROUP_CODE NOT IN ('COB','FG','LFL','LFX','SCSSC','GCADJ','LFOSG','LFF','LFBT','LFP')
  AND SOURCE != 'LFB'
GROUP BY WEEK_NO;
```

---

## 9. Slip Detection

When current-year OOH decreases late in year:

```sql
SELECT WEEK_NO,
    CAST(SUM(CASE WHEN REPORT_TYPE = 'C' AND SHIP_YEAR = '2025' 
         THEN RPT_FOBC ELSE 0 END) AS DECIMAL(38,10)) AS OOH_2025,
    CAST(SUM(CASE WHEN REPORT_TYPE = 'N' AND SHIP_YEAR = '2026' AND SHIP_MONTH IN ('1','2','3')
         THEN RPT_FOBC ELSE 0 END) AS DECIMAL(38,10)) AS Q1_2026
FROM lft.bronze.dds_ssbi_frwshp_hist_snapshot
WHERE WEEK_NO IN ('45', '46', '47', '48')
  AND CAST(SUBSTR(CAST(SNAPSHOT_DATE AS STRING), 1, 4) AS INT) = 2025
  AND TRANS_STATUS = 'A' AND SOURCE != 'LFB'
GROUP BY WEEK_NO ORDER BY WEEK_NO;
```

**Interpretation:**
- 2025 OOH drops $5M, Q1 2026 rises ~$5M → **Slip**
- 2025 OOH drops $5M, Q1 2026 flat → **Cancellation**

---

## 10. Mapping to Financials

### Division

| Orderbook | Financials |
|-----------|------------|
| `REPORT_DIVISION_CODE` = `AE1` | `ou.ent6` = `DIV:AE1_FC` |

```sql
CONCAT('DIV:', ob.REPORT_DIVISION_CODE, '_FC') = ou.ent6
```

### Customer

Use `REPORT_CORPORATE_CUSTOMER_CODE` (not individual `REPORT_CUSTOMER_CODE`):

| Orderbook | Financials |
|-----------|------------|
| `REPORT_CORPORATE_CUSTOMER_CODE` = `AEOI` | `cust.element` = `PCUST:AEOI` |

```sql
CONCAT('PCUST:', ob.REPORT_CORPORATE_CUSTOMER_CODE) = cust.element
```

**Coverage by OG:** Apparel/Firework: 100%, H&A/LFMU: 98-99%, LFEU: ~87%, Promocean: 97%

### OG

| Orderbook | Financials |
|-----------|------------|
| `REPORT_OPERATING_GROUP_DESC` = `Apparel` | `ou.ent3name` = `Apparel` |

---

## 11. Breakdown Dimensions

| Level | Column |
|-------|--------|
| Operating Group | `REPORT_OPERATING_GROUP_DESC` |
| Stream | `REPORT_STREAM_DESC` |
| Product Group | `REPORT_PRODUCT_GROUP_DESC` |
| Division | `REPORT_DIVISION_DESC` |
| Customer | `REPORT_CUSTOMER_NAME` |
| Corporate Customer | `REPORT_CORPORATE_CUSTOMER_NAME` |
| Ship Month | `SHIP_MONTH` |

**Use Corporate Customer** for analysis to avoid splitting across legal entities.

---

## 12. Technical Notes

### SQL Aggregation

Always use SQL `SUM()` with `GROUP BY`. Never fetch raw rows and aggregate in Python (truncation risk).

```sql
CAST(SUM(RPT_FOBC) AS DECIMAL(38, 10)) AS OOH_Value
```

### Data Limits

Results may truncate at ~1000 rows. For high-cardinality dimensions (customer), verify totals match.

### Data Availability

- **Snapshot years:** 2017-present
- **Ship years:** Filter to realistic years (garbage like 2099 exists)
- **Weeks:** 01-52/53 + 99 (special)

---

## 13. Business Context

### Lead Time

| Business | Lead Time |
|----------|-----------|
| SCS (Apparel, H&A) | ~100 days |
| Markets (Onshore) | Up to 6 months |
| Firework | Seasonal (build Jan-Sep, ship Oct-Dec) |

### Seasonal Patterns

- **Peak booking:** March-June for H2 shipments
- **Q4 orders:** Next-year orders from ~Week 40
- **Firework:** Peak fill rate by Week 40

---

## 14. Related Sources

- **[XTS.md](./XTS.md):** Granular order-level data (production country, supplier, product). XTS feeds snapshot; expect 1-5% variance.
- **[FINANCIALS.md](./FINANCIALS.md):** Financial actuals, Plan/CC cube for targets


# FTE (Full-Time Equivalent) Employee Data Dictionary

This document covers the FTE table which provides employee-level headcount data. This is the source of truth for individual employee records and maps to the Plan/CC Cube headcount metric.

**Key Use Cases:**
- Employee roster and headcount by entity/OG/country
- Grade distribution analysis
- FTE allocation across business units (for shared resources)
- Headcount trend analysis over time

**Relationship to Other Data Sources:**
- **FINANCIALS.md**: FTE table totals reconcile to Plan/CC Cube headcount (`ac.level2 = 'Headcount'`) within ~1 FTE
- Entity dimension is the conformed dimension for management reporting

---

## Agent Notes

For **aggregated headcount** by OG/entity, use `query_financial_metric` with `metric="headcount"`.

For **employee-level detail** (roster, grades, allocations), use `run_sql` against the FTE table.

**Key table:** `fna.fna_gold.jedoxviewer_fte`

**Common query patterns:**
- Current headcount: Filter `YM = '202510'` (YYYYMM format)
- By entity: Join to `jedox_dimpc_entity_detail` to map BU to management entity
- By grade: Group by `Grade` column

**Note:** Employees can be split across multiple BUs. Sum FTE column, not count rows.

---

## 1. Core Table

### 1.1. Primary Table: `fna.fna_gold.jedoxviewer_fte`

| Attribute | Value |
|-----------|-------|
| **Granularity** | One row per employee per BU per period |
| **History** | May 2020 (202005) to present |
| **Update Frequency** | Monthly |
| **Total Records** | ~724K rows |
| **Unique Employees** | ~26K (all-time), ~4.2K (current) |

---

## 2. Schema

| Column | Type | Description | Example |
|--------|------|-------------|---------|
| `emp_id` | string | Unique employee identifier | '312162' |
| `emp_name` | string | Employee name | 'John Smith' |
| `country` | string | Employee work location country | 'Hong Kong' |
| `Grade` | string | Employee grade/level | '5A' |
| `BU` | string | Business Unit code (E1 entity) | '215238603' |
| `FTE` | double | FTE allocation (0.0 to 1.0) | 1.0 |
| `YM` | string | Year-Month period (YYYYMM) | '202510' |
| `Co` | string | Legal company code | '00001' |
| `email` | string | Employee email (or 'None') | 'john@lifung.com' |

---

## 3. Key Concepts

### 3.1. FTE Allocation

Employees can be split across multiple Business Units. Each row represents an allocation:

```
emp_id=000178 (Melinda Chu) in Oct 2025:
  BU=001308101  FTE=0.3  → TAL-HK-MG1 (Apparel)
  BU=001308001  FTE=0.3  → ATL-HK-MG1 (Apparel)
  BU=001400702  FTE=0.2  → MAU-HK-SC1 (Apparel)
  BU=001307901  FTE=0.1  → EXP-HK-MG1 (Apparel)
  BU=001447501  FTE=0.1  → CHSAM-HK-SC1 (Apparel)
  Total: 1.0 FTE
```

**Rule:** Sum of FTE per employee per period = 1.0 (or close to it)

### 3.2. Grade Structure

Grades changed from letter-based (A-E) to numeric (1-8 with A/B suffix) in **September 2022**.

**Current Grade Structure (post-Sep 2022):**

| Grade | Band | Typical Count |
|-------|------|---------------|
| S | Senior Leadership | ~17 |
| 1A, 1B | Executive | ~56 |
| 2A, 2B | Senior Management | ~148 |
| 3A, 3B | Management | ~540 |
| 4A, 4B | Professional | ~1,040 |
| 5A, 5B | Specialist | ~1,720 |
| 6A, 6B | Associate | ~524 |
| 7, 8 | Support | ~31 |
| Temp | Temporary | ~9 |
| NA | Not Assigned | ~70 |

**Legacy Grade Structure (pre-Sep 2022):**

| Grade | Approximate Equivalent |
|-------|----------------------|
| A | Executive (1A/1B) |
| B | Senior Mgmt (2A/2B) |
| C | Management (3A/3B) |
| D | Professional (4A/4B) |
| E | Specialist (5A/5B) |

### 3.3. BU to Entity Mapping

The `BU` column contains E1 Business Unit codes that map to the Entity dimension via `jedox_dimpc_entity_detail`:

```sql
-- Map FTE to Entity hierarchy
SELECT 
    f.*,
    ed.parent AS entb,
    ou.ent3name AS OG,
    ou.ent6name AS Division
FROM fna.fna_gold.jedoxviewer_fte f
JOIN fna.fna_silver.jedox_dimpc_entity_detail ed 
    ON f.BU = ed.child
JOIN fna.fna_gold.jedoxviewer_entity ou 
    ON ed.parent = ou.entb 
    AND ou.ent1 = 'OG_Forecast'
WHERE f.YM = '202510';
```

**Coverage:** ~99.8% of FTE records map successfully to Entity dimension.

---

## 4. Current Headcount Summary (Oct 2025)

### By Segment/OG

| Segment | OG | Employees | FTE |
|---------|-----|-----------|-----|
| Markets | LF Europe | 806 | 792 |
| Functions | Functions | 756 | 749 |
| SCS | Home And Accessories | 693 | 680 |
| Logistics | GFS | 639 | 639 |
| SCS | Apparel | 556 | 532 |
| Markets | LF Markets USA | 408 | 384 |
| Markets | Promocean and EU Others | 141 | 116 |
| Corporate | Corporate | 110 | 104 |
| Markets | Firework | 84 | 83 |
| LFX | LFX | 51 | 50 |
| Markets | LF Asia Direct | 31 | 31 |

### By Country (Top 10)

| Country | Employees | FTE |
|---------|-----------|-----|
| China | 1,803 | 1,802 |
| Hong Kong | 490 | 490 |
| India | 356 | 356 |
| Bangladesh | 283 | 283 |
| Vietnam | 193 | 193 |
| United States | 176 | 176 |
| United Kingdom | 175 | 171 |
| Germany | 167 | 158 |
| Turkey | 116 | 116 |
| Indonesia | 82 | 82 |

---

## 5. Standard Query Patterns

### 5.1. Basic Headcount by OG

```sql
WITH fte_entity AS (
    SELECT f.*, ed.parent AS entb
    FROM fna.fna_gold.jedoxviewer_fte f
    JOIN fna.fna_silver.jedox_dimpc_entity_detail ed ON f.BU = ed.child
    WHERE f.YM = '202510'
)
SELECT 
    ou.ent3name AS OG,
    COUNT(DISTINCT fe.emp_id) AS employees,
    ROUND(SUM(fe.FTE), 1) AS total_fte
FROM fte_entity fe
JOIN fna.fna_gold.jedoxviewer_entity ou 
    ON fe.entb = ou.entb AND ou.ent1 = 'OG_Forecast'
GROUP BY ou.ent3name
ORDER BY total_fte DESC;
```

### 5.2. Headcount Trend

```sql
SELECT 
    YM,
    COUNT(DISTINCT emp_id) AS employees,
    ROUND(SUM(FTE), 1) AS total_fte
FROM fna.fna_gold.jedoxviewer_fte
WHERE YM >= '202401'
GROUP BY YM
ORDER BY YM;
```

### 5.3. Grade Distribution

```sql
SELECT 
    CASE 
        WHEN Grade = 'S' THEN '1-Senior Leadership'
        WHEN Grade LIKE '1%' THEN '2-Executive'
        WHEN Grade LIKE '2%' THEN '3-Senior Management'
        WHEN Grade LIKE '3%' THEN '4-Management'
        WHEN Grade LIKE '4%' THEN '5-Professional'
        WHEN Grade LIKE '5%' THEN '6-Specialist'
        WHEN Grade LIKE '6%' THEN '7-Associate'
        WHEN Grade IN ('7','8') THEN '8-Support'
        ELSE '9-Other'
    END AS grade_band,
    COUNT(DISTINCT emp_id) AS employees,
    ROUND(SUM(FTE), 1) AS total_fte
FROM fna.fna_gold.jedoxviewer_fte
WHERE YM = '202510'
GROUP BY 1
ORDER BY 1;
```

### 5.4. Reconcile to Plan/CC Cube

```sql
-- FTE Table
SELECT 'FTE Table' AS source, ROUND(SUM(FTE), 1) AS total
FROM fna.fna_gold.jedoxviewer_fte
WHERE YM = '202510'

UNION ALL

-- Plan/CC Cube Headcount
SELECT 'Plan/CC Cube' AS source, ROUND(SUM(fact.signed), 1) AS total
FROM fna.fna_gold.jedoxviewer_plan_cc_transformed fact
JOIN fna.fna_gold.jedoxviewer_entity ou ON fact.entity_plan = ou.entb
JOIN fna.fna_silver.jedox_dimfh_account_plan ac ON fact.account_plan = ac.element
WHERE ou.ent1 = 'OG_Forecast'
  AND fact.currency = '> USD'
  AND ac.level2 = 'Headcount'
  AND fact.period_key = '2025-10 actual';
```

**Expected:** Values should match within ~1 FTE.

---

## 6. Data Quality Notes

- **Unmapped BUs:** ~0.2% of records have BU codes not in entity hierarchy (typically special/legacy codes)
- **Email field:** Some records have 'None' instead of actual email
- **Country spelling:** Turkey appears as both 'Turkey' and 'Türkiye' in historical data
- **Grade transitions:** Grade structure changed in Sep 2022; historical comparisons need mapping

---

## 7. Glossary

| Term | Definition |
|------|------------|
| **FTE** | Full-Time Equivalent - 1.0 = full-time employee |
| **BU** | Business Unit - E1 entity code for cost allocation |
| **Co** | Company - Legal entity code |
| **Grade** | Employee level in organizational hierarchy |
| **AM** | Account Management - customer-facing roles |
| **SPP** | Sourcing and Production Platform - supply-side roles |
| **MR** | Merchandising roles |
| **QAQC** | Quality Assurance/Quality Control roles |


# Financials — Plan/CC Cube

Canonical reference for financial actuals, budgets, forecasts, P&L, working capital, and Plan/CC Cube analysis. Read the dedicated root document for other data domains.

Related domains: [ORDERBOOK.md](./ORDERBOOK.md), [XTS.md](./XTS.md), [FTE.md](./FTE.md), [E1_DETAIL.md](./E1_DETAIL.md), [AR_AGING.md](./AR_AGING.md), and [AR_AP_SUBLEDGER.md](./AR_AP_SUBLEDGER.md).

---

# 1. Critical Rules & Concepts

This section contains the most important information. Internalize these before writing any queries.

## 1.1 The Golden Rules of SQL Construction

**Strictly adhere to these constraints for every financial query.**

1. **Hierarchy:** ALWAYS filter `WHERE ou.ent1 = 'OG_Forecast'`
   - This selects the correct reporting hierarchy for Actuals, Forecasts, _and_ Budget
   - Do _not_ use `ent1 = 'OG_Budget'` or `_BG` entities unless explicitly requested

2. **Currency:** ALWAYS filter `WHERE fact.currency = '> USD'` (unless requested otherwise)

3. **Scaling:** ALWAYS multiply `SUM(fact.signed)` by **1000** for financial metrics
   - Raw data is in thousands. Failure to scale = incorrect millions/billions
   - **EXCEPTION:** Do **NOT** scale **Headcount** (`ac.level2 = 'Headcount'`) - stored as actual FTEs

4. **Granularity:** **Fetch Broadly.**
   - Query multiple core metrics (Turnover, Revenue, OPEX, C2) in a single pass when possible.
   - It is faster to filter unused data in Python than to incur the latency of multiple narrow queries.
   - **Exception:** If complex `CASE` logic threatens accuracy or performance, then split them.

## 1.2 Known Data Traps

Memorize these to avoid common mistakes:

1. **Double-space in "Direct -  MPC"** - Account description has TWO spaces between hyphen and MPC
2. **OPEX sign convention** - Stored as NEGATIVE in DB, displayed as POSITIVE in reports
3. **Headcount not scaled** - Do NOT multiply by 1000 (unlike financial metrics)
4. **WC in multiple hierarchies** - Filter `ac.level2 = 'WC'` to avoid Balance Sheet duplicates
5. **Unmapped entities (`XXX-XX-XXX`)** - Flag as "unallocated" rather than ignoring - may be significant amounts

## 1.3 Key Terminology: Entity vs Company

**IMPORTANT:** These two concepts are distinct and orthogonal:

| Term | What It Is | Example | Table/Column |
|------|------------|---------|--------------|
| **Management Entity** (`entb`) | Organizational unit for management reporting - like a cost center or division. Used in Plan/CC Cube for P&L reporting. | `AE1-HK-MC1` (AE1 Div), `VOASD-UK-USD` (Voice Apparel Selling) | `jedoxviewer_entity.entb` |
| **Legal Company** (`Co`) | Actual legal entity that books transactions in ERP. Used for statutory reporting and intercompany flows. | `00001` (LI & FUNG TRADING LTD), `00492` (LF FASHION LIMITED) | `jedox_dimfh_company.element`, FTE `Co` column |

**Key points:**
- A single **management entity** can have transactions from **multiple legal companies** (e.g., same division booked via HK and US legal entities)
- A single **legal company** can have transactions in **multiple management entities** (e.g., HK trading company books across many divisions)
- The entity hierarchy (`ent1` → `ent7`) is for **management reporting** (OG → Stream → PG → Division → OU)
- Legal company is tracked separately via `BU` codes in E1 (first 3 digits = company code) or `Co` column in FTE

When users ask about "entity", they almost always mean **management entity** (the `entb`/division concept), not legal company.

# 2. Data Sources Overview

## 2.1 Available Data Domains

| Document | Purpose | Primary Use Cases |
|----------|---------|-------------------|
| **FINANCIALS** | Financial actuals, budgets, forecasts | P&L analysis, variance analysis, margin tracking |
| **[E1_DETAIL.md](./E1_DETAIL.md)** | ERP journal line items | Vendor drill-down, transaction-level investigation, reconciliation |
| **[ORDERBOOK.md](./ORDERBOOK.md)** | Forward-looking order data | Fill rate tracking, OOH trends, demand forecasting |
| **[XTS.md](./XTS.md)** | Order transaction details | Qualitative analysis, supplier/country mix, product breakdown |
| **[FTE.md](./FTE.md)** | Employee headcount data | Headcount by entity/country/grade, employee roster |

## 2.2 Data Source Hierarchy

```
                    +------------------+
                    |   XTS (Source)   |
                    | Order Transactions|
                    +--------+---------+
                             |
              +--------------+--------------+
              |                             |
              v                             v
    +------------------+          +------------------+
    | Forward Shipment |          |   Plan/CC Cube   |
    |    Snapshot      |          |   (Financials)   |
    | (ORDERBOOK.md)   |          |  (this file)     |
    +------------------+          +------------------+
              |                             |
              |   ~1-5% variance expected   |
              +-----------------------------+
```

## 2.3 When to Use Each Source

### Financials / Plan/CC Cube (this file)
**Use for:** Official financial reporting, P&L analysis, budget/forecast comparisons

- Turnover, Revenue, OPEX, C2/C3 metrics
- Actual vs Budget/1QR/2QR/3QR variance
- Year-over-year comparisons
- Headcount analysis
- Working capital and balance sheet items

**Key characteristics:**
- Values in **thousands USD** (multiply by 1000)
- Official source of truth for financial metrics
- Includes management adjustments and allocations

### ORDERBOOK.md (Forward Shipment Snapshot)
**Use for:** Forward-looking analysis, fill rate tracking, demand visibility

- Order on Hand (OOH) by week
- Fill rate vs budget/forecast
- Next year order buildup
- Customer order patterns

**Key characteristics:**
- Values in **actual USD** (no scaling)
- Point-in-time snapshots by week
- Leading indicator for turnover (~100 days ahead for SCS)

### XTS.md (Order Transactions)
**Use for:** Qualitative deep-dives, operational analysis

- Production country mix
- Supplier/vendor concentration
- Product category breakdown
- Customer channel analysis
- Business nature (Agency vs Principal)

**Key characteristics:**
- Values in **actual USD** (no scaling)
- Most granular - order line level
- Rich dimensional attributes (factory, product, geography)
- Cannot perfectly reconcile to financials (~1-5% variance expected)

### ERP Data: jedox_actual vs e1_details

**When to consult [E1_DETAIL.md](./E1_DETAIL.md):**

| Question Type | Start With |
|---------------|------------|
| "What's in company X?" / "TB for legal entity" | `jedox_actual` |
| "Who are we paying?" / "Transactions for vendor X" | `e1_details` |
| "Reconcile Plan/CC to ERP" | `jedox_actual` → `e1_details` |

**Key rule:** Start with `jedox_actual` (Trial Balance) for company-level analysis, then drill into `e1_details` only when you need transaction-level detail (counterparties, document types).

See **[E1_DETAIL.md](./E1_DETAIL.md)** for table schemas, dimension traversal patterns, and joining techniques.

## 2.4 Cross-Reference Mappings

### Entity Codes

| Level | XTS | Financials | Orderbook |
|-------|-----|------------|-----------|
| OG | `OPERATING_GROUP_CODE` = 'SCS1' | `ou.ent3name` = 'Apparel' | `REPORT_OPERATING_GROUP_DESC` = 'Apparel' |
| Stream | `STREAM_CODE` = 'DEN' | `ou.ent4` = 'ST:DEN_FC' | `REPORT_STREAM_CODE` = 'DEN' |
| Division | `DIVISION_CODE` = 'AE1' | `ou.ent6` = 'DIV:AE1_FC' | `REPORT_DIVISION_CODE` = 'AE1' |
| Customer | `CORPORATE_CUSTOMER_CODE` = 'AEOI' | `cust.element` = 'PCUST:AEOI' | `REPORT_CORPORATE_CUSTOMER_CODE` = 'AEOI' |

### Value Scaling

| Source | Scale | Example |
|--------|-------|---------|
| FINANCIALS (Plan/CC Cube) | Thousands | $599,333 = $599.3M |
| ORDERBOOK (Snapshot) | Actual USD | $599,333,000 = $599.3M |
| XTS (Transactions) | Actual USD | $599,333,000 = $599.3M |

---

# 3. Plan/CC Cube (Financials Fact Table)

- **Table:** `fna.fna_gold.jedoxviewer_plan_cc_transformed` (`fact`)
- **Source:** Jedox (Consolidates CC Cube and Plan Cube)

## 3.1 Schema & Key Columns

| Column | Description | Notes |
|--------|-------------|-------|
| `signed` | Value in thousands | Multiply by 1000. **Exception:** Headcount is actual FTEs |
| `period_key` | `'YYYY-MM scenario'` | e.g., `'2025-03 actual'`, `'2025-12 budget'`, `'2025-06 2qr'` |
| `scenario` | Scenario name | `'Budget'`, `'Actual'`, `'1qr'`, `'2qr'`, `'3qr'` |
| `month` | Period month | e.g., `'2024-12'` |
| `entity_plan` | Join key | → `ou.entb` |
| `account_plan` | Join key | → `ac.element` |
| `customer` | Join key | → `cust.element` |
| `job_function` | Job function | `'AM_Non-QAQC'`, `'AM_QAQC & Tech'`, `'SPP_Non-QAQC'`, `'SPP_QAQC & Tech'`, `'NONE'` |
| `production_country_plan` | Production country | `'China'`, `'Vietnam'`, `'Bangladesh'`, etc. |
| `measure` | Measure dimension | See Section 3.2 |
| `currency` | Reporting currency | `'> USD'` (primary), `'> EUR'`, `'> GBP'` |

**YTD Balance Logic:** `SUBSTR(period_key, 1, 4) = 'YYYY' AND SUBSTR(period_key, 6, 2) <= 'MM'`

## 3.2 Measure Dimension

- **"Input"**: The core data source
  - _Actual Scenarios:_ Matches the ERP / Actual Cube
  - _Forecast Scenarios (Forecast Months):_ Planner inputs
  - _Forecast Scenarios (Actualized Months):_ Copy of Actuals

- **Other Measures** (Adjustments, Allocations):
  - Represent Management Adjustments, Reclassifications, or Allocations NOT in ERP
  - **Reconciliation Tip:** Discrepancies between Plan/CC Cube and Actual Cube are usually found here

**Full List:**
- **Core:** `'Input'`
- **Adjustments:** `'Adjustment'`, `'Adjustment 1'`, `'Topside'`, `'Topside Adjustment'`, `'Centric Adjustment'`, `'LastYr Adjustment'`
- **Allocations:** `'AM Allocation'`, `'SPP Allocation'`, `'Country Allocation'`, `'SCSSC Allocation'`, `'Fixed Customer Allocation'`, `'OSG % Allocation'`, `'OSG Direct Input'`, `'OSG Fixed Amount'`, `'Catchall Allocation'`, `'New Catchall Allocation'`, `'GTS Allocation'`, `'GTS Direct Input'`, `'TIC Allocation'`, `'New TIC Allocation'`, `'WCC % Allocation'`, `'VOICES OG Fixed %'`, `'VOICES PC %'`
- **Direct Inputs:** `'Corporate Overhead Direct Input'`, `'Rental Shortfall Direct Input'`
- **Reclassifications:** `'Remapping'`, `'MappingB'`, `'Phase1 AfterAlloc Reclass'`, `'Phase1 BeforeAlloc Reclass'`
- **Historical:** `'HP Historical'`, `'HP Historical Nullify'`

## 3.3 Historical Snapshots

- **Table (Full History):** `fna.fna_gold.jedoxviewer_plan_cc_data_whist_untrimmed`
- **Table (Trimmed for BI):** `fna.fna_gold.jedoxviewer_plan_cc_data_whist`

**CRITICAL: Delta Storage Model**
- The `value` column stores **deltas (changes)**, NOT absolute values
- **To get actual value at a point in time:** Sum all deltas from beginning up to that snapshot date

| Attribute | `_whist_untrimmed` | `_whist` (trimmed) |
|-----------|--------------------|--------------------|
| **Snapshot Count** | ~1,435 | ~56 |
| **History Start** | Sept 2023 | June 2025 |
| **Granularity** | Daily/near-daily | Weekly |
| **Recommended For** | Agent/programmatic queries | Human-facing BI dashboards |

---

# 4. Dimension Tables

## 4.1 Entity Dimension

- **Table:** `fna.fna_gold.jedoxviewer_entity` (`ou`)

### Key Columns

| Column | Description | Example |
|--------|-------------|---------|
| `entb` | Bottom level code (unique within hierarchy root) | `'C5-HK-MC5'` |
| `entbname` | Entity name | `'C5 Div'` |
| `ent1` | Top level (**Critical Filter**) | `'OG_Forecast'` |
| `ent2` / `ent2name` | Segment | `'SCS_FC'` / `'SCS'` |
| `ent3` / `ent3name` | Operating Group (OG) | `'Apparel_FC'` / `'Apparel'` |
| `ent4` / `ent4name` | Stream | `'ST:DSS_FC'` |
| `ent5` / `ent5name` | Product Group (PG) | `'PG:KAG_FC'` |
| `ent6` / `ent6name` | Division | `'DIV:C5_FC'` / `'C5 Div'` |
| `ent7` | Org Unit (OU) | `'C5-HK-MC5'` |
| `hb_name` | Homebase name | `'Hong Kong'` |
| `hb_region` | Homebase region | `'Greater China'`, `'Greater ASEAN'`, `'India Sub'`, `'LATAM'`, `'Rest of the World'` |
| `am_spp` | Classification | `'AM'` (Account Mgmt), `'SPP'` (Sourcing), `'Other'` |

### Hierarchy Values

**Hierarchy Roots (`ent1`):** `'OG_Forecast'` (primary), `'OG_Budget'`, `'OG_2024'`, `'OG_2023'`

**Segments (`ent2name`):**
- Business: `'SCS'`, `'Markets'`
- Non-Business: `'Corporate'`, `'Functions'`, `'Group'`, `'LFX'`, `'Logistics'`, `'Others'`

**Operating Groups (`ent3name`):**
- SCS: `'Apparel'`, `'Home And Accessories'`, `'SCS - Others'`, `'OG: SCS Share Centre'`
- Markets: `'OG: Firework'`, `'OG: LF Asia Direct'`, `'OG: LF Europe'`, `'OG: LF Markets USA'`, `'OG: Promocean and EU Others'`, `'Markets - Others'`

**Key Filters:**
- `ent1 = 'OG_Forecast'` (**Required**)
- **Business OGs Only:** `ou.ent2name IN ('SCS', 'Markets')`
- **Specific OG:** Use `ou.ent3 = 'OG:LFMU_FC'` (Note the `_FC` suffix)

### E1 Business Unit (BU) Codes

BU codes map to Entity dimension via `fna.fna_gold.jedox_dimpc_entity_detail`.

**BU Code Structure:**
- First 3 digits = **Company code** (legal entity)
- Remaining digits = **Business unit identifier**

| BU Code | Company | Entity (entb) |
|---------|---------|---------------|
| `001357908` | 00001 (LF Trading HK) | GTQA-GT-QA1 |
| `326357908` | 00326 (LF Guatemala) | GTQA-GT-QA1 |
| `001448301` | 00001 (LF Trading HK) | LM2-HK-MG1 |

**Lookup query:**
```sql
SELECT child AS bu_code, parent AS entb, description
FROM fna.fna_gold.jedox_dimpc_entity_detail
WHERE child LIKE '001%'  -- All BUs under company 001
  AND parent = 'AE1-HK-MC1';  -- Filter to specific entity
```

## 4.2 Account Dimension

- **Table:** `fna.fna_silver.jedox_dimfh_account_plan` (`ac`)

### Key Columns

| Column | Description |
|--------|-------------|
| `element` | Unique account code (e.g., `'P651101'`) |
| `account_description` | Human-readable name |
| `level1` to `level11` | Hierarchical codes |
| `account_description_1` to `account_description_11` | Descriptions for each level |

### Account Hierarchy Roots (`level2`)

- **Financial Statements:** `'PL'` (P&L), `'BS'` (Balance Sheet), `'WC'` (Working Capital), `'FCF'` (Free Cash Flow)
- **Headcount:** `'Headcount'`, `'Headcount (Average)'`
- **KPI Ratios:** `'Contribution 2 to Turnover %'`, `'Revenue to Turnover %'`, etc.

### P&L Hierarchy (level2 = 'PL')

```
level3 = 'C4MT' → Contribution 4 (Top-level P&L)
  └─ level4 = 'C3MT' → Contribution 3 (Net Profit proxy)
       └─ level5 = 'C2MT' → Contribution 2 (EBIT proxy - MOST USED)
            ├─ level6 = 'TMMT' → Total Revenue (Gross Margin)
            │    └─ level7 = 'GPMT' → Gross Profit
            │         └─ level8 = 'STD_GP' → Standard Gross Profit
            │              ├─ level9 = 'P651000MT' → Net Sales / TURNOVER
            │              └─ level9 = 'P710100MT' → COGS / FOB
            └─ level6 = 'O2MT' → Total OPEX
                 ├─ level7 = 'ODMMT' → Direct - MPC
                 ├─ level7 = 'ODNMT' → Direct - Non MPC
                 ├─ level7 = 'ODXMT' → Direct - Extraordinary
                 ├─ level7 = 'OOMT' → Indirect - OSG
                 └─ level7 = 'OIMT' → Indirect - Rent & Others
```

### Metric Mappings

| Metric | Filter | Notes |
|--------|--------|-------|
| **Turnover** | `ac.level9 = 'P651000MT'` | Net Sales |
| **Revenue** | `ac.level6 = 'TMMT'` | Includes COGS/Margin |
| **Total OPEX** | `ac.level6 = 'O2MT'` | Stored NEGATIVE, display POSITIVE |
| **Contribution 2 (C2)** | `ac.level5 = 'C2MT'` | EBIT proxy - most used |
| **Contribution 3 (C3)** | `ac.level4 = 'C3MT'` | Net Profit proxy |
| **Contribution 4 (C4)** | `ac.level3 = 'C4MT'` | Full P&L |
| **Headcount** | `ac.level2 = 'Headcount'` | Do NOT scale by 1000 |
| **Working Capital** | `ac.level2 = 'WC'` | Use level2 to avoid BS duplicates |

## 4.3 Customer Dimension

- **Table:** `fna.fna_silver.jedox_dimfh_customer` (`cust`)

| Column | Description | Example |
|--------|-------------|---------|
| `element` | Unique code | `'PCUST:ACTD'` |
| `level2` | Channel | `'Agency'`, `'TAP'`, `'Interco'`, `'Jeanswear'` |
| `level3` | Common Name (preferred for grouping) | `'Action'`, `'Walmart'` |
| `corp_customer` | Full corporate name | `'CORP: ACTION SERVICE & DISTRIBUTIE B.V. - ACTD'` |

**Note:** "Others - Agency" is a large aggregate bucket for smaller customers.

---

# 5. Balance Sheet & Working Capital

## 5.1 Data Convention

For Balance Sheet accounts, `fact.signed` represents the **change during that month**, not the closing balance.

**Period-End Balance = Sum from M00 through target month:**
```sql
WHERE SUBSTR(fact.period_key, 6, 2) <= 'MM'  -- Include M00 opening balance
```

## 5.2 Sign Convention

- **Assets** (Inventory, AR) → sum to **Positive**
- **Liabilities** (AP, Factoring) → sum to **Negative**

## 5.3 Working Capital Components

**Sign convention:** Assets = Positive, Liabilities = Negative

| Component | Jedox Code | Sign |
|-----------|------------|------|
| Accounts Receivable | `WC_AR` | + |
| Accounts Payable | `WC_AP` | − |
| Inventory | `B201000` | + |
| **= Gross Trade WC** | `WC_NET` | |
| + Other Receivables (Prepayments) | `B209000` | + |
| + Other Payables (Accruals) | `B305000` | − |
| **= Business WC** | | |
| + Factoring Drawdown | `B208000` | − |
| + LF Credit Settlement | `B304000` | − |
| **= Total Net Working Capital** | `level2 = 'WC'` | ← Query this |

**Query filter:** Always use `ac.level2 = 'WC'` for total WC.
Using `ac.level3 = 'WC_NET'` returns only Gross Trade WC (excludes OROP, Factoring, LF Credit).

---

# 6. E1 Detail & ERP Drill-Down

**IMPORTANT:** Do NOT write E1 queries based on examples in this file. Always consult **[E1_DETAIL.md](./E1_DETAIL.md)** first for:
- Complete table schemas and column definitions
- Entity/Account mapping tables
- Text search strategies (3-field search pattern)
- Document type filters (PT, RC, JE, etc.)
- Reconciliation patterns

Key capabilities:
- Trace Plan/CC Cube balances back to individual E1 journal entries
- Identify vendors, counterparties, and legal entities
- Reconcile Plan/CC Cube (Input measure) to E1 Detail (~0.01% variance expected)
- Search transaction text fields for specific vendors or services

---

# 7. Data Quality & Exclusions

## 7.1 Entities to Be Aware Of

**Management Adjustment Entities:**
- `PMADJ-*` (PC Management Account Adjustments)
- `MAS*-HK-OT1` (Management AC Adj entities)
- `*GP-HK-SC2` (Profit Sharing Adjustments)

**Recommendation:** For operational analysis, exclude:
```sql
WHERE ou.entb NOT LIKE 'PMADJ%'
  AND ou.entb NOT LIKE 'MAS%'
  AND ou.entbname NOT LIKE '%Adj%'
```

## 7.2 OGs with Special Handling

| OG | Notes |
|----|-------|
| `SCS - Others` | Catch-all for unmapped SCS items |
| `Markets - Others` | Catch-all; includes Topside adjustments |
| `OG: SCS Share Centre` | Shared services; typically zero turnover |
| `OG: LF Asia Direct` | Smaller OG; may have negative C2 |

## 7.3 Customer Data Quality

- **`NULL` customer:** Unallocated transactions (~$96M in 2024)
- **`Others - Agency`:** Aggregate bucket (~$184M in 2024)
- **Customer names vary:** Check both `level3` and `corp_customer`

## 7.4 Customer-Level Profitability Reliability

**CRITICAL:** Customer-level C2 reliability varies dramatically by segment:

| Segment | OGs | Customer C2 | Guidance |
|---------|-----|-------------|----------|
| **SCS** | Apparel, H&A | **>90% reliable** | Trust for customer profitability analysis |
| **Markets** | LFEU, LFMU, Firework, Promocean, LFAD | **Do not use** | Stick to OG/Division level only |

**Why the difference:**
- SCS has mature allocation rules refined since 2023
- Markets operates with diverse geography, business models, legal entities, and ERP systems
- Allocation rules have not been prioritized for Markets' complexity

**When analyzing Markets customer profitability:** Redirect to OG-level analysis or caveat heavily that customer-level numbers are unreliable.

## 7.5 Period Considerations

- **Period '00':** Opening balance (for BS/WC only, not P&L)
- **Forecast months beyond actuals:** Planner inputs
- **Actualized months in forecasts:** Copy of actual values

---

# 8. Business Context & Benchmarks

## 8.1 Business Model Overview

- **SCS (Supply Chain Services):** Apparel + Home & Accessories - high volume, low margin (~4% revenue margin)
- **Markets:** LF Europe, LF Markets USA, Promocean, Firework, LF Asia Direct - lower volume, higher margin
- **Agency:** LF acts on behalf of buyer (most SCS business)
- **Principal:** LF takes ownership of goods (some Markets business)
- **Intercompany Agency:** Agency sales to LF entities - treated as "Imputed Principal" at group level

**Revenue Margin by Business Model:**
Margin spread reflects business model, not performance. Compare within model, not across:

| Business Model | Typical OGs | Rev Margin | How It Works |
|----------------|-------------|------------|--------------|
| Agency Sourcing | Apparel, H&A | ~4% | Commission on FOB |
| Wholesale/Principal | LFEU, LFMU, Promocean | ~11-16% | Buy & resell at markup |
| Branded Retail | Firework | ~35% | Own brand, consumer products |

Comparing Firework's 35% margin to SCS's 4% is apples-to-oranges - they are fundamentally different businesses.

## 8.2 Historical Performance (Business OGs: SCS + Markets)

**Typical ranges (for sanity-checking):**
- Revenue margin typically 5.5-6.5% of Turnover
- C2 margin typically 1.9-2.1% of Turnover

Query historical data using SQL patterns in Appendix B when specific numbers needed.

## 8.3 Seasonality Patterns

**Peak months:** June, August, December
**Trough:** February (Chinese New Year impact)
**H1 vs H2:** Relatively balanced, with slight H2 skew

Query historical data to get specific percentages when needed.

## 8.4 OG Performance Benchmarks

**Typical margin ranges by business model (for sanity-checking):**
| Business Model | Typical OGs | Rev Margin | C2 Margin |
|----------------|-------------|------------|-----------|
| Agency Sourcing | Apparel, H&A | 3-5% | 1-2% |
| Wholesale/Principal | LFEU, LFMU | 10-16% | 2-7% |
| Branded Retail | Firework | 30-40% | 10-15% |

## 8.5 Top Customers

Top customers and concentration metrics change year-to-year. Query using SQL pattern B.2 with `ORDER BY turnover DESC`.

## 8.6 Sourcing Geography

Sourcing geography evolves as supply chain diversifies. Query `production_country_plan` dimension for current country mix.

## 8.7 Forecast Scenarios & Timing

| Scenario | Code | Timing | Use Case |
|----------|------|--------|----------|
| Budget | `budget` | Set in Q4 prior year | Annual target baseline |
| 1QR | `1qr` | ~April | First reforecast after Q1 actuals |
| 2QR | `2qr` | ~July | Mid-year reforecast |
| 3QR | `3qr` | ~October | Final reforecast before year-end |
| Actual | `actual` | Monthly close | Realized performance |

**Default Forecast Comparison by Period:**
- **Jan-Mar** → Compare to Budget
- **Apr-Jun** → Compare to 1QR
- **Jul-Sep** → Compare to 2QR
- **Oct-Dec** → Compare to 3QR

**Budget Phasing Bias:**
Budget is set in Q4 of prior year when planners must split annual targets into 12 months. There's a behavioral pattern:
- **H1 tends to beat budget** (planners are conservative on near-term uncertainty)
- **H2 tends to miss budget** (optimism backloads volume to "catch up")
- **Full year often nets out** close to target

**Implication:** When analyzing monthly variance vs budget, first ask "Is this phasing or actual business variance?" Monthly misses may be process artifacts, not performance signals. Full-year variance is more meaningful than individual months.

---

# 9. Standard Analysis Workflows

## 9.1 Standard P&L Report Structure

Financial reports follow this sequence:

1. **Turnover** (Volume)
2. **Revenue** (Gross Margin $) + _Margin %_
3. **Total OPEX** (with 5-component breakdown)
4. **Contribution 2 (C2)** (EBIT proxy) + _C2 Margin %_
5. **Contribution 3 (C3)** (Net Profit proxy)
6. **Free Cash Flow**

**Variance Completeness Rule:** When comparing performance, always analyze ALL four core metrics: Turnover, Revenue, OPEX, C2.

## 9.2 Variance Analysis Priorities

1. **C2 Variance:** The "Bottom Line" impact
2. **Turnover Variance:** Volume impact
3. **Revenue Variance:** Margin/Mix impact
4. **OPEX Variance:** Efficiency/Cost Control impact
5. **FCF/WC Variance:** Cash impact

## 9.3 Common Analysis Patterns

### P&L Performance Review
1. Query key metrics (turnover, revenue, opex, c2)
2. Compare actual vs forecast/budget
3. Drill into variances by OG, customer, or month

### Forward Demand Analysis
1. Get current OOH from ORDERBOOK.md
2. Calculate fill rate vs target
3. Compare to same week last year

### Qualitative Deep-Dive
1. Identify entity/stream of interest from financials
2. Query XTS.md for production country mix, supplier breakdown
3. Analyze product category and customer channel distribution

---

# 10. Related Data Sources

## 10.1 Orderbook Data (Forward-Looking)

See **[ORDERBOOK.md](./ORDERBOOK.md)** for fill rate tracking, OOH trends.

| Aspect | FINANCIALS | ORDERBOOK |
|--------|------------|-----------|
| Time Horizon | Historical | Forward-looking |
| Value Scaling | Thousands (x1000) | Actual USD |
| Lead Time | N/A | ~100 days (SCS) |

## 10.2 XTS Transaction Data

See **[XTS.md](./XTS.md)** for production country mix, supplier breakdown, product categories.

## 10.3 OG Name Mapping (Financials ↔ Orderbook)

| FINANCIALS (`ent3name`) | ORDERBOOK (`REPORT_OPERATING_GROUP_DESC`) |
|------------------------|------------------------------------------|
| Apparel | Apparel |
| Home And Accessories | Home And Accessories |
| OG: LF Markets USA | LF Markets USA |
| OG: LF Europe | LF Europe |

---

# 11. Glossary

| Acronym | Full Name | Description |
|---------|-----------|-------------|
| **OG** | Operating Group | Level 3 in entity hierarchy |
| **OU** | Org Unit | Lowest level entity |
| **SCS** | Supply Chain Solutions | Segment: Apparel + H&A |
| **MPC** | Manpower Costs | ~60% of OPEX; direct staff costs |
| **OSG** | Operations Support Group | Indirect overhead (FNA, HR, IT, CS) |
| **C2** | Contribution 2 | Revenue - OPEX; **EBIT proxy** |
| **C3** | Contribution 3 | C2 - Financial items |
| **TO** | Turnover | Gross sales volume |
| **QR** | Quarterly Reforecast | 1QR, 2QR, 3QR |
| **YTD** | Year To Date | Cumulative from Jan |
| **AM** | Account Management | Customer-facing roles |
| **SPP** | Sourcing/Production | Supply-side roles |
| **FOB** | Free On Board | Product cost basis (COGS for agency business) |
| **Commission** | Agency Commission | Turnover - FOB; the agency fee earned |
| **Commission %** | Commission Rate | Commission / FOB; typically 2-4% for SCS agency |
| **AR** | Accounts Receivable | Customer balances owed |
| **AP** | Accounts Payable | Supplier balances owed |
| **WC** | Working Capital | AR + Inventory - AP |
| **FCF** | Free Cash Flow | Operating cash generation |

---

# Appendix A: Notes & Gotchas

**This is an append-only section.** Add new discoveries, tips, and gotchas here. Periodically review and promote important patterns to structured sections above.

---

## General Query Tips

- **Use the SQL CLI tool** for running queries: `uv run python -m ba_bot.cli.sql "SELECT ..."`
  - Outputs JSON directly, cleaner than inline Python with `run_sql()`
  - Supports file input: `uv run python -m ba_bot.cli.sql -f query.sql`
  - Supports stdin: `echo "SELECT ..." | uv run python -m ba_bot.cli.sql -`
- "OG" refers to the `ent3` column, not `ent2name`
- When asking for metrics for a specific time period (e.g., "YTD Oct 20X5"), default to 'actual' scenario unless specified
- When user asks for forecast scenarios (e.g., "3QR") without qualifying a period, interpret as "Full Year"
- When encountering unfamiliar terms, first search the dimension tables to identify correct mapping
- When checking for specific names (e.g., customers), check multiple hierarchy levels (`level3` AND `corp_customer`)

## FY-to-FY Scenario Selection

When comparing full-year performance across multiple years, use the most meaningful scenario for each year:

| Year Type | Scenario to Use | Rationale |
|-----------|-----------------|-----------|
| **Closed/historical years** | Actual | Final audited numbers |
| **Current year** | Latest forecast (3QR > 2QR > 1QR > Budget) | Best full-year estimate |
| **Future years** | Budget | Only available scenario |

**Why this matters:** Using partial-year actuals for current year creates misleading YoY comparisons. Latest forecast gives a complete full-year view that's comparable to historical actuals.

**Example:** For a query in Nov 2025 asking "Turnover by year for 2023-2026":
- 2023, 2024: Actual
- 2025: 3QR (latest forecast)
- 2026: Budget

## Agency vs Principal Classification

**Do NOT use `cust.level2`** from customer dimension (values like `'Agency'`, `'TAP'`) - this is a static heuristic that is rarely accurate. Instead, use XTS data (`BUSINESS_NATURE` column: `'A'` = Agency, `'P'` = Principal) which captures actual transaction-level business nature.

**Why `cust.level2` is unreliable:** It assumes one customer = one business model. In practice, some customers (e.g., ASDA) operate mixed Agency + Principal models. The static `level2` value cannot capture this transaction-level variation.

## Working Capital Specific

- Always filter `ac.level2 = 'WC'` to avoid double-counting (some elements exist in both 'BS' and 'WC' hierarchies).
- For reliable customer-level Plan/CC working capital, filter `fact.measure = 'Input'`; allocation measures such as `MappingB` can materially distort customer signs and balances.

## Tracking Performance vs Plan

- Compare actuals against **phased Budget/Forecast scenarios** (1QR, 2QR, 3QR), not just YoY
- Forecasts are phased by month, so YTD comparisons reflect planned seasonality
- For seasonal businesses, YoY alone can be misleading

## Seasonal Business: Firework

- **Firework (OG: Firework)** is highly seasonal consumer fireworks:
  - **UK (EONBC/BCUK):** Peak Oct-Nov (Guy Fawkes + NYE)
  - **Germany (EONCM/Comet):** Peak December (Silvester)
  - Business model: Build inventory Jan-Sep, sell Oct-Dec
  - Expect negative C2 most of year, profits in Q4
  - Large returns/chargebacks post-season

## Inventory to Sales Relationship

- Inventory is recorded at **cost**, not selling price
- To estimate sales capacity: `Potential Turnover = Inventory at Cost / (1 - GP%)`
- Example: Firework GP ~46-47%, so $50M inventory ≈ $94M potential turnover

## Entity Restructures

- Watch for entity migrations (old entities zeroed, new ones appear)
- Reversals in one entity offset by new bookings in another may indicate restructure, not operational change
- When analyzing by entity, check for new entities that didn't exist in prior periods

## Agency Commission Calculation

**For agency business (most of SCS), commission is the markup on FOB:**

- **Accounting:** `Dr FOB 100 / Cr Turnover 103` → Commission = 3
- **Formula:** `Commission % = (Turnover - ABS(FOB)) / ABS(FOB) * 100`
- **Typical SCS range:** 2-4% (large customers lower, smaller customers higher)
- **Fixed vs Variable:** Monthly trend reveals if customer has fixed contractual rate (identical %) or variable (product mix driven)

---

# Appendix B: SQL Snippets

**This is an append-only section.** Add useful SQL patterns here. Label clearly.

---

## B.1 Basic C2 Performance (Actual vs Budget)

```sql
SELECT
    SUM(CASE WHEN SUBSTR(fact.period_key, 9) = 'actual' THEN fact.signed ELSE 0 END) * 1000 AS Actual_USD,
    SUM(CASE WHEN SUBSTR(fact.period_key, 9) = 'budget' THEN fact.signed ELSE 0 END) * 1000 AS Budget_USD,
    (SUM(CASE WHEN SUBSTR(fact.period_key, 9) = 'actual' THEN fact.signed ELSE 0 END) -
     SUM(CASE WHEN SUBSTR(fact.period_key, 9) = 'budget' THEN fact.signed ELSE 0 END)) * 1000 AS Variance_USD
FROM fna.fna_gold.jedoxviewer_plan_cc_transformed AS fact
JOIN fna.fna_silver.jedox_dimfh_account_plan AS ac ON fact.account_plan = ac.element
JOIN fna.fna_gold.jedoxviewer_entity AS ou ON fact.entity_plan = ou.entb
WHERE ou.ent1 = 'OG_Forecast'
  AND fact.currency = '> USD'
  AND ac.level5 = 'C2MT'
  AND ou.ent3 = 'OG:LFMU_FC'
  AND SUBSTR(fact.period_key, 1, 4) = 'YYYY'
  AND SUBSTR(fact.period_key, 9) IN ('actual', 'budget');
```

## B.2 Top Customer Drivers (Turnover Variance)

```sql
SELECT
    cust.level3 AS Customer,
    (SUM(CASE WHEN SUBSTR(fact.period_key, 9) = 'actual' THEN fact.signed ELSE 0 END) -
     SUM(CASE WHEN SUBSTR(fact.period_key, 9) = 'budget' THEN fact.signed ELSE 0 END)) * 1000 AS Variance_USD
FROM fna.fna_gold.jedoxviewer_plan_cc_transformed AS fact
JOIN fna.fna_gold.jedoxviewer_entity AS ou ON fact.entity_plan = ou.entb
JOIN fna.fna_silver.jedox_dimfh_account_plan AS ac ON fact.account_plan = ac.element
JOIN fna.fna_silver.jedox_dimfh_customer AS cust ON fact.customer = cust.element
WHERE ou.ent1 = 'OG_Forecast'
  AND fact.currency = '> USD'
  AND ac.level9 = 'P651000MT'
  AND ou.ent3 = 'OG:LFMU_FC'
  AND SUBSTR(fact.period_key, 1, 4) = 'YYYY'
  AND SUBSTR(fact.period_key, 9) IN ('actual', 'budget')
GROUP BY cust.level3
ORDER BY Variance_USD ASC
LIMIT 10;
```

## B.3 Cross-Year Scenario Comparison

For cross-year comparisons:

```sql
-- Compare next year Budget vs current year 3QR for H&A
-- Replace YYYY with current year, ZZZZ with next year
SELECT
    SUM(CASE WHEN SUBSTR(fact.period_key, 1, 4) = 'ZZZZ'
             AND SUBSTR(fact.period_key, 9) = 'budget'
        THEN fact.signed ELSE 0 END) * 1000 AS Budget_NextYr,
    SUM(CASE WHEN SUBSTR(fact.period_key, 1, 4) = 'YYYY'
             AND SUBSTR(fact.period_key, 9) = '3qr'
        THEN fact.signed ELSE 0 END) * 1000 AS QR3_CurrYr
FROM fna.fna_gold.jedoxviewer_plan_cc_transformed AS fact
JOIN fna.fna_silver.jedox_dimfh_account_plan AS ac ON fact.account_plan = ac.element
JOIN fna.fna_gold.jedoxviewer_entity AS ou ON fact.entity_plan = ou.entb
WHERE ou.ent1 = 'OG_Forecast'
  AND fact.currency = '> USD'
  AND ou.ent3name = 'Home And Accessories'
  AND ac.level5 = 'C2MT'
  AND SUBSTR(fact.period_key, 1, 4) IN ('YYYY', 'ZZZZ')
  AND SUBSTR(fact.period_key, 9) IN ('budget', '3qr');
```

## B.4 Customer Multi-Code Matching

Major customers often have multiple codes:
- `PCUST:KOHL` - Main code
- `PCUST:KOHI` - Alternate
- `PCUST:MKO` - Markets
- `Jeanswear - Kohl's` - Channel

**Always search broadly:**
```sql
SELECT DISTINCT element, level3, level2, corp_customer
FROM fna.fna_silver.jedox_dimfh_customer
WHERE LOWER(level3) LIKE '%kohl%'
   OR LOWER(corp_customer) LIKE '%kohl%'
   OR LOWER(element) LIKE '%kohl%';
```

## B.5 YoY Turnover by OG

```sql
SELECT
    ou.ent3name AS OG,
    SUM(CASE WHEN SUBSTR(fact.period_key, 1, 4) = '2024' THEN fact.signed ELSE 0 END) * 1000 AS TY,
    SUM(CASE WHEN SUBSTR(fact.period_key, 1, 4) = '2023' THEN fact.signed ELSE 0 END) * 1000 AS LY,
    (SUM(CASE WHEN SUBSTR(fact.period_key, 1, 4) = '2024' THEN fact.signed ELSE 0 END) -
     SUM(CASE WHEN SUBSTR(fact.period_key, 1, 4) = '2023' THEN fact.signed ELSE 0 END)) * 1000 AS Variance
FROM fna.fna_gold.jedoxviewer_plan_cc_transformed AS fact
JOIN fna.fna_silver.jedox_dimfh_account_plan AS ac ON fact.account_plan = ac.element
JOIN fna.fna_gold.jedoxviewer_entity AS ou ON fact.entity_plan = ou.entb
WHERE ou.ent1 = 'OG_Forecast'
  AND ou.ent2name IN ('SCS', 'Markets')
  AND fact.currency = '> USD'
  AND ac.level9 = 'P651000MT'
  AND SUBSTR(fact.period_key, 9) = 'actual'
  AND SUBSTR(fact.period_key, 6, 2) <= '10'
GROUP BY ou.ent3name
ORDER BY Variance DESC;
```

## B.6 Monthly C2 Trend

```sql
SELECT
    SUBSTR(fact.period_key, 6, 2) AS Month,
    SUM(fact.signed) * 1000 AS C2,
    LAG(SUM(fact.signed) * 1000) OVER (ORDER BY SUBSTR(fact.period_key, 6, 2)) AS Prev_Month
FROM fna.fna_gold.jedoxviewer_plan_cc_transformed AS fact
JOIN fna.fna_silver.jedox_dimfh_account_plan AS ac ON fact.account_plan = ac.element
JOIN fna.fna_gold.jedoxviewer_entity AS ou ON fact.entity_plan = ou.entb
WHERE ou.ent1 = 'OG_Forecast'
  AND ou.ent2name IN ('SCS', 'Markets')
  AND fact.currency = '> USD'
  AND ac.level5 = 'C2MT'
  AND SUBSTR(fact.period_key, 9) = 'actual'
  AND SUBSTR(fact.period_key, 1, 4) = 'YYYY'
GROUP BY SUBSTR(fact.period_key, 6, 2)
ORDER BY Month;
```

## B.7 Historical Snapshot Cumulative Query

```sql
WITH cumulative AS (
  SELECT
    DATE(h.as_of) as snap_date,
    SUM(h.value) * 1000 as daily_delta
  FROM fna.fna_gold.jedoxviewer_plan_cc_data_whist_untrimmed h
  JOIN fna.fna_gold.jedoxviewer_entity ou
    ON h.entity_plan = ou.entb AND ou.ent1 = 'OG_Forecast'
  JOIN fna.fna_silver.jedox_dimfh_account_plan ac
    ON h.account_plan = ac.element
  WHERE h.scenario = 'Budget'
    AND h.currency = '> USD'
    AND SUBSTR(h.month, 1, 4) = 'YYYY'
    AND ou.ent3 = 'OG:LFEU_FC'
    AND ac.level9 = 'P651000MT'
  GROUP BY DATE(h.as_of)
)
SELECT
  CAST(c.snap_date AS STRING) as snapshot_date,
  ROUND(SUM(c2.daily_delta), 0) as running_total
FROM cumulative c
JOIN cumulative c2 ON c2.snap_date <= c.snap_date
GROUP BY c.snap_date
ORDER BY c.snap_date;
```

---

*End of Document*

# E1 Journal Details & ERP Drill-Down

Drill down from Plan/CC Cube to individual E1 (ERP) journal lines for transaction-level investigation, vendor analysis, and reconciliation.

**Key Use Cases:** Trace aggregated values to source transactions | Identify vendors/counterparties | Investigate variances | Reconcile Plan/CC to ERP

**Relationship to [FINANCIALS.md](./FINANCIALS.md):** E1 Detail matches Plan/CC Cube `measure = 'Input'` within ~0.01% (FX rounding). Differences = Management Adjustments (not in E1).

---

## 1. The Three Accounting Layers

E1 data exists in three distinct layers. Selecting the wrong layer leads to incorrect answers or meaningless sums.

### Mental Model

```
┌─────────────────────────────────────────────────────────────────────────┐
│  LAYER 1: ACCRUAL (P&L Recognition)                                     │
│  Accounts: 5xxx (Revenue/COGS), 6xxx (OpEx), 7xxx (Staff), 8xxx (Other) │
│  Question: "How much economic value was recognized this period?"        │
│  Doc Types: UX, VX, VM, JE                                              │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  LAYER 2: INVOICE (Liability/Receivable Creation)                       │
│  Accounts: 2004 (AR), 3003 (AP), 3005 (Accruals)                        │
│  Question: "What legal obligations exist to pay/collect?"               │
│  Doc Types: VL, VM, NO, PN                                              │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  LAYER 3: CASH (Bank Movement)                                          │
│  Accounts: 22xx (Bank accounts)                                         │
│  Question: "What actual money moved?"                                   │
│  Doc Types: PT, RC, PK                                                  │
│  ** HIGHEST RELIABILITY - no estimates, no reversals **                 │
└─────────────────────────────────────────────────────────────────────────┘
```

### Layer Details

| Layer | Accounts | Key Doc Types | Sign Convention | Use When |
|-------|----------|---------------|-----------------|----------|
| **Accrual** | `5xxx`, `6xxx`, `7xxx`, `8xxx` | `UX`, `VX`, `VM`, `JE` | Revenue = Negative, Expense = Positive | "How much did we recognize?" |
| **Invoice** | `2004`, `2005`, `3003`, `3005` | `VL`, `VM`, `NO` | AR = Positive, AP = Negative | "What do we owe / are owed?" |
| **Cash** | `22xx` | `PT`, `RC`, `PK` | Cash in = Positive, Cash out = Negative | "What cash actually moved?" |

### Decision Tree

```
"How much did we spend on vendor X?"
├── Want P&L expense recognized? → Layer 1 (5xxx-8xxx accounts)
├── Want invoiced amount owed? → Layer 2 (3003 AP)
└── Want cash actually paid? → Layer 3 (PT docs or 22xx accounts) ← Most reliable

"How much revenue from customer X?"
├── Want recognized revenue? → Layer 1 (5001 account, UX docs)
├── Want AR balance outstanding? → Layer 2 (2004 account)
└── Want cash collected? → Layer 3 (RC documents) ← Most reliable
```

### Why Layers Matter: Common Divergence Examples

| Scenario | Accrual (P&L) | Invoice (AP/AR) | Cash |
|----------|---------------|-----------------|------|
| **Service received, not yet invoiced** | Expense accrued | Nothing | Nothing |
| **Invoice received, not yet paid** | Expense recognized | AP created | Nothing |
| **Payment made** | No change | AP cleared | Cash out |
| **Tax refund eligibility** | May show estimate | Nothing | Actual check received |

---

## 2. Double-Entry Mechanics & Common Traps

### 2.1 Debit/Credit Sign Convention

E1 uses standard accounting signs (opposite of "intuitive" P&L thinking):

| Account Type | Debit (Dr) | Credit (Cr) | `AMOUNT_USD` Sign |
|--------------|------------|-------------|-------------------|
| **Asset** (Cash, AR, Inventory) | Increase | Decrease | Dr = Positive |
| **Liability** (AP, Loans) | Decrease | Increase | Cr = Negative |
| **Revenue** (Sales) | Decrease | Increase | Cr = Negative |
| **Expense** (OPEX, COGS) | Increase | Decrease | Dr = Positive |

**Key insight:** `AMOUNT_USD > 0` = Debit, `AMOUNT_USD < 0` = Credit.

**Common confusion:**
- A **sale** shows as **negative** (credit to revenue)
- A **payment to vendor** shows as **negative** (credit to bank)
- An **expense** shows as **positive** (debit to expense account)

### 2.2 The Double-Entry Trap: Netting to Zero

Every journal entry balances to zero. When querying by counterparty across all accounts, DR and CR sides cancel:

```sql
-- WRONG: Returns ~$0 (both sides of entries cancel)
SELECT SUM(AMOUNT_USD) FROM jedoxviewer_e1_details 
WHERE CP_ALPHA_NAME LIKE '%VENDOR%';

-- CORRECT: Filter to one layer
-- Option A: Filter by account class (expense accounts only)
SELECT SUM(AMOUNT_USD) FROM jedoxviewer_e1_details 
WHERE CP_ALPHA_NAME LIKE '%VENDOR%' AND ObjSub LIKE '6%';

-- Option B: Filter by document type (cash payments only)
SELECT SUM(AMOUNT_USD) FROM jedoxviewer_e1_details 
WHERE CP_ALPHA_NAME LIKE '%VENDOR%' AND DOCUMENT_TYPE = 'PT';
```

### 2.3 Suspense-Clearing Pattern (Double-Counting Trap)

Many transactions flow through a **suspense account** before final clearing. This creates two entries for the same economic event:

```
Step 1: Invoice received (VL document)
  Dr Expense (6xxx)     +10,000
  Cr AP Suspense (3099) -10,000

Step 2: Invoice cleared to vendor (AE document)  
  Dr AP Suspense (3099) +10,000
  Cr Trade AP (3003)    -10,000
```

**The trap:** Querying "expense by vendor" across ALL accounts counts both the expense entry AND the suspense clearing = **$20,000 overstated**.

**Solutions:**
1. Filter to leaf accounts only (exclude suspense `xx99`, `xx00`)
2. Filter by account class (`6xxx`, `7xxx` for expenses)
3. Use document type filters

### 2.4 Accrual Reversals (JR/JC Documents)

`JR` (Invoice Accrual) and `JC` (Case Accrual) are **auto-reversing**:

```
Month 1: Accrue expense (JR)
  Dr Expense +10,000 / Cr Accrual AP -10,000

Month 2: Auto-reversal (JR with opposite signs)
  Dr Accrual AP +10,000 / Cr Expense -10,000

Month 2: Actual invoice (VL/VM)
  Dr Expense +10,000 / Cr Trade AP -10,000
```

**The trap:** Multi-month query without understanding reversals:
- Month 1: +10,000 (accrual)
- Month 2: -10,000 (reversal) + 10,000 (actual) = +10,000
- **Total: +20,000** (but real expense is only +10,000)

**Solutions:**
1. Exclude reversing doc types for trend analysis: `WHERE DOCUMENT_TYPE NOT IN ('JR', 'JC')`
2. Query single months (reversals net out within period)
3. Focus on final-state documents: `VL`, `VM`, `PT`

**Verified behavior:** JR and JC net to exactly $0 across a full year (confirmed in 2024 data).

### 2.5 Intercompany Elimination

IC transactions appear on both sides (seller AND buyer entity):

```
Entity A sells to Entity B:
  Entity A: Cr IC Revenue (5002)  -100,000
  Entity B: Dr IC Expense (5202)  +100,000
```

**The trap:** Group-wide totals without IC elimination double-count.

**Solutions:**
1. Filter by single company: `WHERE COMPANY = '00001'`
2. Exclude IC accounts: `WHERE ObjSub NOT IN ('5002', '5202')`
3. Use Plan/CC Cube for consolidated view (IC already eliminated)

### 2.6 Quick Reference: Which Layer for Which Question

| Question | Filter Strategy | Avoid |
|----------|-----------------|-------|
| "How much did we pay vendor X?" | `DOCUMENT_TYPE = 'PT'` | Summing all entries |
| "What's our expense with vendor X?" | `ObjSub LIKE '6%' OR '7%'` | Including suspense accounts |
| "What invoices are pending?" | `DOCUMENT_TYPE IN ('VL', 'VM')` | Including JR/JC accruals |
| "Cash received from customer X?" | `DOCUMENT_TYPE = 'RC'` | Summing all entries |
| "External sales only?" | `ObjSub = '5001'` | Including IC (5002) |
| "Clean expense trend?" | `DOCUMENT_TYPE NOT IN ('JR', 'JC')` | Mixing accruals with actuals |

---

## 3. Primary Table: `fna.fna_gold.jedoxviewer_e1_details`

**Data Range:** 2021-01 to present (~107M rows)

| Column | Type | Description | Example |
|--------|------|-------------|---------|
| `ObjSub` | string | ERP object account code | `5001` (Trading Sales) |
| `COMPANY` | string | Legal entity code | `00001` |
| `BU` | string | Business Unit (maps to Plan/CC entity) | `001034502` |
| `GL_DATE` | date | Journal posting date | 2025-10-15 |
| `BATCH_NUMBER` | int | Journal batch identifier | 31058601 |
| `DOC_NUMBER` | int | Document number within batch | 25000223 |
| `LINE_NUMBER` | int | Line number within document | 1 |
| `AMOUNT_BASE` | decimal | Amount in local currency | -9965.31 |
| `AMOUNT_USD` | decimal | Amount in USD (unscaled) | -1284.19 |
| `BASE_CUR_CODE` | string | Local currency code | HKD |
| `CP_ALPHA_NAME` | string | Counterparty/Vendor name | PROGRESS MFG GROUP |
| `YEAR_MONTH` | int | Period in YYYYMM format | 202510 |
| `NAME_ALPHA_EXPLANATION` | string | Journal description | Offset By Document... |
| `NAME_REMARK_EXPLANATION` | string | Remarks/notes field | Openrouter-AI Tools |
| `REFERENCE2` | string | Reference number | 01375719 |
| `CUST_CODE` | string | Customer code (if applicable) | PCUST:1234 |
| `DOCUMENT_TYPE` | string | JDE document type code | `PT` (Payment) |

### ERP Account Classes (ObjSub Prefix)

| Prefix | Class | Examples | Layer |
|--------|-------|----------|-------|
| `1xxx` | Fixed Assets | Equipment, Buildings | - |
| `2xxx` | Current Assets | Bank (`22xx`), AR (`2004`), Inventory (`2010`) | Cash / Invoice |
| `3xxx` | Liabilities | AP (`3003`), Accruals (`3005`) | Invoice |
| `4xxx` | Equity/Reserves | Retained earnings | - |
| `5xxx` | Revenue & COGS | Sales (`5001`), Purchases (`5201`) | Accrual |
| `6xxx` | Operating Expenses | Travel (`6102`), Rent (`6110`) | Accrual |
| `7xxx` | Staff Costs | Salary (`7001`), Benefits (`7303`) | Accrual |
| `8xxx` | Other Income/Expense | Interest (`8210`), FX (`8206`) | Accrual |
| `9xxx` | Suspense/Control | Clearing (`9900`) | None - avoid |

### Key Account Codes

| Code | Description | Management Account | Layer |
|------|-------------|-------------------|-------|
| `5001` | Trading Sales (External) | P651000MT (Turnover) | Accrual |
| `5002` | **Inter-Co Trading Sales** | P651000MT (Turnover) | Accrual |
| `5201` | Trading Purchase (COGS) | P710100MT | Accrual |
| `2004` | Trade Debtor Control | WC_AR (AR) | Invoice |
| `3003` | Trade Creditor Control | WC_AP (AP) | Invoice |
| `22xx` | Bank Accounts | Various | Cash |
| `2010` | Stock - Finished Goods | B201000 (Inventory) | - |

---

## 4. Document Types by Layer

### Layer 1: Accrual (P&L Recognition)

| Code | Name | Description | Volume |
|------|------|-------------|--------|
| `UX` | Sales/Purchase | Core trading - sales invoices, POs | 20.5M |
| `VX` | Voucher Expense | Non-PO expense vouchers | 2.3M |
| `VM` | Voucher Match | Invoice matched to PO (3-way match) | 1.5M |
| `JE` | Journal Entry | Manual adjustments, accruals | 5.0M |
| `OV` | Order Voucher | Inventory voucher (Firework) | 2.4M |

### Layer 2: Invoice (AP/AR)

| Code | Name | Description | Volume |
|------|------|-------------|--------|
| `VL` | Voucher Logged | Invoice logged, pending payment | 1.9M |
| `VM` | Voucher Match | Creates AP when matched | 1.5M |
| `NO` | Netting | AR/AP offset | 46K |
| `PN` | Promissory Note | Payment commitment | 47K |
| `UM` | AR Memo | Debit/credit memo | ~200K |
| `UC` | AR Credit | Credit memo | ~26K |

### Layer 3: Cash (Bank)

| Code | Name | Description | Volume |
|------|------|-------------|--------|
| `PT` | Payment | Cash payment to vendor | 174K |
| `RC` | Receipt | Cash receipt from customer | 524K |
| `PK` | Payment Check | Physical check payment | ~4K |
| `PG` | Payment Group | Grouped/netted payment | - |

### System/Clearing (Avoid in Analysis)

| Code | Name | Description | Notes |
|------|------|-------------|-------|
| `AE` | Auto Entry | System offset entries | 5.4M - these are "other halves" |
| `JR` | Invoice Accrual | **Reversing** - invoice pending | Net to $0 over time |
| `JC` | Case Accrual | **Reversing** - no invoice yet | Net to $0 over time |
| `JX` | Journal FX | FX revaluation | Revalues BS accounts |
| `JI` | Journal Interco | Intercompany entries | - |
| `JA` | Journal Assign | Factoring AR assignment | - |

---

## 5. Entity & Account Mapping

### Entity Mapping (BU → Management Entity)

```
jedoxviewer_entity (ou)              -- Management hierarchy
    └── ou.entb = dimpc.parent
            └── jedox_dimpc_entity_detail (dimpc)
                    └── dimpc.child = BU code (e.g., '001034502')
```

**Table:** `fna.fna_gold.jedox_dimpc_entity_detail`

```sql
-- Get BU codes for an entity
SELECT child AS e1_bu_code 
FROM fna.fna_gold.jedox_dimpc_entity_detail 
WHERE parent = 'AE1-HK-MC1';
```

### Account Mapping (ObjSub → Management Account)

Two-layer traversal through `jedox_dimpc_account_detail`:

```sql
-- Get ERP accounts that roll up to WC_AR
SELECT dimpc2.child AS erp_account, dimpc2.account_description
FROM fna.fna_silver.jedox_dimfh_account_plan ac
JOIN fna.fna_silver.jedox_dimpc_account_detail dimpc1 ON ac.element = dimpc1.parent
JOIN fna.fna_silver.jedox_dimpc_account_detail dimpc2 ON dimpc1.child = dimpc2.parent
WHERE ac.level5 = 'WC_AR' AND ac.level2 = 'WC';  -- Filter level2 to avoid duplicates
```

### Account Type Filters (for Management Reporting)

| Metric | Filter |
|--------|--------|
| Turnover | `level9 = 'P651000MT'` |
| Revenue | `level6 = 'TMMT'` |
| COGS | `level9 = 'P710100MT'` |
| OPEX | `level6 = 'O2MT'` |
| MPC | `level7 = 'ODMMT'` |
| C2 | `level5 = 'C2MT'` |
| AR | `level5 = 'WC_AR'` |
| AP | `level5 = 'WC_AP'` |
| Inventory | `level5 = 'B201000'` |

---

## 6. Text Search Strategy

Vendor info may appear in any of three text fields - search all:

| Field | Content | Example |
|-------|---------|---------|
| `CP_ALPHA_NAME` | Vendor/counterparty name | `PROGRESS MFG GROUP` |
| `NAME_ALPHA_EXPLANATION` | Journal description | `Corp Credit Card 08/25` |
| `NAME_REMARK_EXPLANATION` | Remarks/notes | `Openrouter-AI Tools` |

```sql
SELECT CP_ALPHA_NAME, NAME_ALPHA_EXPLANATION, NAME_REMARK_EXPLANATION, AMOUNT_USD
FROM fna.fna_gold.jedoxviewer_e1_details
WHERE YEAR_MONTH BETWEEN 202501 AND 202510
  AND (UPPER(COALESCE(CP_ALPHA_NAME, '')) LIKE '%SEARCHTERM%'
       OR UPPER(COALESCE(NAME_ALPHA_EXPLANATION, '')) LIKE '%SEARCHTERM%'
       OR UPPER(COALESCE(NAME_REMARK_EXPLANATION, '')) LIKE '%SEARCHTERM%');
```

---

## 7. Reconciliation

### E1 to Plan/CC Cube

| Comparison | Expected Variance |
|------------|-------------------|
| E1 Detail vs Plan/CC Cube (Input) | < 0.01% (FX rounding) |
| Plan/CC Cube (Input) vs Total | Management Adjustments |

```sql
-- E1 Detail Total (for specific entity/account/month)
WITH entity_map AS (
  SELECT child AS bu_code 
  FROM fna.fna_gold.jedox_dimpc_entity_detail 
  WHERE parent = 'AE1-HK-MC1'
)
SELECT ABS(SUM(e1.AMOUNT_USD)) AS e1_total
FROM fna.fna_gold.jedoxviewer_e1_details e1
JOIN entity_map em ON e1.BU = em.bu_code
JOIN fna.fna_gold.jedox_dimfh_account_detail ac ON e1.ObjSub = ac.element
WHERE e1.YEAR_MONTH = 202510 AND ac.level9 = 'P651000MT';

-- Plan/CC Cube Total (Input measure only)
SELECT ABS(SUM(fact.signed) * 1000) AS plancc_input
FROM fna.fna_gold.jedoxviewer_plan_cc_transformed fact
JOIN fna.fna_silver.jedox_dimfh_account_plan ac ON fact.account_plan = ac.element
JOIN fna.fna_gold.jedoxviewer_entity ou ON fact.entity_plan = ou.entb
WHERE ou.ent1 = 'OG_Forecast' AND ou.entb = 'AE1-HK-MC1' AND fact.currency = '> USD'
  AND fact.measure = 'Input' AND ac.level9 = 'P651000MT'
  AND fact.period_key LIKE '2025-10%' AND SUBSTR(fact.period_key, 9) = 'actual';
```

**Investigating discrepancies:**
1. Filter Plan/CC by `measure = 'Input'` to isolate ERP-equivalent data
2. Check measure breakdown: `SELECT measure, SUM(signed)*1000 FROM ... GROUP BY measure`
3. FX differences (<0.5%) normal; larger gaps = missing BU mappings or non-Input measures

---

## 8. Intercompany vs External

E1 distinguishes IC from external via ERP account:

| Account | Type |
|---------|------|
| `5001` | External sales |
| `5002` | Intercompany sales |
| `5201` | External purchases (COGS) |
| `5202` | Intercompany purchases |

```sql
SELECT 
    CASE WHEN e.ObjSub = '5001' THEN 'External' 
         WHEN e.ObjSub = '5002' THEN 'Intercompany' END AS sale_type,
    -SUM(e.AMOUNT_USD)/1e6 AS turnover_m
FROM fna.fna_gold.jedoxviewer_e1_details e
WHERE e.ObjSub IN ('5001', '5002') AND e.YEAR_MONTH BETWEEN 202401 AND 202412
GROUP BY CASE WHEN e.ObjSub = '5001' THEN 'External' WHEN e.ObjSub = '5002' THEN 'Intercompany' END;
```

---

## 9. Key Legal Companies (2024 Profile)

### Trading / Sourcing Hubs

| Code | Name | Ccy | 2024 Sales | Primary OGs |
|------|------|-----|------------|-------------|
| **00001** | LI & FUNG (TRADING) LTD | HKD | $4,358M | H&A, Apparel, LFEU |
| **00288** | LF CENTENNIAL PTE LTD | USD | $1,426M | Apparel, LFEU, LFMU, H&A |

### Regional Selling Entities

| Code | Name | Ccy | 2024 Sales | Primary OGs |
|------|------|-----|------------|-------------|
| **00492** | LF FASHION LIMITED | GBP | $153M | LFEU |
| **00477** | MILES GMBH | EUR | $175M | LFEU (Miles) |
| **00450** | LF MEN'S GROUP LLC | USD | $186M | LFMU |
| **00245** | PRODUCT DEVELOPMENT PARTNERS | HKD | $166M | LFMU, H&A |
| **00236** | MIGHTY HURRICANE HOLDINGS INC | USD | $137M | LFMU |
| **00414** | LF SOURCING (MILLWORK) LLC | USD | $59M | Apparel |
| **00272** | W S TRADING LTD | HKD | $95M | H&A |
| **00214** | LI & FUNG TRADING (SH) LTD | CNY | $26M | LFAD, Promocean, LFEU |
| **00897** | E J ORR LIMITED | GBP | $26M | ST:ORR |

### Promocean

| Code | Name | Ccy | 2024 Sales |
|------|------|-----|------------|
| **00250** | PROMOCEAN THE NETHERLANDS BV | EUR | $62M |
| **00249** | PROMOCEAN SPAIN SL | EUR | $14M |
| **00244** | PROMOCEAN FRANCE | EUR | $4M |

### Firework

| Code | Name | Ccy | 2024 Sales |
|------|------|-----|------------|
| **00271** | SHIU FUNG FIREWORKS (CS) | CNY | $16M |
| **00063** | BLACK CAT FIREWORKS LTD | GBP | $10M |

### GFS (Logistics) - Service entities, no sales

| Code | Name | Ccy |
|------|------|-----|
| **00021** | GFS Shanghai | CNY |
| **00022** | GFS Shanghai - Ningbo | CNY |
| **00839** | GLOBAL FREIGHT SERVICES USA | USD |
| **00636** | GLOBAL FREIGHT SER. (HK) | HKD |

### Consolidation / Elimination

| Code | Name | Notes |
|------|------|-------|
| **00998** | HYPERION (BU MODEL) | Non-E1; Jedox consolidation |
| **00999** | LF GROUP - COST ALLOCATION | Non-E1; IC elimination |

---

## 10. Data Traps

### Duplicate Hierarchy Roots (BS vs WC)

Some accounts exist in **both** `level2 = 'BS'` and `level2 = 'WC'`. Always filter:

```sql
-- WRONG: Double-counts
SELECT SUM(a.value) FROM jedox_actual a JOIN ... ON ac.element = 'B208000';

-- CORRECT: Filter hierarchy
SELECT SUM(a.value) FROM jedox_actual a JOIN ... ON ac.element = 'B208000' AND ac.level2 = 'WC';
```

### BU Mapping Gaps

Not all BUs mapped in `jedox_dimpc_entity_detail`. If totals don't match:
1. Check for missing BU mappings
2. Query E1 without entity join for full picture
3. Unmapped amounts may appear in "XXX" entities

### Scaling

| Source | Scale |
|--------|-------|
| E1 Detail (`AMOUNT_USD`) | Ones (unscaled) |
| Plan/CC Cube (`signed`) | Thousands (multiply by 1000) |
| jedox_actual (`value`) | Ones (unscaled) |

**jedox_actual notes:** The table is `fna.fna_silver.jedox_actual`. Company codes are zero-padded five-digit strings. Currency filter is `'FC > USD'` (not `'> USD'`). Entity key is `entity_detail` (BU code directly). The `value` field is YTD-cumulative and resets at the year boundary; difference consecutive months for monthly flow.

---

## 11. SQL Templates

### Cash Payments to Vendor

```sql
-- Layer 3 query: Actual cash paid
SELECT 
    CAST(YEAR_MONTH/100 AS INT) AS Year, 
    MOD(YEAR_MONTH, 100) AS Month, 
    ROUND(-SUM(AMOUNT_USD), 0) AS Payments_USD
FROM fna.fna_gold.jedoxviewer_e1_details
WHERE LOWER(CP_ALPHA_NAME) LIKE '%vendor%' 
  AND DOCUMENT_TYPE = 'PT' 
  AND YEAR_MONTH >= 202401
GROUP BY CAST(YEAR_MONTH/100 AS INT), MOD(YEAR_MONTH, 100) 
ORDER BY Year DESC, Month DESC;
```

### Cash Receipts from Customer

```sql
-- Layer 3 query: Actual cash collected
SELECT 
    CAST(YEAR_MONTH/100 AS INT) AS Year, 
    MOD(YEAR_MONTH, 100) AS Month, 
    ROUND(SUM(AMOUNT_USD), 0) AS Receipts_USD
FROM fna.fna_gold.jedoxviewer_e1_details
WHERE LOWER(CP_ALPHA_NAME) LIKE '%customer%' 
  AND DOCUMENT_TYPE = 'RC' 
  AND YEAR_MONTH >= 202401
GROUP BY CAST(YEAR_MONTH/100 AS INT), MOD(YEAR_MONTH, 100) 
ORDER BY Year DESC, Month DESC;
```

### P&L Expense by Vendor (Clean Trend)

```sql
-- Layer 1 query: Recognized expense, excluding reversing accruals
SELECT 
    CAST(YEAR_MONTH/100 AS INT) AS Year, 
    MOD(YEAR_MONTH, 100) AS Month, 
    ROUND(SUM(AMOUNT_USD), 0) AS Expense_USD
FROM fna.fna_gold.jedoxviewer_e1_details
WHERE LOWER(CP_ALPHA_NAME) LIKE '%vendor%' 
  AND ObjSub LIKE '6%'  -- Operating expense accounts
  AND DOCUMENT_TYPE NOT IN ('JR', 'JC')  -- Exclude reversing accruals
  AND YEAR_MONTH >= 202401
GROUP BY CAST(YEAR_MONTH/100 AS INT), MOD(YEAR_MONTH, 100) 
ORDER BY Year DESC, Month DESC;
```

### Analyze Document Type Mix for Counterparty

```sql
-- Diagnostic: See which layers a counterparty appears in
SELECT 
    DOCUMENT_TYPE,
    CASE 
        WHEN ObjSub LIKE '5%' OR ObjSub LIKE '6%' OR ObjSub LIKE '7%' OR ObjSub LIKE '8%' THEN 'Accrual'
        WHEN ObjSub LIKE '3%' THEN 'Invoice'
        WHEN ObjSub LIKE '22%' THEN 'Cash'
        ELSE 'Other'
    END AS layer,
    COUNT(*) AS txns,
    ROUND(SUM(CASE WHEN AMOUNT_USD > 0 THEN AMOUNT_USD ELSE 0 END), 0) AS debits,
    ROUND(SUM(CASE WHEN AMOUNT_USD < 0 THEN AMOUNT_USD ELSE 0 END), 0) AS credits,
    ROUND(SUM(AMOUNT_USD), 0) AS net
FROM fna.fna_gold.jedoxviewer_e1_details
WHERE LOWER(CP_ALPHA_NAME) LIKE '%vendor%' 
  AND YEAR_MONTH >= 202401
GROUP BY DOCUMENT_TYPE, 
    CASE 
        WHEN ObjSub LIKE '5%' OR ObjSub LIKE '6%' OR ObjSub LIKE '7%' OR ObjSub LIKE '8%' THEN 'Accrual'
        WHEN ObjSub LIKE '3%' THEN 'Invoice'
        WHEN ObjSub LIKE '22%' THEN 'Cash'
        ELSE 'Other'
    END
ORDER BY txns DESC;
```

### E1 Lines for Entity/Account/Month

```sql
WITH entity_map AS (
  SELECT child AS bu_code 
  FROM fna.fna_gold.jedox_dimpc_entity_detail 
  WHERE parent = 'AE1-HK-MC1'
)
SELECT 
    DATE_FORMAT(e1.GL_DATE, 'yyyy-MM-dd') AS gl_date, 
    e1.BATCH_NUMBER, 
    e1.DOC_NUMBER,
    e1.ObjSub AS erp_account, 
    e1.AMOUNT_USD, 
    e1.CP_ALPHA_NAME AS vendor, 
    e1.COMPANY,
    e1.DOCUMENT_TYPE
FROM fna.fna_gold.jedoxviewer_e1_details e1
JOIN entity_map em ON e1.BU = em.bu_code
JOIN fna.fna_gold.jedox_dimfh_account_detail ac ON e1.ObjSub = ac.element
WHERE e1.YEAR_MONTH = 202510 AND ac.level9 = 'P651000MT'
ORDER BY ABS(e1.AMOUNT_USD) DESC 
LIMIT 50;
```

---

## 12. Forensic Investigation Principles

When tracing specific transactions (e.g., "did we receive payment X?"), standard text search often fails. These principles address common friction points.

### 12.1 Batch Context (The "Row is a Lie" Rule)

**Problem:** Single-row searches assume the answer is contained in one line. In double-entry accounting, meaning is split across the batch.

**Solution:** When you find a vague candidate line, immediately pull the entire `BATCH_NUMBER`:

```sql
-- Found a suspicious line? Get full context
SELECT ObjSub, AMOUNT_USD, CP_ALPHA_NAME, NAME_ALPHA_EXPLANATION, NAME_REMARK_EXPLANATION
FROM fna.fna_gold.jedoxviewer_e1_details
WHERE BATCH_NUMBER = 30501448;
```

The credit side might say "Unknown Receipt" (useless), but the debit side shows "Citibank" (useful), or another line has the invoice number.

### 12.2 State Transition (Suspense Account Pattern)

**Problem:** Searching for the "final state" (e.g., "ERC Income") misses the "initial state" (e.g., "Unknown Receipt").

**Reality:** Financial data matures over time:
- *Day 0:* Cash hits bank → Booked to Suspense/Clearing with vague description
- *Day 5:* Accountant identifies it → Reclassifies to proper account
- *Day 30:* Month-end close → Final allocation

**Solution:** When you can't find the expected "Income" or "Expense," search the suspense/clearing accounts:

```sql
-- Key suspense accounts to search
SELECT GL_DATE, BATCH_NUMBER, ObjSub, AMOUNT_USD, NAME_ALPHA_EXPLANATION
FROM fna.fna_gold.jedoxviewer_e1_details
WHERE ObjSub IN ('3006', '3011', '9900')  -- Receipt in Advance, Accrued Expenses, Clearing
  AND YEAR_MONTH BETWEEN 202505 AND 202506
  AND ABS(AMOUNT_USD) > 100000  -- Adjust threshold as needed
ORDER BY ABS(AMOUNT_USD) DESC;
```

### 12.3 Temporal Fuzziness (GL Date vs Value Date)

**Problem:** Strict month filtering misses transactions that straddle month-end.

**Reality:** The check arrives May 29th, but the reclassification journal is posted June 5th. Depending on which date you query, the transaction appears in different months.

**Solution:** 
1. Apply **±10 day buffer** around target dates
2. For cash timing questions, trust the **Bank Account (22xx) entry date**, not the reclassification date
3. When amounts match but dates don't, trace via `BATCH_NUMBER` to find the original entry

### 12.4 Hierarchy of Truth (Cash > Accrual)

**Problem:** Trusting the P&L to answer "when did we get money?"

**Reality:** The P&L tells you when finance *decided* to recognize income, not when cash arrived.

**Solution:** For questions about actual cash movement:
1. Start with Balance Sheet accounts (Bank `22xx`, AR `2004`, AP `3003`)
2. Use document types that represent real cash: `PT` (Payment), `RC` (Receipt)
3. Only then trace to P&L accounts for the "how was it classified" question

### 12.5 Multi-Entity Allocation Pattern

**Problem:** Group-level receipts (tax refunds, intercompany settlements) are received by one legal entity but allocated across many management entities.

**Example:** ERC refund received by Company 00236 (Mighty Hurricane), then allocated to multiple divisions via JE.

**Solution:** Trace in two steps:
1. **Find the cash:** Query by Company code to find the original bank receipt
2. **Find the allocation:** Use the `BATCH_NUMBER` from step 1 to find related reclassification entries, or search for the same amount in `3011` (Accrued Expenses) being cleared

```sql
-- Step 1: Find cash receipt by company
SELECT GL_DATE, BATCH_NUMBER, ObjSub, AMOUNT_USD, NAME_ALPHA_EXPLANATION
FROM fna.fna_gold.jedoxviewer_e1_details
WHERE COMPANY = '00236' AND ObjSub LIKE '22%' AND AMOUNT_USD > 500000
  AND YEAR_MONTH BETWEEN 202505 AND 202506;

-- Step 2: Trace allocation (using amount as anchor)
SELECT YEAR_MONTH, BU, ObjSub, AMOUNT_USD, NAME_REMARK_EXPLANATION
FROM fna.fna_gold.jedoxviewer_e1_details
WHERE ABS(AMOUNT_USD) BETWEEN 1055000 AND 1056000
  AND YEAR_MONTH BETWEEN 202505 AND 202512;
```

---

## 13. Batches & Document Hierarchy

Understanding how E1 organizes transactions is essential for forensic investigation.

### 13.1 The Document Hierarchy

```
BATCH_NUMBER (Logical grouping of related documents)
  └── Contains one or more DOCUMENTS
        └── Each DOCUMENT identified by (DOCUMENT_TYPE, DOC_NUMBER, DOC_COMPANY)
              └── Each DOCUMENT has one or more LINE_NUMBERs
                    └── Each LINE impacts a COMPANY (booking co, may differ from DOC_COMPANY)
```

### 13.2 The Document Key

The **4-tuple `(DOCUMENT_TYPE, DOC_NUMBER, DOC_COMPANY, LINE_NUMBER)`** is the unique identifier for a journal line in the source ERP system.

**Important caveats in our DWH:**
- This key is **not perfectly unique** in our data (~10% apparent duplicates)
- Most duplicates are **legitimate** (see 13.4 below)
- Some may be DWH loading artifacts

### 13.3 COMPANY vs DOC_COMPANY

These are distinct concepts:

| Column | Meaning | Example |
|--------|---------|---------|
| `DOC_COMPANY` | **Originating company** - where the document was entered | `00001` (HK parent creates JE) |
| `COMPANY` | **Booking company** - legal entity impacted by this line | `00280` (subsidiary affected) |

**When they differ (~0.6% of rows):**
- **JX (FX Revaluation):** Parent company runs month-end revaluation impacting all subsidiaries
- **JE (Manual Journals):** Cross-entity allocations (salary, rent, shared costs)
- **Example:** HK parent (00001) creates a cost allocation JE that books expenses to 40+ subsidiary companies

```sql
-- Find cross-company journal entries
SELECT COMPANY, DOC_COMPANY, DOCUMENT_TYPE, COUNT(*) as lines
FROM fna.fna_gold.jedoxviewer_e1_details
WHERE COMPANY != DOC_COMPANY
GROUP BY COMPANY, DOC_COMPANY, DOCUMENT_TYPE
ORDER BY lines DESC LIMIT 10;
```

### 13.4 The R Column (Reversal Flag)

| Value | Meaning | Use |
|-------|---------|-----|
| `R = 'R'` | Reversal entry | Filter OUT for clean trends |
| `R = ' '` | Original entry | Normal transactions |

**Why duplicates appear for JR/JC documents:**
- Auto-reversing journals create **two rows** with the same document key
- Original entry (Month 1) and reversal (Month 2)
- Different `GL_DATE`, opposite `AMOUNT_USD` signs
- They **net to $0** - this is expected ERP behavior, not a data glitch

```sql
-- JR/JC entries net to zero
SELECT R, COUNT(*) as rows, SUM(AMOUNT_USD) as net
FROM fna.fna_gold.jedoxviewer_e1_details
WHERE DOCUMENT_TYPE = 'JR'
GROUP BY R;
-- R='R' rows net to negative, R=' ' rows net to positive, total = $0
```

### 13.5 Batch Concepts

A **batch** is a logical grouping of related documents, defined by operators (accounting users or system processes).

**Key statistics:**
- ~5.8 million batches in the database
- 99.9998% of batches balance to exactly $0 (only 11 exceptions, all `OV` doc types)
- Batch numbers are sequential and grow over time (~888K to ~31M range)

**Batch size distribution:**

| Size | % of Batches | Typical Pattern |
|------|--------------|-----------------|
| 2 lines | 68% | Simple invoice: Dr Expense / Cr AP |
| 3-5 lines | 15% | Invoice with tax or multi-line |
| 6-10 lines | 8% | Payment clearing multiple items |
| 11-50 lines | 7% | Complex allocation or landed cost |
| 51-1000 lines | 2.6% | Month-end journals, IC settlement |
| 1000+ lines | 0.2% | System interface batches |

### 13.6 Batch Archetypes

**Simple Invoice (68% of batches)**
- 2 lines: Dr Expense / Cr AP
- Doc types: `VX`, `VZ`, `VM`

**Cash Payment/Receipt**
- 2-10 lines
- `PT`: Dr AP / Cr Bank (paying vendor)
- `RC`: Dr Bank / Cr AR (receiving from customer)

**Sales Invoice**
- 10-20 lines per customer
- `UX`: Dr AR / Cr Revenue, Dr COGS / Cr Inventory

**Mass Interface Batch (rare but huge)**
- 100K-275K lines
- Single customer (e.g., Kohl's EDI feed)
- All lines = one system interface run

**FX Revaluation**
- `JX` documents
- Can span 100+ companies in one batch
- DOC_COMPANY = parent, COMPANY = each subsidiary

**Amortization Schedule**
- `JE` spanning 30+ GL_DATEs
- Same amount Dr/Cr each month
- Used for prepaid expenses

### 13.7 The AE (Auto Entry) Pattern

`AE` documents are the **offsetting entries** created automatically by the ERP:

- Present in 73% of all batches
- Common combinations: `VZ+AE`, `VX+AE`, `VM+AE`, `PT+AE`, `RC+AE`
- **Rule:** When you see a transactional doc type, there's almost always an `AE` with the other side

### 13.8 Investigation Strategy by Batch Size

| Batch Size | Likely Pattern | Approach |
|------------|----------------|----------|
| 2 lines | Simple transaction | Read both lines directly |
| 3-10 lines | Standard invoice/payment | Group by `ObjSub` to see structure |
| 11-100 lines | Complex allocation | Group by counterparty and account |
| 100+ lines | Interface batch | Sample first, then aggregate by customer/account |
| 1000+ lines | System interface | Query aggregates only, don't pull raw lines |

```sql
-- Inspect a batch: always start here for forensic work
SELECT ObjSub, DOCUMENT_TYPE, AMOUNT_USD, CP_ALPHA_NAME, NAME_ALPHA_EXPLANATION
FROM fna.fna_gold.jedoxviewer_e1_details
WHERE BATCH_NUMBER = 30501448
ORDER BY AMOUNT_USD DESC;

-- For large batches, aggregate first
SELECT ObjSub, DOCUMENT_TYPE, COUNT(*) as lines, SUM(AMOUNT_USD) as total
FROM fna.fna_gold.jedoxviewer_e1_details
WHERE BATCH_NUMBER = 30021201
GROUP BY ObjSub, DOCUMENT_TYPE
ORDER BY total DESC;
```

### 13.9 Document Type Average Batch Sizes

| Doc Type | Avg Lines/Batch | Notes |
|----------|-----------------|-------|
| `UX` | 459 | High-volume trading interfaces |
| `JA` | 1,715 | Factoring allocation batches |
| `JX` | 80 | Monthly FX revaluation |
| `JE` | 69 | Manual journals, variable complexity |
| `OV` | 84 | Inventory vouchers (Firework) |
| `VL` | 22 | Landed cost breakdown |
| `VM` | 14 | PO matching |
| `RC` | 10 | Customer receipts |
| `PT` | 8 | Vendor payments |
| `VX` | 11 | Non-PO expenses |
| `VZ` | 1 | Simple vouchers |



# AR/AP Subledger Tables (F03B11 / F0411)

This document covers the JDE AR and AP subledger tables used for aging reports and reconciliation to the Actual Cube.

**Key Use Cases:** AR/AP aging reports | Customer/Supplier balance drill-down | Reconciliation to Actual Cube | Cash collection analysis

**Relationship to Other Sources:**
- **Actual Cube (`jedox_actual`)**: F03B11/F0411 are subledger detail; Actual Cube is summarized by account
- **Plan/CC Cube**: WC_AR/WC_AP in Plan/CC includes allocations and adjustments not in subledger
- **E1 Detail (`jedoxviewer_e1_details`)**: E1 is GL journal lines; subledger is customer/supplier open items

---

## 1. Critical Concepts

### 1.1 Amount Storage Convention

**Amounts are stored in BASE CURRENCY CENTS (divide by 100):**

```sql
amount_in_base_currency = RPAAP / 100
```

The base currency (`RPBCRC`) is the **domestic/functional currency** of the company, NOT the transaction currency:
- Company 00001 (LF Trading HK): Base = HKD
- Company 00288 (LF Centennial SG): Base = USD
- Company 00492 (LF Fashion UK): Base = GBP

### 1.2 Currency Conversion to USD

**Use `lft.bronze.f0015` for FX rates.** For Balance Sheet items like AR/AP, use the **latest available rate** (as-of date convention):

```sql
WITH latest_fx AS (
    SELECT 
        CXCRCD as from_ccy,
        CXCRR as rate_to_usd,
        ROW_NUMBER() OVER (PARTITION BY CXCRCD ORDER BY CXEFT DESC) as rn
    FROM lft.bronze.f0015
    WHERE CXCRDC = 'USD'
),
fx_rates AS (
    SELECT from_ccy, rate_to_usd FROM latest_fx WHERE rn = 1
)
SELECT 
    ar.RPAAP / 100 * COALESCE(fx.rate_to_usd, 1) as amount_usd
FROM lft.bronze.f03b11 ar
LEFT JOIN fx_rates fx ON ar.RPBCRC = fx.from_ccy
```

### 1.3 GL Class - The Critical Filter

**`RPGLC` determines what type of AR/AP you're looking at:**

| GL Class | Description | Include in External AR? |
|----------|-------------|------------------------|
| `2004` | Trade Debtors (3rd Party) | **Yes** |
| `4FRD` | Forward/Unbilled AR | **Yes** (partial) |
| `2005` | Other Receivables | Depends |
| `3RSU` | Related Party Suspense (IC) | **No** |
| `8RSU` | Related Party Suspense (IC) | **No** |
| `3RTD` | Related Party Trade (IC) | **No** |
| `6ADV` | Advances | **No** |

**For external trade AR, always filter:**
```sql
WHERE RPGLC IN ('2004', '4FRD')
```

### 1.4 Mapping to Actual Cube Accounts

| Subledger GL Class | Actual Cube Account | Notes |
|-------------------|---------------------|-------|
| `2004` | `2004` | Direct match |
| `4FRD` | `2004FRD` | Partial - some FRD is GL-only |
| N/A | `2004ACC` | **GL-only** - not in subledger |
| N/A | `2004GEN`, `2004REV`, etc. | GL-only adjustments |

**Key insight:** `2004ACC` (Accrued AR) is booked directly in GL, not through the AR subledger. This is normal - accruals don't have a specific customer invoice yet.

---

## 2. Table Schema: F03B11 (AR Subledger)

**Table:** `lft.bronze.f03b11`

### 2.1 Key Columns

| Column | Type | Description | Example |
|--------|------|-------------|---------|
| `RPDOC` | decimal | Document number | 25003110 |
| `RPDCT` | string | Document type | `UX`, `UM`, `UC` |
| `RPSFX` | string | Document suffix | `001` |
| `RPKCO` | string | Document company | `00001` |
| `RPCO` | string | Company (legal entity) | `00001` |
| `RPAN8` | decimal | Customer address number | 107167 |
| `RPALPH` | string | Customer name | `C & J CLARK AMERICA,INC.` |
| `RPMCU` | string | Business Unit | `001034603` (maps to entity) |
| `RPGLC` | string | GL Class | `2004` |
| `RPPST` | string | Payment status | `A`=Open, `P`=Paid |
| `RPAG` | decimal | Gross amount (base ccy cents) | 301244504 |
| `RPAAP` | decimal | Open amount (base ccy cents) | 301244504 |
| `RPBCRC` | string | Base currency | `HKD` |
| `RPCRCD` | string | Transaction currency | `USD` |
| `RPCRR` | decimal | Exchange rate | 7.76 |
| `RPDDJ` | decimal | Due date (Julian CYYDDD) | 126034 |
| `RPDGJ` | decimal | GL date (Julian CYYDDD) | 125301 |
| `RPDIVJ` | decimal | Invoice date (Julian) | 125297 |
| `extraction_date` | date | Snapshot date | 2026-02-04 |

### 2.2 Document Types

| Code | Description | Typical Use |
|------|-------------|-------------|
| `UX` | Sales Invoice | Standard customer invoice |
| `UM` | Debit/Credit Memo | Adjustments |
| `UC` | Credit Memo | Returns, corrections |
| `UA` | Unapplied Receipt | Cash received, not yet applied |
| `UG` | Chargeback | Disputed amounts |
| `UH` | Deduction | Customer deductions |
| `R5` | Receipt | Cash receipt applied |

### 2.3 Julian Date Conversion

JDE uses CYYDDD format (Century-Year-DayOfYear):
- `126034` = Century 1 (2000s), Year 26, Day 34 = Feb 3, 2026

```sql
-- Convert Julian to Date
DATE_ADD(
    CONCAT(
        CASE WHEN FLOOR(RPDDJ/100000) = 0 THEN '19' ELSE '20' END,
        LPAD(CAST(FLOOR(MOD(RPDDJ, 100000)/1000) AS INT), 2, '0'),
        '-01-01'
    ),
    CAST(MOD(RPDDJ, 1000) AS INT) - 1
) as due_date
```

---

## 3. Table Schema: F0411 (AP Subledger)

**Table:** `lft.bronze.f0411`

Similar structure to F03B11 but for Accounts Payable. Key differences:

| Column | Description |
|--------|-------------|
| `RPAN8` | Supplier address number |
| `RPALPH` | Supplier name |

Document types for AP:
| Code | Description |
|------|-------------|
| `VM` | Voucher Match (PO-based) |
| `VX` | Voucher Expense (non-PO) |
| `VL` | Voucher Logged |
| `VH` | Voucher Hold |
| `VP` | Payment |

---

## 4. Entity Mapping

### 4.1 Business Unit to Management Entity

The `RPMCU` field contains the Business Unit code that maps to Plan/CC entities:

```sql
-- Map BU to entity and OG
SELECT 
    ar.RPMCU as bu_code,
    d.parent as entity,
    ou.ent3name as OG
FROM lft.bronze.f03b11 ar
JOIN fna.fna_gold.jedox_dimpc_entity_detail d ON TRIM(ar.RPMCU) = d.child
JOIN fna.fna_gold.jedoxviewer_entity ou ON d.parent = ou.entb AND ou.ent1 = 'OG_Forecast'
```

### 4.2 Company to Entity Relationship

One company can have multiple entities (BUs), and vice versa:
- Company 00001 has BUs across Apparel, H&A, LFEU, LFMU, Corporate
- Apparel OG has BUs in companies 00001, 00288, 00884, etc.

---

## 5. Replay Logic & Reconciliation to Actual Cube

> **Validation Status:** PARTIALLY VALIDATED
> - Tested on single BU (288081601 / TAW Div) in company 00288 (USD base currency)
> - F03B11 replay validated for 2 recent periods (Dec 31, 2025 and Jan 31, 2026)
> - AR Aging table validated for 6 periods (Jan 2025 - Jan 2026) with **$0 variance** to Actual Cube
> - Further testing needed across multiple BUs and companies with non-USD base currency

### 5.1 Replay Concept

The subledger can be "replayed" to any as-of date to reconstruct the AR balance at that point in time. This enables:
- **Perfect reconciliation** to month-end Actual Cube balances
- **Historical aging reports** as of any past date
- **Point-in-time analysis** for audit or investigation

**Key insight:** The subledger retains both open AND recently paid items. By filtering on invoice GL date (`RPDGJ`), we can reconstruct what was open at any point.

### 5.2 Replay Logic by Period Type

**For current period (latest month-end where extraction is after month-end):**
```sql
-- Only currently open items where invoice was booked by month-end
WHERE RPPST = 'A'                    -- Currently open items only
  AND RPDGJ <= {as_of_julian_date}   -- Invoice GL date <= as-of date
```

**For historical periods (prior months):**
```sql
-- Open items + Paid items (which were open at that time)
WHERE RPDGJ <= {as_of_julian_date}   -- Invoice GL date <= as-of date
-- Amount logic:
--   RPPST = 'A' (open): use RPAAP (current open amount)
--   RPPST = 'P' (paid): use RPAG (gross amount - was fully open then)
```

### 5.3 Validated Reconciliation (BU 288081601 / TAW Div)

**Perfect match achieved for GL Class 2004, Account 2004:**

| Period | Subledger Replay | Actual Cube | Diff | Diff % |
|--------|------------------|-------------|------|--------|
| 2025-12-31 | $20,079,396 | $20,079,146 | $250 | 0.001% |
| 2026-01-31 | $28,047,279 | $28,047,279 | $0 | 0.000% |

**Conditions for perfect match:**
- Single BU (9-digit code) for precise entity mapping
- Single GL Class (2004) matching single account_detail (2004)
- Company 00288 where base currency = USD (no FX conversion needed)
- Subledger extraction date (Feb 4) is after the as-of dates

### 5.4 Replay Limitations

**Data retention:** The subledger only retains items back ~2-3 months. Older paid items are purged.
- Example: BU 288081601 earliest invoice = Nov 11, 2025 (Julian 125315)
- Replay to Oct 31, 2025 or earlier will be incomplete

**Partial payments:** If an item was partially paid between historical date and now, the replay may be slightly off. The $250 variance on Dec 31 is likely due to this.

### 5.5 Extraction Date Model & Limitations

**IMPORTANT:** The extraction model uses delta storage, not daily full snapshots.

**Tested hypothesis: Combining multiple extraction dates**

We tested whether combining extractions could reconstruct historical balances:
- `RPSMTJ` (statement date) indicates the last statement an item appeared on
- Items with `RPSMTJ >= Nov 30` should theoretically have been open on Nov 30

**Result:** This approach **over-counts by ~50%**:

| Approach | Result | Target | Gap |
|----------|--------|--------|-----|
| Max stmt_date >= Nov 30 | $29.1M | $19.2M | +52% |
| AR Aging (ground truth) | $19.2M | $19.2M | 0% |

**Root cause:** 760 invoices ($9.9M) appear with stmt_date = Nov 30 but were already **paid before Nov 30**. The statement date indicates when an item was last processed, not when it was open.

**Conclusion:** F03B11 cannot reliably reconstruct historical balances. Use AR Aging for any period beyond current month.

| Extraction Type | Frequency | Contains | Use Case |
|-----------------|-----------|----------|----------|
| **Full Snapshot** | ~Monthly (first week) | All open items (`RPPST='A'`) + recently paid | Use for replay |
| **Delta Extract** | Daily | Only items that changed (mostly newly paid) | Not useful for replay |

**How to identify a full snapshot:**
- Has significant count of `RPPST = 'A'` (open) items
- Latest extraction (Feb 4, 2026) has 186K open items with $85.7B open AR

**Best practice:** Always use the **latest extraction_date** for replay queries. Historical extraction dates contain only delta records and cannot reconstruct point-in-time balances.

```sql
-- Always filter to latest extraction
WHERE extraction_date = (SELECT MAX(extraction_date) FROM lft.bronze.f03b11)
```

### 5.6 Alternative: AR Aging Table for Historical Replay

**For historical periods beyond F03B11 retention, use the AR Aging table instead.**

The AR Aging table (`lft.bronze.dds_jrpt_factoring_ar_aging`) has:
- **Monthly snapshots** back to Dec 2020
- **Invoice-level detail** matching F03B11
- **Perfect reconciliation** to Actual Cube (for GLCLASS 2004)

**Comparison for BU 288081601, Nov 2025:**

| Source | AR USD | Coverage |
|--------|--------|----------|
| AR Aging (AGING_PERIOD='202511') | $19,210,956 | **100%** |
| Actual Cube (2025-11_YTD) | $19,210,956 | 100% |
| F03B11 Replay (latest extraction) | $3,386,051 | 18% (most items purged) |

**AR Aging vs Actual Cube validation (BU 288081601, GLCLASS 2004):**

| Period | AR Aging | Actual Cube | Diff |
|--------|----------|-------------|------|
| 2025-01 | $28,760,200 | $28,760,200 | $0 |
| 2025-06 | $17,198,757 | $17,198,757 | $0 |
| 2025-09 | $29,757,743 | $29,757,743 | $0 |
| 2025-11 | $19,210,956 | $19,210,956 | $0 |
| 2025-12 | $20,079,146 | $20,079,146 | $0 |
| 2026-01 | $28,047,279 | $28,047,279 | $0 |

**Expanded validation - Company 00001 by OG, Dec 2025:**

| OG | F03B11 Replay | AR Aging | Actual Cube | F03B11 Gap |
|----|---------------|----------|-------------|------------|
| Apparel | $100.1M | $128.7M | $128.7M | -22% |
| H&A | $81.9M | $83.3M | $83.3M | -2% |
| LF Europe | $11.9M | $16.7M | $16.7M | -29% |
| LF Markets USA | $0.6M | $0.6M | $0.6M | 0% |

**Current month validation - Company 00001 by OG, Jan 2026:**

| OG | F03B11 Replay | Actual Cube | Diff % |
|----|---------------|-------------|--------|
| Apparel | $132.6M | $133.6M | **-0.7%** |
| H&A | $88.6M | $88.6M | **0.0%** |
| LF Europe | $14.3M | $14.3M | **0.0%** |
| LF Markets USA | $1.6M | $1.6M | **0.0%** |

**Key findings:**
- **Current month:** F03B11 replay matches within 0-1%
- **Historical months (>1-2 months):** F03B11 replay is 20-30% lower (paid items purged)
- **AR Aging perfectly matches** Actual Cube for all historical periods
- Non-USD companies (e.g., 00001 = HKD) require FX conversion using `f0015` rates

**Recommendation:**
- **Current month / recent 1-2 months:** Use F03B11 (has payment status, more detail)
- **Historical months (>2 months back):** Use AR Aging table - it's the only reliable source

See **[AR_AGING.md](./AR_AGING.md)** for AR Aging table documentation.

### 5.5 Replay Query Template (Single BU, USD Base Currency)

```sql
-- Replay AR for a specific BU as of a given date
-- For company 00288 (base currency = USD), no FX conversion needed

-- Parameters:
--   {BU_CODE}: 9-digit business unit (e.g., '288081601')
--   {AS_OF_JULIAN}: Julian date CYYDDD (e.g., 125365 = Dec 31, 2025)
--   {IS_CURRENT}: true if as-of date is in current month

-- Current period logic (extraction is after month-end)
SELECT 
    SUM(RPAAP/100) as ar_usd
FROM lft.bronze.f03b11
WHERE RPCO = '00288'
  AND TRIM(RPMCU) = '{BU_CODE}'
  AND RPGLC = '2004'
  AND RPPST = 'A'  -- Only open items for current period
  AND extraction_date = (SELECT MAX(extraction_date) FROM lft.bronze.f03b11)
  AND RPDGJ <= {AS_OF_JULIAN};

-- Historical period logic (as-of date is prior month)
SELECT 
    SUM(CASE 
        WHEN RPPST = 'A' THEN RPAAP/100   -- Open: current balance
        WHEN RPPST = 'P' THEN RPAG/100    -- Paid: was fully open then
    END) as ar_usd
FROM lft.bronze.f03b11
WHERE RPCO = '00288'
  AND TRIM(RPMCU) = '{BU_CODE}'
  AND RPGLC = '2004'
  AND extraction_date = (SELECT MAX(extraction_date) FROM lft.bronze.f03b11)
  AND RPDGJ <= {AS_OF_JULIAN};
```

### 5.6 Julian Date Reference

| Date | Julian (CYYDDD) |
|------|-----------------|
| Nov 30, 2025 | 125334 |
| Dec 31, 2025 | 125365 |
| Jan 31, 2026 | 126031 |
| Feb 28, 2026 | 126059 |

**Conversion formula:** `Century (1=2000s) + YY + DDD (day of year)`

---

## 6. Aging Report Queries

### 6.1 Basic AR Aging by Customer

```sql
WITH latest_fx AS (
    SELECT 
        CXCRCD as from_ccy,
        CXCRR as rate_to_usd,
        ROW_NUMBER() OVER (PARTITION BY CXCRCD ORDER BY CXEFT DESC) as rn
    FROM lft.bronze.f0015
    WHERE CXCRDC = 'USD'
),
fx_rates AS (
    SELECT from_ccy, rate_to_usd FROM latest_fx WHERE rn = 1
),
ar_with_aging AS (
    SELECT 
        ar.RPCO as company,
        ar.RPALPH as customer_name,
        ar.RPAAP / 100 * COALESCE(fx.rate_to_usd, 1) as amount_usd,
        DATE_ADD(
            CONCAT(
                CASE WHEN FLOOR(ar.RPDDJ/100000) = 0 THEN '19' ELSE '20' END,
                LPAD(CAST(FLOOR(MOD(ar.RPDDJ, 100000)/1000) AS INT), 2, '0'),
                '-01-01'
            ),
            CAST(MOD(ar.RPDDJ, 1000) AS INT) - 1
        ) as due_date
    FROM lft.bronze.f03b11 ar
    LEFT JOIN fx_rates fx ON ar.RPBCRC = fx.from_ccy
    WHERE ar.extraction_date = (SELECT MAX(extraction_date) FROM lft.bronze.f03b11)
      AND ar.RPPST = 'A'
      AND ar.RPGLC IN ('2004', '4FRD')
)
SELECT 
    customer_name,
    SUM(CASE WHEN DATEDIFF(CURRENT_DATE, due_date) < 0 THEN amount_usd ELSE 0 END) as not_due,
    SUM(CASE WHEN DATEDIFF(CURRENT_DATE, due_date) BETWEEN 0 AND 30 THEN amount_usd ELSE 0 END) as days_1_30,
    SUM(CASE WHEN DATEDIFF(CURRENT_DATE, due_date) BETWEEN 31 AND 60 THEN amount_usd ELSE 0 END) as days_31_60,
    SUM(CASE WHEN DATEDIFF(CURRENT_DATE, due_date) BETWEEN 61 AND 90 THEN amount_usd ELSE 0 END) as days_61_90,
    SUM(CASE WHEN DATEDIFF(CURRENT_DATE, due_date) > 90 THEN amount_usd ELSE 0 END) as over_90,
    SUM(amount_usd) as total
FROM ar_with_aging
GROUP BY customer_name
ORDER BY SUM(amount_usd) DESC
LIMIT 20;
```

### 6.2 AR Aging by OG

```sql
WITH latest_fx AS (
    SELECT 
        CXCRCD as from_ccy,
        CXCRR as rate_to_usd,
        ROW_NUMBER() OVER (PARTITION BY CXCRCD ORDER BY CXEFT DESC) as rn
    FROM lft.bronze.f0015
    WHERE CXCRDC = 'USD'
),
fx_rates AS (
    SELECT from_ccy, rate_to_usd FROM latest_fx WHERE rn = 1
),
ar_with_aging AS (
    SELECT 
        COALESCE(ou.ent3name, 'Unmapped') as OG,
        ar.RPAAP / 100 * COALESCE(fx.rate_to_usd, 1) as amount_usd,
        DATE_ADD(
            CONCAT(
                CASE WHEN FLOOR(ar.RPDDJ/100000) = 0 THEN '19' ELSE '20' END,
                LPAD(CAST(FLOOR(MOD(ar.RPDDJ, 100000)/1000) AS INT), 2, '0'),
                '-01-01'
            ),
            CAST(MOD(ar.RPDDJ, 1000) AS INT) - 1
        ) as due_date
    FROM lft.bronze.f03b11 ar
    LEFT JOIN fx_rates fx ON ar.RPBCRC = fx.from_ccy
    LEFT JOIN fna.fna_gold.jedox_dimpc_entity_detail d ON TRIM(ar.RPMCU) = d.child
    LEFT JOIN fna.fna_gold.jedoxviewer_entity ou ON d.parent = ou.entb AND ou.ent1 = 'OG_Forecast'
    WHERE ar.extraction_date = (SELECT MAX(extraction_date) FROM lft.bronze.f03b11)
      AND ar.RPPST = 'A'
      AND ar.RPGLC IN ('2004', '4FRD')
)
SELECT 
    OG,
    ROUND(SUM(CASE WHEN DATEDIFF(CURRENT_DATE, due_date) < 0 THEN amount_usd ELSE 0 END)/1e6, 2) as not_due_m,
    ROUND(SUM(CASE WHEN DATEDIFF(CURRENT_DATE, due_date) BETWEEN 0 AND 30 THEN amount_usd ELSE 0 END)/1e6, 2) as d1_30_m,
    ROUND(SUM(CASE WHEN DATEDIFF(CURRENT_DATE, due_date) BETWEEN 31 AND 60 THEN amount_usd ELSE 0 END)/1e6, 2) as d31_60_m,
    ROUND(SUM(CASE WHEN DATEDIFF(CURRENT_DATE, due_date) BETWEEN 61 AND 90 THEN amount_usd ELSE 0 END)/1e6, 2) as d61_90_m,
    ROUND(SUM(CASE WHEN DATEDIFF(CURRENT_DATE, due_date) > 90 THEN amount_usd ELSE 0 END)/1e6, 2) as over_90_m,
    ROUND(SUM(amount_usd)/1e6, 2) as total_m
FROM ar_with_aging
GROUP BY OG
ORDER BY SUM(amount_usd) DESC;
```

---

## 7. Known Issues & Gotchas

### 7.1 Data Issues

1. **Snapshot Table**: F03B11 contains daily snapshots via `extraction_date`. Always filter to latest:
   ```sql
   WHERE extraction_date = (SELECT MAX(extraction_date) FROM lft.bronze.f03b11)
   ```

2. **IC AR in Wrong GL Class**: Large IC balances appear in `3RSU`/`8RSU`, not `2004`. Always exclude for external AR analysis.

3. **Forward AR Split**: `2004FRD` in Actual Cube is split between:
   - `4FRD` in subledger (partial)
   - GL-only entries (especially for Corporate)

4. **Accrued AR Not in Subledger**: `2004ACC` is booked directly in GL - no customer-level detail available in F03B11.

### 7.2 Reconciliation Gaps

| Gap Type | Typical Cause | Resolution |
|----------|---------------|------------|
| 2-7% variance | Timing (extraction vs month-end) | Acceptable |
| 10-15% variance | Missing GL classes or FX rates | Check `4FRD`, verify FX |
| Large variance on Corporate | FRD/ACC in GL only | Expected - use Actual Cube for Corporate |
| Missing companies | Different GL class or no AR subledger | Check what GL classes exist |

### 7.3 Intercompany Identification

IC customers can be identified by:
1. GL Class: `3RSU`, `8RSU`, `3RTD`
2. Customer name: Contains "LI & FUNG", "LF ", "IDS", "FUNG", "AIR8"

```sql
-- Exclude IC
WHERE ar.RPGLC IN ('2004', '4FRD')
  AND ar.RPALPH NOT LIKE '%LI & FUNG%'
  AND ar.RPALPH NOT LIKE 'LF %'
```

---

## 8. FX Rate Table Reference

**Table:** `lft.bronze.f0015`

| Column | Description |
|--------|-------------|
| `CXCRCD` | From currency |
| `CXCRDC` | To currency |
| `CXCRR` | Exchange rate |
| `CXEFT` | Effective date (Julian) |

**Common rates (as of Feb 2026):**
| Currency | Rate to USD |
|----------|-------------|
| HKD | 0.1289 |
| GBP | 1.3839 |
| EUR | 1.1882 |
| CNY | 0.1440 |
| SGD | 0.7928 |

---

## 9. Relationship to Other Data Sources

### 9.1 Data Source Hierarchy for AR Analysis

```
Question: "What's our AR exposure to Customer X?"
│
├─► Subledger (F03B11) - Invoice-level detail, aging, payment status
│   └─► Best for: Customer aging, collection tracking, dispute analysis
│
├─► Actual Cube (jedox_actual) - Account-level balances, includes GL-only items
│   └─► Best for: Reconciliation to TB, month-end reporting
│
├─► Plan/CC Cube - Management view with allocations
│   └─► Best for: Variance analysis, forecasting
│
└─► E1 Detail - Journal lines
    └─► Best for: Transaction tracing, cash receipts (RC docs)
```

### 9.2 When to Use Each Source

| Question | Use |
|----------|-----|
| Customer aging report | F03B11 subledger |
| AR balance for financial reporting | Actual Cube |
| AR vs Budget/Forecast variance | Plan/CC Cube |
| Trace specific cash receipt | E1 Detail (RC documents) |
| Customer credit exposure | F03B11 + customer master |

---

## Appendix A: Quick Reference

### A.1 Standard AR Query Template

```sql
WITH latest_fx AS (
    SELECT CXCRCD as from_ccy, CXCRR as rate_to_usd,
           ROW_NUMBER() OVER (PARTITION BY CXCRCD ORDER BY CXEFT DESC) as rn
    FROM lft.bronze.f0015 WHERE CXCRDC = 'USD'
),
fx_rates AS (SELECT from_ccy, rate_to_usd FROM latest_fx WHERE rn = 1)

SELECT 
    ar.RPCO as company,
    ar.RPALPH as customer,
    ou.ent3name as OG,
    ROUND(SUM(ar.RPAAP / 100 * COALESCE(fx.rate_to_usd, 1))/1e6, 2) as ar_usd_m
FROM lft.bronze.f03b11 ar
LEFT JOIN fx_rates fx ON ar.RPBCRC = fx.from_ccy
LEFT JOIN fna.fna_gold.jedox_dimpc_entity_detail d ON TRIM(ar.RPMCU) = d.child
LEFT JOIN fna.fna_gold.jedoxviewer_entity ou ON d.parent = ou.entb AND ou.ent1 = 'OG_Forecast'
WHERE ar.extraction_date = (SELECT MAX(extraction_date) FROM lft.bronze.f03b11)
  AND ar.RPPST = 'A'
  AND ar.RPGLC IN ('2004', '4FRD')
GROUP BY ar.RPCO, ar.RPALPH, ou.ent3name
ORDER BY SUM(ar.RPAAP) DESC;
```

### A.2 Conversion Checklist

- [ ] Filter `extraction_date` to latest snapshot
- [ ] Filter `RPPST = 'A'` for open items only
- [ ] Filter `RPGLC IN ('2004', '4FRD')` for trade AR
- [ ] Divide amounts by 100 (cents to dollars)
- [ ] Convert base currency to USD using f0015 rates
- [ ] Join via `RPMCU` for entity/OG mapping
- [ ] Exclude IC via GL class or customer name if needed

---

*Document created: Feb 2026*
*Validated against: jedox_actual Jan 2026 YTD*


# AR Aging Data Dictionary

This document covers the AR (Accounts Receivable) Aging data available for credit/collections analysis.

---

## Critical Interpretation

- `CURRENT_VENDOR_OS_AMT_USD` is back-to-back vendor AP linked to AR invoices; it is only a partial view of book AP, not a complete AP-aging substitute.
- No equivalent complete AP-aging table is currently documented. Use `F0411` with the currency and retention caveats in [AR_AP_SUBLEDGER.md](./AR_AP_SUBLEDGER.md) for AP detail.

## 1. Data Source

**Table:** `lft.bronze.dds_jrpt_factoring_ar_aging`

**Source System:** JRPT (JD Edwards Reporting) - factoring and AR aging report

**Refresh:** Monthly snapshots, partitioned by `AGING_PERIOD`

**Critical Understanding:** This table contains **invoiced AR only** - it excludes accrual/unbilled AR. Plan/CC Cube AR balances will always be higher because they include shipment accruals not yet invoiced.

---

## 2. Key Columns

### Identifiers & Dimensions

| Column | Description | Example |
|--------|-------------|---------|
| `AGING_PERIOD` | Snapshot month (YYYYMM) | `'202511'` |
| `BU` | Business Unit code (9 digits: 3-digit company + 6-digit unit) | `'001034603'` |
| `BU_DESC` | Business Unit description | `'AE1 Division'` |
| `COMPANY` | Legal company code (5 digits) | `'00001'`, `'00383'` |
| `COMPANY_NAME` | Legal company name | `'LI & FUNG TRADING LTD'` |
| `OPGRP_CODE` | Operating Group code (legacy) | `'SCS1'`, `'LFFA'`, `'LFMU'` |
| `STREAM_CODE` | Stream code | `'DEN'`, `'DSS'`, `'ABG'` |
| `DIVISION_CODE` | Division code | `'AE1'`, `'C5'`, `'LAT'` |
| `LFCORP_CODE` / `LFCORP_NAME` | Customer corporate code/name | `'AEOI'` / `'AMERICAN EAGLE OUTFITTERS INC.'` |
| `LFCUST_CODE` / `LFCUST_NAME` | Customer code/name (sub-customer level) | |

### AR Classification

| Column | Values | Description |
|--------|--------|-------------|
| `BUSINESS_NATURE` | `'Agency'`, `'TAP'`, `'-'` | Business model |
| `AR_NATURE` | See below | Insurance/factoring status |
| `IS_TCS` | `'Yes'`, `'No'` | Trade Credit Services eligible |
| `IS_ENTITLED_AR` | `'Yes'`, `'No'` | Entitled AR flag |
| `PAYSTS` | `'A'`, `'P'` | Payment status (A=Active/Open, P=Paid) |

**AR_NATURE values:**
- `'INSURED & FACTORED'`
- `'INSURED & UNFACTORED'`
- `'NON-INSURED & FACTORED'`
- `'NON-INSURED & UNFACTORED'`
- `'INSURED & REJECTED'`
- `'NON-INSURED & REJECTED'`
- `'FACTORING ADVANCE'`

### Amount Columns (USD)

| Column | Description |
|--------|-------------|
| `OPEN_AMT_USD` | Total open AR amount |
| `NOT_DUE_USD` | Not yet due |
| `DAYS_1_30_USD` | 1-30 days past due |
| `DAYS_31_60_USD` | 31-60 days past due |
| `DAYS_61_90_USD` | 61-90 days past due |
| `DAYS_91_120_USD` | 91-120 days past due |
| `DAYS_121_180_USD` | 121-180 days past due |
| `DAYS_181_330_USD` | 181-330 days past due |
| `DAYS_331_365_USD` | 331-365 days past due |
| `DAYS_OVER365_USD` | Over 365 days past due |

**Note:** HKD equivalents also available (`NOT_DUE_HKD`, etc.)

### Invoice Details

| Column | Description |
|--------|-------------|
| `INVOICE_NO` | Invoice number |
| `INV_DAT` | Invoice date |
| `DUE_DAT` | Due date |
| `DUE_DAYS` | Days past due |
| `PAYTERM` / `PAYTERM_DESC` | Payment terms |
| `DOC_NO` / `DOC_TYPE` | Document number and type |

### Credit & Factoring

| Column | Description |
|--------|-------------|
| `CREDIT_RATING` | Customer credit rating |
| `CREDIT_SCORE` / `COMBINED_SCORE` | Credit scores |
| `FACTORING_HOUSE` / `FACTORING_HOUSE_NAME` | Factoring bank |
| `INSURANCE_COMPANY` | Credit insurance provider |
| `CREDIT_LIMIT_*` | Credit limits by insurer (EHH, QA, GEG, ECIC, etc.) |

---

## 3. Joining to Plan/CC Entity Hierarchy (Primary Approach)

Use `BU` to join AR Aging data to Jedox entity hierarchy. This ensures consistency with Plan/CC Cube reporting.

### Join Pattern

```sql
SELECT 
    ou.ent3name as OG,
    ou.ent4name as Stream,
    ou.ent6name as Division,
    ar.*
FROM lft.bronze.dds_jrpt_factoring_ar_aging ar
LEFT JOIN fna.fna_gold.jedox_dimpc_entity_detail ed 
    ON ar.BU = ed.child
LEFT JOIN fna.fna_gold.jedoxviewer_entity ou 
    ON ed.parent = ou.entb 
    AND ou.ent1 = 'OG_Forecast'
WHERE ar.AGING_PERIOD = '202511'
```

### Standard Query by OG

```sql
SELECT 
    ou.ent3name as OG,
    ROUND(SUM(ar.OPEN_AMT_USD)/1000000, 1) as Total_AR_M,
    ROUND(SUM(ar.NOT_DUE_USD)/1000000, 1) as Not_Due_M,
    ROUND(SUM(ar.DAYS_1_30_USD + ar.DAYS_31_60_USD + ar.DAYS_61_90_USD)/1000000, 1) as D1_90_M,
    ROUND(SUM(ar.DAYS_91_120_USD + ar.DAYS_121_180_USD + ar.DAYS_181_330_USD 
              + ar.DAYS_331_365_USD + ar.DAYS_OVER365_USD)/1000000, 1) as Over_90_M
FROM lft.bronze.dds_jrpt_factoring_ar_aging ar
LEFT JOIN fna.fna_gold.jedox_dimpc_entity_detail ed 
    ON ar.BU = ed.child
LEFT JOIN fna.fna_gold.jedoxviewer_entity ou 
    ON ed.parent = ou.entb 
    AND ou.ent1 = 'OG_Forecast'
WHERE ar.AGING_PERIOD = '202511'
  AND ou.ent2name IN ('SCS', 'Markets')
  AND (ar.GLCLASS LIKE '2%' OR TRIM(ar.GLCLASS) = '')
GROUP BY ou.ent3name
ORDER BY Total_AR_M DESC;
```

### OPGRP_CODE Reference (Legacy)

The `OPGRP_CODE` column in AR Aging is a legacy field. Use for quick filtering only when BU join is not needed.

| OPGRP_CODE | Meaning | Plan/CC `ent3name` |
|------------|---------|-------------------|
| `SCS1` | Apparel | `Apparel` |
| `SCS2` | Home & Accessories | `Home And Accessories` |
| `LFMU` | LF Markets USA | `OG: LF Markets USA` |
| **`LFFA`** | **LF Europe** | `OG: LF Europe` |
| **`LFEU`** | **Promocean** | `OG: Promocean and EU Others` |
| `FWK` | Firework | `OG: Firework` |
| `LFAD` | LF Asia Direct | `OG: LF Asia Direct` |

**WARNING:** LFFA and LFEU codes are **swapped** from what you'd expect. This is a historical artifact.

---

## 4. GLCLASS (GL Account Classification)

The `GLCLASS` column categorizes AR by GL account type. Key classifications:

| GLCLASS | Type | Description | Notes |
|---------|------|-------------|-------|
| `2004` | **External Trade AR** | Standard third-party customer receivables | Primary AR type for all OGs |
| `3RTD` | Intercompany Trade | AR from LF Group internal entities | Internal LF: WHALEN, MILES GMBH, COBALT |
| `3RSU` | Intercompany Suspense | Suspense/clearing within LF Group | Often nets to ~zero |
| `3FRD` | Factoring Advance | Advances received from factoring house | Negative amounts (liability offset) |
| `4TX2` | **Air8 Trade AR** | Trade AR booked via Air8 (Company 383) | 100% from Air8 PTE. LTD |
| `4FRD` | TAP Factored AR | TAP AR that has been factored | All TAP, INSURED & FACTORED |
| `8RTD` | Related Co Trade | AR from related companies (cousins) | FCSG, Holdings (BVI), etc. |
| `8RSU` | Related Co Suspense | Suspense/clearing with related companies | Cousin company AR |
| `6ADV` | Advances | Customer advances/deposits | Usually negative (prepayments) |
| `(blank)` | Unclassified | Missing classification | Various |

**Key distinction:**
- **3xxx (Intercompany):** AR within **LF Group internal entities** (same consolidated group)
- **8xxx (Related Company):** AR with **related/cousin companies** (separate but affiliated entities)
- **4TX2 (Air8):** External customer AR but booked via Air8 legal entity - typically **excluded** from standard 3rd-party AR reporting

### Filtering by GLCLASS

**External/Third-Party AR (Recommended Blanket Rule):**
```sql
WHERE (GLCLASS LIKE '2%' OR TRIM(GLCLASS) = '')
```

This rule matches the "Open Amt Usd (AR)" column in AR dashboards within ~1% accuracy across all OGs:

| OG | Dashboard Target ($K) | Rule Result ($K) | Variance |
|----|----------------------|------------------|----------|
| SCS1 (Apparel) | 236,373 | 238,220 | +0.8% |
| LFMU | 105,228 | 104,549 | -0.6% |
| LFFA (LF Europe) | 94,337 | 94,215 | -0.1% |

**What it includes:**
- `2004` - External Trade AR (primary)
- `2001` - External Trade AR
- `2005` - External Trade AR
- Blank GLCLASS - typically R5 doc type (receipts/adjustments)

**What it excludes:**
- `3xxx` - Intercompany AR
- `4TX2` - Air8 Trade AR (external but via Air8 legal entity)
- `4FRD` - TAP Factored AR
- `6ADV` - Customer Advances
- `8xxx` - Related Company AR

**All AR including intercompany:**
```sql
-- No GLCLASS filter needed
```

---

## 5. DOC_TYPE (Document Type)

The `DOC_TYPE` column indicates how the AR was created:

| DOC_TYPE | Description | Include in Standard AR? |
|----------|-------------|------------------------|
| `UX` | Sales Invoice from XTS | **Yes** - primary invoice type |
| `UA` | Sales Invoice from XTS (Repost) | **Yes** - reposted invoices |
| `UU` | Credit Memo - XTS/GR/VMS Repost | **Yes** - system credit memos |
| `UM` | Manual Invoice | **No** - excluded from standard reporting |
| `UH` | Debit Memo | Varies |
| `UD` | Debit Note (XTS) | Varies |
| `UC` | Credit Note (XTS) | Varies |
| `UG` | Credit Memo | Varies |
| `UR` | Advanced Receipt | Usually negative |

### DOC_TYPE Distribution by OG (Nov 2025, GLCLASS 2004)

| OG | UX (XTS Invoice) | UA (Repost) | UM (Manual) | Other |
|----|------------------|-------------|-------------|-------|
| **SCS1** | 230,321 (97%) | 5,861 | 1,294 | 191 |
| **SCS2** | 93,906 (97%) | 2,416 | 336 | 240 |
| **LFMU** | 99,583 (98%) | 1,706 | 55 | (13) |
| **LFFA** | 82,017 (88%) | 2,787 | 11,192 | (2,695) |
| **FWK** | 9,906 (128%) | - | 176 | (2,415) |
| **LFAD** | - | - | 10,938 | (2,627) |

**Key insights:**
- **SCS (Apparel/H&A):** 97%+ is XTS-generated invoices (UX) - highly automated
- **LFMU:** Similar to SCS - 98% XTS invoices
- **LFFA (LF Europe):** 12% Manual Invoice (UM) - more manual processes
- **LFAD:** 100% Manual Invoice - no XTS integration
- **FWK (Firework):** Significant credit notes (UC) due to retail returns

### Standard "Invoiced AR" Filter

To match the "Open Amt Usd (AR)" column in AR dashboards, use the blanket rule:

```sql
WHERE (GLCLASS LIKE '2%' OR TRIM(GLCLASS) = '')
```

This works across all OGs within ~1% accuracy. For exact matching on specific OGs, additional DOC_TYPE filters may be needed (e.g., SCS excludes `DOC_TYPE = 'UM'`).

---

## 6. Reconciliation to Plan/CC Cube

### Why JRPT AR < Plan/CC AR

The AR Aging table will **always be lower** than Plan/CC Cube AR (`ac.level5 = 'WC_AR'`) because:

1. **Invoiced vs Accrual:** JRPT only includes **invoiced AR**; Plan/CC includes shipment accruals (goods shipped but not yet invoiced)
2. **Air8 (4TX2):** Typically excluded from standard 3rd-party AR reporting in JRPT dashboards
3. **Provisions:** Plan/CC includes bad debt provisions; AR Aging is gross
4. **Timing:** Different extraction vs month-end close dates

### Expected Gaps (Nov 2025)

| Component | Apparel Example |
|-----------|-----------------|
| **JRPT Invoiced AR (GLCLASS 2004, excl UM)** | $236M |
| + Shipment Accruals (not in JRPT) | ~$73M |
| + Air8 AR (GLCLASS 4TX2) | ~$66M |
| = **Total AR (closer to Plan/CC)** | ~$375M |

**Recommendation:** Do NOT try to reconcile JRPT to Plan/CC exactly. Use JRPT for credit/collections analysis (invoiced AR), use Plan/CC for financial reporting (total AR including accruals).

---

## 7. Standard Queries

### AR Aging by Customer

```sql
SELECT 
    ar.LFCORP_NAME as Customer,
    ou.ent3name as OG,
    ROUND(SUM(ar.OPEN_AMT_USD)/1000000, 1) as Total_M,
    ROUND(SUM(ar.NOT_DUE_USD)/1000000, 1) as Not_Due_M,
    ROUND(SUM(ar.DAYS_1_30_USD)/1000000, 1) as D1_30_M,
    ROUND(SUM(ar.DAYS_31_60_USD + ar.DAYS_61_90_USD)/1000000, 1) as D31_90_M,
    ROUND(SUM(ar.DAYS_91_120_USD + ar.DAYS_121_180_USD + ar.DAYS_181_330_USD 
              + ar.DAYS_331_365_USD + ar.DAYS_OVER365_USD)/1000000, 1) as Over_90_M
FROM lft.bronze.dds_jrpt_factoring_ar_aging ar
LEFT JOIN fna.fna_gold.jedox_dimpc_entity_detail ed 
    ON ar.BU = ed.child
LEFT JOIN fna.fna_gold.jedoxviewer_entity ou 
    ON ed.parent = ou.entb 
    AND ou.ent1 = 'OG_Forecast'
WHERE ar.AGING_PERIOD = '202511'
  AND ou.ent3name = 'Apparel'
  AND (ar.GLCLASS LIKE '2%' OR TRIM(ar.GLCLASS) = '')
GROUP BY ar.LFCORP_NAME, ou.ent3name
ORDER BY Total_M DESC
LIMIT 20;
```

### Overdue AR Analysis (>90 days)

```sql
SELECT 
    ar.LFCORP_NAME as Customer,
    ou.ent3name as OG,
    ROUND(SUM(ar.DAYS_91_120_USD + ar.DAYS_121_180_USD + ar.DAYS_181_330_USD 
              + ar.DAYS_331_365_USD + ar.DAYS_OVER365_USD)/1000000, 2) as Over_90_M
FROM lft.bronze.dds_jrpt_factoring_ar_aging ar
LEFT JOIN fna.fna_gold.jedox_dimpc_entity_detail ed 
    ON ar.BU = ed.child
LEFT JOIN fna.fna_gold.jedoxviewer_entity ou 
    ON ed.parent = ou.entb 
    AND ou.ent1 = 'OG_Forecast'
WHERE ar.AGING_PERIOD = '202511'
  AND ou.ent2name = 'SCS'
  AND (ar.GLCLASS LIKE '2%' OR TRIM(ar.GLCLASS) = '')
GROUP BY ar.LFCORP_NAME, ou.ent3name
HAVING SUM(ar.DAYS_91_120_USD + ar.DAYS_121_180_USD + ar.DAYS_181_330_USD 
           + ar.DAYS_331_365_USD + ar.DAYS_OVER365_USD) > 1000000  -- >$1M overdue
ORDER BY Over_90_M DESC;
```

### Reproduce Dashboard "Open Amt Usd (AR)" Column

**Blanket rule (works across all OGs within ~1%):**

```sql
SELECT 
    ou.ent3name as OG,
    ROUND(SUM(ar.OPEN_AMT_USD)/1000, 0) as Open_AR_K 
FROM lft.bronze.dds_jrpt_factoring_ar_aging ar
LEFT JOIN fna.fna_gold.jedox_dimpc_entity_detail ed 
    ON ar.BU = ed.child
LEFT JOIN fna.fna_gold.jedoxviewer_entity ou 
    ON ed.parent = ou.entb 
    AND ou.ent1 = 'OG_Forecast'
WHERE ar.AGING_PERIOD = '202511' 
  AND ou.ent2name IN ('SCS', 'Markets')
  AND (ar.GLCLASS LIKE '2%' OR TRIM(ar.GLCLASS) = '')
GROUP BY ou.ent3name
ORDER BY Open_AR_K DESC;
```

### Check Available Periods

```sql
SELECT DISTINCT AGING_PERIOD 
FROM lft.bronze.dds_jrpt_factoring_ar_aging 
WHERE AGING_PERIOD LIKE '2025%'
ORDER BY AGING_PERIOD DESC;
```

---

## 8. Use Cases

| Use Case | Recommended Approach |
|----------|---------------------|
| Customer credit risk | Filter by `LFCORP_NAME`, analyze aging buckets |
| Collection priority | Sort by `Over_90` days, focus on large overdue |
| Factoring analysis | Filter by `AR_NATURE`, check `FACTORING_HOUSE` |
| Insurance coverage | Check `IS_TCS`, `INSURANCE_COMPANY`, credit limits |
| Trend analysis | Compare across `AGING_PERIOD` values |
| OG-level analysis | Join via BU to `jedoxviewer_entity`, filter by `ent3name` |
| Match dashboard metrics | Use `GLCLASS LIKE '2%' OR TRIM(GLCLASS) = ''` |

---

## 9. Notes on BU Join

**Company 00383 (Air8):** BUs from Air8 will map to LFX/Corporate in Jedox. This is correct behavior - Air8 AR (GLCLASS 4TX2) is typically excluded from external 3rd-party AR reporting via the GLCLASS filter.

**Unmapped BUs:** Some BUs may not have a mapping in `jedox_dimpc_entity_detail`. These will have NULL values for entity fields after the LEFT JOIN. Consider filtering with `WHERE ou.entb IS NOT NULL` if you want only mapped records.

---

*Document updated: Dec 2025*

