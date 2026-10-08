# Database Connection Pooling

Every time your application talks to the database, it needs a connection. Opening a connection is expensive — there's a TCP handshake, authentication, memory allocation on both sides. If you open a new connection for every request and close it after, your app becomes slow under any real load.

Connection pooling fixes this. Instead of creating and destroying connections constantly, you keep a pool of pre-opened connections ready to use. When a request comes in, it borrows a connection from the pool, uses it, and returns it. The connection stays open.

---

## How it works

```
App starts → Pool creates N connections to DB (say 10)

Request 1 comes in → borrows connection from pool → runs query → returns connection
Request 2 comes in → borrows another connection → runs query → returns it

If all 10 are in use and request 11 comes in → it waits until one is returned
```

The pool manages all of this. Your application code doesn't open or close connections manually — it just calls `dataSource.getConnection()` and the pool handles the rest.

---

## HikariCP — what Spring Boot uses by default

Spring Boot's default connection pool is HikariCP. It's fast, lightweight, and well-configured out of the box. You usually don't have to do anything to enable it — just add your DB driver and Spring Boot picks up HikariCP automatically.

```java
// You don't write this — Spring Boot creates this automatically
// from your application.properties
HikariDataSource dataSource = new HikariDataSource(config);
```

---

## Configuration in application.properties

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=root
spring.datasource.password=secret
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# HikariCP settings
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
```

What each setting does:

- `maximum-pool-size` — max connections in the pool. Default is 10. Don't blindly increase this — more isn't always better.
- `minimum-idle` — minimum connections to keep open even when idle.
- `connection-timeout` — how long a request waits for a connection before throwing an exception (30 seconds here).
- `idle-timeout` — how long an idle connection stays in the pool before being closed (10 minutes).
- `max-lifetime` — maximum lifetime of a connection in the pool (30 minutes). Prevents using stale connections.

---

## What happens when the pool runs out

If all connections are in use and a new request tries to borrow one, it waits for `connection-timeout` milliseconds. If no connection becomes available in that time, it throws:

```
com.zaxxer.hikari.pool.HikariPool$PoolInitializationException
```

or at runtime:

```
SQLTransientConnectionException: HikariPool-1 - Connection is not available, request timed out after 30000ms
```

This usually means one of two things — either `maximum-pool-size` is too low for your load, or you have a connection leak (a connection borrowed but never returned because of a missing try-with-resources or unclosed statement).

---

## Connection leak — a common mistake

```java
// This leaks a connection if an exception happens before close()
Connection conn = dataSource.getConnection();
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery("SELECT * FROM users");
// If something throws here, conn never gets closed
conn.close();
```

Use try-with-resources:

```java
try (Connection conn = dataSource.getConnection();
     Statement stmt = conn.createStatement();
     ResultSet rs = stmt.executeQuery("SELECT * FROM users")) {
    // conn is automatically returned to the pool when block exits
}
```

In Spring Boot with JPA/Hibernate, you don't manage connections directly — Spring handles it. But if you ever use `JdbcTemplate` or raw `DataSource`, always use try-with-resources.

---

## Checking pool size — a rule of thumb

There's a common formula from the HikariCP docs:

```
pool size = (number of CPU cores * 2) + number of effective spindle disks
```

For most apps on a 4-core machine with a single SSD: roughly 8-10 connections. The default of 10 is fine for most small to medium applications.

The real way to tune it is to test under load and watch your metrics — connection wait time, pool utilization, query time.

---

## Stuff I want to remember

**What is connection pooling and why does it matter?**

Creating a database connection every time a request comes in is slow — it involves network setup and authentication. Connection pooling keeps a set of connections pre-opened and reuses them. A request borrows one, does its work, and returns it. This dramatically reduces latency and lets the app handle much higher load.

**What does HikariCP do that's different from other pools?**

HikariCP is optimized for low overhead. It uses a lock-free design internally, minimal object allocation, and very fast connection acquisition. It's the default in Spring Boot for good reason — you basically get it for free and it performs well without much configuration.

**What does `maximum-pool-size` mean and how do you pick a number?**

It's the max number of concurrent DB connections your app will ever open. Setting it too high doesn't help — your DB has its own connection limit and too many connections actually hurt performance. The HikariCP rule of thumb is `(cores * 2) + spindles`. For most apps, the default 10 is a reasonable starting point.

**What causes a connection timeout error?**

Either the pool is exhausted (all connections in use) or there's a connection leak — a connection was borrowed and never returned. The timeout kicks in after `connection-timeout` milliseconds of waiting. Check your code for places where a connection might be acquired but not properly closed.

---

*Ran into the timeout error during my internship project when too many concurrent requests were hitting a small pool. Bumping `maximum-pool-size` to 20 helped, but the real fix was finding a connection that wasn't being returned in an error path.*