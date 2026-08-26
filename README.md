# Olist E-Commerce Customer Satisfaction Analysis

## Google Data Analytics Capstone Project — Case Study 2

---

## Business Task

What factors drive low review scores on Olist's marketplace, and what should the business prioritize to improve customer satisfaction?

---

## Data Source

- Data was downloaded from the [Olist Brazilian E-Commerce Public Dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) on Kaggle, originally collected by Olist Store.
- The dataset covers ~100,000 orders across 9 relational tables (orders, order items, products, sellers, customers, reviews, payments, and geolocation) from 2016–2018.
- Olist was chosen over a second common capstone dataset because e-commerce/retail is one of the highest-volume sectors for analyst hiring, and the multi-table structure gives more practice with real joins than a single-table dataset would.

### Data Credibility (ROCCC Evaluation)

| Criteria | Assessment |
|---|---|
| **Reliable** | Checked primary and join keys across all tables (Orders, Reviews, Products, Order Items) for nulls and duplicates before trusting them. Order and Customer IDs had no nulls or duplicates; the nulls and duplicates that did exist elsewhere were traced to explainable causes, not data corruption |
| **Original** | We are accessing data directly from the provider (Olist), not through a third-party aggregator |
| **Comprehensive** | 9 tables covering orders, products, customers, sellers, reviews, geolocation, and payments — enough to answer every guiding question below |
| **Current** | Covers 2016–2018. Not current data, but sufficient for demonstrating methodology in a case study context |
| **Cited** | Publicly available on Kaggle under the CC BY-NC-SA 4.0 license |

---

## Tools Used

- **Google BigQuery** — Data cleaning, joins, and SQL analysis
- **Tableau Public** — Data visualization and dashboard
- **GitHub** — Portfolio and documentation

---

## Guiding Questions

The business task was broken into five analyzable sub-questions:

1. Does delivery time (or delay vs. estimate) correlate with review score?
2. Do certain product categories get systematically lower reviews?
3. Do specific sellers have consistently lower ratings — and why (delivery speed, order volume, product mix)?
4. Does order value or payment type/installments relate to satisfaction?
5. Are there regional (customer or seller location) patterns in review scores?

This case study focuses on questions 1, 2, and 3, which together explain the largest share of variation in review scores.

---

## Data Cleaning & Processing

### Orders Table

- **Total rows:** 99,441
- Order ID and Customer ID had no nulls and no duplicates.

| Issue | Count |
|---|---|
| order_delivered_customer_date null (order not delivered) | 2,957 |
| order_delivered_customer_date null (status = delivered) | 8 |
| Canceled orders with a delivery date still recorded | 6 |

**Undelivered / inconsistent orders — Removed.** An order that was never delivered, or whose status and delivery date contradict each other, cannot have a meaningful delivery delay calculated for it. All three groups above were excluded by filtering to `order_status = 'delivered' AND order_delivered_customer_date IS NOT NULL`.

### Reviews Table

- **Total rows:** 99,224
- review_comment_message had 58,247 nulls — not used in this analysis, no action needed.

| Issue | Count |
|---|---|
| Duplicate review_id (same review content, linked to more than one order_id) | 814 |
| Duplicate order_id (genuinely different reviews for the same order) | 551 |

**Duplicate reviews — Removed via a two-pass ROW_NUMBER().** The first pass deduplicates by review_id, since the same review sometimes gets attached to more than one order from a single checkout session — a known quirk in this dataset. The second pass deduplicates by order_id, keeping the most recent review when a customer genuinely left more than one for the same order. This guarantees exactly one review_score per order.

### Products Table

- **Total rows:** 32,951
- Product ID had no nulls and no duplicates. product_category_name had 610 nulls.
- Two near-duplicate category pairs were found: `eletrodomesticos` / `eletrodomesticos_2` and `casa_conforto` / `casa_conforto_2`. Checked order counts and average price for each pair — both differed meaningfully, so these were kept as separate categories rather than merged.

