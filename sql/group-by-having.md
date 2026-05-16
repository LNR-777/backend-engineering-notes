# GROUP BY and HAVING

GROUP BY is used to group rows that have the same value in a column and then perform aggregate functions on each group — like COUNT, SUM, AVG, MAX, MIN.

HAVING is like WHERE but for groups. WHERE filters rows before grouping, HAVING filters after.

---

## Basic GROUP BY

Same tables from the joins note:

```sql
-- users table
+----+--------+
| id | name   |
+----+--------+
| 1  | Rohit  |
| 2  | Sneha  |
| 3  | Arjun  |
+----+--------+

-- orders table
+----+---------+---------+--------+
| id | user_id | product | amount |
+----+---------+---------+--------+
| 1  | 1       | Laptop  | 50000  |
| 2  | 1       | Phone   | 15000  |
| 3  | 2       | Tablet  | 25000  |
| 4  | 2       | Watch   | 8000   |
| 5  | 2       | Earbuds | 3000   |
+----+---------+---------+--------+
```

How many orders does each user have?

```sql
SELECT user_id, COUNT(*) AS total_orders
FROM orders
GROUP BY user_id;
```

Result:

```
+---------+--------------+
| user_id | total_orders |
+---------+--------------+
| 1       | 2            |
| 2       | 3            |
+---------+--------------+
```

GROUP BY collapsed all rows with the same user_id into one row, then COUNT counted how many rows were in each group.

---

## With JOIN — making it readable

```sql
SELECT users.name, COUNT(orders.id) AS total_orders
FROM users
LEFT JOIN orders ON users.id = orders.user_id
GROUP BY users.id, users.name;
```

Result:

```
+-------+--------------+
| name  | total_orders |
+-------+--------------+
| Rohit | 2            |
| Sneha | 3            |
| Arjun | 0            |
+-------+--------------+
```

LEFT JOIN so Arjun shows up with 0. Then GROUP BY groups by user so we get one row per user.

---

## SUM and AVG

Total amount spent per user:

```sql
SELECT user_id, SUM(amount) AS total_spent
FROM orders
GROUP BY user_id;
```

Average order amount per user:

```sql
SELECT user_id, AVG(amount) AS avg_order_value
FROM orders
GROUP BY user_id;
```

---

## HAVING — filtering groups

Now — only show users who have placed more than 2 orders.

This is where people try to use WHERE and it doesn't work:

```sql
-- WRONG — this throws an error
SELECT user_id, COUNT(*) AS total_orders
FROM orders
WHERE COUNT(*) > 2
GROUP BY user_id;
```

WHERE runs before grouping happens so it doesn't know about COUNT yet. Use HAVING instead:

```sql
-- CORRECT
SELECT user_id, COUNT(*) AS total_orders
FROM orders
GROUP BY user_id
HAVING COUNT(*) > 2;
```

Result:

```
+---------+--------------+
| user_id | total_orders |
+---------+--------------+
| 2       | 3            |
+---------+--------------+
```

Only Sneha (user_id 2) had more than 2 orders so only she shows up.

---

## WHERE vs HAVING together

You can use both in the same query. WHERE filters rows first, then GROUP BY groups them, then HAVING filters the groups.

Show users who spent more than 10000 in total, but only consider orders above 5000:

```sql
SELECT user_id, SUM(amount) AS total_spent
FROM orders
WHERE amount > 5000
GROUP BY user_id
HAVING SUM(amount) > 10000;
```

Step by step what happens here:
1. WHERE removes orders where amount <= 5000 (Earbuds 3000 gets removed)
2. GROUP BY groups remaining rows by user_id
3. HAVING keeps only groups where total is > 10000

---

## Order of execution in SQL

This is something worth knowing — SQL doesn't execute in the order you write it:

```
FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

That's why you can't use a column alias from SELECT inside WHERE or HAVING — SELECT hasn't run yet at that point.

---

## Stuff I want to remember

**What's the difference between WHERE and HAVING?**

WHERE filters individual rows before grouping. HAVING filters groups after GROUP BY runs. If I want to filter based on an aggregate like COUNT or SUM, I have to use HAVING — WHERE doesn't have access to those values yet.

**Can you use HAVING without GROUP BY?**

Technically yes but it doesn't make much sense. Without GROUP BY the whole table is treated as one group. You'd almost always use HAVING with GROUP BY.

**Why do you GROUP BY all non-aggregated columns in SELECT?**

Because if you're selecting `name` and `COUNT(*)`, the database needs to know how to group the name. If you don't include it in GROUP BY, the database doesn't know which name to show for each group. Most databases will throw an error, MySQL in strict mode does too.

---

*This made more sense once I stopped thinking of GROUP BY as just "grouping" and started thinking of it as — collapse these rows into one, then apply the aggregate function.*