# Finance and Supply Chain Ad Hoc Analytics using SQL

## Project Overview

In 2018, AtliQ Hardwares — a rapidly growing manufacturer of computer peripherals — faced a major operational disruption when critical Excel planning files became corrupted and irrecoverable. This incident exposed the limitations of relying on spreadsheets for business-critical operations and sparked an internal push toward more reliable, scalable data solutions.

This project focuses on leveraging **SQL-based ad hoc analysis** to address key business challenges in **Finance** and **Supply Chain** functions. Using a structured MySQL database provided by the company, I conducted deep-dive analyses to extract actionable insights, identify inefficiencies, and support data-driven decision-making during AtliQ's growth phase.

This project simulates a real-world scenario where analysts are expected to work within existing data systems to uncover trends, anomalies, and opportunities that drive business value.

## AtliQ Hardwares Business Model

AtliQ Hardwares is an imaginary global company specializing in manufacturing and selling high-quality hardware products, including PCs, keyboards, printers, and other peripherals. Similar to industry leaders such as HP and Dell, AtliQ designs and produces hardware and distributes it through multiple sales channels. AtliQ's business operates through three primary channels:

Retailers: This includes both Brick & Mortar (physical retail stores like BestBuy, Croma, etc.) and E-Commerce (online stores such as Amazon, Flipkart, etc.). These retailers act as intermediaries, purchasing AtliQ products in bulk and selling them directly to the end consumers.

Direct Sales: AtliQ operates its own stores, both online and offline, where they sell directly to consumers. This allows them to control the customer experience and build direct relationships with their user base.

Distributors: In certain regions, AtliQ works with distributors who purchase hardware from them and sell it to various retailers across their respective markets.

AtliQ’s customers fall into two main categories:

Brick & Mortar Stores: Physical stores where consumers can walk in, experience products, and make purchases in person.

E-Commerce Stores: Online platforms where consumers can browse and purchase products digitally.

<figure>
  <img src="https://github.com/user-attachments/assets/53021fc7-9655-45ca-853f-e53ad596b461">
  <div align="center"></div>
</figure>

## Data Sources

This project utilizes data extracted from a structured SQL database (`gdb0041`), designed to reflect various aspects of AtliQ Hardwares business operations across sales, finance, and supply chain. The data spans multiple dimensions and fact tables, allowing for detailed analysis and reporting.

### Dimension Tables

- **`dim_customer`** *(209 records | 7 columns)*  
  Contains customer attributes across 27 markets and 74 unique customers. Includes platform type (E-Commerce or Brick & Mortar) and sales channel (Retailer, Direct, Distributor).

- **`dim_product`** *(397 records | 6 columns)*  
  Details product hierarchy, including division, segment, category, and variant for in-depth product-level analysis.

### Fact Tables

- **`fact_forecast_monthly`** *(1,885,941 records | 5 columns)*  
  Forecasted monthly product demand per customer. All dates are normalized to the first of the month to support accurate time-series modeling.

- **`fact_sales_monthly`** *(1,425,706 records | 5 columns)*  
  Captures actual monthly sales quantities, enabling direct comparison with forecasts for variance analysis.

- **`fact_freight_cost`** *(135 records | 4 columns)*  
  Provides market-level logistics and freight costs, segmented by fiscal year.

- **`fact_gross_price`** *(1,182 records | 3 columns)*  
  Contains annual gross pricing details per product.

- **`fact_manufacturing_cost`** *(1,182 records | 3 columns)*  
  Captures product-level manufacturing costs by fiscal year.

- **`fact_pre_invoice_deductions`** *(1,045 records | 3 columns)*  
  Pre-invoice discount percentages by customer and fiscal year.

- **`fact_post_invoice_deductions`** *(2,063,076 records | 5 columns)*  
  Post-invoice deductions including promotional rebates and discounts applied after sales.

## Finance Analytics

### Business Query 1  
**What are the monthly product-level sales for the customer _Croma India_ in fiscal year 2021?**

