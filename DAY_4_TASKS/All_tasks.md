# Patent Citation Hierarchy Analysis in PostgreSQL

## Objective

This task demonstrates how to model patent citation relationships and analyze citation hierarchies using PostgreSQL. It covers recursive queries, user-defined functions, views, materialized views, and performance comparison on an existing dataset of **1 million patent records**.

---

# Step 1 - Create Patent Citation Table

Each patent can cite multiple older patents.

## Technical Explanation

This table represents a **many-to-many relationship** between patents.

* One patent may cite many patents.
* One patent may be cited by many future patents.
* Composite Primary Key prevents duplicate citations.
* Foreign Keys guarantee referential integrity.

```sql
CREATE TABLE patent_citations (
  citing_publication_number TEXT NOT NULL, 
  cited_publication_number TEXT NOT NULL, 
  PRIMARY KEY (
    citing_publication_number, cited_publication_number
  ), 
  FOREIGN KEY (citing_publication_number) REFERENCES patent_training(publication_number), 
  FOREIGN KEY (cited_publication_number) REFERENCES patent_training(publication_number)
);

```
<img width="442" height="267" alt="Screenshot 2026-07-24 at 2 08 10 PM" src="https://github.com/user-attachments/assets/e0905b71-db3b-44ae-b11c-065042753122" />


---

## Create Indexes

### Technical Explanation

Recursive queries repeatedly search by both columns.

Creating indexes significantly improves lookup performance.

```sql
CREATE INDEX idx_citing
ON patent_citations(citing_publication_number);

CREATE INDEX idx_cited
ON patent_citations(cited_publication_number);
```
<img width="354" height="174" alt="Screenshot 2026-07-24 at 2 08 16 PM" src="https://github.com/user-attachments/assets/ba19504d-372c-436a-8309-31ebf67319bb" />

---

# Step 2 - Generate Citation Data

Each patent should cite **1–5 older patents**.

## Technical Explanation

To avoid invalid future citations:

1. Sort patents by publication date.
2. Assign each patent a row number.
3. Allow citations only to patents with a smaller row number.

This guarantees that every citation points to an older patent.

---

## Create Ordered Patent List

```sql
CREATE TEMP TABLE ordered_patents AS 
SELECT 
  publication_number, 
  publication_date, 
  ROW_NUMBER() OVER (
    ORDER BY 
      publication_date, 
      publication_number
  ) AS rn 
FROM patent_training;

```
<img width="322" height="204" alt="Screenshot 2026-07-24 at 4 06 44 PM" src="https://github.com/user-attachments/assets/f8d7d923-92c3-4230-becd-64d1e4ca6b93" />

# Create Index 

``` sql

CREATE INDEX idx_ordered_patents_rn
ON ordered_patents(rn);

```
<img width="295" height="101" alt="Screenshot 2026-07-24 at 4 26 31 PM" src="https://github.com/user-attachments/assets/759aa425-0ee0-4d95-a727-077836e32540" />

---

## Generate Citation Relationships

```sql
INSERT INTO patent_citations
(
    citing_publication_number,
    cited_publication_number
)
SELECT
    p.publication_number,
    c.publication_number
FROM ordered_patents p
CROSS JOIN LATERAL
(
    SELECT DISTINCT
        (
            GREATEST(1, p.rn - 1000) +
            floor(random() * LEAST(1000, p.rn - 1))
        )::bigint AS random_rn
    FROM generate_series(1,10)
    WHERE p.rn > 1
    LIMIT (floor(random() * 3) + 1)::int
) r
JOIN ordered_patents c
  ON c.rn = r.random_rn
WHERE p.rn > 1;

```
<img width="443" height="485" alt="Screenshot 2026-07-24 at 6 21 12 PM" src="https://github.com/user-attachments/assets/36e3e6d4-9769-4a27-8004-562335feda8a" />


---

## Why CROSS JOIN LATERAL?

`LATERAL` executes the inner query once for every row in the outer query.

For every patent:

* Pick 1–5 random older patents.
* Insert those citations.

Without `LATERAL`, the same random patents would be selected for every row.

---

# Step 3 - Verify Citation Data

