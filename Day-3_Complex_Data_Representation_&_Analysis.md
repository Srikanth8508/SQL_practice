# PostgreSQL Arrays and JSONB - Technical Documentation

The following tasks demonstrate how PostgreSQL supports complex data types such as **ARRAY** and **JSONB** for storing, querying, and optimizing semi-structured and multi-valued data.

| Task | Requirement | PostgreSQL Feature |
|------|-------------|--------------------|
| **Task 1** | Create a representation where each patent contains all its associated inventor names together in a single field. | **ARRAY** (`ARRAY_AGG()`) |
| **Task 2** | Find patents where a specific inventor is associated with the patent. | **ARRAY Search** (`@>`, `ANY`) |
| **Task 3** | Find patents where at least one inventor from a given list is associated with the patent. | **ARRAY Overlap Operator** (`&&`) |
| **Task 4** | Find two patents that have at least one inventor in common. | **ARRAY Intersection** (`&&` between two arrays) |
| **Task 5** | Convert the grouped inventor information back into individual rows and verify that the result matches the original inventor-to-patent relationships. | **UNNEST()** |
| **Task 6** | Store structured patent metadata containing fields such as country, category, status, technology area, and filing date within a single column. | **JSONB**, `jsonb_build_object()` |
| **Task 7** | Query and filter patents based on individual attributes stored within the JSONB metadata. | **JSONB Operators** (`->`, `->>`, `@>`, `?`) |
| **Task 8** | Modify individual attributes inside a JSONB document without replacing the entire object. | **JSONB Update** (`jsonb_set()`) |
| **Task 9** | Generate a structured JSON document for each patent containing patent details and all associated inventors. | **JSON/JSONB Generation** (`jsonb_agg()`, `jsonb_build_object()`) |
| **Task 10** | Generate a hierarchical JSON structure showing **Year → Patents → Patent Details → Inventors**. | **Nested JSON** (`jsonb_build_object()`, `jsonb_agg()`) |
| **Task 11** | Analyze query performance before and after indexing, and compare execution plans. | **GIN Indexes**, **Expression Indexes**, **EXPLAIN ANALYZE** |
| **Task 12** | Research and demonstrate additional PostgreSQL operations supported for ARRAY and JSONB data types. | Advanced **ARRAY** and **JSONB** Functions & Operators |

---

# Dataset Preparation

## Step 1 - Create Patent Inventor Table

```sql
CREATE TABLE patents.patent_inventors
(
    publication_number TEXT,
    inventor_name      TEXT,
    inventor_order     SMALLINT,
    PRIMARY KEY (publication_number, inventor_order)
);
```

### Screenshot

> **Add Screenshot Here**

---

## Step 2 - Populate Patent Inventors

Each patent receives multiple inventors randomly selected from the inventor master table.

```sql
INSERT INTO patents.patent_inventors
(
    publication_number,
    inventor_name,
    inventor_order
)
SELECT
    p.publication_number,
    x.inventor_name,
    x.rn
FROM patents_training p
CROSS JOIN LATERAL
(
    SELECT
        inventor_name,
        ROW_NUMBER() OVER () + 1 AS rn
    FROM
    (
        SELECT inventor_name
        FROM inventor_master
        WHERE inventor_name <> p.inventor_name
        ORDER BY random()
        LIMIT (floor(random()*4)+1)::int
    ) t
) x;
```

### Screenshot

> **Add Screenshot Here**

---

# Task 1 - Create ARRAY Table

Convert inventor rows into a single ARRAY.

```sql
CREATE TABLE patents.patent_inventor_array AS

SELECT
    publication_number,
    ARRAY_AGG(inventor_name ORDER BY inventor_name) AS inventors
FROM patents.patent_inventors
GROUP BY publication_number;
```

### Screenshot

> **Add Screenshot Here**

---

# Performance Test Before Index

```sql
EXPLAIN ANALYZE

SELECT *
FROM patents.patent_inventor_array
WHERE inventors @> ARRAY['Inventor_00467'];
```

### Screenshot

> **Add Screenshot Here**

---

# Create GIN Index

```sql
CREATE INDEX idx_inventors
ON patents.patent_inventor_array
USING GIN(inventors);
```

### Screenshot

> **Add Screenshot Here**

---

