# Finding Patents That Share at Least One Common Inventor

## Objective

Find pairs of patents where at least one inventor is common between them.

---

# Source Table

```sql
SELECT * FROM patents.patent_inventor_array LIMIT 5;
```

<img width="609" height="181" alt="Screenshot 2026-07-23 at 12 50 46 PM" src="https://github.com/user-attachments/assets/0e5eedbd-b2a0-4cfb-b2bf-370aa9ea63f1" />

---

# 1. Using `&&` (Recommended)

## Purpose

Checks whether two arrays have at least one common element.

## Query

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

<img width="375" height="553" alt="Screenshot 2026-07-23 at 12 45 40 PM" src="https://github.com/user-attachments/assets/d08bdef5-c0eb-44bb-bc47-c777ff0579fd" />


## Query Explanation

### `p1` and `p2`

The table is joined with itself (self join).

- `p1` represents the first patent.
- `p2` represents the second patent.

### `p1.inventors && p2.inventors`

Checks whether both inventor arrays have at least one common inventor.

### `p1.publication_number < p2.publication_number`

- Prevents comparing a patent with itself.
- Prevents duplicate pairs such as:
  - Patent A → Patent B
  - Patent B → Patent A

## Advantages

- Fast
- Uses array operators
- Supports GIN indexes
- Recommended for array columns

---

# 2. Using `UNNEST()` (Works but Slower)

## Purpose

Converts both inventor arrays into individual rows and compares each inventor.

## Query

```sql

SELECT DISTINCT
    p1.publication_number,
    p2.publication_number
FROM patents.patent_inventor_array p1
JOIN patents.patent_inventor_array p2
ON p1.publication_number < p2.publication_number
CROSS JOIN LATERAL UNNEST(p1.inventors) AS i1(inventor)
CROSS JOIN LATERAL UNNEST(p2.inventors) AS i2(inventor)
WHERE i1.inventor = i2.inventor
LIMIT 20 ;

```
<img width="424" height="596" alt="Screenshot 2026-07-23 at 12 45 53 PM" src="https://github.com/user-attachments/assets/537c652c-e752-47bf-aa29-188dfb3fa0b7" />

## Query Explanation

### `UNNEST()`

Converts each inventor in the array into a separate row.

### `CROSS JOIN LATERAL`

Executes `UNNEST()` for each patent row.

### `WHERE i1.inventor = i2.inventor`

Returns patent pairs that share a common inventor.

### `DISTINCT`

Removes duplicate patent pairs.

## Advantages

- Easy to understand
- Useful for row-wise processing

## Limitations

- Slower because every array element becomes a row.

---
# 3. Using `EXISTS`

## Purpose

Checks whether two patents share at least one inventor.

## Query

```sql
SELECT
    p1.publication_number,
    p2.publication_number
FROM patents.patent_inventor_array p1
JOIN patents.patent_inventor_array p2
ON p1.publication_number < p2.publication_number
WHERE EXISTS (
    SELECT 1
    FROM UNNEST(p1.inventors) AS i1(inventor)
    JOIN UNNEST(p2.inventors) AS i2(inventor)
      ON i1.inventor = i2.inventor
)
LIMIT 20;
```
<img width="352" height="645" alt="Screenshot 2026-07-23 at 12 46 01 PM" src="https://github.com/user-attachments/assets/9cc03588-ae0e-4fb2-a890-23c1e7b92286" />

---

# 4. Joining the Original Normalized Table

## Purpose

Finds common inventors using the normalized table instead of arrays.

## Query

```sql
SELECT DISTINCT
    p1.publication_number,
    p2.publication_number
FROM patents.patent_inventors p1
JOIN patents.patent_inventors p2
ON p1.inventor_name = p2.inventor_name
AND p1.publication_number < p2.publication_number
LIMIT 20;
```
<img width="371" height="571" alt="Screenshot 2026-07-23 at 12 46 13 PM" src="https://github.com/user-attachments/assets/6f36147f-f256-4f44-88ba-cdd359500de6" />

---

## Performance Comparison

| Method | Uses Arrays | Execution Time | Relative Performance | Recommended Use Case |
|---------|:-----------:|---------------:|:--------------------:|----------------------|
| `&&` | Yes | 25.889 ms | 100% | Best choice for searching array columns using a GIN index. |
| `UNNEST()` | Yes | 68.350 ms | 38% | Reporting, analysis, and row-level processing. |
| `EXISTS` + `UNNEST()` | Yes | 1303.750 ms | 2% | Complex filtering logic using array elements. |
| Join original `patent_inventors` table | No | 1393.213 ms | 2% | Queries against the normalized relational table. |


```
---
```
# Recommendation

- Use `&&` when the inventors are stored as arrays.
- Use `EXISTS` when more flexible matching logic is required.
- Use `UNNEST()` for row-wise analysis or learning purposes.
- Use the normalized `patent_inventors` table when the data is already stored in one row per inventor.
```
