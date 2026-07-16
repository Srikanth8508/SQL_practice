# PostgreSQL Partitioned Table Benchmark (HASH Partitioning)

---

# 1. Create HASH Partitioned Table

## Query

```sql
CREATE TABLE patents.patents_partitioned (
    publication_number TEXT,
    title TEXT,
    abstract TEXT
)
PARTITION BY HASH (publication_number);

CREATE TABLE patents.p0
PARTITION OF patents.patents_partitioned
FOR VALUES WITH (MODULUS 4, REMAINDER 0);

CREATE TABLE patents.p1
PARTITION OF patents.patents_partitioned
FOR VALUES WITH (MODULUS 4, REMAINDER 1);

CREATE TABLE patents.p2
PARTITION OF patents.patents_partitioned
FOR VALUES WITH (MODULUS 4, REMAINDER 2);

CREATE TABLE patents.p3
PARTITION OF patents.patents_partitioned
FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

---

## Technical Explanation

This query creates a **partitioned parent table** using **HASH partitioning**.

Instead of storing rows in a single table, PostgreSQL automatically distributes rows across four child partitions based on the hash value of the `publication_number`.

### Technical Terms

### Parent Table
The main logical table that users query.

```
patents_partitioned
```

It does **not** store data directly.

---

### HASH Partitioning

Rows are distributed using PostgreSQL's internal hash function.

Example

```
Hash(publication_number)
        ↓
 MOD(hash,4)

0 → p0
1 → p1
2 → p2
3 → p3
```

This keeps data evenly distributed.

---

### MODULUS

Defines the total number of partitions.

```
MODULUS 4
```

means

```
Total partitions = 4
```

---

### REMAINDER

Determines which hash bucket belongs to each partition.

```
REMAINDER 0 → p0
REMAINDER 1 → p1
REMAINDER 2 → p2
REMAINDER 3 → p3
```

---

### Partition Routing

Whenever a row is inserted,

```
publication_number
        ↓
Hash Calculation
        ↓
Modulo Operation
        ↓
Correct Partition
```

PostgreSQL automatically routes the row to the correct partition.

---

## Why HASH Partitioning?

Advantages

- Better distribution of data
- Prevents very large single tables
- Improves equality searches
- Better parallel processing
- Reduces table size per partition

---

## Screenshot

> **Add Screenshot Here**
>
> *(Partition table creation output / `\d+ patents.patents_partitioned`)*

---

# 2. Bulk Import Data

## Query

```sql
COPY patents.patents_partitioned
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

---

## Technical Explanation

The `COPY` command performs a **high-speed bulk import** from a CSV file into PostgreSQL.

Although the command targets the parent table, PostgreSQL automatically inserts each row into the appropriate partition based on the hash of `publication_number`.

### Technical Terms

### COPY

Fastest method to import large datasets.

Unlike multiple INSERT statements, COPY writes rows in bulk, reducing overhead.

---

### FORMAT CSV

Specifies that the input file is in CSV format.

---

### HEADER TRUE

Skips the first line because it contains column names.

---

### QUOTE

Defines the character used to enclose text values.

```
"Patent Title"
```

---

### ESCAPE

Defines how embedded quote characters are handled.

Example

```
"John ""Smith"""
```

---

### Automatic Partition Routing

Every imported row is automatically routed.

```
CSV File
      ↓
Parent Table
      ↓
Hash Function
      ↓
Partition Selection
      ↓
p0 / p1 / p2 / p3
```

No manual partition selection is required.

---

## Screenshot

> **Add Screenshot Here**
>
> *(COPY command output showing number of imported rows)*

---

# 3. Publication Number Lookup (Equality Search)

