# sql-datawarehouse-project
Building a modern data warehouse with SQL Server, including ETL processes, data modeling, and analytics.
# Data Warehouse and Analytics Project

Welcome to the **Data Warehouse and Analytics Project** repository! 🚀  
This project demonstrates a comprehensive data warehousing and analytics solution, from building a data warehouse to generating actionable insights. Designed as a portfolio project, it highlights industry best practices in data engineering and analytics.

---
## 🏗️ Data Architecture

The data architecture for this project follows Medallion Architecture **Bronze**, **Silver**, and **Gold** layers:

```mermaid
flowchart LR
  subgraph SRC["Sources"]
    CRM[("CRM CSVs")]
    ERP[("ERP CSVs")]
  end

  subgraph BRONZE["Bronze · Raw"]
    B_CRM["bronze.crm_*"]
    B_ERP["bronze.erp_*"]
  end

  subgraph SILVER["Silver · Cleansed"]
    S_CRM["silver.crm_*"]
    S_ERP["silver.erp_*"]
  end

  subgraph GOLD["Gold · Star Schema"]
    DC["gold.dim_customers"]
    DP["gold.dim_products"]
    FS["gold.fact_sales"]
  end

  BI[["BI / Analytics"]]

  CRM --> B_CRM --> S_CRM
  ERP --> B_ERP --> S_ERP
  S_CRM --> DC
  S_CRM --> DP
  S_CRM --> FS
  S_ERP --> DC
  S_ERP --> DP
  DC --> FS
  DP --> FS
  DC --> BI
  DP --> BI
  FS --> BI
```

**[▶ Open the live, click-through architecture diagram](https://htmlpreview.github.io/?https://github.com/ibrahim-741/SQL_DataWarehouse-Demo/blob/main/docs/architecture.html)** — click a flow tab, then click any node to see its schema and the exact T-SQL transformation rule ([source file](docs/architecture.html), [text version](docs/architecture.md)).

1. **Bronze Layer**: Stores raw data as-is from the source systems. Data is ingested from CSV Files into SQL Server Database.
2. **Silver Layer**: This layer includes data cleansing, standardization, and normalization processes to prepare data for analysis.
3. **Gold Layer**: Houses business-ready data modeled into a star schema required for reporting and analytics.

---
## 📖 Project Overview

This project involves:

1. **Data Architecture**: Designing a Modern Data Warehouse Using Medallion Architecture **Bronze**, **Silver**, and **Gold** layers.
2. **ETL Pipelines**: Extracting, transforming, and loading data from source systems into the warehouse.
3. **Data Modeling**: Developing fact and dimension tables optimized for analytical queries.
4. **Analytics & Reporting**: Creating SQL-based reports and dashboards for actionable insights.
---

## 🚀 Project Requirements

### Building the Data Warehouse (Data Engineering)

#### Objective
Develop a modern data warehouse using SQL Server to consolidate sales data, enabling analytical reporting and informed decision-making.

#### Specifications
- **Data Sources**: Import data from two source systems (ERP and CRM) provided as CSV files.
- **Data Quality**: Cleanse and resolve data quality issues prior to analysis.
- **Integration**: Combine both sources into a single, user-friendly data model designed for analytical queries.
- **Scope**: Focus on the latest dataset only; historization of data is not required.
- **Documentation**: Provide clear documentation of the data model to support both business stakeholders and analytics teams.

---

### BI: Analytics & Reporting (Data Analysis)

#### Objective
Develop SQL-based analytics to deliver detailed insights into:
- **Customer Behavior**
- **Product Performance**
- **Sales Trends**

These insights empower stakeholders with key business metrics, enabling strategic decision-making.  

---

## 🛡️ License

This project is licensed under the [MIT License](LICENSE). You are free to use, modify, and share this project with proper attribution.

## 🌟 About Me

Hi there! I'm **Mohammed Ibrahim**. I’m an IT professional with an experience as Data Engineer.
