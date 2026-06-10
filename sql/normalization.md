# Database Normalization — 1NF, 2NF, 3NF

Normalization is the process of organizing your database tables to reduce redundancy and avoid data anomalies. You split data into smaller, well-structured tables and define relationships between them.

There are multiple normal forms — 1NF, 2NF, 3NF, BCNF and more. In practice 3NF is what most applications aim for.

---

## Why it matters

Without normalization you end up with:

- **Update anomaly** — updating one piece of data requires updating it in multiple places
- **Insert anomaly** — can't insert data without adding unrelated data
- **Delete anomaly** — deleting one record accidentally removes other needed data

---

## Unnormalized table — the starting point

```
+--------+----------+-------------------+----------+---------+
| order  | customer | customer_email    | product  | price   |
+--------+----------+-------------------+----------+---------+
| 1      | Rohit    | rohit@gmail.com   | Laptop   | 50000   |
| 1      | Rohit    | rohit@gmail.com   | Phone    | 15000   |
| 2      | Sneha    | sneha@gmail.com   | Tablet   | 25000   |
+--------+----------+-------------------+----------+---------+
```

Problems — Rohit's email is repeated. If it changes you have to update multiple rows. If you delete order 2, Sneha's info is gone entirely.

---

## First Normal Form (1NF)

Rules:
- Each column must have atomic (single) values — no comma separated lists
- Each row must be unique
- No repeating groups

Bad — violates 1NF:

```
+--------+----------+------------------------+
| order  | customer | products               |
+--------+----------+------------------------+
| 1      | Rohit    | Laptop, Phone          |
| 2      | Sneha    | Tablet                 |
+--------+----------+------------------------+
```

The `products` column has multiple values in one cell. That's not atomic.

Fixed — 1NF:

```
+----------+--------+----------+---------+
| order_id | cust   | product  | price   |
+----------+--------+----------+---------+
| 1        | Rohit  | Laptop   | 50000   |
| 1        | Rohit  | Phone    | 15000   |
| 2        | Sneha  | Tablet   | 25000   |
+----------+--------+----------+---------+
```

One product per row. Each row is unique with a composite key (order_id + product).

---

## Second Normal Form (2NF)

Must be in 1NF first. Then — every non-key column must depend on the entire primary key, not just part of it.

This only applies when the primary key is composite (multiple columns).

In our table the primary key is (order_id + product). But `cust` (customer name) only depends on `order_id`, not on `product`. That's a partial dependency — violates 2NF.

Fix — split into two tables:

```
-- orders table
+----------+-------+
| order_id | cust  |
+----------+-------+
| 1        | Rohit |
| 2        | Sneha |
+----------+-------+

-- order_items table
+----------+----------+---------+
| order_id | product  | price   |
+----------+----------+---------+
| 1        | Laptop   | 50000   |
| 1        | Phone    | 15000   |
| 2        | Tablet   | 25000   |
+----------+----------+---------+
```

Now customer name depends on order_id (its full key). Product and price depend on (order_id + product) — their full composite key.

---

## Third Normal Form (3NF)

Must be in 2NF first. Then — no transitive dependencies. Non-key columns should not depend on other non-key columns.

Say we add customer email to the orders table:

```
+----------+-------+-------------------+----------+
| order_id | cust  | customer_email    | city     |
+----------+-------+-------------------+----------+
| 1        | Rohit | rohit@gmail.com   | Pune     |
| 2        | Sneha | sneha@gmail.com   | Mumbai   |
+----------+-------+-------------------+----------+
```

Here `customer_email` and `city` depend on `cust` (customer name), not on `order_id`. That's a transitive dependency — violates 3NF.

Fix — move customer data to its own table:

```
-- customers table
+---------+-------+-------------------+--------+
| cust_id | name  | email             | city   |
+---------+-------+-------------------+--------+
| 1       | Rohit | rohit@gmail.com   | Pune   |
| 2       | Sneha | sneha@gmail.com   | Mumbai |
+---------+-------+-------------------+--------+

-- orders table
+----------+---------+
| order_id | cust_id |
+----------+---------+
| 1        | 1       |
| 2        | 2       |
+----------+---------+
```

Now everything in each table depends only on its own primary key. No transitive dependencies.

---

## Quick summary

| Normal Form | Rule |
|---|---|
| 1NF | Atomic values, no repeating groups |
| 2NF | No partial dependencies (non-key depends on full key) |
| 3NF | No transitive dependencies (non-key depends only on key) |

---

## Stuff I want to remember

**What is the point of normalization?**

To eliminate redundancy and prevent data anomalies. If the same data exists in multiple places and you update it in one place but forget another, your database is inconsistent. Normalization ensures each piece of data lives in exactly one place.

**What is a transitive dependency?**

When a non-key column depends on another non-key column instead of depending directly on the primary key. Example — if `city` depends on `customer_name` and `customer_name` depends on `order_id`, then `city` has a transitive dependency on `order_id` through `customer_name`. 3NF removes this.

**Is more normalization always better?**

Not always. Highly normalized databases require more joins to fetch data, which can affect performance. Sometimes denormalization (intentionally adding some redundancy) is used in read-heavy systems for speed. It's a tradeoff between data integrity and query performance.

---

*Normalization made much more sense when I stopped thinking of it as rules to memorize and started thinking of it as — where does this data actually belong?*