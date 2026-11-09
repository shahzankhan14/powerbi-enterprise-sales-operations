# Enterprise Power BI Transformation: Turning a Disconnected 23-Table System into an Enterprise Star Schema

## 📌 Project Overview
This project demonstrates an end-to-end data architecture and reporting transformation in Power BI. A fragmented, 23-table operational system was re-engineered into an enterprise Kimball star/galaxy schema with 7 dimensions, 6 fact and bridge tables, a dedicated measures table, and dynamic Row-Level Security (RLS).

The solution powers a 3-page executive dashboard suite:
1. **Executive Sales Overview:** High-level revenue health, margin tracking, and dimensional drill-downs.
2. **Operations & Cash Lifecycle:** Lead-time tracking across order milestones and open receivable monitoring.
3. **Marketing Performance & Campaign Analysis:** Marketing budget utilization, ad spend efficiency, and audience engagement.

---

## 🛠️ Data Model Architecture

The model was restructured from operational source files into a Kimball galaxy schema:

### Dimensions (7 Tables)
* **`dim_customer`**: Master customer records with contact details, credit limits, and regional assignments.
* **`dim_product`**: Product catalog hierarchy with standardized categories and surrogate keys.
* **`dim_date`**: Calendar table for standardized time-intelligence analysis.
* **`dim_geo`**: Geographic master supporting role-playing relationships (shipping and billing locations).
* **`dim_order_junk`**: Junk dimension consolidating order status, priority, and sales channels.
* **`dim_campaign`**: Campaign planning metadata, start/end dates, and planned budgets.
* **`dim_security`**: User-to-region mapping table configured for Row-Level Security.

### Facts & Bridge (6 Tables)
* **`fact_sales`**: Transactional grain (line-item level; baseline total sales: **$526.64K** across 200 rows).
* **`fact_order_proccess`**: Accumulating snapshot tracking lifecycle dates (`order_date` → `ship_date` → `delivery_date` → `invoice_date` → `pay_date`).
* **`fact_inventory`**: Monthly stock snapshots unpivoted from source format.
* **`fact_campaign_spend`**: Daily marketing expenditure, impressions, and clicks.
* **`fact_sales_target`**: Monthly revenue budget targets.
* **`fact_promotiona_coverage`**: Factless bridge table mapping promotional campaigns to covered product SKUs.

### Dedicated Measures Table
* **`_measures`**: Central repository for all DAX calculations organized into structured display folders.

---

## 🔒 Enterprise Governance: Dynamic Row-Level Security (RLS)
* **Configuration:** Built using `dim_security` and filtered by `[user_email] = USERPRINCIPALNAME()`.
* **Propagation:** `dim_security` filters `dim_customer` on `region`, which propagates downstream to transactional facts (`fact_sales` and `fact_order_proccess`).
* **Testing:** Verified via **"View as Roles"** to ensure regional managers can only access transactions within their authorized territory.

---

## 📊 Dashboard Suite Highlights

* **Page 1: Executive Sales Overview**
  * Tracks total sales ($526.64K), target achievement %, order volume (80 orders), and gross margin % (36.9%).
  * Compares shipping vs. billing locations using DAX `USERELATIONSHIP`.
* **Page 2: Operations & Cash Lifecycle**
  * Monitors fulfillment lead times (2.8 days order-to-ship, 5.7 days ship-to-deliver).
  * Evaluates regional delivery times against a 7-day SLA benchmark.
  * Identifies 20 unpaid orders accounting for $78.4K in exposed receivables.
* **Page 3: Marketing Performance & Campaign Analysis**
  * Analyzes marketing spend ($78.84K) across 6 major campaigns against an $81K planned budget (97.33% utilization).
  * Evaluates channel efficiency across 224K clicks with CTR and CPC metrics.

---

## 💡 Key DAX Calculations

```dax
// Role-Playing Billing Geography
Sales by Billing City = 
CALCULATE (
    [Total Sales],
    USERELATIONSHIP ( fact_sales[bill_to_city_key], dim_geo[geo_key] )
)

// Pending Cash Exposure (Unpaid Orders)
Pending Payment Amount = 
SUMX (
    FILTER (
        SUMMARIZE (
            fact_order_proccess,
            fact_order_proccess[order_id],
            "OrderTotal", MAX ( fact_order_proccess[order_total] ),
            "HasBlankPayment", MAX ( IF ( ISBLANK ( fact_order_proccess[pay_date] ), 1, 0 ) )
        ),
        [HasBlankPayment] = 1
    ),
    [OrderTotal]
)
