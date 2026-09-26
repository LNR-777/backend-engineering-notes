# JPA and Hibernate

JPA and Hibernate come up together so much that people often think they're the same thing. They're not — understanding the difference matters.

---

## JPA vs Hibernate

**JPA (Java Persistence API)** is a specification — a set of interfaces and rules for how Java objects should be mapped to database tables. It doesn't do anything by itself. It's like an interface in Java.

**Hibernate** is an implementation of JPA. It's the actual library that does the work — generating SQL, managing sessions, handling caching. There are other JPA implementations (EclipseLink, OpenJPA) but Hibernate is the most popular by far.

```
JPA        = the specification (interfaces, annotations)
Hibernate  = the implementation (does the actual work)
Spring Data JPA = sits on top of JPA, gives you repositories
```

When you use Spring Data JPA in Spring Boot, Hibernate is the default JPA provider underneath.

---

## Entity mapping basics

```java
@Entity
@Table(name = "products")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "product_name", nullable = false, length = 200)
    private String name;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price;

    @Column(name = "is_active")
    private Boolean active = true;

    @CreationTimestamp
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @UpdateTimestamp
    private LocalDateTime updatedAt;
}
```

`@CreationTimestamp` and `@UpdateTimestamp` are Hibernate specific — they auto-populate timestamps.

---

## Relationships

### One to Many / Many to One

```java
@Entity
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Order> orders = new ArrayList<>();
}

@Entity
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;
}
```

`mappedBy = "user"` — tells JPA that the `Order` side owns the relationship (has the foreign key). The `User` side is the inverse.

### Many to Many

```java
@Entity
public class Student {

    @ManyToMany
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private List<Course> courses = new ArrayList<>();
}
```

---

## FetchType — LAZY vs EAGER

This is one of the most important things to understand in JPA.

**LAZY** — related data is loaded only when you access it. Default for `@OneToMany` and `@ManyToMany`.

**EAGER** — related data is loaded immediately with the parent. Default for `@ManyToOne` and `@OneToOne`.

```java
@OneToMany(fetch = FetchType.LAZY)   // orders loaded only when accessed
@ManyToOne(fetch = FetchType.EAGER)  // user loaded immediately with order
```

Always prefer LAZY for collections. EAGER on a collection means every time you load the parent, all children are loaded too — even when you don't need them. That's the N+1 problem waiting to happen.

---

## N+1 Problem

One of the most common JPA performance issues.

Say you load 100 orders. For each order, JPA lazily loads the user. That's 1 query for orders + 100 queries for users = 101 queries. N+1.

Fix with JOIN FETCH:

```java
@Query("SELECT o FROM Order o JOIN FETCH o.user WHERE o.status = :status")
List<Order> findByStatusWithUser(@Param("status") String status);
```

Or with `@EntityGraph`:

```java
@EntityGraph(attributePaths = {"user"})
List<Order> findByStatus(String status);
```

Both load orders and users in a single query.

---

## CascadeType

Controls what happens to child entities when you perform operations on the parent.

```java
@OneToMany(cascade = CascadeType.ALL)
```

| Type | What it does |
|---|---|
| `PERSIST` | Save parent → also saves children |
| `MERGE` | Update parent → also updates children |
| `REMOVE` | Delete parent → also deletes children |
| `REFRESH` | Refresh parent → also refreshes children |
| `ALL` | All of the above |

Be careful with `CascadeType.REMOVE` — deleting a parent deletes all children. Use only when that's actually what you want.

---

## Spring Data JPA — Repository methods

Spring Data JPA generates queries from method names:

```java
public interface UserRepository extends JpaRepository<User, Long> {

    Optional<User> findByEmail(String email);
    List<User> findByStatus(String status);
    List<User> findByNameContaining(String keyword);
    List<User> findByAgeGreaterThan(int age);
    boolean existsByEmail(String email);
    long countByStatus(String status);
    void deleteByStatus(String status);
}
```

No SQL needed — Spring generates it from the method name.

For complex queries use `@Query`:

```java
@Query("SELECT u FROM User u WHERE u.email = :email AND u.status = :status")
Optional<User> findByEmailAndStatus(@Param("email") String email, @Param("status") String status);
```

---

## ddl-auto settings

```properties
spring.jpa.hibernate.ddl-auto=create        # drops and recreates schema on startup
spring.jpa.hibernate.ddl-auto=create-drop   # creates on startup, drops on shutdown
spring.jpa.hibernate.ddl-auto=update        # updates schema to match entities (doesn't drop)
spring.jpa.hibernate.ddl-auto=validate      # validates schema matches entities, no changes
spring.jpa.hibernate.ddl-auto=none          # does nothing
```

- `create` — only for initial dev, data wiped every restart
- `update` — okay for dev, risky for production (can't drop removed columns)
- `validate` — good for production
- `none` — use when you manage schema yourself (Flyway/Liquibase)

Never use `create` or `create-drop` in production.

---

## Stuff I want to remember

**What is the difference between JPA and Hibernate?**

JPA is a specification — just interfaces and annotations, no actual implementation. Hibernate is the most popular implementation of JPA. When you use Spring Data JPA, Hibernate is running underneath doing the actual work of generating SQL and managing database sessions.

**What is the N+1 problem in JPA?**

When you load N entities and then lazily load a related entity for each one — resulting in N+1 database queries. Loading 100 orders and their users without JOIN FETCH means 1 query for orders and 100 queries for users. Fix it with JOIN FETCH or @EntityGraph to load everything in a single query.

**What is the difference between LAZY and EAGER fetching?**

LAZY loads related data only when you access it. EAGER loads it immediately with the parent. Always use LAZY for collections — EAGER on a collection loads all children every time you load the parent, even when you don't need them. That's a performance issue on large datasets.

---

*The N+1 problem caught me off guard the first time — everything looked fine with small test data but slowed down badly with real data. Running the queries with `spring.jpa.show-sql=true` and seeing 100+ queries fire for a simple listing was what made it click.*