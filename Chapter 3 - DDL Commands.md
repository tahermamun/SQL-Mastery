# Chapter 3: DDL Commands

Everything in Chapter 2 was about reading data that already exists. This chapter is about the commands that create and shape the structures that data lives in - tables, columns, constraints. These are the **DDL** (Data Definition Language) commands: `CREATE`, `ALTER`, `DROP`, `TRUNCATE`.

## What you'll be able to do after this chapter

- Explain the difference between DDL and DML
- Create a table with appropriate data types
- Add, modify, and drop columns on an existing table
- Rename tables and columns
- Use constraints (`PRIMARY KEY`, `FOREIGN KEY`, `NOT NULL`, `UNIQUE`, `CHECK`, `DEFAULT`) to enforce rules on your data
- Understand the difference between `DROP` and `TRUNCATE`
- Know why DDL commands are dangerous and how to avoid wrecking a database with one

---

## 1. DDL vs DML

SQL commands are usually split into categories. The two you'll use constantly are:

- **DDL (Data Definition Language)** - defines the *structure*: `CREATE`, `ALTER`, `DROP`, `TRUNCATE`
- **DML (Data Manipulation Language)** - works with the *data inside* that structure: `SELECT`, `INSERT`, `UPDATE`, `DELETE`

Chapter 2 was entirely DML (well, just `SELECT`). This chapter is DDL - you're building the container before you put anything in it.

One important practical difference: most DDL commands **auto-commit**. In many databases, once you run `CREATE TABLE` or `DROP TABLE`, it's done - there's no rolling it back the way you can undo an `UPDATE` inside a transaction. Keep that in mind, especially with `DROP`.

---

## 2. CREATE TABLE

This is how you define a new table - its name, its columns, and each column's data type.

```sql
CREATE TABLE customers (
    customer_id INT,
    first_name  VARCHAR(50),
    last_name   VARCHAR(50),
    city        VARCHAR(50),
    age         INT
);
```

Reading this: create a table called `customers`, with five columns, each given a name and a data type. That's the minimum a `CREATE TABLE` needs - a name and at least one column.

### Common data types

You'll see these constantly, so it's worth knowing what each is actually for rather than just copying examples.

| Type            | Use for                                      | Example        |
| --------------- | --------------------------------------------- | -------------- |
| `INT`           | whole numbers                                 | `25`           |
| `DECIMAL(p, s)` | exact decimal numbers (money, precise values) | `DECIMAL(10,2)` for `1234.56` |
| `VARCHAR(n)`    | variable-length text, up to `n` characters    | `VARCHAR(100)` |
| `CHAR(n)`       | fixed-length text, always `n` characters      | `CHAR(2)` for country codes like `'UK'` |
| `DATE`          | a calendar date                               | `2026-08-18`   |
| `DATETIME`      | date and time together                        | `2026-08-18 14:30:00` |
| `BOOLEAN`       | true/false (some databases use `BIT` instead) | `TRUE` / `FALSE` |
| `TEXT`          | long, unbounded text                          | a blog post body |

`DECIMAL` vs floating-point types matter more than people expect - use `DECIMAL` for money. Floating-point types can introduce tiny rounding errors that you really don't want in financial data.

---

## 3. Constraints

A constraint is a rule attached to a column (or table) that the database enforces automatically. This is where a database actually protects your data instead of just storing whatever you throw at it.

### PRIMARY KEY

Uniquely identifies each row. No two rows can share the same value, and it can't be `NULL`.

```sql
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    first_name  VARCHAR(50)
);
```

Almost every table you build should have one. It's what other tables use to reference a specific row (more on that below with foreign keys).

### NOT NULL

Forces a column to always have a value - you can't leave it empty.

```sql
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    first_name  VARCHAR(50) NOT NULL,
    city        VARCHAR(50)
);
```

Here, `first_name` is required on every row; `city` is allowed to be left blank (`NULL`).

### UNIQUE

Like `PRIMARY KEY` in that values can't repeat, but a table can have several `UNIQUE` columns, and unlike a primary key, `NULL` is usually allowed.

```sql
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    email       VARCHAR(100) UNIQUE
);
```

Two customers can't share an email, but the primary key is still `customer_id`.

### DEFAULT

Supplies a value automatically when one isn't given.

```sql
CREATE TABLE orders (
    order_id   INT PRIMARY KEY,
    order_date DATE DEFAULT CURRENT_DATE,
    status     VARCHAR(20) DEFAULT 'pending'
);
```

If you insert an order without specifying `status`, it becomes `'pending'` automatically.

### CHECK

Restricts what values are allowed based on a condition.

```sql
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    age         INT CHECK (age >= 18),
    salary      DECIMAL(10,2) CHECK (salary > 0)
);
```

Trying to insert an employee aged 15, or a negative salary, gets rejected by the database itself - before it ever becomes a bug in your application code.

### FOREIGN KEY

Links a column in one table to the primary key of another, and stops you from inserting a value that doesn't actually exist over there.

