# REST API Security Basics

Building an API that works is one thing. Building one that's secure is another. These are the fundamentals every backend developer should know.

---

## Authentication vs Authorization

These two come up together but mean different things.

**Authentication** — who are you? Verifying identity. Login with email/password, validating a JWT token.

**Authorization** — what are you allowed to do? Checking permissions after identity is confirmed. An authenticated user trying to access an admin endpoint — authorization decides if that's allowed.

```
Request comes in
  → Authentication: is this a valid token? who is this user?
    → Authorization: does this user have permission to access this endpoint?
      → Process request
```

---

## Always use HTTPS

Never run a production API over plain HTTP. Tokens, passwords, and user data sent over HTTP can be intercepted.

HTTPS encrypts the entire request and response — headers, body, everything. JWT in an Authorization header over HTTP is visible to anyone on the network. Same JWT over HTTPS — encrypted in transit.

---

## Input Validation

Never trust client input. Validate everything before it reaches your business logic.

```java
public class UserRequest {

    @NotBlank(message = "Name is required")
    private String name;

    @Email(message = "Invalid email")
    private String email;

    @Size(min = 8, message = "Password must be at least 8 characters")
    private String password;
}
```

Without validation — someone can send malformed data, extremely long strings, or crafted input to break your application.

---

## SQL Injection — why you should never build queries with string concatenation

```
// dangerous — never do this
String query = "SELECT * FROM users WHERE email = '" + email + "'";

// if someone sends email as: ' OR '1'='1
// the query becomes: SELECT * FROM users WHERE email = '' OR '1'='1'
// returns all users
```

Use parameterized queries or Spring Data JPA — they handle this automatically:

```java
// safe — Spring Data JPA
Optional<User> findByEmail(String email);

// safe — parameterized query
@Query("SELECT u FROM User u WHERE u.email = :email")
Optional<User> findByEmail(@Param("email") String email);
```

JPA uses prepared statements under the hood — user input is never directly embedded in the SQL.

---

## Password Storage

Never store plain text passwords. Always hash them.

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}

// when saving a user
user.setPassword(passwordEncoder.encode(rawPassword));

// when verifying login
boolean matches = passwordEncoder.matches(rawPassword, storedHash);
```

BCrypt is slow by design — makes brute force attacks expensive. Even if your database is compromised, attackers can't easily recover the original passwords.

---

## Rate Limiting

Prevent abuse by limiting how many requests a client can make in a given time window.

Without rate limiting — someone can hammer your API with thousands of requests per second (brute force, DoS).

In Spring Boot you can add rate limiting with a library like Bucket4j:

```java
@GetMapping("/api/login")
public ResponseEntity<?> login(@RequestBody LoginRequest request) {
    // check rate limit before processing
    if (!rateLimiter.tryConsume()) {
        return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS)
            .body("Too many requests. Try again later.");
    }
    // process login
}
```

Status code for rate limit exceeded is `429 Too Many Requests`.

---

## Sensitive Data in Responses

Don't return fields that shouldn't be exposed.

```
// bad — returning the full entity
return ResponseEntity.ok(userRepository.findById(id));

// user entity has: id, name, email, password, internalFlag, createdAt...
// client doesn't need password or internalFlag
```

Use DTOs (Data Transfer Objects) — return only what the client needs:

```java
public class UserResponse {
    private Long id;
    private String name;
    private String email;
    // no password, no internal fields
}

return ResponseEntity.ok(new UserResponse(user));
```

---

## CORS — Cross Origin Resource Sharing

When your frontend (running on `localhost:3000`) calls your backend (running on `localhost:8080`), the browser blocks it by default — different origin. CORS headers tell the browser which origins are allowed.

```java
@CrossOrigin(origins = "http://localhost:3000")
@RestController
public class UserController { }
```

Or globally in SecurityConfig:

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("http://localhost:3000", "https://myapp.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH"));
    config.setAllowedHeaders(List.of("*"));
    
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}
```

In production never set `allowedOrigins("*")` — always specify the exact frontend URL.

---

## Stuff I want to remember

**What is the difference between authentication and authorization?**

Authentication is verifying who you are — validating your token or credentials. Authorization is checking what you're allowed to do — even after you're authenticated, you might not have permission to access a specific resource. Authentication happens first, authorization comes after.

**Why is BCrypt preferred for password hashing?**

BCrypt is intentionally slow and includes a salt automatically — making both brute force attacks and rainbow table attacks expensive. Even if two users have the same password, their BCrypt hashes will be different. MD5 and SHA-1 are too fast — attackers can hash billions of combinations per second against them.

**What is SQL injection and how does JPA prevent it?**

SQL injection is when an attacker sends malicious input that gets embedded directly into a SQL query and changes its logic. JPA uses prepared statements — the query structure is fixed and user input is passed as parameters, never embedded as raw SQL. The database treats input as data, not as SQL code.

---

*Security was something I started taking more seriously after reading about real data breaches. A lot of them happen because of basic mistakes — plain text passwords, no input validation, SQL built with string concatenation. These are all preventable.*