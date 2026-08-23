# Chapter 4: DML Commands

Chapter 3 was about building the containers — tables, columns, constraints. This chapter is about putting data into those containers and changing it once it's there: **DML** (Data Manipulation Language). The three commands are `INSERT`, `UPDATE`, and `DELETE`.

Unlike DDL, these are ordinary transactional statements - they can be wrapped in a transaction and rolled back if something goes wrong, which matters a lot once you're working with real data instead of a personal practice table.

## What you'll be able to do after this chapter

- Insert single and multiple rows with `INSERT`
- Insert data selected from another table
- Update existing rows with `UPDATE`, safely and precisely
- Delete rows with `DELETE`
- Understand `NULL` handling during inserts and updates
- Understand transactions (`COMMIT` / `ROLLBACK`) well enough to undo a mistake before it's permanent
- Know exactly what happens if you forget a `WHERE` clause on `UPDATE` or `DELETE` - and why that's the single most common way to damage a table

---

## 1. INSERT

`INSERT` adds new rows to a table.

### Inserting a single row

```sql
INSERT INTO customers (customer_id, first_name, last_name, city, age)
VALUES (1, 'John', 'Smith', 'Bristol', 25);
```

Reading this: insert into `customers`, listing which columns you're providing values for, then the actual values in the same order.

### Column order matters - but only relative to your own list

```sql
INSERT INTO customers (first_name, customer_id, city, last_name, age)
VALUES ('John', 1, 'Bristol', 'Smith', 25);
```

This works identically to the first example - the columns and values just need to line up with each other, not with the table's original column order.

### Omitting the column list

```sql
INSERT INTO customers
VALUES (1, 'John', 'Smith', 'Bristol', 25);
```

This is allowed, but it silently depends on the table's exact column order - if someone adds a column later, this breaks or (worse) inserts data into the wrong column without erroring. Always list the columns explicitly. It's a few extra characters and it means your `INSERT` still works correctly after the table changes shape.

### Inserting multiple rows at once

```sql
INSERT INTO customers (customer_id, first_name, last_name, city, age)
VALUES
    (2, 'Sarah', 'Khan', 'London', 31),
    (3, 'David', 'Brown', 'Manchester', 28),
    (4, 'Emma', 'Wilson', 'Bristol', 24);
```

One `INSERT`, three rows. This is faster than three separate `INSERT` statements and is the normal way to load a batch of data.

### Leaving columns out

If a column has a `DEFAULT`, allows `NULL`, or is an auto-incrementing key, you can skip it entirely:

```sql
INSERT INTO customers (first_name, last_name, city)
VALUES ('James', 'Taylor', 'Bristol');
```

`age` isn't provided here - it becomes `NULL` unless the column has a `DEFAULT` defined, in which case that default is used instead. If a column is `NOT NULL` with no default and you leave it out, the insert fails.

### Inserting data from another table

Rather than typing values by hand, you can populate a table from the result of a `SELECT`:

```sql
INSERT INTO bristol_customers (customer_id, first_name, city)
SELECT customer_id, first_name, city
FROM customers
WHERE city = 'Bristol';
```

This copies every Bristol customer from `customers` into `bristol_customers` in one statement. There's no `VALUES` clause at all here - the `SELECT` supplies the rows.

---

## 2. UPDATE

`UPDATE` changes values in rows that already exist.

### Basic syntax

```sql
UPDATE table_name
SET column_name = new_value
WHERE condition;
```

Example:

```sql
UPDATE customers
SET city = 'Bath'
WHERE customer_id = 1;
```

This changes customer 1's city to Bath, and touches nothing else.

### Updating multiple columns at once

```sql
UPDATE customers
SET city = 'Bath', age = 26
WHERE customer_id = 1;
```

Comma-separate the `SET` assignments, same as listing columns in a `SELECT`.

### Updating multiple rows

`WHERE` doesn't have to isolate a single row - it filters however broadly you want:

```sql
UPDATE customers
SET city = 'Bristol (Central)'
WHERE city = 'Bristol';
```

Every customer whose city is currently `'Bristol'` gets updated in one statement.

### Updating based on the row's current value

```sql
UPDATE employees
SET salary = salary * 1.05
WHERE department = 'IT';
```

This gives every IT employee a 5% raise - `salary` on the right-hand side refers to each row's existing value before the update, not some external number.

### The WHERE clause is not optional in practice

Technically, `WHERE` is optional syntax-wise. In practice, this:

```sql
UPDATE customers
SET city = 'Bristol';
```

sets **every single row's** city to Bristol - the whole table, no matter how many rows it has. This is one of the most common and most damaging mistakes in SQL. Always write the `WHERE` clause first, mentally, before you even think about the `SET`.

---

## 3. DELETE

`DELETE` removes rows from a table (structure and other rows stay untouched).

### Basic syntax

