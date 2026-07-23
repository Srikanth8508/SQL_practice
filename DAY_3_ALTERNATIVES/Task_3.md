# Finding Patents by Multiple Inventor Names from an Array Column

## Objective

Find patents that contain one or more specific inventors stored in the `inventors` array column.

---

# Source Table

## Query

```sql
SELECT * FROM patents.patent_inventor_array LIMIT 5;
```

> **Output:**

<img width="609" height="181" alt="Screenshot 2026-07-23 at 12 50 46 PM" src="https://github.com/user-attachments/assets/a545b1dd-acf0-4825-b884-a43a8a070493" />

---

# 1. Using `&&` (Recommended)

## Purpose

Checks whether the `inventors` array contains **at least one** inventor from the search array.

This is the recommended approach because it supports **GIN indexes** and provides the best performance for array searches.

## Query

```sql
SELECT *
FROM patents.patent_inventor_array
WHERE inventors && ARRAY[
    'Inventor_00467',
    'Inventor_01385',
    'Inventor_05942'
]
LIMIT 20;
```

> **Output:**

<img width="634" height="545" alt="Screenshot 2026-07-23 at 1 08 54 PM" src="https://github.com/user-attachments/assets/38fdfa53-f6ad-4e8a-b2ca-d2ca964d9462" />


## Query Explanation

### `inventors && ARRAY[...]`

The `&&` operator checks whether both arrays have **at least one common element**.

Example

```
inventors
{Inventor_00467, Inventor_09265}

Search Array
{Inventor_00467, Inventor_01385, Inventor_05942}
```

Since **Inventor_00467** exists in both arrays, the condition returns **TRUE**.

---

## Advantages

- Fast
- Supports GIN indexes
- Best method for array searches
- Simple and readable

---

# 2. Using Multiple `ANY()` Conditions

## Purpose

Checks whether each inventor exists in the array by comparing one value at a time.

## Query

```sql
SELECT *
FROM patents.patent_inventor_array
WHERE
    'Inventor_00467' = ANY(inventors)
    OR 'Inventor_01385' = ANY(inventors)
    OR 'Inventor_05942' = ANY(inventors)
LIMIT 20;
```

> **Output:**

<img width="625" height="551" alt="Screenshot 2026-07-23 at 1 09 02 PM" src="https://github.com/user-attachments/assets/6a19d7f1-62ce-4615-91fc-239f0fa63d6a" />

## Query Explanation

### `'Inventor_00467' = ANY(inventors)`

Checks whether **Inventor_00467** exists in the inventor array.

---

### `OR`

Returns the patent if **any one** of the conditions is true.

---

### `ANY(inventors)`

Compares the given inventor name with every element in the array.

If a match is found, the condition returns **TRUE**.

---

## Advantages

- Easy to understand
- Good for a small number of search values

## Limitations

- Query becomes longer as more inventors are added.
- Usually slower than `&&` for multiple values.

---

# 3. Using `UNNEST()` with `IN`

## Purpose

Converts each inventor in the array into a separate row and checks whether it matches any inventor in the search list.

## Query

```sql
SELECT DISTINCT p.*
FROM patents.patent_inventor_array p
CROSS JOIN LATERAL UNNEST(p.inventors) AS inventor
WHERE inventor IN (
    'Inventor_00467',
    'Inventor_01385',
    'Inventor_05942'
)
LIMIT 20;
```

> **Output:**

<img width="633" height="572" alt="Screenshot 2026-07-23 at 1 09 13 PM" src="https://github.com/user-attachments/assets/77a6cac8-4657-4838-8812-b3ff58b2e829" />


## Query Explanation

### `UNNEST(p.inventors)`

Splits the inventor array into individual rows.

Example

```
{Inventor_001, Inventor_002, Inventor_003}
```

becomes

```
Inventor_001
Inventor_002
Inventor_003
```

---

### `CROSS JOIN LATERAL`

Executes `UNNEST()` for every row in the table.

---

### `WHERE inventor IN (...)`

Checks whether the inventor is one of the specified inventor names.

---

### `DISTINCT`

Removes duplicate patents because one patent can contain multiple matching inventors.

---

## Advantages

- Easy to understand
- Useful for row-wise analysis
- Helpful when joining with other tables

## Limitations

- Slower because every array element is expanded into a separate row.

---

# 4. Using `EXISTS`

## Purpose

Checks whether at least one inventor in the array matches the search list.

## Query

```sql
SELECT *
FROM patents.patent_inventor_array p
WHERE EXISTS (
    SELECT 1
    FROM UNNEST(p.inventors) AS inventor
    WHERE inventor IN (
        'Inventor_00467',
        'Inventor_01385',
        'Inventor_05942'
    )
)
LIMIT 20;
```

> **Output:**

<img width="651" height="639" alt="Screenshot 2026-07-23 at 1 09 19 PM" src="https://github.com/user-attachments/assets/9d98cabf-07ab-4b6d-b3bf-72102676a070" />

## Query Explanation

### `EXISTS`

Returns **TRUE** as soon as one matching inventor is found.

It does not continue checking the remaining inventors.

---

### `SELECT 1`

Checks only whether a matching row exists.

The value `1` is never returned.

---

### `UNNEST(p.inventors)`

Converts the inventor array into individual rows.

---

### `WHERE inventor IN (...)`

Checks whether the inventor exists in the search list.

---


## Advantages

- Stops searching after finding the first match.
- Useful for more complex filtering conditions.

## Limitations

- Still requires expanding the array.
- Slower than the `&&` operator.

---

# Performance Comparison

| Method | Uses Arrays | Execution Time | Relative Performance |
|----------------------------|:-----------:|---------------:|--------------------:|
| `&&` | Yes | **16.116 ms** | **95–100%** |
| Multiple `ANY()` conditions | Yes | **30.440 ms** | **80–90%** |
| `EXISTS` + `UNNEST()` | Yes | **64.392 ms** | **70–85%** |
| `UNNEST()` + `IN` | Yes | **157.886 ms** | **60–75%** |

---

# Recommendation

- Use `&&` when searching for one or more inventor names in an array. It provides the best performance and supports GIN indexes.
- Use multiple `ANY()` conditions when searching for a small number of inventor names.
- Use `EXISTS` when additional filtering logic is required on individual array elements.
- Use `UNNEST()` with `IN` when inventor names need to be processed as individual rows for analysis or joins.
