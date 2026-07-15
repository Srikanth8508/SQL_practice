# PostgreSQL EXPLAIN (ANALYZE, BUFFERS) - Complete Guide

## Objective

`EXPLAIN (ANALYZE, BUFFERS)` is one of the most important PostgreSQL commands used to understand:

- How PostgreSQL executes a query.
- How much time the query takes.
- Whether data is read from memory or disk.
- Which execution plan PostgreSQL chooses.

It is primarily used for **query performance tuning and troubleshooting**.

---

# Syntax

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...
```

Example:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM patents.patents_logged
WHERE publication_number = 'US-4081864-A';
```

---

# Difference Between EXPLAIN and EXPLAIN ANALYZE

## EXPLAIN

```sql
EXPLAIN
SELECT ...
```

### What it does

- Does **NOT** execute the query.
- Shows PostgreSQL's estimated execution plan.
- Uses planner estimates.

Used when you only want to know which execution plan PostgreSQL will choose.

---

## EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE
SELECT ...
```

### What it does

- Executes the query.
- Measures actual execution time.
- Shows actual rows processed.
- Compares estimates with reality.

Used when measuring real query performance.

---

## EXPLAIN (ANALYZE, BUFFERS)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...
```

This provides:

- Execution Plan
- Actual Execution Time
- Actual Rows
- Memory Usage
- Disk Reads
- Buffer Hits

This is the command most DBAs use for performance analysis.

---

# Query Used

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM patents.patents_logged
WHERE publication_number='US-4081864-A';
```

---

# Output

```text
Gather
(cost=1000.00..1145007.48 rows=1 width=870)
(actual time=46.197..6307.120 rows=1 loops=1)

Workers Planned: 2
Workers Launched: 2

Buffers:
shared hit=3496
read=1091269

-> Parallel Seq Scan on patents_logged
(actual time=4184.364..6269.910 rows=0 loops=3)

Filter:
(publication_number='US-4081864-A')

Rows Removed by Filter: 3152723

Planning Time: 0.753 ms

Execution Time: 6310.169 ms
```

---

# Understanding Each Section

---

# 1. Gather

```text
Gather
```

## Meaning

PostgreSQL decided to execute the query using parallel processing.

Instead of one process searching the table, multiple processes search different parts of the table simultaneously.

Diagram

```
                Gather
                   │
      ┌────────────┼────────────┐
      │            │            │
 Main Process   Worker 1    Worker 2
```

---

# 2. Workers Planned

```text
Workers Planned: 2
Workers Launched: 2
```

## Meaning

PostgreSQL planned to create:

- 2 parallel workers

Along with the main PostgreSQL backend process.

Total processes working:

```
Main Backend Process
Worker 1
Worker 2

Total = 3 processes
```

Note:

The main backend process is **not counted** in "Workers Planned".

---

# 3. Parallel Sequential Scan

```text
Parallel Seq Scan
```

## Meaning

PostgreSQL reads the entire table.

Since no index exists on the searched column, PostgreSQL cannot jump directly to the required row.

Instead, it checks every row.

Example

```
Row 1
Row 2
Row 3
...
Row 9,000,000
```

---

## How Parallel Scan Works

Many beginners think workers process rows like this:

```
Worker1 → Row 1

Worker2 → Row 2

Worker3 → Row 3
```

This is **NOT** how PostgreSQL works.

Instead, PostgreSQL divides the table into **pages (blocks)**.

Example

```
Page 1
Rows 1-100

Page 2
Rows 101-200

Page 3
Rows 201-300
```

Workers receive pages.

Example

```
Worker 1

Page 1
Page 2
Page 3

-------------------

Worker 2

Page 4
Page 5
Page 6

-------------------

Main Process

Page 7
Page 8
Page 9
```

When a worker finishes, PostgreSQL assigns more pages dynamically.

---

# 4. Filter

```text
Filter:
publication_number='US-4081864-A'
```

Every row is checked.

Example

```
Row 1

