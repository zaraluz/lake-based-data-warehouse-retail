# 🏗️ Lake-Based Data Warehouse Architecture for Retail Analytics

![AWS](https://img.shields.io/badge/AWS-S3%20%7C%20Athena%20%7C%20Glue-orange)
![Pentaho](https://img.shields.io/badge/Pentaho-Data%20Integration-blue)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-ERP-blue)
![Power BI](https://img.shields.io/badge/Power%20BI-Analytics-yellow)
![Status](https://img.shields.io/badge/Status-Production%20Ready-brightgreen)

<img width="1733" height="783" alt="ETL" src="https://github.com/user-attachments/assets/06aa3e73-4663-477b-aca1-2402d13fbdcc" />


---

## 📚 Table of Contents

- [Project Overview](#project-overview)
- [Business Objective](#business-objective)
- [Solution Architecture](#solution-architecture)
- [Data Flow](#data-flow)
- [Job Orchestration](#job-orchestration)
- [Applied Best Practices](#applied-best-practices)
- [Requirements](#requirements)
- [Results](#results)
- [Future Improvements](#future-improvements)

---

## 📌 Project Overview

This project implements a production-oriented end-to-end ETL pipeline and a lake-based analytical architecture for a retail supermarket operation.

The solution extracts transactional data from a PostgreSQL ERP system, applies structured transformations using Pentaho, organizes data into a dimensional Star Schema model, and delivers a scalable cloud-based analytical layer using Amazon S3, AWS Glue, and Athena.

The pipeline runs automatically on a daily basis (incremental load) and does not require manual intervention.

---

## 🎯 Business Objective

Transform raw ERP transactional data into structured, analytics-ready datasets stored in a cloud-based Data Lake that support strategic retail decision-making.

The replication of the ERP database to Amazon S3 enables direct BI connectivity without overloading the transactional system, while providing a scalable and decoupled environment for building decision-oriented dashboards.

The analytical layer acts as the single source of truth for all BI dashboards, ensuring reliability, scalability, and consistent KPI calculations.

---

## 🏛️ Solution Architecture

```txt
┌──────────────────────────────────────────────────────────────────────────┐
│                         PostgreSQL (ERP VR)                              │
│  Transactional Tables: sales, inventory, losses, purchase_notes, master  │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │ (Table Input / SQL)
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                           Pentaho (Spoon)                                │
│                         TRANSFORMATIONS (KTR)                            │
│                                                                          │
│  [STG_*] Extraction + Standardization                                    │
│   - Select/rename columns, data types, dates                             │
│   - Null handling, trimming, key normalization                           │
│                                                                          │
│  [DIM_*] Dimensions (Type 1)                                             │
│   - dim_product          (product)                                       │
│   - dim_supplier         (supplier)                                      │
│   - dim_store            (store)                                         │
│   - dim_merchandising    (hierarchy/categories)                          │
│   - dim_buyer            (buyer)                                         │
│   - dim_abc_curve + dim_abc_type                                         │
│   - bridge_product_supplier (N:N)                                        │
│                                                                          │
│  [FACT_*] Fact Tables (Granularity & Calculations)                       │
│   - fact_sales                                                           │
│       • total_revenue = sale_price * quantity                            │
│       • total_cost    = cost_with_tax * quantity                         │
│       • gross_margin  = total_revenue - total_cost                       │
│       • keys: product, store, date, supplier (via bridge)                │
│   - fact_inventory                                                       │
│       • inventory_value = cost_with_tax * stock_quantity                 │
│       • snapshot by date (reference date)                                │
│   - fact_losses                                                          │
│       • loss_value = cost_with_tax * loss_quantity                       │
│       • loss_reason (dim_loss_reason)                                    │
│   - fact_purchase_items                                                  │
│       • items per purchase note (value, quantity, product, supplier)     │
│   - inventory_log (audit trail)                                          │
│                                                                          │
│  Observability                                                           │
│   - execution logs, row count validation, basic data checks              │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                         ORCHESTRATION (JOBS - KJB)                       │
│                                                                          │
│  JOB_BACKFILL (Initial Historical Load)                                  │
│   Start → DIM_* → FACT_* → Success                                       │
│                                                                          │
│  JOB_DAILY (Recurring Load)                                              │
│  Start → DIM_* (Type 1 rebuild) → FACT_* (incremental/snapshot) → Success│
│                                                                          │
│  Failure Control                                                         │
│   Fail → Retry (N attempts) → Email/Alert → Abort                        │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                           AWS S3 (Data Lake)                             │
│                                                                          │
│  /dim/                                                                   │
│   - product, supplier, store, merchandising, buyer, abc_curve, ...       │
│                                                                          │
│  /fact/                                                                  │
│   - fact_sales, inventory, losses, purchase_items, ...                   │
│   - inventory_log                                                        │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                   AWS Glue Crawler + Data Catalog                        │
│   Schema discovery and external table updates                            │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                              AWS Athena                                  │
│   SQL queries on fact and dimension tables                               │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                               Power BI                                   │
│   Dashboards: sales, margin, losses, inventory, suppliers                │
└──────────────────────────────────────────────────────────────────────────┘

```

---

## 🔄 Data Flow

The pipeline follows a layered architecture designed for reliability, scalability, and BI decoupling.

---

### 1️⃣ Source Layer — ERP (PostgreSQL)

Transactional retail data is extracted from the ERP using SQL-based steps in Pentaho (PDI).

Core domains:
- Sales  
- Inventory  
- Losses  
- Purchase Notes  
- Master Data (products, suppliers, stores, buyers)

---

### 2️⃣ Processing Layer — Pentaho (PDI)

Transformations (KTR) are organized into three logical layers:

#### 🔹 STG (Standardization)
- Data cleaning and normalization  
- Type enforcement and null handling  
- Key validation  
- Controlled preprocessing for dimensional modeling  

#### 🔹 DIM (Type 1 Dimensions)
- Conformed dimensions (product, supplier, store, buyer, ABC, etc.)  
- Surrogate key generation  
- Many-to-many bridge resolution  

#### 🔹 FACT (Business Events)
Fact tables built at defined grain levels:

- **fact_sales** → revenue, cost, gross margin  
- **fact_inventory** → stock valuation (snapshot)  
- **fact_losses** → shrinkage valuation  
- **fact_purchase_items** → purchase-level granularity  
- **inventory_log** → stock movement tracking  

---

### 3️⃣ Storage Layer — AWS S3

Processed dimension and fact datasets are stored in structured folders (`/dim/`, `/fact/`) inside Amazon S3, enabling scalable cloud storage and analytical decoupling from the ERP.

---

### 4️⃣ Metadata & Query Layer — AWS Glue + Athena

- Schema cataloging via Glue  
- External tables  
- SQL querying through Athena  

---

### 5️⃣ BI Layer — Power BI

Athena connects directly to Power BI, enabling scalable dashboards for sales, margin, inventory, losses, and supplier performance — all powered by a centralized Data Warehouse.

---

## ⚙ Job Orchestration (KJB – Pentaho)

The pipeline is orchestrated using Pentaho Jobs (KJB), ensuring controlled execution, dependency management, and failure handling.

Two main execution flows were implemented: historical backfill and recurring daily load.

---

### 🟢 JOB_BACKFILL — Initial Historical Load

Purpose: Populate the Data Warehouse from scratch.

This job is designed to perform a full historical load and is typically executed during:

- Initial environment setup
- Major structural changes
- Data reprocessing scenarios

Execution flow:

- Start
- Build all DIM_* tables (Type 1)
- Load all FACT_* tables
- Validate row counts
- Success

Key characteristics:

- Full data extraction from ERP
- Complete rebuild of dimensions
- Full historical fact population
- Referential integrity validation

<img width="731" height="377" alt="image" src="https://github.com/user-attachments/assets/4d953b79-a63b-4fbc-a8c4-a33b1e884f20" />

---

### 🔵 JOB_DAILY — Recurring Incremental Load

Purpose: Maintain the Data Warehouse updated with minimal processing cost.

Execution frequency:
- Runs automatically once per day

Execution flow:

- Start
- Rebuild DIM_* (Type 1 overwrite strategy)
- Load FACT_* incrementally (or snapshot-based where applicable)
- Update structured datasets in AWS S3
- Trigger Glue schema refresh (if necessary)
- Success

Key characteristics:

- Incremental extraction logic
- Snapshot handling for inventory
- Controlled update of fact tables
- Cloud storage synchronization

This approach ensures performance efficiency while preserving data consistency.

<img width="742" height="381" alt="image" src="https://github.com/user-attachments/assets/e1ed7833-c7d8-47a4-997b-c16569ff49d9" />

---

### 🔴 Failure Handling & Monitoring

The orchestration layer includes production-oriented failure control mechanisms:

- Automatic retry (N configurable attempts)
- Email notification alerts
- Controlled abort on critical failures
- External log monitoring implemented
- Row count validation between layers

These mechanisms ensure:

- Data reliability
- Controlled execution
- Operational transparency
- Reduced risk of silent data corruption

---

### 🧠 Orchestration Strategy Highlights

- Separation between historical and incremental logic
- Clear dependency ordering (DIM → FACT)
- Idempotent job structure
- Monitoring-first mindset
- Designed for scalability and production stability

---

## 🧩 Applied Best Practices

This solution follows modern lake-based data engineering principles to ensure scalability and production readiness.

- Layered pipeline (STG → DIM → FACT)
- Logical Star Schema for BI optimization
- Decoupled storage (Amazon S3) and compute (Athena)
- Incremental and idempotent ETL processing
- Conformed dimensions with defined grain per fact
- External table exposure via AWS Glue

The architecture is designed for reliability, analytical consistency, and multi-store scalability.

---

## ⚙ Requirements

To run this project, you will need:

- Java (for Pentaho)
- Pentaho Data Integration (Spoon)
- PostgreSQL (ERP access)
- AWS account with:
  - S3
  - Glue
  - Athena
- Power BI Desktop
- Athena ODBC Driver

Valid database and AWS credentials are required.

---

## 📈 Results

The implemented architecture delivers:

- Centralized cloud-based analytical layer  
- Automated daily ETL pipeline  
- Decoupled ERP and BI workloads  
- Reliable and consistent KPI computation  

It enables:

- Revenue and margin monitoring  
- Inventory and loss tracking  
- Supplier performance and ABC analysis  

The solution is designed for scalable growth, multi-store expansion, and future orchestration enhancements.