# Performance Test After Index

```sql
EXPLAIN ANALYZE

SELECT *
FROM patents.patent_inventor_array
WHERE inventors @> ARRAY['Inventor_00467'];
```

### Screenshot

> **Add Screenshot Here**

---

# Task 2 - Find Patents by One Inventor

```sql
SELECT *
FROM patents.patent_inventor_array
WHERE inventors @> ARRAY['Inventor_00467'];
```

### Explanation

`@>` checks whether the array contains the specified inventor.

### Screenshot

> **Add Screenshot Here**

---

# Task 3 - Find Patents Matching Any Inventor

```sql
SELECT *
FROM patents.patent_inventor_array
WHERE inventors && ARRAY
[
    'Inventor_00467',
    'Inventor_01385',
    'Inventor_05942'
];
```

### Explanation

`&&` returns rows where both arrays share at least one common element.

### Screenshot

> **Add Screenshot Here**

---

# Task 4 - Find Patents Sharing Inventors

```sql
SELECT
    p1.publication_number,
    p2.publication_number
FROM patents.patent_inventor_array p1
JOIN patents.patent_inventor_array p2
ON p1.inventors && p2.inventors
AND p1.publication_number < p2.publication_number
LIMIT 20;
```

OR

```sql
WITH sample AS
(
SELECT *
FROM patents.patent_inventor_array
LIMIT 1000
)

SELECT *
FROM sample p1
JOIN sample p2
ON p1.inventors && p2.inventors
AND p1.publication_number < p2.publication_number;
```

### Screenshot

> **Add Screenshot Here**

---

# Task 5 - Convert ARRAY Back to Rows

```sql
SELECT
    publication_number,
    UNNEST(inventors) AS inventor_name
FROM patents.patent_inventor_array

EXCEPT

SELECT
    publication_number,
    inventor_name
FROM patents.patent_inventors;
```

### Explanation

UNNEST converts ARRAY elements back into individual rows.

If no rows are returned, both datasets are identical.

### Screenshot

> **Add Screenshot Here**

---

# Task 6 - Create JSONB Metadata

```sql
CREATE TABLE patents.patent_metadata AS

SELECT
    publication_number,
    jsonb_build_object
    (
        'country','US',
        'category','Mechanical',
        'status','Granted',
        'technology','AI',
        'filing_date','2023-01-01'
    ) AS metadata

FROM patents_training;
```

### Screenshot

> **Add Screenshot Here**

##(OR)

```sql
SELECT
    publication_number,
    jsonb_build_object(
        'country',
            (ARRAY['US','JP','DE','IN'])[floor(random()*4)+1],
        'category',
            (ARRAY['Mechanical','Electronics','Chemical'])[floor(random()*3)+1],
        'status',
            (ARRAY['Granted','Pending'])[floor(random()*2)+1],
        'technology',
            (ARRAY['AI','IoT','Robotics'])[floor(random()*3)+1],
        'filing_date',
            CURRENT_DATE - (random()*1000)::int
    ) AS metadata
FROM patents_training;
```

---

# Performance Test Before JSONB Index

```sql
EXPLAIN ANALYZE

SELECT *
FROM patents.patent_metadata
WHERE metadata->>'technology'='AI';
```

### Screenshot

> **Add Screenshot Here**

---

# Create JSONB GIN Index

```sql
CREATE INDEX idx_metadata
ON patents.patent_metadata
USING GIN(metadata);
```

### Screenshot

> **Add Screenshot Here**

---

# Performance Test After JSONB Index

```sql
EXPLAIN ANALYZE

SELECT *
FROM patents.patent_metadata
WHERE metadata->>'technology'='AI';
```

### Screenshot

> **Add Screenshot Here**

---

# Task 7 - Query JSONB

## Country

```sql
SELECT COUNT(*)
FROM patents.patent_metadata
WHERE metadata->>'country'='US';
```

```sql
SELECT *
FROM patents.patent_metadata
WHERE metadata->>'country'='US'
LIMIT 20;
```

---

## Technology

```sql
SELECT *
FROM patents.patent_metadata
WHERE metadata->>'technology'='AI';
```

---

## Status

```sql
SELECT *
FROM patents.patent_metadata
WHERE metadata->>'status'='Granted';
```

### Screenshot

