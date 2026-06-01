# Exception Handling — Best Practices

We covered `@ControllerAdvice` and custom exceptions in the previous notes. This note is about putting it all together properly — how to structure your error handling so it's clean, consistent, and actually useful for the frontend.

---

## Consistent error response structure

Every error your API returns should look the same. Frontend developers shouldn't have to guess what format the error will be in.

Bad — inconsistent errors:

```
// sometimes this
{ "error": "User not found" }

// sometimes this
{ "message": "Not found", "code": 404 }

// sometimes just a string
"Something went wrong"
```

Good — always the same structure:

```java
public class ErrorResponse {

    private int status;
    private String message;
    private String path;
    private LocalDateTime timestamp;

    public ErrorResponse(int status, String message, String path) {
        this.status = status;
        this.message = message;
        this.path = path;
        this.timestamp = LocalDateTime.now();
    }

    // getters
}
```

```
{
  "status": 404,
  "message": "User not found with id: 42",
  "path": "/api/users/42",
  "timestamp": "2024-01-15T10:30:00"
}
```

---

## Full GlobalExceptionHandler

Putting everything together in one place:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // 404 — resource not found
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(
            ResourceNotFoundException ex, HttpServletRequest request) {
        ErrorResponse error = new ErrorResponse(404, ex.getMessage(), request.getRequestURI());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    // 409 — duplicate resource
    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<ErrorResponse> handleDuplicate(
            DuplicateResourceException ex, HttpServletRequest request) {
        ErrorResponse error = new ErrorResponse(409, ex.getMessage(), request.getRequestURI());
        return ResponseEntity.status(HttpStatus.CONFLICT).body(error);
    }

    // 400 — validation errors
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(
            MethodArgumentNotValidException ex, HttpServletRequest request) {
        String message = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.joining(", "));
        ErrorResponse error = new ErrorResponse(400, message, request.getRequestURI());
        return ResponseEntity.badRequest().body(error);
    }

    // 401 — unauthorized
    @ExceptionHandler(UnauthorizedException.class)
    public ResponseEntity<ErrorResponse> handleUnauthorized(
            UnauthorizedException ex, HttpServletRequest request) {
        ErrorResponse error = new ErrorResponse(401, ex.getMessage(), request.getRequestURI());
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body(error);
    }

    // 400 — bad request
    @ExceptionHandler(BadRequestException.class)
    public ResponseEntity<ErrorResponse> handleBadRequest(
            BadRequestException ex, HttpServletRequest request) {
        ErrorResponse error = new ErrorResponse(400, ex.getMessage(), request.getRequestURI());
        return ResponseEntity.badRequest().body(error);
    }

    // 500 — catch all for anything unexpected
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(
            Exception ex, HttpServletRequest request) {
        ErrorResponse error = new ErrorResponse(500, "Something went wrong", request.getRequestURI());
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}
```

Adding `HttpServletRequest` as a parameter gives you the request path — useful for debugging.

---

## Don't expose internal details in 500 errors

```java
// bad — leaks internal info
ErrorResponse error = new ErrorResponse(500, ex.getMessage(), path);

// good — generic message for unexpected errors
ErrorResponse error = new ErrorResponse(500, "Something went wrong", path);
```

Log the actual exception internally, return a generic message to the client. Stack traces and DB error messages should never reach the API response.

```java
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse> handleGeneral(Exception ex, HttpServletRequest request) {
    log.error("Unexpected error at {}: {}", request.getRequestURI(), ex.getMessage(), ex);
    ErrorResponse error = new ErrorResponse(500, "Something went wrong", request.getRequestURI());
    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
}
```

---

## Where exceptions should come from

```
Controller  →  calls Service  →  Service throws exception
                                      ↓
                              @ControllerAdvice catches it
                                      ↓
                              returns proper ErrorResponse
```

Controller stays clean — no try-catch, no error handling logic. That all lives in the service and the global handler.

```java
@RestController
public class UserController {

    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(userService.findById(id));
        // if user not found, service throws ResourceNotFoundException
        // @ControllerAdvice handles it — controller doesn't care
    }
}
```

---

## Stuff I want to remember

**Should you log exceptions in @ControllerAdvice?**

Yes for unexpected errors (500). For known business exceptions like ResourceNotFoundException you usually don't need to log — it's expected behaviour. For 500s always log the full stack trace so you can debug. Just don't send that stack trace to the client.

**What is the difference between @ControllerAdvice and @RestControllerAdvice?**

`@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`. Since REST APIs return JSON, use `@RestControllerAdvice` — saves adding `@ResponseBody` to every handler method.

**Why should error responses be consistent?**

Because the frontend has to handle them. If every error looks different, the frontend needs different handling logic for each case. Consistent structure means one error handler on the frontend side that works for everything.

---

*Once I set this up properly in a project, debugging became much easier — every error had the path, status, message and timestamp. No more guessing where something broke.*