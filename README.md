# Shopping Trends: End-to-End Data Pipeline (Python + SQL)

An end-to-end project analysing a retail shopping trends dataset (3,900 customer transactions), taking raw data from cleaning and feature engineering in Python through to business-question analysis in SQL. A Tableau dashboard is the next stage of this project and will be added in a future update.

The full data cleaning and feature engineering process lives in [`notebooks/data_cleaning_and_feature_engineering.ipynb`](./notebooks/data_cleaning_and_feature_engineering.ipynb). The SQL analysis lives in [`sql/queries.sql`](./sql/queries.sql), run against a Postgres instance in Docker via pgAdmin. This README summarises the approach and findings.

## Dataset

Each row is a single retail transaction, with customer demographics (age, gender, location), the item purchased and its category, purchase amount, and behavioural fields (subscription status, discount/promo usage, previous purchases, shipping type, and purchase frequency).

## Data Cleaning and Feature Engineering (Python)

- Standardised column names to lowercase with underscores, to keep them consistent between the pandas and SQL stages of the pipeline.
- Checked for missing values and confirmed the dataset was complete.
- Compared the `discount_applied` and `promo_code_used` columns and found them identical across every row, so dropped the redundant `promo_code_used` column.
- Engineered an `age_group` feature (Young Adult / Adult / Middle Aged / Senior) by binning `age` into four equal-sized quantiles.
- Engineered a `purchase_frequency_days` feature, mapping the categorical `frequency_of_purchases` field (e.g. "Fortnightly", "Monthly", "Annually") to a numeric day count for easier aggregation.
- Exported the cleaned, feature-engineered dataset for use downstream.

## SQL Analysis (PostgreSQL in Docker)

The engineered dataset was loaded directly into a PostgreSQL database (`customer_analysis`, running in Docker, see [`docker-compose.yml`](./docker-compose.yml)) using SQLAlchemy at the end of the cleaning notebook, then queried through pgAdmin using aggregations, subqueries, `CASE` based segmentation, and window functions. A few of the findings:

**Revenue by gender**

| Gender | Revenue |
|---|---|
| Male | $157,890 |
| Female | $75,191 |

**Do subscribers spend more?** — No. Subscribers and non-subscribers spend almost the same on average, but non-subscribers generate nearly 3x the total revenue simply by outnumbering subscribers roughly 3 to 1.

| Subscription Status | Customers | Avg. Purchase | Total Revenue |
|---|---|---|---|
| No | 2,847 | $59.87 | $170,436 |
| Yes | 1,053 | $59.49 | $62,645 |

**Customer segmentation** (New: 1 previous purchase, Returning: 2-10, Loyal: 10+), using a `CASE` expression inside a CTE:

| Segment | Customers |
|---|---|
| Loyal | 3,116 |
| Returning | 701 |
| New | 83 |

**Top 3 products per category**, using `ROW_NUMBER()` partitioned by category, e.g. Jewelry, Belts, and Sunglasses lead Accessories, while Pants, Blouses, and Shirts lead Clothing.

**Discount sensitivity**: Hats (50%), Sneakers (49%), and Coats (49%) had the highest share of purchases made with a discount applied, useful for thinking about which product lines are the most price-sensitive.

The full set of 10 queries, including revenue by age group, shipping type comparisons, and repeat-buyer subscription likelihood, is in [`sql/queries.sql`](./sql/queries.sql).

## Running this project

```bash
# Start the Postgres container (creates the customer_analysis database automatically)
docker compose up -d

# Set the same password used in docker-compose.yml, so the notebook can read it
export POSTGRES_PASSWORD=postgres

# Python environment
pip install -r requirements.txt

# Run the notebook: cleans the data, engineers features, and loads the
# result straight into the customer_analysis database in Postgres
jupyter notebook notebooks/data_cleaning_and_feature_engineering.ipynb

# Then run the queries in sql/queries.sql (via pgAdmin, or psql) against
# the shopping_trends table
```

## Next steps

- Build a Tableau dashboard on top of the engineered dataset and SQL findings.
