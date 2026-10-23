---
title: Building Declarative ETL Pipelines with Delta Live Tables
date: 2022-03-30
author: Rabindra Jaiswal
categories:
  - Data Engineering
  - Databricks
  - Delta Live Tables
  - ETL
  - Lakehouse Architecture
---

# Building Declarative ETL Pipelines with Delta Live Tables

Building robust and scalable Extract, Transform, Load (ETL) pipelines is often a complex task, especially when dealing with a mix of batch and streaming data. Data engineers traditionally spend a significant amount of time managing infrastructure, orchestrating tasks, and ensuring data quality, rather than focusing on the core transformation logic. This often leads to delays in delivering fresh, reliable data for analytics and machine learning applications. Delta Live Tables (DLT) on Databricks offers a refreshing declarative approach that simplifies this entire process, allowing teams to build reliable data pipelines with greater efficiency and less operational overhead.

## The Power of Declarative Pipelines

At its core, DLT introduces a declarative framework for defining data transformations. Instead of writing imperative code that dictates *how* each step of the pipeline should execute, data engineers simply declare *what* the desired outcome of their data transformations should be. DLT then takes this high-level definition and automatically manages the underlying infrastructure, orchestrating the execution graph, handling dependencies, and optimising performance. This is achieved by understanding the lineage between tables declared in SQL or Python. For instance, if a `silver` table depends on a `bronze` table, DLT automatically knows the processing order. This shift from imperative to declarative significantly reduces complexity, improves maintainability, and makes pipelines more resilient to changes in data volume or schema.

## A Solid Foundation The Medallion Architecture

A common and highly effective pattern for structuring data lakes is the Medallion Architecture, which organises data into three layers: Bronze, Silver, and Gold. DLT naturally aligns with this approach:

*   **Bronze Layer:** This layer holds raw, unprocessed data directly ingested from source systems. DLT uses `cloud_files` with Auto Loader to efficiently ingest data from cloud storage, automatically inferring schema and handling schema evolution. This is crucial for sources like REST APIs where payloads can vary.
    ```sql
    -- Bronze layer: Ingest raw streaming sales data from cloud files
    CREATE STREAMING LIVE TABLE raw_retail_sales
    COMMENT "Raw streaming sales data from POS systems, ingested via Auto Loader."
    TBLPROPERTIES ("quality" = "bronze")
    AS SELECT *
       FROM cloud_files(
         "/mnt/raw_data/retail_sales_json",
         "json",
         map("cloudFiles.inferColumnTypes", "true", "cloudFiles.maxFilesPerTrigger", "100")
       );
    ```
*   **Silver Layer:** Data in the Silver layer is cleaned, enriched, and transformed, ready for downstream analytics. This is where data quality constraints are enforced.
*   **Gold Layer:** The Gold layer contains highly aggregated and refined data, tailored for specific business intelligence dashboards or machine learning models.

DLT seamlessly manages the flow of data between these layers, ensuring that each transformation is applied incrementally and reliably.

## Ensuring Data Quality and Reliability

One of DLT's standout features is its built-in data quality controls. Data engineers can define expectations (constraints) directly within their table definitions using the `CONSTRAINT` clause. These expectations can dictate how the pipeline should react to violating records:
*   `EXPECT`: Flags the violating record but allows it to pass through.
*   `EXPECT OR DROP ROW`: Drops records that fail the expectation.
*   `EXPECT OR FAIL`: Halts the entire pipeline if an expectation is violated, useful for critical data quality issues.

All data quality metrics and pipeline events are captured in an underlying Delta table, the "event log." This provides a powerful auditing mechanism, allowing data teams to monitor pipeline health, track data quality trends over time, and quickly identify the root cause of issues. This capability provides immense confidence in the data assets being produced.

## Seamless Batch and Streaming Integration

DLT abstracts away the complexities of handling both batch and streaming data. Data engineers define their pipelines once, and DLT intelligently determines whether to process data in batch or streaming mode based on the source and table definitions (`LIVE TABLE` for batch, `STREAMING LIVE TABLE` for streaming). This unified approach simplifies pipeline design significantly, eliminating the need for separate codebases or frameworks for different data ingestion patterns. Auto Loader, deeply integrated with DLT, automatically handles new file arrivals and schema evolution, making it ideal for continuous data ingestion from sources like application logs or IoT device streams.

## Automated Operations for Peace of Mind

The operational burden of managing ETL pipelines is a major challenge for many organisations. DLT addresses this by automating complex and time-consuming activities. It automatically handles task orchestration, ensuring dependencies are met and stages execute in the correct order. Error handling and recovery are also automated, with DLT retrying failed operations or intelligently restarting pipelines from the point of failure. The platform also offers auto-scaling, dynamically adjusting cluster resources based on workload, which optimises performance and controls costs. Furthermore, DLT allows pipelines to run in either a `continuous` mode for near real-time updates or a `triggered` mode for periodic, cost-efficient batch processing, giving flexibility based on business requirements.

## Real World Impact Retail Inventory Management

Consider a large retail chain managing inventory across hundreds of stores. They need to keep track of sales transactions from Point of Sale (POS) systems, stock movements from warehouse scanners, and potentially even shelf availability from IoT sensors. All this data arrives continuously and needs to be consolidated, cleaned, and made available in near real-time for accurate inventory levels, demand forecasting, and supply chain optimisation.

Using DLT, the retail chain can build a robust data pipeline:
*   **Bronze Layer:** Raw JSON data from POS systems, CSV files from warehouse scanners, and sensor data are ingested into separate `STREAMING LIVE TABLE`s using Auto Loader. Schema inference handles variations in POS payloads.
*   **Silver Layer:** Dedicated `STREAMING LIVE TABLE`s process these bronze tables. For instance, a `cleaned_sales` table enforces constraints like `EXPECT (transaction_id IS NOT NULL)` and `EXPECT OR DROP ROW (quantity > 0)`. `APPLY CHANGES INTO` can be used to manage product master data, ensuring product descriptions and pricing are always up-to-date by merging new or updated records into a `product_dimensions` table.
*   **Gold Layer:** Aggregate tables like `realtime_store_inventory` or `daily_sales_summary` provide key metrics for business analysts and automated systems.

This DLT-powered pipeline provides fresh, reliable data, empowering the retail chain to make timely decisions, reduce out-of-stock situations, and improve customer satisfaction, all while significantly reducing the operational overhead on the data engineering team.

## Debugging and Operational Insights

Even with automated pipelines, understanding what is happening under the hood and troubleshooting issues is crucial. DLT offers powerful debugging capabilities. The pipeline's graphical user interface clearly visualises the data lineage, showing the flow from source to target. When issues arise, the underlying event log, which is itself a Delta table, becomes an invaluable resource. Data engineers can query this log using standard SQL to investigate specific events, such as when a pipeline started, when a table was created, or when data quality constraints were violated. Queries on the event log can reveal `data_quality` metrics, `flow_progress` details, and `system_event` records that pinpoint the exact stage where a problem occurred or how a schema change was handled. This detailed visibility ensures that potential problems can be identified and resolved quickly, maintaining pipeline integrity and data reliability.

Delta Live Tables represents a significant step forward in simplifying data engineering workflows. By embracing a declarative approach and automating complex operational tasks, it allows data teams to focus on delivering business value through data, rather than getting bogged down in infrastructure management.

Originally posted at [Medium](https://medium.com/@rabinjais7/building-declarative-etl-pipelines-with-delta-live-tables)