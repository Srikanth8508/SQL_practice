# PostgreSQL JSON / JSONB Operators Guide

## Sample Setup

```sql
kaggle=> CREATE TABLE customer_orders
(
    order_id INT,
    customer TEXT,
    details  JSONB
);
CREATE TABLE

kaggle=> INSERT INTO customer_orders VALUES
(1, 'Alice', '{"item": "Laptop", "price": 1200, "specs": {"ram": "16GB", "storage": "512GB"}, "tags": ["tech", "work"], "active": true}'),
(2, 'Bob',   '{"item": "Monitor", "price": 300, "specs": {"ram": null, "resolution": "4K"}, "tags": ["tech"], "active": false}'),
(3, 'Charlie', '{"item": "Desk", "price": 150, "specs": {}, "tags": ["home", "furniture"], "active": true}'),
(4, 'David', '{"item": "Keyboard", "price": 80, "specs": {"switches": "Mechanical"}, "tags": ["tech", "gaming"], "active": true}');
INSERT 0 4

kaggle=> SELECT * FROM customer_orders;

```

```text
 order_id | customer |                                                       details                                                       
----------+----------+---------------------------------------------------------------------------------------------------------------------
        1 | Alice    | {"item": "Laptop", "tags": ["tech", "work"], "price": 1200, "active": true, "specs": {"ram": "16GB", "storage": "512GB"}}
        2 | Bob      | {"item": "Monitor", "tags": ["tech"], "price": 300, "active": false, "specs": {"ram": null, "resolution": "4K"}}
        3 | Charlie  | {"item": "Desk", "tags": ["home", "furniture"], "price": 150, "active": true, "specs": {}}
        4 | David    | {"item": "Keyboard", "tags": ["tech", "gaming"], "price": 80, "active": true, "specs": {"switches": "Mechanical"}}
(4 rows)

```

---

# `->` (Get JSON Object Field by Key or Index)

## Purpose

Extracts a field from a JSON object (or array element by index) as a **JSON/JSONB object**.

## Syntax

```sql
json_column -> 'key'         -- Extract object key
json_column -> integer_index -- Extract array element (0-indexed)

```

## Working Example

```sql
kaggle=> SELECT order_id, customer, details -> 'specs' AS specs_json
kaggle-> FROM customer_orders;

```

```text
 order_id | customer |               specs_json               
----------+----------+----------------------------------------
        1 | Alice    | {"ram": "16GB", "storage": "512GB"}
        2 | Bob      | {"ram": null, "resolution": "4K"}
        3 | Charlie  | {}
        4 | David    | {"switches": "Mechanical"}
(4 rows)

```

### Array Indexing Example

```sql
kaggle=> SELECT order_id, customer, details -> 'tags' -> 0 AS first_tag_json
kaggle-> FROM customer_orders;

```

```text
 order_id | customer | first_tag_json 
----------+----------+----------------
        1 | Alice    | "tech"
        2 | Bob      | "tech"
        3 | Charlie  | "home"
        4 | David    | "tech"
(4 rows)

```

### Explanation

Notice the quotes around `"tech"` or the JSON object shape `{}` in the output. The `->` operator returns data as **JSON**, meaning string values remain wrapped in double quotes and JSON formatting is preserved.

---

# `->>` (Get JSON Object Field as Text)

## Purpose

Extracts a field from a JSON object (or array element) as plain **text**.

## Syntax

```sql
json_column ->> 'key'
json_column ->> integer_index

```

## Working Example

```sql
kaggle=> SELECT order_id, customer, details ->> 'item' AS item_name
kaggle-> FROM customer_orders
kaggle-> WHERE details ->> 'item' = 'Laptop';

```

```text
 order_id | customer | item_name 
----------+----------+-----------
        1 | Alice    | Laptop
(1 row)

```

### Evaluation Matrix (`details ->> 'item'`)

```text
 order_id | customer | details -> 'item' (JSON) | details ->> 'item' (Text) | Matches 'Laptop'? 
----------+----------+--------------------------+---------------------------+-------------------
        1 | Alice    | "Laptop"                 | Laptop                    | TRUE
        2 | Bob      | "Monitor"                | Monitor                   | FALSE
        3 | Charlie  | "Desk"                   | Desk                      | FALSE
        4 | David    | "Keyboard"               | Keyboard                  | FALSE

```

