# Phase 1 - Patent Citation Table Design and Citation Data Generation

## Objective

The objective of this phase is to design a normalized patent citation table that models the relationship between patents where one patent cites another previously published patent.

The citation table will be used throughout the remaining phases to:

- Traverse citation hierarchies using recursive queries
- Calculate direct and indirect citation counts
- Build summary views and materialized views
- Benchmark recursive query performance on one million patent records

- Each patent cites between **1 and 5** older patents.
- A patent cannot cite itself.
- A patent cannot cite a newer patent.
- Duplicate citation relationships are not allowed.
- The resulting citation graph must be acyclic (Directed Acyclic Graph).

---

# Existing Dataset

The following tables are already available.

| Table | Description |
|---------|------------|
| patent_training | Master patent information |
| patent_inventors | Patent to inventor mapping |
| patent_inventor_array | Array representation of inventors |
| patent_metadata | Additional patent metadata |

The primary key used throughout the implementation is

```text
publication_number
```

---

# Citation Model

A patent citation represents a directed relationship.

```text
Patent A
    │
    │ cites
    ▼
Patent B
```

Meaning

- Patent A references Patent B.
- Patent B must always be published earlier than Patent A.
- One patent can cite multiple patents.
- A patent can be cited by multiple newer patents.

Example

```text
US0000100000
      │
      ├────────► US0000091200
      │
      ├────────► US0000085431
      │
      └────────► US0000078543
```

This forms a Directed Acyclic Graph (DAG), which is suitable for recursive traversal.

---

# Table Design

Create a table named `patent_citations`.

Each record represents one citation relationship.

## Table Structure

| Column | Description |
|---------|-------------|
| citing_publication_number | Patent that references another patent |
| cited_publication_number | Patent being referenced |

---

# Create Table

```sql
DROP TABLE IF EXISTS patent_citations;

CREATE TABLE patent_citations
(
    citing_publication_number TEXT NOT NULL,
    cited_publication_number  TEXT NOT NULL,

    CONSTRAINT pk_patent_citations
    PRIMARY KEY
    (
        citing_publication_number,
        cited_publication_number
    ),

    CONSTRAINT fk_citing_patent
    FOREIGN KEY (citing_publication_number)
    REFERENCES patent_training(publication_number),

    CONSTRAINT fk_cited_patent
    FOREIGN KEY (cited_publication_number)
    REFERENCES patent_training(publication_number),

    CONSTRAINT chk_no_self_reference
    CHECK
    (
        citing_publication_number <>
        cited_publication_number
    )
);
```

---

## Explanation

### Composite Primary Key

```text
(citing_publication_number,
 cited_publication_number)
```

Ensures that

- Duplicate citation relationships cannot exist.
- A patent cannot cite the same patent multiple times.

Example

Allowed

```text
US10 → US08
US10 → US07
US10 → US06
```

Not Allowed

```text
US10 → US08
US10 → US08
```

---

### Foreign Keys

Both columns reference the master patent table.

This guarantees

- Every citing patent exists.
- Every cited patent exists.
- No orphan citation records.

---

### Check Constraint

```sql
CHECK (
    citing_publication_number <>
    cited_publication_number
)
```

Prevents invalid relationships.

Example

Invalid

```text
US0001234567
      │
      ▼
US0001234567
```

---

# Screenshot

> **Insert Screenshot:** Table created successfully.

```
+-------------------------------------------------------+
| CREATE TABLE patent_citations                         |
+-------------------------------------------------------+
```

---

# Create Indexes

Since recursive traversal starts from the citing patent and often searches by the cited patent as well, indexes are required on both columns.

```sql
CREATE INDEX idx_patent_citations_citing
ON patent_citations(citing_publication_number);

CREATE INDEX idx_patent_citations_cited
ON patent_citations(cited_publication_number);
```

---

## Why Two Indexes?

### Index 1

```sql
idx_patent_citations_citing
```

Used for queries like

```sql
SELECT *
FROM patent_citations
WHERE citing_publication_number = 'US0001234567';
```

---

### Index 2

```sql
idx_patent_citations_cited
```

Used for queries like

```sql
SELECT *
FROM patent_citations
WHERE cited_publication_number = 'US0001234567';
```

This is useful for identifying all patents that reference a given patent.

---

# Verify Table Structure

```sql
\d+ patent_citations
```

Expected output

```text
Table "public.patent_citations"

Column
-----------------------------------------
citing_publication_number
cited_publication_number

Indexes

pk_patent_citations
idx_patent_citations_citing
idx_patent_citations_cited

Foreign-key constraints

fk_citing_patent
fk_cited_patent
```

---

# Screenshot

> **Insert Screenshot:** `\d+ patent_citations`

---

# Verify Indexes

```sql
SELECT
    indexname,
    indexdef
FROM pg_indexes
WHERE tablename = 'patent_citations';
```

Expected Result

```text
pk_patent_citations
idx_patent_citations_citing
idx_patent_citations_cited
```

---

# Screenshot

> **Insert Screenshot:** Index verification output.

---

# Verify Constraints

```sql
SELECT
    conname,
    contype
FROM pg_constraint
WHERE conrelid = 'patent_citations'::regclass;
```

Expected Output

