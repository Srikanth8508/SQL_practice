# PostgreSQL ARRAY Operators - Complete Guide

## Sample Table

### Create Table

```sql
kaggle=> CREATE TABLE employee_skills
(
    emp_id INT,
    emp_name TEXT,
    skills TEXT[]
);
CREATE TABLE
```

### Insert Sample Data

```sql
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

```sql
kaggle=> SELECT * FROM employee_skills;
```

```text
 emp_id | emp_name |         skills
--------+----------+--------------------------
    101 | John     | {SQL,Python,AWS}
    102 | Alice    | {Python,Java}
    103 | David    | {SQL,Docker}
    104 | Bob      | {AWS,Linux}
    105 | Sam      | {Python,SQL,Docker}
(5 rows)
```

---

# Operator 1 : = (Equal)

## Purpose

Checks whether two arrays are exactly the same.

### Syntax

```sql
array1 = array2
```

### Query

```sql
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE skills = ARRAY['Python','Java'];
```

### Output

```text
 emp_id | emp_name |     skills
--------+----------+----------------
    102 | Alice    | {Python,Java}
(1 row)
```

### Evaluation

| emp_id | skills | Result |
|--------|------------------------|--------|
|101|{SQL,Python,AWS}|❌ FALSE|
|102|{Python,Java}|✅ TRUE|
|103|{SQL,Docker}|❌ FALSE|
|104|{AWS,Linux}|❌ FALSE|
|105|{Python,SQL,Docker}|❌ FALSE|

### Explanation

Returns **TRUE** only when both arrays have:

- Same elements
- Same order
- Same number of elements

---

## Example (Fails)

### Query

```sql
kaggle=> SELECT *
kaggle-> FROM employee_skills
kaggle-> WHERE skills = ARRAY['Java','Python'];
```

### Output

```text
 emp_id | emp_name | skills
--------+----------+--------
(0 rows)
```

### Evaluation

| emp_id | skills | Result |
|--------|------------------------|--------|
|101|{SQL,Python,AWS}|❌ FALSE|
|102|{Python,Java}|❌ FALSE|
|103|{SQL,Docker}|❌ FALSE|
|104|{AWS,Linux}|❌ FALSE|
|105|{Python,SQL,Docker}|❌ FALSE|

### Explanation

The order is different, so no row satisfies the condition.

---