```sql
CREATE TABLE orders (
    order_id    INT PRIMARY KEY,
    customer_id INT,
    order_date  DATE,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
```

This says: every `customer_id` in `orders` must correspond to a real `customer_id` in `customers`. You physically cannot create an order for a customer that doesn't exist. This is the mechanism that makes joins between tables meaningful later on - it's what guarantees the relationship is real, not just a coincidence of matching numbers.

### Putting it together

```sql
CREATE TABLE orders (
    order_id    INT PRIMARY KEY,
    customer_id INT NOT NULL,
    order_date  DATE DEFAULT CURRENT_DATE,
    amount      DECIMAL(10,2) CHECK (amount > 0),
    status      VARCHAR(20) DEFAULT 'pending',
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
```

Every constraint here is doing a specific job: `PRIMARY KEY` gives each order a unique identity, `NOT NULL` means every order must belong to a customer, `DEFAULT` fills in sensible values automatically, `CHECK` blocks nonsensical amounts, and `FOREIGN KEY` ties the order back to a real customer.

---

## 4. ALTER TABLE

Once a table exists, `ALTER TABLE` is how you change its structure - add a column, drop one, change a type, rename something. You don't need to drop and recreate a table just because you forgot a column.

### Add a column

```sql
ALTER TABLE customers
ADD email VARCHAR(100);
```

Existing rows get `NULL` in the new column unless you specify a `DEFAULT`.

### Drop a column

```sql
ALTER TABLE customers
DROP COLUMN age;
```

This deletes the column **and every value in it**, permanently. There's no confirmation prompt.

### Modify a column's data type

```sql
-- SQL Server
ALTER TABLE customers
ALTER COLUMN first_name VARCHAR(100);

-- MySQL / PostgreSQL syntax differs slightly:
-- MySQL:      ALTER TABLE customers MODIFY first_name VARCHAR(100);
-- PostgreSQL: ALTER TABLE customers ALTER COLUMN first_name TYPE VARCHAR(100);
```

This is one of the few places where syntax genuinely differs by database - worth double-checking the docs for whichever engine you're on.

### Rename a column

```sql
-- SQL Server
EXEC sp_rename 'customers.first_name', 'given_name', 'COLUMN';

-- PostgreSQL / MySQL
ALTER TABLE customers
RENAME COLUMN first_name TO given_name;
```

### Rename a table

```sql
-- SQL Server
EXEC sp_rename 'customers', 'clients';

-- PostgreSQL / MySQL
ALTER TABLE customers
RENAME TO clients;
```

### Add or drop a constraint

```sql
-- add
ALTER TABLE customers
ADD CONSTRAINT chk_age CHECK (age >= 0);

-- drop
ALTER TABLE customers
DROP CONSTRAINT chk_age;
```

Naming your constraints (`chk_age` here) rather than letting the database auto-generate a name is worth doing - it makes them much easier to find and drop later.

---

## 5. DROP TABLE

Deletes an entire table - structure and data, gone.

```sql
DROP TABLE customers;
```

There is no undo. If other tables have a `FOREIGN KEY` pointing at this one, most databases will refuse to drop it until you remove or update those references first - that's the constraint doing its job and protecting you from an inconsistent database.

`DROP TABLE IF EXISTS` is worth knowing - it avoids an error if the table's already gone (handy in scripts you re-run):

```sql
DROP TABLE IF EXISTS customers;
```

---

## 6. TRUNCATE TABLE

Removes **all rows** from a table but keeps the table structure itself.

```sql
TRUNCATE TABLE customers;
```

### TRUNCATE vs DELETE vs DROP

People mix these up constantly, so it's worth being precise:

| Command    | Removes rows? | Removes structure? | Can filter with WHERE? | Notes |
| ---------- | :---: | :---: | :---: | --- |
| `DELETE`   | yes (or a subset) | no | yes | DML, not DDL - logged row by row, slower on big tables, can be rolled back |
| `TRUNCATE` | yes, all of them | no | no | DDL - much faster, resets auto-increment counters, minimal logging |
| `DROP`     | yes | **yes** | no | DDL - the table itself ceases to exist |

If you want to empty a table entirely and don't care about `WHERE`-filtering, `TRUNCATE` is faster than `DELETE`. If you want to remove the table completely, that's `DROP`.

---

## 7. Why DDL is riskier than it looks

A `SELECT` that goes wrong gives you the wrong answer. A DDL command that goes wrong can destroy something. A few habits worth building early:

- Always double-check which environment you're connected to before running `DROP` or `TRUNCATE` - running it against production instead of your local test database is a classic, career-defining mistake.
- Use `IF EXISTS` / `IF NOT EXISTS` in scripts you'll re-run, so they don't error out on a second run.
- Name your constraints. `chk_age` is far easier to work with later than whatever auto-generated name the database picked.
- When you're not sure a change is safe, test it on a copy of the table first, not the real one.

---

## Worked example

Build out a small schema for a simple order system: customers who place orders.

