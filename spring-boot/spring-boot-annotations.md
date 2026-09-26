# Spring Boot Annotations

Annotations are everywhere in Spring Boot. Instead of writing XML configuration, you annotate your classes and methods and Spring handles the rest. This note covers the ones you'll use most and what they actually do.

---

## Core Stereotype Annotations

### @Component
Generic annotation to mark a class as a Spring bean. Spring picks it up during component scan and manages it.

```java
@Component
public class EmailValidator {
    public boolean isValid(String email) {
        return email.contains("@");
    }
}
```

### @Service
Same as @Component functionally — marks the business logic layer. Makes code easier to read and understand.

```java
@Service
public class UserService {
    // business logic here
}
```

### @Repository
Same as @Component but for the data access layer. Also adds persistence exception translation — converts DB-specific exceptions into Spring's `DataAccessException`.

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> { }
```

### @Controller / @RestController
`@Controller` is for MVC — returns view names. `@RestController` is `@Controller` + `@ResponseBody` — returns data directly as JSON/XML. For REST APIs always use `@RestController`.

```java
@RestController
@RequestMapping("/api/users")
public class UserController { }
```

---

## Request Mapping Annotations

```java
@GetMapping("/users")         // GET
@PostMapping("/users")        // POST
@PutMapping("/users/{id}")    // PUT
@PatchMapping("/users/{id}")  // PATCH
@DeleteMapping("/users/{id}") // DELETE
```

All of these are shortcuts for `@RequestMapping(method = RequestMethod.GET)` etc.

---

## Parameter Annotations

```java
@PathVariable   // from URL path: /users/{id}
@RequestParam   // from query string: /users?role=admin
@RequestBody    // from request body (JSON)
@RequestHeader  // from request headers
```

```java
@GetMapping("/{id}")
public ResponseEntity<User> getUser(
        @PathVariable Long id,
        @RequestParam(required = false) String fields,
        @RequestHeader("Authorization") String token) {
    return ResponseEntity.ok(userService.findById(id));
}
```

---

## Configuration Annotations

### @Configuration
Marks a class as a source of bean definitions. Methods inside annotated with `@Bean` produce beans managed by Spring.

```java
@Configuration
public class AppConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### @Bean
Declares a method's return value as a Spring bean. Used inside `@Configuration` classes.

### @Value
Injects values from `application.properties` into fields.

```java
@Value("${jwt.secret}")
private String jwtSecret;

@Value("${server.port:8080}") // with default value
private int serverPort;
```

### @ConfigurationProperties
Binds a group of related properties to a class — cleaner than multiple `@Value` annotations.

```java
@ConfigurationProperties(prefix = "jwt")
@Component
public class JwtProperties {
    private String secret;
    private long expiration;
    // getters and setters
}
```

```properties
jwt.secret=mySecretKey
jwt.expiration=3600000
```

---

## JPA Annotations

```java
@Entity          // marks class as a JPA entity (DB table)
@Table           // specify table name
@Id              // primary key
@GeneratedValue  // auto-generate primary key value
@Column          // customize column mapping
@OneToMany       // one-to-many relationship
@ManyToOne       // many-to-one relationship
@ManyToMany      // many-to-many relationship
@JoinColumn      // specify foreign key column
```

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false, unique = true)
    private String email;

    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    private List<Order> orders;
}
```

---

## Validation Annotations

```java
@NotNull      // not null
@NotBlank     // not null, not empty, not whitespace
@Email        // valid email format
@Min / @Max   // number range
@Size         // string or collection size
@Pattern      // regex match
```

---

## Spring Security Annotations

```java
@EnableWebSecurity       // enables Spring Security config
@PreAuthorize            // check permission before method runs
@PostAuthorize           // check permission after method runs
```

```java
@PreAuthorize("hasRole('ADMIN')")
@DeleteMapping("/{id}")
public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
    userService.delete(id);
    return ResponseEntity.noContent().build();
}
```

---

## Lifecycle Annotations

```java
@PostConstruct   // runs after bean is created and dependencies injected
@PreDestroy      // runs before bean is destroyed
```

---

## Scheduling

```java
@EnableScheduling   // enable scheduling in config class
@Scheduled          // run method on a schedule
```

```java
@Scheduled(cron = "0 0 * * * *") // runs every hour
public void cleanupExpiredTokens() {
    tokenService.removeExpired();
}
```

---

## Stuff I want to remember

**What is the difference between @Component, @Service and @Repository?**

All three register a class as a Spring bean — functionally the same. The difference is semantic and clarity. `@Service` signals business logic layer, `@Repository` signals data access layer and adds exception translation. Using the right annotation makes the codebase easier to navigate.

**What is the difference between @Controller and @RestController?**

`@RestController` is `@Controller` + `@ResponseBody`. Without `@ResponseBody`, Spring tries to resolve method return values as view names (for MVC). With `@ResponseBody`, the return value is serialized directly to JSON/XML and written to the response. For REST APIs always use `@RestController`.

**What does @ConfigurationProperties do differently from @Value?**

`@Value` injects one property at a time. `@ConfigurationProperties` binds a whole group of related properties to a class at once — cleaner when you have multiple related settings like JWT config or database connection pool settings. Also gives you type safety and IDE support.

---

*Took me a while to understand why there are three stereotype annotations (@Component, @Service, @Repository) when they do the same thing. The answer is readability and intent — the annotation tells you which layer the class belongs to.*