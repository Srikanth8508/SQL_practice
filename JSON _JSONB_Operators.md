# PostgreSQL JSON / JSONB Operators Guide

## Sample Setup

```sql
 CREATE TABLE customer_orders
(
    order_id INT,
    customer TEXT,
    details  JSONB
);
CREATE TABLE

 INSERT INTO customer_orders VALUES
(1, 'Alice', '{"item": "Laptop", "price": 1200, "specs": {"ram": "16GB", "storage": "512GB"}, "tags": ["tech", "work"], "active": true}'),
(2, 'Bob',   '{"item": "Monitor", "price": 300, "specs": {"ram": null, "resolution": "4K"}, "tags": ["tech"], "active": false}'),
(3, 'Charlie', '{"item": "Desk", "price": 150, "specs": {}, "tags": ["home", "furniture"], "active": true}'),
(4, 'David', '{"item": "Keyboard", "price": 80, "specs": {"switches": "Mechanical"}, "tags": ["tech", "gaming"], "active": true}');
INSERT 0 4

 SELECT * FROM customer_orders;

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

# `->` (Get JSON Object Field or Array Element)

## Purpose

Extracts a field from a JSON object by **key name** or an array element by **position index** as a **JSON/JSONB object**.

> ⚠️ **Key Concept:** The `->` operator returns data in **JSON format**, keeping string values wrapped in double quotes (`"value"`). Because it returns JSON, you can chain multiple `->` operators together to navigate deeper into nested structures.

---

## Syntax

```sql
json_column -> 'key'          -- Extract object field by key name
json_column -> integer_index  -- Extract array element by position (0-indexed, negative integers count backward)

```

---

## Working Example: Extracting Object Keys

```sql
 SELECT order_id, customer, details -> 'specs' AS specs_json
 FROM customer_orders;

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

---

## Working Example: Extracting Array Elements by Position (Index)

JSON arrays use **0-based indexing** (the first item is at position `0`):

| Index | Position | Description |
| --- | --- | --- |
| **`0`** | 1st Element | First item in the array |
| **`1`** | 2nd Element | Second item in the array |
| **`-1`** | Last Element | Negative numbers count **backward** from the end |

```sql
 SELECT 
   order_id, 
   details -> 'tags' AS full_array,
   details -> 'tags' -> 0  AS first_tag,   -- Position 0 (1st item)
   details -> 'tags' -> 1  AS second_tag,  -- Position 1 (2nd item)
   details -> 'tags' -> -1 AS last_tag     -- Position -1 (Last item)
 FROM customer_orders;

```

```text
 order_id |       full_array       | first_tag | second_tag | last_tag 
----------+------------------------+-----------+------------+----------
        1 | ["tech", "work"]       | "tech"    | "work"     | "work"
        2 | ["tech"]               | "tech"    | NULL       | "tech"
        3 | ["home", "furniture"]  | "home"    | "furniture"| "furniture"
        4 | ["tech", "gaming"]     | "tech"    | "gaming"   | "gaming"
(4 rows)

```

### Explanation

Notice the double quotes around `"tech"` or `"work"` in the output. The `->` operator retains JSON type-formatting. If an array position index is out of bounds (such as requesting index `1` for Bob), SQL returns `NULL`.

---

# `->>` (Get JSON Field or Array Element as Text)

## Purpose

Extracts a JSON object field or array element directly as plain **SQL text**.

## Syntax

```sql
json_column ->> 'key'          -- Extract object field as text
json_column ->> integer_index  -- Extract array element as text by position

```

---

## Working Example: Extracting Object Keys as Text

```sql
 SELECT order_id, customer, details ->> 'item' AS item_name
 FROM customer_orders
 WHERE details ->> 'item' = 'Laptop';

```

```text
 order_id | customer | item_name 
----------+----------+-----------
        1 | Alice    | Laptop
(1 row)

```

---

## Working Example: Extracting Array Positions as Text

