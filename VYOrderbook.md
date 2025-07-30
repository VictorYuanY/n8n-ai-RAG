#   Knowledge Base: Querying Enterprise Sales & Order Data (Databricks SQL)

---

##  Step-by-Step Thinking Logic for Writing SQL for `fna.fna_bronze.vw_gob_spencerorderbook` Dataset

---

##  1. SQL CLAUSE
##  1.1. Select Clause

*   Read carefully what the user needs. Since our dataset only contains **`OOH`** as the measure, you always include `OOH` in the select clause. The default select clause will **ALWAYS** have:
`select sum(OOH)`.
*   **`OOH`** (from `vw_gob_spencerorderbook`):
    *   Primary value metric, often represents FOB cost/value of orders/shipments.
    *   Use `SUM()` with `GROUP BY` in SQL for aggregation (**Mandatory**).


*   Other parts of the Select Clause depend on what the user asks for. For example:
*   If the user asks "which customers...", select `REPORT_CORPORATE_CUSTOMER_NAME` along with `SUM(OOH)`.
*   Similarly, if the user asks for OG (Operating Group) / ST (Stream) / PG (Product Group) level, pick them accordingly and you can refer to the **table 1** below for mapping user terms to SQL columns:
### Table 1
| What users say/write | What it means in SQL                |
|---------------------|-----------------------------------|
| OG/Operating Group   | REPORT_OPERATING_GROUP_DESC_UPDATED|
| stream              | REPORT_STREAM_DESC                 |
| PG/Product group     | REPORT_PRODUCT_GROUP_DESC          |
| division             | REPORT_DIVISION_DESC               |
| customer             | REPORT_CORPORATE_CUSTOMER_NAME    |
| sales/orders         | OOH                              |
| YoY Change           | YoY_Difference                   |
| WoW Change           | WoW_Difference                   |

* For OG/Operating Group, here is a more detailed mapping:

### Table 2
| What users say/write | What it means in SQL         |
|---------------------|-----------------------------|
| LFFA/Fashion        | LF Fashion                  |
| LFUM                | LF Markets USA              |
| Apparel             | Apparel                     |
| Home                | Home And Accessories        |
| Miles               | Miles                       |
| LFAD                | LF Asia Direct              |
| promocean           | Promocean                   |
| firework            | Firework                    |

* **The above OG table can be grouped to 2 Segments:**
*   **SCS (Supply Chain Solutions):** Includes Apparel, Home.
*   **Markets (Onshore Businesses):** Includes LF Markets USA, Miles, Promocean, LF Fashion, Fireworks, C2M, Asia Group.

---
## 1.2. FROM Clause
**Always from `fna.fna_bronze.vw_gob_spencerorderbook`**

## 1.3. WHERE Clause

**IMPORTANT:** *Carefully determine which year or week the user is asking for:*
### 1.3.1. Ship_Year and Week_No Filter - ***THIS IS A MUST IN WHERE CLAUSE***
1. If the user specifies the year and week, filter explicitly:

  ```
  WHERE Ship_Year = <specified_year>
    AND Week_No = <specified_week>
  ```

2. If the user does **not** specify the year and week:

  - If the user says "current week," use the current ISO week number. For example, 2025/7/23 is ISO Week Number 30.

    ```
    -- In WHERE clause
    WHERE Week_No = weekofyear(current_date())-1
    ```

  - If the user says "current year," use the current year:

    ```
    -- In WHERE clause
    WHERE Ship_Year = year(current_date())
    ```
### 1.3.2. REPORT_OPERATING_GROUP_DESC_UPDATED and REPORT_CORPORATE_CUSTOMER_NAME Filters 
* those are optional in where clause depending on what the user ask. If they need OG level or customer level, then we will put this in where clause and can refer **table 1 and table 2**

### 1.4. GROUP BY Clause - 
*   Query aggregated OOH `GROUP BY REPORT_CORPORATE_CUSTOMER_NAME` if user asks for customer level. 
*    Query aggregated OOH `GROUP BY REPORT_OPERATING_GROUP_DESC_UPDATED` if user asks for OG level

### 1.5. ORDER BY Clause - Use this when users asks ranking

### 1.6. LIMIT Clause - from the ranking from ORDER BY, limit how many are showing


## Additional Tips:

- Always parameterize user inputs to prevent SQL injection and improve query safety.
- Use SQL comments to clarify complex logic or future maintenance.
- Take advantage of Databricks SQL built-in functions and Delta Lake optimizations for efficiency.

---

## 2. Sample Questions with the SQL provided
1. Questions: What’s current OOH look like? 
*   Thinking Logic: as the user did not ask level of details, so we just give the total OOH. so select sum(OOH) is okay. in where clause, since ship_year and week_no are must, and the user did not specify which year and which week, but mentioned current, so we just pull the current year and week. same reason as the user did not ask level of details, we can skip group by and order by, then the final sql will be 
*   Answer SQL: 
```sql
select sum(OOH) from fna.fna_bronze.vw_gob_spencerorderbook
where Ship_Year =  year(current_date()) 
and Week_No = weekofyear(current_date())-1
```
2. Question: which customer contributed the most growth vs last year? 
*   Thinking logic:since the user asked for the customer level, will need to select **REPORT_CORPORATE_CUSTOMER_NAME** in the sql and also group by **REPORT_CORPORATE_CUSTOMER_NAME**. As the customer is asking fro the most growth, will need to have YoY_Difference and order by **REPORT_CORPORATE_CUSTOMER_NAME DESC** and limit to a number
*   Answer SQL
``` sql
select REPORT_CORPORATE_CUSTOMER_NAME, sum(YoY_Difference) from fna.fna_bronze.vw_gob_spencerorderbook
where Ship_Year =  year(current_date()) 
and Week_No = weekofyear(current_date())-1
group by REPORT_CORPORATE_CUSTOMER_NAME
order by sum(YoY_Difference) DESC
LIMIT 5
```

## 3. Standard Business Conventions & Terminology

These conventions **MUST** be followed in all business interactions:

### 3.1. OOH (Order on Hand) Definition

*   When business users refer to **OOH**, they almost always mean **orders scheduled to ship within the *current* calendar year** (e.g., 2025 as of the last update date), even if those orders were placed in a prior year or are part of snapshots containing next-year data.
*   **Implementation:** Filter queries accordingly, typically using `SHIP_YEAR = ?current_year?` (e.g., `SHIP_YEAR = 2025`) on the `vw_gob_spencerorderbook` table for standard OOH analysis.

### 3.2. Common Terminology/Aliases

*   **SCS (Supply Chain Services):** This term is sometimes used to refer collectively to the **'Home & Accessories'** and **'Apparel'** Operating Groups. Be aware of this alias when interpreting requests or filtering data. (See Appendix 8 for breakdown dimensions related to these OGs).

### 3.3. Notes on formatting
*   **Structure:** Use prose for context, bullet points for details.
*   **Monetary Values:**
    *   Millions: `USD #,##0.0m` (e.g., USD 1,234.5m)
    *   Thousands: `USD #,##0.0k` (e.g., USD 1,234.5k)
*   **Percentages:** `0.0%` (e.g., 25.7%).
*   **Negative Numbers:** Use descriptive words for clarity:
    *   `Loss of USD 1.0m` (instead of `- USD 1.0m Profit`)
    *   `Decrease of 5.0%` or `Decline of 5.0%` (instead of `- 5.0% Change`)



