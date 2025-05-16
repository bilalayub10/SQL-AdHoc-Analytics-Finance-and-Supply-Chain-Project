# Finance and Supply Chain Ad Hoc Analytics using SQL

## Table of Contents

-  [Project Overview](#project-overview)
-  [AtliQ Hardwares Business Model](#atliq-hardwares-business-model)
-  [Data Sources](#data-sources)
-  [Finance Analytics](#finance-analytics)
     - [Business Query 1](#business-query-1)
     - [Business Query 2](#business-query-2)
     - [Business Query 3](#business-query-3)
     - [Business Query 4](#business-query-4)
     - [Business Query 5](#business-query-5)
     - [Business Query 6](#business-query-6)
     - [Business Query 7](#business-query-7)
     - [Business Query 8](#business-query-8)
     - [Business Query 9](#business-query-9)
- [Supply Chain Analytics](#supply-chain-analytics)
     - [Business Query 1](#business-query-1)
     - [Business Query 2](#business-query-2)
- [Tools & Technologies](#tools--technologies)
- [Let's Connect](#lets-connect)


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
**Create a stored procedure that can determine the market badge based on the following logic:**

**If total sold quantity > 5 million that market is considered Gold else it is Silver.**

---

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
**Identify Top Markets by Net Sales for a Given Fiscal Year.**

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
<figure>
  <img src="https://github.com/user-attachments/assets/ada3a2c7-cb97-42de-abb3-3c2d391e4002">
  <div align="center"></div>
</figure>

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
<figure>
  <img src="https://github.com/user-attachments/assets/c9a02aca-b25e-4381-b8c2-ec85446a4db5">
  <div align="center"></div>
</figure>

#### Step 4: Calculate Net Invoice Sales from View
```sql
SELECT 
    *, 
    (gross_price_total - gross_price_total * pre_invoice_discount_pct) AS net_invoice_sales
FROM 
    sales_preinv_discount;
```
<figure>
  <img src="https://github.com/user-attachments/assets/a26d35ae-7b7a-4141-99f2-344ec6b28da3">
  <div align="center"></div>
</figure>

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

#### Step 6: Create `sales_postinv_discount` View
```sql
CREATE VIEW sales_postinv_discount AS
SELECT 
    s.date,
    s.fiscal_year,
    s.customer_code,
    s.market,
    s.product_code,
    s.product,
    s.variant,
    s.sold_quantity,
    s.gross_price,
    s.gross_price_total,
    s.pre_invoice_discount_pct,
    (1 - s.pre_invoice_discount_pct) * s.gross_price_total AS net_invoice_sales,
    (po.discounts_pct + po.other_deductions_pct) AS post_invoice_discount_pct
FROM 
    sales_preinv_discount s
JOIN fact_post_invoice_deductions po 
    ON po.customer_code = s.customer_code 
    AND po.product_code = s.product_code 
    AND po.date = s.date;
```
<figure>
  <img src="https://github.com/user-attachments/assets/40c994cc-0119-4305-9e63-8986627215fd">
  <div align="center"></div>
</figure>

#### Step 7: Calculate Final Net Sales
```sql
SELECT
    *, 
    (1 - post_invoice_discount_pct) * net_invoice_sales AS net_sales
FROM 
    sales_postinv_discount;
```
<figure>
  <img src="https://github.com/user-attachments/assets/99df2374-6f39-47f5-a501-1075998b0d34">
  <div align="center"></div>
</figure>
  
#### Step 8: Create Final `net_sales` View

```sql
CREATE VIEW net_sales AS
SELECT 
    s.date,
    s.fiscal_year,
    s.customer_code,
    s.market,
    s.product_code,
    s.product,
    s.variant,
    s.sold_quantity,
    s.gross_price,
    s.gross_price_total,
    s.pre_invoice_discount_pct,
    s.net_invoice_sales,
    s.post_invoice_discount_pct,
    (1 - s.post_invoice_discount_pct) * s.net_invoice_sales AS net_sales
FROM 
    sales_postinv_discount s;
```
#### Step 9: Query Top Markets by Net Sales
```sql
SELECT 
    market,
    ROUND(SUM(net_sales)/1000000, 2) AS net_sales_mln
FROM 
    net_sales
WHERE 
    fiscal_year = 2021
GROUP BY 
    market
ORDER BY 
    net_sales_mln DESC
LIMIT 5;
```
<figure>
  <img src="https://github.com/user-attachments/assets/bea25c5e-5172-4f8d-aa1c-80f015b83383">
  <div align="center"></div>
</figure>

#### Stored Procedure: `get_top_n_markets_by_net_sales`

This stored procedure returns the **top N markets** by **net sales** for a given fiscal year. It accepts two parameters:

- `in_fiscal_year` (INT): The fiscal year for which to calculate net sales.
- `in_top_n` (INT): The number of top markets to return.

##### SQL Code

```sql
CREATE PROCEDURE `get_top_n_markets_by_net_sales`(
    IN in_fiscal_year INT,
    IN in_top_n INT
)
BEGIN
    SELECT 
        market,
        ROUND(SUM(net_sales) / 1000000, 2) AS net_sales_mln
    FROM 
        net_sales
    WHERE 
        fiscal_year = in_fiscal_year
    GROUP BY 
        market
    ORDER BY 
        net_sales_mln DESC
    LIMIT 
        in_top_n;
END
```
### Business Query 5
**Identify Top Customers by Net Sales for a Given Fiscal Year.**

This query identifies the **top 5 customers** by net sales for a specific fiscal year.  
It uses the `net_sales` view and joins with the `dim_customer` table to retrieve customer names for reporting.

---

#### SQL Query
```sql
SELECT 
    c.customer,
    ROUND(SUM(net_sales) / 1000000, 2) AS net_sales_mln
FROM 
    net_sales n
JOIN 
    dim_customer c 
    ON n.customer_code = c.customer_code
WHERE 
    fiscal_year = 2021
    AND c.market = 'India'
GROUP BY 
    c.customer
ORDER BY 
    net_sales_mln DESC
LIMIT 5;
```
<figure>
  <img src="https://github.com/user-attachments/assets/917aaac6-f4d7-4543-a61d-fd85a64bf1e4">
  <div align="center"></div>
</figure>

#### Stored Procedure: `get_top_n_customers_by_net_sales`

To make the query for top customers by net sales reusable and dynamic, a stored procedure was created. This procedure accepts parameters for the **market**, **fiscal year**, and the **number of top customers (N)** to return.

##### SQL Code

```sql
CREATE PROCEDURE `get_top_n_customers_by_net_sales`(
    IN in_market VARCHAR(45),
    IN in_fiscal_year INT,
    IN in_top_n INT
)
BEGIN
    SELECT 
        c.customer,
        ROUND(SUM(net_sales)/1000000, 2) AS net_sales_mln
    FROM 
        net_sales n
    JOIN 
        dim_customer c ON n.customer_code = c.customer_code
    WHERE 
        fiscal_year = in_fiscal_year 
        AND c.market = in_market
    GROUP BY 
        c.customer
    ORDER BY 
        net_sales_mln DESC
    LIMIT 
        in_top_n;
END
```
### Business Query 6
**Create a Stored Procedure to Get Top N Products by Net Sales.**

This stored procedure retrieves the **top N products** based on **net sales** for a specified fiscal year.  
It uses the `net_sales` view, which already accounts for both **pre- and post-invoice discounts**, ensuring accurate revenue reporting.

---

#### SQL Code
```sql
CREATE PROCEDURE `get_top_n_products_by_net_sales`(
    IN in_fiscal_year INT,
    IN in_limit INT
)
BEGIN
    SELECT 
        product, 
        ROUND(SUM(net_sales) / 1000000, 2) AS net_sales_mln
    FROM
        net_sales
    WHERE 
        fiscal_year = in_fiscal_year
    GROUP BY
        product
    ORDER BY
        net_sales_mln DESC
    LIMIT
        in_limit;
END;
```
### Business Query 7
**Generate a Region-wise % Net Sales Breakdown by Customers.**

This query generates a **region-wise percentage breakdown of net sales by customers** for a given fiscal year.  
It supports **regional financial analysis** by showing how much each customer contributes to the total sales in their respective region.

---

#### SQL Query

```sql
WITH cte1 AS (
    SELECT
        c.customer,
        c.region,
        ROUND(SUM(net_sales) / 1000000, 2) AS net_sales_mln
    FROM 
        net_sales s
    JOIN
        dim_customer c USING (customer_code)
    WHERE  
        s.fiscal_year = 2021
    GROUP BY
        c.customer, c.region
)
SELECT 
    *,
    net_sales_mln * 100 / SUM(net_sales_mln) OVER(PARTITION BY region) AS pct
FROM 
    cte1
ORDER BY
    region, net_sales_mln DESC;
```
<figure>
  <img src="https://github.com/user-attachments/assets/1ef7f383-f404-4b10-a225-274bf0b4ee49">
  <div align="center"></div>
</figure>

### Business Query 8
**Create a Stored Procedure to get Top N Products per Division by Quantity Sold.**

This stored procedure retrieves the **top N selling products** in each division based on the **quantity sold** for a specified fiscal year.  
It leverages **window functions** to rank products within each division.

---
#### SQL Code 

```sql
CREATE PROCEDURE `get_top_n_products_per_division_by_qty_sold`(
    in_fiscal_year INT,
    in_top_n INT
)
BEGIN
    WITH cte1 AS (
        SELECT 
            p.division, 
            p.product, 
            SUM(s.sold_quantity) AS sold_quantity
        FROM 
            fact_sales_monthly s
        JOIN
            dim_product p ON s.product_code = p.product_code
        WHERE 
            s.fiscal_year = in_fiscal_year
        GROUP BY
            p.division, p.product
    ),
    cte2 AS (
        SELECT
            *, 
            DENSE_RANK() OVER (PARTITION BY division ORDER BY sold_quantity DESC) AS drnk
        FROM 
            cte1
    )
    SELECT 
        division,
        product,
        sold_quantity
    FROM
        cte2
    WHERE
        drnk <= in_top_n;
END
```
### Business Query 9
**Retrieve the Top 2 Markets in Every Region by Gross Sales.**

This query identifies the **top 2 performing markets** within each region based on their **gross sales** for the fiscal year **2021**.  
It uses the `DENSE_RANK()` window function to handle ties in rankings accurately.

---
#### SQL Query

```sql
WITH cte1 AS (
    SELECT
        g.market, 
        c.region, 
        ROUND(SUM(g.gross_price_total)/1000000, 2) AS gross_sales_mln
    FROM 
        gross_sales g
    JOIN 
        dim_customer c ON g.customer_code = c.customer_code
    WHERE
        g.fiscal_year = 2021
    GROUP BY 
        g.market, c.region
),
cte2 AS (
    SELECT
        *, 
        DENSE_RANK() OVER (PARTITION BY region ORDER BY gross_sales_mln DESC) AS drnk
    FROM 
        cte1
)
SELECT
    market,
    region,
    gross_sales_mln
FROM 
    cte2
WHERE 
    drnk <= 2;
```
<figure>
  <img src="https://github.com/user-attachments/assets/9e5ad801-cbc5-4caf-9d09-df266f73569e">
  <div align="center"></div>
</figure>

## Supply Chain Analytics

### Business Query 1 
**Generate an Aggregate Forecast Accuracy Report for Customers for a Given Fiscal Year.**

This analysis calculates how accurately **sales forecasts match actual sales** at the **customer level**.  
It helps identify **forecasting performance gaps** to improve **inventory planning** and **demand management**.

---
#### Approach

##### Helper Table Creation (`fact_act_est`)
A combined table is created by merging **actual sales** and **forecast data**, aligned by **date**, **product**, and **customer**.  
Missing values in `sold_quantity` or `forecast_quantity` are replaced with zeros to maintain data integrity.

##### Forecast Accuracy Calculation
Using the helper table, the query computes:
- **Total sold quantity** and **forecast quantity** per customer
- **Net forecast error** and **net error percentage**
- **Absolute forecast error** and **absolute error percentage**
- **Forecast accuracy** as:  
  `Forecast Accuracy = 100 - Absolute Error %` (capped at 100%)

#### SQL Query

```sql
-- Step 1: Create helper table combining actual sales and forecast quantities
CREATE TABLE fact_act_est AS
(
    SELECT
        s.date,
        s.fiscal_year,
        s.product_code,
        s.customer_code,
        s.sold_quantity,
        f.forecast_quantity
    FROM 
        fact_sales_monthly s 
    LEFT JOIN 
        fact_forecast_monthly f 
    USING (date, customer_code, product_code)
)
UNION
(
    SELECT
        f.date,
        f.fiscal_year,
        f.product_code,
        f.customer_code,
        s.sold_quantity,
        f.forecast_quantity
    FROM fact_forecast_monthly f 
    LEFT JOIN fact_sales_monthly s
    USING (date, product_code, customer_code)
);

-- Step 2: Replace NULLs in sold_quantity and forecast_quantity with zeros
UPDATE fact_act_est
SET sold_quantity = 0
WHERE sold_quantity IS NULL;

UPDATE fact_act_est
SET forecast_quantity = 0
WHERE forecast_quantity IS NULL;

-- Step 3: Calculate forecast accuracy metrics per customer for fiscal year 2021
WITH forecast_err_table AS 
(
    SELECT 
        customer_code,
        SUM(sold_quantity) AS total_sold_quantity,
        SUM(forecast_quantity) AS total_forecast_quantity,
        SUM(forecast_quantity - sold_quantity) AS net_err,
        SUM(forecast_quantity - sold_quantity) * 100 / SUM(forecast_quantity) AS net_err_pct,
        SUM(ABS(forecast_quantity - sold_quantity)) AS abs_err,
        SUM(ABS(forecast_quantity - sold_quantity)) * 100 / SUM(forecast_quantity) AS abs_err_pct
    FROM 
        fact_act_est
    WHERE 
        fiscal_year = 2021
    GROUP BY
        customer_code
)
SELECT 
    e.*,
    c.market,
    c.customer,
    IF(abs_err_pct > 100, 0, 100 - abs_err_pct) AS forecast_accuracy
FROM 
    forecast_err_table e
JOIN
    dim_customer c 
USING
    (customer_code)
ORDER BY
    forecast_accuracy DESC;
```
<figure>
  <img src="https://github.com/user-attachments/assets/768af90e-36ab-420a-9056-4c1b475925a2">
  <div align="center"></div>
</figure>

#### Stored Procedure: `get_forecast_accuracy`

To streamline the retrieval of **forecast accuracy metrics** for any fiscal year, a **stored procedure** has been created.  
This procedure encapsulates the logic for calculating **forecast errors and accuracy by customer**, making it easy to run this analysis on demand.

##### SQL Code

```sql
CREATE PROCEDURE `get_forecast_accuracy`(
    in_fiscal_year INT
)
BEGIN
    WITH forecast_err_table AS 
    (
        SELECT 
            s.customer_code,
            SUM(s.sold_quantity) AS total_sold_quantity,
            SUM(s.forecast_quantity) AS total_forecast_quantity,
            SUM(forecast_quantity - sold_quantity) AS net_err,
            SUM(forecast_quantity - sold_quantity) * 100 / SUM(forecast_quantity) AS net_err_pct,
            SUM(ABS(forecast_quantity - sold_quantity)) AS abs_err,
            SUM(ABS(forecast_quantity - sold_quantity)) * 100 / SUM(forecast_quantity) AS abs_err_pct
        FROM 
            fact_act_est s
        WHERE 
            s.fiscal_year = in_fiscal_year
        GROUP BY
            customer_code
    )
    SELECT 
        e.*,
        c.market,
        c.customer,
        IF(abs_err_pct > 100, 0, 100 - abs_err_pct) AS forecast_accuracy
    FROM 
        forecast_err_table e
    JOIN
        dim_customer c 
    USING
        (customer_code)
    ORDER BY
        forecast_accuracy DESC;
END
```
### Business Query 2
**Identify Customers with Declining Forecast Accuracy (2020 → 2021).**

This analysis helps **supply chain managers** monitor deteriorating forecast performance by identifying **customers whose forecast accuracy declined** from fiscal year **2020 to 2021**.

---
#### Steps Involved

1. **Create temporary tables** to calculate forecast accuracy for 2020 and 2021:
   - Total sold quantity
   - Total forecast quantity
   - Net forecast error (%)
   - Absolute forecast error (%)
   - Forecast accuracy: `100 - Absolute Error %` (capped at 0% if error > 100%)

2. **Compare** accuracy across both years

3. **Identify customers** whose forecast accuracy has **declined**

#### SQL Query

```sql
-- Step 1: Compute Forecast Accuracy for 2021
DROP TABLE IF EXISTS forecast_accuracy_2021;
CREATE TEMPORARY TABLE forecast_accuracy_2021
WITH forecast_err_table AS 
(
    SELECT 
        s.customer_code AS customer_code,
        c.customer AS customer_name,
        c.market AS market,
        SUM(s.sold_quantity) AS total_sold_qty,
        SUM(s.forecast_quantity) AS total_forecast_qty,
        SUM(s.forecast_quantity - s.sold_quantity) AS net_err,
        ROUND(SUM(s.forecast_quantity - s.sold_quantity) * 100 / SUM(s.forecast_quantity), 1) AS net_err_pct,
        SUM(ABS(s.forecast_quantity - s.sold_quantity)) AS abs_err,
        ROUND(SUM(ABS(s.forecast_quantity - s.sold_quantity)) * 100 / SUM(s.forecast_quantity), 2) AS abs_err_pct
    FROM 
        fact_act_est s
    JOIN
        dim_customer c 
        ON s.customer_code = c.customer_code
    WHERE 
        s.fiscal_year = 2021
    GROUP BY
        s.customer_code
)
SELECT 
    *,
    IF(abs_err_pct > 100, 0, 100.0 - abs_err_pct) AS forecast_accuracy
FROM 
    forecast_err_table;

-- Step 2: Compute Forecast Accuracy for 2020
DROP TABLE IF EXISTS forecast_accuracy_2020;
CREATE TEMPORARY TABLE forecast_accuracy_2020
WITH forecast_err_table AS 
(
    SELECT 
        s.customer_code AS customer_code,
        c.customer AS customer_name,
        c.market AS market,
        SUM(s.sold_quantity) AS total_sold_quantity,
        SUM(s.forecast_quantity) AS total_forecast_quantity,
        SUM(s.forecast_quantity - s.sold_quantity) AS net_err,
        ROUND(SUM(s.forecast_quantity - s.sold_quantity) * 100 / SUM(s.forecast_quantity), 1) AS net_err_pct,
        SUM(ABS(s.forecast_quantity - s.sold_quantity)) AS abs_err,
        ROUND(SUM(ABS(s.forecast_quantity - s.sold_quantity)) * 100 / SUM(s.forecast_quantity), 2) AS abs_err_pct
    FROM 
        fact_act_est s
    JOIN
        dim_customer c 
        ON s.customer_code = c.customer_code
    WHERE 
        s.fiscal_year = 2020
    GROUP BY
        s.customer_code
)
SELECT 
    *,
    IF(abs_err_pct > 100, 0, 100.0 - abs_err_pct) AS forecast_accuracy
FROM 
    forecast_err_table;

-- Step 3: Compare and Identify Declines
SELECT 
    f_2020.customer_code,
    f_2020.customer_name,
    f_2020.market,
    f_2020.forecast_accuracy AS forecast_acc_2020,
    f_2021.forecast_accuracy AS forecast_acc_2021
FROM
    forecast_accuracy_2020 f_2020
JOIN
    forecast_accuracy_2021 f_2021
    ON f_2020.customer_code = f_2021.customer_code
WHERE
    f_2021.forecast_accuracy < f_2020.forecast_accuracy
ORDER BY
    f_2020.forecast_accuracy DESC;
```
<figure>
  <img src="https://github.com/user-attachments/assets/71eac859-936c-4060-a075-517a34f09631">
  <div align="center"></div>
</figure>

## Tools & Technologies

- **SQL**: Core language used to perform ad hoc analysis and extract insights from financial and supply chain data.
- **MySQL Workbench**: Primary environment for writing and executing SQL queries.
- **Microsoft Excel**: Briefly used for support tasks where needed.

## Let's Connect

If you found this project interesting or have questions, feel free to connect with me:

- [LinkedIn](https://www.linkedin.com/in/muhammad-bilal-ayub/) 

























