## Query

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM patents.patents_partitioned
WHERE publication_number = 'US-4081864-A';
```

---

## Technical Explanation

This query searches for a single patent using an exact match on the partition key.

Because the filter uses the partition key, PostgreSQL performs **Partition Pruning**, scanning only the partition that could contain the matching row instead of scanning all partitions.

### Technical Terms

### EXPLAIN

Displays the execution plan chosen by PostgreSQL.

---

### ANALYZE

Executes the query and reports actual execution statistics.

Examples include:

- Actual execution time
- Number of rows processed
- Loop count

---

### BUFFERS

Shows how PostgreSQL accessed data pages.

Examples

- Shared Hit
- Shared Read
- Temp Read
- Temp Write

This helps determine whether the query used cached data or disk I/O.

---

### Equality Search

```
publication_number = value
```

Performs an exact lookup.

---

### Partition Pruning

Since `publication_number` is the partition key, PostgreSQL identifies the correct partition before execution.

Instead of scanning all partitions,

```
p0
p1
p2
p3
```

only one partition is accessed.

This reduces CPU usage and execution time.

---

## Expected Observation

- Partition pruning occurs
- Only one partition is scanned
- Faster execution compared to scanning all partitions

---

## Screenshot

> **Add Screenshot Here**
>
> *(Execution plan showing Partition Pruning)*

---

# 4. Prefix Search on Publication Number

## Query

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM patents.patents_partitioned
WHERE publication_number LIKE 'US-40818%';
```

---

## Technical Explanation

This query searches for all publication numbers beginning with `US-40818`.

Although the filter references the partition key, PostgreSQL generally **cannot determine the target partition** from a LIKE pattern because the partitioning is based on the hash value of the complete string.

As a result, all partitions are scanned.

### Technical Terms

### LIKE

Pattern matching operator.

```
%
```

matches any number of characters.

Example

```
US-40818%
```

matches

```
US-4081801
US-4081864
US-4081899
```

---

### Hash Partition Limitation

Hash partitioning supports efficient equality lookups.

Pattern searches such as LIKE do not reveal the final hash value, so PostgreSQL must search every partition.

---

## Expected Observation

- All partitions scanned
- No partition pruning
- More buffer reads
- Higher execution time than equality search

---

## Screenshot

> **Add Screenshot Here**
>
> *(Execution plan for LIKE search)*

---

# 5. Search by Title

## Query

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM patents.patents_partitioned
WHERE title ILIKE '%helmet%';
```

---

## Technical Explanation

This query performs a case-insensitive search for the word **helmet** anywhere within the title.

Since `title` is **not** the partition key, PostgreSQL must search every partition.

### Technical Terms

### ILIKE

Case-insensitive version of LIKE.

Example

Matches

```
Helmet
helmet
HELMET
HeLmEt
```

---

### Full Table Scan

Without a suitable index, PostgreSQL reads every row in every partition.

---

### Sequential Scan

Rows are read sequentially from beginning to end.

This is efficient when most rows need to be examined.

---

## Expected Observation

- Sequential Scan
- All partitions scanned
- No partition pruning
- Higher buffer usage

---

## Screenshot

> **Add Screenshot Here**
>
> *(Execution plan for Title search)*

---

# 6. Search in Abstract

## Query

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT publication_number, title
FROM patents.patents_partitioned
WHERE abstract ILIKE '%water%';
```

---

## Technical Explanation

This query searches the abstract for the word **water** and returns only the publication number and title.

Because the filter is on `abstract` (which is not the partition key), PostgreSQL scans every partition.

### Technical Terms

### Projection

Only selected columns are returned.

```
publication_number
title
```

instead of all columns.

Reducing the number of returned columns decreases network transfer and memory usage.

---

### Case-Insensitive Pattern Search

```
ILIKE '%water%'
```

Matches

```
Water
WATER
waterproof
underwater
```

---

### Sequential Scan

Without a supporting index, PostgreSQL reads every row to evaluate the search condition.

---

## Expected Observation

- Sequential Scan
- All partitions scanned
- Higher execution time
- Higher buffer usage
- No partition pruning

---

## Screenshot

> **Add Screenshot Here**
>
> *(Execution plan for Abstract search)*

---
