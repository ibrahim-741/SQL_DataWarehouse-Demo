# Architecture — Sales Data Warehouse

Interactive version: [`architecture.html`](./architecture.html) (open in a browser — click a flow tab, then click any node, or press Play).

## Overview

A SQL Server data warehouse built on the **Medallion architecture** (Bronze → Silver → Gold). Two source systems — a CRM and an ERP, each exported as flat CSV files — are loaded as-is, cleansed, and reshaped into a star schema for reporting. There is no live source connection and no historization requirement (per `README.md`): each load re-truncates and reloads the latest snapshot.

```
source_crm/*.csv ─┐                                            ┌─→ gold.dim_customers ─┐
source_erp/*.csv ─┴─→ bronze.* ─→ silver.* (cleansed) ─→ ──────┼─→ gold.dim_products  ─┼─→ BI / Analytics
                                                                └─→ gold.fact_sales    ─┘
```

## Components

| Node | Layer | What it is |
|---|---|---|
| CRM Source / ERP Source | Source | Flat CSV exports (`datasets/source_crm/*.csv`, `datasets/source_erp/*.csv`) — 3 files each |
| Bronze · CRM / Bronze · ERP | Bronze | `bronze.crm_*` / `bronze.erp_*` tables. Loaded by the `bronze.load_bronze` stored procedure: `TRUNCATE` + `BULK INSERT`, one file per table, no transformation |
| Silver · CRM / Silver · ERP | Silver | `silver.crm_*` / `silver.erp_*` tables, each with a `dwh_create_date` audit column. Loaded by `silver.load_silver`: `TRUNCATE` + `INSERT…SELECT` with cleansing rules (below) |
| gold.dim_customers | Gold | View: CRM `crm_cust_info` as base, enriched with ERP `erp_cust_az12` (birthdate, gender fallback) and `erp_loc_a101` (country) |
| gold.dim_products | Gold | View: CRM `crm_prd_info` as base (current rows only), enriched with ERP `erp_px_cat_g1v2` (category/subcategory/maintenance) |
| gold.fact_sales | Gold | View: CRM `crm_sales_details` as the grain (one row per order line), joined to `dim_customers`/`dim_products` by natural key to resolve surrogate keys |
| BI / Analytics | Consumer | Reporting/dashboard queries against the gold views directly — no materialization. Documented in `docs/data_catalog.md` |

Orchestration: `scripts/init_database.sql` creates the `DataWarehouse` database and the three schemas (destructive — drops the DB if it already exists). There's no scheduler in the repo; `bronze.load_bronze` and `silver.load_silver` are run manually, in order. Gold has no load step — the views compute live on query. `tests/quality_checks_silver.sql` and `tests/quality_checks_gold.sql` validate each layer (nulls, dupes, trims, domain values, PK uniqueness, fact→dim referential integrity).

## Flows

### 1. CRM Lineage
`source_crm → bronze_crm → silver_crm → {dim_customers, dim_products, fact_sales}`

1. **Bulk-load CRM CSVs** — `bronze.load_bronze` truncates and `BULK INSERT`s `cust_info.csv`, `prd_info.csv`, `sales_details.csv` as-is.
2. **Cleanse crm_cust_info** — trims names, maps marital status/gender codes to text, dedupes to the latest row per `cst_id` via `ROW_NUMBER() OVER(PARTITION BY cst_id ORDER BY cst_create_date DESC)`.
3. **Cleanse crm_prd_info** — derives `cat_id` from the `prd_key` prefix, strips that prefix from `prd_key`, defaults null cost to 0, maps line codes (M/R/S/T), computes `prd_end_dt` from the next row's start date via `LEAD()`.
4. **Cleanse crm_sales_details** — nulls out malformed int-encoded dates, recomputes `sales`/`price` when the source math is inconsistent.
5. **Base row for dim_customers** — `crm_cust_info` anchors the customer dimension and is the master source for gender.
6. **Base row for dim_products** — `crm_prd_info` anchors the product dimension, filtered to `prd_end_dt IS NULL`.
7. **Base grain for fact_sales** — `crm_sales_details` supplies the one-row-per-order-line grain.

### 2. ERP Lineage
`source_erp → bronze_erp → silver_erp → {dim_customers, dim_products}`

1. **Bulk-load ERP CSVs** — same truncate + bulk-insert pattern for `CUST_AZ12.csv`, `LOC_A101.csv`, `PX_CAT_G1V2.csv`.
2. **Cleanse erp_cust_az12** — strips a stray `NAS` prefix from `cid`, nulls out future birthdates, normalizes gender text.
3. **Cleanse erp_loc_a101** — strips dashes from `cid` to match the CRM key format, maps country codes (`DE`→Germany, `US`/`USA`→United States, blank/null→`n/a`).
4. **Cleanse erp_px_cat_g1v2** — straight pass-through, already clean.
5. **Enrich dim_customers** — `LEFT JOIN` on `cst_key = cid` adds birthdate and country; gender fills in only where CRM's is `n/a`.
6. **Enrich dim_products** — `LEFT JOIN` on `cat_id = id` adds category, subcategory, maintenance flag.

### 3. Gold Star Schema & BI
`silver → gold views → BI`

1. **fact_sales grain** — one row per order line, straight from `silver.crm_sales_details`.
2. **dim_customers resolved** — CRM base + ERP enrichment collapse into one `customer_key`-surrogate row per customer.
3. **dim_products resolved** — CRM base + ERP enrichment collapse into one `product_key`-surrogate row per current product.
4. **Resolve customer_key** — `fact_sales LEFT JOIN dim_customers ON sls_cust_id = customer_id`.
5. **Resolve product_key** — `fact_sales LEFT JOIN dim_products ON sls_prd_key = product_number` — a view joining two other views.
6. **BI queries the star schema** — analysts query the gold views directly for customer behavior, product performance, and sales trend reporting; `tests/quality_checks_gold.sql` guards PK uniqueness and fact→dim referential integrity.

## Notes on the diagram

- No dev/prod or online/offline mode toggle — this is a single static pipeline, so the mode picker was omitted rather than left as a non-functional decoration.
- Node colors: orange = source files, violet = bronze (raw), amber = silver (cleansed), sky = gold dimensions, magenta = gold fact, mint = BI/analytics.
- Click any node to jump to the first step in the current flow that touches it; drag nodes to reposition (layout persists per-browser via `localStorage`).