```sql
 SELECT 
   order_id, 
   customer, 
   details -> 'tags' ->> 0 AS first_tag_text
 FROM customer_orders
 WHERE details -> 'tags' ->> 0 = 'tech';

```

```text
 order_id | customer | first_tag_text 
----------+----------+----------------
        1 | Alice    | tech
        2 | Bob      | tech
        4 | David    | tech
(3 rows)

```

---

## Evaluation Matrix (`details -> 'item'` vs `details ->> 'item'`)

```text
 order_id | customer | details -> 'item' (JSON) | details ->> 'item' (Text) | Matches 'Laptop'? 
----------+----------+--------------------------+---------------------------+-------------------
        1 | Alice    | "Laptop"                 | Laptop                    | TRUE
        2 | Bob      | "Monitor"                | Monitor                   | FALSE
        3 | Charlie  | "Desk"                   | Desk                      | FALSE
        4 | David    | "Keyboard"               | Keyboard                  | FALSE

```

---

## Side-by-Side Comparison: `->` vs `->>` in `WHERE` Clauses

### 1. Using `->` (JSON Output)

Because `->` outputs a **JSON object**, strings remain double-quoted (`"Laptop"`). Comparing `->` to plain SQL text fails unless you explicitly include double quotes inside single quotes (`'"Laptop"'`).

```sql
-- ❌ FAILS (0 rows returned) because JSON "Laptop" != SQL Text Laptop
 SELECT * FROM customer_orders WHERE details -> 'item' = 'Laptop';

```

```text
 order_id | customer | details 
----------+----------+---------
(0 rows)

```

```sql
-- ✅ WORKS (but requires escaping quotes in your query)
 SELECT order_id, customer FROM customer_orders WHERE details -> 'item' = '"Laptop"';

```

```text
 order_id | customer 
----------+----------
        1 | Alice
(1 row)

```

### 2. Using `->>` (Text Output)

The `->>` operator strips away JSON quotes and outputs native **SQL text**, allowing clean comparisons.

```sql
-- ✅ WORKS directly with standard SQL strings
 SELECT order_id, customer FROM customer_orders WHERE details ->> 'item' = 'Laptop';

```

```text
 order_id | customer 
----------+----------
        1 | Alice
(1 row)

```
---
# `#>` (Get Nested JSON Object at Path)

## Purpose

The `#>` operator retrieves a value from a **nested JSON path** and returns it as **JSON/JSONB**.

It is useful when the value is inside multiple levels of a JSON object.

---

## Syntax

```sql
json_column #> '{key1,key2,...}'
```

---

## Working Example

```sql
SELECT
    order_id,
    customer,
    details #> '{specs,ram}' AS ram_json
FROM customer_orders;
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

---

## Equivalent Query Using `->`

```sql
SELECT
    order_id,
    customer,
    details -> 'specs' -> 'ram' AS ram_json
FROM customer_orders;
```

The output is exactly the same.

---

## Explanation

Both queries return the same JSON value.

The only difference is how the path is written.

| Operator | Navigation Style |
|----------|------------------|
| `->` | One level at a time |
| `#>` | Entire path in one operator |

Example:

```sql
details -> 'specs' -> 'ram'
```

is equivalent to

```sql
details #> '{specs,ram}'
```

---

# `#>>` (Get Nested JSON Field as Text)

## Purpose

The `#>>` operator retrieves a value from a **nested JSON path** and returns it as **plain text**.

---

## Syntax

```sql
json_column #>> '{key1,key2,...}'
```

---

## Working Example

```sql
SELECT
    order_id,
    customer,
    details #>> '{specs,ram}' AS ram_text
FROM customer_orders;
```

```text
 order_id | customer | ram_text
----------+----------+----------
        1 | Alice    | 16GB
        2 | Bob      |
        3 | Charlie  |
        4 | David    |
(4 rows)
```

---

## Equivalent Query Using `->>`

```sql
SELECT
    order_id,
    customer,
    details -> 'specs' ->> 'ram' AS ram_text
FROM customer_orders;
```

The output is exactly the same.

---

## Explanation

Both queries return the same text value.