This report analyzes how individual products performed on a monthly basis for a key customer — Croma India — during FY 2021. The goal is to evaluate demand trends, optimize forecasting, and understand revenue contributions per product.

---

#### Approach
- Filtered transactions for Croma India (`customer_code = 90002002`) from the `fact_sales_monthly` table.
- Joined product and pricing data from `dim_product` and `fact_gross_price`.
- Calculated **monthly gross revenue** by multiplying `sold_quantity × gross_price`.
- Used a custom SQL function to determine fiscal years.

#### Custom Function: `get_fiscal_year()`

```sql
-- Returns fiscal year for a given date. Assumes fiscal year starts in September.

CREATE FUNCTION `get_fiscal_year`(
    calendar_date DATE) RETURNS INT
    DETERMINISTIC
BEGIN
    DECLARE fiscal_year INT;
    SET fiscal_year = YEAR(DATE_ADD(calendar_date, INTERVAL 4 MONTH));
    RETURN fiscal_year;
END
```

#### SQL Query

```sql
SELECT 
    s.date,
    s.product_code,
    p.product,
    p.variant,
    s.sold_quantity,
    ROUND(g.gross_price, 2) AS gross_price,
    ROUND(g.gross_price * s.sold_quantity, 2) AS gross_price_total
FROM 
    fact_sales_monthly s
JOIN 
    dim_product p ON s.product_code = p.product_code
JOIN 
    fact_gross_price g ON g.product_code = s.product_code
                      AND g.fiscal_year = get_fiscal_year(s.date)
WHERE 
    customer_code = 90002002
    AND get_fiscal_year(s.date) = 2021
ORDER BY 
    s.date ASC;
```
<figure>
  <img src="https://github.com/user-attachments/assets/6edb5dbc-db55-4263-ad6d-c9f0bb8f91e9">
  <div align="center"></div>
</figure>

### Business Query 2  
**What is the aggregated monthly gross sales for the customer _Croma India_?**

This query summarizes the total gross sales per month for Croma India. It's useful for identifying monthly revenue patterns, detecting seasonality, and evaluating customer performance over time.

---
#### Approach
- Filtered sales data for Croma India.
- Used a join with `fact_gross_price` to get price information per product.
- Applied `SUM(gross_price × sold_quantity)` to compute total monthly revenue.
- Grouped results by transaction month and sorted chronologically.

#### SQL Query

```sql
SELECT
    s.date,
    SUM(g.gross_price * s.sold_quantity) AS gross_price_total
FROM 
    fact_sales_monthly s
JOIN 
    fact_gross_price g
    ON s.product_code = g.product_code 
    AND g.fiscal_year = get_fiscal_year(s.date)
WHERE
    customer_code = 90002002
GROUP BY
    s.date
ORDER BY
    s.date ASC;
```
<figure>
  <img src="https://github.com/user-attachments/assets/2301d05a-9809-4215-82c3-b2527518bb91">
  <div align="center"></div>
</figure>

#### Stored Procedure — `get_monthly_gross_sales_for_customer`

To improve scalability and reusability of monthly gross sales reporting, I created a stored procedure that accepts **multiple customer codes** and returns their **aggregated monthly gross sales**.

##### Purpose
This stored procedure enables dynamic gross sales reporting across one or more customers without rewriting the query logic. Useful for both **manual analysis** and **automated pipelines**.

##### Key Benefits
- Accepts multiple customer codes as a comma-separated string.
- Reduces query duplication.
- Improves maintainability in BI tools or dashboards.

##### SQL Code

```sql
CREATE PROCEDURE `get_monthly_gross_sales_for_customer`(
    in_customer_codes TEXT)
BEGIN
    SELECT
        s.date,
        SUM(g.gross_price * s.sold_quantity) AS gross_price_total
    FROM 
        fact_sales_monthly s
    JOIN 
        fact_gross_price g
        ON s.product_code = g.product_code 
        AND g.fiscal_year = get_fiscal_year(s.date)
    WHERE
        FIND_IN_SET(s.customer_code, in_customer_codes) > 0
    GROUP BY
        date;
END
```
### Business Query 3
**Determine Market Badge Based on Sales Volume**

