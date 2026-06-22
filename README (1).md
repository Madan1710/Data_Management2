# Data Management 2 — Final Project
## Sales Data Pipeline on Databricks (Medallion Architecture)

**Student:** Madan Kumar Kumar
**Matriculation No.:** 100004225
**Course:** Data Management 2 — Data Curation and Data Management
**Instructor:** Esam Sharaf — SRH Campus Hamburg
**Submission date:** 22.06.2026

---

## 1. Objective
Combine two different data sources through a **Medallion (Bronze -> Silver -> Gold)** pipeline on Databricks and report **two business objectives** in an AI/BI dashboard. The pipeline is **incremental**: when new data is added to a source, re-running it updates the dashboard.

## 2. Data source
A single public dataset is used and split into two derived sources. Deriving data from the used dataset is permitted by the project rules; no data was fabricated.

> Chen, D. (2015). *Online Retail* [Dataset]. UCI Machine Learning Repository. https://doi.org/10.24432/C5BW33
> License: CC BY 4.0.

Transactional data from a UK-based online retailer (Dec 2010 - Dec 2011), with fields: `InvoiceNo, StockCode, Description, Quantity, InvoiceDate, UnitPrice, CustomerID, Country`.

## 3. Two data sources (different types)
To satisfy the "diverse source types" requirement, the dataset is split into two sources of **distinct types**:

1. **Products - a Delta table** (`workspace.default.products`): product master data (`StockCode, Description, UnitPrice`). Represents a **database (DB)** source.
2. **Orders - JSON files** in a Unity Catalog Volume folder (`/Volumes/workspace/default/dm2_raw/orders_stream/`), ingested with **Auto Loader (Structured Streaming)**. Represents a **streaming (Stream)** source.

**Join key:** `StockCode`.

## 4. Architecture (Medallion)
`Bronze (raw) -> Silver (cleaned + joined) -> Gold (aggregated per objective) -> AI/BI Dashboard`

- **Bronze** - raw landing, untouched except an ingestion timestamp:
  - `bronze_products` - batch read from the products table.
  - `bronze_orders` - Auto Loader stream from the orders folder, using the `availableNow` trigger, so re-running ingests **only new files** (tracked via a checkpoint).
- **Silver** (`silver_orders_enriched`) - type casting, null/duplicate removal, and the **join of the two sources** on `StockCode`; computes `revenue = Quantity * UnitPrice`.
- **Gold** - one aggregated table per objective:
  - `gold_revenue_by_region`
  - `gold_top_products`

**Rationale:** Bronze preserves raw source data for traceability; Silver centralises cleaning and the cross-source join in one place; Gold pre-aggregates so each dashboard tile is a single, simple read. Silver and Gold are rebuilt with `overwrite` and Bronze ingests incrementally via streaming append — together this keeps the whole pipeline **re-runnable** end to end.

## 5. Business objectives (dashboard tiles)
1. **Revenue by region** - total revenue per country (`gold_revenue_by_region`).
2. **Top 10 products by units sold** (`gold_top_products`).

Both are built as charts in the Databricks AI/BI dashboard **"Online Retail Sales Dashboard"**, alongside two KPI counters (Total Revenue, Total Units Sold).

## 6. Incremental test (proof the pipeline reacts to new data)
The test appends new orders (existing products, 5,000 units each, region = Germany) to the **stream** source, then re-runs Bronze -> Silver -> Gold. After refreshing the dashboard, **Germany's revenue rose from ~293K to ~386K**, while all other regions stayed unchanged. This demonstrates that adding data to a source propagates through every layer to the dashboard, as required by the testing criterion. The Silver layer's de-duplication keeps the test **idempotent** across repeated runs.

## 7. How to run
1. Upload the dataset CSV (`Online Retail.csv`) to the Volume `/Volumes/workspace/default/dm2_raw/`.
2. Open the notebook (`Code`) and use **Run All**, top to bottom.
3. Open **"Online Retail Sales Dashboard"** and refresh.
4. To test incrementality: run the **Step 7** cells, then refresh the dashboard.

## 8. Tools used
- **Platform:** Databricks Free Edition (serverless), Unity Catalog, Delta Lake, Auto Loader, Structured Streaming, Databricks AI/BI Dashboards.
- **Language:** PySpark (Python).
- **AI assistance:** an AI assistant was used for guidance and PySpark code generation. The prompts used are listed in `AI_Prompts.txt`.

## 9. Submission contents
- `DM2_Project_Screenshots.pdf` - the report (proof of completion; screenshots of each stage embedded)
- `Code.ipynb`  - the pipeline notebook (with markdown documentation per step)
- `AI_Prompts.pdf` - AI prompts used during development
- `Problems_Encountered.pdf` - problems faced during the project and how they were solved
- `README.md` - this file