The only difference is how the path is written.

| Operator | Navigation Style |
|----------|------------------|
| `->>` | One level at a time |
| `#>>` | Entire path in one operator |

Example:

```sql
details -> 'specs' ->> 'ram'
```

is equivalent to

```sql
details #>> '{specs,ram}'
```

---

# Comparison of All Four Operators

| Operator | Returns | Used For | Example |
|----------|----------|----------|---------|
| `->` | JSON | Access one key/index at a time | `details -> 'specs' -> 'ram'` |
| `#>` | JSON | Access a nested path in one operator | `details #> '{specs,ram}'` |
| `->>` | TEXT | Access one key/index at a time and return text | `details -> 'specs' ->> 'ram'` |
| `#>>` | TEXT | Access a nested path in one operator and return text | `details #>> '{specs,ram}'` |

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
 SELECT order_id, customer, details ->> 'item' AS item
 FROM customer_orders
 WHERE details @> '{"active": true, "tags": ["tech"]}';

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
SELECT order_id, customer, details -> 'tags' ? 'furniture' AS sells_furniture
FROM customer_orders;

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

### Example 2: WHERE Clause Filtering

```sql
-- Filter rows where 'tags' array contains the string 'furniture'
SELECT order_id, customer, details -> 'tags' AS tags
FROM customer_orders
WHERE details -> 'tags' ? 'furniture';

```

```text
 order_id | customer |         tags          
----------+----------+-----------------------
        3 | Charlie  | ["home", "furniture"]
(1 row)

```

---

## Explanation & Key Notes

1. **Boolean Output:** The `?` operator evaluates directly to boolean `TRUE` (`t`) or `FALSE` (`f`), making it ideal for `WHERE` filters and boolean flag columns in `SELECT`.
2. **Targets:** It checks top-level keys inside JSON objects or direct string elements inside JSON arrays.
3. **Data Type Requirement:** The right-hand side of `?` **must be a single text string** (e.g., `'furniture'`).

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
 SELECT order_id, customer, details ->> 'item' AS item
 FROM customer_orders
 WHERE details -> 'tags' ?| ARRAY['furniture', 'gaming'];

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
 SELECT order_id, customer, details ->> 'item' AS item
 FROM customer_orders
 WHERE details -> 'tags' ?& ARRAY['tech', 'work'];

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
 SELECT order_id, customer
 FROM customer_orders
 WHERE order_id IN (2, 4) 
   AND details -> 'tags' ?& ARRAY['tech', 'work'];

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
 SELECT order_id, details - 'tags' - 'active' AS trimmed_details
 FROM customer_orders
 WHERE order_id = 1;

```

```text
 order_id |                                   trimmed_details                                   
----------+-------------------------------------------------------------------------------------
        1 | {"item": "Laptop", "price": 1200, "specs": {"ram": "16GB", "storage": "512GB"}}
(1 row)

```

## Working Example (Deleting an Array Element)

```sql
 SELECT order_id, details -> 'tags' AS original_tags, (details -> 'tags') - 0 AS tags_after_delete
 FROM customer_orders
 WHERE order_id = 1;

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
 SELECT order_id, details #- '{specs, ram}' AS specs_without_ram
 FROM customer_orders
 WHERE order_id = 1;

```

```text
 order_id |                                                      specs_without_ram                                                      
----------+-----------------------------------------------------------------------------------------------------------------------------
        1 | {"item": "Laptop", "tags": ["tech", "work"], "price": 1200, "active": true, "specs": {"storage": "512GB"}}
(1 row)

```

### Explanation

Notice how `specs` now only contains `"storage": "512GB"`. The targeted `"ram"` key inside the nested object was removed cleanly.
Here is a detailed guide for PostgreSQL **JSON and JSONB operators**, complete with working SQL, accurate ASCII outputs, evaluation matrices, and explanations.

---

# PostgreSQL JSON & JSONB Operators Guide

## Sample Setup

```sql
 CREATE TABLE customer_orders
