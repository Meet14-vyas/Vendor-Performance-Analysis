# 🍷 Vendor Performance Analysis — Retail Liquor & Wine Inventory

An end-to-end data analytics project that analyzes **vendor, brand and inventory performance** for a liquor & wine retail chain. It covers the full workflow: raw CSV ingestion → SQL-based data modeling → cleaning & feature engineering → exploratory and statistical analysis → an interactive **Power BI dashboard**.

> **Goal:** Help the business decide which vendors to rely on, which brands need pricing/promotion changes, where bulk buying saves money, and where capital is stuck in unsold inventory.

---

## 📌 Table of Contents
- [Business Problem](#-business-problem)
- [Dataset](#-dataset)
- [Tech Stack](#-tech-stack)
- [Project Architecture](#-project-architecture)
- [Repository Structure](#-repository-structure)
- [Data Pipeline](#-data-pipeline)
- [Feature Engineering](#-feature-engineering)
- [Key Analysis & Findings](#-key-analysis--findings)
- [Power BI Dashboard](#-power-bi-dashboard)
- [How to Run](#-how-to-run)
- [Recommendations](#-recommendations)
- [Future Improvements](#-future-improvements)

---

## 🎯 Business Problem

Retail distributors work with hundreds of vendors and thousands of brands. Without a consolidated view, it is hard to answer:

1. Which **vendors and brands** drive the most sales?
2. Which vendors contribute most to **total procurement** — and how dependent is the business on them?
3. Does **bulk purchasing** actually reduce unit cost?
4. Which vendors have **slow-moving stock** and how much **capital is locked** in unsold inventory?
5. Which brands have **low sales but high margins** (candidates for promotion or pricing changes)?
6. Do **top-performing** and **low-performing** vendors have statistically different profit margins?

---

## 📂 Dataset

Source: [Vendor Sales Analysis Datasets on Kaggle](https://www.kaggle.com/datasets/meetvyas11/vendor-sales-analysis-datasets)

The raw data (large, so not stored in this repo) consists of 6 CSV tables covering the year 2024:

| Table | Records | Description |
|---|---|---|
| `purchases` | ~2.37 M | Purchase orders: vendor, brand, PO/receiving/invoice/pay dates, price, quantity, dollars |
| `sales` | ~12.83 M | Store-level sales transactions: quantity, sales dollars, price, excise tax, vendor |
| `purchase_prices` | ~12.3 K | Brand-level actual retail price, volume, purchase price and vendor |
| `vendor_invoice` | ~5.5 K | Vendor invoices incl. freight cost per PO |
| `begin_inventory` | ~206 K | Opening stock on hand per store/brand |
| `end_inventory` | ~224 K | Closing stock on hand per store/brand |

> ⚠️ Download the raw files from Kaggle and place them in a `data/` folder (this folder is git-ignored).

The final analytical table, **`vendor_sales_summary.csv`** (included in this repo), contains **8,564 vendor–brand records across 119 vendors** after filtering out loss-making / zero-sales rows.

---

## 🛠 Tech Stack

| Area | Tools |
|---|---|
| Language | Python 3 |
| Data handling | Pandas, NumPy |
| Database | SQLite, SQLAlchemy |
| Querying | SQL (CTEs, joins, aggregations) |
| Visualization | Matplotlib, Seaborn, **Power BI** |
| Statistics | SciPy (t-test, confidence intervals) |
| Other | Jupyter Notebook, Python `logging` |

---

## 🏗 Project Architecture

```
Raw CSVs (Kaggle)
      │
      ▼
ingestion_db.py  ──►  SQLite (inventory.db)  ──  6 raw tables
      │
      ▼
get_vendor_summary.py
  • SQL CTEs: Freight + Purchase + Sales summaries
  • Cleaning & feature engineering
      │
      ▼
vendor_sales_summary (table / CSV)
      │
      ├──►  Jupyter Notebooks (EDA, statistics, insights)
      └──►  Power BI Dashboard
```

---

## 📁 Repository Structure

```
├── ingestion_db.py                          # Loads raw CSVs into SQLite
├── get_vendor_summary.py                    # Builds & cleans the vendor summary table
├── Exploratory_Data_Analysis.ipynb          # Table exploration & SQL query design
├── Vendor_performance_Analysis.ipynb        # Full analysis, visuals & hypothesis testing
├── Vendor_performance_Analysis_Dashboard.pbix   # Power BI dashboard
├── vendor_sales_summary.csv                 # Final cleaned analytical dataset
├── logs/                                    # Pipeline logs (git-ignored)
├── data/                                    # Raw Kaggle CSVs (git-ignored)
└── .gitignore
```

---

## ⚙️ Data Pipeline

### 1. Ingestion — `ingestion_db.py`
- Reads every CSV from `data/` and loads it into a SQLite database (`inventory.db`) using `pandas.to_sql`.
- Logs each file and total time taken (~6.2 minutes for all 6 files, with `sales.csv` being the heaviest).

### 2. Vendor Summary — `get_vendor_summary.py`
Merges multiple tables using three CTEs:

- **FreightSummary** – total freight cost per vendor from `vendor_invoice`
- **PurchaseSummary** – purchase quantity & dollars per vendor/brand (joined with `purchase_prices`, excluding zero-price rows)
- **SalesSummary** – sales quantity, dollars, price and excise tax per vendor/brand

The result is cleaned (type casting, null handling, whitespace stripping) and stored back to SQLite as `vendor_sales_summary`.

---

## 🧮 Feature Engineering

| Feature | Formula | Meaning |
|---|---|---|
| `GrossProfit` | `TotalSalesDollars − TotalPurchaseDollars` | Profit before other costs |
| `ProfitMargin` | `GrossProfit / TotalSalesDollars × 100` | Profitability in % |
| `StockTurnover` | `TotalSalesQuantity / TotalPurchaseQuantity` | How fast stock sells (<1 = slow moving) |
| `SalesToPurchaseRatio` | `TotalSalesDollars / TotalPurchaseDollars` | Revenue generated per $ purchased |
| `UnitPurchasePrice` | `TotalPurchaseDollars / TotalPurchaseQuantity` | Effective cost per unit |
| `OrderSize` | Quantile split (Small / Medium / Large) | Bulk-buying segment |
| `UnsoldInventoryValue` | `(PurchaseQty − SalesQty) × PurchasePrice` | Capital locked in stock |

---

## 🔍 Key Analysis & Findings

### Data Quality Handling
- Found negative gross profit, zero sales and `-inf` profit margins (products purchased but never sold).
- Filtered the analysis dataset to `GrossProfit > 0`, `ProfitMargin > 0`, `TotalSalesQuantity > 0`.

### 1. Top Vendors & Brands
- **Top vendors by sales:** Diageo North America (~$68.0M), Martignetti Companies (~$39.3M), Pernod Ricard USA (~$32.1M), Jim Beam Brands (~$31.4M), Bacardi USA (~$24.9M).
- **Top brands by sales:** Jack Daniel's No 7 Black (~$7.96M), Tito's Handmade Vodka (~$7.40M), Grey Goose (~$7.21M), Captain Morgan Spiced Rum (~$6.36M), Absolut 80 Proof (~$6.24M).

### 2. Procurement Dependency (Pareto Analysis)
- **Top 10 vendors account for ~65.7% of total purchase dollars**, with Diageo alone at ~16.3% — a notable supplier-concentration risk.

### 3. Bulk Purchasing Effect
| Order Size | Avg. Unit Purchase Price |
|---|---|
| Small | $39.07 |
| Medium | $15.49 |
| Large | $10.78 |

Large orders have roughly a **72% lower unit cost** than small orders.

### 4. Inventory Efficiency
- Identified vendors with low stock turnover (<1), e.g., Alisa Carr Beverages, Highland Wine Merchants, Park Street Imports.
- **~$2.71M of capital is locked in unsold inventory**, led by Diageo (~$722K), Jim Beam (~$555K), Pernod Ricard (~$471K) and William Grant & Sons (~$402K).

### 5. Promotion / Pricing Candidates
- Isolated **198 brands** with low sales (bottom 15%) but high margins (top 15%) — good targets for promotions or price adjustments.

### 6. Statistical Testing
| Group | Mean Profit Margin | 95% CI |
|---|---|---|
| Top-performing vendors | 31.18% | 30.74 – 31.61 |
| Low-performing vendors | 41.57% | 40.50 – 42.64 |

A Welch two-sample t-test gave **t = −17.67, p < 0.001**, so the null hypothesis is rejected: low-selling vendors earn significantly **higher margins**, while top sellers rely on volume.

### 7. Correlation Insights
- Purchase quantity vs. sales quantity: **0.999** (very strong).
- Profit margin vs. sales price: **−0.179** (weak negative).
- Stock turnover has a weak negative correlation with margin, so faster turnover does not automatically mean higher profit.

---

## 📊 Power BI Dashboard

File: `Vendor_performance_Analysis_Dashboard.pbix`

The dashboard presents KPI cards plus the following visuals:
- 🍩 **Purchase contribution** by vendor (donut chart)
- 📊 **Top vendors** and **top brands** by sales (bar charts)
- 🔻 **Low-performing** vendors/brands (funnel chart)
- 🔵 **Low-performing** brands analysis (scatter chart)

> 💡 *Tip: add a dashboard screenshot at `images/dashboard.png` and uncomment the line below.*

<!-- ![Dashboard](images/dashboard.png) -->

---

## 🚀 How to Run

```bash
# 1. Clone the repository
git clone https://github.com/<your-username>/<your-repo-name>.git
cd <your-repo-name>

# 2. Install dependencies
pip install pandas numpy matplotlib seaborn scipy sqlalchemy jupyter

# 3. Create required folders
mkdir data logs

# 4. Download the dataset from Kaggle and place all 6 CSVs inside data/
#    https://www.kaggle.com/datasets/meetvyas11/vendor-sales-analysis-datasets

# 5. Load raw data into SQLite
python ingestion_db.py

# 6. Build the vendor summary table
python get_vendor_summary.py

# 7. Open the notebooks
jupyter notebook
```

**Notes**
- `inventory.db` does **not** need to be created manually. It is generated automatically in the project folder when you run `python ingestion_db.py`, and `get_vendor_summary.py` then adds the `vendor_sales_summary` table to the same file.
- The `data/` and `logs/` folders are **not** created by the scripts, so create them first (step 3). Without `logs/`, the scripts will fail at startup.
- Always run `ingestion_db.py` **before** `get_vendor_summary.py`, and run both from the project root folder.
- The first run takes roughly 6–7 minutes because `sales.csv` has ~12.8M rows. `inventory.db` is git-ignored because of its large size.

To skip the heavy ingestion step, you can use the included `vendor_sales_summary.csv` directly for analysis or the Power BI dashboard.

---

## 💡 Recommendations

1. **Reduce supplier concentration risk** — the top 10 vendors drive ~66% of procurement; negotiate better terms and develop alternates.
2. **Leverage bulk discounts** — large orders cut unit cost sharply, but balance against holding cost for slow movers.
3. **Clear slow-moving stock** — run discounts/bundles for vendors with turnover < 1 to release the ~$2.7M locked capital.
4. **Promote high-margin, low-sales brands** — the 198 identified brands can add profit with modest marketing.
5. **Review pricing on top sellers** — their margins are lower; selective price optimization could add profit at scale.

---

## 🔮 Future Improvements

- Time-series / seasonality analysis and demand forecasting
- Store- and city-level performance breakdown
- Include inventory (begin/end) tables to compute precise carrying cost
- Automate the pipeline (scheduling, tests, Docker)
- Vendor scoring model combining margin, turnover, freight and lead time

---

## 👤 Author

**Meet Vyas**

⭐ If you found this project useful, consider giving it a star!
