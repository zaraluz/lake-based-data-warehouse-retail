# 🏗️ ETL & Data Warehouse Architecture – Retail Supermarket

![AWS](https://img.shields.io/badge/AWS-S3%20%7C%20Athena%20%7C%20Glue-orange)
![Pentaho](https://img.shields.io/badge/Pentaho-Data%20Integration-blue)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-ERP-blue)
![Power BI](https://img.shields.io/badge/Power%20BI-Analytics-yellow)
![Status](https://img.shields.io/badge/Status-Production%20Ready-brightgreen)

---

## 📚 Table of Contents

- [Project Overview](#project-overview)
- [Business Objective](#business-objective)
- [Solution Architecture](#solution-architecture)
- [Data Flow](#data-flow)
- [Dimensional Modeling](#dimensional-modeling)
- [Job Orchestration](#job-orchestration)
- [Applied Best Practices](#applied-best-practices)
- [Requirements](#requirements)
- [Results](#results)
- [Future Improvements](#future-improvements)

---

## 📌 Project Overview

This project implements a production-oriented end-to-end ETL pipeline and Data Warehouse architecture for a retail supermarket operation.

The solution extracts transactional data from a PostgreSQL ERP system, applies structured transformations using Pentaho (Kettle), organizes data into a dimensional Star Schema model, and delivers a scalable cloud-based analytical layer using AWS S3 and Athena.

Unlike dashboard-only projects, this repository focuses on the engineering foundation that enables reliable, scalable, and automated business intelligence.

The architecture was designed to:

- Separate transactional and analytical workloads
- Ensure data consistency and reproducibility
- Support incremental and historical loads
- Enable cloud scalability
- Provide a stable foundation for executive dashboards

The pipeline runs automatically on a daily basis (incremental load) and does not require manual intervention.

---

## 🎯 Business Objective

The primary objective of this project is to transform raw ERP transactional data into structured, analytics-ready datasets that support strategic retail decision-making.

This architecture enables:

- Revenue and gross margin analysis
- Inventory valuation and stock monitoring
- Loss tracking and shrinkage control
- Supplier performance evaluation
- Purchase and ABC curve analysis
- Turnover and operational efficiency indicators

Beyond analytics, the architecture also aims to:

- Reduce dependency on local servers
- Improve data reliability and traceability
- Ensure reproducibility of KPI calculations
- Prepare the environment for multi-store or multi-client scalability

The Data Warehouse serves as the single source of truth for all BI dashboards.

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

The data pipeline follows a layered architecture designed to ensure separation of concerns, data reliability, and analytical scalability.

### 1️⃣ Source Layer — ERP (PostgreSQL)

Transactional data is extracted from a retail ERP system.

**Core transactional tables include:**

- sales
- inventory
- losses
- purchase_notes
- master_data (products, suppliers, stores, buyers)

Extraction is performed using SQL-based Table Input steps within Pentaho (PDI).

---

### 2️⃣ Processing Layer — Pentaho Data Integration (PDI)

Transformations (KTR) are structured into three logical layers:

---

#### 🔹 STG Layer (Standardization)

Purpose: Clean and standardize raw ERP data before dimensional modeling.

Main responsibilities:

- Column selection and renaming
- Data type enforcement
- Date normalization
- Null handling and trimming
- Key formatting and validation
- Basic row count checks

This layer ensures controlled and reproducible downstream transformations.

---

#### 🔹 DIM Layer (Type 1 Dimensions)

Purpose: Build conformed dimensions for analytical consistency.

Dimensions are rebuilt using Type 1 logic (overwrite strategy):

- dim_product
- dim_supplier
- dim_store
- dim_merchandising
- dim_buyer
- dim_abc_curve
- dim_abc_type
- bridge_product_supplier (many-to-many resolution)

Surrogate keys are generated to guarantee referential integrity.

---

#### 🔹 FACT Layer (Granularity & Business Calculations)

Purpose: Store measurable retail events at defined grain levels.

**fact_sales**
- total_revenue = sale_price × quantity
- total_cost = cost_with_tax × quantity
- gross_margin = total_revenue – total_cost
- grain: product × store × date × supplier

**fact_inventory**
- inventory_value = cost_with_tax × stock_quantity
- snapshot-based (by reference date)

**fact_losses**
- loss_value = cost_with_tax × loss_quantity
- linked to dim_loss_reason

**fact_purchase_items**
- purchase items at note-level granularity
- value, quantity, product, supplier

**inventory_log**
- audit trail for stock movement tracking

---

### 3️⃣ Storage Layer — AWS S3 (Data Lake)

Processed dimension and fact datasets are stored in structured folders:

- /dim/
- /fact/

This enables scalable storage and decouples transformation from analytics consumption.

---

### 4️⃣ Metadata Layer — AWS Glue + Data Catalog

- Automatic schema discovery
- External table updates
- Centralized metadata governance

---

### 5️⃣ Analytical Layer — AWS Athena + Power BI

- SQL querying on dimensional model
- BI dashboards for sales, margin, losses, suppliers, and inventory
- Single source of truth for retail KPIs

---

## 📊 Dimensional Modeling

The Data Warehouse follows a Star Schema design to optimize analytical performance and simplify BI consumption.

---

### ⭐ Fact Tables

The fact tables represent measurable retail events at defined grain levels:

- **fact_sales** → transactional sales events
- **fact_inventory** → inventory snapshot values
- **fact_losses** → recorded product losses
- **fact_purchase_items** → purchase note line items

Each fact table includes surrogate foreign keys linking to conformed dimensions.

---

### 📘 Dimension Tables (Type 1)

Dimensions provide descriptive business context and are rebuilt using Type 1 logic:

- dim_product
- dim_supplier
- dim_store
- dim_merchandising (category hierarchy)
- dim_buyer
- dim_abc_curve
- dim_abc_type

A bridge table resolves many-to-many relationships:

- bridge_product_supplier

---

### 🧩 Design Principles Applied

- Star Schema for performance optimization
- Conformed dimensions across facts
- Surrogate keys for referential integrity
- Defined grain per fact table
- Separation of transactional and analytical environments
- KPI reproducibility and consistency

The dimensional model guarantees fast analytical queries, scalable growth, and maintainable business logic.

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

This project follows modern data engineering and Data Warehouse best practices to ensure reliability, scalability, and maintainability.

### 🏗️ Architectural Principles

- Layered architecture (STG → DIM → FACT)
- Clear separation between transactional and analytical environments
- Decoupled storage (S3) and compute (Athena)
- Single source of truth for KPI calculation

---

### 📐 Data Modeling Best Practices

- Star Schema design for analytical performance
- Conformed dimensions across fact tables
- Surrogate keys to ensure referential integrity
- Defined grain per fact table
- Type 1 dimension strategy for simplified management

---

### 🔄 ETL & Processing Best Practices

- Incremental loading for performance optimization
- Snapshot strategy for inventory
- Idempotent job execution structure
- Controlled rebuild logic for dimensions
- Explicit business calculation layer (margin, cost, revenue)

---

### 🔎 Observability & Reliability

- Row count validation between layers
- Retry logic on job failure
- Email alert notification
- External log monitoring
- Controlled abort strategy to prevent silent corruption

---

### ☁️ Cloud & Scalability

- Data Lake storage pattern (/dim/ and /fact/)
- Schema management via AWS Glue Data Catalog
- Query decoupling via Athena
- Architecture designed for multi-client scalability

This ensures the solution is production-oriented and not a proof-of-concept pipeline.

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

### 🏢 Operational Impact

- Centralized analytical environment in the cloud
- Automated daily pipeline execution
- Reduced dependency on local database querying
- Reliable KPI computation across dashboards

---

### 📊 Analytical Capabilities Enabled

- Revenue and gross margin monitoring
- Loss tracking and shrinkage control
- Inventory valuation snapshots
- Supplier performance analysis
- ABC curve and purchasing insights

---

### ⚙ Engineering Improvements

- Decoupled transactional and analytical workloads
- Scalable cloud-based storage
- Production-oriented failure control
- Reproducible and auditable data transformations

---

### 🚀 Scalability Readiness

The architecture is prepared for:

- Multi-store expansion
- Multi-client adaptation
- Larger dataset growth
- Migration to orchestration tools such as Airflow (future roadmap)

This project establishes a robust data engineering foundation for retail analytics.

---

## 🚀 Future Improvements

While the current architecture is production-ready, the following improvements could further enhance scalability and governance:

- Implement partitioned Parquet storage for performance optimization
- Introduce Airflow for advanced workflow orchestration
- Add automated data quality validation checks
- Implement CDC (Change Data Capture) logic
- Adopt dbt for transformation standardization and testing
- Prepare infrastructure provisioning via Terraform

These enhancements would evolve the solution into a fully mature and scalable data platform.
