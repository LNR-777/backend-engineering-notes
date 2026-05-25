# JWT — JSON Web Token

JWT is a way to securely transmit information between client and server as a token. It's the most common approach for authentication in REST APIs.

Before JWT, people used session-based auth — server stored session data, client sent a session ID. JWT flips that — the server stores nothing, all the info is in the token itself.

---

## Structure of a JWT

A JWT looks like this:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI0MiIsIm5hbWUiOiJSb2hpdCIsImlhdCI6MTcwNTI5NjQwMH0.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

Three parts separated by dots — header, payload, signature.

---

### Header

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

Just says what algorithm is used to sign the token. Base64 encoded.

---

### Payload

```json
{
  "sub": "42",
  "name": "Rohit",
  "role": "USER",
  "iat": 1705296400,
  "exp": 1705300000
}
```

This is the actual data — called claims. `sub` is the subject (usually user id), `iat` is issued at, `exp` is expiry time. You can add custom claims like role.

Important — payload is just Base64 encoded, not encrypted. Anyone can decode it. Never put sensitive data like passwords here.

---

### Signature

```
HMACSHA256(
  base64(header) + "." + base64(payload),
  secretKey
)
```

Server creates this using the header, payload, and a secret key only the server knows. This is what prevents tampering — if someone changes the payload, the signature won't match anymore.

---

## How JWT authentication works end to end

```
1. User sends login request with email + password

2. Server validates credentials

3. Server creates a JWT with user id, role, expiry — signs it with secret key

4. Server sends JWT back to client

5. Client stores the token (usually localStorage or httpOnly cookie)

6. Every subsequent request — client sends the token in Authorization header

7. Server validates the signature, checks expiry, reads user info from payload

8. Server processes the request
```

In code:

```java
// Client sends this header with every request
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

---

## Generating a JWT in Spring Boot

Using the `jjwt` library:

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.11.5</version>
</dependency>
```

```java
@Component
public class JwtUtil {

    @Value("${jwt.secret}")
    private String secretKey;

    @Value("${jwt.expiration}")
    private long expiration; // in milliseconds

    public String generateToken(String userId, String role) {
        return Jwts.builder()
            .setSubject(userId)
            .claim("role", role)
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + expiration))
            .signWith(SignatureAlgorithm.HS256, secretKey)
            .compact();
    }

    public String extractUserId(String token) {
        return Jwts.parser()
            .setSigningKey(secretKey)
            .parseClaimsJws(token)
            .getBody()
            .getSubject();
    }

    public boolean isTokenValid(String token) {
        try {
            Jwts.parser().setSigningKey(secretKey).parseClaimsJws(token);
            return true;
        } catch (JwtException e) {
            return false;
        }
    }
}
```

---

## Access Token vs Refresh Token

One JWT token isn't enough in real apps. You use two:

**Access Token** — short lived (15 mins to 1 hour). Used to access protected endpoints. If stolen, damage is limited because it expires quickly.

**Refresh Token** — long lived (days or weeks). Used only to get a new access token when the old one expires. Stored more securely.

```
Access token expires
    → Client sends refresh token to /auth/refresh
        → Server validates refresh token
            → Server issues new access token
                → Client continues normally
```

Without refresh tokens — user has to login again every time the access token expires. Bad UX.

---

## JWT vs Session based auth

| | JWT | Session |
|---|---|---|
| Server stores state | No | Yes |
| Scalability | Easy — any server can validate | Hard — need shared session store |
| Token size | Larger (carries data) | Small (just a session ID) |
| Logout | Tricky — token valid until expiry | Easy — delete session |
| Mobile friendly | Yes | Harder with cookies |

JWT works well for stateless REST APIs, especially when you have multiple servers or microservices. Sessions work fine for traditional web apps.

---

## Stuff I want to remember

**Can someone decode a JWT and read the payload?**

Yes — payload is just Base64 encoded, not encrypted. Anyone with the token can decode and read the claims. That's why you never put passwords or sensitive data in the payload. The signature just proves the token wasn't tampered with, it doesn't hide the data.

**What happens if a JWT is stolen?**

The attacker can use it until it expires — that's why short expiry times matter. You can't truly invalidate a JWT on the server side without maintaining a blacklist, which defeats the stateless nature. This is one of the known tradeoffs of JWT.

**What's the difference between authentication and authorization in the context of JWT?**

Authentication is verifying who you are — validating the token signature and checking it hasn't expired. Authorization is checking what you're allowed to do — reading the role claim from the payload and deciding if you can access that endpoint.

---

*JWT clicked for me when I stopped thinking of it as a "login token" and started thinking of it as a self-contained identity card — the server doesn't need to remember you because everything it needs to know about you is already in the token.*