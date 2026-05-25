# Implementing JWT Authentication in Spring Boot

In the last note we covered what JWT is and how it works conceptually. This note is about actually wiring it up in a Spring Boot project with Spring Security.

---

## Dependencies

```xml
<dependencies>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>

    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.11.5</version>
    </dependency>

    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.11.5</version>
    </dependency>

    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>0.11.5</version>
    </dependency>

</dependencies>
```

---

## The flow we're building

```
POST /auth/login  →  validate credentials  →  generate JWT  →  return token
GET /api/users    →  read token from header  →  validate  →  process request
```

Four main pieces — JwtUtil, JwtFilter, SecurityConfig, AuthController.

---

## 1. JwtUtil — generate and validate tokens

```java
@Component
public class JwtUtil {

    private final String SECRET_KEY = "your-256-bit-secret-key-here";
    private final long EXPIRATION = 1000 * 60 * 60; // 1 hour

    public String generateToken(String email, String role) {
        return Jwts.builder()
            .setSubject(email)
            .claim("role", role)
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + EXPIRATION))
            .signWith(Keys.hmacShaKeyFor(SECRET_KEY.getBytes()), SignatureAlgorithm.HS256)
            .compact();
    }

    public String extractEmail(String token) {
        return getClaims(token).getSubject();
    }

    public String extractRole(String token) {
        return getClaims(token).get("role", String.class);
    }

    public boolean isTokenValid(String token) {
        try {
            getClaims(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }

    private Claims getClaims(String token) {
        return Jwts.parserBuilder()
            .setSigningKey(Keys.hmacShaKeyFor(SECRET_KEY.getBytes()))
            .build()
            .parseClaimsJws(token)
            .getBody();
    }
}
```

Keep the secret key in `application.properties`, not hardcoded. Using `@Value` to inject it is cleaner.

---

## 2. JwtFilter — intercept every request

This filter runs before every request. It reads the token, validates it, and sets the authentication in Spring Security's context.

```java
@Component
public class JwtFilter extends OncePerRequestFilter {

    @Autowired
    private JwtUtil jwtUtil;

    @Autowired
    private UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        String authHeader = request.getHeader("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        String token = authHeader.substring(7); // remove "Bearer "

        if (jwtUtil.isTokenValid(token)) {
            String email = jwtUtil.extractEmail(token);
            UserDetails userDetails = userDetailsService.loadUserByUsername(email);

            UsernamePasswordAuthenticationToken authToken =
                new UsernamePasswordAuthenticationToken(
                    userDetails, null, userDetails.getAuthorities()
                );

            SecurityContextHolder.getContext().setAuthentication(authToken);
        }

        filterChain.doFilter(request, response);
    }
}
```

---

## 3. SecurityConfig — configure which endpoints are protected

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Autowired
    private JwtFilter jwtFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf().disable()
            .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
            .authorizeHttpRequests()
                .requestMatchers("/auth/**").permitAll()
                .anyRequest().authenticated()
            .and()
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config)
            throws Exception {
        return config.getAuthenticationManager();
    }
}
```

`SessionCreationPolicy.STATELESS` — tells Spring Security not to create or use HTTP sessions. JWT handles state.

---

## 4. AuthController — login endpoint

```java
@RestController
@RequestMapping("/auth")
public class AuthController {

    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    private JwtUtil jwtUtil;

    @Autowired
    private UserRepository userRepository;

    @PostMapping("/login")
    public ResponseEntity<String> login(@RequestBody LoginRequest request) {
        authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(
                request.getEmail(), request.getPassword()
            )
        );

        User user = userRepository.findByEmail(request.getEmail())
            .orElseThrow(() -> new ResourceNotFoundException("User not found"));

        String token = jwtUtil.generateToken(user.getEmail(), user.getRole());
        return ResponseEntity.ok(token);
    }
}
```

If `authenticationManager.authenticate()` fails — wrong email or password — it throws an exception automatically. No manual credential checking needed.

---

## How a protected request flows

```
Request hits /api/users
    → JwtFilter intercepts
        → reads Authorization header
            → validates token
                → sets authentication in SecurityContext
                    → Spring Security allows the request through
                        → Controller handles it
```

If token is missing or invalid — SecurityContext stays empty — Spring Security blocks the request with 401.

---

## Stuff I want to remember

**Why do we extend OncePerRequestFilter?**

It guarantees the filter runs exactly once per request. Some filters can run multiple times in a request lifecycle — `OncePerRequestFilter` prevents that.

**Why disable CSRF for JWT APIs?**

CSRF attacks work by tricking a browser into making requests using existing session cookies. Since JWT APIs are stateless and don't use cookies for auth, CSRF isn't a risk. So we disable it to avoid it interfering with our API calls.

**What is SecurityContextHolder?**

It's where Spring Security stores the authentication info for the current request. When our filter sets the authentication there, Spring Security knows the user is authenticated for the rest of that request's lifecycle.

---

*This took me a while to get right the first time — the filter chain order matters a lot. Once I understood that JwtFilter runs before the default authentication filter and just sets context, it all made sense.*