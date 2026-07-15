# PostgreSQL Performance Benchmark – LOGGED vs UNLOGGED Tables

---

# Environment

| Component | Version |
|-----------|---------|
| OS | Ubuntu 24.04 |
| PostgreSQL | 17 |
| Dataset | USPTO Patent Dataset |
| Dataset Size | ~10 Million Records |
| Data Format | Parquet → CSV |

---

# Step 1 - Download Patent Dataset

The patent dataset is downloaded directly from Kaggle using the Kaggle API.

```bash
#!/bin/bash

curl -L -o uspto-all-patents-after-1975.zip \
https://www.kaggle.com/api/v1/datasets/download/aerdem4/uspto-all-patents-after-1975
```

### Description

- Downloads the USPTO patent dataset from Kaggle.
- The downloaded file contains the patent data in Parquet format.
- Approximately 10 million patent records are available for benchmarking.


---

# Step 2 - Convert Parquet to CSV

DuckDB is used because it can efficiently convert very large Parquet files into CSV without writing custom scripts.

Install DuckDB

```bash
brew install duckdb
```

Start DuckDB

```bash
duckdb
```

Convert the dataset

```sql
COPY (
    SELECT *
    FROM 'all_patents.parquet'
)
TO 'all_patents.csv'
(
    FORMAT CSV,
    HEADER TRUE,
    DELIMITER ','
);
```

Exit DuckDB

```sql
.exit
```

### Description

- Reads the Parquet dataset.
- Converts it into CSV format.
- PostgreSQL COPY performs significantly better with CSV during bulk loading.

---

# Step 3 - Create LOGGED Table

Create the table.

```sql
CREATE TABLE patents.patents_logged (
    publication_number TEXT,
    title              TEXT,
    abstract           TEXT
);
```

### Description

A LOGGED table stores all write operations inside PostgreSQL Write Ahead Log (WAL). This provides crash recovery and data durability.

<img width="408" height="127" alt="Screenshot 2026-07-15 at 3 52 06 PM" src="https://github.com/user-attachments/assets/560571c9-1d5a-42c3-acc8-e14e266e8393" />


---

# Step 4 - Import Data into LOGGED Table

```sql
COPY patents.patents_logged
(
    publication_number,
    title,
    abstract
)
FROM '/tmp/all_patents.csv'
WITH (
    FORMAT CSV,
    HEADER TRUE,
    QUOTE '"',
    ESCAPE '"'
);
```

### Description

The PostgreSQL COPY command performs a high-speed bulk import directly from the CSV file.

This step is used to measure:

- Import Time

<img width="589" height="190" alt="Screenshot 2026-07-15 at 3 55 28 PM" src="https://github.com/user-attachments/assets/889fbcac-fcd7-46aa-a228-ec02a86775e4" />

---

# Step 5 - Storage Size Comparison

## Table Size

```sql
SELECT pg_size_pretty(
    pg_relation_size('patents.patents_unlogged')
) AS table_size;
```

### Description

Displays the physical storage occupied by the table.

### Screenshot
<img width="400" height="200" alt="Screenshot 2026-07-15 at 4 28 29 PM" src="https://github.com/user-attachments/assets/4893072a-fd8d-46fe-844a-400a1c0a8cc5" />

---

## Index Size

```sql
SELECT pg_size_pretty(
    pg_indexes_size('patents.patents_unlogged')
) AS index_size;
```

### Description

Displays the total disk space used by indexes.

If no indexes exist, the size will be reported as 0 bytes.

### Screenshot

<img width="409" height="187" alt="Screenshot 2026-07-15 at 4 28 36 PM" src="https://github.com/user-attachments/assets/8345a38b-182d-4950-91e0-2d980430f7a5" />


# Step 6 - Query Performance on LOGGED Table

## Query 1 - Primary Key Lookup

```sql
EXPLAIN (ANALYZE, BUFFERS)

SELECT *
FROM patents.patents_logged
WHERE publication_number='US-4081864-A';
```

### Purpose

Checks how quickly PostgreSQL retrieves a single patent using the publication number.

### Screenshot

<img width="1117" height="393" alt="Screenshot 2026-07-15 at 4 08 37 PM" src="https://github.com/user-attachments/assets/f54220ac-9264-415f-8822-d731369c1a42" />


---

## Query 2 - Prefix Search

```sql
EXPLAIN (ANALYZE, BUFFERS)

SELECT *
FROM patents.patents_logged
WHERE publication_number LIKE 'US-40818%';
```

### Purpose

Measures query performance for prefix-based searches.

### Screenshot

<img width="1119" height="447" alt="Screenshot 2026-07-15 at 4 08 57 PM" src="https://github.com/user-attachments/assets/f6025475-a069-4b40-98ea-1920ec6fa9b8" />


---

## Query 3 - Search by Title

```sql
EXPLAIN (ANALYZE, BUFFERS)

SELECT *
FROM patents.patents_logged
WHERE title ILIKE '%helmet%';
```

### Purpose

Searches for patents containing the keyword "helmet" in the title.

This query demonstrates sequential scanning performance without a text index.

### Screenshot

<img width="1119" height="411" alt="Screenshot 2026-07-15 at 4 09 08 PM" src="https://github.com/user-attachments/assets/34549ecc-5e75-44eb-9506-d5854e0274e3" />

