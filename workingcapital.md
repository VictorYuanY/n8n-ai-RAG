# Working Capital Dashboard Guide

This guide explains the working capital dataset behind the Tableau dashboard, how each metric is defined, and how to answer common business questions with SQL.

The main dataset is:

```sql
fna.fna_bronze.working_capital_metrics
```

The materialized view SQL is maintained separately in:

```text
C:\Users\VictorYuan\OneDrive - Li & Fung\Victor Analysis\2026 Project\Working Capital Project\working_capital_metrics.txt
```

It is designed as a Tableau-friendly long table. Each row is one metric value for one period, one operating group, one working capital part, and one month selection.

## 1. Purpose

The dashboard helps users understand:

- How much working capital is tied up by operating group.
- Which parts drive working capital: AR, AP, Inventory, Factoring, OROP, and LF Credit.
- How actual performance compares with Budget, 1QR, and 2QR.
- How working capital efficiency is moving through DSO, DPO, and DIO.

The dataset can also be queried directly to answer ad hoc questions such as:

- What is Apparel March 2026 Total WC?
- What is LFMU YTD May DIO?
- What is Total OG Year End AR vs Budget?
- What are the SCS and Markets subtotals by month?

## 2. Dataset Grain

The table is at this grain:

```text
period + og + wc_part + unit + month
```

Main columns:

| Column | Meaning |
|---|---|
| `period` | Scenario/year, such as `2026 Actual`, `2026 Budget`, `2026 1QR`, `2026 2QR`, `2025 Actual` |
| `og` | Operating group or subtotal, such as `Apparel`, `SCS`, `Markets`, `Total OG` |
| `wc_part` | Working capital part or efficiency metric |
| `unit` | `USD M` for balance metrics, `days` for DSO/DPO/DIO |
| `month` | `Jan` to `Dec`, plus `Average` and `Year End` |
| `month_sort` | Sort helper for Tableau |
| `value` | Metric value |

## 3. Periods

The dataset currently includes:

| Period | Meaning |
|---|---|
| `2025 Actual` | Prior-year actual |
| `2026 Actual` | Current-year actual |
| `2026 Budget` | 2026 budget |
| `2026 1QR` | First quarter reforecast |
| `2026 2QR` | Second quarter reforecast |

For current-year actual, future months are returned as `0` in the dataset. For line charts, Tableau can use a calculated field to show future actual months as `NULL` so the line does not drop to zero.

## 4. Operating Groups

Base OGs:

| OG | Definition |
|---|---|
| `Apparel` | Jedox entity group where `ent3name = 'Apparel'` |
| `Home and Accessories` | Jedox entity group where `ent3name = 'Home And Accessories'` |
| `LFMU` | Jedox entity group where `ent3name = 'OG: LF Markets USA'` |
| `LFFA` | Residual LF Europe, excluding Miles, PromOcean, and Orrsum |
| `Miles` | Entity where `ent4name = 'Miles'` |
| `PromOcean` | Entity where `ent4name = 'PromOcean'` |
| `Orrsum` | Entity where `ent4name = 'Orrsum'` |
| `LFAD` | Entity where `ent3name = 'OG: LF Asia Direct'` |
| `Firework` | Entity where `ent3name = 'OG: Firework'` |

Subtotal OGs:

| OG | Definition |
|---|---|
| `SCS` | `Apparel + Home and Accessories` |
| `Markets` | `LFMU + LFFA + Miles + PromOcean + Orrsum + LFAD + Firework` |
| `Total OG` | All base OGs included in this dashboard |

Important: If a chart shows base OGs together with `SCS`, `Markets`, or `Total OG`, those subtotal bars are not additive with the base OG bars. They are rollups.

## 5. Working Capital Parts

The balance metrics use signed Jedox values. The dashboard does not reverse signs manually; it follows the source sign convention.

| WC Part | Business Meaning | Account Codes |
|---|---|---|
| `AR` | Trade receivables and AR-related provisions | `B203000`, `B204000`, `B205000`, `B206000`, `B206100`, `B207000`, `B207100` |
| `AP` | Trade payables and related company payables | `B302000`, `B303000` |
| `Inventory` | Inventory balance | `B201000` |
| `Factoring` | Factoring drawdown / AR financing balance | `B208000` |
| `OROP` | Other receivables and other payables combined | `B209000`, `B305000` |
| `LF Credit` | AP financing / LFX payable settlement balance | `B304000` |
| `Total WC` | Sum of AR, AP, Inventory, Factoring, OROP, and LF Credit | Calculated |

