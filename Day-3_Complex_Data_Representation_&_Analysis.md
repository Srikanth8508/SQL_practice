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
| **Task 11** | Research and demonstrate additional PostgreSQL operations supported for ARRAY and JSONB data types. | Advanced **ARRAY** and **JSONB** Functions & Operators |

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

<img width="399" height="166" alt="Screenshot 2026-07-21 at 4 04 27 PM" src="https://github.com/user-attachments/assets/4dfc4738-7be7-4d03-89e5-8c71255a737e" />

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

<img width="382" height="485" alt="Screenshot 2026-07-21 at 4 05 14 PM" src="https://github.com/user-attachments/assets/4ffa197c-82a5-4fed-8e3d-bccdbeb56181" />

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

<img width="474" height="212" alt="Screenshot 2026-07-21 at 4 05 47 PM" src="https://github.com/user-attachments/assets/499ee256-dc61-4eff-878b-0b29f6ecfe00" />

---

# Performance Test Before Index

```sql
EXPLAIN ANALYZE

SELECT *
FROM patents.patent_inventor_array
WHERE inventors @> ARRAY['Inventor_00467'];
```

### Screenshot

<img width="969" height="325" alt="Screenshot 2026-07-21 at 4 06 00 PM" src="https://github.com/user-attachments/assets/2b86cd93-4316-4e58-bc60-cf32bb918d20" />

---

# Create GIN Index

```sql
CREATE INDEX idx_inventors
ON patents.patent_inventor_array
USING GIN(inventors);
```

### Screenshot

<img width="301" height="115" alt="Screenshot 2026-07-21 at 4 06 17 PM" src="https://github.com/user-attachments/assets/74fd37b4-d4d1-40f3-94b4-b7106b15a5a7" />

---

# Performance Test After Index

```sql
EXPLAIN ANALYZE

SELECT *
FROM patents.patent_inventor_array
WHERE inventors @> ARRAY['Inventor_00467'];
```

### Screenshot
<img width="909" height="309" alt="Screenshot 2026-07-21 at 4 06 49 PM" src="https://github.com/user-attachments/assets/820e9d15-b2a0-4b16-9bbb-ec61ec5f9d93" />

---

# Task 2 - Find Patents by One Inventor

```sql
SELECT *
FROM patents.patent_inventor_array
WHERE inventors @> ARRAY['Inventor_00467']
LIMIT 20 ;
```

### Explanation

`@>` checks whether the array contains the specified inventor.

### Screenshot

<img width="501" height="504" alt="Screenshot 2026-07-21 at 4 07 53 PM" src="https://github.com/user-attachments/assets/aac44fa3-ac33-48d1-acbd-2618539f538c" />

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

<img width="619" height="591" alt="Screenshot 2026-07-21 at 4 08 39 PM" src="https://github.com/user-attachments/assets/9731a9e0-2ea0-45c2-ab87-d841ba6c387f" />

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

<img width="397" height="567" alt="Screenshot 2026-07-21 at 4 09 08 PM" src="https://github.com/user-attachments/assets/c79bbf59-0dfd-4f8d-831c-94861f6f0a2c" />

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

<img width="316" height="312" alt="Screenshot 2026-07-21 at 4 09 48 PM" src="https://github.com/user-attachments/assets/50d230c1-d720-475a-b267-2e553599c550" />

---

# Task 6 - Create JSONB Metadata

```sql
CREATE TABLE patents.patent_metadata AS

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
### Screenshot

<img width="577" height="340" alt="Screenshot 2026-07-21 at 4 10 51 PM" src="https://github.com/user-attachments/assets/4c70ce3f-09f6-4410-8021-f60cdb5722fb" />

---

# Performance Test Before JSONB Index

```sql
EXPLAIN ANALYZE

SELECT *
FROM patents.patent_metadata
WHERE metadata->>'technology'='AI';
```

### Screenshot

<img width="958" height="326" alt="Screenshot 2026-07-21 at 4 10 59 PM" src="https://github.com/user-attachments/assets/d5c80ba2-413f-47dd-b33e-4bd0062c3752" />

---

# Create JSONB GIN Index

```sql
CREATE INDEX idx_metadata
ON patents.patent_metadata
USING GIN(metadata);
```

### Screenshot

<img width="270" height="132" alt="Screenshot 2026-07-21 at 4 11 24 PM" src="https://github.com/user-attachments/assets/3a377314-7999-42ec-b0d6-81139ae2d8c7" />

---

# Performance Test After JSONB Index

```sql
EXPLAIN ANALYZE

SELECT *
FROM patents.patent_metadata
WHERE metadata->>'technology'='AI';
```

### Screenshot

<img width="962" height="323" alt="Screenshot 2026-07-21 at 4 11 46 PM" src="https://github.com/user-attachments/assets/797c20c9-444d-4aad-b323-9780500ff382" />

---

# Task 7 - Query JSONB

## Country

```sql
SELECT COUNT(*)
FROM patents.patent_metadata
WHERE metadata->>'country'='US';
```
---

## Technology

```sql
SELECT COUNT(*)
FROM patents.patent_metadata
WHERE metadata->>'technology'='AI';
```

<img width="272" height="184" alt="Screenshot 2026-07-21 at 4 12 49 PM" src="https://github.com/user-attachments/assets/7c5f1edd-d4ae-4f5e-b10a-8aa091a3ddc7" />

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

<img width="1048" height="484" alt="Screenshot 2026-07-21 at 4 16 38 PM" src="https://github.com/user-attachments/assets/e447f35a-98d5-4e4d-8864-1d2efad3d60e" />


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

<img width="1407" height="551" alt="Screenshot 2026-07-21 at 4 17 49 PM" src="https://github.com/user-attachments/assets/d1e27f6d-3926-4ce1-8cef-75053319cab9" />

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

<img width="1917" height="858" alt="Screenshot 2026-07-21 at 4 18 19 PM" src="https://github.com/user-attachments/assets/5f106b11-1fa4-4256-a03d-125ed49e4ee9" />

---

# Task 11 - ARRAY Functions

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

<img width="769" height="396" alt="Screenshot 2026-07-21 at 4 19 23 PM" src="https://github.com/user-attachments/assets/ce42cfd9-bee1-4a5d-8c05-e56135a27494" />

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

<img width="726" height="372" alt="Screenshot 2026-07-21 at 4 19 41 PM" src="https://github.com/user-attachments/assets/05fb7214-7e3d-46fe-8b8c-30d551bb618c" />


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

<img width="1163" height="229" alt="Screenshot 2026-07-21 at 4 20 07 PM" src="https://github.com/user-attachments/assets/2a47b6e8-cac0-42d1-b74d-d37057240d5e" />

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

<img width="883" height="232" alt="Screenshot 2026-07-21 at 4 20 25 PM" src="https://github.com/user-attachments/assets/066266b8-af03-4d89-b7a1-14dff5c2aecc" />

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

<img width="445" height="361" alt="Screenshot 2026-07-21 at 4 20 49 PM" src="https://github.com/user-attachments/assets/bc8d1914-e876-4892-a69b-bf7304d0ffbf" />

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

<img width="629" height="376" alt="Screenshot 2026-07-21 at 4 21 23 PM" src="https://github.com/user-attachments/assets/42b611a1-225a-4957-900f-af53fa4ea0f4" />

---