### Explanation

`->>` strips away JSON quotes and converts the output to native SQL `TEXT`. You must use `->>` (not `->`) when comparing JSON string fields against SQL text values in a `WHERE` clause.

### Common Mistake

```sql
-- FAILS to match because "Laptop" (JSON string) != Laptop (SQL text)
kaggle=> SELECT * FROM customer_orders WHERE details -> 'item' = 'Laptop';

```

```text
 order_id | customer | details 
----------+----------+---------
(0 rows)

```

---

# `#>` (Get Nested JSON Object at Path)

## Purpose

Navigates down a nested JSON path and returns the value as a **JSON/JSONB object**.

## Syntax

```sql
json_column #> '{path, keys}'

```

## Working Example

```sql
kaggle=> SELECT order_id, customer, details #> '{specs, ram}' AS ram_json
kaggle-> FROM customer_orders;

```

```text
 order_id | customer | ram_json 
----------+----------+----------
        1 | Alice    | "16GB"
        2 | Bob      | null
        3 | Charlie  | 
        4 | David    | 
(4 rows)

```

### Explanation

Returns the targeted property at deep nest levels. Notice how Bob returns explicit JSON `null`, whereas Charlie and David return SQL `NULL` (empty cell) because the `ram` key does not exist in their `specs` object.

---

# `#>>` (Get Nested JSON Field at Path as Text)

## Purpose

Navigates down a nested JSON path and returns the value as plain **text**.

## Syntax

```sql
json_column #>> '{path, keys}'

```

## Working Example

```sql
kaggle=> SELECT order_id, customer, details #>> '{specs, ram}' AS ram_text
kaggle-> FROM customer_orders
kaggle-> WHERE details #>> '{specs, ram}' IS NOT NULL;

```

```text
 order_id | customer | ram_text 
----------+----------+----------
        1 | Alice    | 16GB
(1 row)

```

### Evaluation Matrix (`details #>> '{specs, ram}'`)

```text
 order_id | customer | #> '{specs, ram}' | #>> '{specs, ram}' | Result / Notes 
----------+----------+-------------------+--------------------+---------------------------------
        1 | Alice    | "16GB"            | 16GB               | Evaluates to Text '16GB'
        2 | Bob      | null              | [NULL]             | Converts JSON null to SQL NULL
        3 | Charlie  | [NULL]            | [NULL]             | Path doesn't exist
        4 | David    | [NULL]            | [NULL]             | Path doesn't exist

```

### Explanation

Use `#>>` to extract nested data cleanly as text for filtering, joining, or casting to numeric/date types (e.g., `(details #>> '{price}')::numeric`).

---

# `@>` (JSONB Contains)

## Purpose

Checks if the left JSONB document contains the right JSONB structure (Key-Value pairs or array elements).

## Syntax

```sql
jsonb_column @> '{"key": "value"}'::jsonb

```

## Working Example

```sql
-- Find active orders containing the 'tech' tag
kaggle=> SELECT order_id, customer, details ->> 'item' AS item
kaggle-> FROM customer_orders
kaggle-> WHERE details @> '{"active": true, "tags": ["tech"]}';

```

```text
 order_id | customer |   item   
----------+----------+----------
        1 | Alice    | Laptop
        4 | David    | Keyboard
(2 rows)

```

### Evaluation Matrix

```text
 order_id | customer | Contains {"active": true, "tags": ["tech"]}? | Reason 
----------+----------+----------------------------------------------+----------------------------------
        1 | Alice    | TRUE                                         | Active is true, tags has 'tech'
        2 | Bob      | FALSE                                        | Tags has 'tech', but active=false
        3 | Charlie  | FALSE                                        | Active=true, but missing 'tech'
        4 | David    | TRUE                                         | Active is true, tags has 'tech'

```

### Explanation

The `@>` operator can match multiple nested criteria simultaneously and takes full advantage of **GIN indexes** for high-performance JSONB querying.

---

# `?` (Key Exists)

## Purpose

