---
layout: post
title: "Handling Schema Drift in Databricks: Type Widening, Quarantine, and Alerting"
date: 2026-10-23
categories: [Databricks, Data Architecture]
tags: [schema-evolution, delta-lake, auto-loader, data-quality]
description: "A practical architecture for handling schema drift in Databricks using controlled type widening, quarantined records, and operational alerting."
---

# Handling Schema Drift in Databricks: Type Widening, Quarantine, and Alerting

Schema drift is rarely just an ingestion problem. An upstream application can introduce a new field, change a numeric data type, or send a value that no longer matches the expected schema. The immediate effect might be a failed write, but the larger risk is that downstream transformations continue with incomplete or incorrectly interpreted data.

A resilient data architecture should not treat every schema change as a failure, nor should it accept every change automatically.

For Databricks and Delta Lake, a practical strategy combines controlled schema evolution, record-level quarantine, and operational alerting.

## 1. Not All Schema Changes Are Equal

Before implementing schema evolution, classify the changes the pipeline can encounter.

- **Additive changes:** A new optional field appears in the source.
- **Compatible type widening:** An integer column needs to accommodate larger values, such as `INT` to `BIGINT`.
- **Incompatible type changes:** A field changes from an integer to a string, or its meaning changes.
- **Malformed records:** The incoming payload cannot be parsed correctly.

These changes require different handling. Adding a new field might be safe, while changing the meaning or representation of an existing field could invalidate downstream business logic.

The first architectural decision is therefore to define which changes can be accepted automatically and which require intervention.

## 2. Use Bronze to Preserve Incoming Data

The Bronze layer should provide a durable landing point for source data, with enough information to investigate and replay problematic records.

For file-based ingestion, Databricks Auto Loader supports schema inference, schema evolution, and rescued data. Its `_rescued_data` column can capture fields that do not match the expected schema, including certain type mismatches and unexpected columns.

For sources with unpredictable structures, retaining the original payload or using an appropriate semi-structured representation can provide additional flexibility.

However, preserving raw data does not eliminate the need for schema validation. It provides a recovery mechanism when assumptions about the source turn out to be incorrect.

Documentation: [Configure schema inference and evolution in Auto Loader](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/schema).

## 3. Enable Type Widening for Compatible Changes

Delta Lake supports type widening for selected conversions without rewriting existing data files. For example, an integer column can be widened to a larger integer type when the conversion is supported.

For an existing Delta table, type widening can be enabled using:

```sql
ALTER TABLE catalog_name.schema_name.customer
SET TBLPROPERTIES (
    'delta.enableTypeWidening' = 'true'
);
```

'''python
.option(
    "cloudFiles.schemaEvolutionMode",
    "addNewColumnsWithTypeWidening"
)
'''


'''python
from pyspark.sql.functions import col, lit

def process_batch(batch_df, batch_id):
    clean_df = batch_df.filter(
        col("_rescued_data").isNull()
    ).drop("_rescued_data")

    quarantine_df = (
        batch_df
        .filter(col("_rescued_data").isNotNull())
        .withColumn(
            "quarantine_reason",
            lit("Schema mismatch or unexpected fields")
        )
    )

    # Write valid records to the main Delta table
    (
        clean_df.write
        .format("delta")
        .mode("append")
        .option("mergeSchema", "true")
        .saveAsTable("main.bronze.customers")
    )

    # Write rescued records to quarantine
    (
        quarantine_df.write
        .format("delta")
        .mode("append")
        .option("mergeSchema", "true")
        .saveAsTable("main.bronze.customers_quarantine")
    )


query = (
    df.writeStream
    .foreachBatch(process_batch)
    .option("checkpointLocation", checkpoint_path)
    .start()
)
'''
This capability is available on supported Delta tables in Databricks Runtime 15.4 LTS and above. Enabling the feature also upgrades the table's reader and writer protocols, which should be considered when evaluating compatibility with other Delta Lake clients.

Type widening is not a general-purpose solution for arbitrary schema changes. A conversion from `INT` to `STRING`, for example, is not a supported widening operation.

For Auto Loader, Databricks documents an additional schema evolution mode, `addNewColumnsWithTypeWidening`, which automatically handles supported column additions and type widening. This mode is documented as Public Preview in Databricks Runtime 16.4 and above.