**Category names — Translated.** Category names are stored in Portuguese, so they were joined to Olist's English translation table, and nulls were bucketed as `'unknown'` using COALESCE. After the join, the `'unknown'` bucket grew from 610 to 623 — 2 real categories (`portateis_cozinha_e_preparadores_de_alimentos` and `pc_gamer`) turned out to be missing from the translation table entirely, so those two were translated manually instead of being lumped in with the true nulls.

### Order Items Table

- **Total rows:** 112,650
- Order ID and Product ID had no nulls.
- 29,957 unique order_id + product_id combinations; 10,990 exact duplicate rows, consistent with an order containing more than one unit of the same product.
- Verified every product_id and order_id has a match in Products and Orders — 0 orphan rows found.

**No cleaning required.** Nulls, duplicates, and orphan keys were all checked explicitly rather than assumed clean. The duplicate rows don't distort category- or seller-level analysis, since the analysis only checks whether an order touched a given category or seller, not how many item rows it produced.

### Multi-Category and Multi-Seller Orders

Since a single order can only map to one review_score, we checked what share of orders touch more than one category or more than one seller before running the category- and seller-level analysis:

| Check | Result |
|---|---|
| Orders containing products from more than one category | 0.74% |
| Orders containing products from more than one seller | 1.29% |

Given how small both shares are, these orders were excluded from their respective analyses rather than letting one review_score fan out across multiple categories or sellers.

### New Column Added

One new column, `delay_days`, was calculated as `delivered_customer_date − estimated_delivery_date` (positive = late, negative = early), then grouped into four buckets: Early, On Time, Late (1–7 days), Very Late (8+ days).

### Cleaning Query

```sql
WITH
  cleaned_orders AS (
    SELECT
        order_id,
        DATE_DIFF(order_delivered_customer_date, order_estimated_delivery_date, DAY) AS delay_days
    FROM `OLIST_REVIEWS.Orders`
    WHERE order_delivered_customer_date IS NOT NULL
      AND order_status = 'delivered'
  ),
  reviews_step1 AS (
    SELECT order_id, review_id, review_score, review_creation_date
    FROM `OLIST_REVIEWS.Reviews`
    QUALIFY ROW_NUMBER() OVER (PARTITION BY review_id ORDER BY review_creation_date DESC) = 1
  ),
  cleaned_review AS (
    SELECT order_id, review_score
    FROM reviews_step1
    QUALIFY ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY review_creation_date DESC) = 1
  )
SELECT
    CASE
        WHEN delay_days < 0 THEN '1. Early'
        WHEN delay_days = 0 THEN '2. On Time'
        WHEN delay_days BETWEEN 1 AND 7 THEN '3. Late (1-7 days)'
        ELSE '4. Very Late (8+ days)'
    END AS delivery_bucket,
    COUNT(*) AS order_count,
    ROUND(AVG(review_score), 2) AS avg_review_score
FROM cleaned_orders AS O
INNER JOIN cleaned_review AS R ON O.order_id = R.order_id
GROUP BY delivery_bucket
ORDER BY delivery_bucket;
```

All 10 queries used in this project — including the products, order items, category, and seller cleaning and analysis — are available in the `queries/` folder.

### After Cleaning

- **Rows removed:** 2,971 (Orders) and 1,089 (Reviews)
- Products and Order Items required no row removal — only category labels were adjusted.

---

## Analysis & Key Findings

### 1. Delivery Delay vs. Review Score

| Delivery Status | Order Count | Avg Review Score |
|---|---|---|
| Early | 86,271 | 4.30 |
| On Time | 2,719 | 4.10 |
| Late (1–7 days) | 3,587 | 2.72 |
| Very Late (8+ days) | 2,760 | 1.69 |

**Finding:** Orders delivered after the estimated arrival date get substantially lower ratings — average score drops from 4.30 for early deliveries to 1.69 for orders more than a week late.