```sql
DELETE FROM table_name
WHERE condition;
```

Example:

```sql
DELETE FROM customers
WHERE customer_id = 4;
```

Removes exactly one row: the customer with ID 4.

### Deleting multiple rows

```sql
DELETE FROM customers
WHERE city = 'Manchester';
```

Every customer from Manchester is removed.

### Same trap as UPDATE: no WHERE means everything goes

```sql
DELETE FROM customers;
```

This deletes **every row** in the table. The table itself still exists afterward — this is different from `DROP TABLE`, which was covered in Chapter 3 — but every record is gone. As with `UPDATE`, get in the habit of writing and double-checking `WHERE` before you run a `DELETE`.

### DELETE vs TRUNCATE, revisited

Chapter 3 covered this from the DDL side; worth repeating from here since it's genuinely easy to reach for the wrong one:

| | `DELETE FROM table WHERE ...` | `TRUNCATE TABLE table` |
| --- | --- | --- |
| Can filter specific rows | yes | no — always all rows |
| Speed on large tables | slower (logs each row) | faster (minimal logging) |
| Can be rolled back in a transaction | yes | usually not, or only partially, depending on the database |
| Resets auto-increment counters | no | usually yes |

If you need to remove *some* rows, or you might want to undo it, use `DELETE`. If you're clearing an entire table and don't need a rollback safety net, `TRUNCATE` is the better tool.

---

## 4. NULL handling

`NULL` means "no value" - not zero, not an empty string, genuinely unknown or absent. It behaves differently from what people expect the first time they hit it.

### Inserting NULL explicitly

```sql
INSERT INTO customers (customer_id, first_name, city)
VALUES (5, 'Alex', NULL);
```

`city` is now unknown for this row — not blank text, `NULL`.

### You can't filter NULL with =

```sql
SELECT * FROM customers WHERE city = NULL;   -- returns nothing, ever
SELECT * FROM customers WHERE city IS NULL;  -- correct
```

`= NULL` never matches, because `NULL` isn't a value that can equal anything — including another `NULL`. Always use `IS NULL` / `IS NOT NULL`.

### Setting a value back to NULL with UPDATE

```sql
UPDATE customers
SET city = NULL
WHERE customer_id = 5;
```

This is a valid, deliberate way to clear a value rather than delete the row.

---

## 5. Transactions: COMMIT and ROLLBACK

DML statements can be wrapped in a **transaction** - a block of changes that either all happen together or none of them do. This is the safety net that DDL mostly doesn't give you (see Chapter 3's note on DDL auto-committing).

```sql
BEGIN TRANSACTION;

UPDATE customers
SET city = 'Bath'
WHERE customer_id = 1;

DELETE FROM customers
WHERE customer_id = 4;

-- check the results look right, then either:
COMMIT;      -- make the changes permanent

-- or, if something looks wrong:
ROLLBACK;    -- undo everything since BEGIN TRANSACTION
```

This is the practical answer to "what if I run a bad `UPDATE` or `DELETE`": if you're inside a transaction and haven't committed yet, `ROLLBACK` undoes it completely. It's good practice to wrap risky `UPDATE`/`DELETE` statements in a transaction, `SELECT` the affected rows first to sanity-check the `WHERE` clause, and only `COMMIT` once you're sure.

```sql
BEGIN TRANSACTION;

-- sanity check first — see exactly what would be affected
SELECT * FROM customers WHERE city = 'Bristol';

-- if that looks right, run the actual change
UPDATE customers
SET city = 'Bristol (Central)'
WHERE city = 'Bristol';

COMMIT;
```

---
## Worked example

Starting from the `customers` and `orders` tables built in Chapter 3:

**Add three new customers:**

```sql
INSERT INTO customers (customer_id, first_name, email, home_city)
VALUES
    (1, 'John', 'john@example.com', 'Bristol'),
    (2, 'Sarah', 'sarah@example.com', 'London'),
    (3, 'David', 'david@example.com', 'Manchester');
```

**Record an order for customer 1:**

```sql
INSERT INTO orders (order_id, customer_id, amount)
VALUES (100, 1, 49.99);
```

`order_date` and `status` weren't provided — they fall back to their `DEFAULT` values from Chapter 3 (`CURRENT_DATE` and `'pending'`).

**Mark that order as shipped:**

```sql
UPDATE orders
SET status = 'shipped'
WHERE order_id = 100;
```

**Remove a customer who no longer exists, safely, using a transaction:**

```sql
BEGIN TRANSACTION;

SELECT * FROM customers WHERE customer_id = 3;  -- confirm it's the right row

DELETE FROM customers
WHERE customer_id = 3;

COMMIT;
```

Note: if `orders` had a row referencing `customer_id = 3` via the `FOREIGN KEY` from Chapter 3, this `DELETE` would be rejected by the database until that order is dealt with first — the constraint is protecting you from leaving an order pointing at a customer that no longer exists.