## 6. AR Definition

AR includes these accounts:

| Account Description | Account Code |
|---|---|
| Accounts Receivable | `B203000` |
| Related Co Receivable (Trade) | `B204000` |
| Prov for dilution | `B205000` |
| Prov for BD (Specific)-AR | `B206000` |
| Prov for BD related co (Specific)-AR | `B206100` |
| Prov for BD (General)-AR | `B207000` |
| Prov for BD related co (General)-AR | `B207100` |

The provisions are included in AR so that AR reflects the net receivable position used for working capital reporting.

## 7. AP Definition

AP includes:

| Account Description | Account Code |
|---|---|
| Accounts Payable | `B302000` |
| Related Co Payable | `B303000` |

LF Credit is not included inside AP. It is shown separately because it represents AP-related financing through LFX rather than direct supplier payable.

## 8. Factoring

Factoring represents AR financing.

Business interpretation:

```text
The company sells or finances receivables with a bank/factor and receives cash earlier.
```

In the dashboard, factoring is shown separately from AR because it is not normal customer receivable balance. It is a financing balance connected to receivables.

Source account:

```text
B208000 = Factoring Drawdown
```

Typical business flow:

```text
Receivable is financed or sold
Company receives cash earlier
Factoring balance tracks the related drawdown / financing position
```

## 9. LF Credit

LF Credit represents AP financing through LFX.

Business interpretation:

```text
LFX pays suppliers on behalf of the business.
The supplier payable is settled or transferred.
The business then owes LFX.
```

Source account:

```text
B304000 = LF Credit settlement
```

It is connected to AP economically, but it is shown separately because users should be able to distinguish:

```text
Normal supplier AP
vs.
AP financed or settled through LFX
```

## 10. Total WC

Total WC is calculated as:

```text
Total WC = AR + AP + Inventory + Factoring + OROP + LF Credit
```

The calculation uses signed source values. This means AP or financing-related balances may reduce or increase Total WC depending on the sign convention in Jedox.

## 11. Month Logic

The dataset has normal months:

```text
Jan, Feb, Mar, Apr, May, Jun, Jul, Aug, Sep, Oct, Nov, Dec
```

It also has two special month selections:

```text
Average
Year End
```

### Average

For current-year actual:

```text
Average = average of months from Jan through the current calendar month
```

Example: if today is in June 2026, `2026 Actual` Average uses Jan-Jun divided by 6.

For other periods/scenarios:

```text
Average = Jan-Dec average
```

### Year End

For current-year actual:

```text
Year End = latest closed month-end value
```

Example: if today is June 25, 2026, the latest closed month is May, so `2026 Actual` Year End uses May.

For prior years and plan scenarios:

```text
Year End = Dec value
```

## 12. DSO, DPO, and DIO Overview

DSO, DPO, and DIO are calculated from Jedox sequence tables. The dashboard does not calculate these by directly dividing AR, AP, or Inventory by simple monthly sales/COGS. Instead, it uses prebuilt sequence buckets.

Source tables:

| Metric | Source Table |
|---|---|
| `DSO` | `fna.fna_gold.jedoxviewer_dso_sequences` |
| `DPO` | `fna.fna_gold.jedoxviewer_dpo_sequences` |
| `DIO` | `fna.fna_gold.jedoxviewer_dio_sequences` |

Each sequence table provides:

```text
balance
P1, P2, P3, ... P12
```

The SQL applies a waterfall calculation to convert the balance and buckets into days.

## 13. DSO Formula

DSO answers:

```text
How many days of sales are represented by the current AR balance?
```

Conceptual logic:

```text
Start with current month AR balance.
Compare it against the current month sales bucket.
If AR is not fully covered, move back to the prior month sales bucket.
Continue moving backward until the AR balance is fully covered.
Each fully used bucket adds 30 days.
The final partially used bucket adds a prorated number of days.
```

Formula:

```text
DSO = 30 * number of fully covered buckets
      + remaining AR balance / final bucket amount * 30
```

Example:

```text
Current AR balance = 130
Current month sales bucket = 100
Prior month sales bucket = 100

First 100 is fully covered by current month sales = 30 days
Remaining 30 is covered by 30% of prior month sales = 9 days

DSO = 30 + 9 = 39 days
```

The dashboard uses the DSO sequence table as the source of truth for the AR balance and sales buckets used in this waterfall.

## 14. DPO Formula

DPO answers:

```text
How many days of purchases/cost are represented by the current AP balance?
```

Conceptual logic:

```text
Start with current month AP balance.
Compare it against the current month purchase/cost bucket.
If AP is not fully covered, move back to the prior month bucket.
Continue moving backward until the AP balance is fully covered.
Each fully used bucket adds 30 days.
The final partially used bucket adds a prorated number of days.
```

Formula:

```text
DPO = 30 * number of fully covered buckets
      + remaining AP balance / final bucket amount * 30
```

Example:

```text
Current AP balance = 180
Current month cost bucket = 120
Prior month cost bucket = 100

First 120 is fully covered = 30 days
Remaining 60 is 60% of prior month bucket = 18 days

DPO = 30 + 18 = 48 days
```

The dashboard uses the DPO sequence table as the source of truth for the AP balance and purchase/cost buckets used in this waterfall.

## 15. DIO Formula

DIO answers:

```text
How many days of future consumption are covered by current inventory?
```

DIO is different from DSO and DPO because it looks forward, not backward.

Conceptual logic:

```text
Start with current month inventory balance.
Compare it against the next future consumption/COGS bucket.
If inventory is not fully consumed, move forward to the next future bucket.
Continue moving forward until the inventory balance is fully consumed.
Each fully used future bucket adds 30 days.
The final partially used bucket adds a prorated number of days.
```

Formula:

```text
DIO = 30 * number of fully covered future buckets
      + remaining inventory balance / final future consumption bucket * 30
```

Example:

```text
Current inventory balance = 150
Next month consumption bucket = 100
Following month consumption bucket = 100

First 100 is fully covered = 30 days
Remaining 50 is 50% of following month bucket = 15 days

DIO = 30 + 15 = 45 days
```

Special rule for Actual DIO:

```text
Actual DIO balance = Actual inventory balance
Forward consumption buckets = Budget buckets
```

This rule was needed because actual future consumption is not fully available for future months, and it ties to checked examples such as LFMU and Miles.

## 16. Waterfall Calculation Used in SQL

The same waterfall structure is used for DSO, DPO, and DIO.

For each bucket:

```text
previous cumulative amount = sum of all prior buckets
current bucket amount = P bucket amount
balance = AR / AP / Inventory balance from sequence table
```

Days contributed by each bucket:

```text
If balance <= previous cumulative amount:
    0 days

If balance > previous cumulative amount + current bucket amount:
    30 days

If balance is partially covered by current bucket:
    (balance - previous cumulative amount) / current bucket amount * 30
```

Total days:

```text
sum of bucket day contributions
```

## 17. Important Validation Points

Examples that were checked during build:

| Metric | Period | OG | Month | Expected Value |
|---|---|---|---|---|
| `DIO` | `2026 Actual` | `LFMU` | `Mar` | `42.6 days` |
| `DPO` | `2026 Actual` | `LFMU` | `May` | `67.8 days` |
| `DIO` | `2026 Actual` | `LFMU` | `May` | `21.5 days` |
| `DPO` | `2026 Actual` | `Home and Accessories` | `May` | `18.6 days` |
| `DIO` | `2026 Actual` | `Miles` | `May` | `31.6 days` |

If future changes break these anchors, review the DSO/DPO/DIO source tables and DIO Actual/Budget forward bucket logic first.

## 18. Tableau Usage

Recommended dashboard title:

```text
Working Capital Overview
```

Recommended default filters:

```text
period = 2026 Actual
og = Total OG
month = latest closed month or Year End
```

Recommended dashboard sections:

| Section | Chart |
|---|---|
| Top | KPI cards for Total WC, AR, AP, Inventory, DSO, DPO, DIO |
| Middle | Total WC trend line: Actual vs Budget / 1QR / 2QR |
| Middle | Working capital composition by WC part |
| Bottom | Total WC by OG bar chart |
| Bottom | DSO/DPO/DIO trend |

