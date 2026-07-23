# Verifying Array Data Against the Original Normalized Table

## Objective

The `patent_inventor_array` table stores inventors as an array (`TEXT[]`), while the `patent_inventors` table stores one inventor per row.

The goal is to verify that both tables contain the same inventor information.

If the verification queries return **0 rows**, it means there are no differences and both tables contain identical data.

---

# Source Tables

## Array Table

```sql
SELECT * FROM patents.patent_inventor_array LIMIT 5;
```

> **Output:**
<img width="609" height="181" alt="Screenshot 2026-07-23 at 12 50 46 PM" src="https://github.com/user-attachments/assets/1ec8edc4-4f91-4448-9e82-43b3a610c52c" />

---

## Normalized Table

```sql
SELECT * FROM patents.patent_inventors LIMIT 5;
```

> **Output:** 

<img width="414" height="192" alt="Screenshot 2026-07-23 at 1 36 14 PM" src="https://github.com/user-attachments/assets/dafb2ef7-3ab2-4542-8261-13ea53e21b54" />

---

# 1. Using `EXCEPT`

## Purpose

Finds inventor records that exist in the array table but do not exist in the normalized table.

## Query

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

> **Output:**
<img width="319" height="294" alt="Screenshot 2026-07-23 at 1 26 50 PM" src="https://github.com/user-attachments/assets/33bf75c3-a038-4e04-ae77-ab3849e1ee2a" />


## Query Explanation

### `UNNEST(inventors)`

Converts each inventor stored in the array into a separate row.

---

### `EXCEPT`

Returns rows from the first query that are **not present** in the second query.

---

### Second SELECT

Returns inventor records from the normalized table.

---

### Result

If **0 rows** are returned, every inventor in the array table exists in the normalized table.

## Advantages

- Simple to write
- Easy to understand
- Good for complete data validation

## Limitations

- Checks only one direction.
- Does not identify rows that exist only in the normalized table.

---

# 2. Using `NOT EXISTS` (Recommended Alternative)

## Purpose

Checks whether each inventor from the array table exists in the normalized table.

## Query

```sql
SELECT
    p.publication_number,
    inventor_name
FROM patents.patent_inventor_array p
CROSS JOIN LATERAL UNNEST(inventors) AS inventor_name
WHERE NOT EXISTS (
    SELECT 1
    FROM patents.patent_inventors pi
    WHERE pi.publication_number = p.publication_number
      AND pi.inventor_name = inventor_name
);
```

> **Output:**

<img width="396" height="275" alt="Screenshot 2026-07-23 at 1 26 57 PM" src="https://github.com/user-attachments/assets/92c69acc-3a8f-4eda-aa8e-6fe7a6ef1411" />


## Query Explanation

### `UNNEST(inventors)`

Converts the inventor array into individual rows.

---

### `CROSS JOIN LATERAL`

Runs `UNNEST()` for each patent.

---

### `NOT EXISTS`

Checks whether the inventor is missing from the normalized table.

---

### `SELECT 1`

Only checks for existence.

The value `1` is never returned.

---

### Result

If **0 rows** are returned, every inventor exists in the normalized table.

## Advantages

- Faster than `EXCEPT` in many cases
- Stops checking once a match is found
- Good alternative for validation

---

# 3. Using `LEFT JOIN`

## Purpose

Finds inventors that exist in the array table but have no matching row in the normalized table.

## Query

```sql
SELECT
    p.publication_number,
    u.inventor_name
FROM patents.patent_inventor_array p
CROSS JOIN LATERAL UNNEST(p.inventors) AS u(inventor_name)
LEFT JOIN patents.patent_inventors pi
ON p.publication_number = pi.publication_number
AND u.inventor_name = pi.inventor_name
WHERE pi.publication_number IS NULL;
```

> **Output:**

