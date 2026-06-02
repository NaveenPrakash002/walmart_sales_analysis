<div align="center">

# Walmart Sales Data Analysis

**A complete data pipeline from raw transaction records to business insights — built with Python and MySQL.**

![Python](https://img.shields.io/badge/Python-3.8+-3776AB?style=flat-square&logo=python&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-8.0-4479A1?style=flat-square&logo=mysql&logoColor=white)
![Pandas](https://img.shields.io/badge/Pandas-2.0-150458?style=flat-square&logo=pandas&logoColor=white)
![SQLAlchemy](https://img.shields.io/badge/SQLAlchemy-2.0-D71F00?style=flat-square&logo=sqlalchemy&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-22c55e?style=flat-square)

</div>

<br/>

## What This Project Does

I built this to practice working with messy real-world data end to end — not just writing queries, but handling everything from a raw CSV with formatting issues all the way through to answering real business questions in SQL.

The dataset is a collection of Walmart sales transactions across multiple branches. The raw file had dollar signs in numeric columns, duplicates, and missing values — all things you'd normally encounter before you can do anything useful with data. Once cleaned, I loaded it into MySQL and wrote nine queries targeting different aspects of branch performance, customer behaviour, and revenue trends.

---

## Project Structure

```
walmart-sales-analysis/
│
├── Walmart.csv                  # Raw source data
├── walmart_clean_data.csv       # Cleaned output (auto-generated)
├── data_cleaning.py             # Python ETL script
├── analysis_queries.sql         # SQL business queries
├── .env.example                 # Credential template
├── .gitignore
└── README.md
```

---

## Pipeline Overview

```
Walmart.csv  ──►  data_cleaning.py  ──►  walmart_clean_data.csv
                        │
                        ▼
                    MySQL (walmart_db)
                        │
                        ▼
               analysis_queries.sql
                        │
                        ▼
               Business Insights
```

The Python script handles all the cleaning and loads the result directly into MySQL. The SQL file runs on top of that loaded table.

---

## Getting Started

### Prerequisites

- Python 3.8+
- MySQL 8.0 running locally
- A `walmart_db` database already created

### Installation

```bash
git clone https://github.com/your-username/walmart-sales-analysis.git
cd walmart-sales-analysis
pip install pandas pymysql sqlalchemy python-dotenv
```

### Environment Setup

Don't hardcode your database credentials. Copy the example file and fill it in:

```bash
cp .env.example .env
```

`.env.example`:
```
DB_USER=root
DB_PASSWORD=your_password_here
DB_HOST=localhost
DB_PORT=3306
DB_NAME=walmart_db
```

Then in `data_cleaning.py`, load them like this:

```python
import os
from dotenv import load_dotenv
load_dotenv()

engine_mysql = create_engine(
    f"mysql+pymysql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}"
    f"@{os.getenv('DB_HOST')}:{os.getenv('DB_PORT')}/{os.getenv('DB_NAME')}"
)
```

### Running the Pipeline

```bash
python data_cleaning.py
```

Then open `analysis_queries.sql` in MySQL Workbench, DBeaver, or any client you prefer and run the queries against `walmart_db`.

---

## Data Cleaning

The raw CSV had a few issues that needed fixing before anything else could happen:

| Problem | Fix |
|---------|-----|
| Duplicate rows | `drop_duplicates()` |
| Missing values | `dropna()` |
| `unit_price` stored as string with `$` prefix | Strip `$`, cast to `float` |
| No revenue column | Derived `total = unit_price × quantity` |

After cleaning, the data was exported to `walmart_clean_data.csv` and pushed to MySQL using `.to_sql()`.

---

## Business Questions Answered

### 1. How are customers paying?
Breakdown of transaction count and total units sold per payment method (Cash, Credit Card, E-wallet).

### 2. Which product category performs best at each branch?
Uses `RANK()` over average customer rating, partitioned by branch, to find the top-rated category at each location.

### 3. What is the busiest day of the week per branch?
Extracts the day name from each transaction date using `DAYNAME(STR_TO_DATE(...))` and ranks by volume.

### 4. Which payment method moves the most product?
Total quantity sold grouped by payment method — a pure volume cut separate from transaction count.

### 5. How do customer ratings vary across cities and categories?
Average, minimum, and maximum ratings for each city–category pair. Useful for spotting underperformers regionally.

### 6. Which categories generate the most profit?
Calculates `SUM(unit_price × quantity × profit_margin)` per category and sorts descending.

### 7. What is the preferred payment method at each branch?
A CTE ranks payment methods by frequency per branch and returns the top one for each.

### 8. When is each branch busiest during the day?
Buckets transactions into Morning, Afternoon, and Evening shifts using `HOUR(TIME(time))` with a `CASE` statement.

### 9. Which branches declined the most in revenue from 2022 to 2023?
Two CTEs compute yearly revenue per branch, then a join calculates the percentage decrease. Returns the five worst-performing branches year-over-year.

---

## Featured Query

The year-over-year revenue decline query was the most interesting to write. It uses two CTEs to isolate each year's revenue, then joins them to calculate how much each branch dropped:

```sql
WITH revenue_2022 AS (
    SELECT branch, SUM(total) AS revenue
    FROM walmart
    WHERE YEAR(STR_TO_DATE(date, '%d/%m/%Y')) = 2022
    GROUP BY branch
),
revenue_2023 AS (
    SELECT branch, SUM(total) AS revenue
    FROM walmart
    WHERE YEAR(STR_TO_DATE(date, '%d/%m/%Y')) = 2023
    GROUP BY branch
)
SELECT
    r2022.branch,
    r2022.revenue AS last_year_revenue,
    r2023.revenue AS current_year_revenue,
    ROUND(((r2022.revenue - r2023.revenue) / r2022.revenue) * 100, 2) AS revenue_decrease_ratio
FROM revenue_2022 AS r2022
JOIN revenue_2023 AS r2023 ON r2022.branch = r2023.branch
WHERE r2022.revenue > r2023.revenue
ORDER BY revenue_decrease_ratio DESC
LIMIT 5;
```

---

## SQL Concepts Covered

Working through these queries involved a range of SQL techniques beyond basic aggregation:

- **Window functions** — `RANK() OVER(PARTITION BY ...)` for per-group rankings without collapsing rows
- **CTEs** — `WITH` clauses to break complex logic into readable steps
- **Date and time functions** — `STR_TO_DATE`, `YEAR`, `DAYNAME`, `HOUR`, `TIME`
- **Conditional logic** — `CASE WHEN` for bucketing transactions by time of day
- **Self-joins via CTEs** — comparing the same table across two time periods

---

## Dataset Schema

| Column | Type | Notes |
|--------|------|-------|
| `invoice_id` | VARCHAR | Unique transaction identifier |
| `branch` | VARCHAR | Store branch code |
| `city` | VARCHAR | Branch city |
| `category` | VARCHAR | Product category |
| `unit_price` | FLOAT | Cleaned from raw `$x.xx` string |
| `quantity` | INT | Units sold per transaction |
| `date` | VARCHAR | Format: `dd/mm/yy` |
| `time` | VARCHAR | Format: `HH:MM:SS` |
| `payment_method` | VARCHAR | Cash / Credit card / E-wallet |
| `rating` | FLOAT | Customer rating, 1–10 |
| `profit_margin` | FLOAT | Margin as a decimal |
| `total` | FLOAT | Derived: `unit_price × quantity` |

---

## .gitignore

```
.env
walmart_clean_data.csv
__pycache__/
*.pyc
.ipynb_checkpoints/
```

---

## License

MIT — free to use and adapt. Dataset sourced from publicly available Walmart sales data, used here for educational purposes.