> **Add Screenshot Here**

---

# Task 8 - Update JSONB

```sql
SELECT publication_number
FROM patents_training
LIMIT 10;
```

```sql
UPDATE patents.patent_metadata
SET metadata =
jsonb_set
(
    metadata,
    '{status}',
    '"Expired"'
)
WHERE publication_number='US0000000001';
```

### Screenshot

> **Add Screenshot Here**

---

# Task 9 - Aggregate Inventors as JSON

```sql
SELECT
    p.publication_number,
    p.title,
    jsonb_agg(pi.inventor_name)
FROM patents_training p
JOIN patents.patent_inventors pi
USING (publication_number)
GROUP BY
    p.publication_number,
    p.title
LIMIT 20;
```

### Screenshot

> **Add Screenshot Here**

---

# Task 10 - Build Nested JSON

```sql
WITH sample AS
(
    SELECT *
    FROM patents_training
    WHERE EXTRACT(YEAR FROM publication_date)=2023
    LIMIT 100
)

SELECT jsonb_build_object
(
    'year',
    EXTRACT(YEAR FROM publication_date),

    'patents',

    jsonb_agg
    (
        jsonb_build_object
        (
            'publication_number',
            publication_number,

            'title',
            title,

            'inventors',

            (
                SELECT jsonb_agg(inventor_name)
                FROM patents.patent_inventors pi
                WHERE pi.publication_number=s.publication_number
            )
        )
    )
)

AS patent_hierarchy

FROM sample s

GROUP BY
EXTRACT(YEAR FROM publication_date);
```

### Screenshot

> **Add Screenshot Here**

---

# Task 11 - Performance Comparison

Run **EXPLAIN ANALYZE** before and after creating indexes.

Example:

```sql
EXPLAIN ANALYZE

SELECT *
FROM patents.patent_inventor_array
WHERE inventors @> ARRAY['Inventor_00467'];
```

```sql
EXPLAIN ANALYZE

SELECT *
FROM patents.patent_metadata
WHERE metadata->>'technology'='AI';
```

### Screenshot Before Index

> **Add Screenshot Here**

### Screenshot After Index

> **Add Screenshot Here**

---

# Task 12 - ARRAY Functions

---

## 1. Find Inventor Position

```sql
SELECT
    publication_number,
    inventors,
    array_position(inventors,'Inventor_09348') AS inventor_position
FROM patents.patent_inventor_array
WHERE inventors @> ARRAY['Inventor_09348'];
```

### Screenshot

> **Add Screenshot Here**

---

## 2. Count Inventors

```sql
SELECT
    publication_number,
    inventors,
    cardinality(inventors) AS inventor_count
FROM patents.patent_inventor_array
ORDER BY inventor_count DESC
LIMIT 10;
```

### Screenshot

> **Add Screenshot Here**

---

## 3. Append Inventor

```sql
SELECT
    publication_number,
    inventors,
    array_append(inventors,'Inventor_99999') AS updated_inventors
FROM patents.patent_inventor_array
WHERE publication_number='US0000001133';
```

### Screenshot

> **Add Screenshot Here**

---

## 4. Remove Inventor

```sql
SELECT
    publication_number,
    inventors,
    array_remove(inventors,'Inventor_06431') AS updated_inventors
FROM patents.patent_inventor_array
WHERE publication_number='US0000001136';
```

### Screenshot

> **Add Screenshot Here**

---

## 5. Slice Array

```sql
SELECT
    publication_number,
    inventors[1:2] AS first_two_inventors
FROM patents.patent_inventor_array
WHERE cardinality(inventors)>=2
LIMIT 10;
```

### Screenshot

> **Add Screenshot Here**

---

## 6. Check Whether an Inventor Exists

```sql
SELECT
    publication_number,
    inventors
FROM patents.patent_inventor_array
WHERE 'Inventor_09915'=ANY(inventors);
```

### Screenshot

> **Add Screenshot Here**

---

# Conclusion

This exercise demonstrates:

- Efficient storage of one-to-many relationships using ARRAY
- Searching arrays with GIN indexes
- JSONB document creation and querying
- Updating JSONB values
- Nested JSON generation
- ARRAY utility functions
- Performance optimization using EXPLAIN ANALYZE
- Comparison of execution plans before and after indexing
