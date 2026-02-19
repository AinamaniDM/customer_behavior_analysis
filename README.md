# 🛍️ Customer Behavior Analysis

> An end-to-end data analysis project exploring customer spending patterns, product preferences, subscription behavior, and demographic segmentation across **3,900 retail transactions**.

---

## 📋 Project Overview

This project analyzes a retail dataset to uncover actionable insights about customer behavior. The analysis was conducted in two stages: **Exploratory Data Analysis (EDA) in Python**, followed by **structured business querying in PostgreSQL**.

---

## 📦 Dataset

| Property | Detail |
|----------|--------|
| Rows | 3,900 |
| Columns | 18 |
| Source | Retail purchase transactions |

### Column Categories

**Customer Demographics**
- Customer ID, Age, Gender, Location, Subscription Status

**Purchase Information**
- Item Purchased, Category, Purchase Amount (USD), Size, Color, Season

**Shopping Behavior**
- Discount Applied, Shipping Type, Review Rating, Previous Purchases, Frequency of Purchases, Payment Method

---

## 🛠️ Tools & Technologies

| Tool | Purpose |
|------|---------|
| Python (Pandas) | Data loading, cleaning, and feature engineering |
| Jupyter Notebooks | Interactive analysis environment |
| SQLAlchemy | PostgreSQL connection and data ingestion |
| PostgreSQL | Business analysis and structured queries |
| SQL (CTEs, Window Functions) | Advanced analytical queries |

---

## 🔍 Methodology

### 1. Exploratory Data Analysis (Python)

**Initial Exploration**
```python
df.head()       # Preview first 5 rows
df.shape        # (3900, 18)
df.info()       # Data types and null counts
df.describe()   # Summary statistics
```

**Data Quality Checks**
- No duplicate records found (`df.duplicated().sum()` → 0)
- 37 missing values in `Review Rating` column only

**Handling Missing Values**
```python
# Impute with category median (not overall mean)
df['Review Rating'] = df.groupby('Category')['Review Rating'] \
                        .transform(lambda x: x.fillna(x.median()))
```

**Column Standardization**
```python
df.columns = df.columns.str.lower().str.replace(' ', '_')
df = df.rename(columns={'purchase_amount_(usd)': 'purchase_amount'})
```

**Feature Engineering**
```python
# Age segmentation
labels = ['Young Adult', 'Adult', 'Middle Aged', 'Senior']
df['age_group'] = pd.qcut(df['age'], q=4, labels=labels)

# Purchase frequency in days
frequency_mapping = {
    'Weekly': 7, 'Fortnightly': 14, 'Bi-Weekly': 14,
    'Monthly': 30, 'Quarterly': 90, 'Every 3 Months': 90, 'Annually': 365
}
df['purchase_frequency_days'] = df['frequency_of_purchases'].map(frequency_mapping)
```

**Redundant Column Removal**
After inspection, `discount_applied` and `promo_code_used` were identical across all rows. `promo_code_used` was dropped.

---

### 2. Database Connection

```python
from sqlalchemy import create_engine

engine = create_engine(f"postgresql+psycopg2://{username}:{password}@{host}:{port}/{database}")
df.to_sql("customers", engine, if_exists="replace", index=False)
```

---

### 3. SQL Analysis (PostgreSQL)

12 business questions were answered using SQL:

#### Q1. Revenue by Gender
```sql
SELECT gender, SUM(purchase_amount) AS total_revenue
FROM customers
GROUP BY gender;
```
| Gender | Total Revenue |
|--------|--------------|
| Female | $75,191 |
| Male | $157,890 |

#### Q2. High-Value Discount Customers
```sql
SELECT customer_id, purchase_amount
FROM customers
WHERE purchase_amount > (SELECT ROUND(AVG(purchase_amount), 2) FROM customers)
  AND discount_applied = 'Yes'
LIMIT 10;
```

#### Q3. Top 5 Highest-Rated Products
```sql
SELECT item_purchased, ROUND(AVG(review_rating)::numeric, 2) AS avg_review
FROM customers
GROUP BY item_purchased
ORDER BY avg_review DESC
LIMIT 5;
```
| Product | Avg Rating |
|---------|-----------|
| Gloves | 3.86 |
| Sandals | 3.84 |
| Boots | 3.82 |
| Hat | 3.80 |
| Skirt | 3.78 |