| Days Late | Order Count | % of Late Orders | Avg Review Score |
|---|---|---|---|
| 1 | 818 | 22.80% | 3.73 |
| 2 | 534 | 14.89% | 3.18 |
| 3 | 493 | 13.74% | 2.68 |
| 4 | 435 | 12.13% | 2.49 |
| 5 | 434 | 12.10% | 2.19 |
| 6 | 405 | 11.29% | 1.81 |
| 7 | 468 | 13.05% | 1.92 |

**Finding:** The score decreases gradually day by day rather than dropping instantly at a single threshold — every additional day of delay costs further satisfaction. Day 7's small uptick sits within normal sample variation, since its order count is close to day 6's.

**Additional insight:** We also compared the length of the *promised* delivery window against the effect of actually *missing* that promise. Scores declined only mildly (roughly 4.7 down to 4.0) as the promised window grew from 2 to 50 days, compared to a steep ~2.5-point drop once an order actually arrived late. This tells us that meeting the delivery promise matters far more to customer satisfaction than how long that promise is.

### 2. Product Category vs. Review Score

Restricted to single-category orders and categories with at least 90 reviews, to avoid a handful of orders driving a category's average.

**Top 5 categories:**

| Category | Avg Review Score | Total Reviews |
|---|---|---|
| books_general_interest | 4.47 | 503 |
| costruction_tools_tools | 4.43 | 94 |
| books_technical | 4.42 | 252 |
| food_drink | 4.38 | 219 |
| luggage_accessories | 4.34 | 1,014 |

**Bottom 5 categories:**

| Category | Avg Review Score | Total Reviews |
|---|---|---|
| fixed_telephony | 3.89 | 212 |
| construction_tools_safety | 3.87 | 159 |
| audio | 3.85 | 341 |
| fashion_male_clothing | 3.72 | 109 |
| office_furniture | 3.63 | 1,241 |

**Finding:** Product category has a real but much smaller effect on review scores than delivery delay. Among reliable categories, scores cluster between 3.63 and 4.47 — a far narrower spread than delivery delay's 1.69–4.30 swing. Office furniture and men's fashion stand out as consistent underperformers even with solid sample sizes.

### 3. Seller Performance vs. Review Score

Olist has thousands of individual sellers, most with very few reviews. Before ranking sellers, we checked the distribution of review counts to decide a reliability cutoff:

| Reviews per Seller | Number of Sellers |
|---|---|
| 1 review | 572 |
| 2–5 reviews | 871 |
| 6–10 reviews | 451 |
| 11–20 reviews | 401 |
| 21–50 reviews | 379 |
| 51–100 reviews | 207 |
| 101–200 reviews | 123 |
| 201–300 reviews | 30 |
| 301–400 reviews | 25 |
| 401–500 reviews | 5 |
| 501–1000 reviews | 16 |
| 1001–1500 reviews | 7 |
| 1501–2000 reviews | 3 |

About 61% of sellers have 10 or fewer total reviews. Given this long tail, sellers were required to have more than 20 reviews to be included in the final ranking — leaving roughly 795 sellers — and restricted to single-seller orders only.

**Finding:** Among reliable sellers, average review scores range from 2.19 to 5.0 — a spread nearly as wide as delivery delay's, and wider than product category's. Only 11 sellers average below 3.0, and 55 average below 3.5, so genuine underperformers are the exception rather than the rule. Seller IDs are long, non-descriptive strings rather than readable names, so individual sellers aren't listed here — the full ranked breakdown is available on the dashboard.

**Limitation:** a seller who ships late themselves would also show up as low-rated in this analysis, so seller performance and delivery delay likely overlap to some degree rather than being fully independent drivers. Worth flagging rather than treating seller quality as a wholly separate root cause.

---

## Overall Conclusion

Delivery performance is the dominant driver of customer satisfaction on Olist's marketplace, with seller performance a close second — reliable sellers span nearly the same range as delivery delay does. Product category plays a real but smaller role, with office furniture and men's fashion the clearest underperformers. Because a seller's rating may partly reflect their own shipping reliability, seller and delivery effects likely overlap rather than being fully independent — even so, seller vetting is a lever Olist can act on directly, since it doesn't control every leg of the delivery journey.

