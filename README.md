# NZ Market Rent ELT Pipeline (Experience Snapshot)

This repository documents an end-to-end ELT analytics pipeline built around a public-sector rental market data source exposed via REST APIs.  
To keep the focus on architecture and outcomes (and to respect data sharing constraints), this repo shares **the data architecture** and **Power BI dashboard deliverables**, rather than transformation code or source endpoints.

---

## Overview

**Goal:** Deliver reliable, refreshable rental market analytics (trends + distribution metrics) for BI consumption.

**Key outcomes**
- Automated monthly ingestion with parameterised backfills
- Immutable raw extracts for traceability and reprocessing
- Dimensional **star-schema metrics mart** (aggregated statistical measures)
- Data quality validation (tests) and reproducible refreshes
- Power BI dashboard built on the serving/mart layer only

---

## Architecture

![Data Architecture](./images/nz-market-rent-architecture.png)

### Flow (high level)
1. **Source (Rental Market Data Provider)**  
   REST API pull for monthly rental market statistics.

2. **Orchestration (Azure Data Factory)**  
   - Parameterised ingestion (monthly + historical backfill runs)  
   - Retries / monitoring  
   - Run logging  
   - Re-runnable pipeline design (idempotent loads)

3. **Raw Storage (ADLS Gen2)**  
   - Lands **immutable** JSON extracts  
   - Partitioned by month (e.g., `period_ym=YYYY-MM`)  
   - Source-of-truth for replay/backfills

4. **Warehouse (Snowflake: RENT.RAW / RENT.STG / RENT.MART)**  
   - **RAW (Secure Landing):** stores raw semi-structured JSON (VARIANT) + file metadata for traceability  
   - **STG (Staging Layer):** flattens & normalises raw JSON (VARIANT → rows), type casting, standardised schema, DQ checks & dedup rules  
   - **MART (Serving Layer / Metrics Mart):** publishes **aggregated, non-identifiable statistical measures** using a dimensional star schema for BI

5. **Transformation (dbt)**  
   - Models: `RAW → STG → MART`  
   - Tests: `not_null`, `unique`, `accepted_values` + statistical sanity checks  
   - Documentation + lineage (dbt docs)  
   - Incremental builds for monthly refresh & backfills (triggered by ADF runs)

6. **Consume (Power BI)**  
   - Connects to **Serving/MART only**  
   - Dashboards for trends and distribution metrics

---

## Data Model (Star Schema)

**Grain (fact table):** `period_ym × area × dwelling_type × bedrooms`

**Dimensions**
- `dim_period` — month/year attributes for time slicing
- `dim_area` — area/city attributes
- `dim_dwelling` — dwelling types (house/apartment/flat/etc.)
- `dim_bedrooms` — bedroom categories (1–5, 5+, etc.)

**Fact**
- `fct_market_rent_stats` — aggregated statistical measures:
  - Distribution: `median`, `mean`, `lower_quartile`, `upper_quartile`, `std_dev`
  - Counts: `n_current`, `n_lodged`, `n_closed` (coverage/sample indicators)

---

## Data Quality & Reliability

- **Re-runnable orchestration:** idempotent monthly refresh with parameterised backfills
- **Traceability:** immutable raw files + warehouse landing metadata to support reprocessing
- **Quality checks:** dimensional key integrity, accepted value constraints, and sanity rules for statistical measures

---

## Power BI Dashboard

> Screenshots below show example pages built on the **Serving/MART layer**.

![Dashboard - Trends](./images/powerbi-trends.png)
![Dashboard - Distribution](./images/powerbi-distribution.png)

### Example analytics included
- Monthly rental trends by area / dwelling / bedroom segment
- Distribution metrics (median and quartiles) to avoid over-relying on averages
- Coverage indicators using available counts

---

## Notes on Data Sharing

This repo intentionally avoids publishing raw extracts, transformation code, or source endpoints.  
The focus is to showcase **production-style architecture**, layered modelling (RAW/STG/MART), data quality practices, and BI outputs. The serving layer is designed as a **metrics mart** containing aggregated, non-identifiable measures suitable for reporting.

---

## Tech Stack

Azure Data Factory • ADLS Gen2 • Snowflake • dbt • Power BI • SQL • Python • Git
