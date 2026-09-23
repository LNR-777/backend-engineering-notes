# SQL Constraints

Constraints are rules applied to columns or tables that enforce data integrity at the database level. Even if your application code has validation, constraints are the last line of defense — the database itself rejects data that violates them.

---

## NOT NULL

Column must have a value — null is not allowed.

```sql
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) NOT NULL
);
```

```sql
-- this will fail
INSERT INTO users (id, name) VALUES (1, 'Rohit');
-- ERROR: Column 'email' cannot be null
```

---

## UNIQUE

All values in the column must be different. Unlike PRIMARY KEY, a table can have multiple UNIQUE constraints and they allow NULL (one NULL per column in most databases).

```sql
CREATE TABLE users (
    id INT PRIMARY KEY,
    email VARCHAR(100) UNIQUE,
    phone VARCHAR(15) UNIQUE
);
```

```sql
-- this will fail if email already exists
INSERT INTO users (id, email) VALUES (2, 'rohit@gmail.com');
-- ERROR: Duplicate entry 'rohit@gmail.com' for key 'email'
```

In Spring Boot — this is what causes `DataIntegrityViolationException` when you try to save a duplicate email.

---

## PRIMARY KEY

Uniquely identifies each row. Combination of NOT NULL + UNIQUE. A table can have only one primary key.

```sql
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100)
);
```

Composite primary key — multiple columns together form the unique identifier:

```sql
CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);
```

---

## FOREIGN KEY

Links a column to the primary key of another table. Enforces referential integrity — you can't have an order for a user that doesn't exist.

```sql
CREATE TABLE orders (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT,
    amount DECIMAL(10,2),
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

```sql
-- this will fail if user with id 999 doesn't exist
INSERT INTO orders (user_id, amount) VALUES (999, 5000);
-- ERROR: Cannot add or update a child row: a foreign key constraint fails
```

---

## ON DELETE and ON UPDATE behavior

What happens to child rows when the parent is deleted or updated:

```sql
FOREIGN KEY (user_id) REFERENCES users(id)
    ON DELETE CASCADE    -- delete user → delete all their orders too
    ON UPDATE CASCADE    -- update user id → update it in orders too
```

```sql
FOREIGN KEY (user_id) REFERENCES users(id)
    ON DELETE SET NULL   -- delete user → set user_id to NULL in orders
```

```sql
FOREIGN KEY (user_id) REFERENCES users(id)
    ON DELETE RESTRICT   -- can't delete user if they have orders (default)
```

`CASCADE` is useful but be careful — one delete can trigger a chain of deletes across multiple tables.

---

## CHECK

Validates that values meet a specific condition.

```sql
CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10,2) CHECK (price > 0),
    stock INT CHECK (stock >= 0)
);
```

```sql
-- this will fail
INSERT INTO products (id, name, price) VALUES (1, 'Laptop', -5000);
-- ERROR: Check constraint 'products_chk_1' is violated
```

MySQL supports CHECK constraints from version 8.0.16+.

---

## DEFAULT

Sets a default value when none is provided.

```sql
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100),
    status VARCHAR(20) DEFAULT 'ACTIVE',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

```sql
-- status and created_at filled automatically
INSERT INTO users (name) VALUES ('Rohit');
```

---

## Adding constraints to existing tables

```sql
-- add NOT NULL
ALTER TABLE users MODIFY email VARCHAR(100) NOT NULL;

-- add UNIQUE
ALTER TABLE users ADD CONSTRAINT unique_email UNIQUE (email);

-- add FOREIGN KEY
ALTER TABLE orders ADD CONSTRAINT fk_user
    FOREIGN KEY (user_id) REFERENCES users(id);

-- add CHECK
ALTER TABLE products ADD CONSTRAINT chk_price CHECK (price > 0);

-- drop a constraint
ALTER TABLE users DROP CONSTRAINT unique_email;
```

---

## Constraints in JPA/Hibernate

You can define these at the entity level too — Hibernate generates the DDL with constraints when `ddl-auto=create` or `update`:

```java
@Entity
@Table(name = "users", uniqueConstraints = {
    @UniqueConstraint(columnNames = "email")
})
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false, unique = true)
    private String email;

    @Column(nullable = false)
    @Enumerated(EnumType.STRING)
    private UserStatus status = UserStatus.ACTIVE;
}
```

---

## Stuff I want to remember

**What is the difference between UNIQUE and PRIMARY KEY?**

Both enforce uniqueness but PRIMARY KEY also enforces NOT NULL and a table can only have one. UNIQUE allows NULL values and a table can have multiple UNIQUE constraints. PRIMARY KEY is the main identifier of a row, UNIQUE is for alternate unique fields like email.

**What does ON DELETE CASCADE do?**

When a parent record is deleted, all related child records are automatically deleted too. If you delete a user with ON DELETE CASCADE on the orders foreign key, all their orders get deleted automatically. Useful but dangerous — one delete can wipe a lot of data across tables.

**Why define constraints at the database level when you already validate in Spring Boot?**

Application validation can be bypassed — direct DB inserts, scripts, other services accessing the same DB. Database constraints are enforced regardless of how data enters the database. They're the final safety net that ensures data integrity even when application code fails or is bypassed.

---

*The foreign key constraint error used to confuse me early on — trying to insert a record with a reference to a non-existent parent. Once I understood referential integrity it made complete sense why the DB rejects it.*