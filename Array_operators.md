# PostgreSQL ARRAY Operators Guide

## Sample Setup

```sql
kaggle=> CREATE TABLE employee_skills
(
    emp_id   INT,
    emp_name TEXT,
    skills   TEXT[]
);
CREATE TABLE

kaggle=> INSERT INTO employee_skills VALUES
(101, 'John',  ARRAY['SQL','Python','AWS']),
(102, 'Alice', ARRAY['Python','Java']),
(103, 'David', ARRAY['SQL','Docker']),
(104, 'Bob',   ARRAY['AWS','Linux']),
(105, 'Sam',   ARRAY['Python','SQL','Docker']);
INSERT 0 5

kaggle=> SELECT * FROM employee_skills;

```

```text
 emp_id | emp_name |        skills        
--------+----------+----------------------
    101 | John     | {SQL,Python,AWS}
    102 | Alice    | {Python,Java}
    103 | David    | {SQL,Docker}
    104 | Bob      | {AWS,Linux}
    105 | Sam      | {Python,SQL,Docker}
(5 rows)

```

---

# `=` (Equal)

## Purpose

Checks whether two arrays are exactly identical in elements, order, and length.

## Syntax

```sql
array1 = array2

```

## Working Example

```sql
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE skills = ARRAY['Python','Java'];

```

```text
 emp_id | emp_name |    skills     
--------+----------+---------------
    102 | Alice    | {Python,Java}
(1 row)

```

### Evaluation Matrix

```text
 emp_id |        skills        | Result 
--------+----------------------+--------
    101 | {SQL,Python,AWS}     | FALSE
    102 | {Python,Java}        | TRUE
    103 | {SQL,Docker}         | FALSE
    104 | {AWS,Linux}          | FALSE
    105 | {Python,SQL,Docker}  | FALSE

```

### Explanation

Returns `TRUE` only when both arrays contain the same elements in the exact same order.

### Failing Example (Order Sensitivity)

```sql
-- Same elements as Alice, but swapped order
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE skills = ARRAY['Java','Python'];

```

```text
 emp_id | emp_name | skills 
--------+----------+--------
(0 rows)

```

---

# `<>` (Not Equal)

## Purpose

Returns `TRUE` when two arrays are not identical (differing in elements, element order, or length).

## Syntax

```sql
array1 <> array2

```

## Working Example

```sql
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE skills <> ARRAY['Python','Java'];

```

```text
 emp_id | emp_name |        skills        
--------+----------+----------------------
    101 | John     | {SQL,Python,AWS}
    103 | David    | {SQL,Docker}
    104 | Bob      | {AWS,Linux}
    105 | Sam      | {Python,SQL,Docker}
(4 rows)

```

### Evaluation Matrix

```text
 emp_id |        skills        | Result 
--------+----------------------+--------
    101 | {SQL,Python,AWS}     | TRUE
    102 | {Python,Java}        | FALSE
    103 | {SQL,Docker}         | TRUE
    104 | {AWS,Linux}          | TRUE
    105 | {Python,SQL,Docker}  | TRUE

```

### Explanation

Returns `TRUE` for every row except the one where the array matches completely.

### Failing Example

```sql
-- Evaluates to FALSE for John because the array is an exact match
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE emp_id = 101 AND skills <> ARRAY['SQL','Python','AWS'];

```

```text
 emp_id | emp_name | skills 
--------+----------+--------
(0 rows)

```

---

# `@>` (Contains)

## Purpose

Returns `TRUE` when the left array contains **all** elements specified in the right array. Order does not matter.

## Syntax

```sql
array1 @> array2

```

## Working Example

```sql
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE skills @> ARRAY['SQL'];

```

```text
 emp_id | emp_name |        skills        
--------+----------+----------------------
    101 | John     | {SQL,Python,AWS}
    103 | David    | {SQL,Docker}
    105 | Sam      | {Python,SQL,Docker}
(3 rows)

```

### Evaluation Matrix

```text
 emp_id |        skills        | Contains ['SQL']? 
--------+----------------------+-------------------
    101 | {SQL,Python,AWS}     | TRUE
    102 | {Python,Java}        | FALSE
    103 | {SQL,Docker}         | TRUE
    104 | {AWS,Linux}          | FALSE
    105 | {Python,SQL,Docker}  | TRUE

```