(
    order_id INT,
    customer TEXT,
    details  JSONB
);
CREATE TABLE

 INSERT INTO customer_orders VALUES
(1, 'Alice',   '{"item": "Laptop", "price": 1200, "specs": {"ram": "16GB", "storage": "512GB"}, "tags": ["tech", "work"], "active": true}'),
(2, 'Bob',     '{"item": "Monitor", "price": 300, "specs": {"ram": null, "resolution": "4K"}, "tags": ["tech"], "active": false}'),
(3, 'Charlie', '{"item": "Desk", "price": 150, "specs": {}, "tags": ["home", "furniture"], "active": true}'),
(4, 'David',   '{"item": "Keyboard", "price": 80, "specs": {"switches": "Mechanical"}, "tags": ["tech", "gaming"], "active": true}');
INSERT 0 4

 SELECT * FROM customer_orders;

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

# `->` (Get JSON Object Field or Array Element)

## Purpose

Extracts a field from a JSON object by **key name** or an array element by **index position** as a **JSON/JSONB object**.

## Syntax

```sql
json_column -> 'key'          -- Extract object field
json_column -> integer_index  -- Extract array element (0-indexed, negative numbers count backward)

```

## Working Example (Extract Object Key)

```sql
 SELECT order_id, customer, details -> 'specs' AS specs_json
 FROM customer_orders;

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

## Working Example (Extract Array Position Index)

| Index Position | Array Reference | Description |
| --- | --- | --- |
| **`0`** | 1st Element | First item in the array |
| **`1`** | 2nd Element | Second item in the array |
| **`-1`** | Last Element | Negative numbers count **backward** from the end |

```sql
 SELECT 
   order_id, 
   details -> 'tags' AS full_array,
   details -> 'tags' -> 0  AS first_tag,
   details -> 'tags' -> 1  AS second_tag,
   details -> 'tags' -> -1 AS last_tag
 FROM customer_orders;

```

```text
 order_id |       full_array       | first_tag | second_tag | last_tag 
----------+------------------------+-----------+------------+----------
        1 | ["tech", "work"]       | "tech"    | "work"     | "work"
        2 | ["tech"]               | "tech"    | NULL       | "tech"
        3 | ["home", "furniture"]  | "home"    | "furniture"| "furniture"
        4 | ["tech", "gaming"]     | "tech"    | "gaming"   | "gaming"
(4 rows)

```

### Explanation

The `->` operator retains JSON type-formatting (e.g., strings remain double-quoted as `"tech"`). If an array index is out of bounds (like requesting index `1` for Bob), SQL returns `NULL`.

---

# `->>` (Get JSON Field or Array Element as Text)

## Purpose

Extracts a JSON object field or array element directly as plain **text**.

## Syntax

```sql
json_column ->> 'key'
json_column ->> integer_index

```

## Working Example

```sql
 SELECT order_id, customer, details ->> 'item' AS item_name
 FROM customer_orders
 WHERE details ->> 'item' = 'Laptop';

```

```text
 order_id | customer | item_name 
----------+----------+-----------
        1 | Alice    | Laptop
(1 row)

```

### Evaluation Matrix

```text
 order_id | customer | details -> 'item' (JSON) | details ->> 'item' (Text) | Matches 'Laptop'? 
----------+----------+--------------------------+---------------------------+-------------------
        1 | Alice    | "Laptop"                 | Laptop                    | TRUE
        2 | Bob      | "Monitor"                | Monitor                   | FALSE
        3 | Charlie  | "Desk"                   | Desk                      | FALSE
        4 | David    | "Keyboard"               | Keyboard                  | FALSE

```

### Explanation

`->>` strips quotes from JSON strings and converts numeric or boolean values to native SQL `TEXT`. You must use `->>` when filtering JSON text values in a `WHERE` clause.

---

# `#>` (Get Nested Field at Path)

## Purpose

Navigates down a deeply nested path of keys and/or array indexes, returning the result as a **JSON/JSONB object**.

## Syntax

```sql
json_column #> '{path_array}'

```

## Working Example

