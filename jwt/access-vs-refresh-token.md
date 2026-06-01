# Access Token vs Refresh Token

In the JWT basics note we touched on this briefly. Let's go deeper because this comes up a lot in interviews and it's something you'll implement in real projects.

---

## Why two tokens

If you use one JWT with a long expiry — say 30 days — and it gets stolen, the attacker has 30 days of access. You can't invalidate it because JWT is stateless.

If you use one JWT with a short expiry — say 15 minutes — users have to login again every 15 minutes. Terrible experience.

Two tokens solves both problems.

---

## Access Token

Short lived — usually 15 minutes to 1 hour.

Sent with every API request in the Authorization header. Used to access protected endpoints.

```http
GET /api/orders
Authorization: Bearer <access_token>
```

Because it expires quickly, even if stolen the damage window is small.

---

## Refresh Token

Long lived — days or weeks.

Stored securely on the client. Only used for one thing — getting a new access token when the old one expires. Never sent to regular API endpoints.

```http
POST /auth/refresh
{
  "refreshToken": "<refresh_token>"
}
```

Server validates the refresh token and issues a new access token.

---

## The full flow

```
1. User logs in with email + password

2. Server returns both tokens:
   - access token (expires in 1 hour)
   - refresh token (expires in 7 days)

3. Client stores both — access token in memory, refresh token in httpOnly cookie

4. Client uses access token for every API call

5. Access token expires

6. Client sends refresh token to /auth/refresh

7. Server validates refresh token, issues new access token

8. Client continues with new access token

9. If refresh token also expires — user has to login again
```

---

## Implementation in Spring Boot

```java
@PostMapping("/login")
public ResponseEntity<AuthResponse> login(@RequestBody LoginRequest request) {
    // validate credentials
    authenticationManager.authenticate(
        new UsernamePasswordAuthenticationToken(request.getEmail(), request.getPassword())
    );

    User user = userRepository.findByEmail(request.getEmail())
        .orElseThrow(() -> new ResourceNotFoundException("User not found"));

    String accessToken = jwtUtil.generateAccessToken(user.getEmail(), user.getRole());
    String refreshToken = jwtUtil.generateRefreshToken(user.getEmail());

    return ResponseEntity.ok(new AuthResponse(accessToken, refreshToken));
}

@PostMapping("/refresh")
public ResponseEntity<String> refresh(@RequestBody RefreshRequest request) {
    String refreshToken = request.getRefreshToken();

    if (!jwtUtil.isTokenValid(refreshToken)) {
        throw new UnauthorizedException("Invalid or expired refresh token");
    }

    String email = jwtUtil.extractEmail(refreshToken);
    String newAccessToken = jwtUtil.generateAccessToken(email, "USER");

    return ResponseEntity.ok(newAccessToken);
}
```

In JwtUtil — two separate generation methods with different expiry:

```java
public String generateAccessToken(String email, String role) {
    return Jwts.builder()
        .setSubject(email)
        .claim("role", role)
        .setIssuedAt(new Date())
        .setExpiration(new Date(System.currentTimeMillis() + 1000 * 60 * 60)) // 1 hour
        .signWith(Keys.hmacShaKeyFor(SECRET_KEY.getBytes()), SignatureAlgorithm.HS256)
        .compact();
}

public String generateRefreshToken(String email) {
    return Jwts.builder()
        .setSubject(email)
        .setIssuedAt(new Date())
        .setExpiration(new Date(System.currentTimeMillis() + 1000 * 60 * 60 * 24 * 7)) // 7 days
        .signWith(Keys.hmacShaKeyFor(SECRET_KEY.getBytes()), SignatureAlgorithm.HS256)
        .compact();
}
```

---

## Where to store tokens on the client

This is a common interview question too.

**Access token** — in memory (JavaScript variable). Gone when tab closes. Not vulnerable to XSS if stored in memory.

**Refresh token** — in httpOnly cookie. JavaScript can't access it so XSS attacks can't steal it. Server sets it automatically on login response.

```
// setting refresh token as httpOnly cookie in Spring Boot
ResponseCookie cookie = ResponseCookie.from("refreshToken", refreshToken)
    .httpOnly(true)
    .secure(true)
    .path("/auth/refresh")
    .maxAge(Duration.ofDays(7))
    .build();

response.addHeader(HttpHeaders.SET_COOKIE, cookie.toString());
```

Never store tokens in localStorage — vulnerable to XSS.

---

## Stuff I want to remember

**Why use two tokens instead of one long lived token?**

One long lived token is a security risk — if stolen, attacker has access until it expires and you can't revoke it. Two tokens balances security and UX — short access token limits damage window, long refresh token means users don't have to keep logging in.

**What happens if the refresh token is stolen?**

This is a known limitation of stateless JWT. One mitigation is refresh token rotation — every time the refresh token is used to get a new access token, the old refresh token is invalidated and a new one is issued. If a stolen refresh token is used, the legitimate user's next refresh will fail, alerting them to re-login.

**Why store the refresh token in httpOnly cookie and not localStorage?**

httpOnly cookies can't be accessed by JavaScript — so even if there's an XSS vulnerability in your frontend, the attacker's script can't read the cookie. localStorage is accessible via JavaScript so it's vulnerable to XSS attacks.

---

*The httpOnly cookie part took me a while to understand. Once I realized the whole point is "JavaScript shouldn't be able to touch the refresh token" it made sense why httpOnly is the right choice.*