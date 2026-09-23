# Stored Procedures

A stored procedure is a block of SQL code saved in the database that you can call by name. Think of it like a method in Java — define it once, call it whenever you need it, pass parameters in and get results out.

---

## Basic syntax in MySQL

```sql
DELIMITER //

CREATE PROCEDURE GetUserById(IN userId INT)
BEGIN
    SELECT id, name, email
    FROM users
    WHERE id = userId;
END //

DELIMITER ;
```

`DELIMITER //` changes the statement terminator temporarily so MySQL doesn't interpret the semicolons inside the procedure as end of statement.

Call it:

```sql
CALL GetUserById(42);
```

---

## IN, OUT, INOUT parameters

```sql
DELIMITER //

CREATE PROCEDURE GetOrderCount(
    IN userId INT,
    OUT orderCount INT
)
BEGIN
    SELECT COUNT(*) INTO orderCount
    FROM orders
    WHERE user_id = userId;
END //

DELIMITER ;
```

```sql
-- call it
CALL GetOrderCount(42, @count);

-- read the output
SELECT @count;
```

- `IN` — input parameter, passed into the procedure
- `OUT` — output parameter, procedure sets a value you can read after
- `INOUT` — both input and output

---

## Procedure with logic

```sql
DELIMITER //

CREATE PROCEDURE TransferMoney(
    IN fromAccount INT,
    IN toAccount INT,
    IN amount DECIMAL(10,2)
)
BEGIN
    DECLARE currentBalance DECIMAL(10,2);
    
    -- check balance
    SELECT balance INTO currentBalance
    FROM accounts WHERE id = fromAccount;
    
    IF currentBalance < amount THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Insufficient funds';
    ELSE
        UPDATE accounts SET balance = balance - amount WHERE id = fromAccount;
        UPDATE accounts SET balance = balance + amount WHERE id = toAccount;
    END IF;
END //

DELIMITER ;
```

```sql
CALL TransferMoney(1, 2, 5000.00);
```

---

## Dropping a procedure

```sql
DROP PROCEDURE IF EXISTS GetUserById;
```

---

## Viewing existing procedures

```sql
SHOW PROCEDURE STATUS WHERE Db = 'your_database_name';
```

---

## Stored procedures vs application code

This is something worth thinking about — not everything should be in a stored procedure.

**Stored procedures make sense when:**
- Complex multi-step DB operations that run frequently
- Logic that needs to be shared across multiple applications using the same DB
- Operations where minimizing network trips matters (procedure runs entirely on DB server)
- Database-level security — expose only procedures, not direct table access

**Application code is better when:**
- Business logic that changes often — stored procedures are harder to version control and deploy
- You want to unit test the logic easily
- The team is more comfortable with Java than SQL
- You're using an ORM like JPA/Hibernate — mixing stored procedures with JPA adds complexity

In most modern Spring Boot projects the trend is to keep logic in the service layer and use JPA for DB operations. Stored procedures are still used but mainly for performance-critical batch operations or legacy systems.

---

## Calling a stored procedure from Spring Boot

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    @Procedure(procedureName = "GetUserById")
    User getUserById(Integer userId);
}
```

Or with `EntityManager`:

```java
StoredProcedureQuery query = entityManager
    .createStoredProcedureQuery("GetOrderCount")
    .registerStoredProcedureParameter("userId", Integer.class, ParameterMode.IN)
    .registerStoredProcedureParameter("orderCount", Integer.class, ParameterMode.OUT)
    .setParameter("userId", 42);

query.execute();
Integer count = (Integer) query.getOutputParameterValue("orderCount");
```

---

## Stuff I want to remember

**What is a stored procedure?**

A reusable block of SQL saved in the database with a name. You call it like a function — pass parameters in, get results back. Runs entirely on the database server. Useful for complex multi-step operations or operations shared across multiple apps.

**What is the difference between a stored procedure and a function in SQL?**

A function must return a value and can be used inside a SELECT statement. A stored procedure may or may not return values through OUT parameters and is called with CALL. Functions can't have side effects (no INSERT/UPDATE/DELETE in most databases), procedures can.

**Why do most Spring Boot projects avoid stored procedures?**

Because business logic in stored procedures is harder to version control, test, and deploy compared to Java code. When you update a stored procedure you're directly modifying the database — no compile step, no unit tests, harder to roll back. JPA/Spring Data keeps the logic in the application where it's easier to manage.

---

*Used stored procedures at my internship for some reporting queries — the logic was too complex for JPA and the DB team had already built procedures for it. Calling them from Spring Boot with EntityManager was straightforward once I understood the parameter modes.*