```sql
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    first_name  VARCHAR(50) NOT NULL,
    email       VARCHAR(100) UNIQUE,
    city        VARCHAR(50)
);

CREATE TABLE orders (
    order_id    INT PRIMARY KEY,
    customer_id INT NOT NULL,
    order_date  DATE DEFAULT CURRENT_DATE,
    amount      DECIMAL(10,2) CHECK (amount > 0),
    status      VARCHAR(20) DEFAULT 'pending',
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
```

Say you later realise you need to track loyalty points on each customer, and rename `city` to `home_city`:

```sql
ALTER TABLE customers
ADD loyalty_points INT DEFAULT 0;

ALTER TABLE customers
RENAME COLUMN city TO home_city;
```

And if you want to clear out test data from `orders` without touching the table structure or the `customers` table:

```sql
TRUNCATE TABLE orders;
```

---

## Common mistakes

**Forgetting `NOT NULL` on columns that should always have a value.**
Without it, nothing stops a row being inserted with a missing `customer_id` on an order, which usually breaks logic somewhere downstream.

**Using `DROP` when you meant `TRUNCATE` (or vice versa).**
```sql
DROP TABLE orders;      -- table is gone entirely
TRUNCATE TABLE orders;  -- table stays, rows are gone
```
Know which one you actually want before running it.

**Adding a `FOREIGN KEY` referencing a table that doesn't exist yet.**
```sql
CREATE TABLE orders (
    customer_id INT,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)  -- fails if customers doesn't exist yet
);
```
Create the referenced table first, or restructure so parent tables come before child tables.

**Trying to `DROP` a table another table still references.**
```sql
DROP TABLE customers;  -- fails if orders.customer_id still references it
```
Drop or alter the referencing table's foreign key first, or drop child tables before parent tables.

**Forgetting that `ALTER TABLE ... DROP COLUMN` deletes data permanently.**
There's no confirmation step. If you're not sure, back up the table (or at least the column's data) first.

---

## Quick reference

```sql
-- create a table
CREATE TABLE table_name (
    column1 datatype constraints,
    column2 datatype constraints
);

-- constraints
column INT PRIMARY KEY
column VARCHAR(50) NOT NULL
column VARCHAR(50) UNIQUE
column INT DEFAULT 0
column INT CHECK (column > 0)
FOREIGN KEY (column) REFERENCES other_table(column)

-- alter table
ALTER TABLE table_name ADD column_name datatype;
ALTER TABLE table_name DROP COLUMN column_name;
ALTER TABLE table_name RENAME COLUMN old_name TO new_name;
ALTER TABLE table_name RENAME TO new_table_name;
ALTER TABLE table_name ADD CONSTRAINT name CHECK (condition);
ALTER TABLE table_name DROP CONSTRAINT name;

-- remove data / structure
DELETE FROM table_name WHERE condition;   -- DML, removes some/all rows, can roll back
TRUNCATE TABLE table_name;                -- DDL, removes all rows, keeps structure
DROP TABLE table_name;                    -- DDL, removes rows and structure entirely
DROP TABLE IF EXISTS table_name;
```

---

## Practice

**Beginner**
1. Create a `products` table with `product_id` (primary key), `product_name` (required text), and `price` (a decimal that must be greater than 0).
2. Add a `stock_quantity` column to `products`, defaulting to 0.
3. Rename `products` to `inventory`.
4. Remove all rows from `inventory` without deleting the table itself.

**Intermediate**

5. Create a `categories` table (`category_id` primary key, `category_name` unique and required).
6. Add a `category_id` column to `inventory`, and a foreign key linking it to `categories`.
7. Drop the `stock_quantity` column from `inventory`.
8. Delete the `categories` table entirely, assuming no other table references it.

<details>
<summary>Answers</summary>

```sql
-- 1
CREATE TABLE products (
    product_id   INT PRIMARY KEY,
    product_name VARCHAR(100) NOT NULL,
    price        DECIMAL(10,2) CHECK (price > 0)
);

-- 2
ALTER TABLE products
ADD stock_quantity INT DEFAULT 0;

-- 3
ALTER TABLE products
RENAME TO inventory;

-- 4
TRUNCATE TABLE inventory;

-- 5
CREATE TABLE categories (
    category_id   INT PRIMARY KEY,
    category_name VARCHAR(50) UNIQUE NOT NULL
);

-- 6
ALTER TABLE inventory
ADD category_id INT;

ALTER TABLE inventory
ADD CONSTRAINT fk_category
FOREIGN KEY (category_id) REFERENCES categories(category_id);

-- 7
ALTER TABLE inventory
DROP COLUMN stock_quantity;

-- 8
DROP TABLE categories;
```

</details>

---

## Before moving on

You should be comfortable creating a table with sensible data types and constraints, altering an existing table without breaking it, and knowing exactly which command to reach for - `DELETE`, `TRUNCATE`, or `DROP` - depending on whether you want to remove some rows, all rows, or the table entirely.

## Next chapter

**`04-dml-commands`** - `INSERT`, `UPDATE`, and `DELETE`: how you actually get data into the tables you now know how to build, and how to change or remove it safely once it's there.