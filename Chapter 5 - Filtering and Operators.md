# Chapter 5: Filtering and Operators

Chapter 1 introduced `WHERE` with basic comparisons (`=`, `>`, `<`, etc.). Real filtering usually needs more than one condition at a time - "customers from Bristol **and** over 30", "orders that are pending **or** cancelled", "names that start with J". This chapter covers the operators that make `WHERE` actually flexible: `AND`, `OR`, `NOT`, `BETWEEN`, `IN`, `LIKE`, and the `NULL` checks from Chapter 4.

## What you'll be able to do after this chapter

- Combine multiple conditions with `AND` and `OR`
- Understand why mixing `AND` and `OR` without parentheses is a common source of bugs
- Invert a condition with `NOT`
- Filter a range of values with `BETWEEN`
- Filter against a list of values with `IN`
- Pattern-match text with `LIKE` and wildcards
- Check for missing values with `IS NULL` / `IS NOT NULL`
- Combine all of the above into realistic, multi-condition filters

---

## 1. AND

`AND` requires **every** condition to be true for a row to be included.

```sql
SELECT *
FROM customers
WHERE city = 'Bristol' AND age > 25;
```

Only customers who are from Bristol **and** older than 25 are returned. If a Bristol customer is 24, they're excluded - one condition failing is enough to drop the row.

You can chain as many `AND`s as you need:

```sql
SELECT *
FROM employees
WHERE department = 'IT' AND salary > 40000 AND age < 40;
```

## 2. OR

`OR` requires **at least one** condition to be true.

```sql
SELECT *
FROM customers
WHERE city = 'Bristol' OR city = 'London';
```

Customers from either city are returned. Unlike `AND`, a row only needs to satisfy one side.

## 3. Mixing AND and OR - use parentheses

This is where people get tripped up. SQL evaluates `AND` before `OR` (similar to how multiplication is evaluated before addition in maths), so writing conditions without parentheses can silently give you a different result than you intended.

```sql
SELECT *
FROM customers
WHERE city = 'Bristol' OR city = 'London' AND age > 30;
```

It's tempting to read this as "Bristol or London, and over 30" - but that's not what it does. Because `AND` binds tighter, it's actually evaluated as:

```sql
WHERE city = 'Bristol' OR (city = 'London' AND age > 30)
```

That returns **every** Bristol customer regardless of age, plus only the London customers over 30 - probably not what was intended. If you want "either city, but only if over 30", you need explicit parentheses:

```sql
SELECT *
FROM customers
WHERE (city = 'Bristol' OR city = 'London') AND age > 30;
```

**Rule of thumb: any time you mix `AND` and `OR` in the same `WHERE`, use parentheses to make the grouping explicit** - even in cases where the default precedence happens to give you the right answer. It removes any ambiguity for the next person reading the query, including future you.

## 4. NOT

`NOT` inverts a condition - it returns rows where the condition is **false**.

```sql
SELECT *
FROM customers
WHERE NOT city = 'Bristol';
```

Equivalent to `city <> 'Bristol'` from Chapter 1, but `NOT` is more useful once conditions get more complex, since it can invert something bigger than a single comparison:

```sql
SELECT *
FROM customers
WHERE NOT (city = 'Bristol' AND age > 30);
```

This returns everyone **except** the Bristol customers over 30 - i.e., non-Bristol customers of any age, plus Bristol customers 30 or younger.

---

## 5. BETWEEN

`BETWEEN` filters a range, and it's inclusive on both ends.

```sql
SELECT *
FROM customers
WHERE age BETWEEN 25 AND 35;
```

This includes customers aged exactly 25 and exactly 35, not just the values strictly in between. It's shorthand for:

```sql
WHERE age >= 25 AND age <= 35
```

`BETWEEN` also works on dates and text, not just numbers:

```sql
SELECT *
FROM orders
WHERE order_date BETWEEN '2026-01-01' AND '2026-01-31';
```

**Careful with `DATETIME` columns** - if `order_date` actually stores a time component, `'2026-01-31'` means midnight at the very start of the 31st, so anything later that day gets excluded. When in doubt with datetime ranges, it's safer to write it as `>=` start `AND <` the day *after* the end.

You can invert it with `NOT BETWEEN`:

```sql
SELECT *
FROM customers
WHERE age NOT BETWEEN 25 AND 35;
```

## 6. IN

`IN` checks whether a value matches any item in a list - it replaces a long chain of `OR`s.

```sql
SELECT *
FROM customers
WHERE city IN ('Bristol', 'London', 'Manchester');
```

