# Updating Values in a JSONB Column

## Objective

Update the value of an existing key (`status`) inside the `metadata` JSONB column using different PostgreSQL methods and compare them.

---

# Source Table

## Query

```sql
SELECT * FROM patent_1.patent_metadata_jsonb
WHERE publication_number = 'US0000000001';
```

> **Output:** Attach screenshot here.

---

# 1. Using `jsonb_set()` (Recommended)

## Purpose

Updates the value of an existing key without affecting the remaining JSON data.

## Query

```sql
UPDATE patent_metadata_jsonb
SET metadata = jsonb_set(
    metadata,
    '{status}',
    '"Expired"'
)
WHERE publication_number = 'US0000000001';
```

> **Output:** Attach screenshot here.

## Query Explanation

### `jsonb_set()`

Updates a specific key inside a JSONB document.

---

### `metadata`

The JSONB column to update.

---

### `'{status}'`

Specifies the path of the key to update.

---

### `"Expired"`

The new value assigned to the `status` key.

---

### `WHERE`

Updates only the specified patent.

## Advantages

- Updates only one key.
- Does not affect other keys.
- Best method for updating JSONB values.
- Easy to understand.

---

# 2. Using JSONB Concatenation (`||`)

## Purpose

Updates a key by merging another JSONB object.

If the key already exists, its value is replaced.

## Query

```sql
UPDATE patent_metadata_jsonb
SET metadata = metadata || '{"status":"Expired"}'::jsonb
WHERE publication_number = 'US0000000001';
```

> **Output:** Attach screenshot here.

## Query Explanation

### `||`

Merges two JSONB objects.

---

### `'{"status":"Expired"}'::jsonb`

Creates a new JSONB object containing the updated value.

---

### Merge Operation

If `status` already exists, its value is replaced.

If it does not exist, a new key is added.

## Advantages

- Short and simple.
- Can update multiple keys together.
- Easy to read.

## Limitations

- Replaces the entire key value.
- Cannot update nested values directly.

---

# 3. Using Remove (`-`) and Add (`||`)

## Purpose

Removes the existing key and then adds it back with a new value.

## Query

```sql
UPDATE patent_metadata_jsonb
SET metadata =
    (metadata - 'status')
    || '{"status":"Granted"}'::jsonb
WHERE publication_number = 'US0000000001';
```

> **Output:** Attach screenshot here.

## Query Explanation

### `metadata - 'status'`

Removes the `status` key.

---

### `||`

Adds the new `status` key with its updated value.

---

### Result

The old key is deleted and recreated.

## Advantages

- Useful when replacing an entire key.
- Simple for top-level keys.

## Limitations

- Performs two operations.
- Less efficient than `jsonb_set()`.

---

# Performance Comparison

| Method | Purpose | Updates Nested Key | Relative Performance |
|----------------------|---------------------------------|:------------------:|--------------------:|
| `jsonb_set()` | Update an existing key | Yes | **95–100%** |
| `||` Concatenation | Replace or add key | No | **90–95%** |
| `-` + `||` | Remove and recreate key | No | **80–90%** |

---

# Recommendation

- Use `jsonb_set()` when updating existing JSONB values. It is the safest and most flexible method, especially for nested keys.
- Use the `||` operator when replacing or adding one or more top-level keys.
- Use `-` with `||` only when you need to remove a key before adding it again.