For monthly trend charts, filter:

```text
month_sort <= 12
```

This removes `Average` and `Year End` from monthly trend charts.

For KPI cards, use:

```text
month = Year End
```

or use the latest closed month directly.

## 19. Common Tableau Setup

### Total WC Trend: Actual vs Plan

Use:

```text
Columns: month
Rows: SUM(value)
Color: period
Marks: Line
```

Filters:

```text
wc_part = Total WC
og = selected OG
period = 2026 Actual, 2026 Budget, 2026 1QR, 2026 2QR
month_sort <= 12
```

Sort `month` by `month_sort`.

### Working Capital Composition

Use:

```text
Columns: wc_part
Rows: SUM(value)
Color: wc_part
Marks: Bar
```

Filters:

```text
period = selected period
og = selected OG
month = selected month or Year End
wc_part = AR, AP, Inventory, Factoring, OROP, LF Credit
```

### Total WC by OG

Use:

```text
Columns: og
Rows: SUM(value)
Marks: Bar
```

Filters:

```text
wc_part = Total WC
period = selected period
month = selected month or Year End
```

If showing `Total OG`, format it with a distinct color. Remember that `SCS`, `Markets`, and `Total OG` are subtotals.

### DSO/DPO/DIO Trend

Use:

```text
Columns: month
Rows: SUM(value)
Color: wc_part
Marks: Line
```

Filters:

```text
period = selected period
og = selected OG
wc_part = DSO, DPO, DIO
month_sort <= 12
unit = days
```

## 20. Showing Future Actual Months as Null in Tableau

The SQL returns future current-year Actual months as `0`. For line charts, it is often better to hide those future points.

Create a Tableau calculated field:

```text
Actual Chart Value
```

Formula:

```tableau
IF [period] = STR(YEAR(TODAY())) + " Actual"
   AND [month_sort] <= 12
   AND [month_sort] > DATEPART('month', DATEADD('month', -1, TODAY()))
THEN NULL
ELSE [value]
END
```

Use `SUM(Actual Chart Value)` instead of `SUM(value)` for trend charts.

## 21. Common SQL Patterns

### Get one metric

Example: 2026 Actual Apparel March Total WC.

```sql
SELECT
  period,
  og,
  wc_part,
  unit,
  month,
  value
FROM fna.fna_bronze.working_capital_metrics
WHERE period = '2026 Actual'
  AND og = 'Apparel'
  AND wc_part = 'Total WC'
  AND month = 'Mar';
```

### Get all WC parts for one OG and month

Example: Apparel March 2026 Actual.

```sql
SELECT
  wc_part,
  unit,
  value
FROM fna.fna_bronze.working_capital_metrics
WHERE period = '2026 Actual'
  AND og = 'Apparel'
  AND month = 'Mar'
  AND wc_part IN ('AR', 'AP', 'Inventory', 'Factoring', 'OROP', 'LF Credit', 'Total WC')
ORDER BY
  CASE wc_part
    WHEN 'AR' THEN 1
    WHEN 'AP' THEN 2
    WHEN 'Inventory' THEN 3
    WHEN 'Factoring' THEN 4
    WHEN 'OROP' THEN 5
    WHEN 'LF Credit' THEN 6
    WHEN 'Total WC' THEN 7
  END;
```

### Compare Actual vs Budget/QR

Example: Total OG Total WC by month.

```sql
SELECT
  period,
  month,
  month_sort,
  value
FROM fna.fna_bronze.working_capital_metrics
WHERE period IN ('2026 Actual', '2026 Budget', '2026 1QR', '2026 2QR')
  AND og = 'Total OG'
  AND wc_part = 'Total WC'
  AND month_sort <= 12
ORDER BY
  period,
  month_sort;
```

### Get KPI values

Example: Total OG Year End KPI cards.

```sql
SELECT
  wc_part,
  unit,
  value
FROM fna.fna_bronze.working_capital_metrics
WHERE period = '2026 Actual'
  AND og = 'Total OG'
  AND month = 'Year End'
  AND wc_part IN ('Total WC', 'AR', 'AP', 'Inventory', 'DSO', 'DPO', 'DIO')
ORDER BY
  CASE wc_part
    WHEN 'Total WC' THEN 1
    WHEN 'AR' THEN 2
    WHEN 'AP' THEN 3
    WHEN 'Inventory' THEN 4
    WHEN 'DSO' THEN 5
    WHEN 'DPO' THEN 6
    WHEN 'DIO' THEN 7
  END;
```

