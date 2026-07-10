# RetailMax-Group-Business-Intelligence-Analytics-End-to-End-BI-Pipeline
Excel + PostgreSQL + Python + Power BI Project | Retail Transactional Data | From Raw Data to Interactive Business Intelligence Dashboard

## Table Of Contents
* [Project Overview](#project-overview)
* [Tools & Technologies](#tools--technologies)
* [Dataset Overview](#dataset-overview)
* [Data Cleaning](#data-cleaning)
* [Exploratory Analysis](#exploratory-analysis)
* [Data Modeling](#data-modeling)
* [Power BI Dashboard](#power-bi-dashboard)
* [Key Metrics](#key-metrics)
* [Key Insights](#key-insights)
* [Key Conversion Insights](#key-conversion-insights)
* [Executive Summary](#executive-summary)
* [Recommendations](#recommendations)

---

### Project Overview
This project delivers a fully end-to-end Business Intelligence solution for RetailMax Group, a multi-channel retail organisation operating across Online, Store, and Marketplace sales channels. Using 50,000 transactional records spanning three years (2023–2025), the pipeline covers complete data cleaning, exploratory analysis in both Python and SQL, a PostgreSQL relational database, and a 6-page interactive Power BI dashboard — enabling leadership to make evidence-based decisions around pricing, customer targeting, channel investment, and revenue leakage for the first time from a single, unified analytical framework.

---

### Tools & Technologies
* **Microsoft Excel / Google Sheets** → Initial data inspection, casing standardisation, and format correction
* **Python (Jupyter Notebook)** → Data cleaning, exploratory data analysis, and data type corrections
* **PostgreSQL (pgAdmin 4)** → Relational database design, SQL cleaning, and business question queries
* **pandas / NumPy / SQLAlchemy / psycopg2** → Data processing and Python-to-PostgreSQL pipeline
* **Power BI Desktop** → 6-page interactive dashboard with DAX measures, calculated columns, and star schema model
* **GitHub** → Version control and project documentation
* **SQL** → Star schema design, data cleaning, and analytical queries

---

### Dataset Overview

**Sales Transactions Table**

| Column | Description |
|--------|-------------|
| Transaction_ID | Unique identifier for each sales transaction |
| Order_Number | Order reference number |
| Customer_ID | Foreign key linking to Customer table |
| Product_ID | Foreign key linking to Product table |
| Store_ID | Foreign key linking to Store table |
| Date_ID | Date of transaction stored as YYYYMMDD integer |
| Quantity | Number of units sold |
| Unit_Price | Price per unit |
| Discount_Percent | Discount applied as a decimal (e.g. 0.15 = 15%) |
| Revenue | Total transaction revenue |
| Cost | Total transaction cost |
| Profit | Transaction profit (Revenue - Cost) |
| Sales_Channel | Channel through which sale was made (Online, Store, Marketplace) |
| Payment_Method | Payment method used (Card, Cash, PayPal) |
| Order_Status | Transaction status (Completed, Returned, Cancelled) |

**Customer Table**

| Column | Description |
|--------|-------------|
| Customer_ID | Unique customer identifier |
| Customer_Name | Customer full name |
| Gender | Customer gender (Female, Male) |
| Age | Customer age in years |
| City | Customer city of residence |
| Region | UK statistical region |
| Registration_Date | Date customer account was created |
| Loyalty_Member | Boolean flag indicating loyalty programme membership |
| Customer_Segment | Customer tier (VIP, Gold, Silver, Bronze, Unclassified) |
| Email | Customer email address |

**Product Table**

| Column | Description |
|--------|-------------|
| Product_ID | Unique product identifier |
| Product_Name | Product name |
| Category | Product category (Electronics, Home Appliances, Lifestyle, Office Tech) |
| Subcategory | Product subcategory (Sub-1 through Sub-20) |
| Brand | Product brand (Nova, Elite, Vertex, Prime) |
| Unit_Cost | Cost price per unit |
| Selling_Price | Retail selling price |
| Launch_Date | Product launch date |

**Store Table**

| Column | Description |
|--------|-------------|
| Store_ID | Unique store identifier |
| Store_Name | Store name |
| Region | Store region (North, South, East, West) |
| Store_Type | Store type (Online, Physical) |
| Opening_Date | Store opening date |
| Manager_ID | Manager identifier |

**Sample Preview**

| Transaction_ID | Customer_ID | Product_ID | Revenue | Profit | Sales_Channel | Order_Status |
|----------------|-------------|------------|---------|--------|---------------|-------------|
| 1 | 4166 | 256 | 333.64 | 181.19 | Online | Cancelled |
| 2 | 1555 | 425 | 335.34 | -8.50 | Marketplace | Returned |
| 3 | 2847 | 752 | 6,226.73 | 1,595.28 | Online | Completed |

---

### Data Cleaning

10 data quality issues were identified and resolved across all four tables:

1. **Sales Channel casing** — Standardised "online" and "Online" to a single consistent value "Online" across 12,512 rows
2. **Gender values** — Consolidated "F"/"Female" and "M"/"Male" into two clean labels: "Female" and "Male" across 5,000 customer records
3. **City casing** — Unified "manchester", "Manchester", and "MANCHESTER" into consistent title case across all city names using `.str.title()` in Python and `INITCAP(TRIM(...))` in SQL
4. **Customer Region** — Built a City→Region lookup (London → London, Manchester/Liverpool → North West, Birmingham → Midlands, Leeds → Yorkshire) replacing blanks with correct ONS UK statistical regions instead of defaulting to "Unknown"
5. **Missing Email addresses** — Generated unique placeholder emails (e.g. customer42@mail.com) for 310 blank records using each customer's own ID
6. **Orphaned Customer IDs** — Added 50 placeholder customer rows for IDs 5001–5050 referenced in Sales but missing from the Customer table (476 affected transactions)
7. **Orphaned Product IDs** — Added 10 placeholder product rows for IDs 1001–1010 referenced in Sales but missing from the Product table (515 affected transactions)
8. **Age column data type bug** — Fixed a silent bug where adding placeholder rows changed Age from numeric to text (object dtype); corrected using `pd.to_numeric()` after root cause investigation
9. **Date_ID conversion** — Converted the 8-digit integer date (e.g. 20240914) into a proper calendar date using `pd.to_datetime(..., format='%Y%m%d')` for time-based analysis
10. **Negative Revenue rows** — Confirmed 48 negative revenue/quantity rows as return adjustments and retained them in the dataset for analytical transparency

---

### Exploratory Analysis

- Total Revenue, Profit and Margin
```sql
SELECT
  ROUND(SUM(revenue)::NUMERIC, 2) AS total_revenue,
  ROUND(SUM(profit)::NUMERIC, 2) AS total_profit,
  ROUND(SUM(profit) / SUM(revenue) * 100, 2) AS profit_margin_pct
FROM sales_transactions;
```

- Revenue by Sales Channel
```sql
SELECT
  sales_channel,
  ROUND(SUM(revenue)::NUMERIC, 2) AS revenue,
  ROUND(SUM(profit)::NUMERIC, 2) AS profit,
  COUNT(*) AS orders,
  ROUND(SUM(profit) / SUM(revenue) * 100, 2) AS margin_pct
FROM sales_transactions
GROUP BY sales_channel
ORDER BY revenue DESC;
```

- Revenue Lost to Returns and Cancellations
```sql
SELECT
  SUM(CASE WHEN order_status = 'Completed' THEN revenue ELSE 0 END) AS actual_revenue,
  SUM(CASE WHEN order_status = 'Returned'  THEN revenue ELSE 0 END) AS returned_revenue,
  SUM(CASE WHEN order_status = 'Cancelled' THEN revenue ELSE 0 END) AS cancelled_revenue,
  SUM(revenue) AS total_gross_revenue,
  ROUND((SUM(CASE WHEN order_status IN ('Returned','Cancelled') THEN revenue ELSE 0 END) / SUM(revenue)) * 100, 2) AS leakage_pct
FROM sales_transactions;
```

- Top Product Category by Revenue and Profit
```sql
SELECT
  p.category,
  ROUND(SUM(s.revenue)::NUMERIC, 2) AS revenue,
  ROUND(SUM(s.profit)::NUMERIC, 2) AS profit,
  COUNT(*) AS orders,
  ROUND(SUM(s.profit) / SUM(s.revenue) * 100, 2) AS margin_pct
FROM sales_transactions s
JOIN product p ON s.product_id = p.product_id
GROUP BY p.category
ORDER BY revenue DESC;
```

- Discount Band vs Profit Margin
```sql
SELECT
  CASE
    WHEN discount_percent = 0 THEN 'No discount'
    WHEN discount_percent <= 0.10 THEN '0-10%'
    WHEN discount_percent <= 0.20 THEN '10-20%'
    WHEN discount_percent <= 0.30 THEN '20-30%'
    ELSE '30%+'
  END AS discount_band,
  ROUND(SUM(revenue)::NUMERIC, 2) AS revenue,
  ROUND(SUM(profit)::NUMERIC, 2) AS profit,
  ROUND(SUM(profit) / SUM(revenue) * 100, 2) AS margin_pct,
  COUNT(*) AS orders
FROM sales_transactions
GROUP BY discount_band
ORDER BY MIN(discount_percent);
```

- Top 20 Customers by Customer Lifetime Value
```sql
SELECT
  c.customer_id,
  c.customer_name,
  ROUND(SUM(s.profit)::NUMERIC, 2) AS lifetime_profit,
  COUNT(*) AS total_orders,
  ROUND(SUM(s.revenue)::NUMERIC, 2) AS lifetime_revenue
FROM sales_transactions s
JOIN customer c ON s.customer_id = c.customer_id
GROUP BY c.customer_id, c.customer_name
ORDER BY lifetime_profit DESC
LIMIT 20;
```

- Revenue by Age Band
```sql
SELECT
  CASE
    WHEN c.age IS NULL OR c.age < 25 THEN 'Unknown/<25'
    WHEN c.age <= 34 THEN '25-34'
    WHEN c.age <= 44 THEN '35-44'
    WHEN c.age <= 54 THEN '45-54'
    WHEN c.age <= 64 THEN '55-64'
    ELSE '65+'
  END AS age_band,
  ROUND(SUM(s.revenue)::NUMERIC, 2) AS revenue,
  ROUND(SUM(s.profit)::NUMERIC, 2) AS profit,
  COUNT(DISTINCT s.customer_id) AS customers
FROM sales_transactions s
JOIN customer c ON s.customer_id = c.customer_id
GROUP BY age_band
ORDER BY MIN(c.age);
```

- Store-level Leaderboard
```sql
SELECT
  st.store_name,
  st.region,
  ROUND(SUM(s.revenue)::NUMERIC, 2) AS revenue,
  ROUND(SUM(s.profit)::NUMERIC, 2) AS gross_profit,
  COUNT(*) AS orders
FROM sales_transactions s
JOIN store st ON s.store_id = st.store_id
GROUP BY st.store_name, st.region
ORDER BY gross_profit DESC
LIMIT 10;
```

---

### Data Modeling

Implemented a Star Schema for performance and clarity:

**Fact Table:** Sales Transactions (transaction-level data)

**Dimensions:**
- Customer (Customer ID, Name, Gender, Age, City, Region, Loyalty Member, Segment)
- Product (Product ID, Name, Category, Subcategory, Brand, Unit Cost, Selling Price)
- Store (Store ID, Store Name, Region, Store Type, Opening Date)
- Calendar (Date, Month Number, Month Name, Quarter, Year)

**Relationships (One-to-Many):**
- Customer[Customer_ID] → Sales_Transactions[Customer_ID]
- Product[Product_ID] → Sales_Transactions[Product_ID]
- Store[Store_ID] → Sales_Transactions[Store_ID]
- Calendar[Date] → Sales_Transactions[Sale_Date]

**Calendar Table:**
- Built using DAX `CALENDAR()` function covering the full 2023–2025 period
- Columns added: Year, Quarter, MonthNumber, MonthName, Weekday
- Connected to Sales_Transactions[Sale_Date] to enable all time intelligence calculations

**Calculated Columns:**
- `Discount_Band` — groups each transaction's discount percentage into five labelled bands (No Discount, 0–10%, 10–20%, 20–30%, 30%+) using `SWITCH(TRUE(), ...)`
- `Age_Band` — groups customers into six age ranges (Unknown/<25, 25–34, 35–44, 45–54, 55–64, 65+) using `SWITCH(TRUE(), ...)`, with blank ages placed transparently in the Unknown group

**DAX Measures:**
- Total Revenue, Actual Revenue, Gross Profit, Actual Profit, Profit Margin %, Actual Profit Margin %
- Total Orders, Actual Order Number, Return Order Number, Cancellation Order Number
- Online Revenue, Actual Online Revenue, Returned Revenue, Cancelled Revenue
- Revenue Lost to R/C, Revenue Leakage %, Return Rate %, Cancellation Rate %
- Total Customers, Active Customers, Inactive Customers, Customer CLV, Retention Rate %
- Actual Avg Discount %, No Discount Revenue, Discounted Revenue, Total Stores

---

### Power BI Dashboard

The Power BI dashboard includes 5 pages with 30+ DAX measures:

* **Executive Sales Overview** — Total Revenue, Total Orders, Gross Profit, Profit Margin %, Avg Transaction Value; Revenue by Month, Profit by Channel and Category, Revenue by Region, Revenue by Sales Channel, Customers by Gender, Profit Margin % by Discount Band
  <img width="709" height="404" alt="Screenshot 2026-07-10 170224" src="https://github.com/user-attachments/assets/766815e1-33fc-43a8-9a4b-ee6158b018d9" />

* **Revenue & Profit Analysis** — Total Revenue, Online Revenue, Return Order %, Revenue Leakage %, Revenue Lost to R/C; Top Product Category, Revenue Lost to R/C by Category, Revenue by Business Traffic, Top Brand by Profit, Channel by Revenue vs Gross Profit, Sales Channel vs Order Status, Revenue by Subcategory
  <img width="713" height="404" alt="Screenshot 2026-07-10 170433" src="https://github.com/user-attachments/assets/1241bdbb-8279-4fa2-b700-7b66b3a3e81a" />

* **Customer Intelligence** — Total Revenue, Online Revenue, Gross Profit, Total Stores, Avg % of Discount Applied; Revenue by Discount Band, Top 20 Customers, Revenue by Age Band, Best Selling Product, Payment Method, Profit by Loyalty Member, Store-level Leaderboard
  <img width="711" height="401" alt="Screenshot 2026-07-10 170555" src="https://github.com/user-attachments/assets/7375b1ac-d11b-4da9-be57-2d46b90c6d01" />

* **Executive Scorecard** — 12 KPI cards across Revenue, Profit, Order, and Customer dimensions in one unified leadership view
  <img width="715" height="401" alt="Screenshot 2026-07-10 170657" src="https://github.com/user-attachments/assets/527af42f-70c2-4699-931b-2b44618ce653" />

* **Order Performance & Fulfilment** — Order Status by Category, Order Status by Channel, Return Trend by Month, Leakage by Region, Profit Margin % by Discount Band, Order Count by Discount Band
  <img width="711" height="404" alt="Screenshot 2026-07-10 170802" src="https://github.com/user-attachments/assets/a9b06017-993b-48cd-a7c5-43ef05b37aea" />


**Connection:** Import Mode → PostgreSQL (retailmax database via pgAdmin 4)

---

### Key Metrics
- **Total Transactions:** 50,000
- **Total Revenue (Gross):** £112.08M
- **Actual Revenue (Completed):** ~£37.6M
- **Gross Profit:** £27.87M
- **Profit Margin %:** 24.87%
- **Revenue Lost to Returns/Cancellations:** £74.47M
- **Revenue Leakage %:** 66.45%
- **Return Order %:** 33.17%
- **Total Customers:** 5,000
- **Total Stores:** 50
- **Average Transaction Value:** £2,241.62
- **Average Discount Applied:** 20%
- **Date Range:** January 2023 – December 2025
- **Top Revenue Region:** West (£38M)
- **Top Store:** Store 1, North (£590,296 gross profit)
- **Top Age Band by Revenue:** 65+ (£50.3M)

---

### Key Insights

1. **66.45% of gross revenue never converts** — £74.47M is lost to returns and cancellations across all three years, with only one-third of all 50,000 transactions reaching Completed status. Lifestyle and Home Appliances carry the largest leakage burden at £19M each.
2. **Discounting past 10% destroys margin** — Profit margin falls steadily from 40.3% on undiscounted orders to just 7.1% on orders discounted above 30%, with no evidence that heavier discounting drives enough additional volume to compensate.
3. **Online leads but does not dominate** — Online generates £56.16M (exactly half of gross revenue), with Store and Marketplace splitting the remainder almost equally at £28M each. No single channel carries a disproportionate risk.
4. **The 65+ customer segment drives disproportionate revenue** — Customers aged 65 and above generate £50.3M — more than the next three age bands combined, reflecting a genuinely older-skewing customer base confirmed across multiple independent analyses.
5. **Non-loyalty customers currently out-profit loyalty members** — Non-loyalty customers contribute £14.87M in profit versus £13.0M from loyalty members, suggesting the programme is not yet reaching or incentivising the business's highest-value buyers.
6. **East region lags significantly** — East generates £18M in revenue versus West's £38M and North's £34M, flagging a potential growth opportunity or execution gap worth investigating at store level.
7. **Margin is remarkably consistent across channels and regions** — All three channels and all four regions hold within 24–25% margin, confirming the profitability challenge is structural and business-wide, not channel or region specific.
8. **Store 1 (North) leads all 50 stores** — £590,296 in gross profit, meaningfully ahead of the next stores, suggesting a model worth studying and replicating across underperforming locations.

---

### Key Conversion Insights

1. **£74.47M in revenue never converts to income** — With only one-third of transactions completing, reducing leakage by even 10% would add over £7M directly to net income.
2. **Discounting past 10% destroys margin without recovering it** — Margin halves once discount exceeds 10% and falls to 7% beyond 30%. No evidence of sufficient volume uplift to compensate.
3. **Online is the dominant channel but no single channel carries the risk** — The balanced three-way split means no single channel failure would collapse the business, but Online's efficiency advantage is not yet being fully leveraged.
4. **The 65+ customer segment drives disproportionate revenue** — £50.3M from customers aged 65+, more than four times any other single age band. Marketing and product decisions assuming a younger customer base are misaligned with where the actual revenue comes from.
5. **Non-loyalty customers currently out-profit loyalty members** — The loyalty programme is either attracting the wrong segment, offering misaligned incentives, or has not yet reached the customers who drive the most profit — all addressable with the customer intelligence now available in this dashboard.

---

### Executive Summary

RetailMax Group generated £112.08M in gross revenue across 50,000 transactions and 50 stores during the 2023–2025 period, at a consistent 24.87% profit margin across all channels and regions. However, two structural issues are quietly costing the business significant profit. First, 66.45% of gross revenue — £74.47M — is lost to returns and cancellations before it ever converts to real income, with only one-third of all transactions completing successfully. Second, the business's discounting strategy is materially eroding margin: profit drops from 40.3% on undiscounted orders to just 7.1% on orders discounted above 30%, with an average discount of 20% applied across the entire transaction base.

This Business Intelligence dashboard was built to address a core gap identified at the outset: the absence of a centralised analytical framework giving leadership a unified, reliable view of business performance across channels, regions, products, and customers. The solution transforms four raw Excel datasets — sales, customers, products, and stores — through a full cleaning and modelling pipeline into a 6-page interactive Power BI dashboard, enabling evidence-based decisions around pricing, customer targeting, channel investment, and operational planning for the first time.

---

### Recommendations

* **Investigate return and cancellation drivers by category** — Lifestyle and Home Appliances account for £19M in lost revenue each. Start root-cause analysis there before any business-wide policy change, since these are also the top revenue-generating categories.
* **Cap or tier discounting above 10%** — Review whether deep discounts are genuinely driving incremental volume, or simply eroding margin on sales that would have happened at full price. A 10% cap would protect the 36.45% margin band without meaningful volume risk.
* **Review the loyalty programme's targeting** — Non-loyalty customers currently out-profit loyalty members. Investigate who the programme is actually attracting and whether its incentives align with the customers RetailMax most wants to retain.
* **Study Store 1 (North) as a model** — RetailMax's top-performing store by gross profit. Document what differs — staffing, product mix, local market conditions — and assess whether it can be replicated at underperforming stores, particularly in the East region.
* **Prioritise the 65+ customer segment in marketing and product decisions** — This segment generates more than four times the revenue of any other age band. Current strategies that do not explicitly account for this demographic are leaving significant revenue potential on the table.
* **Separate Actual Revenue from Gross Revenue in all executive reporting** — Presenting £112.08M as the headline figure without clearly surfacing the £74.47M leakage alongside it risks giving leadership a misleadingly optimistic view of the business's true conversion performance.

---

*By Tunde Adebayo | Data Analytics Consultant | Amdari UK*
*Tools: Excel · Python · PostgreSQL · Power BI · SQL*
*GitHub: [tundesta-data](https://github.com/tundesta-data)*
