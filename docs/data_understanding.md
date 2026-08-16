# Data Understanding Report

**Project:** Olist E-Commerce Performance Analysis
**Phase:** 1 — Data Understanding
**Source notebook:** `notebooks/01_data_understanding.ipynb`

---

## 1. Overview

The dataset contains 9 relational tables covering Olist marketplace orders, customers, sellers, products, payments, and reviews. All tables load cleanly with no encoding or parsing issues. Row counts and schema summaries are below.

| Table | Rows | Columns | Join Key(s) |
|---|---|---|---|
| orders | 99,441 | 8 | order_id → customer_id |
| customers | 99,441 | 5 | customer_id |
| order_items | 112,650 | 7 | order_id, product_id, seller_id |
| payments | 103,886 | 5 | order_id |
| reviews | 99,224 | 7 | order_id (review_id not unique — see §3) |
| products | 32,951 | 9 | product_id |
| sellers | 3,095 | 4 | seller_id |
| geolocation | 1,000,163 | 5 | zip_code_prefix (many-to-many) |
| category_translation | 71 | 2 | product_category_name |

**Schema logic:** `orders` is the central table. `order_items` links orders to products and sellers (one row per item, so multi-item orders span multiple rows). `payments` and `reviews` link to orders 1-to-many and 1-to-1 respectively (with the exception noted in §3). `customers`/`sellers` link to `geolocation` via zip code prefix, which is a many-to-many lookup, not a clean 1-to-1 key.

## 2. Usable Date Range

Raw timestamp range: **2016-09-04 to 2018-10-17**.

However, monthly order volume shows the actual usable range is narrower:

| Period | Orders | Status |
|---|---|---|
| Sep–Dec 2016 | 4 / 324 / 0 / 1 | Not usable — negligible volume, includes a zero month |
| **Jan 2017 – Aug 2018** | **800 to 7,544/month** | **Usable range for all trend analysis** |
| Sep–Oct 2018 | 16 / 4 | Not usable — data collection cutoff, not a real decline |

**Decision:** All time-series and trend analysis in this project will be scoped to **January 2017 – August 2018**. Data outside this window will be excluded from trend charts and flagged if referenced elsewhere, to avoid implying business activity dropped to zero when this is actually a data boundary artifact.

## 3. Data Quality Issues

**Orders table** — missing delivery-chain dates are expected, not errors:
- `order_approved_at`: 160 missing
- `order_delivered_carrier_date`: 1,783 missing
- `order_delivered_customer_date`: 2,965 missing

These nulls correlate with non-`delivered` order statuses (see below) — an order that was canceled or is still processing naturally has no delivery date yet.

**Order status distribution:**

| Status | Count | % |
|---|---|---|
| delivered | 96,478 | 97.0% |
| shipped | 1,107 | 1.1% |
| canceled | 625 | 0.6% |
| unavailable | 609 | 0.6% |
| invoiced | 314 | 0.3% |
| processing | 301 | 0.3% |
| created | 5 | <0.1% |
| approved | 2 | <0.1% |

**Decision:** Revenue and delivery-performance analysis will be scoped to `order_status == 'delivered'` only. Non-delivered orders (~3% of the dataset) will be reported separately as a cancellation/fulfillment-failure rate rather than mixed into revenue figures, since including them would understate delivery performance and misrepresent realized revenue.

**Reviews table — duplicate `review_id`:**
Whole-row duplicate check returned 0, but checking `review_id` alone found **814 duplicate values**. This reflects Olist's review resubmission process (a customer can update a review, generating a new row with the same `review_id`). **Decision:** deduplicate by keeping the most recent `review_answer_timestamp` per `review_id` before joining reviews to orders, to avoid inflating order counts through a 1-to-many join.

**Products table:** 610 products missing category, name length, description length, and photo count (same incomplete-listing pattern — likely legacy/unpublished listings). 2 products missing all physical dimensions. **Decision:** exclude these from category-level analysis; too small a share (1.9%) to materially affect findings, but will be noted rather than silently dropped.

**Geolocation table:** 261,831 duplicate rows out of 1,000,163. This is expected — the table logs repeated lat/lng/zip combinations from multiple source records. **Decision:** deduplicate to one representative row per `zip_code_prefix` (e.g., first occurrence or centroid) before using this table for any city/state-level joins, since it is not natively a clean lookup table.

## 4. Limitations

- **Single platform, single country.** Findings describe Olist's Brazilian marketplace only and should not be generalized to e-commerce broadly or to other platforms.
- **Time-bound.** Usable data covers 20 months (Jan 2017–Aug 2018). Any seasonal claims (e.g., Black Friday effects) are based on a maximum of two observed cycles and should be stated with appropriate caution.
- **Free-text review fields** are largely unpopulated (title 88% missing, message 59% missing) and in Portuguese — not suitable for reliable sentiment/NLP analysis without translation and significant missing-data handling.
- **Geolocation is approximate**, keyed to zip-prefix rather than exact address, so any distance/logistics analysis is directional, not precise.

## 5. Next Steps

Proceed to Phase 2 (cleaning + core analysis) using the decisions documented above:
1. Filter to `delivered` orders for revenue/delivery analysis; report cancellation rate separately
2. Deduplicate `reviews` by `review_id` (keep latest) before joining
3. Deduplicate `geolocation` by `zip_code_prefix` before any location joins
4. Scope all trend/time-series work to Jan 2017–Aug 2018
5. Flag category-level findings for the 610 products with missing category data