#### Q4. Average Spend by Shipping Type
```sql
SELECT shipping_type, ROUND(AVG(purchase_amount), 2) AS avg_purchase
FROM customers
WHERE shipping_type IN ('Standard', 'Express', 'Store Pickup')
GROUP BY shipping_type
ORDER BY avg_purchase DESC;
```
| Shipping Type | Avg Purchase |
|--------------|-------------|
| Express | $60.48 |
| Store Pickup | $59.89 |
| Standard | $58.46 |

#### Q5. Subscribers vs. Non-Subscribers
```sql
SELECT subscription_status, COUNT(*) AS total_customers,
       ROUND(AVG(purchase_amount), 2) AS avg_spend,
       SUM(purchase_amount) AS total_revenue
FROM customers
GROUP BY subscription_status;
```
| Status | Customers | Avg Spend | Total Revenue |
|--------|-----------|-----------|--------------|
| No | 2,847 | $59.87 | $170,436 |
| Yes | 1,053 | $59.49 | $62,645 |

#### Q6 & Q7. Discount Usage by Product
```sql
-- Products most purchased with discount
SELECT item_purchased,
       ROUND(100 * SUM(CASE WHEN discount_applied='Yes' THEN 1 ELSE 0 END) / COUNT(*), 2)
       AS pct_with_discount
FROM customers
GROUP BY item_purchased
ORDER BY pct_with_discount DESC
LIMIT 5;
```
| Product | % With Discount | % Without Discount |
|---------|----------------|-------------------|
| Hat | 50% | — |
| Sneakers | 49% | — |
| Socks | — | 67% |
| Blouse | — | 66% |

#### Q8. Customer Loyalty Segmentation
```sql
WITH customer_type AS (
  SELECT customer_id,
    CASE
      WHEN previous_purchases <= 5 THEN 'New'
      WHEN previous_purchases BETWEEN 6 AND 20 THEN 'Returning'
      ELSE 'Loyal'
    END AS customer_segment
  FROM customers)
SELECT customer_segment, COUNT(*) AS count
FROM customer_type
GROUP BY customer_segment;
```
| Segment | Count |
|---------|-------|
| New | 424 |
| Returning | 1,137 |
| Loyal | 2,339 |

#### Q9. Top 3 Products per Category
```sql
WITH product_counts AS (
  SELECT category, item_purchased,
         COUNT(customer_id) AS total_orders,
         ROW_NUMBER() OVER (PARTITION BY category ORDER BY COUNT(customer_id) DESC) AS rnk
  FROM customers
  GROUP BY category, item_purchased)
SELECT rnk, category, item_purchased, total_orders
FROM product_counts
WHERE rnk <= 3;
```

#### Q10. Repeat Buyers & Subscription Likelihood
```sql
SELECT subscription_status, COUNT(*) AS repeat_customers
FROM customers
WHERE previous_purchases > 5
GROUP BY subscription_status;
```
| Status | Repeat Customers |
|--------|----------------|
| No | 2,518 |
| Yes | 958 |

#### Q11. Revenue by Age Group
```sql
SELECT age_group, SUM(purchase_amount) AS total_revenue
FROM customers
GROUP BY age_group
ORDER BY total_revenue DESC;
```
| Age Group | Revenue |
|-----------|---------|
| Young Adult | $62,143 |
| Middle Aged | $59,197 |
| Adult | $55,978 |
| Senior | $55,763 |

#### Q12. Subscription Rate by Gender
```sql
SELECT gender, subscription_status, COUNT(*) AS total
FROM customers
GROUP BY gender, subscription_status;
```
All 1,053 subscribers are male — there are zero female subscribers in the dataset. Female customers shop and spend comparably but have not been converted by the subscription program at all.

---

## 📊 Power BI Dashboard

The cleaned dataset was visualized in Power BI to provide an interactive summary for non-technical stakeholders. All visuals respond dynamically to four slicers: **Shipping Type**, **Category**, **Subscription Status**, and **Gender**.
![The customer behavior dashboard build in power bi](.dashboard.png)


### KPI Cards

| Metric | Value |
|--------|-------|
| Average Purchase Amount | $59.76 |
| Average Review Rating | 3.75 |
| Number of Customers | 3,900 |
| Total Unique Items | 25 |
| Total Revenue | $233,081 |

