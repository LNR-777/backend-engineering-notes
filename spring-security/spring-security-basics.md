# Spring Security Basics

Spring Security is a framework that handles authentication (who are you?) and authorization (what can you do?) for Spring Boot applications. It's the standard way to secure a Spring Boot app and it shows up in almost every real project.

The first time you add it to a project it can feel overwhelming — filters, security chains, authentication managers. This note breaks it down to what actually matters.

---

## What Spring Security does under the hood

Every HTTP request in a Spring Boot app goes through a chain of filters before it reaches your controller. Spring Security adds its own filters to this chain.

```
HTTP Request
    → Filter 1 (logging, CORS, etc.)
    → UsernamePasswordAuthenticationFilter  ← Spring Security
    → BasicAuthenticationFilter             ← Spring Security
    → SecurityContextPersistenceFilter      ← Spring Security
    → Your Controller
```

The most important filter for JWT-based apps is `OncePerRequestFilter` — you extend this to write your own filter that reads the JWT from the request header, validates it, and sets the authentication in the `SecurityContext`.

---

## Adding Spring Security to your project

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

The moment you add this, every endpoint becomes protected. Spring generates a default password and prints it in the console. You'll use `user` as username and the printed password to access anything.

That default behavior is for development only. You replace it with your own configuration.

---

## SecurityFilterChain — the main configuration

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

Breaking this down:

- `.csrf().disable()` — disable CSRF protection. For stateless REST APIs this is fine because you're not using cookies for auth (you're using JWT in headers).
- `STATELESS` — tells Spring not to create or use HTTP sessions. Every request must authenticate itself.
- `.permitAll()` — public endpoints. Login and register don't need auth.
- `.hasRole("ADMIN")` — only users with ADMIN role can access these.
- `.anyRequest().authenticated()` — everything else requires authentication.
- `.addFilterBefore(jwtFilter, ...)` — plug in your custom JWT filter before Spring's default auth filter.

---

## UserDetailsService — loading user from DB

Spring Security needs to know how to load a user. You implement `UserDetailsService` for this:

```java
@Service
public class UserDetailsServiceImpl implements UserDetailsService {

    @Autowired
    private UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByEmail(username)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));

        return org.springframework.security.core.userdetails.User.builder()
            .username(user.getEmail())
            .password(user.getPassword())
            .roles(user.getRole())
            .build();
    }
}
```

This is called during login. Spring Security calls `loadUserByUsername`, gets the `UserDetails`, checks the password, and if everything matches, creates an `Authentication` object.

---

## PasswordEncoder — never store plain passwords

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

When registering a user:

```java
user.setPassword(passwordEncoder.encode(request.getPassword()));
userRepository.save(user);
```

When authenticating, Spring Security calls `passwordEncoder.matches(rawPassword, encodedPassword)` automatically. You don't do the comparison yourself.

BCrypt adds a random salt each time so even the same password hashes differently. The hash includes the salt so Spring can still verify it.

---

## AuthenticationManager — used in login endpoint

```java
@Bean
public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
    return config.getAuthenticationManager();
}
```

In your auth controller:

```java
@PostMapping("/login")
public ResponseEntity<String> login(@RequestBody LoginRequest request) {
    Authentication auth = authenticationManager.authenticate(
        new UsernamePasswordAuthenticationToken(request.getEmail(), request.getPassword())
    );
    String token = jwtUtil.generateToken((User) auth.getPrincipal());
    return ResponseEntity.ok(token);
}
```

`authenticationManager.authenticate()` internally calls `loadUserByUsername`, checks the password using the `PasswordEncoder`, and either succeeds or throws `BadCredentialsException`.

---

## SecurityContext — how Spring knows who's logged in

Once you validate a JWT in your filter, you set the authentication in the `SecurityContext`:

```java
UsernamePasswordAuthenticationToken authToken =
    new UsernamePasswordAuthenticationToken(userId, null, List.of(new SimpleGrantedAuthority("ROLE_" + role)));

SecurityContextHolder.getContext().setAuthentication(authToken);
```

After this, any code in the same request can get the current user:

```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String userId = (String) auth.getPrincipal();
```

Because the session is STATELESS, the `SecurityContext` is cleared after every request. The next request has to authenticate again through your JWT filter.

---

## Stuff I want to remember

**What does Spring Security do when you first add it?**

It locks down every endpoint automatically and generates a random password. Until you configure it yourself, nothing is accessible without that default credential. This is intentional — secure by default.

**Why disable CSRF for REST APIs?**

CSRF attacks work by tricking a browser into making a request using an existing session cookie. Since REST APIs use token-based auth (JWT in headers), not cookies, CSRF isn't a threat. Disabling it simplifies things and doesn't reduce security.

**What's the difference between authentication and authorization in Spring Security?**

Authentication is verifying identity — "who are you?" — done via `UserDetailsService` and `PasswordEncoder`. Authorization is checking permissions — "what can you do?" — done via `.hasRole()`, `.hasAuthority()` in the security config. Both happen in the filter chain on every request.

**What is SecurityContextHolder?**

It's a static container that holds the `SecurityContext` for the current thread. After you validate a JWT and set the `Authentication`, anything in the same request can call `SecurityContextHolder.getContext().getAuthentication()` to get the current user. It's thread-local, so each request has its own context.

**Why do you need `@EnableWebSecurity`?**

It enables Spring Security's web security support and allows you to define a custom `SecurityFilterChain`. Without it, Spring Boot's auto-configuration handles security — which is fine for defaults but you lose control over the filter chain.

---

*Spring Security's default "lock everything" behavior caught me off guard the first time I added the dependency. My entire API went dark — every endpoint returned 401. Once I understood the filter chain and SecurityFilterChain configuration, it made a lot more sense.*