```sql
SELECT * FROM patent_citations LIMIT 20;
```
<img width="452" height="452" alt="Screenshot 2026-07-24 at 6 23 42 PM" src="https://github.com/user-attachments/assets/f2b61f0e-1954-4e4d-9542-a150806c7d64" />


---

# Step 4 - Retrieve Citation Hierarchy

Example hierarchy:

```
Patent A
│
├── Patent B
│      │
│      └── Patent D
│
└── Patent C
       │
       └── Patent E
```

Expected output

| Patent | Depth |
| ------ | ----: |
| B      |     1 |
| C      |     1 |
| D      |     2 |
| E      |     2 |

---

## Technical Explanation

A patent can cite another patent, which itself cites older patents.

This forms a **recursive hierarchy (tree/graph)**.

PostgreSQL solves this using a **Recursive CTE**.

---

## Recursive Query

```sql
WITH RECURSIVE citation_tree AS
(
    SELECT
        citing_publication_number,
        cited_publication_number,
        1 AS depth
    FROM patent_citations
    WHERE citing_publication_number = 'US0000054181'

    UNION

    SELECT
        ct.citing_publication_number,
        pc.cited_publication_number,
        ct.depth + 1
    FROM citation_tree ct
    JOIN patent_citations pc
      ON pc.citing_publication_number = ct.cited_publication_number
    WHERE ct.depth < 10
)
SELECT *
FROM citation_tree;

```

![Uploading Screenshot 2026-07-27 at 12.09.42 PM.png…]()

---

## How Recursive CTE Works

### Anchor Query

Returns direct citations.

```
Patent A
    ↓
Patent B
Patent C
```

Depth = 1

---

### Recursive Query

Uses previous results to continue searching.

```
Patent B
    ↓
Patent D

Patent C
    ↓
Patent E
```

Depth = 2

The recursion stops automatically when no more cited patents are found.

---

# Step 5 - Create Database Function

## Technical Explanation

Instead of writing the recursive query repeatedly, encapsulate it inside a reusable SQL function.

Benefits:

* Reusable
* Easier maintenance
* Cleaner SQL
* Can be called for one or many patents

---

```sql
CREATE OR REPLACE FUNCTION get_patent_citation_hierarchy(patent_no TEXT)
RETURNS TABLE
(
    cited_patent TEXT,
    depth INT
)
LANGUAGE SQL
AS
$$
WITH RECURSIVE hierarchy AS
(
    SELECT
        cited_publication_number,
        1 AS depth
    FROM patent_citations
    WHERE citing_publication_number = patent_no

    UNION

    SELECT
        pc.cited_publication_number,
        h.depth + 1
    FROM hierarchy h
    JOIN patent_citations pc
      ON pc.citing_publication_number = h.cited_publication_number
    WHERE h.depth < 10
)
SELECT
    cited_publication_number,
    depth
FROM hierarchy;
$$;
```

---

## Execute Function

```sql
SELECT *
FROM get_patent_citation_hierarchy('US0001234567');
```
<img width="439" height="122" alt="Screenshot 2026-07-27 at 12 11 44 PM" src="https://github.com/user-attachments/assets/85cc64e4-95bd-4593-9b08-da2fb3b1abb4" />

---

# Step 6 - Use Function for Multiple Patents

## Technical Explanation

`CROSS JOIN LATERAL` executes the function once for every patent.

```sql
SELECT
    p.publication_number,
    c.cited_patent,
    c.depth
FROM
(
    SELECT publication_number
    FROM patent_training
    LIMIT 3
) p
CROSS JOIN LATERAL
(
    SELECT *
    FROM get_patent_citation_hierarchy(p.publication_number)
    LIMIT 10
) c;

```
<img width="510" height="875" alt="Screenshot 2026-07-27 at 12 30 19 PM" src="https://github.com/user-attachments/assets/7511a19a-794c-4163-aeac-b0b1d3515fc8" />

---

# Step 7 - Create Summary View

The view should contain:

* Publication Number
* Inventor Name
* Publication Date
* Direct Citation Count
* Total Citation Count
* Maximum Citation Depth

---

## Technical Explanation

A View stores only the SQL definition.

Whenever queried, PostgreSQL executes the underlying query again.

Advantages:

* Always up-to-date
* No extra storage

Disadvantages:

* Slower for expensive recursive queries

---

