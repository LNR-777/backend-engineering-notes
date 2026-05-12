# HTTP Status Codes

Server uses these to tell the client what happened with the request. As a backend dev you're responsible for returning the right code — not just 200 for everything and 500 when something breaks.

---

## 2xx — Success

### 200 OK
Request worked, here's your data.

```java
return ResponseEntity.ok(productService.getAll());
```

Use this for GET requests, or any successful operation that returns data.

---

### 201 Created
Something new was created successfully.

```java
return ResponseEntity.status(HttpStatus.CREATED).body(savedUser);
```

Use this for POST when you're creating a new resource — new user, new order, new record. Don't just return 200 for everything, 201 is more accurate here.

---

### 204 No Content
Request worked but there's nothing to return.

```java
return ResponseEntity.noContent().build();
```

This is what you return after a DELETE. Resource is gone, nothing to send back.

---

## 4xx — Client did something wrong

### 400 Bad Request
The data the client sent is invalid. Missing fields, wrong format, failed validation.

```java
return ResponseEntity.badRequest().body("Email is required");
```

---

### 401 Unauthorized
User is not authenticated. They haven't logged in or their token is missing/expired.

```java
return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body("Login required");
```

---

### 403 Forbidden
User is authenticated but doesn't have permission. They're logged in — just not allowed to do this.

```java
return ResponseEntity.status(HttpStatus.FORBIDDEN).body("Access denied");
```

---

### 401 vs 403 — the one interviewers always ask

This trips a lot of people up.

- **401** → server doesn't know who you are. No token, expired token, wrong credentials.
- **403** → server knows exactly who you are, but you're not allowed. Like an employee trying to hit an admin-only endpoint.

Simple way to remember: 401 = *who are you?* 403 = *I know who you are, but no.*

---

### 404 Not Found
Resource doesn't exist.

```java
return ResponseEntity.notFound().build();
// or
return ResponseEntity.status(HttpStatus.NOT_FOUND).body("User not found");
```

---

### 409 Conflict
Request is valid but conflicts with existing data. Most common case — trying to register with an email that already exists.

```java
return ResponseEntity.status(HttpStatus.CONFLICT).body("Email already registered");
```

I used this in my project when handling duplicate user registration. Instead of throwing a 500, returning 409 tells the frontend exactly what went wrong.

---

## 5xx — Server messed up

### 500 Internal Server Error
Something broke on the backend. Unhandled exception, DB connection failure, null pointer — anything unexpected.

This is the one you don't want to return intentionally. If you're seeing a lot of 500s it means your error handling needs work.

In Spring Boot, unhandled exceptions automatically return 500. That's why `@ControllerAdvice` exists — to catch those and return something more meaningful.

---

## Quick Reference

| Code | Meaning |
|---|---|
| 200 | OK — worked, here's data |
| 201 | Created — new resource made |
| 204 | No Content — worked, nothing to return |
| 400 | Bad Request — client sent bad data |
| 401 | Unauthorized — not logged in |
| 403 | Forbidden — logged in but no permission |
| 404 | Not Found — resource doesn't exist |
| 409 | Conflict — duplicate or conflicting data |
| 500 | Internal Server Error — backend broke |

---

## Stuff I want to remember

**What's the difference between 401 and 403?**

401 means the server doesn't know who you are — no token or invalid token. 403 means the server knows who you are but you're not allowed to access that resource. Classic example — a regular user trying to hit an admin endpoint gets 403, not 401.

**Why not just return 200 for everything?**

You can, and a lot of bad APIs do. But it makes the frontend's job harder — they have to read the response body to figure out what happened. Proper status codes mean the frontend knows immediately from the response whether to show a success message, redirect to login, or show an error.

**When do you use 409?**

When the request itself is valid but it conflicts with something that already exists. Duplicate email on registration is the most common case. Instead of letting it throw a 500 from a DB constraint violation, you catch it and return 409 with a clear message.

---

*Updated after understanding how @ControllerAdvice works — makes a lot more sense to handle these centrally rather than in every controller method.*
