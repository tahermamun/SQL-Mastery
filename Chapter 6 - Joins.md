# Chapter 6: Joins

Every chapter so far has queried one table at a time. Real databases split data across many tables on purpose - `customers` in one table, `orders` in another - connected by the `FOREIGN KEY` relationships from Chapter 3. A **join** is how you pull related rows from two or more tables back together into a single result.

This is arguably the most important chapter in the whole repo. Almost every real query you write from here on will involve a join.

## What you'll be able to do after this chapter

- Explain why data is split across tables in the first place
- Write an `INNER JOIN` to combine matching rows from two tables
- Write a `LEFT JOIN` (and understand how it differs from `INNER JOIN`)
- Write a `RIGHT JOIN` and a `FULL OUTER JOIN`
- Understand what a `CROSS JOIN` does and why you rarely want one by accident
- Join more than two tables together
- Recognise and avoid the most common join mistakes, especially the ones that silently return the wrong number of rows

---

## 1. Why joins exist

Consider two tables built up across earlier chapters:

**customers**

| customer_id | first_name | city    |
| ----------: | ---------- | ------- |
|           1 | John       | Bristol |
|           2 | Sarah      | London  |
|           3 | David      | Bristol |

**orders**

| order_id | customer_id | amount |
| -------: | ----------: | -----: |
|      100 |           1 |  49.99 |
|      101 |           1 |  15.00 |
|      102 |           2 |  89.50 |

`orders` doesn't repeat the customer's name or city - it just stores `customer_id`, which points back to `customers` via the `FOREIGN KEY` from Chapter 3. This avoids duplicating "John, Bristol" on every single order John places. If John moves to Bath, you update one row in `customers`, not every order he's ever placed.

The trade-off is that to answer a question like "show me each order along with the customer's name," you need to combine the two tables. That's exactly what a join does.

---

## 2. INNER JOIN

`INNER JOIN` returns rows where a match exists in **both** tables. If a customer has no orders, they're left out entirely; if an order somehow had no matching customer, it would be left out too.

```sql
SELECT
    customers.first_name,
    orders.order_id,
    orders.amount
FROM customers
INNER JOIN orders
    ON customers.customer_id = orders.customer_id;
```

| first_name | order_id | amount |
| ---------- | -------: | -----: |
| John       |      100 |  49.99 |
| John       |      101 |  15.00 |
| Sarah      |      102 |  89.50 |

Notice David is missing - he has no rows in `orders`, so there's nothing for him to match against, and `INNER JOIN` excludes him entirely.

### Reading the syntax

```sql
FROM customers
INNER JOIN orders
    ON customers.customer_id = orders.customer_id
```

- `FROM customers` - start with this table
- `INNER JOIN orders` - combine it with this table
- `ON customers.customer_id = orders.customer_id` - using this rule to decide which rows from each table belong together

The `ON` clause is doing the real work - it tells the database exactly how the two tables relate. Get the `ON` condition wrong (or leave it out - more on that later) and you get nonsense results, not an error.

### Using table aliases

Chapter 2 introduced table aliases; joins are where they really start pulling their weight, since typing `customers.` and `orders.` repeatedly gets tedious fast.

```sql
SELECT
    c.first_name,
    o.order_id,
    o.amount
FROM customers AS c
INNER JOIN orders AS o
    ON c.customer_id = o.customer_id;
```

Same query, much less noise. This is the standard way joins are written in practice.

### JOIN alone means INNER JOIN

```sql
FROM customers
JOIN orders ON customers.customer_id = orders.customer_id
```

Just `JOIN` (no keyword before it) defaults to `INNER JOIN`. Both are common in real code; being explicit with `INNER JOIN` is slightly clearer to read, especially next to `LEFT JOIN` and `RIGHT JOIN`.

---

## 3. LEFT JOIN

`LEFT JOIN` (also written `LEFT OUTER JOIN`) returns **every** row from the left-hand table, whether it has a match on the right or not. Where there's no match, the right-hand columns come back as `NULL`.

```sql
SELECT
    c.first_name,
    o.order_id,
    o.amount
FROM customers AS c
LEFT JOIN orders AS o
    ON c.customer_id = o.customer_id;
```

| first_name | order_id | amount |
| ---------- | -------: | -----: |
| John       |      100 |  49.99 |
| John       |      101 |  15.00 |
| Sarah      |      102 |  89.50 |
| David      |     NULL |   NULL |

David shows up this time, with `NULL` in the order columns - he's a real customer, he just hasn't placed an order yet.

### The classic use case: "find rows with no match"

`LEFT JOIN` combined with `WHERE ... IS NULL` is the standard pattern for finding rows on the left that have nothing matching on the right:

```sql
SELECT c.first_name
FROM customers AS c
LEFT JOIN orders AS o
    ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;
```

