# JWT vs OAuth2

People often mix these up or think they're competing technologies. They're not — they solve different problems and actually work together a lot of the time.

---

## The core difference

**JWT** is a token format — a way to structure and sign data so it can be safely transmitted and verified.

**OAuth2** is an authorization framework — a set of rules for how a user can grant a third-party application limited access to their resources, without sharing their password.

JWT answers "how do we structure and verify a token." OAuth2 answers "how does a user authorize an app to act on their behalf."

---

## Where they overlap

OAuth2 doesn't specify what format the access token must be in — but JWT is a very common choice for it. So when you "login with Google" and Google gives you an access token, that token is often a JWT.

```
OAuth2 flow → produces an access token → that token is often formatted as JWT
```

---

## A simple example to understand OAuth2

Say you want to use a third party app that needs access to your Google Calendar.

```
1. App redirects you to Google's login page
2. You login on Google's site (app never sees your password)
3. Google asks — "Allow this app to access your calendar?"
4. You approve
5. Google redirects back to the app with an authorization code
6. App exchanges that code for an access token
7. App uses the access token to call Google Calendar API on your behalf
```

This is the core idea of OAuth2 — delegated access without sharing credentials.

---

## OAuth2 roles

- **Resource Owner** — you, the user
- **Client** — the third party app requesting access
- **Authorization Server** — issues tokens (Google's auth server)
- **Resource Server** — holds the actual data (Google Calendar API)

---

## When to use which

**Use plain JWT auth** (what we built in the jwt-implementation note) when:
- You own both the frontend and backend
- Simple login system — username, password, your own server issues the token
- No need for third party access delegation

**Use OAuth2** when:
- You want "Login with Google/GitHub/Facebook"
- A third party app needs limited access to a user's data on another service
- You're building a system where external apps need to integrate with your API on behalf of users

---

## Spring Security support for both

For plain JWT — what we already covered, custom filter + JwtUtil.

For OAuth2:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

```properties
spring.security.oauth2.client.registration.google.client-id=YOUR_CLIENT_ID
spring.security.oauth2.client.registration.google.client-secret=YOUR_CLIENT_SECRET
spring.security.oauth2.client.registration.google.scope=email,profile
```

Spring Security handles most of the OAuth2 flow automatically once configured — you don't write the redirect logic yourself.

---

## OpenID Connect — one more term that comes up

OAuth2 handles authorization (what you can access). It doesn't define authentication (who you are) on its own.

OpenID Connect (OIDC) is built on top of OAuth2 and adds an identity layer — it introduces an ID Token (which is a JWT) that proves who the user is.

```
OAuth2        → authorization (can this app access this resource?)
OpenID Connect → authentication (who is this user?) — built on top of OAuth2
```

That's why "Login with Google" actually uses OIDC, not just plain OAuth2 — because logging in requires knowing who the user is, not just what they can access.

---

## Stuff I want to remember

**What is the difference between JWT and OAuth2?**

JWT is a token format — how data is structured and signed. OAuth2 is a framework for delegated authorization — how a user grants limited access to a third party app. They're often used together — OAuth2 defines the flow, JWT is frequently the format of the resulting access token.

**When would you use OAuth2 instead of building your own JWT auth?**

When you need users to login via Google, GitHub, or similar, or when a third-party application needs limited access to your API on behalf of a user. If you're just building your own simple login system for your own app, plain JWT auth is simpler and sufficient.

**What is OpenID Connect and how does it relate to OAuth2?**

OIDC is built on top of OAuth2 and adds authentication — proving who the user is, via an ID token. OAuth2 alone only handles authorization (what an app can access), not identity. "Login with Google" uses OIDC because it needs to confirm your identity, not just grant access to a resource.

---

*Used to think OAuth2 and JWT were two ways of doing the same thing. Understanding that one is a token format and the other is an authorization flow — and that they work together — cleared up a lot of confusion.*