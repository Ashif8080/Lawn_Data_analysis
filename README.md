# Lawn_Org Marketplace Analytics: ETL Pipeline & Metric Data Engineering

## Overview
This repository contains the end-to-end data transformation and exploratory data analysis (EDA) pipeline for **Lawn_Org** marketplace booking data[cite: 1, 2]. The project focuses on data quality engineering, arithmetic and statistical integrity, and optimizing relational data structures for downstream Business Intelligence (BI) tools, specifically **Google Looker Studio**.

---

## Technical Architecture & Design Decisions

### 1. Data Quality & Type Safety Engineering
* **Geographic Field Types:** `zip_code` was explicitly cast to string (`VARCHAR`/`object`) rather than integer to prevent leading zero truncation (e.g., `02108`) and to maintain compatibility with Looker Studio’s **Geo -> Postal Code** data engine.
* **Date Parsing & Time Series:** `booking_date` was converted to `datetime64[ns]` ISO formats, enabling native time-series drill-downs and period-over-period aggregations across 2025 (248 records) and 2026 (752 records).

### 2. Missing Value (`NULL`) Strategy & Statistical Preservation
Rather than applying naive imputation (e.g., calling `.fillna(0)` which distorts distribution metrics without modifying DataFrames in-place unless explicitly assigned), missing values were intentionally preserved as `NULL` to maintain statistical validity:
* **Duration Metrics (`duration_minutes`):** `NULL` values (48 missing entries) were retained. Imputing `0` for unlogged or cancelled jobs distorts the true arithmetic mean ($\mu$) and variance ($\sigma^2$), artificially pulling down average service completion times.
* **Financial Metrics (`final_price` & `quoted_price`):** Retained `NULL` entries (73 missing `final_price`, 42 missing `quoted_price`) for unbilled or incomplete transactions to ensure aggregate metrics like `AVG()` and `SUM()` in Looker Studio and SQL compute strictly over valid financial records.
* **Geographic Data (`zip_code`):** Preserved 518 missing values as explicit null/unspecified dimensions to avoid misclassifying non-mapped customer regions.

---

## Database & BI Schema Optimization

| Column Name | Input Type | Transformed Type | Looker Studio Semantic Type | Data Engineering Operations & Logic |
| :--- | :--- | :--- | :--- | :--- |
| `booking_id` | Int | String | Text / Identifier | Primary Key tracking (1,000 unique records) |
| `customer_id` | Int | Int64 | Metric / Dimension | Customer repeat rate & retention modeling (1:1 record ratio)[cite: 2] |
| `provider_id` | Int | Int64 | Metric / Dimension | Supply-side fulfillment & provider tracking (1:1 record ratio)[cite: 2] |
| `booking_date` | String | Datetime | Date (YYYY-MM-DD) | Extracted dimensions for YOY / MOM time-series analysis |
| `service_type` | String | String | Category / Dimension | Standardized categories: Hedge Trimming (353), Lawn Mowing (345), Weed Control (302) |
| `status` | String | String | Category / Dimension | Standardized stages: Pending (335), Completed (333), Confirmed (332) |
| `quoted_price` | Float | Float64 | Currency (USD) | Baseline quote pricing for variance calculation[cite: 1, 2] |
| `final_price` | Float | Float64 | Currency (USD) | Billed revenue; retains `NULL` for open/unbilled orders |
| `duration_minutes`| Float | Float64 | Numeric Metric | Service duration; preserves `NULL` to protect arithmetic mean |
| `zip_code` | Mixed | String | Geo -> Postal Code | Standardized postal string (482 present, 518 missing) |
| `customer_rating` | Int | Int64 | Numeric Metric | Customer satisfaction rating (Average: 2.98/5 across uniform 1–5 scale) |

---

## Statistical & Arithmetic Analysis Highlights

1. **Revenue Variance Analysis:**
   $$\text{Pricing Variance} = \text{final\_price} - \text{quoted\_price}$$
   Evaluates pricing leakage and refund anomalies across provider service categories without distorting unbilled orders.

2. **Fulfillment & Supply Efficiency:**
   Tracks marketplace fulfillment ratios while isolating unlogged job durations using explicit boolean data quality flags:
   $$\text{Is Missing Duration} = (\text{status} = \text{'completed'}) \land (\text{duration\_minutes} \text{ IS NULL})$$

3. **Customer Satisfaction Distribution:**
   Average rating across completed and logged orders is **2.98 / 5.0**, uniformly distributed (~200 occurrences per rating band 1–5), providing a clear benchmark for provider performance monitoring.

4. **BI Metric Aggregations:**
   Designed to seamlessly interface with Google Looker Studio calculated fields:
   * **Cleaned Geo Mapping:** `CASE WHEN zip_code IS NULL THEN 'Unspecified' ELSE zip_code END`
   * **Active Completion Rate:** `COUNT(CASE WHEN status = 'completed' THEN booking_id END) / COUNT(booking_id)`

---
