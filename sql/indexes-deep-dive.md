# SQL Indexes — Deep Dive

We covered the basics of indexes earlier. This note goes deeper — types of indexes, when to add them, when not to, and how to check if your queries are using them.

---

## Quick recap

An index is a separate data structure that speeds up data retrieval. Without an index, MySQL scans every row to find a match (full table scan). With an index, it goes directly to the right rows.

The tradeoff — indexes speed up reads but slow down writes (INSERT, UPDATE, DELETE) because the index also needs to be updated.

---

## Types of indexes

### Primary Index

Created automatically on the primary key. Always unique. Every table has one.

```sql
CREATE TABLE users (
    id INT PRIMARY KEY,  -- index created automatically
    name VARCHAR(100)
);
```

---

### Unique Index

Ensures all values in a column are unique. Created automatically for UNIQUE constraints.

```sql
CREATE TABLE users (
    id INT PRIMARY KEY,
    email VARCHAR(100) UNIQUE  -- unique index created automatically
);
```

Or explicitly:

```sql
CREATE UNIQUE INDEX idx_email ON users(email);
```

---

### Composite Index

Index on multiple columns. Order matters.

```sql
CREATE INDEX idx_name_city ON users(name, city);
```

This index helps queries that filter on `name` alone or `name + city` together. It does NOT help queries that filter on `city` alone.

```sql
-- uses the index
SELECT * FROM users WHERE name = 'Rohit';
SELECT * FROM users WHERE name = 'Rohit' AND city = 'Pune';

-- does NOT use the index
SELECT * FROM users WHERE city = 'Pune';
```

This is called the leftmost prefix rule — the index is only used starting from the leftmost column.

---

### Full Text Index

For searching text content. Regular indexes can't efficiently handle `LIKE '%word%'` searches.

```sql
CREATE FULLTEXT INDEX idx_content ON articles(content);

SELECT * FROM articles WHERE MATCH(content) AGAINST('spring boot');
```

---

## When to add an index

Add an index on columns that are:

- Frequently used in WHERE clauses
- Used in JOIN conditions
- Used in ORDER BY or GROUP BY
- Foreign keys

```sql
-- if you frequently query orders by user_id
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- if you frequently filter products by category and sort by price
CREATE INDEX idx_category_price ON products(category, price);
```

---

## When NOT to add an index

- Small tables — full table scan is fast enough, index overhead isn't worth it
- Columns with very few distinct values — like a boolean `is_active` column. Index on a column with only 2 values doesn't help much
- Tables with heavy write operations — every INSERT/UPDATE/DELETE updates all indexes on that table
- Columns rarely used in queries

Don't index everything by default. Too many indexes hurt write performance.

---

## EXPLAIN — check if your query uses an index

```sql
EXPLAIN SELECT * FROM users WHERE email = 'rohit@gmail.com';
```

Output:

```
+----+-------------+-------+------+---------------+-----------+---------+-------+------+-------+
| id | select_type | table | type | possible_keys | key       | key_len | ref   | rows | Extra |
+----+-------------+-------+------+---------------+-----------+---------+-------+------+-------+
| 1  | SIMPLE      | users | ref  | idx_email     | idx_email | 403     | const | 1    | NULL  |
+----+-------------+-------+------+---------------+-----------+---------+-------+------+-------+
```

Key things to check:
- `key` — which index is being used. If NULL, no index is used.
- `type` — `const` or `ref` is good. `ALL` means full table scan — bad for large tables.
- `rows` — how many rows MySQL estimated it needs to scan.

---

## Common mistake — index not used because of function on column

```sql
-- index on created_at exists, but this won't use it
SELECT * FROM orders WHERE YEAR(created_at) = 2024;

-- this will use the index
SELECT * FROM orders WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';
```

Wrapping a column in a function prevents the index from being used. Always filter on the raw column value.

---

## Stuff I want to remember

**What is the leftmost prefix rule for composite indexes?**

A composite index on (col1, col2, col3) can be used by queries filtering on col1, or col1+col2, or col1+col2+col3. But not col2 alone or col3 alone. The query must start from the leftmost column of the index. So the order of columns in a composite index matters — put the most commonly filtered column first.

**How do you check if a query is doing a full table scan?**

Use EXPLAIN before the query. Look at the `type` column — if it says `ALL` that's a full table scan. Also check the `key` column — if it's NULL, no index is being used. Both together tell you the query needs optimization.

**Why do indexes slow down writes?**

Because when you INSERT, UPDATE, or DELETE a row, MySQL has to update all the indexes on that table too — not just the data. More indexes means more work on every write. That's why you should only add indexes that are actually needed, not index every column by default.

---

*Used EXPLAIN for the first time when a query was taking 3+ seconds on a large table. Turned out it was doing a full table scan. Added an index on the WHERE column and it dropped to milliseconds.*