### Get Average and Year End

```sql
SELECT
  period,
  og,
  wc_part,
  unit,
  month,
  value
FROM fna.fna_bronze.working_capital_metrics
WHERE period IN ('2026 Actual', '2026 Budget', '2026 1QR', '2026 2QR')
  AND og = 'Total OG'
  AND wc_part = 'Total WC'
  AND month IN ('Average', 'Year End')
ORDER BY
  period,
  month_sort;
```

## 22. Runnable Databricks SQL Templates

These SQL templates can be copied into a Databricks SQL editor or passed to the local Databricks query tool.

Local tool pattern:

```powershell
python C:\Users\VictorYuan\Opencode\BD_Chatbot_prompt\run_db.py "<SQL HERE>"
```

### Dashboard KPI Cards

Use this for the top KPI cards: Total WC, AR, AP, Inventory, DSO, DPO, and DIO.

```sql
SELECT
  wc_part,
  unit,
  value
FROM fna.fna_bronze.working_capital_metrics
WHERE period = '2026 Actual'
  AND og = 'Total OG'
  AND month = 'Year End'
  AND wc_part IN ('Total WC', 'AR', 'AP', 'Inventory', 'DSO', 'DPO', 'DIO')
ORDER BY
  CASE wc_part
    WHEN 'Total WC' THEN 1
    WHEN 'AR' THEN 2
    WHEN 'AP' THEN 3
    WHEN 'Inventory' THEN 4
    WHEN 'DSO' THEN 5
    WHEN 'DPO' THEN 6
    WHEN 'DIO' THEN 7
  END;
```

### Total WC Trend: Actual vs Plan

Use this for the line chart showing Actual vs Budget / 1QR / 2QR.

```sql
SELECT
  period,
  month,
  month_sort,
  value
FROM fna.fna_bronze.working_capital_metrics
WHERE period IN ('2026 Actual', '2026 Budget', '2026 1QR', '2026 2QR')
  AND og = 'Total OG'
  AND wc_part = 'Total WC'
  AND month_sort <= 12
ORDER BY
  CASE period
    WHEN '2026 Actual' THEN 1
    WHEN '2026 Budget' THEN 2
    WHEN '2026 1QR' THEN 3
    WHEN '2026 2QR' THEN 4
  END,
  month_sort;
```

### Total WC Trend With Future Actual Months as Null

Use this version if a line chart should not drop to zero for future Actual months.

```sql
SELECT
  period,
  month,
  month_sort,
  CASE
    WHEN period = concat(CAST(year(current_date()) AS STRING), ' Actual')
      AND month_sort <= 12
      AND month_sort > month(add_months(current_date(), -1))
    THEN NULL
    ELSE value
  END AS value
FROM fna.fna_bronze.working_capital_metrics
WHERE period IN ('2026 Actual', '2026 Budget', '2026 1QR', '2026 2QR')
  AND og = 'Total OG'
  AND wc_part = 'Total WC'
  AND month_sort <= 12
ORDER BY
  CASE period
    WHEN '2026 Actual' THEN 1
    WHEN '2026 Budget' THEN 2
    WHEN '2026 1QR' THEN 3
    WHEN '2026 2QR' THEN 4
  END,
  month_sort;
```

### Working Capital Composition

Use this for the bar chart showing which WC parts drive Total WC.

```sql
SELECT
  wc_part,
  unit,
  value
FROM fna.fna_bronze.working_capital_metrics
WHERE period = '2026 Actual'
  AND og = 'Total OG'
  AND month = 'Year End'
  AND wc_part IN ('AR', 'AP', 'Inventory', 'Factoring', 'OROP', 'LF Credit')
ORDER BY
  CASE wc_part
    WHEN 'AR' THEN 1
    WHEN 'AP' THEN 2
    WHEN 'Inventory' THEN 3
    WHEN 'Factoring' THEN 4
    WHEN 'OROP' THEN 5
    WHEN 'LF Credit' THEN 6
  END;
```

### Total WC by OG

Use this for the bottom bar chart by operating group.

