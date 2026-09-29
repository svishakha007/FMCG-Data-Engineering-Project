# 🏗️ FMCG Sales Data Platform: Databricks + AWS S3

An end-to-end data engineering project that takes raw CSV files from **AWS S3**, processes them through a **Bronze → Silver → Gold** medallion architecture in **Databricks (Unity Catalog)**, loads the fact table **incrementally**, and serves a live **AI/BI sales dashboard**.

![Dashboard](images/dashboard.png)

---

## 📌 Highlights

| Metric | Value |
|---|---|
| Total Revenue | ₹120B |
| Total Quantity Sold | 38M units |
| Order Count | 101.17K |
| Avg Order Value | ₹1M |

- Medallion architecture (Bronze / Silver / Gold) governed by Unity Catalog
- Star schema in the Gold layer for analytics
- Full load + **incremental load** pattern for the fact table
- Multi-task Databricks Job orchestrating the whole flow
- Interactive AI/BI dashboard on top of the Gold layer

---

## 🧱 Architecture

```
                ┌────────────────────┐
                │      AWS S3        │
                │   bucket: sports-sb│
                │ customers/         │
                │ products/          │
                │ gross_price/       │
                │ orders/            │
                └─────────┬──────────┘
                          │  CSV ingestion
                          ▼
   ┌───────────┐    ┌───────────┐    ┌───────────┐
   │  BRONZE   │ →  │  SILVER   │ →  │   GOLD    │
   │ raw data  │    │ cleaned & │    │ star      │
   │           │    │ staged    │    │ schema    │
   └───────────┘    └───────────┘    └─────┬─────┘
                                           │
                                           ▼
                                  ┌──────────────────┐
                                  │ AI/BI Dashboard  │
                                  └──────────────────┘
```

**Unity Catalog:** `fmcg` catalog with `bronze`, `silver` and `gold` schemas.

---

## 📂 Data Sources (S3)

Bucket: `sports-sb`

| Folder | Description |
|---|---|
| `customers/` | Customer master data |
| `products/` | Product master data (sports goods: arm guards, cricket pads, badminton rackets, etc.) |
| `gross_price/` | Gross pricing data |
| `orders/` | Order transactions (`orders/landing/` is the landing zone for new files) |

---

## ⭐ Gold Layer: Star Schema

| Table | Type | Description |
|---|---|---|
| `fact_orders` | Fact | Order-level transactions |
| `dim_customer` | Dimension | Customer attributes |
| `dim_product` | Dimension | Product and category attributes |
| `dim_gross_price` | Dimension | Pricing information |
| `dim_date` | Dimension | Calendar dimension |

---

## ⚙️ Pipeline & Orchestration

A Databricks Job (`Pipeline`) runs four dependent tasks on **serverless compute**:

```
dim_processing_customers → dim_processing_products → dim_processing_prices → fact_processing_orders
```

| Task | Purpose |
|---|---|
| `dim_processing_customers` | Processes customer data into the customer dimension |
| `dim_processing_products` | Processes product data into the product dimension |
| `dim_processing_prices` | Processes pricing data into the price dimension |
| `fact_processing_orders` | Incremental load of the orders fact table |

> **Trigger:** The job can be set to start on file arrival at `s3://sports-sb/orders/landing/`.
> <!-- Confirm this trigger is applied in your workspace before publishing -->

---

## 🔄 Incremental Load Logic

Rather than reloading the entire fact table, the incremental notebook:

1. Reads new rows from the Silver staging table (`{catalog}.{silver_schema}.staging_{data_source}`)
2. Truncates `order_placement_date` to the month (`start_month`)
3. Extracts the **distinct months** that contain new data
4. Registers them as a temp view (`incremental_months`) so only those months are processed and merged into the Gold fact table

Notebooks are parameterised with widgets (`catalog`, `data_source`) so the same logic can be reused across sources.

---

## 📓 Notebooks

| Notebook | Description |
|---|---|
| `dim_date_table_creation` | Builds the date dimension |
| `1_customer_data_processing` | Customer pipeline |
| `2_products_data_processing` | Product pipeline |
| `3_pricing_data_processing` | Pricing pipeline |
| `1_full_load_fact` | Initial full load of `fact_orders` |
| `2_incremental_load_fact` | Ongoing incremental load of `fact_orders` |

---

## 📊 Dashboard

The **FMCG Sales Dashboard** (Databricks AI/BI) includes:

- **KPIs:** Total Revenue, Total Quantity, Order Count, Avg Order Value
- **Revenue Trend:** monthly revenue over time
- **Revenue by Market**
- **Revenue by Category**
- **Revenue by Channel:** Retailer, Direct, Acquisition
- **Product Details:** product-level revenue and quantity table

---

## 🛠️ Tech Stack

- **Platform:** Databricks (Serverless, Unity Catalog, Jobs, AI/BI Dashboards)
- **Storage:** AWS S3
- **Processing:** PySpark, Spark SQL
- **Table format:** Delta Lake
- **Architecture:** Medallion (Bronze / Silver / Gold), Star Schema

---

## 🚀 Getting Started

1. **Create the S3 bucket** and upload the source CSVs into `customers/`, `products/`, `gross_price/` and `orders/`.
2. **Connect Databricks to S3** (external location / storage credential in Unity Catalog).
3. **Create the catalog and schemas:**
   ```sql
   CREATE CATALOG IF NOT EXISTS fmcg;
   CREATE SCHEMA IF NOT EXISTS fmcg.bronze;
   CREATE SCHEMA IF NOT EXISTS fmcg.silver;
   CREATE SCHEMA IF NOT EXISTS fmcg.gold;
   ```
4. **Import the notebooks** from `/notebooks` into your Databricks workspace.
5. **Run in this order:** `dim_date_table_creation` → dimension notebooks → `1_full_load_fact`.
6. **Create the Job** with the four tasks above and (optionally) a file-arrival trigger.
7. **Use `2_incremental_load_fact`** for subsequent loads when new order files arrive.
8. **Build the dashboard** on the Gold tables.

---

## 📁 Repository Structure

```
.
├── notebooks/
│   ├── dim_date_table_creation
│   ├── 1_customer_data_processing
│   ├── 2_products_data_processing
│   ├── 3_pricing_data_processing
│   ├── 1_full_load_fact
│   └── 2_incremental_load_fact
├── images/
│   ├── dashboard.png
│   ├── pipeline.png
│   ├── catalog.png
│   └── s3_bucket.png
└── README.md
```

---

## 🔍 Known Limitations & Next Steps

- `market` and `channel` contain `null` values in some records; add null handling / data quality checks in the Silver layer
- Add automated data quality tests (e.g. expectations or constraints)
- Add alerting for job failures
- Parameterise the job for multiple data sources
- Add CI/CD (e.g. Databricks Asset Bundles)

---

## 👤 Author

**Your Name**
[LinkedIn](https://linkedin.com/in/your-profile) · [GitHub](https://github.com/your-username)

⭐ If you found this project useful, consider giving it a star!
