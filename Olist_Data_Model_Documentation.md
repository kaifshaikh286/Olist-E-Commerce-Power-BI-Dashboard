# Olist Brazilian E-Commerce: Power BI Data Model Documentation

This document describes every DAX measure, calculated column and relationship I added to the Olist semantic model in Power BI Desktop. All measures live in the `_Measures` table and are organised into Display Folders. Currency is Brazilian Real (R$).

## Contents

1. [Model overview](#model-overview)
2. [Relationships](#relationships)
3. [Calculated columns](#calculated-columns)
4. [Measures](#measures)
   - [1 Core](#1-core)
   - [2 Delivery and Reviews](#2-delivery-and-reviews)
   - [3 Time Intelligence](#3-time-intelligence)
   - [4 Customers and Products](#4-customers-and-products)
   - [5 Advanced](#5-advanced)
5. [Headline results](#headline-results)
6. [Data quality notes](#data-quality-notes)

---

## Model overview

| Type | Tables |
|---|---|
| Fact tables | `fact_orders`, `fact_order_items`, `fact_order_payments`, `fact_order_reviews` |
| Dimension tables | `dim_customers`, `dim_products`, `dim_sellers`, `dim_location`, `dim_date` |
| Measure table | `_Measures` (39 measures, no data) |

`dim_date` is marked as the date table and is the basis for all time intelligence.

---

## Relationships

All relationships are many-to-one with single-direction filtering, except where noted.

| # | From (many side) | To (one side) | Status | Purpose |
|---|---|---|---|---|
| 1 | `fact_orders[order_purchase_date]` | `dim_date[Date]` | **Active** | Main date relationship: orders by purchase date |
| 2 | `fact_orders[order_delivered_date]` | `dim_date[Date]` | **Inactive** | Orders by delivery date, activated with `USERELATIONSHIP` |
| 3 | `dim_sellers[seller_zip_code_prefix]` | `dim_location[geolocation_zip_code_prefix]` | **Inactive** | Seller locations, activated with `USERELATIONSHIP` |
| 4 | `dim_customers[customer_zip_code_prefix]` | `dim_location[geolocation_zip_code_prefix]` | Active | Customer locations |
| 5 | `fact_orders[customer_id]` | `dim_customers[customer_id]` | Active | One-to-one, filters both directions |
| 6 | `fact_order_items[order_id]` | `fact_orders[order_id]` | Active | Items belong to an order |
| 7 | `fact_order_payments[order_id]` | `fact_orders[order_id]` | Active | Payments belong to an order |
| 8 | `fact_order_reviews[order_id]` | `fact_orders[order_id]` | Active | Reviews belong to an order |
| 9 | `fact_order_items[product_id]` | `dim_products[product_id]` | Active | Item to product |
| 10 | `fact_order_items[seller_id]` | `dim_sellers[seller_id]` | Active | Item to seller |

Power BI also created hidden auto date/time tables (`LocalDateTable_...`) for the other date columns (approved, carrier, customer-delivered, estimated, shipping limit, review creation and review answer dates). These are not used in any measure.

**Why two relationships between the same tables?** Power BI allows only one active relationship between two tables. Relationships 2 and 3 are inactive so they do not conflict with relationships 1 and 4. A measure turns them on temporarily with `USERELATIONSHIP`.

**Why `order_purchase_date` and not the timestamp?** `order_purchase_timestamp` includes a time (for example 17:30:18), while `dim_date[Date]` holds whole dates at midnight. A relationship on exact matches linked only 1 of 99,441 orders. The date-only column fixes this, and all 99,441 orders now reach `dim_date`.

---

## Calculated columns

| Table | Column | Folder | Business meaning |
|---|---|---|---|
| `fact_orders` | `order_purchase_date` | n/a | Purchase date with the time removed. Join key to `dim_date`. |
| `fact_orders` | `order_delivered_date` | n/a | Date the customer received the order, time removed. Blank if not delivered. |
| `fact_orders` | `delivery_days` | n/a | Actual delivery time in days, from purchase date to delivered date. |
| `fact_orders` | `estimated_delivery_days` | n/a | Promised delivery time in days, from purchase date to estimated date. |
| `fact_orders` | `delivery_delay_days` | n/a | Delivered date minus estimated date. Positive means late. |
| `fact_orders` | `delivery_status` | n/a | `Not Delivered`, `On Time` or `Late`, compared at day level. |
| `fact_order_reviews` | `review_category` | n/a | Groups the 1-5 score into Positive, Neutral or Negative. |
| `fact_order_items` | `price_band` | n/a | Price tier of the item: Low, Medium, High or Premium. |

Calculated columns are not placed in display folders. They sit in their own tables.

### `fact_orders[order_purchase_date]`
```dax
INT(fact_orders[order_purchase_timestamp])
```

### `fact_orders[order_delivered_date]`
```dax
IF(
    ISBLANK(fact_orders[order_delivered_customer_date]),
    BLANK(),
    INT(fact_orders[order_delivered_customer_date])
)
```

### `fact_orders[estimated_delivery_days]`
```dax
DATEDIFF(
    fact_orders[order_purchase_date],
    INT(fact_orders[order_estimated_delivery_date]),
    DAY
)
```

### `fact_orders[delivery_days]`
```dax
IF(
    ISBLANK(fact_orders[order_delivered_date]),
    BLANK(),
    DATEDIFF(fact_orders[order_purchase_date], fact_orders[order_delivered_date], DAY)
)
```

### `fact_orders[delivery_delay_days]`
```dax
IF(
    ISBLANK(fact_orders[order_delivered_date]),
    BLANK(),
    DATEDIFF(
        INT(fact_orders[order_estimated_delivery_date]),
        fact_orders[order_delivered_date],
        DAY
    )
)
```

### `fact_orders[delivery_status]`
```dax
IF(
    ISBLANK(fact_orders[order_delivered_date]),
    "Not Delivered",
    IF(
        fact_orders[order_delivered_date] <= INT(fact_orders[order_estimated_delivery_date]),
        "On Time",
        "Late"
    )
)
```

### `fact_order_reviews[review_category]`
```dax
SWITCH(
    TRUE(),
    fact_order_reviews[review_score] >= 4, "Positive",
    fact_order_reviews[review_score] = 3, "Neutral",
    "Negative"
)
```

### `fact_order_items[price_band]`
```dax
SWITCH(
    TRUE(),
    fact_order_items[price] < 50, "Low",
    fact_order_items[price] < 150, "Medium",
    fact_order_items[price] <= 500, "High",
    "Premium"
)
```
Boundaries: under R$ 50 is Low, R$ 50 up to 150 is Medium, R$ 150 up to and including R$ 500 is High, and above R$ 500 is Premium.

---

## Measures

All 39 measures are in the `_Measures` table.

### 1 Core

#### Total Sales
Total product sales: the sum of item prices, excluding freight. Format: `R$ #,0`
```dax
SUM(fact_order_items[price])
```

#### Total Freight
Total shipping charges: the sum of freight on all order items. Format: `R$ #,0`
```dax
SUM(fact_order_items[freight_value])
```

#### Total Revenue
Product sales plus shipping charges, i.e. everything customers paid for items and delivery. Format: `R$ #,0`
```dax
[Total Sales] + [Total Freight]
```

#### Total Orders
Number of unique orders placed, regardless of status. Format: `#,0`
```dax
DISTINCTCOUNT(fact_orders[order_id])
```

#### Orders With Items
Number of unique orders that have at least one item line. Orders with no items are excluded. Format: `#,0`
```dax
DISTINCTCOUNT(fact_order_items[order_id])
```

#### Items Sold
Total number of item lines sold. Each row in the order items table is one item. Format: `#,0`
```dax
COUNTROWS(fact_order_items)
```

#### Average Order Value
Average product sales per order: Total Sales divided by orders that have items. `DIVIDE` avoids divide-by-zero errors. Format: `R$ #,0`
```dax
DIVIDE([Total Sales], [Orders With Items])
```

#### Avg Items per Order
Average basket size: items sold divided by orders with items. Format: `0.00`
```dax
DIVIDE([Items Sold], [Orders With Items])
```

#### Unique Customers
Number of real, distinct people. A person gets a new `customer_id` for every order, so `customer_unique_id` identifies them across orders. Format: `#,0`
```dax
DISTINCTCOUNT(dim_customers[customer_unique_id])
```

#### Delivered Orders
Number of orders whose status is `delivered`. Format: `#,0`
```dax
CALCULATE([Total Orders], fact_orders[order_status] = "delivered")
```

#### Canceled Orders
Number of orders whose status is `canceled`. Format: `#,0`
```dax
CALCULATE([Total Orders], fact_orders[order_status] = "canceled")
```

#### Cancellation Rate
Share of all orders that were canceled. Format: `0.0%`
```dax
DIVIDE([Canceled Orders], [Total Orders])
```

#### Average Review Score
Average customer review rating on the 1 to 5 star scale. Format: `0.00`
```dax
AVERAGE(fact_order_reviews[review_score])
```

#### Total Payment Value
Total amount customers paid, summed from the payments table. It can differ slightly from sales plus freight. Format: `R$ #,0`
```dax
SUM(fact_order_payments[payment_value])
```

#### Avg Installments
Average number of installments chosen per payment record. Format: `0.00`
```dax
AVERAGE(fact_order_payments[payment_installments])
```

### 2 Delivery and Reviews

#### Avg Delivery Days
Average number of days from purchase to delivery, for orders that were delivered. Format: `0.0`
```dax
AVERAGE(fact_orders[delivery_days])
```

#### Avg Delivery Delay
Average days versus the promised date. Negative means delivered early, positive means late. Format: `0.0`
```dax
AVERAGE(fact_orders[delivery_delay_days])
```

#### On-Time Delivery %
Share of delivered orders that arrived on or before the estimated date. Orders with no delivered date are excluded from the denominator. Format: `0.0%`
```dax
DIVIDE(
    CALCULATE([Total Orders], fact_orders[delivery_status] = "On Time"),
    CALCULATE([Total Orders], fact_orders[delivery_status] <> "Not Delivered")
)
```

#### Late Orders
Number of orders delivered after the estimated delivery date. Format: `#,0`
```dax
CALCULATE([Total Orders], fact_orders[delivery_status] = "Late")
```

#### % Positive Reviews
Share of reviews that are Positive (4 or 5 stars). Format: `0.0%`
```dax
DIVIDE(
    CALCULATE(COUNTROWS(fact_order_reviews), fact_order_reviews[review_category] = "Positive"),
    COUNTROWS(fact_order_reviews)
)
```

#### Avg Review Score of Late Orders
Average review score for orders delivered late. Compare with the on-time version to show that lateness hurts reviews. Format: `0.00`
```dax
CALCULATE([Average Review Score], fact_orders[delivery_status] = "Late")
```

#### Avg Review Score of On Time Orders
Average review score for orders delivered on time or early. Format: `0.00`
```dax
CALCULATE([Average Review Score], fact_orders[delivery_status] = "On Time")
```

### 3 Time Intelligence

All measures in this folder use `dim_date`.

#### Sales YTD
Year-to-date sales: total sales from 1 January of the selected year up to the latest date in the current filter. Format: `R$ #,0`
```dax
TOTALYTD([Total Sales], dim_date[Date])
```

#### Sales Previous Month
Total sales in the month before the one in the current filter. Format: `R$ #,0`
```dax
CALCULATE([Total Sales], DATEADD(dim_date[Date], -1, MONTH))
```

#### Sales MoM %
Month-over-month sales growth: (this month minus last month) divided by last month. Blank if there is no previous month. Format: `0.0%`
```dax
DIVIDE([Total Sales] - [Sales Previous Month], [Sales Previous Month])
```

#### Sales Previous Year
Total sales for the same period one year earlier. Format: `R$ #,0`
```dax
CALCULATE([Total Sales], SAMEPERIODLASTYEAR(dim_date[Date]))
```

#### Sales YoY %
Year-over-year sales growth: (this period minus same period last year) divided by last year. Blank if there is no prior year. Format: `0.0%`
```dax
DIVIDE([Total Sales] - [Sales Previous Year], [Sales Previous Year])
```

#### Cumulative Sales
Running total of sales from the first date in the data up to the latest date in the current filter. Format: `R$ #,0`
```dax
CALCULATE(
    [Total Sales],
    FILTER(ALL(dim_date[Date]), dim_date[Date] <= MAX(dim_date[Date]))
)
```

#### Orders Previous Month
Number of orders placed in the month before the one in the current filter. Format: `#,0`
```dax
CALCULATE([Total Orders], DATEADD(dim_date[Date], -1, MONTH))
```

#### Sales MoM Indicator
Text label for cards and tables: an up or down arrow with the month-over-month sales change, for example `▲ 12.3%`. Shows a dash if there is no previous month. Returns text.
```dax
VAR MoM = [Sales MoM %]
RETURN
    IF(
        ISBLANK(MoM),
        "–",
        IF(MoM >= 0, "▲ ", "▼ ") & FORMAT(ABS(MoM), "0.0%")
    )
```

### 4 Customers and Products

#### Repeat Customers
Number of real customers (`customer_unique_id`) who placed more than one distinct order. Format: `#,0`
```dax
COUNTROWS(
    FILTER(
        VALUES(dim_customers[customer_unique_id]),
        CALCULATE(DISTINCTCOUNT(fact_orders[order_id])) > 1
    )
)
```

#### Repeat Customer Rate
Share of unique customers who ordered more than once: Repeat Customers divided by Unique Customers. Format: `0.0%`
```dax
DIVIDE([Repeat Customers], [Unique Customers])
```

#### Sales % of Total
Each row's share of total sales within the current selection. `ALLSELECTED` keeps slicer filters but ignores the row or category on the visual. Format: `0.0%`
```dax
DIVIDE([Total Sales], CALCULATE([Total Sales], ALLSELECTED()))
```

#### Category Sales Rank
Rank of each product category by total sales within the current selection. 1 is the best-selling category and ties share a rank. Format: `0`
```dax
IF(
    ISBLANK([Total Sales]),
    BLANK(),
    RANKX(ALLSELECTED(dim_products[product_category]), [Total Sales], , DESC, DENSE)
)
```

#### Top 10 Sellers Sales
Total sales made by the 10 best-selling sellers in the current selection. Shows how concentrated sales are among top sellers. Format: `R$ #,0`
```dax
CALCULATE(
    [Total Sales],
    TOPN(10, ALLSELECTED(dim_sellers[seller_id]), [Total Sales], DESC)
)
```

#### Payment Share %
Each payment type's share of total payment value (credit card, boleto and so on) within the current selection. Format: `0.0%`
```dax
DIVIDE(
    [Total Payment Value],
    CALCULATE([Total Payment Value], ALLSELECTED(fact_order_payments[payment_type]))
)
```

#### Freight % of Sales
Shipping cost as a percentage of product sales: Total Freight divided by Total Sales. Format: `0.0%`
```dax
DIVIDE([Total Freight], [Total Sales])
```

### 5 Advanced

#### Sales by Delivery Date
Total sales grouped by the date the order was delivered instead of the purchase date. Uses the inactive relationship between `dim_date` and the delivered date, switched on only inside this measure with `USERELATIONSHIP`. Format: `R$ #,0`
```dax
CALCULATE(
    [Total Sales],
    USERELATIONSHIP(dim_date[Date], fact_orders[order_delivered_date])
)
```

#### Sellers by Location
Number of distinct sellers located in the selected zip codes. Uses the inactive relationship from `dim_location` to `dim_sellers` on zip code, switched on only inside this measure with `USERELATIONSHIP`. Format: `#,0`
```dax
CALCULATE(
    DISTINCTCOUNT(dim_sellers[seller_id]),
    USERELATIONSHIP(dim_location[geolocation_zip_code_prefix], dim_sellers[seller_zip_code_prefix])
)
```

---

## Headline results

| Metric | Value |
|---|---|
| Total Orders | 99,441 |
| Unique Customers | 96,096 |
| Total Sales | R$ 13,591,644 |
| Total Freight | R$ 2,251,910 |
| Average Order Value | R$ 138 |
| Cancellation Rate | 0.6% |
| Average Review Score | 4.09 |
| On-Time Delivery % | 93.2% |
| Avg Review Score of Late Orders | 2.27 |
| Avg Review Score of On Time Orders | 4.29 |
| Repeat Customer Rate | 3.1% |
| Freight % of Sales | 16.6% |
| Credit card share of payments | 78.3% |
| Top category by sales | health_beauty (9.3%) |

Late deliveries score about 2 stars lower than on-time ones (2.27 against 4.29), and only about 3% of customers ever order twice.

---

## Data quality notes

- **2016 is almost empty.** Only September, October and December 2016 have sales, so month-over-month and year-over-year values for early periods can be extreme (for example Dec 2017 YoY of 68,248%). Charts for growth should start in 2017.
- **September and October 2018 are partial.** Order data ends on 17 October 2018, so those months look like large drops.
- **Two orders** have status `delivered` but no delivery date, so `Delivered Orders` (96,478) is slightly higher than the count of orders with a delivered date (96,476).
- **2,965 orders have no delivered date.** Their sales (R$ 370,749) cannot land in any month in `Sales by Delivery Date`.
- **7 of 3,095 sellers** have zip codes that are missing from `dim_location`, so they will not appear on a map.
- **A payment type of `not_defined`** exists with a payment value of R$ 0.
- **`fact_orders` to `dim_customers` is one-to-one with filters in both directions.** It is needed so customer attributes can filter orders.