This returns only customers who have never placed an order - David, in this example. `INNER JOIN` couldn't answer this question at all, since it drops unmatched rows rather than surfacing them.

---

## 4. RIGHT JOIN

`RIGHT JOIN` is the mirror image of `LEFT JOIN` - every row from the right-hand table is kept, with `NULL`s filled in on the left where there's no match.

```sql
SELECT
    c.first_name,
    o.order_id
FROM customers AS c
RIGHT JOIN orders AS o
    ON c.customer_id = o.customer_id;
```

In practice, `RIGHT JOIN` is used far less than `LEFT JOIN` - anything a `RIGHT JOIN` can do, swapping the table order and using `LEFT JOIN` does too, and most people find left-to-right reading order easier to follow. You'll see `RIGHT JOIN` in the wild occasionally, but it's rarely the first choice when writing a new query.

---

## 5. FULL OUTER JOIN

`FULL OUTER JOIN` keeps everything - every row from both tables, matched where possible, `NULL`-filled on whichever side has no match.

```sql
SELECT
    c.first_name,
    o.order_id
FROM customers AS c
FULL OUTER JOIN orders AS o
    ON c.customer_id = o.customer_id;
```

This would show David (customer with no order) and, if it existed, an order with no matching customer - both sides' unmatched rows appear, with `NULL` filling the gap. Not every database supports `FULL OUTER JOIN` directly (MySQL notably doesn't - it has to be simulated with a `UNION` of a `LEFT JOIN` and a `RIGHT JOIN`), so check what your engine supports.

---

## 6. CROSS JOIN

`CROSS JOIN` combines **every** row from one table with **every** row from the other - no `ON` condition, no matching logic. If `customers` has 3 rows and `sizes` has 4 rows, a `CROSS JOIN` between them produces 12 rows.

```sql
SELECT c.first_name, s.size_name
FROM customers AS c
CROSS JOIN sizes AS s;
```

This is occasionally genuinely useful - generating every possible combination of two sets, like every product paired with every size for a catalogue. But it's also exactly what happens **by accident** if you write a join and forget the `ON` clause, which is why it's worth understanding clearly rather than skipping past it.

```sql
-- accidental cross join - no ON clause
SELECT c.first_name, o.order_id
FROM customers, orders;
```

If `customers` has 100 rows and `orders` has 500, this silently returns 50,000 rows of nonsense - every customer paired with every order, whether related or not. No error, just a wrong (and often huge) result. Always double-check your join has an `ON` clause connecting the tables correctly.

---

## 7. Joining more than two tables

Joins chain - you can connect a third table the same way you connected the second.

```sql
SELECT
    c.first_name,
    o.order_id,
    p.product_name
FROM customers AS c
INNER JOIN orders AS o
    ON c.customer_id = o.customer_id
INNER JOIN order_items AS oi
    ON o.order_id = oi.order_id
INNER JOIN products AS p
    ON oi.product_id = p.product_id;
```

Each `JOIN` adds one more table into the mix, and each needs its own `ON` clause describing how it connects to what's already there. Read it top to bottom: start with customers, bring in their orders, bring in the items on those orders, bring in the product details for those items.

---

## Worked example

Using `customers`, `orders`, and a new `products`/`order_items` pair to represent items within an order:

**Show every customer's name next to their total number of orders, including customers with zero orders.**

```sql
SELECT
    c.first_name,
    COUNT(o.order_id) AS order_count
FROM customers AS c
LEFT JOIN orders AS o
    ON c.customer_id = o.customer_id
GROUP BY c.first_name;
```

(`GROUP BY` and `COUNT` are properly covered in a later chapter on aggregation - included here just to show how naturally joins combine with them. `LEFT JOIN` is essential here specifically because we want zero-order customers to still show up, with a count of 0.)

**Find customers who have never placed an order.**

```sql
SELECT c.first_name
FROM customers AS c
LEFT JOIN orders AS o
    ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;
```

**Show order details together with the customer's city, but only for Bristol customers.**

```sql
SELECT
    o.order_id,
    o.amount,
    c.first_name,
    c.city
FROM orders AS o
INNER JOIN customers AS c
    ON o.customer_id = c.customer_id
WHERE c.city = 'Bristol';
```

Note the `WHERE` here filters *after* the join has combined the rows - it's operating on the combined result, not on `customers` in isolation.

---

## Common mistakes

**Forgetting the ON clause (accidental CROSS JOIN).**
```sql
SELECT * FROM customers, orders;                                  -- every combination, huge and wrong
SELECT * FROM customers JOIN orders ON customers.customer_id = orders.customer_id;  -- correct
```

**Using INNER JOIN when you actually need LEFT JOIN.**
If you want *every* customer, including ones with no orders, `INNER JOIN` will silently drop them - no error, just missing rows. This is one of the sneakiest join bugs because the query runs fine, it just quietly gives you less than you asked for.

