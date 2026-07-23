# PostgreSQL - Generating Hierarchical JSON (Year → Patents → Patent Details → Inventors)

## Objective

Generate a hierarchical JSON structure in the following format:

```json
{
  "year": 2022,
  "patents": [
    {
      "publication_number": "US000000001",
      "title": "Artificial Intelligence System",
      "inventors": [
        "John",
        "David"
      ]
    }
  ]
}
```

---

# Why ROW_NUMBER() is Used?

All the queries below use the same logic:

```sql
ROW_NUMBER() OVER
(
    PARTITION BY EXTRACT(YEAR FROM publication_date)
    ORDER BY publication_number
)
```

### Purpose

Assigns a sequential number to patents **within each publication year**.

Example

| Year | Patent | Row Number |
|------|---------|-----------|
|2020|US001|1|
|2020|US002|2|
|2020|US003|3|
|2021|US010|1|
|2021|US011|2|

Later,

```sql
WHERE rn <= 10
```

returns only the **first 10 patents for every year**, not the first 10 patents from the entire table.

---

# Option 1 (Recommended)

## jsonb_build_object() + jsonb_agg()

### Description

Uses PostgreSQL JSONB functions to build the complete hierarchy.

- `jsonb_build_object()` creates JSON objects.
- `jsonb_agg()` creates JSON arrays.
- Inventors are also stored as JSON arrays.

### Advantages

- Recommended PostgreSQL approach.
- Produces fully structured JSONB.
- Easy to update using JSONB operators.
- Supports indexing with GIN indexes.
- Better for querying nested JSON.
- Faster for manipulation compared to JSON.

### Limitations

- JSONB consumes slightly more storage than JSON.
- Formatting is normalized (key order is not preserved).
- Slightly higher conversion cost during creation.

### Best Use Case

Production applications where JSON will be searched, filtered, or updated later.

---

# Option 2

## json_build_object() + json_agg()

### Description

Same hierarchy as Option 1, but produces **JSON** instead of **JSONB**.

### Advantages

- Simple to understand.
- Preserves original formatting.
- Slightly faster when only generating JSON output.
- Good for API responses.

### Limitations

- Cannot efficiently use JSONB operators.
- No GIN indexing support.
- Slower for searching and updating nested values.

### Best Use Case

Returning JSON directly to client applications without modifying it later.

---

# Option 3

## array_agg()

### Description

Inventors are collected using PostgreSQL arrays instead of JSON arrays.

Example

```text
{"John","David","Steve"}
```

### Advantages

- Very simple syntax.
- Efficient aggregation.
- Arrays can still be converted to JSON automatically.
- Useful when inventors are already stored as arrays.

### Limitations

- PostgreSQL array format is database-specific.
- Less portable than JSON arrays.
- Not suitable if consumers expect standard JSON everywhere.

### Best Use Case

Internal PostgreSQL processing where arrays are frequently used.

---

# Option 4

## string_agg()

### Description

Inventors are combined into one comma-separated string.

Example

```text
John, David, Steve
```

### Advantages

- Very compact output.
- Easy for reports.
- Human readable.
- Smallest output size.

### Limitations

- Loses array structure.
- Cannot easily extract individual inventors.
- Difficult to search specific inventors later.
- Requires string splitting if array is needed.

### Best Use Case

Reports, dashboards, exports, CSV generation.

---

# Option 5

## row_to_json()

### Description

Automatically converts an entire SQL row into a JSON object.

Instead of writing

```sql
json_build_object
(
    'publication_number',
    publication_number,
    'title',
    title
)
```

you simply convert the whole row.

### Advantages

- Less manual coding.
- Automatically includes all selected columns.
- Easy to maintain.
- Useful when many columns exist.

### Limitations

- Less control over JSON field names.
- Includes every selected column.
- Harder to customize compared to json_build_object().

### Best Use Case

Large tables where many columns must be converted into JSON.

---

# Option 6

## to_jsonb()

### Description

Converts an entire PostgreSQL row into JSONB and then appends additional fields.

Example

```sql
to_jsonb(p)
||
jsonb_build_object(...)
```

### Advantages

- Very little manual coding.
- Automatically converts complete rows.
- Easy to append new JSON fields.
- Excellent for dynamic JSON generation.

### Limitations

- Converts every selected column.
- May include unwanted fields.
- Slightly larger JSON if unnecessary columns exist.

### Best Use Case

Dynamic APIs where entire rows need to become JSON documents.

---

# Function Comparison

| Function | Output Type | Purpose |
|-----------|-------------|---------|
|jsonb_build_object()|JSONB Object|Create JSONB objects manually|
|json_build_object()|JSON Object|Create JSON objects manually|
|jsonb_agg()|JSONB Array|Aggregate rows into JSONB array|
|json_agg()|JSON Array|Aggregate rows into JSON array|
|array_agg()|PostgreSQL Array|Aggregate values into arrays|
|string_agg()|Text|Combine values into a single string|
|row_to_json()|JSON Object|Convert entire row to JSON|
|to_jsonb()|JSONB Object|Convert entire row to JSONB|

---

# Overall Comparison

| Option | Main Functions | Inventor Format | Readability | Flexibility | Query Performance | Best For |
|---------|----------------|-----------------|-------------|-------------|-------------------|----------|
|1|jsonb_build_object() + jsonb_agg()|JSON Array|★★★★★|★★★★★|★★★★★|Production applications (Recommended)|
|2|json_build_object() + json_agg()|JSON Array|★★★★★|★★★★☆|★★★★☆|API responses|
|3|array_agg()|PostgreSQL Array|★★★★☆|★★★☆☆|★★★★☆|Internal PostgreSQL processing|
|4|string_agg()|Comma-separated Text|★★★★★|★★☆☆☆|★★★★★|Reports and exports|
|5|row_to_json()|JSON Object|★★★★☆|★★★★☆|★★★★☆|Large tables with many columns|
|6|to_jsonb()|JSONB Object|★★★★☆|★★★★★|★★★★★|Dynamic JSON document generation|

---

# Advantages Summary

| Option | Major Advantage |
|---------|-----------------|
|Option 1|Best overall balance of performance, flexibility, and JSON querying.|
|Option 2|Simple JSON generation for API responses.|
|Option 3|Efficient PostgreSQL array aggregation.|
|Option 4|Most compact and human-readable output.|
|Option 5|Automatically converts rows to JSON with less code.|
|Option 6|Automatically converts rows to JSONB and easily appends additional fields.|

---

# Limitations Summary

| Option | Major Limitation |
|---------|------------------|
|Option 1|Slightly larger storage due to JSONB.|
|Option 2|No JSONB indexing or advanced JSONB operators.|
|Option 3|Database-specific array format.|
|Option 4|Array structure is lost.|
|Option 5|Less control over generated JSON fields.|
|Option 6|May include unnecessary columns if not filtered.|

---

# Recommendation

| Scenario | Recommended Option |
|----------|--------------------|
|Production systems|✅ Option 1|
|REST API output|✅ Option 2|
|PostgreSQL internal processing|✅ Option 3|
|Reports / CSV exports|✅ Option 4|
|Large tables with many columns|✅ Option 5|
|Dynamic JSON document creation|✅ Option 6|

---

## Final Recommendation

For most PostgreSQL applications, **Option 1 (`jsonb_build_object()` + `jsonb_agg()`)** is the preferred approach because it provides:

- Excellent performance
- Fully structured JSONB
- Support for JSONB operators
- GIN indexing
- Easy updates to nested values
- Better long-term maintainability