This is equivalent to, but much easier to read than:

```sql
WHERE city = 'Bristol' OR city = 'London' OR city = 'Manchester'
```

`IN` also works with numbers:

```sql
SELECT *
FROM orders
WHERE customer_id IN (1, 4, 7, 12);
```

Invert it with `NOT IN`:

```sql
SELECT *
FROM customers
WHERE city NOT IN ('Bristol', 'London');
```

**One trap worth knowing:** if the list passed to `NOT IN` contains a `NULL`, the whole condition can unexpectedly return no rows at all, because comparing anything to `NULL` is neither true nor false - it's unknown, and that propagates through the whole `NOT IN`. If there's any chance of `NULL`s being in that list, filter them out first or use a different approach (this comes up properly once you get to subqueries later in the repo).

### IN with a subquery

`IN` doesn't just take a hardcoded list - it can take the result of another query:

```sql
SELECT *
FROM customers
WHERE customer_id IN (
    SELECT customer_id FROM orders WHERE amount > 100
);
```

This returns customers who have placed at least one order over £100. Subqueries get their own proper chapter later, but this is worth knowing exists now since it's such a natural extension of `IN`.

---

## 7. LIKE

`LIKE` matches text against a pattern, using two wildcards:

- `%` - matches any sequence of characters (including none)
- `_` - matches exactly one character

```sql
SELECT *
FROM customers
WHERE first_name LIKE 'J%';
```

Matches any name starting with `J` - `John`, `James`, `Jane`, `J`. The `%` absorbs whatever comes after.

```sql
SELECT *
FROM customers
WHERE first_name LIKE '%a';
```

Matches any name ending in `a` - `Emma`, `Sarah`.

```sql
SELECT *
FROM customers
WHERE first_name LIKE '%ar%';
```

Matches any name containing `ar` anywhere - `Sarah`, `Mark`, `Carl`.

```sql
SELECT *
FROM customers
WHERE first_name LIKE '_ohn';
```

Matches a four-letter name ending in `ohn` - `John` matches, `Johnathan` does not (the underscore matches exactly one character, not "any number").

### Case sensitivity

Whether `LIKE` is case-sensitive depends on the database and its configuration. SQL Server, in a typical default setup, is case-**insensitive** for `LIKE` (`'john'` matches `'John'`). PostgreSQL is case-**sensitive** by default (use `ILIKE` there for case-insensitive matching). Worth checking rather than assuming, if a query isn't matching what you expect.

### Escaping literal % or _

If you actually need to match a literal `%` or `_` character rather than treat it as a wildcard, most databases support an `ESCAPE` clause:

```sql
SELECT *
FROM products
WHERE product_code LIKE '50\%%' ESCAPE '\';
```

This looks for codes literally starting with `50%`. Not something you'll need often, but worth knowing it exists rather than being stuck when a product code genuinely contains a percent sign.

---

## 8. IS NULL / IS NOT NULL

Covered briefly in Chapter 4 for `INSERT`/`UPDATE`; it belongs here too since it's fundamentally a filtering operator.

```sql
SELECT *
FROM customers
WHERE city IS NULL;
```

Returns customers with no city on record. As covered in Chapter 4, `= NULL` never works - `NULL` isn't a value you can be equal to, it has to be checked with `IS`.

```sql
SELECT *
FROM customers
WHERE city IS NOT NULL;
```

Returns customers who *do* have a city recorded.

---

## Worked example

Combining everything in this chapter against the `customers` table from earlier chapters:

**Find customers from Bristol or London, aged between 25 and 40, whose name starts with 'S', excluding anyone with no email on file.**

```sql
SELECT *
FROM customers
WHERE (city = 'Bristol' OR city = 'London')
  AND age BETWEEN 25 AND 40
  AND first_name LIKE 'S%'
  AND email IS NOT NULL;
```

Note the parentheses around the `OR` - without them, this query would silently behave differently (per the precedence rule earlier in this chapter).

**Find orders that are pending or cancelled, but not from customer 1.**

```sql
SELECT *
FROM orders
WHERE status IN ('pending', 'cancelled')
  AND customer_id <> 1;
```

**Find employees earning either under £30,000 or over £70,000 - i.e., excluding the middle band.**

```sql
SELECT *
FROM employees
WHERE salary < 30000 OR salary > 70000;
```

Equivalently, using `NOT BETWEEN`:

```sql
SELECT *
FROM employees
WHERE salary NOT BETWEEN 30000 AND 70000;
```

---

## Common mistakes

