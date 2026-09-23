# SQL Views

A view is a saved SQL query that you can treat like a table. It doesn't store data itself — every time you query a view, the underlying query runs and returns fresh results.

---

## Why use views

Say you have a complex join query that you run all the time:

```sql
SELECT 
    u.id,
    u.name,
    u.email,
    COUNT(o.id) AS total_orders,
    SUM(o.amount) AS total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name, u.email;
```

Instead of writing this every time, wrap it in a view:

```sql
CREATE VIEW user_order_summary AS
SELECT 
    u.id,
    u.name,
    u.email,
    COUNT(o.id) AS total_orders,
    SUM(o.amount) AS total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name, u.email;
```

Now query it like a table:

```sql
SELECT * FROM user_order_summary;
SELECT * FROM user_order_summary WHERE total_spent > 10000;
SELECT name, total_orders FROM user_order_summary ORDER BY total_orders DESC;
```

Much cleaner.

---

## Creating a view

```sql
CREATE VIEW active_users AS
SELECT id, name, email
FROM users
WHERE status = 'ACTIVE';
```

```sql
-- query it
SELECT * FROM active_users;
```

---

## Updating a view

```sql
CREATE OR REPLACE VIEW active_users AS
SELECT id, name, email, created_at
FROM users
WHERE status = 'ACTIVE';
```

`CREATE OR REPLACE` updates the view if it exists, creates it if it doesn't.

---

## Dropping a view

```sql
DROP VIEW IF EXISTS active_users;
```

---

## Views for security

Views can restrict which columns users can see. Instead of giving direct table access, give access to a view that only exposes safe columns.

```sql
-- users table has: id, name, email, password, salary, internal_notes
-- create a view that hides sensitive columns
CREATE VIEW public_user_info AS
SELECT id, name, email
FROM users;

-- give external users access only to this view, not the base table
```

---

## Can you update data through a view

Simple views on a single table — yes, sometimes:

```sql
UPDATE active_users SET name = 'Rohit Kumar' WHERE id = 42;
```

But views with JOINs, GROUP BY, DISTINCT, aggregate functions — generally not updatable. Those are read-only views.

---

## Views vs Materialized Views

Regular view — runs the query every time you select from it. Always fresh data, but can be slow for complex queries.

Materialized view — stores the query result physically. Fast reads but data can be stale until you refresh it.

MySQL doesn't support materialized views natively. PostgreSQL does. In MySQL you'd simulate it with a table that you refresh periodically.

---

## Stuff I want to remember

**What is a view and how is it different from a table?**

A view is a saved SQL query — it doesn't store data itself. When you query a view, the underlying query runs and returns current results. A table actually stores data. Views are useful for simplifying complex queries, reusing logic, and restricting column access.

**Can you always update data through a view?**

No. Simple single-table views without aggregates or DISTINCT are usually updatable. Views with JOINs, GROUP BY, aggregate functions or subqueries are typically read-only. Whether a view is updatable depends on the database and the view definition.

**What is the difference between a view and a materialized view?**

A regular view runs its query every time it's accessed — always fresh but can be slow. A materialized view stores the result physically — fast reads but needs to be refreshed to get updated data. MySQL doesn't support materialized views natively, PostgreSQL does.

---

*Started using views after getting tired of copying the same 20-line join query into different places. One view definition, used everywhere — much easier to maintain.*