```sql
-- Extract 'ram' inside the nested 'specs' object
 SELECT order_id, customer, details #> '{specs, ram}' AS ram_json
 FROM customer_orders;

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

* **Alice (Row 1):** Returns `"16GB"` (JSON string).
* **Bob (Row 2):** Returns explicit JSON `null` because `"ram": null` is explicitly declared.
* **Charlie & David (Rows 3 & 4):** Return SQL `NULL` (empty) because the key does not exist.

---

# `#>>` (Get Nested Field at Path as Text)

## Purpose

Navigates down a nested JSON path and returns the target value as plain **text**.

## Syntax

```sql
json_column #>> '{path_array}'

```

## Working Example

```sql
 SELECT order_id, customer, details #>> '{specs, ram}' AS ram_text
 FROM customer_orders
 WHERE details #>> '{specs, ram}' IS NOT NULL;

```

```text
 order_id | customer | ram_text 
----------+----------+----------
        1 | Alice    | 16GB
(1 row)

```

### Evaluation Matrix

```text
 order_id | customer | #> '{specs, ram}' | #>> '{specs, ram}' | Evaluation / Result 
----------+----------+-------------------+--------------------+-----------------------
        1 | Alice    | "16GB"            | 16GB               | Evaluates as SQL Text
        2 | Bob      | null              | [NULL]             | JSON null -> SQL NULL
        3 | Charlie  | [NULL]            | [NULL]             | Path does not exist
        4 | David    | [NULL]            | [NULL]             | Path does not exist

```

---

# `@>` (JSONB Contains)

## Purpose

Checks if the left JSONB document contains all key-value pairs or array elements specified in the right JSONB operand.

## Syntax

```sql
jsonb_column @> '{"key": "value"}'::jsonb

```

## Working Example

```sql
-- Find orders where active is true AND tags array contains 'tech'
 SELECT order_id, customer, details ->> 'item' AS item
 FROM customer_orders
 WHERE details @> '{"active": true, "tags": ["tech"]}';

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
        2 | Bob      | FALSE                                        | Active is false (mismatch)
        3 | Charlie  | FALSE                                        | Missing 'tech' tag
        4 | David    | TRUE                                         | Active is true, tags has 'tech'

```

---

# `<@` (JSONB Contained By)

## Purpose

Checks if the left JSONB document is entirely subsetted by (contained inside) the right JSONB document.

## Syntax

```sql
jsonb_column <@ '{"key": "value"}'::jsonb

```

## Working Example

```sql
-- Test if an employee's specs object is contained within a maximum configuration limit
 SELECT order_id, customer, details -> 'specs' AS specs
 FROM customer_orders
 WHERE (details -> 'specs') <@ '{"ram": "16GB", "storage": "512GB", "switches": "Mechanical"}'::jsonb;

```

```text
 order_id | customer |               specs               
----------+----------+-----------------------------------
        1 | Alice    | {"ram": "16GB", "storage": "512GB"}
        3 | Charlie  | {}
        4 | David    | {"switches": "Mechanical"}
(3 rows)

```

### Explanation

Returns `TRUE` if **every** key-value pair in the left object exists in the right object. Bob (Row 2) is excluded because his `"resolution": "4K"` key is not part of the allowed reference object.

---

# `?` (Key Exists)

## Purpose

Checks whether a top-level **string key** (in an object) or **string element** (in an array) exists inside a JSONB document.

## Syntax

```sql
jsonb_column ? 'key_or_string'

```

## Working Example

```sql
-- Check if the array at details->'tags' contains the string element 'furniture'
 SELECT order_id, customer, details -> 'tags' ? 'furniture' AS sells_furniture
 FROM customer_orders;

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

---

# `?|` (Exist Any)

## Purpose

Returns `TRUE` if **any** of the strings in the given array exist as top-level keys or array elements in the JSONB document.

## Syntax

```sql
jsonb_column ?| ARRAY['key1', 'key2']

