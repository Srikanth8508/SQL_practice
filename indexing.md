# PostgreSQL Indexing Benchmark

## Objective
This benchmark evaluates the effect of maintaining a unique MD5 expression index during bulk data loading and query execution. Two scenarios are compared:

- **`patents_idx`** – Index exists before `COPY` (rows are indexed during loading).
- **`patents_n_idx`** – Data is loaded first, then the index is created (recommended approach for bulk loading).



---

# Scenario 1 – Index Created Before Data Load

## Create Table

```sql
CREATE TABLE patents.patents_idx (
    publication_number TEXT,
    title TEXT,
    abstract TEXT
);
```

**Technical Explanation**

- Creates a heap table for storing patent records.
- No indexes exist initially.

---

## Create Unique MD5 Expression Index

```sql
CREATE UNIQUE INDEX idx_patents_md5
ON patents.patents_idx (
    md5(
        COALESCE(publication_number, '') ||
        COALESCE(title, '') ||
        COALESCE(abstract, '')
    )
);
```
# Verify Table

```sql
\d patents.patents_idx
```

**Technical Explanation**

- Creates a **unique B-tree expression index**.
- Stores the MD5 hash of the concatenated column values.
- `COALESCE()` converts NULL values to empty strings to produce deterministic hashes.
- Prevents duplicate patent records having identical values across all three columns.
- During `COPY`, PostgreSQL updates the index for every inserted row, increasing write I/O and load time.
- Displays table definition, columns, storage details and attached indexes.

 
<img width="944" height="649" alt="Screenshot 2026-07-16 at 6 24 31 PM" src="https://github.com/user-attachments/assets/6064ba6b-a297-4491-ad77-7d413038862d" />

---

## Bulk Load

```sql
COPY patents.patents_idx
(
    publication_number,
    title,
    abstract
)
FROM '/var/lib/postgresql/import/all_patents.csv'
WITH (
    FORMAT CSV,
    HEADER TRUE,
    QUOTE '"',
    ESCAPE '"'
);
```

**Technical Explanation**

- Uses PostgreSQL server-side bulk loading.
- Since the index already exists, every inserted row updates the index immediately.
- This increases CPU usage, WAL generation and random disk writes compared to loading into an unindexed table.

 

<img width="692" height="448" alt="Screenshot 2026-07-16 at 6 43 44 PM" src="https://github.com/user-attachments/assets/36651fa6-77ba-4d4c-91de-683ca7791fc4" />
<img width="752" height="753" alt="Screenshot 2026-07-16 at 6 25 03 PM" src="https://github.com/user-attachments/assets/6443307f-bdd7-4a67-b11d-a09ce31db1f8" />

---

## Storage Analysis

### Table Size

```sql
SELECT pg_size_pretty(pg_relation_size('patents.patents_idx'));
```

**Technical Explanation**

Returns only heap storage.

 
<img width="405" height="204" alt="Screenshot 2026-07-16 at 7 09 17 PM" src="https://github.com/user-attachments/assets/5a597e21-ab49-47c5-9606-4d2e50aba191" />


### Index Size

```sql
SELECT pg_size_pretty(pg_indexes_size('patents.patents_idx'));
```

**Technical Explanation**

Returns total disk space consumed by all indexes.

 
<img width="423" height="202" alt="Screenshot 2026-07-16 at 7 19 48 PM" src="https://github.com/user-attachments/assets/1392ddd2-fa11-4c0e-9b80-abb4aab658f8" />

---

# Query Performance Tests

## 1. Exact Publication Number Lookup

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM patents.patents_idx
WHERE publication_number = 'US-4081864-A';
```

**Technical Explanation**

- This query filters on `publication_number`.
- The existing index is on the MD5 expression, **not** on `publication_number`.
- Therefore PostgreSQL cannot use the MD5 index for this predicate.
- Expect a **Parallel Sequential Scan** unless a separate B-tree index exists on `publication_number`.

**Observe**

- Scan type
- Execution Time
- Shared Buffers
- Rows Removed by Filter

 

<img width="1086" height="452" alt="Screenshot 2026-07-16 at 6 44 03 PM" src="https://github.com/user-attachments/assets/5fa4273c-09bc-4561-8b34-7efb2cb48eb3" />


---

## 2. Prefix Search

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM patents.patents_idx
WHERE publication_number LIKE 'US-40818%';
```

