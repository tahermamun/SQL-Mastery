# Chapter 7: Set Operations

Every chapter so far has combined tables *sideways* - joins add columns from another table onto each row. Set operations combine queries *vertically* instead - they stack the results of two `SELECT` statements on top of each other into one result. This chapter covers `UNION`, `UNION ALL`, `INTERSECT`, and `EXCEPT` (also called `MINUS` in some databases).

## What you'll be able to do after this chapter

- Explain the difference between combining tables with a join versus with a set operation
- Combine results from two queries with `UNION` and `UNION ALL`, and know which one to reach for
- Find rows common to two queries with `INTERSECT`
- Find rows in one query but not another with `EXCEPT` / `MINUS`
- Know the rules both queries in a set operation must follow
- Recognise when a set operation is the right tool versus when a join or `WHERE ... IN` is actually what you want

---

## 1. Joins vs set operations - the key difference

A join combines two tables **side by side**, matching rows on a key, producing wider rows with columns from both tables. A set operation combines two queries **top to bottom**, stacking their rows into one column set - the two queries need to return the *same shape* of result, not related tables.

Think of it this way: joins answer "for each customer, what are their orders" (relational, side-by-side). Set operations answer "give me one list combining current customers and prospective customers" (same shape, stacked together).

---

## 2. UNION

`UNION` combines the results of two `SELECT` statements into one result set, and **removes duplicate rows**.

```sql
SELECT city FROM customers
UNION
SELECT city FROM suppliers;
```

If both tables happen to have a customer and a supplier in `'Bristol'`, `'Bristol'` still only appears once in the combined result - `UNION` de-duplicates automatically, the same way `DISTINCT` does within a single query.

### Rules both queries must follow

- **Same number of columns** in each `SELECT`
- **Compatible data types** in matching positions (column 1 of the first query lines up with column 1 of the second, and so on - types don't need to be identical, but they need to be comparable)
- **Column names in the result come from the first query** - the second query's column names are ignored for output purposes

```sql
SELECT first_name AS name, city FROM customers
UNION
SELECT supplier_name, city FROM suppliers;
```

Even though the second query's column is called `supplier_name`, the combined result's column is labelled `name`, because that's what the first query called it.

### A realistic example

```sql
SELECT first_name, 'customer' AS source
FROM customers
WHERE city = 'Bristol'

UNION

SELECT contact_name, 'supplier' AS source
FROM suppliers
WHERE city = 'Bristol';
```

This produces one combined list of every Bristol contact - customers and suppliers together - with a `source` column showing which table each row came from. This pattern (adding a literal string as an extra column to tag where each row originated) is extremely common with `UNION`.

---

## 3. UNION ALL

`UNION ALL` does the same thing as `UNION`, but **keeps duplicates** instead of removing them.

```sql
SELECT city FROM customers
UNION ALL
SELECT city FROM suppliers;
```

If `'Bristol'` appears in both tables, it appears twice in the result here - once per source. Every row from both queries survives, no de-duplication pass.

### Why UNION ALL matters, not just as a variant

`UNION` has to compare every row against every other row to find and remove duplicates, which is genuinely expensive on large result sets. `UNION ALL` skips that check entirely, so it's meaningfully faster.

**The practical rule: use `UNION ALL` unless you specifically need duplicates removed.** A lot of code reaches for `UNION` out of habit when `UNION ALL` would give the same correct result faster - for instance, if you already know the two queries can't produce overlapping rows (like querying disjoint date ranges, or the `'customer'`/`'supplier'` tagged example above where the `source` column alone guarantees no true duplicates).

---

## 4. INTERSECT

`INTERSECT` returns only the rows that appear in **both** queries' results.

```sql
SELECT city FROM customers
INTERSECT
SELECT city FROM suppliers;
```

This returns only cities that have *both* a customer and a supplier - not every city from either table, just the overlap.

Same rules apply as `UNION`: matching column count, compatible types, and duplicates are removed from the result automatically (much like `UNION`, not `UNION ALL`).

**Note:** not every database supports `INTERSECT` directly - check your engine. Where it's missing, the same result can be built using `INNER JOIN` or `WHERE ... IN (SELECT ...)`, covered below.

---

## 5. EXCEPT (or MINUS)

`EXCEPT` returns rows that appear in the **first** query but **not** in the second. SQL Server and PostgreSQL use `EXCEPT`; Oracle uses `MINUS` for the same thing.

```sql
SELECT city FROM customers
EXCEPT
SELECT city FROM suppliers;
```

This returns cities that have customers but **no** suppliers. Order matters here, unlike `UNION`/`INTERSECT` - swapping the two queries changes the answer:

```sql
SELECT city FROM suppliers
EXCEPT
SELECT city FROM customers;
```

This instead returns cities that have suppliers but no customers - a different (and not necessarily equal-sized) result.

### A realistic example: finding customers with no orders, the set-operation way

Chapter 6 solved this with `LEFT JOIN ... WHERE ... IS NULL`. `EXCEPT` gives you an alternative route to the same answer:

```sql
SELECT customer_id FROM customers
EXCEPT
SELECT customer_id FROM orders;
```

Customer IDs that exist in `customers` but never appear in `orders` - i.e., customers who've never placed one. Same result as the `LEFT JOIN` approach from Chapter 6, different technique. Which one you reach for often comes down to readability: this version reads almost like plain English ("customers, except the ones with orders"), while the join version is more flexible if you also need other columns from `orders` in the result.

---

## 6. Set operations vs WHERE ... IN

Since `INTERSECT` and `EXCEPT` aren't supported everywhere, and even where they are, `WHERE ... IN` (or `NOT IN`) using a subquery - from Chapter 5 - can express the same idea:

```sql
-- INTERSECT version
SELECT city FROM customers
INTERSECT
SELECT city FROM suppliers;

-- equivalent using IN
SELECT DISTINCT city FROM customers
WHERE city IN (SELECT city FROM suppliers);
```

```sql
-- EXCEPT version
SELECT customer_id FROM customers
EXCEPT
SELECT customer_id FROM orders;

-- equivalent using NOT IN
SELECT DISTINCT customer_id FROM customers
WHERE customer_id NOT IN (SELECT customer_id FROM orders);
```

Worth knowing both, since which is available (or which reads more clearly) varies by situation and by database engine. Remember the `NOT IN` + `NULL` trap from Chapter 5 if you go this route - a `NULL` sneaking into that subquery's results can silently zero out the whole thing, which `EXCEPT` doesn't have the same issue with.

---

## Worked example

Using `customers (customer_id, first_name, city)` and `suppliers (supplier_id, contact_name, city)`:

**Every city with a business presence - customer or supplier - listed once each.**

```sql
SELECT city FROM customers
UNION
SELECT city FROM suppliers;
```

**A full combined contact list, keeping every row even if the same city shows up multiple times across both tables.**

```sql
SELECT first_name AS contact, city, 'customer' AS type FROM customers
UNION ALL
SELECT contact_name, city, 'supplier' FROM suppliers;
```

**Cities that have both a customer and a supplier - worth investigating for possible in-person meetings.**

```sql
SELECT city FROM customers
INTERSECT
SELECT city FROM suppliers;
```

**Cities with customers but no local supplier - could indicate a delivery cost/coverage gap.**

```sql
SELECT city FROM customers
EXCEPT
SELECT city FROM suppliers;
```

---

## Common mistakes

**Mismatched column counts.**
```sql
SELECT first_name, city FROM customers
UNION
SELECT contact_name FROM suppliers;    -- error: 2 columns vs 1
```
Both sides must return the same number of columns.

**Assuming column names from the second query survive.**
```sql
SELECT first_name AS name FROM customers
UNION
SELECT contact_name FROM suppliers;
-- result column is "name", not "contact_name" - the first query wins
```

**Using UNION when UNION ALL would do, on a large result set.**
If you know there can't be duplicates (or don't care about them), `UNION ALL` avoids an unnecessary and potentially expensive de-duplication pass.