```text
pk_patent_citations
fk_citing_patent
fk_cited_patent
chk_no_self_reference
```

---

# Screenshot

> **Insert Screenshot:** Constraint verification.

---

# Data Generation Strategy

Since patents can only cite patents that were published earlier, the generation process uses the publication date to identify eligible citations.

For each patent:

1. Find all patents published before it.
2. Randomly select up to five older patents.
3. Insert the citation relationship into `patent_citations`.

This approach naturally prevents cyclic dependencies.

---

# Generate Citation Relationships

```sql
INSERT INTO patent_citations
(
    citing_publication_number,
    cited_publication_number
)
SELECT
    p.publication_number,
    c.publication_number
FROM patent_training p
CROSS JOIN LATERAL
(
    SELECT publication_number
    FROM patent_training older
    WHERE older.publication_date < p.publication_date
    ORDER BY random()
    LIMIT (floor(random() * 5) + 1)::INT
) c
ON CONFLICT DO NOTHING;
```

---

## Explanation

### Step 1

Iterate through every patent.

```sql
FROM patent_training p
```

---

### Step 2

For the current patent, retrieve only older patents.

```sql
WHERE older.publication_date < p.publication_date
```

Example

```text
Patent A
Publication Date

2020-08-01
```

Eligible citations

```text
2019
2018
2015
2010
```

Not Eligible

```text
2021
2022
2023
```

---

### Step 3

Randomize the candidates.

```sql
ORDER BY random()
```

---

### Step 4

Select between one and five patents.

```sql
LIMIT
(floor(random()*5)+1)
```

Possible values

```text
1
2
3
4
5
```

---

### Step 5

Insert into citation table.

```text
Patent A
      │
      ├────────► Patent B
      ├────────► Patent C
      └────────► Patent D
```

---

# Screenshot

> **Insert Screenshot:** Citation generation completed successfully.

---

# Verify Total Citation Records

```sql
SELECT COUNT(*)
FROM patent_citations;
```

Expected Result

```text
Approximately

3,000,000

to

5,000,000

citation records
```

(The exact count depends on random generation.)

---

# Screenshot

> **Insert Screenshot:** Total citation count.

---

# Verify Sample Records

```sql
SELECT *
FROM patent_citations
LIMIT 20;
```

Example

```text
citing_publication_number   cited_publication_number

US0000100245               US0000087411
US0000100245               US0000074122
US0000100245               US0000068532

US0000100246               US0000098422
US0000100246               US0000090011
```

---

# Screenshot

> **Insert Screenshot:** Sample citation records.

---

# Verify No Self-Citations

```sql
SELECT COUNT(*)
FROM patent_citations
WHERE citing_publication_number =
      cited_publication_number;
```

Expected Result

```text
0
```

---

# Screenshot

> **Insert Screenshot:** Self-citation verification.

---

# Verify Duplicate Relationships

```sql
SELECT
    citing_publication_number,
    cited_publication_number,
    COUNT(*)
FROM patent_citations
GROUP BY
    citing_publication_number,
    cited_publication_number
HAVING COUNT(*) > 1;
```

Expected Result

```text
0 rows
```

---

# Screenshot

> **Insert Screenshot:** Duplicate verification.

---

# Verify Older Patent Rule

```sql
SELECT
    pc.citing_publication_number,
    p1.publication_date AS citing_date,
    pc.cited_publication_number,
    p2.publication_date AS cited_date
FROM patent_citations pc
JOIN patent_training p1
ON pc.citing_publication_number =
   p1.publication_number
JOIN patent_training p2
ON pc.cited_publication_number =
   p2.publication_number
WHERE p2.publication_date >=
      p1.publication_date
LIMIT 20;
```

Expected Result

```text
0 rows
```

This confirms that every cited patent is older than the citing patent.

---

# Screenshot

> **Insert Screenshot:** Older patent validation.

---

# Citation Distribution

Calculate the number of citations created per patent.

```sql
SELECT
    COUNT(*) AS citation_count,
    COUNT(DISTINCT citing_publication_number) AS patents
FROM patent_citations;
```

---

# Distribution by Citation Count

```sql
SELECT
    citation_count,
    COUNT(*) AS patents
FROM
(
    SELECT
        citing_publication_number,
        COUNT(*) AS citation_count
    FROM patent_citations
    GROUP BY citing_publication_number
) t
GROUP BY citation_count
ORDER BY citation_count;
```

Example

```text
Citation Count

1      198452
2      201118
3      199875
4      200643
5      199912
```

---

# Screenshot

> **Insert Screenshot:** Citation distribution.

---

# Analyze Table Size

```sql
SELECT
    pg_size_pretty(
        pg_relation_size('patent_citations')
    ) AS table_size;
```

---

# Analyze Index Size

```sql
SELECT
    indexrelname,
    pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes
WHERE relname='patent_citations';
```

---

# Screenshot

> **Insert Screenshot:** Table size and index size.

---

# Analyze Query Performance

Lookup citations for one patent.

```sql
EXPLAIN ANALYZE
SELECT *
FROM patent_citations
WHERE citing_publication_number =
'US0000058066';
```

Expected Plan

```text
Index Scan

using

idx_patent_citations_citing
```

---

# Screenshot

> **Insert Screenshot:** EXPLAIN ANALYZE output.

---
