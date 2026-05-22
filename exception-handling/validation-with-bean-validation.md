# Input Validation with Bean Validation

Validating request data is something you have to do in every API. You don't want null emails, empty names, or negative amounts reaching your service layer.

Spring Boot has Bean Validation built in — you just add annotations to your request class and Spring handles the rest.

---

## Setup

Add this to your `pom.xml` if it's not already there:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

---

## Annotate your request class

```java
public class UserRequest {

    @NotBlank(message = "Name is required")
    private String name;

    @NotBlank(message = "Email is required")
    @Email(message = "Enter a valid email")
    private String email;

    @NotNull(message = "Age is required")
    @Min(value = 18, message = "Age must be at least 18")
    @Max(value = 100, message = "Age must be under 100")
    private Integer age;

    @NotBlank(message = "Password is required")
    @Size(min = 8, message = "Password must be at least 8 characters")
    private String password;

    // getters and setters
}
```

---

## Enable validation in the controller

Add `@Valid` before `@RequestBody`:

```java
@PostMapping("/users")
public ResponseEntity<User> createUser(@Valid @RequestBody UserRequest request) {
    return ResponseEntity.status(HttpStatus.CREATED).body(userService.save(request));
}
```

That's it. If validation fails, Spring throws `MethodArgumentNotValidException` automatically before your method even runs.

---

## Handle the validation error in @ControllerAdvice

Without a handler you'll get an ugly 400 response. Add this to your `GlobalExceptionHandler`:

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
    List<String> errors = ex.getBindingResult()
        .getFieldErrors()
        .stream()
        .map(error -> error.getField() + ": " + error.getDefaultMessage())
        .collect(Collectors.toList());

    return ResponseEntity
        .status(HttpStatus.BAD_REQUEST)
        .body(new ErrorResponse(400, errors.toString()));
}
```

Now if someone sends invalid data they get:

```json
{
  "status": 400,
  "message": "[email: Enter a valid email, age: Age must be at least 18]",
  "timestamp": "2024-01-15T10:30:00"
}
```

---

## Common validation annotations

| Annotation | What it does |
|---|---|
| `@NotNull` | Field cannot be null |
| `@NotBlank` | String cannot be null or empty or just spaces |
| `@NotEmpty` | Collection or string cannot be null or empty |
| `@Email` | Must be a valid email format |
| `@Min(value)` | Number must be >= value |
| `@Max(value)` | Number must be <= value |
| `@Size(min, max)` | String or collection size within range |
| `@Pattern(regexp)` | Must match a regex pattern |
| `@Positive` | Number must be positive |

---

## NotNull vs NotBlank vs NotEmpty

This is where people get confused:

- `@NotNull` — just checks if it's not null. Empty string `""` passes this.
- `@NotEmpty` — not null and not empty. But `"   "` (spaces) passes this.
- `@NotBlank` — not null, not empty, not just whitespace. Strictest of the three.

For String fields like name or email always use `@NotBlank`. `@NotNull` alone is not enough.

---

## Validating path variables and query params

`@Valid` works for request body. For path variables and query params use `@Validated` on the class and add constraints directly:

```java
@RestController
@RequestMapping("/api/users")
@Validated
public class UserController {

    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(
        @PathVariable @Positive(message = "ID must be positive") Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }
}
```

If this validation fails it throws `ConstraintViolationException` — add a handler for that too in your `GlobalExceptionHandler`.

---

## Stuff I want to remember

**What's the difference between @NotNull, @NotEmpty, and @NotBlank?**

`@NotNull` only checks for null — empty string passes. `@NotEmpty` checks null and empty but spaces pass. `@NotBlank` is strictest — null, empty, and whitespace all fail. For text fields I always use `@NotBlank`.

**Where should validation happen — controller or service?**

Basic input validation like not-null, format checks — controller level with `@Valid`. Business validation like "does this email already exist" — service layer. Two different types of validation, two different places.

**What exception does @Valid throw when validation fails?**

`MethodArgumentNotValidException`. You handle it in `@ControllerAdvice`. If you don't handle it, Spring returns a 400 but with its default error format which isn't great.

---

*Before I knew about Bean Validation I was doing all this manually in the service — null checks everywhere. This cleaned up a lot of code.*