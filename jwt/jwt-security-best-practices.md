# JWT Security Best Practices

JWT is easy to implement but also easy to implement badly. These are the things that matter when you're building a production-ready JWT setup.

---

## Use a strong secret key

Your secret key signs every token. If someone gets the key they can generate valid tokens for any user.

```
# bad — too short, too simple
jwt.secret=mysecret

# good — long, random, stored as environment variable
jwt.secret=${JWT_SECRET}
```

Generate a proper key — at least 256 bits for HS256:

```
// generate a secure key
SecretKey key = Keys.secretKeyFor(SignatureAlgorithm.HS256);
String encoded = Base64.getEncoder().encodeToString(key.getEncoded());
System.out.println(encoded); // store this as your secret
```

Never hardcode the secret in your code or commit it to GitHub.

---

## Keep access tokens short lived

```properties
# 15 minutes to 1 hour is reasonable
jwt.access-token.expiration=3600000
```

Short expiry limits the damage if a token is stolen. Pair with refresh tokens so users don't have to re-login constantly.

---

## Always validate the signature

Never skip signature verification. The whole security of JWT depends on it.

```
// wrong — parsing without signature verification
Jwts.parserBuilder()
    .build()
    .parseClaimsJwt(token); // parses unsigned or signed tokens — dangerous

// correct — always verify with your secret key
Jwts.parserBuilder()
    .setSigningKey(secretKey)
    .build()
    .parseClaimsJws(token); // throws exception if signature doesn't match
```

`parseClaimsJwt` vs `parseClaimsJws` — the `s` in `Jws` means signed. Always use `parseClaimsJws`.

---

## Check token expiry

The jjwt library throws `ExpiredJwtException` automatically when the token is expired — but only if you're using `parseClaimsJws`. Handle it properly:

```java
public boolean isTokenValid(String token) {
    try {
        Jwts.parserBuilder()
            .setSigningKey(secretKey)
            .build()
            .parseClaimsJws(token);
        return true;
    } catch (ExpiredJwtException e) {
        log.warn("Token expired: {}", e.getMessage());
        return false;
    } catch (JwtException e) {
        log.warn("Invalid token: {}", e.getMessage());
        return false;
    }
}
```

Don't catch a generic `Exception` here — be specific so you know what went wrong.

---

## Don't put sensitive data in the payload

JWT payload is base64 encoded — not encrypted. Anyone with the token can decode it.

```
// bad — sensitive data in payload
.claim("password", user.getPassword())
.claim("creditCard", user.getCardNumber())

// good — only what you need for authorization
.setSubject(user.getId().toString())
.claim("role", user.getRole())
```

Put only what's needed for the server to identify and authorize the user. Nothing more.

---

## Use HTTPS only

JWT in a header over plain HTTP can be intercepted. Always run your API over HTTPS in production. Never send tokens over HTTP.

---

## Handle token in Authorization header correctly

```
String authHeader = request.getHeader("Authorization");

// always check format first
if (authHeader == null || !authHeader.startsWith("Bearer ")) {
    filterChain.doFilter(request, response);
    return;
}

String token = authHeader.substring(7); // "Bearer " is 7 characters
```

Don't assume the header is present or formatted correctly. Null checks and prefix checks first.

---

## Refresh token rotation

Every time a refresh token is used to get a new access token — issue a new refresh token and invalidate the old one.

```java
@PostMapping("/refresh")
public ResponseEntity<AuthResponse> refresh(@RequestBody RefreshRequest request) {
    String oldRefreshToken = request.getRefreshToken();

    if (!jwtUtil.isTokenValid(oldRefreshToken)) {
        throw new UnauthorizedException("Invalid refresh token");
    }

    String email = jwtUtil.extractEmail(oldRefreshToken);

    // invalidate old refresh token
    refreshTokenStore.invalidate(oldRefreshToken);

    // generate new tokens
    String newAccessToken = jwtUtil.generateAccessToken(email);
    String newRefreshToken = jwtUtil.generateRefreshToken(email);

    // store new refresh token
    refreshTokenStore.save(email, newRefreshToken);

    return ResponseEntity.ok(new AuthResponse(newAccessToken, newRefreshToken));
}
```

If a stolen refresh token is used — the legitimate user's next refresh fails (their token was already rotated), alerting them to re-login.

---

## Stuff I want to remember

**What happens if the JWT secret key is exposed?**

An attacker can generate valid tokens for any user — including admin accounts. The only fix is to rotate the secret key, which immediately invalidates all existing tokens and forces everyone to re-login. That's why the secret must never be in code or version control.

**Why is refresh token rotation important?**

Without rotation, a stolen refresh token can be used indefinitely until it expires. With rotation, each use invalidates the old token. If a stolen token is used, the real user's token is already different — their next request fails and they know something is wrong.

**Should JWT be encrypted?**

Standard JWT (JWS) is signed but not encrypted. If you need the payload to be unreadable, use JWE (JSON Web Encryption). But for most APIs signing is enough — just don't put sensitive data in the payload.

---

*Most of these I learned after reading about JWT vulnerabilities. The "none algorithm" attack (where attacker changes alg to none to skip verification) is a real thing — that's why always using parseClaimsJws with a key matters.*