```sql
SELECT
  og,
  unit,
  value
FROM fna.fna_bronze.working_capital_metrics
WHERE period = '2026 Actual'
  AND wc_part = 'Total WC'
  AND month = 'Year End'
  AND og IN (
    'Apparel',
    'Home and Accessories',
    'SCS',
    'LFMU',
    'LFFA',
    'Miles',
    'PromOcean',
    'Orrsum',
    'LFAD',
    'Firework',
    'Markets',
    'Total OG'
  )
ORDER BY
  CASE og
    WHEN 'Apparel' THEN 1
    WHEN 'Home and Accessories' THEN 2
    WHEN 'SCS' THEN 3
    WHEN 'LFMU' THEN 4
    WHEN 'LFFA' THEN 5
    WHEN 'Miles' THEN 6
    WHEN 'PromOcean' THEN 7
    WHEN 'Orrsum' THEN 8
    WHEN 'LFAD' THEN 9
    WHEN 'Firework' THEN 10
    WHEN 'Markets' THEN 11
    WHEN 'Total OG' THEN 12
  END;
```

### DSO, DPO, and DIO Trend

Use this for the efficiency trend chart.

```sql
SELECT
  wc_part,
  month,
  month_sort,
  value
FROM fna.fna_bronze.working_capital_metrics
WHERE period = '2026 Actual'
  AND og = 'Total OG'
  AND wc_part IN ('DSO', 'DPO', 'DIO')
  AND month_sort <= 12
ORDER BY
  CASE wc_part
    WHEN 'DSO' THEN 1
    WHEN 'DPO' THEN 2
    WHEN 'DIO' THEN 3
  END,
  month_sort;
```

### One Business Question

Use this pattern for a one-number answer.

```sql
SELECT
  period,
  og,
  wc_part,
  unit,
  month,
  value
FROM fna.fna_bronze.working_capital_metrics
WHERE period = '2026 Actual'
  AND og = 'Apparel'
  AND wc_part = 'Total WC'
  AND month = 'Mar';
```

### Detail View for One OG and Month

Use this to explain what is inside Total WC for one OG/month.

```sql
SELECT
  wc_part,
  unit,
  value
FROM fna.fna_bronze.working_capital_metrics
WHERE period = '2026 Actual'
  AND og = 'Apparel'
  AND month = 'Mar'
  AND wc_part IN ('AR', 'AP', 'Inventory', 'Factoring', 'OROP', 'LF Credit', 'Total WC')
ORDER BY
  CASE wc_part
    WHEN 'AR' THEN 1
    WHEN 'AP' THEN 2
    WHEN 'Inventory' THEN 3
    WHEN 'Factoring' THEN 4
    WHEN 'OROP' THEN 5
    WHEN 'LF Credit' THEN 6
    WHEN 'Total WC' THEN 7
  END;
```

## 23. How To Ask Questions Against This Dataset

When asking for a SQL query, include:

```text
1. Period: Actual, Budget, 1QR, 2QR, or prior year
2. OG: Apparel, Home and Accessories, SCS, LFMU, Markets, Total OG, etc.
3. WC part: AR, AP, Inventory, Factoring, OROP, LF Credit, Total WC, DSO, DPO, DIO
4. Month: Jan-Dec, Average, or Year End
5. Output shape: one value, trend by month, comparison by period, or comparison by OG
```

Example questions:

```text
Give me 2026 Actual Total WC by OG for Year End.
```

```text
Show Apparel AR, AP, Inventory, Factoring, OROP, LF Credit, and Total WC for March 2026.
```

```text
Compare Total OG Total WC Actual vs Budget vs 1QR vs 2QR by month.
```

```text
Give me LFMU DSO, DPO, and DIO trend for 2026 Actual.
```

## 24. Caveats

- `Total WC` uses signed Jedox values.
- DSO/DPO/DIO use sequence tables as the source of truth, not simple manual division.
- `SCS`, `Markets`, and `Total OG` are subtotal rows and should not be added together with base OGs.
- For current-year actual, future months are `0` in the dataset. Use a Tableau calculated field if charts should show blanks instead.
- For Actual DIO, the balance is Actual but the future consumption buckets are Budget.
- `Average` for current-year Actual uses months from Jan through the current calendar month.
- `Year End` for current-year Actual uses the latest closed month.