---

## Dashboard

**[View Interactive Dashboard on Tableau Public](#)** — *add your published Tableau Public link here*

The dashboard brings together six views: delivery delay buckets, the daily delay trend, and top/bottom 5 charts for both categories and sellers, color-coded blue (reliable/good-performing) and red (concerning) throughout.

### Building the Dashboard

A few decisions came up specifically while building the Tableau dashboard that are worth documenting alongside the SQL work:

- **Delay Days had to be set to Discrete, not Continuous.** Tableau treated it as a numeric measure by default and summed all seven days into a single point instead of plotting them separately — converting it to a discrete field fixed this.
- **Reliability filters needed to be added as Context Filters.** Tableau calculates a Top N ranking before applying regular filters, so a plain "Total Reviews ≥ 90" filter still let small-sample categories into the Top 5 first, only removing them afterward. Marking the reviews filter as a context filter forces it to run first.
- **Top 5 / Bottom 5 instead of showing everything.** With ~74 categories and ~2,689 sellers, full lists weren't practical — Top/Bottom 5 was used instead, applied on top of the same reliability threshold used in BigQuery.
- **Seller IDs were truncated to 8 characters** via a calculated field, since the full IDs are long and meaningless to a viewer. Checked for collisions across the shortlisted sellers before using it.
- **Every review-score axis was fixed to a 0–5 range** across all charts, so bar lengths are directly comparable at a glance instead of each chart auto-scaling to its own data.

---

## Recommendations

Based on the analysis, we recommend the following four strategies to improve customer satisfaction:

**1. Prioritize Delivery Estimate Accuracy Over Speed**

Since satisfaction drops sharply as soon as an order passes its promised date and keeps declining with each additional day late, Olist should invest in more accurate, buffer-adjusted delivery estimates and stronger carrier reliability, rather than promising faster windows it can't consistently hit.

**2. Set Proactive Alerts for At-Risk Deliveries**

Given how steep the satisfaction drop is even at just one day late, Olist should flag orders trending toward missing their estimate in real time, so customer service can proactively notify customers or expedite shipping before dissatisfaction sets in.

**3. Vet and Monitor Seller Performance**

Seller-level variation is nearly as wide as delivery delay's impact, making seller vetting and ongoing performance monitoring a lever Olist can act on directly as the marketplace operator — separate from its own logistics investments.

**4. Investigate Underperforming Categories**

Office furniture and men's fashion show below-average satisfaction with solid sample sizes, independent of delivery delay. These categories warrant a focused root-cause review — packaging damage, sizing accuracy, and seller quality are reasonable starting points.

---

## Project Files

- `queries/` — All 10 BigQuery SQL queries used in this project, in the order they were run
- `data/` — Summary CSV exports used to build the Tableau dashboard

---

## Queries

| Query | Description |
|---|---|
| `01_orders_cleaning.sql` | Filters Orders to delivered status with a valid delivery date |
| `02_reviews_cleaning.sql` | Two-pass ROW_NUMBER() dedup to guarantee one review_score per order |
| `03_delivery_delay_buckets.sql` | Delay bucket vs. avg review_score |
| `04_delivery_delay_daily.sql` | Avg review_score by exact day of delay (1–7) |
| `05_products_cleaning.sql` | Null handling, English translation, manual category overrides |
| `06_order_items_checks.sql` | Null, duplicate, and orphan-key checks |
| `07_multi_category_check.sql` | % of orders spanning multiple categories |
| `08_category_analysis.sql` | Avg review_score per category, single-category orders only |
| `09_multi_seller_check.sql` | % of orders spanning multiple sellers |
| `10_seller_performance.sql` | Avg review_score per seller, single-seller orders, 20+ reviews |

---

*This project was completed as part of the Google Data Analytics Professional Certificate capstone. See also: [Cyclistic Bike-Share Analysis](https://github.com/Abdul446223/Cyclistic-Bike-Share-Analysis) (Case Study 1).*
