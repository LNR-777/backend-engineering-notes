# SQL Joins

Joins are used to fetch data from multiple tables based on a related column between them.

Most real-world queries involve more than one table — so understanding joins is not optional. This is also one of the most common topics in backend interviews.

I'll use two simple tables throughout this note:

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
+----+---------+---------+
| id | user_id | product |
+----+---------+---------+
| 1  | 1       | Laptop  |
| 2  | 1       | Phone   |
| 3  | 2       | Tablet  |
+----+---------+---------+
```

Arjun (id=3) has no orders. Keep that in mind — it matters for understanding the difference between join types.

---

## INNER JOIN

Returns only the rows where there's a match in both tables.

```sql
SELECT users.name, orders.product
FROM users
INNER JOIN orders ON users.id = orders.user_id;
```

Result:

```
+-------+---------+
| name  | product |
+-------+---------+
| Rohit | Laptop  |
| Rohit | Phone   |
| Sneha | Tablet  |
+-------+---------+
```

Arjun doesn't show up because he has no orders. No match = not included.

---

## LEFT JOIN

Returns all rows from the left table, and matched rows from the right. If there's no match, right side columns come back as NULL.

```sql
SELECT users.name, orders.product
FROM users
LEFT JOIN orders ON users.id = orders.user_id;
```

Result:

```
+-------+---------+
| name  | product |
+-------+---------+
| Rohit | Laptop  |
| Rohit | Phone   |
| Sneha | Tablet  |
| Arjun | NULL    |
+-------+---------+
```

Arjun shows up now with NULL for product. Left join keeps everyone from the left table regardless of whether they have a match.

---

## RIGHT JOIN

Opposite of LEFT JOIN. Returns all rows from the right table, matched rows from the left.

```sql
SELECT users.name, orders.product
FROM users
RIGHT JOIN orders ON users.id = orders.user_id;
```

In this case the result is same as INNER JOIN because every order has a valid user. Right join would show NULLs if there were orders with no matching user.

Honestly I rarely use RIGHT JOIN in practice — most of the time you can just flip the tables and use LEFT JOIN instead. Same result, easier to read.

---

## FULL OUTER JOIN

Returns all rows from both tables. NULLs wherever there's no match on either side.

```sql
SELECT users.name, orders.product
FROM users
FULL OUTER JOIN orders ON users.id = orders.user_id;
```

Result:

```
+-------+---------+
| name  | product |
+-------+---------+
| Rohit | Laptop  |
| Rohit | Phone   |
| Sneha | Tablet  |
| Arjun | NULL    |
+-------+---------+
```

Note — MySQL doesn't support FULL OUTER JOIN directly. You'd simulate it with a LEFT JOIN + UNION + RIGHT JOIN.

```sql
SELECT users.name, orders.product
FROM users
LEFT JOIN orders ON users.id = orders.user_id

UNION

SELECT users.name, orders.product
FROM users
RIGHT JOIN orders ON users.id = orders.user_id;
```

---

## A more real example

In a backend project — say you want all users and their total number of orders:

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
| Sneha | 1            |
| Arjun | 0            |
+-------+--------------+
```

LEFT JOIN here because we want all users — including Arjun who has 0 orders. INNER JOIN would have excluded him.

---

## Quick comparison

| Join Type | What it returns |
|---|---|
| INNER JOIN | Only matching rows from both tables |
| LEFT JOIN | All rows from left + matched from right (NULL if no match) |
| RIGHT JOIN | All rows from right + matched from left (NULL if no match) |
| FULL OUTER JOIN | All rows from both tables (NULL where no match) |

---

## Stuff I want to remember

**What's the difference between INNER JOIN and LEFT JOIN?**

INNER JOIN only returns rows where there's a match in both tables. LEFT JOIN returns everything from the left table — even if there's no match on the right side, those rows still show up with NULL. So if I want all users regardless of whether they have orders or not, I use LEFT JOIN.

**When would you use LEFT JOIN over INNER JOIN?**

When I don't want to lose records just because there's no related data in the other table. Like showing all users with their order count — users with zero orders should still appear, so LEFT JOIN with COUNT is the right approach.

**Does MySQL support FULL OUTER JOIN?**

No, MySQL doesn't have FULL OUTER JOIN. You simulate it by doing a LEFT JOIN and a RIGHT JOIN and combining them with UNION.

---

*Practiced these on a users-orders schema while building a small project — made a lot more sense once I ran the queries and saw which rows showed up and which didn't.*