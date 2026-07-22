# PostgreSQL ARRAY Operators - Complete Guide

## Sample Table

### Create Table

``` sql
kaggle=> CREATE TABLE employee_skills
(
    emp_id   INT,
    emp_name TEXT,
    skills   TEXT[]
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
(104,'Bob',  ARRAY['AWS','Linux']),
(105,'Sam',  ARRAY['Python','SQL','Docker']);

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

# 1. = (Equal)

**Purpose**

Returns TRUE only when both arrays are exactly the same.

**Syntax**

``` sql
array1 = array2
```

### Working Example

``` sql
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE skills = ARRAY['Python','Java'];
```

``` text
 emp_id | emp_name |     skills
--------+----------+----------------
    102 | Alice    | {Python,Java}
(1 row)
```

### Evaluation

    emp_id   Result
  -------- ----------
       101  ❌ FALSE
       102  ✅ TRUE
       103  ❌ FALSE
       104  ❌ FALSE
       105  ❌ FALSE

**Explanation**

Same elements, same order and same number of elements.

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

# 2. \<\> (Not Equal)

**Purpose**

Returns TRUE when arrays are not exactly the same.

### Working Example

``` sql
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE skills <> ARRAY['Python','Java'];
```

``` text
 emp_id | emp_name |         skills
--------+----------+--------------------------
    101 | John     | {SQL,Python,AWS}
    103 | David    | {SQL,Docker}
    104 | Bob      | {AWS,Linux}
    105 | Sam      | {Python,SQL,Docker}
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

Only Alice exactly matches `{Python,Java}`.

------------------------------------------------------------------------

# 3. @\> (Contains)

**Purpose**

Returns TRUE when the left array contains all elements of the right
array.

### Working Example

``` sql
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE skills @> ARRAY['SQL'];
```

``` text
 emp_id | emp_name |         skills
--------+----------+--------------------------
    101 | John     | {SQL,Python,AWS}
    103 | David    | {SQL,Docker}
    105 | Sam      | {Python,SQL,Docker}
(3 rows)
```

### Evaluation

    emp_id   Result
  -------- ----------
       101  ✅ TRUE
       102  ❌ FALSE
       103  ✅ TRUE
       104  ❌ FALSE
       105  ✅ TRUE

**Explanation**

Every value in the right array must exist in the left array.

### Failing Example

``` sql
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE skills @> ARRAY['SQL','Java'];
```

``` text
 emp_id | emp_name | skills
--------+----------+--------
(0 rows)
```

------------------------------------------------------------------------

# 4. \<@ (Contained By)

**Purpose**

Returns TRUE when every element of the left array exists in the right
array.

### Working Example

``` sql
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE ARRAY['SQL'] <@ skills;
```

``` text
 emp_id | emp_name |         skills
--------+----------+--------------------------
    101 | John     | {SQL,Python,AWS}
    103 | David    | {SQL,Docker}
    105 | Sam      | {Python,SQL,Docker}
(3 rows)
```

### Evaluation

    emp_id   Result
  -------- ----------
       101  ✅ TRUE
       102  ❌ FALSE
       103  ✅ TRUE
       104  ❌ FALSE
       105  ✅ TRUE

**Explanation**

All values in the left array must be present in the right array.

### Failing Example

``` sql
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE ARRAY['SQL','Java'] <@ skills;
```

``` text
 emp_id | emp_name | skills
--------+----------+--------
(0 rows)
```

------------------------------------------------------------------------

# 5. && (Overlap)

**Purpose**

Returns TRUE when two arrays have at least one common element.

### Working Example

``` sql
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE skills && ARRAY['Java','SQL'];
```

``` text
 emp_id | emp_name |         skills
--------+----------+--------------------------
    101 | John     | {SQL,Python,AWS}
    102 | Alice    | {Python,Java}
    103 | David    | {SQL,Docker}
    105 | Sam      | {Python,SQL,Docker}
(4 rows)
```

### Evaluation

    emp_id   Result
  -------- ----------
       101  ✅ TRUE
       102  ✅ TRUE
       103  ✅ TRUE
       104  ❌ FALSE
       105  ✅ TRUE

**Explanation**

At least one common element is enough.

### Failing Example

``` sql
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE skills && ARRAY['Go','Rust'];
```

``` text
 emp_id | emp_name | skills
--------+----------+--------
(0 rows)
```

------------------------------------------------------------------------

# 6. \|\| (Concatenate)

**Purpose**

Combines two arrays into one array.

### Example

``` sql
kaggle=> SELECT *,
kaggle->        skills || ARRAY['Docker'] AS updated_skills
kaggle-> FROM employee_skills
kaggle-> WHERE emp_id = 101;
```

``` text
 emp_id | emp_name |      skills       |        updated_skills
--------+----------+-------------------+--------------------------------
    101 | John     | {SQL,Python,AWS}  | {SQL,Python,AWS,Docker}
(1 row)
```

**Explanation**

The original array is not modified. The concatenated array is returned
only in the query result.

------------------------------------------------------------------------