**Technical Explanation**

- Prefix search requires an index on `publication_number`.
- The MD5 expression index cannot support prefix matching.
- PostgreSQL scans the table and evaluates the LIKE predicate for every row.

 

<img width="1094" height="429" alt="Screenshot 2026-07-16 at 7 21 11 PM" src="https://github.com/user-attachments/assets/edab5b56-d334-4abe-8c3f-420b25cf69cb" />

---

## 3. Title Search

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM patents.patents_idx
WHERE title ILIKE '%helmet%';
```

**Technical Explanation**

- `ILIKE '%text%'` cannot use a normal B-tree index.
- PostgreSQL performs a Parallel Sequential Scan.
- For production workloads, a GIN index with `pg_trgm` is recommended.

 
<img width="1088" height="427" alt="Screenshot 2026-07-16 at 7 21 21 PM" src="https://github.com/user-attachments/assets/d7af51c6-da6d-4ffd-9e56-d83f1135a786" />

---

## 4. Abstract Search

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT publication_number,title
FROM patents.patents_idx
WHERE abstract ILIKE '%water%';
```

**Technical Explanation**

- Leading wildcard prevents B-tree index usage.
- Large result sets increase execution time due to tuple retrieval.

 
<img width="1130" height="446" alt="Screenshot 2026-07-16 at 7 21 33 PM" src="https://github.com/user-attachments/assets/09ef6724-9375-4f9c-ab90-dc1e7f1fae2d" />

---

# Scenario 2 – Load First, Build Index Later

(Repeat the same SQL statements for `patents_n_idx`.)

---

# Analysis Notes (Based on EXPLAIN Output)

The attached execution plan for `patents_n_idx` shows:

## Exact Lookup
- Planner selected **Parallel Sequential Scan**.
- Approximately 9.45 million rows were scanned.
- Over 1 million shared buffers were read.
- Execution time ≈ **5.47 seconds**.

## Prefix Search
- Also performed a **Parallel Sequential Scan**.
- MD5 expression index cannot satisfy `LIKE 'US-40818%'`.
- Execution time ≈ **5.37 seconds**.

## Title Search
- `ILIKE '%helmet%'` resulted in a **Parallel Sequential Scan**.
- Around 5,148 matching rows were returned.
- Execution time ≈ **6.22 seconds**.

## Abstract Search
- Returned approximately **361,777 rows**.
- Required scanning nearly the entire table.
- Execution time ≈ **21.4 seconds** because of the high number of matching rows and tuple retrieval.

---

# Important Observation

Although an MD5 unique index exists, **none of the benchmark queries use it** because every query filters on original text columns (`publication_number`, `title`, or `abstract`) rather than on the indexed MD5 expression.

The MD5 index is useful only for:
- Duplicate detection
- Enforcing uniqueness
- Searching by the same MD5 expression

It **does not** accelerate:
- Equality search on `publication_number`
- Prefix (`LIKE`) search
- `ILIKE` text search
- Full-text pattern matching

To optimize these benchmark queries, consider:

| Query | Recommended Index |
|--------|-------------------|
| publication_number = value | B-tree on publication_number |
| publication_number LIKE 'ABC%' | B-tree with text_pattern_ops (or appropriate collation) |
| title ILIKE '%helmet%' | GIN + pg_trgm |
| abstract ILIKE '%water%' | GIN + pg_trgm |



---

# Scenario 2 – Load Data First, Create Index Later

## Create Table

```sql
CREATE TABLE patents.patents_n_idx (
    publication_number TEXT,
    title TEXT,
    abstract TEXT
);
```

**Technical Explanation**

- Creates a heap table identical to `patents_idx`.
- No indexes exist at this stage, allowing PostgreSQL to insert rows without maintaining index structures.

 ** *(Attach here)*

---

## Bulk Load