---

## Query 4 - Search by Abstract

```sql
EXPLAIN (ANALYZE, BUFFERS)

SELECT publication_number,
       title
FROM patents.patents_logged
WHERE abstract ILIKE '%water%';
```

### Purpose

Searches patent abstracts containing the keyword "water".

Useful for evaluating large text search performance.

### Screenshot

<img width="1151" height="425" alt="Screenshot 2026-07-15 at 4 09 17 PM" src="https://github.com/user-attachments/assets/9d12ded3-e897-4fb0-8623-e616d3e7fd9c" />


---

# Step 7 - Create UNLOGGED Table

```sql
CREATE UNLOGGED TABLE patents.patents_unlogged (
    publication_number TEXT,
    title              TEXT,
    abstract           TEXT
);
```

### Description

UNLOGGED tables bypass PostgreSQL Write Ahead Logging (WAL), resulting in much faster bulk imports.

These tables are not crash-safe and are automatically truncated after an unexpected server crash.

### Screenshot

<img width="489" height="137" alt="Screenshot 2026-07-15 at 4 17 28 PM" src="https://github.com/user-attachments/assets/cbf2e44a-011f-49ae-bff4-c7eb926bcfc3" />

---

# Step 8 - Import Data into UNLOGGED Table

```sql
COPY patents.patents_unlogged
(
    publication_number,
    title,
    abstract
)
FROM '/tmp/all_patents.csv'
WITH (
    FORMAT CSV,
    HEADER TRUE,
    QUOTE '"',
    ESCAPE '"'
);
```

### Description

Imports the same dataset into the UNLOGGED table.

The import duration will later be compared with the LOGGED table.

This step is used to measure:

- Import Time
- CPU Usage

### Screenshot

<img width="629" height="194" alt="Screenshot 2026-07-15 at 4 22 38 PM" src="https://github.com/user-attachments/assets/a8c51d1b-0fc6-4c4f-a197-bfe4c58bdb1a" />
<img width="1920" height="563" alt="Screenshot 2026-07-15 at 4 19 02 PM" src="https://github.com/user-attachments/assets/2ab57c63-6d39-437b-a776-49d3e641309d" />

---

# Step 9 - Storage Size Comparison

## Table Size

```sql
SELECT pg_size_pretty(
    pg_relation_size('patents.patents_unlogged')
) AS table_size;
```

### Description

Displays the physical storage occupied by the table.

### Screenshot

<img width="406" height="173" alt="Screenshot 2026-07-15 at 4 24 02 PM" src="https://github.com/user-attachments/assets/4e0ea48c-2c21-41e6-a30e-de25f37df30b" />

---

## Index Size

```sql
SELECT pg_size_pretty(
    pg_indexes_size('patents.patents_unlogged')
) AS index_size;
```

### Description

Displays the total disk space used by indexes.

If no indexes exist, the size will be reported as 0 bytes.

### Screenshot

<img width="389" height="189" alt="Screenshot 2026-07-15 at 4 24 09 PM" src="https://github.com/user-attachments/assets/f90a2874-61d6-4a1c-a46c-b0168a5b5048" />

---

# Step 10 - Query Performance on UNLOGGED Table

## Query 1 - Primary Key Lookup

```sql
EXPLAIN (ANALYZE, BUFFERS)

SELECT *
FROM patents.patents_unlogged
WHERE publication_number='US-4081864-A';
```

### Purpose

Measures lookup performance on the UNLOGGED table.

### Screenshot

<img width="1129" height="469" alt="Screenshot 2026-07-15 at 4 26 59 PM" src="https://github.com/user-attachments/assets/01e8c0b4-306d-4bd7-ae91-1e08ca3a710d" />

---

## Query 2 - Prefix Search

```sql
EXPLAIN (ANALYZE, BUFFERS)

SELECT *
FROM patents.patents_unlogged
WHERE publication_number LIKE 'US-40818%';
```

### Purpose

Measures prefix search performance on the UNLOGGED table.

### Screenshot

<img width="1146" height="497" alt="Screenshot 2026-07-15 at 4 27 08 PM" src="https://github.com/user-attachments/assets/10465812-29d3-4e65-8142-3746fa2fd690" />

---

## Query 3 - Search by Title

```sql
EXPLAIN (ANALYZE, BUFFERS)

SELECT *
FROM patents.patents_unlogged
WHERE title ILIKE '%helmet%';
```

### Purpose

Evaluates title search performance using a sequential scan.

### Screenshot

<img width="1154" height="439" alt="Screenshot 2026-07-15 at 4 27 18 PM" src="https://github.com/user-attachments/assets/17576478-6abc-4e33-bdf8-95e6c53e1d0a" />

---

## Query 4 - Search by Abstract

```sql
EXPLAIN (ANALYZE, BUFFERS)

SELECT publication_number,
       title
FROM patents.patents_unlogged
WHERE abstract ILIKE '%water%';
```

### Purpose

Evaluates abstract search performance on the UNLOGGED table.

### Screenshot

<img width="1158" height="474" alt="Screenshot 2026-07-15 at 4 27 27 PM" src="https://github.com/user-attachments/assets/7730baab-2e83-46a4-a765-71fd3e829b1e" />

---