```

## Working Example

```sql
 SELECT order_id, customer, details ->> 'item' AS item
 FROM customer_orders
 WHERE details -> 'tags' ?| ARRAY['furniture', 'gaming'];

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
 order_id | customer | tags                  | Has 'furniture' OR 'gaming'? 
----------+----------+-----------------------+------------------------------
        1 | Alice    | ["tech", "work"]      | FALSE
        2 | Bob      | ["tech"]              | FALSE
        3 | Charlie  | ["home", "furniture"] | TRUE ('furniture')
        4 | David    | ["tech", "gaming"]    | TRUE ('gaming')

```

---

# `?&` (Exist All)

## Purpose

Returns `TRUE` only if **all** of the specified strings exist as top-level keys or array elements in the JSONB document.

## Syntax

```sql
jsonb_column ?& ARRAY['key1', 'key2']

```

## Working Example

```sql
 SELECT order_id, customer, details ->> 'item' AS item
 FROM customer_orders
 WHERE details -> 'tags' ?& ARRAY['tech', 'work'];

```

```text
 order_id | customer |  item  
----------+----------+--------
        1 | Alice    | Laptop
(1 row)

```

### Failing Example

```sql
-- Searching for tags containing BOTH 'tech' and 'gaming'
 SELECT order_id, customer
 FROM customer_orders
 WHERE details -> 'tags' ?& ARRAY['tech', 'gaming'] AND order_id = 1;

```

```text
 order_id | customer 
----------+----------
(0 rows)

```

---

# `||` (Concatenate / Merge)

## Purpose

Merges two JSONB objects or concatenates two JSONB arrays together.

## Syntax

```sql
jsonb_column || jsonb_operand

```

## Working Example

```sql
 SELECT order_id, customer, details || '{"shipped": true}'::jsonb AS updated_details
 FROM customer_orders
 WHERE order_id = 1;

```

```text
 order_id | customer |                                                           updated_details                                                           
----------+----------+-------------------------------------------------------------------------------------------------------------------------------------
        1 | Alice    | {"item": "Laptop", "tags": ["tech", "work"], "price": 1200, "active": true, "specs": {"ram": "16GB", "storage": "512GB"}, "shipped": true}
(1 row)

```

---

# `-` (Delete Key or Array Element)

## Purpose

Deletes an object key or array element from a JSONB document by name or index.

## Syntax

```sql
jsonb_column - 'key_or_element'  -- Delete object key or matching string element
jsonb_column - integer_index     -- Delete array element by index position

```

## Working Example (Deleting Object Keys)

```sql
 SELECT order_id, details - 'tags' - 'active' AS trimmed_details
 FROM customer_orders
 WHERE order_id = 1;

```

```text
 order_id |                                   trimmed_details                                   
----------+-------------------------------------------------------------------------------------
        1 | {"item": "Laptop", "price": 1200, "specs": {"ram": "16GB", "storage": "512GB"}}
(1 row)

```

## Working Example (Deleting Array Elements by Index)

```sql
 SELECT order_id, details -> 'tags' AS original_tags, (details -> 'tags') - 0 AS modified_tags
 FROM customer_orders
 WHERE order_id = 1;

```

```text
 order_id |   original_tags   | modified_tags 
----------+-------------------+---------------
        1 | ["tech", "work"]  | ["work"]
(1 row)

```

---

# `#-` (Delete Field at Path)

## Purpose

Deletes a nested field or array element at a specified path array.

## Syntax

```sql
jsonb_column #- '{path_array}'

```

## Working Example

```sql
 SELECT order_id, details #- '{specs, ram}' AS specs_without_ram
 FROM customer_orders
 WHERE order_id = 1;

```

```text
 order_id |                                                      specs_without_ram                                                      
----------+-----------------------------------------------------------------------------------------------------------------------------
        1 | {"item": "Laptop", "tags": ["tech", "work"], "price": 1200, "active": true, "specs": {"storage": "512GB"}}
(1 row)

```

### Explanation

Notice that inside `specs`, `"ram": "16GB"` was removed while preserving `"storage": "512GB"`.


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
