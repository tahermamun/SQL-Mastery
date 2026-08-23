# Project: RetailFlow — A Mini E-Commerce Database

A single project that forces you to actually use everything from Chapters 1–7 together, instead of each concept living in its own isolated exercise. Build it once, properly, and you'll remember all of it a lot better than doing scattered practice questions — and it's something real you can point to on a CV or in an interview.

## The scenario

You're the sole database person for **RetailFlow**, a small online retailer. You need to design the database from scratch, load it with realistic data, and then answer a set of real business questions the (imaginary) management team is asking you. That's genuinely what the job looks like in practice — this isn't a toy exercise dressed up as a project, it's a compressed version of real work.

---

## What you'll end up with

By the end you'll have, in your own repo:

- A schema design (an ER diagram, even a rough hand-drawn one photographed and added to the README)
- A `schema.sql` file — every `CREATE TABLE` statement, with proper constraints
- A `seed_data.sql` file — enough realistic sample data to make queries meaningful
- A `queries.sql` file — the business questions below, answered and commented
- A short `README.md` explaining the project, the schema, and what you learned

This is the version that's actually worth putting on a CV: not "completed SQL course" but "designed and built a normalised e-commerce database, wrote 25+ business-intelligence queries against it." Employers can ask you to walk through it in an interview, and you'll actually be able to.

---

## Phase 1 — Design the schema (uses Chapter 3: DDL)

Design and create these tables. Don't just copy structure from earlier chapters — think about each one properly:

- **customers** — customer_id, first_name, last_name, email, city, signup_date
- **categories** — category_id, category_name
- **products** — product_id, product_name, category_id (FK), price, stock_quantity
- **suppliers** — supplier_id, supplier_name, city
- **product_suppliers** — links products to suppliers (a product can have more than one supplier — this is a many-to-many relationship, which needs its own linking table)
- **orders** — order_id, customer_id (FK), order_date, status
- **order_items** — order_id (FK), product_id (FK), quantity, price_at_purchase

### Requirements to actually apply what you learned

- Every table has a sensible `PRIMARY KEY`
- Every relationship is enforced with a real `FOREIGN KEY` — don't just name columns similarly and hope
- `NOT NULL` on anything that should always have a value (an order without a customer_id shouldn't be possible)
- At least one `CHECK` constraint (price > 0, quantity > 0 — pick a couple)
- At least one `DEFAULT` (order status defaulting to `'pending'`, signup_date defaulting to today)
- A `UNIQUE` constraint on customer email

Write this as one `schema.sql` file, with tables created in an order that respects foreign key dependencies (parent tables before the tables that reference them).

---

## Phase 2 — Load realistic data (uses Chapter 4: DML)

Insert data using `INSERT`, not by hand-typing one row at a time forever — batch inserts where sensible.

Minimum realistic size to make the later queries actually interesting:

- 20+ customers, across at least 5 different cities
- 5–8 categories
- 30+ products, spread across those categories, with a real spread of prices
- 5+ suppliers
- 40+ orders, spread across several months, with a mix of statuses (`pending`, `shipped`, `cancelled`, `delivered`)
- Order items linking orders to products — several items per order on average

**Deliberately include some messiness**, because real data is never clean and your later queries should have to deal with it:

- A few customers with a `NULL` email or `NULL` city
- At least one customer with zero orders
- At least one product that's never been ordered
- A few orders with a `quantity` that would cause a divide-by-zero if you're not careful (ties back to Chapter 8's `NULLIF` trick, if you've done that chapter too)

Then practice a couple of `UPDATE`/`DELETE` statements deliberately, inside a transaction, as covered in Chapter 4 — e.g. mark a batch of old pending orders as cancelled, or update a product's price.

---

## Phase 3 — Answer the business questions

This is the real practice. Each question maps to a chapter — try to answer it cold before checking which chapter it's testing, since that's closer to how it works in an actual job (nobody labels the problem "this is a JOIN question" for you).

### Filtering (Chapter 5)

1. List all customers from Bristol or Manchester who signed up in the last 6 months.
2. Find all products priced between £10 and £50.
3. Find all orders that are either `pending` or `processing`, excluding a specific customer.
4. Find all customers whose email is missing.
5. Find all products whose name contains the word "wireless" (case-insensitive).

### Joins (Chapter 6)

6. List every order with the customer's name and city.
7. List every customer along with their total number of orders, including customers with zero orders.
8. Find customers who have never placed an order.
9. Find products that have never been ordered.
10. List every order item with the product name, category name, and customer name — a four-table join.
11. Find every supplier along with the products they supply, including suppliers currently supplying nothing.

### Set operations (Chapter 7)

12. Produce a single list of every city that has either a customer or a supplier.
13. Find cities that have both a customer and a supplier based there.
14. Find cities that have a customer but no supplier — a potential "delivery coverage gap" list for the business.
15. Combine two customer segments — "signed up in the last month" and "placed an order over £100" — into one deduplicated contact list.

### Mixed / realistic (everything together)

16. Find the 5 most recent orders, with customer name and order total.
17. Find all customers in Bristol who have placed at least one cancelled order.
18. List categories where every product is currently priced under £20.
19. Update every `pending` order older than 30 days to `cancelled` — inside a transaction, checking the affected rows first.
20. Find suppliers who supply products in more than one category.
21. Produce a "customer contact sheet": name, email (or `'no email on file'` if missing), city (or `'Unknown'`), and total lifetime spend — using `COALESCE` if you've reached Chapter 8, otherwise just `WHERE ... IS NULL` handling.
22. Find the customer(s) with the highest total spend.
23. Find products that are in stock (`stock_quantity > 0`) but haven't sold a single unit.
24. List every order, and whether it was placed by a "new" customer (signed up in the same month as the order) or an "existing" one — a `CASE` expression, if you've done Chapter 8.
25. Write your own question — pick something you'd actually want to know if you ran this business, and answer it.

---

## Stretch goals (optional, but good CV material)

- Add a `reviews` table (customer_id, product_id, rating, review_text) and answer: which products have the best average rating? (This edges into aggregation — a good preview of the next chapter.)
- Export the results of a couple of your favourite queries to CSV and build a very simple chart from them (even in Excel) — shows you can take a query result somewhere useful.
- Write a short paragraph in your README on one query that surprised you, or one mistake you made and fixed (an accidental cross join, a `LEFT JOIN` that quietly turned into an `INNER JOIN` because of a misplaced `WHERE` — anything from the "common mistakes" sections in earlier chapters is fair game here).

---

## Why this is worth doing properly

A lot of SQL practice is disconnected — one exercise per concept, nothing carried forward. This project is deliberately the opposite: the schema you design in Phase 1 has consequences in Phase 2 (bad constraints mean bad data gets in), and the data you load in Phase 2 has consequences in Phase 3 (lazy sample data makes for boring, easy queries). That's a much closer simulation of an actual job than isolated drills — and it's the version of "I practiced SQL" that's actually interesting to talk about in an interview.

Put it in its own GitHub repo, with `schema.sql`, `seed_data.sql`, `queries.sql`, and a README. That's a genuinely presentable portfolio piece.