**Expecting ORDER BY to work mid-query.**
```sql
SELECT city FROM customers ORDER BY city
UNION
SELECT city FROM suppliers;      -- error or ignored, depending on the engine
```
`ORDER BY` belongs at the very end of the whole combined statement, applying to the final result - not attached to one side of the `UNION`:
```sql
SELECT city FROM customers
UNION
SELECT city FROM suppliers
ORDER BY city;
```

**Forgetting EXCEPT is order-sensitive.**
`A EXCEPT B` and `B EXCEPT A` are generally different results - unlike `UNION` and `INTERSECT`, where the order of the two queries doesn't change the answer.

---

## Quick reference

```sql
-- union: combine, remove duplicates
SELECT col FROM table_a
UNION
SELECT col FROM table_b;

-- union all: combine, keep duplicates (faster)
SELECT col FROM table_a
UNION ALL
SELECT col FROM table_b;

-- intersect: rows in both
SELECT col FROM table_a
INTERSECT
SELECT col FROM table_b;

-- except / minus: rows in the first but not the second
SELECT col FROM table_a
EXCEPT                    -- Oracle: MINUS
SELECT col FROM table_b;

-- ORDER BY applies to the whole combined result, at the very end
SELECT col FROM table_a
UNION
SELECT col FROM table_b
ORDER BY col;
```

---

## Practice

Assume `customers (customer_id, first_name, city)` and `suppliers (supplier_id, contact_name, city)`.

**Beginner**
1. Return a single combined list of every distinct city that appears in either `customers` or `suppliers`.
2. Return a combined list of every `first_name` from `customers` and every `contact_name` from `suppliers`, keeping duplicates.
3. Return cities that appear in both `customers` and `suppliers`.
4. Return cities that have a supplier but no customer.

**Intermediate**
5. Return a tagged combined list of contacts (`first_name`/`contact_name`, `city`, and a `type` column of `'customer'` or `'supplier'`), sorted by city.
6. Rewrite question 3 using `WHERE ... IN` instead of `INTERSECT`.
7. Rewrite question 4 using `WHERE ... NOT IN` instead of `EXCEPT`.

<details>
<summary>Answers</summary>

```sql
-- 1
SELECT city FROM customers
UNION
SELECT city FROM suppliers;

-- 2
SELECT first_name FROM customers
UNION ALL
SELECT contact_name FROM suppliers;

-- 3
SELECT city FROM customers
INTERSECT
SELECT city FROM suppliers;

-- 4
SELECT city FROM suppliers
EXCEPT
SELECT city FROM customers;

-- 5
SELECT first_name AS contact, city, 'customer' AS type FROM customers
UNION ALL
SELECT contact_name, city, 'supplier' FROM suppliers
ORDER BY city;

-- 6
SELECT DISTINCT city FROM customers
WHERE city IN (SELECT city FROM suppliers);

-- 7
SELECT DISTINCT city FROM suppliers
WHERE city NOT IN (SELECT city FROM customers);
```

</details>

---

## Before moving on

You should be comfortable explaining the difference between a join and a set operation, know when to reach for `UNION` versus `UNION ALL`, and be able to use `INTERSECT`/`EXCEPT` (or their `WHERE ... IN` / `NOT IN` equivalents) to compare two result sets.

## Next chapter

**`08-aggregation`** - `GROUP BY`, `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, and `HAVING`. Set operations combine query results vertically; aggregation compresses rows within a single result, which is the next essential skill for summarising the joined data from Chapter 6.