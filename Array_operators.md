# PostgreSQL ARRAY Operators Guide

## Sample Table

### Create Table

``` sql
kaggle=> CREATE TABLE employee_skills
(
    emp_id INT,
    emp_name TEXT,
    skills TEXT[]
);
CREATE TABLE
```

### Insert Sample Data

``` sql
kaggle=> INSERT INTO employee_skills
VALUES
(101,'John', ARRAY['SQL','Python','AWS']),
(102,'Alice',ARRAY['Python','Java']),
(103,'David',ARRAY['SQL','Docker']),
(104,'Bob',ARRAY['AWS','Linux']),
(105,'Sam',ARRAY['Python','SQL','Docker']);

INSERT 0 5
```

### Verify Data

``` sql
kaggle=> SELECT * FROM employee_skills;
```

``` text
 emp_id | emp_name |         skills
--------+----------+--------------------------
    101 | John     | {SQL,Python,AWS}
    102 | Alice    | {Python,Java}
    103 | David    | {SQL,Docker}
    104 | Bob      | {AWS,Linux}
    105 | Sam      | {Python,SQL,Docker}
(5 rows)
```

------------------------------------------------------------------------

# 1. `=` (Equal)

## Purpose

Checks whether two arrays are exactly the same.

## Syntax

``` sql
array1 = array2
```

### Working Example

``` sql
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE skills = ARRAY['Python','Java'];
```

**Output**

``` text
 emp_id | emp_name |     skills
--------+----------+----------------
    102 | Alice    | {Python,Java}
(1 row)
```

### Evaluation

    emp_id skills                  Result
  -------- --------------------- ----------
       101 {SQL,Python,AWS}       ❌ FALSE
       102 {Python,Java}          ✅ TRUE
       103 {SQL,Docker}           ❌ FALSE
       104 {AWS,Linux}            ❌ FALSE
       105 {Python,SQL,Docker}    ❌ FALSE

**Explanation**

Returns TRUE only when both arrays have: - Same elements - Same order -
Same number of elements

### Failing Example

``` sql
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE skills = ARRAY['Java','Python'];
```

``` text
(0 rows)
```

------------------------------------------------------------------------

# 2. `<>` (Not Equal)

## Purpose

Returns TRUE when two arrays are not exactly the same.

``` sql
kaggle=> SELECT emp_id,emp_name
kaggle-> FROM employee_skills
kaggle-> WHERE skills <> ARRAY['Python','Java'];
```

**Output**

``` text
 emp_id | emp_name
--------+---------
101 | John
103 | David
104 | Bob
105 | Sam
(4 rows)
```

### Evaluation

    emp_id   Result
  -------- ----------
       101  ✅ TRUE
       102  ❌ FALSE
       103  ✅ TRUE
       104  ✅ TRUE
       105  ✅ TRUE

**Explanation**

Only Alice has an array exactly equal to `{Python,Java}`.

------------------------------------------------------------------------

# 3. `@>` (Contains)

## Purpose

Returns TRUE when the left array contains all elements of the right
array.

``` sql
kaggle=> SELECT emp_id,emp_name,skills
kaggle-> FROM employee_skills
kaggle-> WHERE skills @> ARRAY['SQL'];
```

**Output**

``` text
 emp_id | emp_name | skills
--------+----------+----------------------
101 | John | {SQL,Python,AWS}
103 | David | {SQL,Docker}
105 | Sam | {Python,SQL,Docker}
(3 rows)
```

### Evaluation

    emp_id  Contains SQL    Result
  -------- -------------- ----------
       101      Yes        ✅ TRUE
       102       No        ❌ FALSE
       103      Yes        ✅ TRUE
       104       No        ❌ FALSE
       105      Yes        ✅ TRUE

**Explanation**

Every element in the right array must exist in the left array.

### Failing Example

``` sql
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE skills @> ARRAY['SQL','Java'];
```

``` text
(0 rows)
```

------------------------------------------------------------------------

# 4. `<@` (Contained By)

## Purpose

Returns TRUE when every element of the left array exists in the right
array.

``` sql
kaggle=> SELECT ARRAY['SQL'] <@ ARRAY['SQL','Python','AWS'];
```

**Output**

``` text
 ?column?
----------
 t
(1 row)
```

### Failing Example

``` sql
kaggle=> SELECT ARRAY['SQL','Java'] <@ ARRAY['SQL','Python','AWS'];
```

``` text
 ?column?
----------
 f
(1 row)
```

**Explanation**

All values from the left array must be present in the right array.

------------------------------------------------------------------------

# 5. `&&` (Overlap)

## Purpose

Returns TRUE when two arrays have at least one common element.

``` sql
kaggle=> SELECT emp_id,emp_name
kaggle-> FROM employee_skills
kaggle-> WHERE skills && ARRAY['Java','SQL'];
```

**Output**

``` text
 emp_id | emp_name
--------+----------
101 | John
102 | Alice
103 | David
105 | Sam
(4 rows)
```

### Evaluation

    emp_id Common Element     Result
  -------- ---------------- ----------
       101 SQL               ✅ TRUE
       102 Java              ✅ TRUE
       103 SQL               ✅ TRUE
       104 None              ❌ FALSE
       105 SQL               ✅ TRUE

**Explanation**

Only one matching element is enough.

### Failing Example

``` sql
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE skills && ARRAY['Go','Rust'];
```

``` text
(0 rows)
```

------------------------------------------------------------------------
