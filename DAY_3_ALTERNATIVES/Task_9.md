## Objective

Generate a structured document for each patent containing:

- Publication Number
- Patent Title
- All Associated Inventors

This document compares different aggregation methods available in PostgreSQL.

---

# Source Tables

## Patent Table

```sql
SELECT * FROM patents.patents_training LIMIT 5;
```

> **Output:** 

<img width="1513" height="564" alt="Screenshot 2026-07-23 at 6 40 27 PM" src="https://github.com/user-attachments/assets/bbe3767b-b663-4166-a7ee-9769199a1b88" />

---

## Patent Inventor Table

```sql
SELECT * FROM patents.patent_inventors LIMIT 5;
```

> **Output:** 

<img width="434" height="186" alt="Screenshot 2026-07-23 at 6 40 47 PM" src="https://github.com/user-attachments/assets/f4ed60a1-95f2-4be9-a8b2-3ab815fb6e78" />

---

# 1. Using `jsonb_agg()` (Recommended)

## Purpose

Aggregates inventor names into a JSONB array for each patent.

## Query

```sql
SELECT
    p.publication_number,
    p.title,
    jsonb_agg(pi.inventor_name) AS inventors
FROM patents.patents_training p
JOIN patents.patent_inventors pi
USING (publication_number)
GROUP BY
    p.publication_number,
    p.title
LIMIT 5;
```

> **Output:** 

<img width="1405" height="550" alt="Screenshot 2026-07-23 at 6 33 46 PM" src="https://github.com/user-attachments/assets/34c01915-0057-4b2e-b2b9-02227aa993f1" />


## Query Explanation

### `SELECT`

Returns the publication number, title, and inventors.

---

### `jsonb_agg(pi.inventor_name)`

Collects all inventor names into a JSONB array.

Example

```json
[
  "Inventor_06186",
  "Inventor_04817",
  "Inventor_00866",
  "Inventor_04740"
]
```

---

### `JOIN`

Joins patents with their inventors using `publication_number`.

---


## Advantages

- Returns JSONB.
- Supports JSONB operators.
- Suitable for APIs and JSONB storage.
- Easy to query and update.

---

# 2. Using `json_agg()`

## Purpose

Aggregates inventor names into a JSON array.

## Query

```sql
SELECT
    p.publication_number,
    p.title,
    json_agg(pi.inventor_name) AS inventors
FROM patents.patents_training p
JOIN patents.patent_inventors pi
USING (publication_number)
GROUP BY
    p.publication_number,
    p.title
LIMIT 5;
```

> **Output:**

<img width="1403" height="566" alt="Screenshot 2026-07-23 at 6 33 59 PM" src="https://github.com/user-attachments/assets/5f9d2681-d558-45bb-8919-8188f366eb4f" />


## Query Explanation

The query is identical to `jsonb_agg()`.

The only difference is that it returns **JSON** instead of **JSONB**.

## Advantages

- Produces JSON output.
- Useful for API responses.
- Easy to read.

## Limitations

- Does not support JSONB indexing.
- Less flexible for updates.

---

# 3. Using `array_agg()`

## Purpose

Aggregates inventor names into a PostgreSQL array.

## Query

```sql
SELECT
    p.publication_number,
    p.title,
    array_agg(pi.inventor_name) AS inventors
FROM patents.patents_training p
JOIN patents.patent_inventors pi
USING (publication_number)
GROUP BY
    p.publication_number,
    p.title
LIMIT 5;
```

> **Output:** 

<img width="1410" height="547" alt="Screenshot 2026-07-23 at 6 34 13 PM" src="https://github.com/user-attachments/assets/9a606da0-d856-4ee4-a57a-911dff664a50" />


## Query Explanation

### `array_agg()`

Collects inventor names into a PostgreSQL array.

Example

```text
{Inventor_06186,Inventor_04817,Inventor_00866,Inventor_04740}
```

## Advantages

- Supports PostgreSQL array operators.
- Useful for array processing.
- Easy to search using array functions.

## Limitations

- Not suitable for JSON APIs.

---

# 4. Using `string_agg()`

## Purpose

Combines inventor names into a comma-separated text string.

## Query

```sql
SELECT
    p.publication_number,
    p.title,
    string_agg(pi.inventor_name, ', ') AS inventors
FROM patents.patents_training p
JOIN patents.patent_inventors pi
USING (publication_number)
GROUP BY
    p.publication_number,
    p.title
LIMIT 5;
```