```sql
COPY patents.patents_n_idx
(
    publication_number,
    title,
    abstract
)
FROM '/var/lib/postgresql/import/all_patents.csv'
WITH (
    FORMAT CSV,
    HEADER TRUE,
    QUOTE '"',
    ESCAPE '"'
);
```

**Technical Explanation**

- Loads the entire dataset into a table without indexes.
- PostgreSQL writes only heap pages during loading.
- No index maintenance occurs for each inserted row.
- This is the recommended approach for loading very large datasets because it significantly reduces write amplification, WAL generation, CPU utilization, and random I/O.

 ** *(Attach here)*

---

## Create Unique MD5 Expression Index

```sql
CREATE UNIQUE INDEX n_idx_patents_md5
ON patents.patents_n_idx (
    md5(
        COALESCE(publication_number, '') ||
        COALESCE(title, '') ||
        COALESCE(abstract, '')
    )
);
```

**Technical Explanation**

- After the data load completes, PostgreSQL performs a single scan of the table and builds the index.
- Index creation is generally much faster than maintaining the index during every row insertion.
- The resulting index is identical to the one created in Scenario 1 but is built more efficiently.

 ** *(Attach here)*

---

## Verify Table

```sql
\d+ patents.patents_n_idx
```

**Technical Explanation**

Displays table definition, storage information, indexes, table size and additional metadata.

 ** *(Attach here)*

---

## Storage Analysis

### Table Size

```sql
SELECT pg_size_pretty(
    pg_relation_size('patents.patents_n_idx')
) AS table_size;
```

**Technical Explanation**

Returns the heap table size only.

 ** *(Attach here)*

### Index Size

```sql
SELECT pg_size_pretty(
    pg_indexes_size('patents.patents_n_idx')
) AS index_size;
```

**Technical Explanation**

Returns the total storage occupied by indexes.

 ** *(Attach here)*

---

# Query Performance Tests

## 1. Exact Publication Number Lookup

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM patents.patents_n_idx
WHERE publication_number = 'US-4081864-A';
```

**Technical Explanation**

- PostgreSQL performs a **Parallel Sequential Scan** because the only index is on the MD5 expression.
- The predicate filters on `publication_number`, which has no dedicated B-tree index.
- The planner scans the complete table and evaluates the filter for every row.

**Major Observations (Based on Execution Plan)**

- Parallel Sequential Scan
- Workers Planned: 2
- Workers Launched: 2
- Rows Removed by Filter: ~9.45 million
- Shared Buffers: ~1.09 million pages read
- Execution Time: **~5.47 seconds**

 ** *(Attach EXPLAIN output here)*

---

## 2. Prefix Search

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM patents.patents_n_idx
WHERE publication_number LIKE 'US-40818%';
```

**Technical Explanation**

- The MD5 expression index cannot support prefix matching.
- PostgreSQL performs a Parallel Sequential Scan over the entire table.

**Major Observations**

- Parallel Sequential Scan
- Nearly every row evaluated
- Execution Time: **~5.37 seconds**
- High buffer reads indicate a full table scan.

 ** *(Attach here)*

---

## 3. Search by Title

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM patents.patents_n_idx
WHERE title ILIKE '%helmet%';
```

**Technical Explanation**

- `ILIKE '%helmet%'` cannot use a standard B-tree index.
- PostgreSQL scans every row and performs a case-insensitive pattern comparison.
- A GIN index with `pg_trgm` is recommended for this workload.

**Major Observations**

- Parallel Sequential Scan
- Returned 5,148 rows
- Rows Removed by Filter: ~9.45 million
- Execution Time: **~6.22 seconds**

 ** *(Attach here)*

---

## 4. Search in Abstract

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT publication_number, title
FROM patents.patents_n_idx
WHERE abstract ILIKE '%water%';
```

**Technical Explanation**

- Leading wildcard prevents B-tree index usage.
- PostgreSQL scans the complete table.
- Large number of matching rows further increases execution time because many tuples must be returned.

**Major Observations**

- Parallel Sequential Scan
- Returned approximately 361,777 rows
- Shared Buffers: ~1.09 million pages read
- Execution Time: **~21.4 seconds**

 ** *(Attach here)*

---

