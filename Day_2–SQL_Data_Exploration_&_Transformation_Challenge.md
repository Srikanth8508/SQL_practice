# Day 2 - SQL Data Exploration & Performance Optimization
**Project:** Patent Analytics using PostgreSQL

---

# Dataset Information

| Property | Value |
|----------|-------|
| Database | PostgreSQL |
| Schema | patents |
| Table | patents_training |
| Records | 1,000,000 |
| Purpose | SQL Transformation & Performance Practice |

---

# 1. Create Patent Training Table

## Table Creation

```sql
CREATE TABLE patents.patents_training
(
    publication_number TEXT PRIMARY KEY,
    inventor_name      TEXT,
    publication_date   DATE,
    title              TEXT,
    abstract           TEXT
);
```

---

### Purpose

This table stores the patent information used throughout the exercises.

---

### Screenshot


---

# 2. Load Sample Patent Dataset

Populate the table with **1 million synthetic patent records**.

```sql

INSERT INTO patents.patents_training
(
    publication_number,
    inventor_name,
    publication_date,
    title,
    abstract
)
SELECT
    'US' || LPAD(gs::text, 10, '0'),

    'Inventor_' || LPAD((1 + floor(random() * 10000))::int::text, 5, '0'),

    DATE '2010-01-01' + (random() * 5843)::int,

    concat_ws(
        ' ',
        adjectives[(random()*19+1)::int],
        technologies[(random()*19+1)::int],
        actions[(random()*19+1)::int],
        'for',
        domains[(random()*19+1)::int],
        'using',
        technologies[(random()*19+1)::int],
        connectors[(random()*19+1)::int],
        features[(random()*19+1)::int],
        architectures[(random()*19+1)::int],
        'with',
        benefits[(random()*19+1)::int],
        qualities[(random()*19+1)::int],
        'performance',
        'and',
        security[(random()*19+1)::int],
        'based',
        'system',
        'architecture',
        versions[(random()*19+1)::int]
    ),

    concat_ws(
        ' ',
        'This invention provides',
        adjectives[(random()*19+1)::int],
        technologies[(random()*19+1)::int],
        'for',
        domains[(random()*19+1)::int],
        'using',
        architectures[(random()*19+1)::int],
        'to improve',
        benefits[(random()*19+1)::int],
        'performance, scalability, availability and security.'
    )

FROM generate_series(1,1000000) gs

CROSS JOIN
(
SELECT

ARRAY[
'Machine','Cloud','Distributed','Neural','Artificial',
'Quantum','Blockchain','Battery','Electric','Medical',
'Image','Wireless','Sensor','Edge','Cyber',
'Speech','Network','Autonomous','Virtual','Digital'
] technologies,

ARRAY[
'System','Method','Framework','Architecture','Platform',
'Engine','Application','Solution','Algorithm','Model',
'Protocol','Mechanism','Workflow','Service','Module',
'Controller','Interface','Process','Device','Analytics'
] actions,

ARRAY[
'Healthcare','Manufacturing','Finance','Retail',
'Agriculture','Education','Automotive','IoT',
'Cloud','Robotics','Satellite','Security',
'Telecommunication','Energy','Payments',
'Logistics','SupplyChain','Medical','SmartCity','Defence'
] domains,

ARRAY[
'advanced','intelligent','scalable','distributed',
'secure','adaptive','dynamic','predictive',
'robust','optimized','highspeed','cloudnative',
'autonomous','wireless','efficient','reliable',
'virtualized','parallel','automated','flexible'
] adjectives,

ARRAY[
'through','via','leveraging','utilizing',
'combining','integrating','supporting','enabling',
'optimizing','processing','executing','monitoring',
'controlling','managing','analyzing','predicting',
'detecting','improving','accelerating','transforming'
] connectors,

ARRAY[
'realtime','streaming','analytics','monitoring',
'optimization','automation','processing','storage',
'routing','prediction','classification','compression',
'authentication','authorization','encryption','synchronization',
'visualization','coordination','diagnostics','recovery'
] features,

ARRAY[
'framework','architecture','pipeline','database',
'cluster','engine','platform','gateway',
'controller','scheduler','processor','interface',
'network','application','repository','cache',
'queue','service','module','workflow'
] architectures,

ARRAY[
'high','better','enhanced','maximum',
'greater','improved','efficient','fast',
'optimized','reliable','stable','consistent',
'predictable','secure','scalable','flexible',
'faulttolerant','continuous','intelligent','dynamic'
] benefits,

ARRAY[
'overall','operational','computational','system',
'business','enterprise','network','application',
'resource','processing','storage','service',
'cloud','database','transaction','runtime',
'deployment','production','execution','workflow'
] qualities,

ARRAY[
'protection','encryption','authentication','validation',
'authorization','monitoring','verification','integrity',
'privacy','compliance','governance','availability',
'confidentiality','resilience','isolation','backup',
'auditing','detection','prevention','recovery'
] security,

ARRAY[
'v1','v2','v3','generation',
'nextgeneration','release','edition','model',
'series','prototype','variant','revision',
'phase1','phase2','phase3','alpha',
'beta','gamma','enterprise','premium'
] versions

) x;

```