### Explanation

All items in the right-hand array must exist inside the left-hand array.

### Failing Example

```sql
-- Searching for rows containing BOTH SQL and Java
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE skills @> ARRAY['SQL','Java'];

```

```text
 emp_id | emp_name | skills 
--------+----------+--------
(0 rows)

```

---

# `<@` (Contained By)

## Purpose

Returns `TRUE` when **all** elements in the left array exist within the right array.

## Syntax

```sql
array1 <@ array2

```

## Working Example

```sql
-- Find employees whose skills are completely subsetted by this reference set
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE skills <@ ARRAY['SQL','Python','AWS','Docker'];

```

```text
 emp_id | emp_name |        skills        
--------+----------+----------------------
    101 | John     | {SQL,Python,AWS}
    103 | David    | {SQL,Docker}
    105 | Sam      | {Python,SQL,Docker}
(3 rows)

```

### Evaluation Matrix

```text
 emp_id |        skills        | Contained in reference set? 
--------+----------------------+-----------------------------
    101 | {SQL,Python,AWS}     | TRUE
    102 | {Python,Java}        | FALSE ('Java' missing)
    103 | {SQL,Docker}         | TRUE
    104 | {AWS,Linux}          | FALSE ('Linux' missing)
    105 | {Python,SQL,Docker}  | TRUE

```

### Explanation

Useful for filtering records that only possess skills from an approved or pre-defined set.

### Failing Example

```sql
-- Alice (Python, Java) is not contained because 'Java' is missing from the right side
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE emp_id = 102 
kaggle->   AND skills <@ ARRAY['SQL','Python','AWS'];

```

```text
 emp_id | emp_name | skills 
--------+----------+--------
(0 rows)

```

---

# `&&` (Overlap)

## Purpose

Returns `TRUE` if the left and right arrays share **at least one** common element.

## Syntax

```sql
array1 && array2

```

## Working Example

```sql
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE skills && ARRAY['Java','SQL'];

```

```text
 emp_id | emp_name |        skills        
--------+----------+----------------------
    101 | John     | {SQL,Python,AWS}
    102 | Alice    | {Python,Java}
    103 | David    | {SQL,Docker}
    105 | Sam      | {Python,SQL,Docker}
(4 rows)

```

### Evaluation Matrix

```text
 emp_id |        skills        | Shares element with ['Java', 'SQL']? 
--------+----------------------+---------------------------------------
    101 | {SQL,Python,AWS}     | TRUE (SQL)
    102 | {Python,Java}        | TRUE (Java)
    103 | {SQL,Docker}         | TRUE (SQL)
    104 | {AWS,Linux}          | FALSE
    105 | {Python,SQL,Docker}  | TRUE (SQL)

```

### Explanation

Unlike `@>` which requires *all* targets to match, `&&` acts as an "ANY match" condition.

### Failing Example

```sql
-- Searching for skills that overlap with Go or Rust
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE skills && ARRAY['Go','Rust'];

```

```text
 emp_id | emp_name | skills 
--------+----------+--------
(0 rows)

```

---

# `||` (Concatenate)

## Purpose

Appends or prepends arrays, adding elements together into a single combined array.

## Syntax

```sql
array1 || array2

```

## Working Example

```sql
kaggle=> SELECT emp_id, emp_name, skills || ARRAY['Git'] AS updated_skills
kaggle-> FROM employee_skills;

```

```text
 emp_id | emp_name |           updated_skills           
--------+----------+------------------------------------
    101 | John     | {SQL,Python,AWS,Git}
    102 | Alice    | {Python,Java,Git}
    103 | David    | {SQL,Docker,Git}
    104 | Bob      | {AWS,Linux,Git}
    105 | Sam      | {Python,SQL,Docker,Git}
(5 rows)

```

### Explanation

`||` is an arithmetic-style operator for arrays (it creates a new value instead of returning boolean `TRUE`/`FALSE`). It can concatenate arrays to arrays or single elements to arrays.

### Edge Case: Concatenating `NULL`

```sql
-- Concatenating NULL to an array produces NULL in PostgreSQL
kaggle=> SELECT emp_id, skills || NULL AS result
kaggle-> FROM employee_skills
kaggle-> WHERE emp_id = 101;

```

```text
 emp_id | result 
--------+--------
    101 | 
(1 row)

```

```

```