---

**Objective:**  
Create a stored procedure to classify markets into badges (**Gold** or **Silver**) based on total sold quantity during a given fiscal year.

**Logic:**
- If a market’s total sold quantity > 5 million → **Gold**
- Otherwise → **Silver**

This classification helps in **performance benchmarking across markets** and supports **data-driven investment or incentive decisions**.

#### SQL Code

```sql
CREATE PROCEDURE get_market_badge(
    IN in_fiscal_year YEAR,
    IN in_market VARCHAR(45),
    OUT out_badge VARCHAR(45)
)
BEGIN
    DECLARE qty INT DEFAULT 0;

    -- Set default market to India if input is empty
    IF in_market = '' THEN 
        SET in_market = 'India';
    END IF;

    -- Get total sold quantity for given market and fiscal year
    SELECT 
        SUM(s.sold_quantity) INTO qty
    FROM 
        fact_sales_monthly s
    JOIN 
        dim_customer c ON s.customer_code = c.customer_code
    WHERE
        get_fiscal_year(s.date) = in_fiscal_year
        AND c.market = in_market;

    -- Classify market badge based on quantity
    IF qty > 5000000 THEN
        SET out_badge = 'Gold';
    ELSE 
        SET out_badge = 'Silver';
    END IF;
END
```
<figure>
  <img src="https://github.com/user-attachments/assets/d88c1f1e-1f72-4441-bb41-daf4d6ed5c01">
  <div align="center"></div>
</figure>
<figure>
  <img src="https://github.com/user-attachments/assets/cbb2180b-741a-4b16-9e39-07c275657f35">
  <div align="center"></div>
</figure>

### Business Query 4
**Top Markets by Net Sales for a Given Fiscal Year**

---

**Objective:**  
Create a report for the **top markets by net sales** for a selected fiscal year. To achieve this, we must progressively calculate `net_sales` by incorporating both **pre-invoice** and **post-invoice discounts**. This involves a series of queries and performance optimizations.

#### Step 1: Generate Pre-Invoice Discount Report

```sql
SELECT 
    s.date, s.product_code, p.product, p.variant, s.sold_quantity,
    ROUND(g.gross_price, 2) AS gross_price,
    ROUND(g.gross_price * s.sold_quantity, 2) AS gross_price_total,
    pre.pre_invoice_discount_pct
FROM 
    fact_sales_monthly s
JOIN dim_product p ON s.product_code = p.product_code
JOIN fact_gross_price g ON g.product_code = s.product_code AND g.fiscal_year = get_fiscal_year(s.date)
JOIN fact_pre_invoice_deductions pre ON pre.customer_code = s.customer_code AND pre.fiscal_year = get_fiscal_year(s.date)
WHERE 
    get_fiscal_year(date) = 2021
LIMIT 1000000;
```
<figure>
  <img src="https://github.com/user-attachments/assets/aef6675a-d0ed-4ae5-ad9f-f6603345d6b9">
  <div align="center"></div>
</figure>

##### Performance Improvement #1: Add `dim_date` Table

By introducing the `dim_date` table, we avoid applying functions directly on date columns in the `WHERE` clause and improve join logic across time-based tables. This enhances query performance and supports better filtering and aggregation.

```sql
SELECT 
    s.date, 
    s.product_code, 
    p.product, 
    p.variant, 
    s.sold_quantity,
    ROUND(g.gross_price, 2) AS gross_price,
    ROUND(g.gross_price * s.sold_quantity, 2) AS gross_price_total,
    pre.pre_invoice_discount_pct
FROM 
    fact_sales_monthly s
JOIN dim_product p 
    ON s.product_code = p.product_code
JOIN dim_date dt 
    ON dt.calendar_date = s.date
JOIN fact_gross_price g 
    ON g.product_code = s.product_code 
    AND g.fiscal_year = dt.fiscal_year
JOIN fact_pre_invoice_deductions pre 
    ON pre.customer_code = s.customer_code 
    AND pre.fiscal_year = dt.fiscal_year
WHERE 
    dt.fiscal_year = 2021
LIMIT 1000000;
```
##### Performance Improvement #2: Add `fiscal_year` Column in `fact_sales_monthly`