Match?

No

↓

Row 2

Match?

No

↓

Row 5498

Match?

YES

↓

Store Result
```

Even after finding the row, PostgreSQL continues scanning because there may be more matching rows.

Without an index, it cannot stop early.

---

# 5. Rows Removed by Filter

```text
Rows Removed by Filter: 3152723
```

This means each worker rejected approximately

```
3,152,723 rows
```

Since three processes participated,

approximately

```
3,152,723 × 3

≈ 9.45 Million rows
```

were checked.

Only one row matched.

---

# 6. Buffers

```
Buffers:

shared hit=3496

read=1091269
```

Buffers show where PostgreSQL obtained the data.

---

## shared hit

```
shared hit=3496
```

Meaning

Data already existed inside PostgreSQL's shared memory.

No disk read required.

Very fast.

---

## read

```
read=1091269
```

Meaning

PostgreSQL had to read

```
1,091,269
```

database pages from disk.

Each PostgreSQL page is usually

```
8 KB
```

Approximate data read

```
1,091,269 × 8 KB

≈ 8.3 GB
```

Reading from disk is much slower than reading from RAM.

---

# 7. rows=1

```
rows=1
```

This is PostgreSQL's estimate before execution.

It predicted

```
One row will match.
```

The estimate was correct.

---

# 8. width=870

```
width=870
```

Meaning

Average size of one returned row is approximately

```
870 bytes
```

Since the query uses

```sql
SELECT *
```

every column contributes to the row size.

---

# 9. actual time

Example

```
actual time=4184.364..6269.910
```

This contains two numbers.

```
4184 ms
```

Time until the first row was produced.

```
6269 ms
```

Time until the node completed.

Think of it as

```
Start searching

↓

Found first result

↓

Finished searching entire table
```

---

# 10. Execution Time

```
Execution Time: 6310 ms
```

Execution Time represents the total time for the entire query.

It includes

- Planning
- Parallel Processing
- Scanning
- Filtering
- Gathering results
- Returning results

This is the total query runtime.

---

# 11. Planning Time

```
Planning Time: 0.753 ms
```

Before executing the query, PostgreSQL decides

- Which indexes to use
- Whether to use parallel processing
- Which scan method to use

This decision-making took

```
0.753 milliseconds
```

---

# Why Was This Query Slow?

Because there is **no index** on

```
publication_number
```

PostgreSQL had no shortcut.

Instead it searched the entire table.

Process

```
Start

↓

Read every page

↓

Check every row

↓

Compare publication_number

↓

Discard non-matching rows

↓

Return matching row

↓

Finish scan
```

Total execution time

```
6.31 Seconds
```

---

# What Happens If an Index Exists?

Example

```sql
CREATE INDEX idx_publication_number
ON patents.patents_logged(publication_number);
```

Now PostgreSQL can

```
Use Index

↓

Find Row Location

↓

Jump Directly

↓

Read One Row

↓

Return Result
```

Instead of scanning millions of rows.

Execution time typically reduces from

```
Seconds

↓

Milliseconds
```

---

# Key Terms Summary

| Term | Meaning |
|------|---------|
| EXPLAIN | Shows estimated execution plan without running the query |
| ANALYZE | Executes the query and measures actual performance |
| BUFFERS | Shows whether data came from RAM or Disk |
| Gather | Collects results from parallel processes |
| Workers Planned | Number of helper worker processes PostgreSQL intended to start |
| Parallel Seq Scan | Reads the entire table using multiple processes |
| Filter | Condition applied to each row |
| Rows Removed by Filter | Rows rejected because they didn't satisfy the condition |
| shared hit | Data served from PostgreSQL shared memory |
| read | Data fetched from disk |
| rows | Estimated number of rows returned |
| width | Estimated average row size in bytes |
| actual time | Time until first row and time until node completion |
| Planning Time | Time spent creating the execution plan |
| Execution Time | Total time taken by the query |

---
