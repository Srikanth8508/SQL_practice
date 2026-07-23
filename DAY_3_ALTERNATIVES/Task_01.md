# Aggregate Functions for Grouping Multiple Inventors

## Objective

A patent can have multiple inventors. Instead of storing one row per inventor, we can group all inventors belonging to the same patent into a single field using PostgreSQL aggregate functions.

The examples below create different representations of the same data.

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

<img width="727" height="267" alt="Screenshot 2026-07-23 at 11 29 19 AM" src="https://github.com/user-attachments/assets/f99892f3-d91f-41db-96a4-73a99e1f537a" />


### ARRAY_AGG(inventor_name ORDER BY inventor_name)

Collects all inventor names into one PostgreSQL array.

The `ORDER BY` inside `ARRAY_AGG()` sorts inventor names alphabetically before creating the array.

Without `ORDER BY`, the order may vary.

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
<img width="708" height="267" alt="Screenshot 2026-07-23 at 11 29 40 AM" src="https://github.com/user-attachments/assets/e727b8fd-171f-4549-9b2c-d822c4b9ae4f" />


## Query Explanation

### STRING_AGG(inventor_name, ', ' ORDER BY inventor_name)

- Combines inventor names into one text value.
- `', '` is the separator.
- `ORDER BY` sorts inventor names before joining.

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

<img width="778" height="264" alt="Screenshot 2026-07-23 at 11 29 24 AM" src="https://github.com/user-attachments/assets/8875f2e8-42c6-4c13-bf28-d03ab47d2772" />


## Query Explanation

### JSON_AGG(inventor_name ORDER BY inventor_name)

Creates a JSON array containing inventor names.

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

<img width="771" height="266" alt="Screenshot 2026-07-23 at 11 29 33 AM" src="https://github.com/user-attachments/assets/47426234-29a7-4cbb-8d2f-1403836a5ddb" />

## Query Explanation

### JSONB_AGG(inventor_name ORDER BY inventor_name)

Creates a JSONB array.

JSONB is stored in a binary format that supports indexing and faster processing.

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