*(Use the INSERT statement provided in the exercise.)*

---

### Data Generation Highlights

- Random inventor names
- Random publication dates
- Automatically generated patent titles
- Automatically generated patent abstracts

---


---

# 3. Create Inventor Master

## Objective

Normalize inventor information by storing only unique inventor names.

---

## Create Table

```sql
CREATE TABLE patents.inventor_master
(
    inventor_id BIGSERIAL PRIMARY KEY,
    inventor_name TEXT UNIQUE
);
```

---

## Populate Inventor Master

```sql
INSERT INTO patents.inventor_master(inventor_name)

SELECT DISTINCT
       TRIM(inventor_name)
FROM patents.patents_training
WHERE inventor_name IS NOT NULL
AND TRIM(inventor_name) <> '';
```

---

## Create Index

```sql
CREATE INDEX idx_inventor_name
ON patents.inventor_master(inventor_name);
```

---

### Expected Result

- Duplicate inventor names removed
- Unique inventors stored
- Faster inventor lookup using index

---

### Screenshot

<img width="375" height="398" alt="Screenshot 2026-07-17 at 12 09 54 PM" src="https://github.com/user-attachments/assets/122c2a95-63da-478d-94da-7cbb8541931d" />


---

# 4. Patent Title Analysis

## Objective

Extract frequently used words from patent titles.

---

## Create Word Frequency Table

```sql
CREATE TABLE patents.title_word_frequency
(
    word TEXT PRIMARY KEY,
    frequency BIGINT,
    rank INT
);
```

---

## Insert Top 100 Words

```sql
INSERT INTO patents.title_word_frequency(word, frequency, rank)

WITH words AS
(
    SELECT
        regexp_split_to_table(lower(title), '[^a-z0-9]+') AS word
    FROM patents.patents_training
),

filtered AS
(
    SELECT word
    FROM words
    WHERE length(word) > 3
),

freq AS
(
    SELECT
        word,
        COUNT(*) AS frequency
    FROM filtered
    GROUP BY word
)

SELECT
    word,
    frequency,

    DENSE_RANK()
    OVER (ORDER BY frequency DESC) AS rank

FROM freq

ORDER BY frequency DESC

LIMIT 100;
```

---

## View Top Words

```sql
SELECT *
FROM patents.title_word_frequency
ORDER BY rank
LIMIT 50;
```

---

### Concepts Covered

- `regexp_split_to_table()`
- Regular Expressions
- Common Table Expressions (CTEs)
- Aggregation
- Window Functions
- `DENSE_RANK()`

---

### Screenshot

<img width="559" height="755" alt="Screenshot 2026-07-17 at 12 13 01 PM" src="https://github.com/user-attachments/assets/3764043d-5d17-42e7-85b7-dd60cc7f3595" />

<img width="300" height="968" alt="Screenshot 2026-07-17 at 12 15 07 PM" src="https://github.com/user-attachments/assets/ce9a8bf1-df43-4a29-9e73-a4e6c70fef77" />

---

# 5. Patent Coverage Analysis

## Objective

Find patents whose titles do **not contain** any of the Top 100 frequent words.

---

```sql
SELECT *
FROM patents.patents_training p
WHERE NOT EXISTS
(
    SELECT 1
    FROM patents.title_word_frequency t
    WHERE lower(p.title) ~
    ('(^|[^a-z0-9])' ||
     t.word ||
     '([^a-z0-9]|$)')
);
```

---

### Concepts Covered

- Correlated Subquery
- NOT EXISTS
- Regular Expressions
- Pattern Matching

---

### Screenshot

<img width="473" height="262" alt="Screenshot 2026-07-17 at 12 35 39 PM" src="https://github.com/user-attachments/assets/af7476c5-98a5-42c6-95dd-978301563fc9" />

---

# 6. Patent Trend Analysis

## Objective

Analyze publication trends over the years.

---

## Patent Count by Year & Highest Patent Publication Year