> **Output:** 

<img width="1415" height="548" alt="Screenshot 2026-07-23 at 6 34 31 PM" src="https://github.com/user-attachments/assets/e60da30f-8fd2-4dd1-ac91-73fa9644908f" />


## Query Explanation

### `string_agg()`

Concatenates inventor names into a single text value.

Example

```text
Inventor_06186, Inventor_04817, Inventor_00866, Inventor_04740
```

## Advantages

- Easy to display.
- Suitable for reports.
- Human-readable output.

## Limitations

- Cannot use JSON or array operators.
- Difficult to search individual inventors.

---

# 5. Using `jsonb_build_object()` + `jsonb_agg()`

## Purpose

Creates a complete JSONB document for each patent.

## Query

```sql
SELECT jsonb_build_object(
    'publication_number', p.publication_number,
    'title', p.title,
    'inventors', jsonb_agg(pi.inventor_name)
) AS patent_document
FROM patents.patents_training p
JOIN patents.patent_inventors pi
USING (publication_number)
GROUP BY
    p.publication_number,
    p.title
LIMIT 5;
```

> **Output:**

<img width="1920" height="594" alt="Screenshot 2026-07-23 at 6 34 55 PM" src="https://github.com/user-attachments/assets/71b68c63-3430-442f-ac0b-f48686428be9" />


## Query Explanation

### `jsonb_build_object()`

Creates a JSONB object using key-value pairs.

---

### `publication_number`

Stores the patent number.

---

### `title`

Stores the patent title.

---

### `jsonb_agg()`

Stores all inventors as a JSONB array.

### Example Output

```json
{
  "publication_number": "US0000000001",
  "title": "Patent Title",
  "inventors": [
    "Inventor_06186",
    "Inventor_04817",
    "Inventor_00866",
    "Inventor_04740"
  ]
}
```

## Advantages

- Creates a complete JSONB document.
- Best for document-oriented applications.
- Supports JSONB indexing and updates.

---

# 6. Using `json_build_object()` + `json_agg()`

## Purpose

Creates a complete JSON document for each patent.

## Query

```sql
SELECT json_build_object(
    'publication_number', p.publication_number,
    'title', p.title,
    'inventors', json_agg(pi.inventor_name)
) AS patent_document
FROM patents.patents_training p
JOIN patents.patent_inventors pi
USING (publication_number)
GROUP BY
    p.publication_number,
    p.title
LIMIT 5;
```

> **Output:** 

<img width="1920" height="603" alt="Screenshot 2026-07-23 at 6 35 13 PM" src="https://github.com/user-attachments/assets/6c0d3860-0fb4-471f-8bf8-ecdb7b47342a" />


## Query Explanation

The query is similar to `jsonb_build_object()`.

The difference is that it returns **JSON** instead of **JSONB**.

## Advantages

- Creates a complete JSON document.
- Good for exporting JSON.
- Simple to use.

## Limitations

- Does not support JSONB indexing.
- Less efficient for updates.

---

# Performance Comparison

| Method | Output Type | Execution Time | Relative Performance |
|----------------------------------------------|---------------------------|---------------:|--------------------:|
| `json_agg()` | JSON Array | **1.453 ms** | **95–100%** |
| `jsonb_agg()` | JSONB Array | **1.850 ms** | **90–95%** |
| `array_agg()` | PostgreSQL Array | **2.756 ms** | **80–85%** |
| `jsonb_build_object()` + `jsonb_agg()` | Complete JSONB Document | **2.821 ms** | **75–80%** |
| `json_build_object()` + `json_agg()` | Complete JSON Document | **2.948 ms** | **70–75%** |
| `string_agg()` | Text String | **5.425 ms** | **60–65%** |

---

# Recommendation

- Use `jsonb_agg()` when you need a JSONB array that supports PostgreSQL JSONB operators and indexing.
- Use `json_agg()` when generating JSON output for APIs or data exchange.
- Use `array_agg()` when working with PostgreSQL array functions and operators.
- Use `string_agg()` when a comma-separated text value is needed for reports or display.
- Use `jsonb_build_object()` with `jsonb_agg()` to generate a complete JSONB document containing patent details and inventor information.
- Use `json_build_object()` with `json_agg()` when a complete JSON document is required instead of JSONB.
