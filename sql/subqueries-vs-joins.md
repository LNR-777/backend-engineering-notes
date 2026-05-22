# Subqueries vs JOINs

Both can give you the same result in a lot of cases. The question is — which one to use when.

A subquery is a query inside another query. A JOIN combines rows from multiple tables. Sometimes they're interchangeable, sometimes one is clearly better than the other.

---

## Same result, different approach

Say you want to find all users who have placed at least one order.

**Using JOIN:**

```sql
SELECT DISTINCT users.name
FROM users
INNER JOIN orders ON users.id = orders.user_id;
```

**Using Subquery:**

```sql
SELECT name
FROM users
WHERE id IN (SELECT user_id FROM orders);
```

Both return the same thing. Which one is better depends on the situation.

---

## Types of Subqueries

### Subquery in WHERE

Most common. Filter rows based on a result from another query.

```sql
-- users who spent more than the average order amount
SELECT name
FROM users
WHERE id IN (
    SELECT user_id
    FROM orders
    WHERE amount > (SELECT AVG(amount) FROM orders)
);
```

You can't easily do this with a simple JOIN — nested aggregates like this are where subqueries shine.

---

### Subquery in FROM (Derived Table)

Treat the subquery result as a temporary table.

```sql
SELECT user_id, total
FROM (
    SELECT user_id, SUM(amount) AS total
    FROM orders
    GROUP BY user_id
) AS order_totals
WHERE total > 20000;
```

The inner query runs first, produces a result set, then the outer query filters it. This is useful when you need to filter on an aggregated value.

---

### Correlated Subquery

This one runs once for every row in the outer query — so it can be slow on large datasets.

```sql
-- find users whose total orders are above average
SELECT name
FROM users u
WHERE (
    SELECT COUNT(*)
    FROM orders o
    WHERE o.user_id = u.id
) > 2;
```

See how the inner query references `u.id` from the outer query? That's what makes it correlated. It re-runs for every user row.

---

## When to use JOIN vs Subquery

**Use JOIN when:**
- You need columns from multiple tables in the result
- Performance matters — JOINs are generally faster and the query optimizer handles them better
- You're doing straightforward lookups across related tables

```sql
-- need both user name and order details — JOIN makes sense
SELECT users.name, orders.product, orders.amount
FROM users
INNER JOIN orders ON users.id = orders.user_id;
```

**Use Subquery when:**
- You only need data from one table but the filter depends on another
- You're working with aggregates inside conditions
- The logic is cleaner to read as a subquery

```sql
-- only need user names, filter depends on orders — subquery is cleaner
SELECT name
FROM users
WHERE id IN (SELECT user_id FROM orders WHERE amount > 10000);
```

---

## EXISTS vs IN

Two ways to write subqueries for existence checks — and they behave differently with NULLs.

```sql
-- using IN
SELECT name FROM users
WHERE id IN (SELECT user_id FROM orders);

-- using EXISTS
SELECT name FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);
```

`EXISTS` stops as soon as it finds one match — it's faster when the subquery result is large. `IN` fetches all results first then checks. If the subquery can return NULLs, `IN` can give unexpected results — `EXISTS` is safer in those cases.

---

## Stuff I want to remember

**When would you choose a subquery over a JOIN?**

When I only need data from one table but the condition depends on another. Also when I need to filter on an aggregate — like finding users whose total spend is above average. That kind of nested logic is cleaner as a subquery. For most other cases I prefer JOINs because they're easier to read and generally perform better.

**What is a correlated subquery and why is it slow?**

A correlated subquery references a column from the outer query, so it has to re-execute for every row the outer query processes. If the outer table has 10,000 rows, the subquery runs 10,000 times. That's expensive. Better to rewrite it as a JOIN or use EXISTS where possible.

**What's the difference between IN and EXISTS?**

IN fetches all results from the subquery and then checks membership. EXISTS just checks if at least one row matches and stops — so it's faster when the subquery returns a lot of rows. Also EXISTS handles NULLs more predictably than IN.

---

*Rewrote a correlated subquery as a JOIN in a project once and the query went from taking 3 seconds to under 100ms. That's when I actually understood why this matters.*