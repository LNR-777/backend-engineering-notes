# Exception Handling in Spring Boot

When something goes wrong in your API — invalid input, resource not found, unexpected server error — you need to return a proper response. Not a stack trace, not a blank 500. Something the frontend can actually work with.

Spring Boot gives you `@ControllerAdvice` for this. Instead of handling exceptions in every single controller, you handle them in one place.

---

## The problem without it

Without any exception handling, if a user hits an endpoint with an invalid ID:

```json
{
  "timestamp": "2024-01-15T10:30:00",
  "status": 500,
  "error": "Internal Server Error",
  "path": "/api/users/abc"
}
```

That's Spring's default error response. Not great — it leaks internal details and the status code is wrong (should be 400, not 500).

---

## Custom Exception Class

First, create your own exception:

```java
public class ResourceNotFoundException extends RuntimeException {

    public ResourceNotFoundException(String message) {
        super(message);
    }
}
```

Now throw it wherever needed:

```java
public User findById(Long id) {
    return userRepository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("User not found with id: " + id));
}
```

---

## @ControllerAdvice — handling it globally

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(404, ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.joining(", "));
        ErrorResponse error = new ErrorResponse(400, message);
        return ResponseEntity.badRequest().body(error);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception ex) {
        ErrorResponse error = new ErrorResponse(500, "Something went wrong");
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}
```

---

## ErrorResponse — a clean response structure

Instead of returning raw strings, use a proper response object:

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

Now every error looks like this:

```json
{
  "status": 404,
  "message": "User not found with id: 42",
  "timestamp": "2024-01-15T10:30:00"
}
```

Clean, consistent, useful.

---

## How the flow works

```
Request comes in
    → Controller calls Service
        → Service throws ResourceNotFoundException
            → @ControllerAdvice catches it
                → Returns 404 with proper ErrorResponse
```

The controller doesn't need to know about exception handling at all. `@ControllerAdvice` intercepts it before the response goes out.

---

## Stuff I want to remember

**Why use @ControllerAdvice instead of try-catch in every controller?**

Because try-catch in every method is repetitive and messy. If you want to change the error format later you'd have to update every controller. With `@ControllerAdvice` it's in one place — change it once, applies everywhere.

**What's the difference between @ControllerAdvice and @RestControllerAdvice?**

`@RestControllerAdvice` is just `@ControllerAdvice` + `@ResponseBody` combined. Since we're building REST APIs that return JSON, I use `@RestControllerAdvice` so I don't have to add `@ResponseBody` separately.

**What if no @ExceptionHandler matches the exception?**

It falls through to the generic `Exception.class` handler if you have one. If you don't, Spring Boot's default error handling kicks in and you get that ugly default response. That's why having a catch-all `Exception.class` handler is a good safety net.

---

*This clicked for me when I realized that without @ControllerAdvice, every unhandled exception was returning 500 even for things like "user not found" — which should be a 404. Centralizing it fixed all of that.*