<img width="440" height="248" alt="Screenshot 2026-07-23 at 1 27 08 PM" src="https://github.com/user-attachments/assets/f3716ce6-a7b6-4ddc-9101-211a0e1e2729" />


## Query Explanation

### `UNNEST()`

Converts inventor arrays into individual rows.

---

### `LEFT JOIN`

Attempts to match every inventor with the normalized table.

---

### `WHERE pi.publication_number IS NULL`

Returns only inventors that do not have a matching row.

---

### Result

If **0 rows** are returned, all inventors are present in both tables.

## Advantages

- Easy to understand
- Common SQL validation technique

---

# 4. Using `FULL OUTER JOIN`

## Purpose

Finds differences in both tables.

Unlike `EXCEPT`, this method checks both directions in a single query.

## Query

```sql
SELECT *
FROM (
    SELECT
        publication_number,
        UNNEST(inventors) AS inventor_name
    FROM patents.patent_inventor_array
) a
FULL OUTER JOIN patents.patent_inventors b
ON a.publication_number = b.publication_number
AND a.inventor_name = b.inventor_name
WHERE a.publication_number IS NULL
   OR b.publication_number IS NULL;
```

> **Output:**

<img width="640" height="314" alt="Screenshot 2026-07-23 at 1 27 19 PM" src="https://github.com/user-attachments/assets/3b0af37c-ddc0-4f96-a027-93db31bf973a" />


## Query Explanation

### Subquery

Converts the inventor array into rows.

---

### `FULL OUTER JOIN`

Returns all rows from both tables.

Matching rows are combined.

---

### `WHERE`

Keeps only rows that exist in one table but not the other.

---

### Result

If **0 rows** are returned, both tables contain identical inventor data.

## Advantages

- Validates both tables in one query
- Detects missing rows in either table

---

# 5. Using `INTERSECT`

## Purpose

Returns only the inventor records that exist in both tables.

Unlike the previous methods, this query shows **matching data**, not differences.

## Query

```sql
SELECT
    publication_number,
    UNNEST(inventors) AS inventor_name
FROM patents.patent_inventor_array

INTERSECT

SELECT
    publication_number,
    inventor_name
FROM patents.patent_inventors
LIMIT 20;
```

> **Output:**

<img width="364" height="617" alt="Screenshot 2026-07-23 at 1 27 33 PM" src="https://github.com/user-attachments/assets/164bbe39-5245-489b-8e71-f67ab09538aa" />


## Query Explanation

### `UNNEST(inventors)`

Converts inventor arrays into rows.

---

### `INTERSECT`

Returns only rows that are common to both queries.

---

## Advantages

- Shows matching data
- Useful for confirming common records

## Limitations

- Does not identify missing records
- Cannot fully verify data consistency by itself

---

# Performance Comparison

| Method | Purpose | Can Fully Verify? | Execution Time | Relative Performance |
|-----------------|----------------------------------|:-----------------:|---------------:|--------------------:|
| `NOT EXISTS` | Find unmatched rows | Yes | **303.652 ms** | **95–100%** |
| `FULL OUTER JOIN` | Find differences in both tables | Yes | **442.546 ms** | **90–95%** |
| `LEFT JOIN` | Find unmatched rows | Yes | **469.137 ms** | **85–90%** |
| `INTERSECT` | Find matching rows | Partially | **1759.485 ms** | **70–75%** |
| `EXCEPT` | Find unmatched rows | Yes | **2515.411 ms** | **60–65%** |

---

# Recommendation

- Use `NOT EXISTS` when you need a fast alternative for validating that every inventor in the array table exists in the normalized table.
- Use `FULL OUTER JOIN` when you want to identify differences in both tables using a single query.
- Use `LEFT JOIN` for a simple and widely used approach to finding unmatched rows.
- Use `EXCEPT` to verify that all rows from the array table exist in the normalized table.
- Use `INTERSECT` to display records that are common to both tables, but do not rely on it alone for complete validation.