```sql
SELECT
    EXTRACT(YEAR FROM publication_date)::INT AS year,
    COUNT(*) AS patent_count
FROM patents.patents_training
GROUP BY year
ORDER BY patent_count DESC
LIMIT 10;


SELECT
    EXTRACT(YEAR FROM publication_date)::INT AS year,
    COUNT(*) AS patent_count
FROM patents.patents_training
GROUP BY year
ORDER BY patent_count DESC
LIMIT 1;


```

---

### Screenshot

<img width="495" height="616" alt="Screenshot 2026-07-17 at 12 42 01 PM" src="https://github.com/user-attachments/assets/0354984c-76d6-4da4-89bf-c16722b1ac41" />

---

## Year-over-Year Growth Analysis

```sql
WITH yearly
AS
(
    SELECT
        EXTRACT(YEAR FROM publication_date)::INT AS year,
        COUNT(*) AS patent_count
    FROM patents.patents_training
    GROUP BY year
)

SELECT
    year,
    patent_count,
    LAG(patent_count)
    OVER (ORDER BY year)
    AS previous_year,
    ROUND
    (
        (
            patent_count -
            LAG(patent_count)
            OVER (ORDER BY year)
        )
        *100.0
        /
        NULLIF
        (
            LAG(patent_count)
            OVER (ORDER BY year),
            0
        ),
        2
    ) AS growth_percentage
FROM yearly
ORDER BY year;
```

---

### Concepts Covered

- `LAG()`
- Window Functions
- Growth Percentage
- NULLIF
- CTE

---

### Screenshot

<img width="487" height="1021" alt="Screenshot 2026-07-17 at 12 42 09 PM" src="https://github.com/user-attachments/assets/150f6e26-d4a4-49ac-a97a-8fa9cbb9bbd5" />

---

# 7. Performance Optimization & Functional GIN Index on Title

## Objective

Improve query performance using indexes.

---

## Index on Publication Date

```sql
CREATE INDEX idx_patent_date
ON patents.patents_training(publication_date);

CREATE INDEX idx_lower_title
ON patents.patents_training
USING gin
(
    to_tsvector('simple', lower(title))
);

ANALYZE patents.patents_training;


```

---

### Purpose

Improves filtering by publication date.
Optimizes full-text search on patent titles.
Refresh planner statistics for better execution plans.

---

### Screenshot

<img width="351" height="267" alt="Screenshot 2026-07-17 at 4 49 07 PM" src="https://github.com/user-attachments/assets/717aa195-2b28-4dfe-830e-6ca85e4c7e4e" />

---

# 8. Query Performance Benchmarking

Use `EXPLAIN ANALYZE` to compare execution plans.

---

## Example 1

```sql
EXPLAIN ANALYZE

SELECT
    EXTRACT(YEAR FROM publication_date),
    COUNT(*)
FROM patents.patents_training
GROUP BY 1;
```

### Screenshot

<img width="1317" height="486" alt="Screenshot 2026-07-17 at 4 50 10 PM" src="https://github.com/user-attachments/assets/a52d698f-0d10-47db-abf4-8a06acd2f080" />

---

## Example 2

```sql
EXPLAIN ANALYZE

SELECT *
FROM patents.patents_training
WHERE publication_date >= DATE '2020-01-01';
```

### Screenshot

<img width="981" height="314" alt="Screenshot 2026-07-17 at 4 50 17 PM" src="https://github.com/user-attachments/assets/e854737e-a181-41c4-8f34-d0eae19f7b5b" />

---

## Example 3

```sql
EXPLAIN ANALYZE

SELECT *
FROM patents.patents_training
WHERE publication_date
BETWEEN '2020-01-01'
AND '2020-12-31';
```

### Screenshot

<img width="963" height="323" alt="Screenshot 2026-07-17 at 5 01 50 PM" src="https://github.com/user-attachments/assets/63d9e329-64b1-4a4e-afb7-64225265a242" />

---

## Example 4

```sql
EXPLAIN ANALYZE

SELECT *
FROM patents.patents_training
WHERE title LIKE '%battery%';
```

### Screenshot

<img width="960" height="311" alt="Screenshot 2026-07-17 at 5 02 03 PM" src="https://github.com/user-attachments/assets/cd15e9f4-3bf4-4dd4-8f91-53837dcfe7c2" />

---

## Example 5

```sql
EXPLAIN ANALYZE

SELECT
    COUNT(*),
    MIN(publication_date),
    MAX(publication_date)
FROM patents.patents_training;
```

### Screenshot

<img width="1264" height="351" alt="Screenshot 2026-07-17 at 5 02 12 PM" src="https://github.com/user-attachments/assets/49dc8a2a-df98-4c70-8f24-fbb64739c14f" />

---
