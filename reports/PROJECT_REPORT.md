# Project Report: Customer Shopping Behavior Analysis

**Author:** Subhranshu Panda
**Date:** July 2026
**Tools:** Python (pandas, SQLAlchemy), PostgreSQL, pgAdmin4, Power BI

## 1. Business Problem

A retail business wants to understand its customers better: who is generating the most revenue, whether discounts and subscriptions are actually driving spend, which products perform best, and how loyal the customer base is. This project answers those questions using a transaction-level dataset of 3,900 customers, moving from raw data to a queryable database to a stakeholder-facing dashboard.

## 2. Data

- **Source file:** `data/customer_shopping_behavior.csv`
- **Size:** 3,900 rows, 18 raw columns
- **Grain:** one row per customer transaction record
- **Raw fields:** Customer ID, Age, Gender, Item Purchased, Category, Purchase Amount (USD), Location, Size, Color, Season, Review Rating, Subscription Status, Shipping Type, Discount Applied, Promo Code Used, Previous Purchases, Payment Method, Frequency of Purchases

## 3. Data Cleaning & Feature Engineering (Python)

Performed in `notebooks/Customer_Shopping_Behaviour_Analysis.ipynb`:

1. **Missing values:** `Review Rating` had 37 nulls (out of 3,900). Imputed using the **median rating within each product category** rather than a global median, to avoid biasing categories with systematically higher/lower ratings.
2. **Column standardization:** lowercased all column names and converted to snake_case (e.g. `Purchase Amount (USD)` → `purchase_amount`) for cleaner SQL querying.
3. **Redundant column removal:** confirmed `discount_applied` and `promo_code_used` were identical for all 3,900 rows and dropped `promo_code_used`.
4. **Feature engineering:**
   - `age_group`: quartile-based bucketing of `age` into Young Adult / Adult / Middle-aged / Senior using `pd.qcut`.
   - `purchase_frequency_days`: mapped categorical purchase-frequency labels (e.g. "Weekly", "Fortnightly", "Annually") to a numeric day count, enabling future quantitative frequency analysis.
5. **Load to database:** the cleaned dataframe was loaded into a PostgreSQL table (`customer`) inside the `customer_behaviour` database via SQLAlchemy, ready for SQL analysis.

## 4. SQL Analysis — Full Results

Queries executed in pgAdmin4 against the `customer` table. Full query text in `sql/customer_behavior_sql_queries.sql`; raw output screenshots in `assets/sql_screenshots/`.

### Q1. Total revenue by gender

| Gender | Revenue |
|--------|--------:|
| Male | $157,890 |
| Female | $75,191 |

Male customers account for roughly two-thirds of total revenue, consistent with a male-skewed customer base (2,652 of 3,900 customers are male).

### Q2. Customers who used a discount but still spent above the average

839 customers used a discount **and** spent more than the overall average purchase amount ($59.76) — meaning discounting isn't just attracting price-sensitive, low-spend customers; a meaningful share of discount users are already high-value.

### Q3. Top 5 products by average review rating

| Rank | Product | Avg. Rating |
|------|---------|------:|
| 1 | Gloves | 3.86 |
| 2 | Sandals | 3.84 |
| 3 | Boots | 3.82 |
| 4 | Hat | 3.80 |
| 5 | Skirt | 3.78 |

### Q4. Average purchase amount: Standard vs. Express shipping

| Shipping Type | Avg. Purchase Amount |
|----------------|------:|
| Express | $60.48 |
| Standard | $58.46 |

Only a ~$2 gap — shipping preference is a weak predictor of basket size.

### Q5. Subscribers vs. non-subscribers

| Subscription Status | Customers | Avg. Spend | Total Revenue |
|---|--:|--:|--:|
| Yes | 1,053 | $59.49 | $62,645 |
| No | 2,847 | $59.87 | $170,436 |

