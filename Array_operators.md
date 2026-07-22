# PostgreSQL ARRAY Operators Guide

## Sample Table

``` sql
kaggle=> CREATE TABLE employee_skills
(
 emp_id INT,
 emp_name TEXT,
 skills TEXT[]
);
CREATE TABLE

kaggle=> INSERT INTO employee_skills VALUES
(101,'John',ARRAY['SQL','Python','AWS']),
(102,'Alice',ARRAY['Python','Java']),
(103,'David',ARRAY['SQL','Docker']),
(104,'Bob',ARRAY['AWS','Linux']),
(105,'Sam',ARRAY['Python','SQL','Docker']);
INSERT 0 5

kaggle=> SELECT * FROM employee_skills;
```

``` text
 emp_id | emp_name |         skills
--------+----------+--------------------------
101 | John  | {SQL,Python,AWS}
102 | Alice | {Python,Java}
103 | David | {SQL,Docker}
104 | Bob   | {AWS,Linux}
105 | Sam   | {Python,SQL,Docker}
(5 rows)
```

------------------------------------------------------------------------

# = (Equal)

## Purpose

Checks whether two arrays are exactly the same.

## Syntax

``` sql
array1 = array2
```

## Working Example

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

``` text
    emp_id skills                  Result
  -------- --------------------- ----------
       101 {SQL,Python,AWS}       ❌ FALSE
       102 {Python,Java}          ✅ TRUE
       103 {SQL,Docker}           ❌ FALSE
       104 {AWS,Linux}            ❌ FALSE
       105 {Python,SQL,Docker}    ❌ FALSE
```

### Explanation

Returns TRUE only when both arrays have the same elements, order and
length.

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

# \<\> (Not Equal)

## Purpose

Returns TRUE when arrays are not exactly the same.

## Syntax

``` sql
array1 <> array2
```

## Working Example

``` sql
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE skills <> ARRAY['Python','Java'];
```

``` text
 emp_id | emp_name |         skills
--------+----------+--------------------------
101 | John | {SQL,Python,AWS}
103 | David | {SQL,Docker}
104 | Bob | {AWS,Linux}
105 | Sam | {Python,SQL,Docker}
(4 rows)
```

### Evaluation

    emp_id skills                  Result
  -------- --------------------- ----------
       101 {SQL,Python,AWS}       ✅ TRUE
       102 {Python,Java}          ❌ FALSE
       103 {SQL,Docker}           ✅ TRUE
       104 {AWS,Linux}            ✅ TRUE
       105 {Python,SQL,Docker}    ✅ TRUE

### Explanation

Returns TRUE when arrays differ.

### Failing Example

``` sql
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE skills <> ARRAY['SQL','Python','AWS','Java'];
```

``` text
(0 rows)
```

------------------------------------------------------------------------

# @\> (Contains)

## Purpose

Returns TRUE when left array contains all elements of right array.

## Syntax

``` sql
array1 @> array2
```

## Working Example

``` sql
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE skills @> ARRAY['SQL'];
```

``` text
 emp_id | emp_name |         skills
--------+----------+--------------------------
101 | John | {SQL,Python,AWS}
103 | David | {SQL,Docker}
105 | Sam | {Python,SQL,Docker}
(3 rows)
```

### Evaluation

    emp_id skills                  Result
  -------- --------------------- ----------
       101 {SQL,Python,AWS}       ✅ TRUE
       102 {Python,Java}          ❌ FALSE
       103 {SQL,Docker}           ✅ TRUE
       104 {AWS,Linux}            ❌ FALSE
       105 {Python,SQL,Docker}    ✅ TRUE

### Explanation

All elements on the right must exist on the left.

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

# \<@ (Contained By)

## Purpose

Returns TRUE when left array is contained in right array.

## Syntax

``` sql
array1 <@ array2
```

## Working Example

``` sql
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE ARRAY['SQL'] <@ skills;
```

``` text
 emp_id | emp_name |         skills
--------+----------+--------------------------
101 | John | {SQL,Python,AWS}
103 | David | {SQL,Docker}
105 | Sam | {Python,SQL,Docker}
(3 rows)
```

### Evaluation

    emp_id skills                  Result
  -------- --------------------- ----------
       101 {SQL,Python,AWS}       ✅ TRUE
       102 {Python,Java}          ❌ FALSE
       103 {SQL,Docker}           ✅ TRUE
       104 {AWS,Linux}            ❌ FALSE
       105 {Python,SQL,Docker}    ✅ TRUE

### Explanation

Every element of the left array must exist in the right array.

### Failing Example

``` sql
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE ARRAY['SQL','Java'] <@ skills;
```

``` text
(0 rows)
```

------------------------------------------------------------------------

# && (Overlap)

## Purpose

Returns TRUE when arrays share at least one common element.

## Syntax

``` sql
array1 && array2
```

## Working Example

``` sql
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE skills && ARRAY['Java','SQL'];
```

``` text
 emp_id | emp_name |         skills
--------+----------+--------------------------
101 | John | {SQL,Python,AWS}
102 | Alice | {Python,Java}
103 | David | {SQL,Docker}
105 | Sam | {Python,SQL,Docker}
(4 rows)
```

### Evaluation

    emp_id skills                  Result
  -------- --------------------- ----------
       101 {SQL,Python,AWS}       ✅ TRUE
       102 {Python,Java}          ✅ TRUE
       103 {SQL,Docker}           ✅ TRUE
       104 {AWS,Linux}            ❌ FALSE
       105 {Python,SQL,Docker}    ✅ TRUE

### Explanation

Only one common element is required.

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

# \|\| (Concatenate)

## Purpose

Joins two arrays into one array.

## Syntax

``` sql
array1 || array2
```

## Working Example

``` sql
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE skills || ARRAY['Git'];
```

``` text
 emp_id | emp_name |              ?column?
--------+----------+----------------------------------------
101 | John | {SQL,Python,AWS,Git}
102 | Alice | {Python,Java,Git}
103 | David | {SQL,Docker,Git}
104 | Bob | {AWS,Linux,Git}
105 | Sam | {Python,SQL,Docker,Git}
(5 rows)
```

### Explanation

Appends the second array to the first.

### Failing Example

``` sql
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE ARRAY[]::text[] || ARRAY[]::text[];
```

``` text
(0 rows)
```