Checks whether a top-level **string key** or **array element** exists inside the JSONB document.

## Syntax

```sql
jsonb_column ? 'key_or_element'

```

## Working Example

```sql
-- Check if the array inside 'tags' contains the string 'furniture'
kaggle=> SELECT order_id, customer, details -> 'tags' ? 'furniture' AS sells_furniture
kaggle-> FROM customer_orders;

```

```text
 order_id | customer | sells_furniture 
----------+----------+-----------------
        1 | Alice    | FALSE
        2 | Bob      | FALSE
        3 | Charlie  | TRUE
        4 | David    | FALSE
(4 rows)

```

### Explanation

The `?` operator evaluates directly to boolean `TRUE` or `FALSE`. It checks top-level keys inside objects, or direct elements inside arrays.

---

# `?|` (Exist Any)

## Purpose

Returns `TRUE` if **any** of the specified array of keys/strings exist inside the JSONB document.

## Syntax

```sql
jsonb_column ?| array['key1', 'key2']

```

## Working Example

```sql
-- Check if tags contain EITHER 'furniture' OR 'gaming'
kaggle=> SELECT order_id, customer, details ->> 'item' AS item
kaggle-> FROM customer_orders
kaggle-> WHERE details -> 'tags' ?| ARRAY['furniture', 'gaming'];

```

```text
 order_id | customer |   item   
----------+----------+----------
        3 | Charlie  | Desk
        4 | David    | Keyboard
(2 rows)

```

### Evaluation Matrix

```text
 order_id | customer | tags                 | Has 'furniture' OR 'gaming'? 
----------+----------+----------------------+------------------------------
        1 | Alice    | ["tech", "work"]     | FALSE
        2 | Bob      | ["tech"]             | FALSE
        3 | Charlie  | ["home", "furniture"]| TRUE ('furniture')
        4 | David    | ["tech", "gaming"]   | TRUE ('gaming')

```

---

# `?&` (Exist All)

## Purpose

Returns `TRUE` only if **all** of the specified keys/strings exist inside the JSONB document.

## Syntax

```sql
jsonb_column ?& array['key1', 'key2']

```

## Working Example

```sql
-- Find records where tags contain BOTH 'tech' AND 'work'
kaggle=> SELECT order_id, customer, details ->> 'item' AS item
kaggle-> FROM customer_orders
kaggle-> WHERE details -> 'tags' ?& ARRAY['tech', 'work'];

```

```text
 order_id | customer |  item  
----------+----------+--------
        1 | Alice    | Laptop
(1 row)

```

### Failing Example

```sql
-- Bob and David have 'tech', but neither has 'work'
kaggle=> SELECT order_id, customer
kaggle-> FROM customer_orders
kaggle-> WHERE order_id IN (2, 4) 
kaggle->   AND details -> 'tags' ?& ARRAY['tech', 'work'];

```

```text
 order_id | customer 
----------+----------
(0 rows)

```

---

# `-` (Delete Key or Array Element)

## Purpose

Deletes a key (and its value) from a JSON object, or deletes an element from a JSON array by key name or index position.

## Syntax

```sql
jsonb_column - 'key_to_delete'
jsonb_column - integer_index

```

## Working Example (Deleting an Object Key)

```sql
kaggle=> SELECT order_id, details - 'tags' - 'active' AS trimmed_details
kaggle-> FROM customer_orders
kaggle-> WHERE order_id = 1;

```

```text
 order_id |                                   trimmed_details                                   
----------+-------------------------------------------------------------------------------------
        1 | {"item": "Laptop", "price": 1200, "specs": {"ram": "16GB", "storage": "512GB"}}
(1 row)

```

## Working Example (Deleting an Array Element)

```sql
kaggle=> SELECT order_id, details -> 'tags' AS original_tags, (details -> 'tags') - 0 AS tags_after_delete
kaggle-> FROM customer_orders
kaggle-> WHERE order_id = 1;

```

```text
 order_id |   original_tags   | tags_after_delete 
----------+-------------------+-------------------
        1 | ["tech", "work"]  | ["work"]
(1 row)

```

### Explanation

The `-` operator returns a modified JSONB object without altering the original database state (unless combined with an `UPDATE` statement).