Average spend is essentially identical between subscribers and non-subscribers (~$0.38 difference). The much larger total revenue from non-subscribers is purely a volume effect (73% of the base). **Conclusion: the subscription program is not currently associated with higher per-customer spend** — it may be functioning more as a retention/engagement mechanism than a revenue lever, worth validating with a controlled test before investing further in subscription incentives.

### Q6. Top 5 products by percentage of purchases with a discount applied

| Rank | Product | Discount Rate |
|------|---------|------:|
| 1 | Hat | 50% |
| 2 | Sneakers | 49% |
| 3 | Coat | 49% |
| 4 | Sweater | 48% |
| 5 | Pants | 47% |

### Q7. Customer segmentation by purchase history

Segments: New (1 previous purchase), Returning (2–10), Loyal (11+).

| Segment | Customers |
|---------|--:|
| Loyal | 3,116 |
| Returning | 701 |
| New | 83 |

80% of the customer base is already "Loyal" by this definition. This dataset represents a mature, retained customer population rather than a new-acquisition funnel — any growth strategy should weigh retention economics heavily, since new-customer volume is a small slice of the current base.

### Q8. Top 3 best-selling products per category (by order count)

| Category | #1 | #2 | #3 |
|----------|----|----|----|
| Accessories | Jewelry (171) | Sunglasses (161) | Belt (161) |
| Clothing | Blouse (171) | Pants (171) | Shirt (169) |
| Footwear | Sandals (160) | Shoes (150) | Sneakers (145) |
| Outerwear | Jacket (163) | Coat (161) | — |

*(Outerwear has only 2 products in the dataset, so only 2 ranks are populated.)*

### Q9. Are repeat buyers (5+ previous purchases) more likely to be subscribers?

| Subscription Status | Repeat Buyers (5+ purchases) |
|---|--:|
| No | 2,518 |
| Yes | 958 |

72% of repeat buyers are **not** subscribed — repeat purchasing behavior in this dataset is largely independent of the subscription program, reinforcing the Q5 finding.

### Q10. Revenue contribution by age group

| Age Group | Total Revenue |
|-----------|--:|
| Young Adult | $62,143 |
| Middle-aged | $59,197 |
| Adult | $55,978 |
| Senior | $55,763 |

Revenue is broadly evenly distributed across age groups (a ~$6.4K spread across the four quartile-based groups), with no single age segment dominating.

## 5. Dashboard (Power BI)

`dashboard/customer_behavior_dashboard.pbix` — an interactive dashboard built on the same cleaned dataset, with slicers for subscription status, gender, category, and shipping type. Headline KPIs: 3.9K customers, $59.76 average purchase amount, 3.75 average review rating, 27% subscription rate. Visuals break down revenue and sales volume by product category and by age group, letting a stakeholder filter live rather than read a static report.

![Dashboard Preview](../assets/dashboard_preview.png)

## 6. Recommendations

1. **Re-evaluate the subscription program's ROI.** Since subscribers don't out-spend non-subscribers on average (Q5, Q9), consider testing whether subscription perks should shift toward higher-margin incentives (e.g. early access, exclusive items) rather than discounts, or investigate whether subscription is being marketed to the wrong segment.
2. **Double down on retention over acquisition.** With 80% of customers already "Loyal" (Q7), retention-focused campaigns (loyalty tiers, personalized offers) likely have a larger addressable audience than acquisition campaigns.
3. **Audit discount strategy by product.** A handful of products (Hats, Sneakers, Coats) carry disproportionately high discount rates (Q6) — worth checking whether this is intentional clearance activity or margin leakage.
4. **Investigate the gender revenue gap.** Male customers drive ~68% of revenue (Q1); worth understanding whether this reflects the marketing funnel, product mix, or an underserved female customer segment with growth potential.
5. **Shipping tier is not a strong lever for basket size** (Q4) — pricing/promotion experiments are more likely to move average order value than shipping options.

## 7. Repository

Source code, SQL, notebook, dashboard file, and this report are available in this repository. See the [README](../README.md) for setup instructions and repository structure.
