# Knowledge Base: Querying Enterprise Sales & Order Data (Databricks SQL)

---

### Step-by-Step Thinking Logic for Writing SQL for `fna.fna_bronze.vw_gob_spencerorderbook` Dataset

---

## 1.1 Select Clause

* Read carefully what the user needs. Since our dataset only contains `OOH` as the measure, you always include `OOH` in the select clause. The default select clause will **ALWAYS** have:
`select sum(OOH)`.


* Other parts of the Select Clause depend on what the user asks for. For example:
- If the user asks "which customers...", select `REPORT_CORPORATE_CUSTOMER_NAME` along with `SUM(OOH)`.
- Similarly, if the user asks for OG (Operating Group) / ST (Stream) / PG (Product Group) level, pick them accordingly and you can refer to the table below for mapping user terms to SQL columns:

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
- **SCS (Supply Chain Solutions):** Includes Apparel, Home.
- **Markets (Onshore Businesses):** Includes LF Markets USA, Miles, Promocean, LF Fashion, Fireworks, C2M, Asia Group.

---
## 1.2 FROM Clause
**Always from `fna.fna_bronze.vw_gob_spencerorderbook`**

## 1.3 WHERE Clause

* Carefully determine which year or week the user is asking for:
### 1.3.1 Ship_Year and Week_No Filter - ***THIS IS A MUST IN WHERE CLAUSE***
1. If the user specifies the year and week, filter explicitly:

  ```
  WHERE Ship_Year = <specified_year>
    AND Week_No = <specified_week>
  ```

2. If the user does **not** specify the year and week:

  - If the user says "current week," use the current ISO week number. For example, 2025/7/23 is ISO Week Number 30.

    ```
    -- In WHERE clause
    WHERE Week_No = weekofyear(current_date())
    ```

  - If the user says "current year," use the current year:

    ```
    -- In WHERE clause
    WHERE Ship_Year = year(current_date())
    ```
### 1.3.2 
**REPORT_OPERATING_GROUP_DESC_UPDATED** and **REPORT_CORPORATE_CUSTOMER_NAME** are optional in where clause depending on what the user ask. If they need OG level or customer level, then we will put this in where clause and can refer 

---

## Additional Tips:

- Always parameterize user inputs to prevent SQL injection and improve query safety.
- Use SQL comments to clarify complex logic or future maintenance.
- Take advantage of Databricks SQL built-in functions and Delta Lake optimizations for efficiency.

---

*If you need, I can also help draft sample SQL templates based on this logic.*



