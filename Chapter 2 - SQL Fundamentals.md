# Chapter 2: SQL Fundamentals

This is the main chapter of SQL Mastery. Everything later in this repo — joins, CTEs, subqueries, window functions, database design, data engineering — builds on the concepts here, so it's worth actually understanding this rather than skimming it.

## What you'll be able to do after this chapter

- Explain what SQL, a database, a table, a row, and a column actually are
- Retrieve data with `SELECT`
- Remove duplicates with `DISTINCT`
- Filter rows with `WHERE`
- Sort results with `ORDER BY`
- Limit how many rows come back
- Rename columns in your output using aliases
- Read a basic SQL query and know what order it actually runs in

---

## 1. What is SQL?

SQL (Structured Query Language) is how you talk to a relational database. With it you can read data, write data, change data, delete data, and change the structure of the database itself.

```sql
SELECT *
FROM customers;
```

That query is just asking: *give me everything in the customers table.*

## 2. Databases and tables

A **database** is an organised collection of data. An e-commerce company's database might hold customers, products, orders, payments, and employees - usually each of those as its own **table**.

A table stores data as **rows and columns**. Here's a `customers` table:

| customer_id | first_name | last_name | city       | age |
| ----------: | ---------- | --------- | ---------- | --: |
|           1 | John       | Smith     | Bristol    |  25 |
|           2 | Sarah      | Khan      | London     |  31 |
|           3 | David      | Brown     | Manchester |  28 |
|           4 | Emma       | Wilson    | Bristol    |  24 |

**Columns** describe an attribute (first_name, city, age). **Rows** are individual records — row 1 is John Smith, row 2 is Sarah Khan, and so on. Each column usually has a data type behind it too, e.g. `customer_id` is an `INTEGER`, `first_name` is a `VARCHAR`.

## 3. The basic shape of a query

```sql
SELECT column_name
FROM table_name
WHERE condition
ORDER BY column_name;
```

Example:

```sql
SELECT first_name, city
FROM customers
WHERE city = 'Bristol'
ORDER BY first_name;
```




Reading this left to right: pick `first_name` and `city`, from `customers`, keep only Bristol rows, sort by first name.

---

## 4. SELECT

`SELECT` retrieves data.

```sql
SELECT first_name
FROM customers;
```

| first_name |
| ---------- |
| John       |
| Sarah      |
| David      |
| Emma       |

**Multiple columns** are comma-separated:

```sql
SELECT first_name, last_name, city
FROM customers;
```

Don't forget the commas - `SELECT first_name last_name city` will error or misbehave depending on the engine.

**`SELECT *`** means "every column." It's fine for exploring a table you don't know yet, but in real code prefer naming the columns you actually need:

```sql
SELECT customer_id, first_name, city
FROM customers;
```

Reasons this matters in practice: it's clearer what the query depends on, it avoids pulling data you don't use, and it stops your query from silently breaking (or silently changing behaviour) when someone adds a column to the table later.

## 5. DISTINCT

`DISTINCT` removes duplicate rows from the result.

```sql
SELECT city
FROM customers;
```

| city       |
| ---------- |
| Bristol    |
| London     |
| Bristol    |
| Manchester |
| Bristol    |

```sql
SELECT DISTINCT city
FROM customers;
```

| city       |
| ---------- |
| Bristol    |
| London     |
| Manchester |

With more than one column, `DISTINCT` applies to the **combination**, not each column separately:

```sql
SELECT DISTINCT city, age
FROM customers;
```

This gives you unique (city, age) pairs - not a unique list of cities and a separate unique list of ages.

## 6. WHERE and comparison operators

```sql
SELECT *
FROM customers
WHERE city = 'Bristol';
```

| customer_id | first_name | city    |
| ----------: | ---------- | ------- |
|           1 | John       | Bristol |
|           4 | Emma       | Bristol |

| Operator | Meaning                  |
| -------- | ------------------------ |
| `=`      | equal to                 |
| `<>` / `!=` | not equal to          |
| `>`      | greater than             |
| `<`      | less than                |
| `>=`     | greater than or equal to |
| `<=`     | less than or equal to    |

```sql
WHERE age = 25       -- exactly 25
WHERE age > 25        -- older than 25
WHERE age < 30        -- younger than 30
WHERE age >= 30        -- 30 or older
WHERE age <= 30        -- 30 or younger
WHERE city <> 'Bristol'  -- not from Bristol
```

**Strings need quotes, numbers don't.**

```sql
WHERE city = 'Bristol'   -- correct
WHERE age = 25            -- correct
WHERE age = '25'          -- avoid — works in some databases via implicit conversion, but don't rely on it
```

## 7. ORDER BY

```sql
SELECT *
FROM customers
ORDER BY age;
```

Default sort direction is ascending. Be explicit with `ASC` or `DESC` if it matters:

```sql
ORDER BY age ASC    -- smallest to largest / A to Z
ORDER BY age DESC   -- largest to smallest / Z to A
```

You can sort by more than one column - the second column only matters for breaking ties in the first:

```sql
SELECT *
FROM customers
ORDER BY city ASC, age DESC;
```

This sorts by city alphabetically, and within each city, oldest to youngest.

## 8.1 . LIMIT

`LIMIT` caps how many rows come back - useful for exploring large tables, and essential when paired with `ORDER BY`.

```sql
SELECT *
FROM customers
LIMIT 3;
```

The combination `ORDER BY ... LIMIT ...` is one of the most common patterns in SQL - it's how you answer "top N" or "bottom N" questions:

```sql
-- three oldest customers
SELECT *
FROM customers
ORDER BY age DESC
LIMIT 3;

-- three youngest customers
SELECT *
FROM customers
ORDER BY age ASC
LIMIT 3;
```

## 8.2. TOP - SQL Server

In **Microsoft SQL Server**, `LIMIT` is not supported. Instead, use `TOP` to control how many rows are returned.

```sql
SELECT TOP 3 *
FROM customers;
```

`TOP` is useful when exploring large tables or when you only need a specific number of rows.

The combination of **`ORDER BY` + `TOP`** is commonly used to answer **"top N"** or **"bottom N"** questions.

```sql
-- three oldest customers
SELECT TOP 3 *
FROM customers
ORDER BY age DESC;

-- three youngest customers
SELECT TOP 3 *
FROM customers
ORDER BY age ASC;
```

### Remember

* `TOP 3` → return only 3 rows
* `ORDER BY age DESC` → oldest first
* `ORDER BY age ASC` → youngest first

**Note:** Always use `ORDER BY` when you care *which* rows are returned. Without it, SQL Server does not guarantee the order of the results.



## 9. Aliases

An alias renames a column (or table) just for the output - it doesn't touch the actual schema.

```sql
SELECT
    first_name AS name,
    city AS location
FROM customers;
```

| name  | location   |
| ----- | ---------- |
| John  | Bristol    |
| Sarah | London     |

`AS` can technically be omitted (`SELECT first_name name`), but keep it in - it makes queries much easier to read at a glance.

**Table aliases** matter even more once you get to joins:

```sql
SELECT c.first_name, c.city
FROM customers AS c;
```

Now `c` stands in for `customers`, so you write `c.first_name` instead of `customers.first_name`.

## 10. Comments

```sql
-- single-line comment
SELECT *
FROM customers;

/*
multi-line comment,
useful for explaining
more complex logic
*/
SELECT *
FROM customers
WHERE city = 'Bristol';
```

Use these to explain *why* a query exists or why some logic is non-obvious — not to narrate lines that are already self-explanatory.

## 11. A few syntax notes

- Statements are usually terminated with `;`. Some tools don't require it, but get in the habit anyway.
- SQL keywords aren't case-sensitive (`SELECT` = `select`), but uppercase keywords are the convention because they're easier to scan.
- Formatting matters. Compare:

```sql
SELECT first_name,city FROM customers WHERE city='Bristol' ORDER BY first_name;
```

```sql
SELECT
    first_name,
    city
FROM customers
WHERE city = 'Bristol'
ORDER BY first_name;
```

The second one is the one you want to be looking at six months from now.

## 12. The order things actually happen in

This is the part that trips people up. The order you **write** a query isn't the order the database **evaluates** it.

You write:

```
SELECT → FROM → WHERE → ORDER BY → LIMIT
```

It's logically processed roughly as:

```
1. FROM      — get the table
2. WHERE     — filter rows
3. SELECT    — pick columns
4. ORDER BY  — sort
5. LIMIT     — cut down to N rows
```

Take this query:

```sql
SELECT
    first_name,
    age
FROM customers
WHERE age > 25
ORDER BY age DESC
LIMIT 3;
```

Conceptually the database: starts with `customers`, filters to `age > 25`, picks out `first_name` and `age`, sorts by `age` descending, then keeps only the top 3 rows. Once you internalise this order, a lot of "why doesn't this work" moments later on (aliases not being usable in `WHERE`, for instance) stop being mysterious.

---

## Worked examples

**Find the three oldest customers from Bristol, showing name and age.**

| customer_id | first_name | last_name | city       | age |
| ----------: | ---------- | --------- | ---------- | --: |
|           1 | John       | Smith     | Bristol    |  25 |
|           2 | Sarah      | Khan      | London     |  31 |
|           3 | David      | Brown     | Bristol    |  28 |
|           4 | Emma       | Wilson    | Manchester |  24 |
|           5 | James      | Taylor    | Bristol    |  35 |

```sql
SELECT
    first_name AS name,
    age
FROM customers
WHERE city = 'Bristol'
ORDER BY age DESC
LIMIT 3;
```