Adding a `fiscal_year` column to the `fact_sales_monthly` table simplifies joins with other fact tables and eliminates the need for a date dimension join in certain use cases, improving performance and clarity.

```sql
SELECT 
    s.date, 
    s.product_code, 
    p.product, 
    p.variant, 
    s.sold_quantity,
    ROUND(g.gross_price, 2) AS gross_price,
    ROUND(g.gross_price * s.sold_quantity, 2) AS gross_price_total,
    pre.pre_invoice_discount_pct
FROM 
    fact_sales_monthly s
JOIN dim_product p 
    ON s.product_code = p.product_code
JOIN fact_gross_price g 
    ON g.product_code = s.product_code AND g.fiscal_year = s.fiscal_year
JOIN fact_pre_invoice_deductions pre 
    ON pre.customer_code = s.customer_code AND pre.fiscal_year = s.fiscal_year
WHERE 
    s.fiscal_year = 2021
LIMIT 1000000;
```
#### Step 2: Calculate Net Invoice Sales Using CTE
```sql
WITH cte1 AS (
    SELECT 
        s.date, 
        s.product_code, 
        p.product, 
        p.variant, 
        s.sold_quantity,
        ROUND(g.gross_price, 2) AS gross_price,
        ROUND(g.gross_price * s.sold_quantity, 2) AS gross_price_total,
        pre.pre_invoice_discount_pct
    FROM 
        fact_sales_monthly s
    JOIN dim_product p 
        ON s.product_code = p.product_code
    JOIN fact_gross_price g 
        ON g.product_code = s.product_code AND g.fiscal_year = s.fiscal_year
    JOIN fact_pre_invoice_deductions pre 
        ON pre.customer_code = s.customer_code AND pre.fiscal_year = s.fiscal_year
    WHERE 
        s.fiscal_year = 2021
    LIMIT 1000000
)
SELECT 
    *, 
    (gross_price_total - gross_price_total * pre_invoice_discount_pct) AS net_invoice_sales
FROM 
    cte1;
```
#### Step 3: Create `sales_preinv_discount` View
```sql
CREATE VIEW sales_preinv_discount AS
SELECT 
    s.date,
    s.fiscal_year,
    s.customer_code,
    c.market,
    s.product_code,
    p.product,
    p.variant,
    s.sold_quantity,
    ROUND(g.gross_price, 2) AS gross_price,
    ROUND(g.gross_price * s.sold_quantity, 2) AS gross_price_total,
    pre.pre_invoice_discount_pct
FROM 
    fact_sales_monthly s
JOIN dim_customer c 
    ON c.customer_code = s.customer_code
JOIN dim_product p 
    ON s.product_code = p.product_code
JOIN fact_gross_price g 
    ON g.product_code = s.product_code AND g.fiscal_year = s.fiscal_year
JOIN fact_pre_invoice_deductions pre 
    ON pre.customer_code = s.customer_code AND pre.fiscal_year = s.fiscal_year;
```
#### Step 4: Calculate Net Invoice Sales from View
```sql
SELECT 
    *, 
    (gross_price_total - gross_price_total * pre_invoice_discount_pct) AS net_invoice_sales
FROM 
    sales_preinv_discount;
```
#### Step 5: Incorporate Post-Invoice Discounts
```sql
SELECT
    *, 
    (1 - pre_invoice_discount_pct) * gross_price_total AS net_invoice_sales,
    (po.discounts_pct + po.other_deductions_pct) AS post_invoice_discount_pct
FROM 
    sales_preinv_discount s
JOIN fact_post_invoice_deductions po 
    ON po.customer_code = s.customer_code 
    AND po.product_code = s.product_code 
    AND po.date = s.date;
```






























