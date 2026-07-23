# Finding Patents by Inventor Name from an Array Column

## Objective

The `inventors` column is stored as a PostgreSQL array (`TEXT[]`). There are multiple ways to check whether a particular inventor exists in the array.

Example inventor:

```text
Inventor_00467
```

---

# Sample Data

```sql
SELECT * FROM patent_1.patent_inventor_array LIMIT 3;
```

| publication_number | inventors |
|--------------------|----------------------------------------------|
| US100001 | {Inventor_00012,Inventor_00467,Inventor_00891} |
| US100002 | {Inventor_00125,Inventor_00345} |
| US100003 | {Inventor_00467,Inventor_00987} |

---

# 1. Using `@>` (Recommended ✅)

## Purpose

Checks whether an array contains another array.

This is the recommended approach because it supports **GIN indexes** and performs well on large tables.

## Query

```sql
SELECT *
FROM patent_1.patent_inventor_array
WHERE inventors @> ARRAY['Inventor_00467']
LIMIT 20;
```
<img width="500" height="194" alt="Screenshot 2026-07-23 at 12 25 05 PM" src="https://github.com/user-attachments/assets/ade8451b-5bd0-4d74-8ebe-5dcfc88b7a33" />


### `@>`

The **contains** operator.

Returns `TRUE` if the left array contains all elements of the right array.

### `ARRAY['Inventor_00467']`

Creates an array containing one inventor.

---

## Advantages

- Fast with GIN indexes
- Recommended for array searches
- Supports searching for one or multiple values

---

# 2. Using `ANY()`

## Purpose

Checks whether a value exists in an array.

## Query

```sql
SELECT *
FROM patent_1.patent_inventor_array
WHERE 'Inventor_00467' = ANY(inventors)
LIMIT 20;
```
<img width="502" height="180" alt="Screenshot 2026-07-23 at 12 25 11 PM" src="https://github.com/user-attachments/assets/b8c899cd-3c8d-4677-b9bf-8f5b1d3e24cc" />


### `ANY(inventors)`

Compares the given value with every element in the array.

Returns `TRUE` if any element matches.

---

## Advantages

- Easy to read
- Good for checking a single value

---

## Limitations

- Does not use array containment logic
- Usually slower than `@>` on indexed array columns

---

# 3. Using `&&` (Overlap Operator)

## Purpose

Checks whether two arrays have at least one common element.

## Query

```sql
SELECT *
FROM patent_1.patent_inventor_array
WHERE inventors && ARRAY[
    'Inventor_00467',
    'Inventor_00891',
    'Inventor_01000'
];
```
<img width="518" height="169" alt="Screenshot 2026-07-23 at 12 25 16 PM" src="https://github.com/user-attachments/assets/e7d26448-d90c-450b-8543-6688e9209885" />

### `&&`

Returns patents containing **any one** of these inventors.

---

## Advantages

- Fast with GIN indexes
- Best when searching for multiple possible values

---

# 4. Using `UNNEST()`

## Purpose

Converts every array element into a separate row before filtering.

## Query

```sql
SELECT DISTINCT p.*
FROM patent_1.patent_inventor_array p
CROSS JOIN LATERAL UNNEST(inventors) AS inventor
WHERE inventor = 'Inventor_00467';
```
<img width="516" height="147" alt="Screenshot 2026-07-23 at 12 25 26 PM" src="https://github.com/user-attachments/assets/06808e15-7b65-4b37-bd96-3ad246ec0f2f" />


## Query Explanation

### `UNNEST(inventors)`

Splits the array into individual rows.

### `CROSS JOIN LATERAL`

Runs `UNNEST()` separately for each row.

### `WHERE inventor = 'Inventor_00467'`

Keeps only rows containing the required inventor.

### `DISTINCT`

Removes duplicate patent rows after unnesting.

---

## Advantages

- Flexible
- Useful for joins and detailed analysis

---

## Limitations

- Slower than array operators
- Expands every array into multiple rows

---

# 5. Using `ARRAY_POSITION()`

## Purpose

Returns the position of a value inside an array.

If the value is not found, it returns `NULL`.

## Query

```sql
SELECT *
FROM patent_1.patent_inventor_array
WHERE ARRAY_POSITION(inventors, 'Inventor_00467') IS NOT NULL;
```
<img width="500" height="163" alt="Screenshot 2026-07-23 at 12 25 32 PM" src="https://github.com/user-attachments/assets/95f8029b-72d0-4847-9f18-dafbcb7a06c3" />

## Query Explanation

### `ARRAY_POSITION(inventors, 'Inventor_00467')`

Returns the position of the inventor.

If the inventor does not exist:

```
NULL
```

The `WHERE` clause keeps only rows where the position is not `NULL`.

---

## Advantages

- Can determine the element position
- Useful when order matters

---

## Limitations

- Not intended for high-performance searching
- Generally slower than `@>` on indexed arrays

---

## Performance Comparison

| Method | Search Type | GIN Index Support | Execution Time | Relative Performance |
|--------|-------------|:-----------------:|---------------:|:--------------------:|
| `ANY()` | Single value lookup | No | 5.351 ms | 100% |
| `&&` | Match any value from multiple elements | Yes | 9.497 ms | 56% |
| `ARRAY_POSITION()` | Find the position of an element in an array | No | 11.825 ms | 45% |
| `@>` | Check whether an array contains one or more specified values | Yes | 21.397 ms | 25% |
| `UNNEST()` | Expand array elements into rows for filtering and analysis | No | 22.955 ms | 23% |

---

# Recommendation

- Use **`@>`** for most array searches because it is efficient and supports GIN indexes.
- Use **`ANY()`** for simple membership checks when indexing is not important.
- Use **`&&`** when searching for **any one** of multiple values.
- Use **`UNNEST()`** when you need to work with individual array elements or join them with other tables.
- Use **`ARRAY_POSITION()`** when you also need to know the location of an element within the array.
