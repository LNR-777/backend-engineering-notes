# SQL Indexing Basics

Indexing is used to improve database query performance.

Indexes help the database find records faster without scanning the entire table.

---

## Why Indexing Is Important

Without indexing:
- queries become slower for large tables

With indexing:
- searching becomes faster
- query performance improves

---

## Common Use Cases

Indexes are commonly added on:
- user email
- username
- product ID
- order ID

Example:
```sql
CREATE INDEX idx_email
ON users(email);
```

---

## Advantages

- Faster data retrieval
- Improved search performance
- Better query optimization

---

## Disadvantages

- Extra storage is required
- Insert and update operations can become slightly slower

---

## My Learning

While working with backend projects, I understood that indexing becomes important when handling large amounts of data and frequent search operations.