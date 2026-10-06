# JWT Claims and Payload

We covered JWT structure in the basics note. This note goes deeper into the payload — what claims are, the different types, and how to design the payload for real applications.

---

## What is a claim

A claim is a piece of information asserted about a subject — usually the user. Claims live in the JWT payload as key-value pairs.

```
{
  "sub": "42",
  "name": "Rohit",
  "role": "ADMIN",
  "iat": 1705296400,
  "exp": 1705300000
}
```

Every key here is a claim. `sub` is the subject (user id), `role` is a custom claim, `iat` is issued at, `exp` is expiry.

---

## Three types of claims

### 1. Registered Claims

Predefined by the JWT spec (RFC 7519). Not required but recommended. Short names to keep the token compact.

| Claim | Full name | Meaning |
|---|---|---|
| `sub` | Subject | Who the token is about — usually user id |
| `iss` | Issuer | Who issued the token — your app name or URL |
| `aud` | Audience | Who the token is intended for |
| `iat` | Issued At | When the token was created (Unix timestamp) |
| `exp` | Expiration | When the token expires (Unix timestamp) |
| `nbf` | Not Before | Token not valid before this time |
| `jti` | JWT ID | Unique identifier for this token |

---

### 2. Public Claims

Claims you define yourself and can register with IANA to avoid collision with other apps. In practice most teams skip the IANA registration and just name custom claims carefully to avoid conflicts.

---

### 3. Private Claims

Custom claims agreed between your frontend and backend. These are the ones you use most in real projects.

```
{
  "role": "ADMIN",
  "permissions": ["read", "write", "delete"],
  "department": "engineering"
}
```

---

## Designing the payload in Spring Boot

```java
public String generateToken(User user) {
    Map<String, Object> claims = new HashMap<>();
    claims.put("role", user.getRole());
    claims.put("email", user.getEmail());

    return Jwts.builder()
        .setClaims(claims)
        .setSubject(String.valueOf(user.getId()))
        .setIssuer("myapp.com")
        .setIssuedAt(new Date())
        .setExpiration(new Date(System.currentTimeMillis() + expiration))
        .signWith(Keys.hmacShaKeyFor(secretKey.getBytes()), SignatureAlgorithm.HS256)
        .compact();
}
```

---

## Reading claims from a token

```java
private Claims getClaims(String token) {
    return Jwts.parserBuilder()
        .setSigningKey(Keys.hmacShaKeyFor(secretKey.getBytes()))
        .build()
        .parseClaimsJws(token)
        .getBody();
}

public String extractUserId(String token) {
    return getClaims(token).getSubject();
}

public String extractRole(String token) {
    return getClaims(token).get("role", String.class);
}

public String extractEmail(String token) {
    return getClaims(token).get("email", String.class);
}

public Date extractExpiration(String token) {
    return getClaims(token).getExpiration();
}
```

---

## What to put in the payload — and what not to

Good to include:
- User id (`sub`)
- Role or roles
- Email (if needed on every request without a DB hit)
- Permissions (for fine-grained access control)
- Token type (access vs refresh)

Never include:
- Password or password hash
- Credit card or financial data
- Sensitive personal information
- Large amounts of data — keep the token small

Remember — the payload is base64 encoded, not encrypted. Anyone with the token can decode and read it.

---

## Using role from JWT in Spring Security

Instead of hitting the database on every request to check the user's role, read it directly from the token:

```java
if (token != null && jwtUtil.isTokenValid(token)) {
    String userId = jwtUtil.extractUserId(token);
    String role = jwtUtil.extractRole(token);

    SimpleGrantedAuthority authority =
        new SimpleGrantedAuthority("ROLE_" + role);

    UsernamePasswordAuthenticationToken auth =
        new UsernamePasswordAuthenticationToken(
            userId, null, List.of(authority)
        );

    SecurityContextHolder.getContext().setAuthentication(auth);
}
```

No database call. The role is already in the token — just read and use it.

---

## Token type claim

Useful when you have both access and refresh tokens — helps reject a refresh token being used where an access token is expected:

```java
public String generateAccessToken(User user) {
    return Jwts.builder()
        .setSubject(String.valueOf(user.getId()))
        .claim("type", "access")
        .claim("role", user.getRole())
        .setExpiration(new Date(System.currentTimeMillis() + ACCESS_EXPIRY))
        .signWith(key, SignatureAlgorithm.HS256)
        .compact();
}

public String generateRefreshToken(User user) {
    return Jwts.builder()
        .setSubject(String.valueOf(user.getId()))
        .claim("type", "refresh")
        .setExpiration(new Date(System.currentTimeMillis() + REFRESH_EXPIRY))
        .signWith(key, SignatureAlgorithm.HS256)
        .compact();
}
```

In the filter, validate the type:

```java
String type = jwtUtil.extractClaim(token, "type");
if (!"access".equals(type)) {
    response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
    return;
}
```

---

## Stuff I want to remember

**What is the difference between registered, public and private claims?**

Registered claims are predefined by the JWT spec — sub, iss, exp, iat etc. Public claims are custom ones meant to be shared publicly, optionally registered with IANA. Private claims are custom ones used only between your own frontend and backend — role, permissions etc. Private claims are what you use in practice.

**Why use sub for user id instead of a custom claim?**

`sub` is the standard registered claim for identifying the subject of the token. Using standard claims makes the token more interoperable and clearly signals intent. You could use a custom claim but `sub` is the conventional place for the primary identifier.

**Should you put the user's role in the JWT?**

Yes, for most applications it's fine. It avoids a database call on every request just to check the role. The tradeoff is that if the role changes, the change doesn't take effect until the current token expires. For most apps that's acceptable. If immediate revocation is critical, you'd need a token blacklist or very short expiry.

---

*The token type claim was something I added after realising someone could technically send a refresh token to a protected endpoint and it would pass signature validation. Adding type and checking it in the filter closes that gap.*