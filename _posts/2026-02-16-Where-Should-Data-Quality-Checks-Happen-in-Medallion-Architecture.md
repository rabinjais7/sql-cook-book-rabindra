---
title: Where Should Data Quality Checks Happen in Medallion Architecture
date: 2026-02-16
author: Rabindra Jaiswal
categories: [Architecture, databricks]
---
# Where Should Data Quality Checks Happen in Medallion Architecture?

In a Medallion Architecture, data moves through three layers: Bronze, Silver, and Gold. Each layer serves a different purpose, and data quality checks should be applied at every stage.

A common question is: **At which layer should we perform data quality checks?**

The short answer is that data quality should be checked at all three layers, but the type and depth of validation should vary depending on the layer's responsibility.

## 1. Bronze Layer: Validate Data Ingestion

The Bronze layer stores raw data ingested from source systems. The primary goal is to preserve the original data while ensuring that ingestion has completed successfully.

Typical checks include:

- Schema and file format validation
- File corruption and ingestion failure detection
- Required ingestion metadata validation
- Duplicate file or batch ingestion detection
- Basic checks for unexpected schema changes

**Example:** If a source system delivers a customer CSV file, the Bronze layer can validate whether the file is readable, contains the expected columns, and has been ingested successfully.

At this stage, we generally avoid extensive business transformations because Bronze should retain the raw source data for traceability and reprocessing.

## 2. Silver Layer: Enforce Data Quality Rules

The Silver layer is where comprehensive data quality enforcement generally takes place. Data is cleaned, standardized, deduplicated, and validated against structural and business requirements.

Typical checks include:

- Mandatory field and null validation
- Duplicate business key detection
- Data type and format validation
- Referential integrity checks
- Valid date ranges and domain constraints
- Business rule validation
- Handling invalid records through quarantine or rejection

**Example:** Suppose a customer record contains a missing customer ID or an invalid email address. Silver-layer validation can identify the issue and either reject the record, quarantine it, or flag it for correction, depending on the defined rules.

Not every invalid record must be discarded. The appropriate action depends on the severity of the failure and the business requirements.

The objective is to make Silver data reliable and consistent for downstream processing.

## 3. Gold Layer: Validate Business Results

The Gold layer contains business-ready datasets, such as aggregated tables, reporting models, and analytical data products.

At this stage, validation focuses on whether the resulting data correctly represents the intended business metrics and calculations.

Typical checks include:

- Aggregate and KPI validation
- Revenue and financial reconciliation
- Record count reconciliation between layers
- Validation of calculated metrics
- Consistency checks across reporting datasets
- Business-specific threshold validation

**Example:** If Silver contains transaction-level data and Gold contains daily revenue by region, a reconciliation check can verify that the Gold revenue totals match the corresponding Silver transaction totals after accounting for the intended filters and business rules.

The objective is to ensure that reports and dashboards use trustworthy business results.

## Summary: Data Quality Across the Three Layers

| Layer | Primary focus | Example |
|---|---|---|
| Bronze | Ingestion quality | File readability and schema validation |
| Silver | Data quality enforcement | Null checks, deduplication, and business rules |
| Gold | Business result validation | Revenue reconciliation and KPI accuracy |

## Final Recommendation

Treat data quality as a continuous process rather than a single validation step.

- **Bronze:** Detect ingestion and source-data issues early.
- **Silver:** Apply the majority of structural, consistency, and business-level quality rules.
- **Gold:** Verify aggregates, KPIs, and business outcomes.

Silver is generally the primary layer for comprehensive data quality enforcement, but it does not replace validation in Bronze or Gold.

The exact placement of each rule should depend on its purpose, the consequences of failure, and the requirements of the data product.

**Key takeaway:** Bronze protects raw data integrity, Silver establishes data reliability, and Gold validates business correctness.