```sql
CREATE VIEW patent_citation_summary AS 
SELECT 
  p.publication_number, 
  p.inventor_name, 
  p.publication_date, 
  (
    SELECT 
      COUNT(*) 
    FROM 
      patent_citations pc 
    WHERE 
      pc.citing_publication_number = p.publication_number
  ) AS direct_citation_count, 
  (
    SELECT 
      COUNT(*) 
    FROM 
      get_patent_citation_hierarchy (p.publication_number)
  ) AS total_citation_count, 
  (
    SELECT 
      MAX(depth) 
    FROM 
      get_patent_citation_hierarchy (p.publication_number)
  ) AS max_depth 
FROM 
  patent_training p;

```

---

# Step 8 - Create Materialized View

## Technical Explanation

Unlike a View, a Materialized View stores the query result physically on disk.

Advantages

* Much faster reads
* Suitable for reporting
* Avoids repeated recursive computation

Disadvantages

* Data becomes stale until refreshed

---

```sql
CREATE MATERIALIZED VIEW patent_citation_summary_mv AS 
SELECT * 
FROM patent_citation_summary;

```

---

## Create Unique Index

Required for concurrent refresh.

```sql
CREATE UNIQUE INDEX idx_mv_patent
ON patent_citation_summary_mv (publication_number);

```

---

# Step 9 - Performance Comparison

## Direct Query

```sql
EXPLAIN ANALYZE 
SELECT 
  p.publication_number, 
  (
    SELECT 
      COUNT(*) 
    FROM patent_citations pc 
    WHERE 
      pc.citing_publication_number = p.publication_number
  ) 
FROM patent_training p;

```

---

## View

```sql
EXPLAIN ANALYZE

SELECT *
FROM patent_citation_summary
LIMIT 100;
```

---

## Materialized View

```sql
EXPLAIN ANALYZE

SELECT *
FROM patent_citation_summary_mv
LIMIT 100;
```

---

## Expected Performance

| Method            | Query Execution         | Storage | Expected Speed          |
| ----------------- | ----------------------- | ------- | ----------------------- |
| Direct Query      | Executes SQL every time | No      | Slowest                 |
| View              | Executes SQL every time | No      | Similar to Direct Query |
| Materialized View | Reads precomputed data  | Yes     | Fastest                 |

---

# Step 10 - Test View vs Materialized View

Insert a new citation.

```sql
INSERT INTO patent_citations
VALUES ( 'US0009999999',
  'US0000000005' );

```

---

## Query the View

```sql
SELECT *
FROM patent_citation_summary
WHERE publication_number = 'US0009999999';

```

Result:

* Shows the new citation immediately.

---

## Query the Materialized View

```sql
SELECT *
FROM patent_citation_summary_mv
WHERE publication_number =
'US0009999999';
```

Result:

* Still returns the old snapshot (or no row if it wasn't present when the materialized view was created).

---

# Refresh Materialized View

```sql
REFRESH MATERIALIZED VIEW patent_citation_summary_mv;
```

---

## Concurrent Refresh

```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY
patent_citation_summary_mv;
```

### Technical Explanation

A normal refresh locks the materialized view while rebuilding it.

A concurrent refresh:

* Keeps the materialized view readable during refresh.
* Requires a UNIQUE INDEX on the materialized view.
* Is recommended for production environments.

---

# Summary of PostgreSQL Concepts Covered

| Concept                                | Purpose                                          |
| -------------------------------------- | ------------------------------------------------ |
| Foreign Keys                           | Maintain valid citation relationships            |
| Composite Primary Key                  | Prevent duplicate citations                      |
| B-tree Index                           | Speed up citation lookups                        |
| CROSS JOIN LATERAL                     | Execute a subquery/function for each row         |
| Recursive CTE                          | Traverse citation hierarchies of unlimited depth |
| SQL Function                           | Reuse recursive hierarchy logic                  |
| View                                   | Dynamic query executed on demand                 |
| Materialized View                      | Persisted query results for faster reads         |
| EXPLAIN ANALYZE                        | Measure execution performance                    |
| REFRESH MATERIALIZED VIEW              | Synchronize materialized data with base tables   |
| REFRESH MATERIALIZED VIEW CONCURRENTLY | Refresh without blocking read operations         |

---
