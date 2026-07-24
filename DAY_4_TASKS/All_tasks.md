# Patent Citation Hierarchy Analysis in PostgreSQL

## Objective

This task demonstrates how to model patent citation relationships and analyze citation hierarchies using PostgreSQL. It covers recursive queries, user-defined functions, views, materialized views, and performance comparison on an existing dataset of **1 million patent records**.

---

# Existing Tables

## patents_training

Contains the master patent information.

| Column             | Description                      |
| ------------------ | -------------------------------- |
| publication_number | Unique patent publication number |
| inventor_name      | Primary inventor                 |
| publication_date   | Patent publication date          |
| title              | Patent title                     |
| abstract           | Patent abstract                  |

---

## patent_inventors

Stores one row per inventor for each patent.

| Column             | Description       |
| ------------------ | ----------------- |
| publication_number | Patent number     |
| inventor_name      | Inventor          |
| inventor_order     | Order of inventor |

---

## patent_inventor_array

Stores inventors as an array.

| Column             | Description        |
| ------------------ | ------------------ |
| publication_number | Patent number      |
| inventors          | Array of inventors |

---

## patent_metadata

Stores additional metadata using JSONB.

| Column             | Description     |
| ------------------ | --------------- |
| publication_number | Patent number   |
| metadata           | Patent metadata |

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
CREATE TABLE patent_citations
(
    citing_publication_number TEXT NOT NULL,
    cited_publication_number  TEXT NOT NULL,

    PRIMARY KEY
    (
        citing_publication_number,
        cited_publication_number
    ),

    FOREIGN KEY (citing_publication_number)
        REFERENCES patents_training(publication_number),

    FOREIGN KEY (cited_publication_number)
        REFERENCES patents_training(publication_number)
);
```

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

    ROW_NUMBER() OVER
    (
        ORDER BY publication_date,
                 publication_number
    ) AS rn

FROM patents_training;
```

---

## Generate Citation Relationships

```sql
INSERT INTO patent_citations

SELECT DISTINCT

    p.publication_number,
    older.publication_number

FROM ordered_patents p

CROSS JOIN LATERAL
(
    SELECT publication_number

    FROM ordered_patents

    WHERE rn < p.rn

    ORDER BY random()

    LIMIT floor(random()*5 + 1)

) older

WHERE p.rn > 5;
```

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
SELECT *

FROM patent_citations

LIMIT 20;
```

Expected output:

| Citing Patent | Cited Patent |
| ------------- | ------------ |
| US0000001001  | US0000000005 |
| US0000001001  | US0000000032 |
| US0000001002  | US0000000044 |

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

    WHERE citing_publication_number = 'US0001234567'

    UNION ALL

    SELECT
        ct.citing_publication_number,
        pc.cited_publication_number,
        ct.depth + 1

    FROM citation_tree ct

    JOIN patent_citations pc

      ON ct.cited_publication_number =
         pc.citing_publication_number
)

SELECT *

FROM citation_tree;
```

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
CREATE OR REPLACE FUNCTION
get_patent_citation_hierarchy
(
    patent_no TEXT
)

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

    UNION ALL

    SELECT
        pc.cited_publication_number,
        h.depth + 1

    FROM hierarchy h

    JOIN patent_citations pc

      ON h.cited_publication_number =
         pc.citing_publication_number
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

FROM get_patent_citation_hierarchy
(
    'US0001234567'
);
```

---

# Step 6 - Use Function for Multiple Patents

## Technical Explanation

`CROSS JOIN LATERAL` executes the function once for every patent.

```sql
SELECT

    p.publication_number,

    c.cited_patent,

    c.depth

FROM patents_training p

CROSS JOIN LATERAL

get_patent_citation_hierarchy
(
    p.publication_number
) c

LIMIT 100;
```

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
        SELECT COUNT(*)

        FROM patent_citations pc

        WHERE pc.citing_publication_number =
              p.publication_number

    ) AS direct_citation_count,

    (
        SELECT COUNT(*)

        FROM get_patent_citation_hierarchy
        (
            p.publication_number
        )

    ) AS total_citation_count,

    (
        SELECT MAX(depth)

        FROM get_patent_citation_hierarchy
        (
            p.publication_number
        )

    ) AS max_depth

FROM patents_training p;
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
CREATE MATERIALIZED VIEW patent_citation_summary_mv

AS

SELECT *

FROM patent_citation_summary;
```

---

## Create Unique Index

Required for concurrent refresh.

```sql
CREATE UNIQUE INDEX idx_mv_patent

ON patent_citation_summary_mv
(
    publication_number
);
```

---

# Step 9 - Performance Comparison

## Direct Query

```sql
EXPLAIN ANALYZE

SELECT

    p.publication_number,

    (
        SELECT COUNT(*)

        FROM patent_citations pc

        WHERE pc.citing_publication_number =
              p.publication_number
    )

FROM patents_training p;
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

VALUES

(
    'US0009999999',
    'US0000000005'
);
```

---

## Query the View

```sql
SELECT *

FROM patent_citation_summary

WHERE publication_number =
'US0009999999';
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

# Expected Outcome

After completing this task, you will have:

* A normalized patent citation model.
* Realistic citation data for 1 million patents.
* Recursive traversal of citation hierarchies.
* A reusable SQL function for hierarchy retrieval.
* Summary reporting through both a View and Materialized View.
* Performance benchmarks comparing direct queries, views, and materialized views.
* A clear understanding of refresh behavior and maintenance for materialized views.
