---
title: Databricks Batch Ingestion COPY INTO vs Table Write Operations
date: 2022-07-08
author: Rabindra Jaiswal
categories: [Architecture, databricks]
---

## Databricks Batch Ingestion: COPY INTO vs Table Write Operations

When designing batch ingestion pipelines in Databricks, it helps to separate two different concerns:

1. **How do I ingest data from the source?**
2. **How do I write or maintain data in the target table?**

This distinction makes it much easier to decide between `COPY INTO`, `INSERT INTO`, `INSERT OVERWRITE`, and `MERGE INTO`.

---

# Part 1: Data Ingestion

## COPY INTO — Ingest Files into a Table

`COPY INTO` is primarily an **ingestion mechanism**.

Use it when your source data is sitting in cloud storage such as S3 or ADLS and you need to load those files into a Delta table.

```sql
COPY INTO bronze.sales
FROM '/mnt/raw/sales/'
FILEFORMAT = PARQUET;
```

The typical pattern is:

```text
Cloud Storage
     │
     │ CSV / JSON / Parquet
     ▼
 COPY INTO
     │
     ▼
Bronze Delta Table
```

For example, suppose files arrive every day:

```text
/mnt/raw/sales/
    sales_2026-10-05.parquet
    sales_2026-10-06.parquet
    sales_2026-10-07.parquet
```

`COPY INTO` can incrementally ingest the files into the target table.

### When should you use COPY INTO?

Use it when:

- The source is files in cloud storage
- You are building a file-based batch ingestion pipeline
- You want Databricks to track which files have already been loaded

The important point is:

> **`COPY INTO` answers: "How do I get files into my table?"**

It is different from deciding how to maintain an existing table.

---

# Part 2: Table Write Operations

Once data is available as a dataset/DataFrame/table, the next question is:

> **What should happen to the existing data in the target table?**

This is where `INSERT INTO`, `INSERT OVERWRITE`, and `MERGE INTO` come in.

---

# 1. INSERT INTO — Append

Use `INSERT INTO` when the incoming data consists of **new records that should be added to the existing table**.

```sql
INSERT INTO target_sales
SELECT *
FROM new_sales;
```

Example:

```text
Existing table:

1  A
2  B

Incoming batch:

3  C
4  D
```

After `INSERT INTO`:

```text
1  A
2  B
3  C
4  D
```

The existing records remain untouched.

### Typical scenario

An append-only transaction or event table:

```text
Daily batch
     │
     ▼
New records
     │
     ▼
INSERT INTO
     │
     ▼
Target table
```

Use `INSERT INTO` when:

> **Existing data should stay, and incoming data is simply appended.**

---

# 2. INSERT OVERWRITE — Replace

Use `INSERT OVERWRITE` when the incoming data should **replace the existing data being targeted**.

```sql
INSERT OVERWRITE target_sales
SELECT *
FROM new_sales;
```

Example:

```text
Existing:

1  A
2  B
3  C

Incoming:

1  X
2  Y
```

After overwrite:

```text
1  X
2  Y
```

The previous target data is replaced.

A common batch scenario is partition replacement.

For example:

```text
sales
├── 2026-10-05
├── 2026-10-06
└── 2026-10-07
```

If the incoming batch contains the complete data for `2026-10-07`, you can replace that batch/partition rather than trying to identify individual changes.

### Typical scenarios

- Full refresh
- Reprocessing a batch
- Replacing a partition
- Rebuilding a specific slice of data

Use `INSERT OVERWRITE` when:

> **The incoming dataset is the new authoritative version of the data being replaced.**

---

# 3. MERGE INTO — Upsert

Use `MERGE INTO` when the incoming batch can contain both:

- Records that already exist and need to be updated
- Records that don't exist and need to be inserted

```sql
MERGE INTO target_sales t
USING source_sales s
ON t.customer_id = s.customer_id

WHEN MATCHED THEN
    UPDATE SET *

WHEN NOT MATCHED THEN
    INSERT *;
```

Example:

```text
Existing:

1  A
2  B
3  C

Incoming:

2  B_updated
4  D
```

After `MERGE`:

```text
1  A
2  B_updated
3  C
4  D
```

### Typical scenarios

`MERGE` is commonly used for:

- CDC
- Upserts
- Incremental source extracts
- SCD Type 1
- SCD Type 2

Use `MERGE` when:

> **The incoming batch needs to be matched against existing records to determine whether to insert or update.**

---

# COPY INTO vs Table Write Operations

This is the most important distinction.

```text
                 BATCH PIPELINE
                      │
          ┌───────────┴───────────┐
          │                       │
      INGESTION              TABLE WRITE
          │                       │
     COPY INTO             INSERT INTO
                              INSERT OVERWRITE
                              MERGE INTO
```

`COPY INTO` deals primarily with:

```text
Files → Table
```

Whereas table-write operations deal with:

```text
Dataset → Existing Table
```

So a pipeline could conceptually look like:

```text
                Cloud Storage
                     │
                     │
                 COPY INTO
                     │
                     ▼
                  Bronze
                     │
                     │ Transform
                     ▼
              Incoming Dataset
                     │
          ┌──────────┼──────────┐
          │          │          │
          ▼          ▼          ▼
       INSERT     OVERWRITE    MERGE
          │          │          │
          └──────────┼──────────┘
                     ▼
               Target Table
```

# Decision Guide

| Question | Operation |
|---|---|
| How do I ingest files from cloud storage? | `COPY INTO` |
| Do I only need to append new records? | `INSERT INTO` |
| Should the incoming data replace the existing data? | `INSERT OVERWRITE` |
| Do I need to update existing records and insert new ones? | `MERGE INTO` |

The simplest mental model is:

```text
COPY INTO
    ↓
"Ingest these files"

INSERT INTO
    ↓
"Add these rows"

INSERT OVERWRITE
    ↓
"Replace this data"

MERGE INTO
    ↓
"Compare and apply these changes"
```

This separation is useful when designing batch pipelines because **file ingestion and target-table maintenance are two different decisions**.