Documentation: [Delta Lake type widening](https://docs.databricks.com/aws/en/tables/features/type-widening) and [Automatic type widening with Auto Loader](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/type-widening).

The architectural principle is simple: automate compatible changes, but do not interpret every type change as compatible.

## 4. Quarantine Records That Cannot Be Processed Safely

Type widening reduces failures caused by supported type promotions, but it does not resolve every schema mismatch.

Consider a source that normally sends:

```json
{
  "customer_id": 101,
  "customer_name": "Amit"
}
```

A later record arrives with an unexpected representation:

```json
{
  "customer_id": "unknown",
  "customer_name": "Amit"
}
```

If the target contract requires `customer_id` to be numeric, the pipeline needs an explicit policy for this record.

One option is to preserve the record in a quarantine table rather than allowing an invalid value into the curated dataset.

A quarantine record should retain the original payload or rescued fields, source information, ingestion timestamp, failure reason, and processing status where applicable. This makes the issue traceable and supports later remediation.

Auto Loader's `_rescued_data` column can serve as an input to this process, but it is not itself a complete dead-letter queue. The pipeline must explicitly identify the affected records and route them to the appropriate destination.

Malformed JSON should also be handled separately. A type mismatch and a syntactically invalid JSON document are different failure conditions and may require different parser settings.

## 5. Add Alerting and Operational Visibility

Quarantining data protects the curated dataset, but it does not tell the team whether the issue is isolated or indicates a breaking upstream change.

Monitoring should distinguish at least three events:

- **Schema change detected:** A new field or supported type change has occurred.
- **Data quarantined:** One or more records could not be processed under the current contract.
- **Pipeline failure:** The ingestion or transformation process could not complete.

Not every event needs the same severity. A new optional field may generate an informational notification, while a sudden increase in quarantined records may require investigation. A failed critical pipeline may need an immediate operational alert.

In Databricks, Lakeflow Jobs supports notifications for job and task events, including failures and duration thresholds. SQL alerts can also be incorporated into jobs for condition-based monitoring.

For quarantine monitoring, measure the number of newly rejected records over a defined interval, along with the rejection rate and affected source. Alerting whenever a historical quarantine table contains any records can create repeated notifications even when no new problem has occurred.

Documentation: [Lakeflow Jobs notifications](https://docs.databricks.com/aws/en/jobs) and [SQL alert tasks for jobs](https://docs.databricks.com/aws/en/jobs/tasks/alert).

## 6. Enforce Contracts in Silver and Gold

Automatic schema evolution should not remove the need for data contracts.

The Silver layer should apply explicit projections, type conversions, required-field checks, and business validations. The Gold layer should expose stable business entities and metrics rather than inherit every upstream schema change.

For example, adding a new source field might be safe to accommodate in Bronze. Promoting that field into Silver should depend on its purpose, expected type, and validation rules. A breaking change to a critical business identifier may require a controlled migration instead of automatic acceptance.

This separation allows ingestion to remain flexible without making downstream consumers unpredictable.


## Configure Auto Loader with schema evolution

'''python
from pyspark.sql.functions import col, current_timestamp

source_path = "/Volumes/main/bronze/landing/customers/"
schema_path = "/Volumes/main/bronze/checkpoints/customer_schema/"
checkpoint_path = "/Volumes/main/bronze/checkpoints/customer_stream/"

df = (
    spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", schema_path)
    .option("cloudFiles.schemaEvolutionMode", "addNewColumns")
    .option("rescuedDataColumn", "_rescued_data")
    .load(source_path)
    .withColumn("ingested_at", current_timestamp())
)'''

## Putting the Strategy Together

The resulting architecture has four responsibilities:

1. **Preserve:** Land incoming data in Bronze and retain enough information for recovery.
2. **Evolve:** Apply supported schema changes and type widening according to an explicit policy.
3. **Isolate:** Quarantine records that violate the expected data contract.
4. **Observe:** Alert on schema changes, quarantined records, and pipeline failures.

These responsibilities should be implemented independently, with clear ownership and recovery procedures.

A schema change that is safe for ingestion may still be unsafe for business reporting. Conversely, rejecting every new field can create unnecessary operational work. The goal is to automate the changes that are understood and isolate those that are not.

## Conclusion

Schema drift should be treated as a governed data-quality concern rather than simply a Spark configuration problem.

In Databricks, type widening, rescued data, quarantine tables, and workflow notifications provide useful building blocks. Their effectiveness depends on how they are combined with explicit data contracts and downstream validation.

The key architectural decision is not whether to allow schema evolution. It is deciding **which changes can be accepted automatically, which records must be isolated, and which events require human intervention**.
