# Transactions and ACID Properties

A transaction is a group of database operations that execute as a single unit — either all of them succeed, or none of them do. ACID is the set of properties that guarantee transactions behave reliably.

---

## Why transactions matter

Classic example — bank transfer. Move 1000 from account A to account B.

```sql
UPDATE accounts SET balance = balance - 1000 WHERE id = 1; -- debit A
UPDATE accounts SET balance = balance + 1000 WHERE id = 2; -- credit B
```

If the server crashes after the first statement but before the second — A lost 1000, B never received it. Money disappeared. Transactions prevent this.

```sql
START TRANSACTION;

UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
UPDATE accounts SET balance = balance + 1000 WHERE id = 2;

COMMIT;
```

If anything fails between START TRANSACTION and COMMIT, you can ROLLBACK and nothing changes.

```sql
START TRANSACTION;

UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
-- something goes wrong here
ROLLBACK; -- undoes the above update, balance unchanged
```

---

## ACID — the four properties

### Atomicity

All operations in a transaction succeed, or all fail. No partial execution.

In the bank transfer example — both updates happen, or neither does. Never just one.

---

### Consistency

The database moves from one valid state to another valid state. Any constraints (foreign keys, unique constraints, checks) must hold true before and after the transaction.

If a constraint says balance can't go negative, and a transaction would make it negative, the transaction fails and rolls back — database stays consistent.

---

### Isolation

Concurrent transactions don't interfere with each other. Even if 100 transactions run at the same time, the end result should be as if they ran one after another.

```sql
-- Transaction 1
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1; -- reads 5000
UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
COMMIT;

-- Transaction 2 running at the same time
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1; -- should this see 5000 or 4000?
```

This is where isolation levels come in — they control how much transactions can see each other's uncommitted changes. More on this below.

---

### Durability

Once a transaction is committed, it stays committed — even if the server crashes immediately after. The database writes to disk, not just memory.

---

## Isolation Levels

This is the part that comes up most in interviews. Four standard levels, from least to most strict:

### Read Uncommitted
Transactions can see uncommitted changes from other transactions. Causes "dirty reads" — reading data that might get rolled back.

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
```

Rarely used — too risky.

---

### Read Committed
Transactions only see committed changes from others. No dirty reads. But the same query run twice in one transaction might return different results if another transaction committed in between — "non-repeatable read".

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

This is the default in many databases like PostgreSQL and SQL Server.

---

### Repeatable Read
Once you read a row in a transaction, it stays the same for the rest of that transaction — even if another transaction commits changes to it. Prevents non-repeatable reads. But "phantom reads" can still happen — new rows matching your query condition can appear.

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

This is MySQL's default (InnoDB).

---

### Serializable
Strictest level. Transactions run as if they happened one after another, completely isolated. No dirty reads, no non-repeatable reads, no phantom reads. But significantly slower due to locking.

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

Used rarely — only when absolute consistency is critical and performance is less of a concern.

---

## Quick comparison

| Level | Dirty Read | Non-repeatable Read | Phantom Read |
|---|---|---|---|
| Read Uncommitted | Possible | Possible | Possible |
| Read Committed | No | Possible | Possible |
| Repeatable Read | No | No | Possible |
| Serializable | No | No | No |

---

## Stuff I want to remember

**What are the ACID properties?**

Atomicity — all or nothing. Consistency — valid state to valid state. Isolation — concurrent transactions don't interfere with each other. Durability — once committed, stays committed even after a crash. Classic example to explain it is a bank transfer — both the debit and credit must happen together, or neither happens.

**What is the default isolation level in MySQL?**

Repeatable Read for InnoDB. It prevents dirty reads and non-repeatable reads but phantom reads can still occur.

**What is a dirty read?**

When a transaction reads data that another transaction has changed but not yet committed. If that other transaction rolls back, you've read data that never actually existed in the database. Read Uncommitted allows this, all higher isolation levels prevent it.

---

*ACID made more sense once I thought about what could go wrong without each property — atomicity prevents partial updates, isolation prevents transactions from seeing each other's half-finished work, durability prevents losing committed data on crash.*