**Mixing AND/OR without parentheses.**
```sql
WHERE city = 'Bristol' OR city = 'London' AND age > 30   -- ambiguous, evaluates AND first
WHERE (city = 'Bristol' OR city = 'London') AND age > 30  -- explicit, correct
```

**Using `=` with a list instead of `IN`.**
```sql
WHERE city = 'Bristol', 'London'      -- invalid syntax
WHERE city IN ('Bristol', 'London')    -- correct
```

**Forgetting BETWEEN is inclusive.**
`age BETWEEN 18 AND 65` includes both 18 and 65 exactly. If you actually want to exclude one end, use `>` / `<` explicitly instead:
```sql
WHERE age > 18 AND age <= 65
```

**Using `=` instead of `LIKE` when a wildcard is intended.**
```sql
WHERE first_name = 'J%'    -- looks for a name that is literally the text "J%", matches nothing
WHERE first_name LIKE 'J%'  -- correct, matches names starting with J
```

**Using `= NULL` instead of `IS NULL`.**
```sql
WHERE city = NULL       -- never matches anything
WHERE city IS NULL      -- correct
```

**NOT IN with a NULL in the list.**
```sql
WHERE city NOT IN ('Bristol', NULL)   -- can unexpectedly return zero rows
```
If the list might contain `NULL`s (especially when it comes from a subquery), filter them out first or restructure the query.

---

## Quick reference

```sql
-- AND / OR / NOT
WHERE condition1 AND condition2
WHERE condition1 OR condition2
WHERE NOT condition
WHERE (condition1 OR condition2) AND condition3   -- parenthesize when mixing

-- BETWEEN (inclusive both ends)
WHERE column BETWEEN low AND high
WHERE column NOT BETWEEN low AND high

-- IN
WHERE column IN (value1, value2, value3)
WHERE column NOT IN (value1, value2, value3)
WHERE column IN (SELECT column FROM other_table WHERE condition)

-- LIKE
WHERE column LIKE 'A%'     -- starts with A
WHERE column LIKE '%A'      -- ends with A
WHERE column LIKE '%A%'     -- contains A anywhere
WHERE column LIKE '_A%'     -- second character is A
WHERE column NOT LIKE 'A%'

-- NULL
WHERE column IS NULL
WHERE column IS NOT NULL
```

---

## Practice

Assume `customers (customer_id, first_name, last_name, city, age, email)` and `orders (order_id, customer_id, order_date, amount, status)`.

**Beginner**
1. Return customers from Bristol who are also older than 30.
2. Return customers from either Bristol or Manchester.
3. Return customers whose age is between 20 and 30 inclusive.
4. Return customers whose first name starts with 'D'.
5. Return customers with no email on file.

**Intermediate**
6. Return customers from Bristol, London, or Leeds, ordered by age descending.
7. Return orders with a status of either 'pending' or 'processing', with an amount over £50.
8. Return customers whose last name ends in 'son'.
9. Return customers who are NOT from Bristol and NOT from London.
10. Return orders placed in March 2026 (assume `order_date` has no time component).

<details>
<summary>Answers</summary>

```sql
-- 1
SELECT * FROM customers
WHERE city = 'Bristol' AND age > 30;

-- 2
SELECT * FROM customers
WHERE city IN ('Bristol', 'Manchester');

-- 3
SELECT * FROM customers
WHERE age BETWEEN 20 AND 30;

-- 4
SELECT * FROM customers
WHERE first_name LIKE 'D%';

-- 5
SELECT * FROM customers
WHERE email IS NULL;

-- 6
SELECT * FROM customers
WHERE city IN ('Bristol', 'London', 'Leeds')
ORDER BY age DESC;

-- 7
SELECT * FROM orders
WHERE status IN ('pending', 'processing')
  AND amount > 50;

-- 8
SELECT * FROM customers
WHERE last_name LIKE '%son';

-- 9
SELECT * FROM customers
WHERE city NOT IN ('Bristol', 'London');

-- 10
SELECT * FROM orders
WHERE order_date BETWEEN '2026-03-01' AND '2026-03-31';
```

</details>

---

## Before moving on

You should be comfortable combining conditions with `AND`/`OR` and knowing when parentheses are non-negotiable, filtering ranges with `BETWEEN`, filtering lists with `IN`, pattern-matching with `LIKE`, and checking for missing data with `IS NULL`/`IS NOT NULL` - and combining several of these in a single realistic `WHERE` clause.

## Next chapter

**`06-joins`** - combining rows from two or more tables based on a related column. This is where `customers` and `orders` stop being separate tables you query one at a time, and start being something you can query together.