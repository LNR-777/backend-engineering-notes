# Exception Handling Patterns

We've covered `@ControllerAdvice`, custom exceptions, and best practices. This note is about patterns — how to structure exception handling in a real project so it stays clean as the codebase grows.

---

## The layered exception approach

Different layers of the application should throw exceptions that make sense for that layer.

```
Controller layer  →  doesn't throw business exceptions, just calls service
Service layer     →  throws business exceptions (ResourceNotFoundException, DuplicateResourceException)
Repository layer  →  throws DataAccessException (Spring handles this automatically)
```

The service layer is where most of your custom exceptions come from. Keep business logic and its exceptions in the same place.

---

## Exception hierarchy — build your own

Instead of creating independent exception classes, build a small hierarchy:

```java
// base exception for all app exceptions
public class AppException extends RuntimeException {

    private final int statusCode;

    public AppException(String message, int statusCode) {
        super(message);
        this.statusCode = statusCode;
    }

    public int getStatusCode() {
        return statusCode;
    }
}
```

```java
public class ResourceNotFoundException extends AppException {
    public ResourceNotFoundException(String message) {
        super(message, 404);
    }
}

public class DuplicateResourceException extends AppException {
    public DuplicateResourceException(String message) {
        super(message, 409);
    }
}

public class BadRequestException extends AppException {
    public BadRequestException(String message) {
        super(message, 400);
    }
}
```

Now in `@ControllerAdvice` you only need one handler for all custom exceptions:

```java
@ExceptionHandler(AppException.class)
public ResponseEntity<ErrorResponse> handleAppException(
        AppException ex, HttpServletRequest request) {
    ErrorResponse error = new ErrorResponse(
        ex.getStatusCode(),
        ex.getMessage(),
        request.getRequestURI()
    );
    return ResponseEntity.status(ex.getStatusCode()).body(error);
}
```

One handler covers all business exceptions. Add a new exception — no changes needed in the handler as long as it extends `AppException`.

---

## Problem — exception message leaking

```
// bad — exposing internal message directly
throw new ResourceNotFoundException(
    "No value present in Optional.get() for user_id = " + id
);

// good — meaningful message for the client
throw new ResourceNotFoundException("User not found with id: " + id);
```

Exception messages go into your API response. Write them for the frontend developer who reads the response, not for yourself debugging.

---

## Logging strategy

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    // known business exception — no stack trace needed, just a warning
    @ExceptionHandler(AppException.class)
    public ResponseEntity<ErrorResponse> handleAppException(
            AppException ex, HttpServletRequest request) {
        log.warn("Business exception at {}: {}", request.getRequestURI(), ex.getMessage());
        ErrorResponse error = new ErrorResponse(ex.getStatusCode(), ex.getMessage(), request.getRequestURI());
        return ResponseEntity.status(ex.getStatusCode()).body(error);
    }

    // unexpected exception — log full stack trace
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(
            Exception ex, HttpServletRequest request) {
        log.error("Unexpected error at {}: {}", request.getRequestURI(), ex.getMessage(), ex);
        ErrorResponse error = new ErrorResponse(500, "Something went wrong", request.getRequestURI());
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}
```

Two different log levels — `warn` for expected business exceptions, `error` with full stack trace for unexpected ones. Don't log stack traces for every 404 — your logs will be full of noise.

---

## Handling database constraint violations

Sometimes your DB throws a constraint violation before your service layer validation catches it. Handle it:

```java
@ExceptionHandler(DataIntegrityViolationException.class)
public ResponseEntity<ErrorResponse> handleDataIntegrity(
        DataIntegrityViolationException ex, HttpServletRequest request) {

    String message = "Data integrity violation";

    if (ex.getMessage().contains("Duplicate entry")) {
        message = "A record with this value already exists";
    }

    ErrorResponse error = new ErrorResponse(409, message, request.getRequestURI());
    return ResponseEntity.status(HttpStatus.CONFLICT).body(error);
}
```

Better to catch this at the handler level than let a 500 go out for something that's a 409.

---

## Putting the full handler together

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(AppException.class)
    public ResponseEntity<ErrorResponse> handleAppException(AppException ex, HttpServletRequest request) {
        log.warn("Business exception at {}: {}", request.getRequestURI(), ex.getMessage());
        return ResponseEntity.status(ex.getStatusCode())
            .body(new ErrorResponse(ex.getStatusCode(), ex.getMessage(), request.getRequestURI()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex, HttpServletRequest request) {
        String message = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return ResponseEntity.badRequest()
            .body(new ErrorResponse(400, message, request.getRequestURI()));
    }

    @ExceptionHandler(DataIntegrityViolationException.class)
    public ResponseEntity<ErrorResponse> handleDataIntegrity(DataIntegrityViolationException ex, HttpServletRequest request) {
        String message = ex.getMessage().contains("Duplicate entry")
            ? "A record with this value already exists"
            : "Data integrity violation";
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(new ErrorResponse(409, message, request.getRequestURI()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception ex, HttpServletRequest request) {
        log.error("Unexpected error at {}: {}", request.getRequestURI(), ex.getMessage(), ex);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse(500, "Something went wrong", request.getRequestURI()));
    }
}
```

---

## Stuff I want to remember

**Why build an exception hierarchy instead of standalone exception classes?**

One handler in `@ControllerAdvice` covers all custom exceptions instead of writing a separate handler for each. Also easier to add new exception types — just extend the base class, no other changes needed. Cleaner and more scalable as the project grows.

**Should the service layer know about HTTP status codes?**

No. The service layer should only throw meaningful business exceptions. The HTTP status code is a concern of the web layer — the `@ControllerAdvice` handler maps exceptions to status codes. Keeping them separate means your service can be reused outside of HTTP context if needed.

**What is DataIntegrityViolationException?**

It's a Spring exception that wraps database constraint violations — duplicate keys, foreign key violations, not-null constraints. Spring converts the raw JDBC/JPA exception into this. Catching it in your global handler lets you return a 409 instead of a 500 for duplicate data scenarios.

---

*The exception hierarchy pattern saved a lot of repetitive handler code in a project. One base exception, one handler — adding new exception types became trivial.*