| name  | age |
| ----- | --: |
| James |  35 |
| David |  28 |
| John  |  25 |

**Find the two highest-paid employees in IT.**

| employee_id | name  | department | salary |
| ----------: | ----- | ---------- | -----: |
|           1 | John  | IT         |  40000 |
|           2 | Sarah | HR         |  35000 |
|           3 | David | IT         |  50000 |
|           4 | Emma  | Sales      |  42000 |
|           5 | James | IT         |  60000 |

```sql
SELECT
    name,
    salary
FROM employees
WHERE department = 'IT'
ORDER BY salary DESC
LIMIT 2;
```

| name  | salary |
| ----- | -----: |
| James |  60000 |
| David |  50000 |

---

## Common mistakes

**No FROM clause.**
```sql
SELECT name;              -- wrong
SELECT name FROM employees; -- right
```

**Double quotes for strings.** Some dialects tolerate this, but single quotes are the standard:
```sql
WHERE city = "Bristol";  -- avoid
WHERE city = 'Bristol';  -- use this
```

**Missing commas between columns.**
```sql
SELECT name city age FROM customers;        -- wrong
SELECT name, city, age FROM customers;       -- right
```

**WHERE after ORDER BY.**
```sql
SELECT * FROM customers ORDER BY age WHERE age > 25;  -- wrong
SELECT * FROM customers WHERE age > 25 ORDER BY age;   -- right
```

**LIMIT before ORDER BY.** If you want the top N by some criterion, you have to sort first, *then* limit — otherwise you're just grabbing an arbitrary 3 rows and sorting those.
```sql
SELECT * FROM customers LIMIT 3 ORDER BY age DESC;   -- wrong
SELECT * FROM customers ORDER BY age DESC LIMIT 3;    -- right
```

---

## Quick reference

```sql
-- select
SELECT column_name FROM table_name;
SELECT column1, column2 FROM table_name;
SELECT * FROM table_name;

-- distinct
SELECT DISTINCT column_name FROM table_name;

-- filter
SELECT * FROM table_name WHERE condition;

-- sort
SELECT * FROM table_name ORDER BY column_name ASC;
SELECT * FROM table_name ORDER BY column_name DESC;

-- limit
SELECT * FROM table_name LIMIT 10;

-- alias
SELECT column_name AS new_name FROM table_name;

-- all together
SELECT
    column1,
    column2 AS renamed_column
FROM table_name
WHERE condition
ORDER BY column1 DESC
LIMIT 10;
```

---

## Practice

Assume an `employees` table with: `employee_id, first_name, last_name, department, city, salary, age`.

**Beginner**
1. Return all employees.
2. Return only first_name, last_name, department.
3. Return employees in the IT department.
4. Return employees earning more than £40,000.
5. Return employees younger than 30.

**Intermediate**

6. Return all unique departments.
7. Return employees ordered by salary, highest first.
8. Return the five highest-paid employees.
9. Return employees from Bristol, alphabetical by first name.
10. Return the three youngest employees.
11. Return first_name and salary for employees earning over £50,000, renamed to `employee_name` and `annual_salary`.
12. Find the five highest-paid employees in IT.

<details>
<summary>Answers</summary>

```sql
-- 1
SELECT * FROM employees;

-- 2
SELECT first_name, last_name, department FROM employees;

-- 3
SELECT * FROM employees WHERE department = 'IT';

-- 4
SELECT * FROM employees WHERE salary > 40000;

-- 5
SELECT * FROM employees WHERE age < 30;

-- 6
SELECT DISTINCT department FROM employees;

-- 7
SELECT * FROM employees ORDER BY salary DESC;

-- 8
SELECT * FROM employees ORDER BY salary DESC LIMIT 5;

-- 9
SELECT * FROM employees WHERE city = 'Bristol' ORDER BY first_name ASC;

-- 10
SELECT * FROM employees ORDER BY age ASC LIMIT 3;

-- 11
SELECT
    first_name AS employee_name,
    salary AS annual_salary
FROM employees
WHERE salary > 50000;

-- 12
SELECT
    first_name,
    last_name,
    salary
FROM employees
WHERE department = 'IT'
ORDER BY salary DESC
LIMIT 5;
```

</details>

---

## Before moving on

You should be able to explain, without notes, what a database/table/row/column is, and use `SELECT`, `DISTINCT`, `WHERE`, `ORDER BY`, `LIMIT`, and aliases confidently — plus recite the logical processing order (`FROM → WHERE → SELECT → ORDER BY → LIMIT`).

Next chapter: **02 – Filtering and Operators**, covering `AND`, `OR`, `NOT`, `BETWEEN`, `IN`, `LIKE`, `IS NULL` / `IS NOT NULL`, and combining conditions in more realistic filtering problems.

The point of this chapter was never to memorise syntax — it's to start thinking in terms of tables, rows, and filters. That mindset is what carries you through everything after this.