---

# `#-` (Delete Field at Path)

## Purpose

Deletes a deeply nested key or array element at a specified target path.

## Syntax

```sql
jsonb_column #- '{path, target}'

```

## Working Example

```sql
kaggle=> SELECT order_id, details #- '{specs, ram}' AS specs_without_ram
kaggle-> FROM customer_orders
kaggle-> WHERE order_id = 1;

```

```text
 order_id |                                                      specs_without_ram                                                      
----------+-----------------------------------------------------------------------------------------------------------------------------
        1 | {"item": "Laptop", "tags": ["tech", "work"], "price": 1200, "active": true, "specs": {"storage": "512GB"}}
(1 row)

```

### Explanation

Notice how `specs` now only contains `"storage": "512GB"`. The targeted `"ram"` key inside the nested object was removed cleanly.

---

# `JSON` vs `JSONB`: Key Comparison & Trade-Offs

PostgreSQL provides two distinct data types for storing JSON data: `JSON` and `JSONB`. The fundamental difference lies in **how they are stored and processed internally**.

* **`JSON`** stores an **exact text copy** of the input string (including whitespace and key order).
* **`JSONB`** stores data in a **decomposed binary format**, stripping extra whitespace and eliminating duplicate keys.

---

## Direct Comparison Matrix

| Feature / Trait | `JSON` (Text Storage) | `JSONB` (Binary Storage) |
| --- | --- | --- |
| **Storage Format** | Plain text string | Decomposed binary format |
| **Insert Speed** | ⚡ **Faster** (No conversion needed) | 🐢 Slower (Parsing overhead on write) |
| **Query Speed** | 🐢 Slower (Re-parsed on *every* read) | ⚡ **Faster** (Direct binary traversal) |
| **Index Support** | Limited (Expression indexes only) | **Full Support** (GIN, btree, hash) |
| **Operator Support** | Basic (`->`, `->>`, `#>`, `#>>`) | **All Operators** (`@>`, `?`, `? |
| **Preserves Whitespace** | ✅ Yes | ❌ No |
| **Preserves Key Order** | ✅ Yes | ❌ No (Keys are re-ordered internally) |
| **Duplicate Keys** | ✅ Keeps duplicate keys | ❌ Keeps only the **last** duplicate key value |

---

## Pros & Cons

### 1. `JSON` (Plain Text)

#### **Pros**

* **Exact Fidelity:** Preserves identical formatting, spacing, and key ordering as inserted.
* **Faster Ingestion:** Excellent for high-throughput write-heavy audit logs or streaming ingestion pipelines where JSON payload processing isn't immediately required.

#### **Cons**

* **Slow Query Processing:** Every query or extraction (`->>`) forces PostgreSQL to re-parse the raw text string from scratch.
* **No Specialized Indexing:** Cannot be indexed with GIN indexes for fast key/value searches.
* **No Advanced Operators:** Specialized JSON operators like containment (`@>`) or key check (`?`) do not work on raw `JSON`.

---

### 2. `JSONB` (Binary Format)

#### **Pros**

* **High Query Performance:** Data is pre-parsed on write, making field extraction and deeply nested queries significantly faster.
* **GIN Indexing:** Supports GIN (Generalized Inverted Index) indexes, enabling millisecond lookup times across millions of records for conditions like `WHERE details @> '{"active": true}'`.
* **Richer Operator Support:** Unlocks the full suit of path manipulation, containment, and search operators.
* **Storage Efficiency:** Strips duplicate keys and unnecessary whitespace, saving space on large datasets.

#### **Cons**

* **Write Overhead:** Parsing, validating, and converting raw JSON into binary format incurs a slight CPU penalty during `INSERT` and `UPDATE` operations.
* **Does Not Preserve Formatting:** Key ordering is altered (PostgreSQL sorts keys by length then alphabetically), and indentations/spaces are discarded.

---

## When to Use Which?

> 💡 **Rule of Thumb:** **Use `JSONB` by default for almost all use cases.**
> Use plain `JSON` only if you have strict compliance requirements to preserve original raw text formatting or if write speed is your sole bottleneck.

```

```
