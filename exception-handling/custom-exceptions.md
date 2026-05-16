# Custom Exceptions in Spring Boot

In the last note we used `@ControllerAdvice` to handle exceptions globally. But we were throwing generic exceptions. In real projects you want your own exception classes — ones that are specific to your domain and carry meaningful messages.

---

## Why custom exceptions

If you just throw `RuntimeException("something went wrong")` everywhere — your `@ControllerAdvice` can't distinguish between a "user not found" and a "duplicate email" situation. Both would be caught by the same generic handler and return the same response.

Custom exceptions let you handle different error scenarios differently.

---

## Basic custom exception

```java
public class ResourceNotFoundException extends RuntimeException {

    public ResourceNotFoundException(String message) {
        super(message);
    }
}
```

That's it. Extend `RuntimeException` so it's unchecked — you don't have to declare it in method signatures everywhere.

Throw it like this:

```java
public User findById(Long id) {
    return userRepository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("User not found with id: " + id));
}
```

---

## A few more you'd typically create

```java
// when trying to create something that already exists
public class DuplicateResourceException extends RuntimeException {

    public DuplicateResourceException(String message) {
        super(message);
    }
}
```

```java
// when business logic validation fails
public class BadRequestException extends RuntimeException {

    public BadRequestException(String message) {
        super(message);
    }
}
```

```java
// when user doesn't have permission
public class UnauthorizedException extends RuntimeException {

    public UnauthorizedException(String message) {
        super(message);
    }
}
```

---

## Handling them in @ControllerAdvice

Now each exception gets its own handler with the right status code:

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
    public ResponseEntity<ErrorResponse> handleDuplicate(DuplicateResourceException ex) {
        return ResponseEntity
            .status(HttpStatus.CONFLICT)
            .body(new ErrorResponse(409, ex.getMessage()));
    }

    @ExceptionHandler(BadRequestException.class)
    public ResponseEntity<ErrorResponse> handleBadRequest(BadRequestException ex) {
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(new ErrorResponse(400, ex.getMessage()));
    }

    @ExceptionHandler(UnauthorizedException.class)
    public ResponseEntity<ErrorResponse> handleUnauthorized(UnauthorizedException ex) {
        return ResponseEntity
            .status(HttpStatus.UNAUTHORIZED)
            .body(new ErrorResponse(401, ex.getMessage()));
    }

    // catch-all for anything unexpected
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception ex) {
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse(500, "Something went wrong"));
    }
}
```

---

## Real usage in a service

```java
@Service
public class UserService {

    public User findById(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User not found with id: " + id));
    }

    public User createUser(UserRequest request) {
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new DuplicateResourceException("Email already registered: " + request.getEmail());
        }
        return userRepository.save(new User(request));
    }
}
```

Clean. No try-catch in the service, no status code logic anywhere except the handler. Each exception class carries just the message, the handler decides the status code.

---

## Adding more context to exceptions (optional)

Sometimes you want to pass more than just a message — like an error code or field name:

```java
public class ValidationException extends RuntimeException {

    private final String field;

    public ValidationException(String field, String message) {
        super(message);
        this.field = field;
    }

    public String getField() {
        return field;
    }
}
```

Then in the handler:

```java
@ExceptionHandler(ValidationException.class)
public ResponseEntity<ErrorResponse> handleValidation(ValidationException ex) {
    return ResponseEntity
        .status(HttpStatus.BAD_REQUEST)
        .body(new ErrorResponse(400, ex.getField() + ": " + ex.getMessage()));
}
```

---

## Stuff I want to remember

**Why extend RuntimeException and not Exception?**

RuntimeException is unchecked — you don't have to declare it with `throws` in every method signature. Checked exceptions (extending Exception directly) force you to handle or declare them everywhere which gets messy fast in a Spring app.

**Where should you throw custom exceptions — controller or service?**

Service layer. The controller should just call the service and return the response. Business logic and validation lives in the service, so that's where the exceptions should come from. Controller stays thin.

**What's the difference between @ControllerAdvice and @RestControllerAdvice?**

`@RestControllerAdvice` combines `@ControllerAdvice` and `@ResponseBody`. Since we're building REST APIs that return JSON, use `@RestControllerAdvice` — saves you from adding `@ResponseBody` to every handler method.

---

*Started using custom exceptions after realizing all my errors were returning 500 even for simple "not found" cases. Once I set this up the API responses became much more predictable.*