# Error Codes vs Exceptions

These are two different things that work together. Exceptions are how Java handles failures internally. HTTP error codes are how your API tells the client what went wrong. Knowing how to map one to the other correctly is a big part of building a clean REST API.

---

## What's the difference

An **exception** is a Java mechanism. When something goes wrong at runtime — user not found, database down, null value where you didn't expect one — Java throws an exception and unwinds the call stack until something catches it.

An **HTTP status code** is a number in the HTTP response that tells the client what happened. `200` means success, `404` means not found, `500` means the server messed up.

They're not the same thing but they're related. When your API throws an exception, you need to decide what HTTP status code goes back to the client. That mapping is your job.

---

## The mapping

This is roughly how exceptions should map to HTTP status codes:

```
ResourceNotFoundException      → 404 Not Found
DuplicateResourceException     → 409 Conflict
IllegalArgumentException       → 400 Bad Request
MethodArgumentNotValidException → 400 Bad Request
AccessDeniedException          → 403 Forbidden
AuthenticationException        → 401 Unauthorized
Exception (anything else)      → 500 Internal Server Error
```

The wrong mapping is a common mistake. If a user isn't found and you return 500, the client thinks the server crashed. If the issue is actually invalid input and you return 404, that's misleading too. The status code is a contract — it tells the client how to respond.

---

## Doing the mapping in @ControllerAdvice

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse(404, ex.getMessage()));
    }

    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<ErrorResponse> handleConflict(DuplicateResourceException ex) {
        return ResponseEntity
            .status(HttpStatus.CONFLICT)
            .body(new ErrorResponse(409, ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors()
            .stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(new ErrorResponse(400, message));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception ex) {
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse(500, "Something went wrong"));
    }
}
```

The `Exception.class` handler at the bottom is the safety net. Anything not caught by a specific handler falls through to this one. You don't want stack traces going back to the client.

---

## HTTP status codes you'll actually use

Not all 70+ HTTP status codes are useful in practice. Here are the ones that come up constantly in backend projects:

**2xx — Success**

| Code | Meaning | When to use |
|---|---|---|
| `200 OK` | Request succeeded | GET, PUT, PATCH responses |
| `201 Created` | Resource created | POST that creates something |
| `204 No Content` | Success, nothing to return | DELETE responses |

**4xx — Client error**

| Code | Meaning | When to use |
|---|---|---|
| `400 Bad Request` | Invalid input | Validation failures, malformed request |
| `401 Unauthorized` | Not authenticated | No token, expired token |
| `403 Forbidden` | Authenticated but no permission | Wrong role, can't access this resource |
| `404 Not Found` | Resource doesn't exist | User id not in DB |
| `409 Conflict` | Duplicate or state conflict | Email already registered |
| `422 Unprocessable Entity` | Request understood but semantically wrong | Valid JSON but business logic rejects it |

**5xx — Server error**

| Code | Meaning | When to use |
|---|---|---|
| `500 Internal Server Error` | Unexpected server failure | Catch-all for unhandled exceptions |
| `503 Service Unavailable` | Server can't handle requests | Database down, circuit breaker open |

---

## 401 vs 403 — the one everyone mixes up

`401 Unauthorized` means "who are you?" — the request has no valid authentication. Missing token, expired token, invalid token.

`403 Forbidden` means "I know who you are but you can't do this." — authenticated but not authorized. A `ROLE_USER` trying to hit an admin endpoint gets a 403, not a 401.

Mix these up in an interview and the interviewer will notice.

---

## What not to do

```
// Don't swallow exceptions silently
try {
    userService.delete(id);
} catch (Exception e) {
    // nothing here — caller has no idea it failed
}
```

```
// Don't throw the wrong status
throw new ResponseStatusException(HttpStatus.INTERNAL_SERVER_ERROR, "User not found");
// This should be 404, not 500
```

```
// Don't leak stack traces or internal info
{
  "error": "org.hibernate.exception.ConstraintViolationException: could not execute statement"
}
// The client doesn't need to know your ORM or DB internals
```

---

## ErrorResponse should include the status code

A small thing but it matters. The status code is already in the HTTP response header, but including it in the body too means the client doesn't have to check two places:

```java
public class ErrorResponse {
    private int status;
    private String message;
    private LocalDateTime timestamp;

    public ErrorResponse(int status, String message) {
        this.status = status;
        this.message = message;
        this.timestamp = LocalDateTime.now();
    }

    // getters
}
```

Response body:

```
{
  "status": 404,
  "message": "User not found with id: 99",
  "timestamp": "2024-01-15T10:30:00"
}
```

Consistent across all error responses. Frontend knows exactly what to expect.

---

## Stuff I want to remember

**What's the difference between a Java exception and an HTTP error code?**

Exception is an internal Java mechanism for signaling failures at runtime. HTTP error code is the number in the response that tells the client what happened. You catch the exception in `@ControllerAdvice` and decide which HTTP code maps to it. They work together but they're different things.

**What's the difference between 401 and 403?**

`401` means you're not authenticated — no valid token. `403` means you are authenticated but you don't have permission. A regular user trying to access an admin route gets 403, not 401.

**Why return 409 for duplicate email instead of 400?**

`400 Bad Request` means the request itself is malformed or invalid input. `409 Conflict` means the request is valid but it conflicts with the current state of the resource — the email already exists. It's more precise and the client can handle it differently (show "email already taken" vs "invalid input").

**Should you include the HTTP status in the response body if it's already in the header?**

Yes, it's good practice. Clients sometimes work with the body only (logging, error aggregation tools) and having the status code in the body means they don't need the HTTP headers. It also makes debugging easier — the body alone tells you everything about what went wrong.

---

*The 401 vs 403 confusion cost me an interview question once. Kept thinking 401 = not allowed, 403 = needs login. It's the opposite. 401 = prove who you are, 403 = I know who you are and the answer is no.*