---

## Common mistakes

**Running UPDATE or DELETE without a WHERE clause.**
```sql
UPDATE customers SET city = 'Bristol';   -- updates every row
DELETE FROM customers;                    -- deletes every row
```
The single most common way to accidentally wreck a table. Write the `WHERE` first.

**Using `=` to compare against NULL.**
```sql
WHERE city = NULL      -- wrong, never matches
WHERE city IS NULL     -- right
```

**Relying on column order instead of naming columns in INSERT.**
```sql
INSERT INTO customers VALUES (1, 'John', 'Smith', 'Bristol', 25);  -- fragile
INSERT INTO customers (customer_id, first_name, last_name, city, age)
VALUES (1, 'John', 'Smith', 'Bristol', 25);                         -- explicit, safer
```

**Forgetting that a FOREIGN KEY will block a DELETE.**
```sql
DELETE FROM customers WHERE customer_id = 1;
-- fails if orders still has rows referencing customer_id = 1
```
This isn't a bug — it's the constraint from Chapter 3 doing exactly what it's supposed to. Delete or reassign the referencing rows first.

**Assuming DELETE and TRUNCATE are interchangeable.**
They usually get you to the same end state (empty table) but behave very differently under the hood — see the comparison table above.

**Committing before checking.**
Running a broad `UPDATE`/`DELETE` and immediately committing, instead of checking the affected rows first inside a transaction. Once committed, a rollback may no longer be possible.

---

## Quick reference

```sql
-- insert one row
INSERT INTO table_name (col1, col2) VALUES (val1, val2);

-- insert multiple rows
INSERT INTO table_name (col1, col2)
VALUES (val1, val2), (val3, val4), (val5, val6);

-- insert from a SELECT
INSERT INTO table_name (col1, col2)
SELECT col1, col2 FROM other_table WHERE condition;

-- update
UPDATE table_name
SET col1 = value1, col2 = value2
WHERE condition;

-- delete
DELETE FROM table_name
WHERE condition;

-- null comparisons
WHERE column IS NULL
WHERE column IS NOT NULL

-- transaction
BEGIN TRANSACTION;
-- ... statements ...
COMMIT;     -- or ROLLBACK;
```

---

## Practice

Assume `customers (customer_id, first_name, last_name, city, age)` and `orders (order_id, customer_id, order_date, amount, status)` from Chapter 3.

**Beginner**
1. Insert a new customer: id 10, first name "Priya", last name "Patel", city "Leeds", age 27.
2. Insert three customers in a single statement (make up the details).
3. Update customer 10's city to "Sheffield".
4. Delete the customer with id 10.

**Intermediate**

5. Give every employee in a `salary` column a 10% raise, but only for the `'Sales'` department.
6. Set every order's `status` to `'cancelled'` where the order is older than `'2025-01-01'`.
7. Insert into a `vip_customers` table by selecting every customer over age 30 from `customers`.
8. Wrap a `DELETE` of all orders with `status = 'cancelled'` inside a transaction, checking the rows first with a `SELECT`, then commit.

<details>
<summary>Answers</summary>

```sql
-- 1
INSERT INTO customers (customer_id, first_name, last_name, city, age)
VALUES (10, 'Priya', 'Patel', 'Leeds', 27);

-- 2
INSERT INTO customers (customer_id, first_name, last_name, city, age)
VALUES
    (11, 'Tom', 'Reed', 'York', 33),
    (12, 'Alice', 'Grant', 'Oxford', 29),
    (13, 'Sam', 'Lee', 'Cardiff', 41);

-- 3
UPDATE customers
SET city = 'Sheffield'
WHERE customer_id = 10;

-- 4
DELETE FROM customers
WHERE customer_id = 10;

-- 5
UPDATE employees
SET salary = salary * 1.10
WHERE department = 'Sales';

-- 6
UPDATE orders
SET status = 'cancelled'
WHERE order_date < '2025-01-01';

-- 7
INSERT INTO vip_customers (customer_id, first_name, city, age)
SELECT customer_id, first_name, city, age
FROM customers
WHERE age > 30;

-- 8
BEGIN TRANSACTION;

SELECT * FROM orders WHERE status = 'cancelled';

DELETE FROM orders
WHERE status = 'cancelled';

COMMIT;
```

</details>

---

## Before moving on

You should be comfortable inserting single and multiple rows, updating rows precisely with a correct `WHERE` clause, deleting rows safely, handling `NULL` correctly with `IS NULL`/`IS NOT NULL`, and using a transaction to check your work before committing a risky change.

## Next chapter

**`05-filtering-and-operators`** going deeper on `WHERE` with `AND`, `OR`, `NOT`, `BETWEEN`, `IN`, `LIKE`, and combining multiple conditions to write more realistic, precise filters.