### Visuals

**Items & Revenue per Category** — Clothing leads in both volume (1,737 purchases) and revenue ($104,264). Accessories generates strong revenue relative to its purchase volume, suggesting higher average transaction values.

**Revenue & Items by Age Group** — Young Adults contribute the most revenue ($62,143) and purchases (1,028), but all four age groups fall within a narrow range — confirming broad product appeal across age segments.

**Customers by Subscription Status** — The donut chart shows 2,847 non-subscribers (73%) vs. 1,053 subscribers (27%). When filtered by Gender, it immediately reveals that all 1,053 subscribers are male — zero female customers subscribe.

### Interactive Filters

| Filter | Use Case |
|--------|----------|
| Shipping Type | Compare spend across delivery preferences |
| Category | Isolate trends for a single product group |
| Subscription Status | Compare subscriber vs. non-subscriber behavior |
| Gender | Instantly surface the zero female subscriber finding |

---

### Revenue & Demographics
- Male customers contribute **68% of total revenue** ($157,890 vs. $75,191), largely due to a higher volume of male customers in the dataset
- **Young Adults** are the top revenue-generating age group ($62,143), despite potentially lower average incomes
- Revenue contribution is remarkably balanced across all four age groups (within ~$6,000 of each other)

### Subscription & Loyalty
- Only **27% of customers subscribe** (1,053 out of 3,900)
- Subscribed customers spend nearly the same per transaction as non-subscribers (~$59.50 vs. ~$59.87) — subscriptions drive retention, not higher spend
- **All 1,053 subscribers are male** — there are zero female subscribers. Female customers spend comparably but the subscription program has completely failed to convert them — a clear conversion failure, not a data issue

### Product Performance
- **Gloves, Sandals, and Boots** are the highest-rated products
- **Socks (67%), Blouses (66%), and Sandals (63%)** sell most often at full price — strong inherent demand
- **Hats (50%), Sneakers (49%), and Coats (49%)** rely heavily on discounts — possible pricing or value perception issues
- Purchase volumes are well-balanced across top products within each category — a healthy portfolio

### Shipping & Discounts
- Shipping type has minimal impact on spend (< $2 difference between Express and Standard)
- High-value customers who use discounts still spend above the average transaction amount

---

## 💡 Recommendations

### 1. Grow the Subscription Program
- **Target Loyal customers** (2,339) who haven't subscribed — they already demonstrate commitment and are the highest-value conversion targets
- **The subscription program has zero female subscribers** despite female customers being active shoppers with comparable spend. This is a conversion failure. A dedicated female-targeted subscription campaign — with different benefit framing, product curation, or communication channels — should be treated as a top priority
- **Reframe the subscription value proposition** — since average spend doesn't increase with subscriptions, the program should emphasize exclusive perks, early access, or members-only discounts

### 2. Optimize Pricing & Discounting
- **Protect full-price products** (Socks, Blouses, Sandals) from unnecessary promotions — they sell well without incentives
- **Investigate discount-heavy items** (Hats, Sneakers, Coats) — consider repositioning, bundling, or price adjustments rather than relying on discounts
- **Create a high-value loyalty tier** for customers who spend above average even with discounts applied — they show high-ceiling potential

### 3. Demographic Targeting
- **Focus Young Adult marketing** through digital-first channels (social media, influencer partnerships)
- **Develop new customer acquisition programs** — first-purchase incentives or onboarding flows to grow the 11% New customer segment

---


```

## 🚀 How to Run

1. **Clone the repository**
   ```bash
   git clone https://github.com/yourusername/customer-behavior-analysis.git
   ```

2. **Install Python dependencies**
   ```bash
   pip install pandas sqlalchemy psycopg2-binary jupyter
   ```

3. **Set up PostgreSQL** — create a database named `customer_behavior_db`

4. **Run the notebook** — open `notebooks/eda_cleaning.ipynb` and execute all cells to clean the data and load it into PostgreSQL

5. **Run SQL queries** — execute any query from the `sql/` folder in your PostgreSQL client of choice

---

## 📬 Contact

Feel free to open an issue or reach out if you have questions or suggestions about this project.

---

*Built with Python, PostgreSQL, and curiosity.*
