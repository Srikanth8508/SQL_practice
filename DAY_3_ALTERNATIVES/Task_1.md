# Aggregate Functions for Grouping Multiple Inventors

## Objective

A patent can have multiple inventors. Instead of storing one row per inventor, we can group all inventors belonging to the same patent into a single field using PostgreSQL aggregate functions.

The examples below create different representations of the same data.

---

# Sample Source Table

```sql
SELECT * FROM patents.patent_inventors LIMIT 8;
```

| publication_number | inventor_name |
|--------------------|---------------|
| US100001           | Alice         |
| US100001           | Bob           |
| US100001           | David         |
| US100002           | Charlie       |
| US100002           | Emma          |
| US100003           | Frank         |
| US100003           | Grace         |
| US100003           | Henry         |

Notice that each inventor is stored in a separate row.

---

# 1. ARRAY_AGG() (Recommended)

## Purpose

Collects multiple values into a PostgreSQL array.

This is the most commonly used method when working with PostgreSQL because arrays support many built-in operators and functions.

## Query

```sql
CREATE TABLE patent_1.patent_inventor_array AS
SELECT
    publication_number,
    ARRAY_AGG(inventor_name ORDER BY inventor_name) AS inventors
FROM patents.patent_inventors
GROUP BY publication_number
LIMIT 10000;
```

## Query Explanation

### CREATE TABLE ... AS

Creates a new table and stores the query result.

---

### publication_number

Groups inventors belonging to the same patent.

---

### ARRAY_AGG(inventor_name ORDER BY inventor_name)

Collects all inventor names into one PostgreSQL array.

The `ORDER BY` inside `ARRAY_AGG()` sorts inventor names alphabetically before creating the array.

Without `ORDER BY`, the order may vary.

---

### GROUP BY publication_number

Creates one output row for each patent.

---

### LIMIT 10000

Stores only the first 10,000 grouped patents.

---

## Sample Output

| publication_number | inventors |
|--------------------|--------------------------------|
| US100001 | {Alice,Bob,David} |
| US100002 | {Charlie,Emma} |
| US100003 | {Frank,Grace,Henry} |

---

## Advantages

- Native PostgreSQL array
- Supports array operators
- Fast searching with GIN indexes
- Best choice for PostgreSQL applications

---

# 2. STRING_AGG()

## Purpose

Combines multiple values into a single text string.

Useful for reports and displaying data.

## Query

```sql
CREATE TABLE patent_1.patent_inventor_text AS
SELECT
    publication_number,
    STRING_AGG(inventor_name, ', ' ORDER BY inventor_name) AS inventors
FROM patents.patent_inventors
GROUP BY publication_number
LIMIT 10000;
```

## Query Explanation

### STRING_AGG(inventor_name, ', ' ORDER BY inventor_name)

- Combines inventor names into one text value.
- `', '` is the separator.
- `ORDER BY` sorts inventor names before joining.

---

## Sample Output

| publication_number | inventors |
|--------------------|-------------------------|
| US100001 | Alice, Bob, David |
| US100002 | Charlie, Emma |
| US100003 | Frank, Grace, Henry |

---

## Advantages

- Easy to read
- Suitable for reports
- Good for exporting data

---

## Limitations

- Not suitable for array operators
- Searching individual inventors is difficult

---

# 3. JSON_AGG()

## Purpose

Collects multiple values into a JSON array.

Useful when applications or APIs expect JSON output.

## Query

```sql
CREATE TABLE patent_1.patent_inventor_json AS
SELECT
    publication_number,
    JSON_AGG(inventor_name ORDER BY inventor_name) AS inventors
FROM patents.patent_inventors
GROUP BY publication_number
LIMIT 10000;
```

## Query Explanation

### JSON_AGG(inventor_name ORDER BY inventor_name)

Creates a JSON array containing inventor names.

---

## Sample Output

| publication_number | inventors |
|--------------------|------------------------------------|
| US100001 | ["Alice","Bob","David"] |
| US100002 | ["Charlie","Emma"] |
| US100003 | ["Frank","Grace","Henry"] |

---

## Advantages

- Standard JSON format
- Easy to use in REST APIs
- Supported by most programming languages

---

## Limitations

- Slower than JSONB for searching
- Cannot be indexed efficiently

---

# 4. JSONB_AGG()

## Purpose

Collects multiple values into a JSONB array.

JSONB stores JSON in a binary format that is optimized for PostgreSQL.

## Query

```sql
CREATE TABLE patent_1.patent_inventor_jsonb AS
SELECT
    publication_number,
    JSONB_AGG(inventor_name ORDER BY inventor_name) AS inventors
FROM patents.patent_inventors
GROUP BY publication_number
LIMIT 10000;
```

## Query Explanation

### JSONB_AGG(inventor_name ORDER BY inventor_name)

Creates a JSONB array.

JSONB is stored in a binary format that supports indexing and faster processing.

---

## Sample Output

| publication_number | inventors |
|--------------------|------------------------------------|
| US100001 | ["Alice","Bob","David"] |
| US100002 | ["Charlie","Emma"] |
| US100003 | ["Frank","Grace","Henry"] |

Although the output looks like JSON, PostgreSQL stores it internally as JSONB.

---

## Advantages

- Faster JSON processing
- Supports GIN indexes
- Better for searching
- Recommended over JSON when working inside PostgreSQL

---

# Comparison

| Aggregate Function | Result Type | Example Output |
|--------------------|------------|----------------|
| ARRAY_AGG() | Array | `{Alice,Bob,David}` |
| STRING_AGG() | Text | `Alice, Bob, David` |
| JSON_AGG() | JSON | `["Alice","Bob","David"]` |
| JSONB_AGG() | JSONB | `["Alice","Bob","David"]` |

---

# When to Use Which?

| Aggregate Function | Result Type | Suitable For |
|--------------------|-------------|--------------|
| `ARRAY_AGG()` | PostgreSQL Array | ✅ Best for PostgreSQL array operations |
| `STRING_AGG()` | Text | Reports and display |
| `JSON_AGG()` | JSON | APIs and JSON output |
| `JSONB_AGG()` | JSONB | JSON processing and indexing |
| `XMLAGG()` | XML | XML integrations |

---

# Recommendation

For PostgreSQL applications:

- Use **ARRAY_AGG()** when you need PostgreSQL array operators and functions.
- Use **STRING_AGG()** when you only need a readable text value.
- Use **JSON_AGG()** when returning JSON to applications or APIs.
- Use **JSONB_AGG()** when JSON data needs searching, indexing, or further processing inside PostgreSQL.

For most PostgreSQL workloads, **ARRAY_AGG()** and **JSONB_AGG()** are the preferred choices.