**Ambiguous column reference.**
```sql
SELECT customer_id FROM customers JOIN orders ON customers.customer_id = orders.customer_id;
-- error: customer_id exists in both tables - which one?
```
Once two tables share a column name, you have to qualify it: `c.customer_id` or `o.customer_id`.

**Joining on the wrong columns.**
```sql
FROM customers AS c
JOIN orders AS o ON c.customer_id = o.order_id   -- wrong - these aren't the same thing
```
This runs without error but produces meaningless matches. Always double check the `ON` clause connects a foreign key to the actual primary key it references.

**Putting a filter on the "outer" side in the wrong place with LEFT JOIN.**
```sql
-- this quietly turns the LEFT JOIN back into something like an INNER JOIN
SELECT c.first_name, o.order_id
FROM customers AS c
LEFT JOIN orders AS o ON c.customer_id = o.customer_id
WHERE o.amount > 50;
```
Since `WHERE` runs after the join, filtering on `o.amount > 50` throws away David's `NULL` row too (`NULL > 50` isn't true), even though the whole point of the `LEFT JOIN` was to keep him. If you need to filter the right-hand table's condition *without* losing unmatched left rows, move the condition into the `ON` clause instead:
```sql
FROM customers AS c
LEFT JOIN orders AS o ON c.customer_id = o.customer_id AND o.amount > 50;
```

---

## Quick reference

```sql
-- inner join: only matching rows from both sides
SELECT ...
FROM table_a AS a
INNER JOIN table_b AS b ON a.key = b.key;

-- left join: everything from the left, matched where possible
SELECT ...
FROM table_a AS a
LEFT JOIN table_b AS b ON a.key = b.key;

-- right join: everything from the right, matched where possible
SELECT ...
FROM table_a AS a
RIGHT JOIN table_b AS b ON a.key = b.key;

-- full outer join: everything from both sides
SELECT ...
FROM table_a AS a
FULL OUTER JOIN table_b AS b ON a.key = b.key;

-- cross join: every combination of rows
SELECT ...
FROM table_a AS a
CROSS JOIN table_b AS b;

-- find unmatched rows on the left
SELECT ...
FROM table_a AS a
LEFT JOIN table_b AS b ON a.key = b.key
WHERE b.key IS NULL;

-- joining three tables
SELECT ...
FROM table_a AS a
JOIN table_b AS b ON a.key = b.a_key
JOIN table_c AS c ON b.key = c.b_key;
```

---

## Practice

Assume `customers (customer_id, first_name, city)`, `orders (order_id, customer_id, amount, status)`, and `products (product_id, product_name, price)` joined via an `order_items (order_id, product_id, quantity)` table.

**Beginner**
1. Return every order along with the customer's first name.
2. Return every customer along with their orders, including customers with no orders.
3. Return only customers who have never placed an order.

**Intermediate**

4. Return every order along with the customer's city, but only for orders over £50.
5. Return every customer, their order count (including zero), sorted by order count descending. (Just write the join + `GROUP BY`/`COUNT` shape - full aggregation detail comes in a later chapter.)
6. Return order_id, product_name, and quantity for every item ordered, joining `orders`, `order_items`, and `products` together.

<details>
<summary>Answers</summary>

```sql
-- 1
SELECT o.order_id, c.first_name
FROM orders AS o
INNER JOIN customers AS c ON o.customer_id = c.customer_id;

-- 2
SELECT c.first_name, o.order_id
FROM customers AS c
LEFT JOIN orders AS o ON c.customer_id = o.customer_id;

-- 3
SELECT c.first_name
FROM customers AS c
LEFT JOIN orders AS o ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;

-- 4
SELECT o.order_id, o.amount, c.city
FROM orders AS o
INNER JOIN customers AS c ON o.customer_id = c.customer_id
WHERE o.amount > 50;

-- 5
SELECT c.first_name, COUNT(o.order_id) AS order_count
FROM customers AS c
LEFT JOIN orders AS o ON c.customer_id = o.customer_id
GROUP BY c.first_name
ORDER BY order_count DESC;

-- 6
SELECT o.order_id, p.product_name, oi.quantity
FROM orders AS o
JOIN order_items AS oi ON o.order_id = oi.order_id
JOIN products AS p ON oi.product_id = p.product_id;
```

</details>

---

## Before moving on

You should be comfortable explaining the difference between `INNER JOIN` and `LEFT JOIN` in plain terms, know how to find "unmatched" rows using `LEFT JOIN ... WHERE ... IS NULL`, understand why a missing `ON` clause is dangerous, and be able to join three tables together correctly.

## Next chapter

**`07-aggregation`** - `GROUP BY`, `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, and `HAVING`. This is where joins start getting genuinely powerful - combining tables and then summarising the combined result, like "total spend per customer" or "average order value per city."