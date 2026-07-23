# Creating JSON Metadata in PostgreSQL

## Objective

Create patent metadata as a single JSON document using different PostgreSQL functions and compare their performance.

The metadata contains the following fields:

- country
- category
- status
- technology
- filing_date

---

# Source Table

## Query

```sql
SELECT * FROM patents.patents_training LIMIT 5;
```

> **Output:** 

<img width="1480" height="537" alt="Screenshot 2026-07-23 at 4 20 11 PM" src="https://github.com/user-attachments/assets/4d17e736-cb18-4765-a530-0d19fffcbbac" />

---

# 1. Using `jsonb_build_object()` (Recommended)

## Purpose

Creates a JSONB object by specifying each key and its corresponding value.

## Query

```sql
CREATE TABLE patent_1.patent_metadata_jsonb AS
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
            CURRENT_DATE - (random()*10)::int
    ) AS metadata
FROM patents.patents_training
LIMIT 1000;
```

> **Output:**

<img width="1037" height="261" alt="Screenshot 2026-07-23 at 4 17 43 PM" src="https://github.com/user-attachments/assets/0bedc2be-315e-4d9d-9ea0-009d5c86432c" />


## Query Explanation

### `jsonb_build_object()`

Creates a JSONB object using key-value pairs.

---

### `'country', value`

Creates a key named `country` and assigns a randomly selected country.

---

### `ARRAY[...]`

Stores possible values.

---

### `floor(random()*4)+1`

Randomly selects one element from the array.

---

### `CURRENT_DATE - (random()*10)::int`

Generates a filing date within the last 10 days.

---

## Advantages

- Produces JSONB directly.
- Easy to read and maintain.
- Supports JSONB indexing and operators.
- Best choice for PostgreSQL applications.

---

# 2. Using `json_build_object()`

## Purpose

Creates a JSON object using key-value pairs.

## Query

```sql
CREATE TABLE patent_1.patent_metadata_json AS
SELECT
    publication_number,
    json_build_object(
        'country',
            (ARRAY['US','JP','DE','IN'])[floor(random()*4)+1],
        'category',
            (ARRAY['Mechanical','Electronics','Chemical'])[floor(random()*3)+1],
        'status',
            (ARRAY['Granted','Pending'])[floor(random()*2)+1],
        'technology',
            (ARRAY['AI','IoT','Robotics'])[floor(random()*3)+1],
        'filing_date',
            CURRENT_DATE - (random()*10)::int
    ) AS metadata
FROM patents.patents_training
LIMIT 1000;
```

> **Output:**

<img width="1022" height="257" alt="Screenshot 2026-07-23 at 4 17 33 PM" src="https://github.com/user-attachments/assets/770ead35-8225-4ff4-b2f4-b241a21e9f90" />


## Query Explanation

The query is similar to `jsonb_build_object()`.

The difference is that it returns the **JSON** data type instead of **JSONB**.

## Advantages

- Easy to create JSON documents.
- Suitable when only storing or exporting JSON.

## Limitations

- Does not support JSONB indexing.
- Slower for searching and updating compared to JSONB.

---

# 3. Using `to_jsonb()`

## Purpose

Converts an existing row into a JSONB object.

## Query

```sql
CREATE TABLE patent_1.patent_metadata_conv AS
SELECT
    publication_number,
    to_jsonb(
        ROW(country, category, status, technology, filing_date)
    ) AS metadata
FROM (
    SELECT
        publication_number,
        (ARRAY['US','JP','DE','IN'])[floor(random()*4)+1] AS country,
        (ARRAY['Mechanical','Electronics','Chemical'])[floor(random()*3)+1] AS category,
        (ARRAY['Granted','Pending'])[floor(random()*2)+1] AS status,
        (ARRAY['AI','IoT','Robotics'])[floor(random()*3)+1] AS technology,
        CURRENT_DATE - (random()*10)::int AS filing_date
    FROM patents.patents_training
) t
LIMIT 1000;
```

> **Output:**

<img width="865" height="263" alt="Screenshot 2026-07-23 at 4 17 50 PM" src="https://github.com/user-attachments/assets/9c930b5e-e8ac-43d2-87b3-509bb50089b8" />


## Query Explanation

### `ROW(...)`

Creates a PostgreSQL row containing multiple values.

---

### `to_jsonb()`

Converts the entire row into a JSONB object.

---

### Subquery

Generates all metadata values before converting them to JSONB.

---

## Advantages

- Useful when converting existing rows to JSONB.
- Reduces manual key-value construction.

## Limitations

- Less readable.
- Generates generic field names (`f1`, `f2`, etc.) unless using composite types.
- Slower than `jsonb_build_object()`.

---

# Performance Comparison

| Method | Return Type | Execution Time | Relative Performance |
|----------------------|-------------|---------------:|--------------------:|
| `jsonb_build_object()` | JSONB | **12.729 ms** | **95–100%** |
| `json_build_object()` | JSON | **15.445 ms** | **85–90%** |
| `to_jsonb(ROW(...))` | JSONB | **27.920 ms** | **60–70%** |

---

# Recommendation

- Use `jsonb_build_object()` when creating JSONB documents in PostgreSQL. It provides the best performance and supports JSONB indexing and operators.
- Use `json_build_object()` when JSON output is required and JSONB features are not needed.
- Use `to_jsonb()` when converting existing rows or records into JSONB instead of manually creating key-value pairs.
