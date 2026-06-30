# Checked vs Unchecked Exceptions

Java has two categories of exceptions and the difference affects how you write and handle code, especially in Spring Boot services.

---

## Checked Exceptions

Must be either caught or declared with `throws`. The compiler forces you to handle them.

```java
public void readFile(String path) throws IOException {
    FileReader reader = new FileReader(path); // IOException is checked
}
```

If you don't catch or declare it, the code won't compile.

Examples — `IOException`, `SQLException`, `ClassNotFoundException`.

---

## Unchecked Exceptions

Extend `RuntimeException`. The compiler doesn't force you to catch or declare them.

```java
public int divide(int a, int b) {
    return a / b; // ArithmeticException is unchecked, no throws needed
}
```

Examples — `NullPointerException`, `ArithmeticException`, `IllegalArgumentException`, `ArrayIndexOutOfBoundsException`.

---

## Why this matters for Spring Boot

All our custom exceptions in the exception-handling notes extend `RuntimeException` — and that's intentional.

```java
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}
```

If we extended `Exception` (checked) instead, every method that could throw it would need `throws ResourceNotFoundException` in its signature — including the controller, the service, everywhere up the call chain. That gets messy fast, especially with Spring's `@ControllerAdvice` pattern which relies on exceptions bubbling up automatically.

```
// with checked exception — ugly
public User findById(Long id) throws ResourceNotFoundException {
    return userRepository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("not found"));
}

@GetMapping("/{id}")
public ResponseEntity<User> getUser(@PathVariable Long id) throws ResourceNotFoundException {
    return ResponseEntity.ok(userService.findById(id));
}
```

```
// with unchecked exception — clean, no throws needed anywhere
public User findById(Long id) {
    return userRepository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("not found"));
}

@GetMapping("/{id}")
public ResponseEntity<User> getUser(@PathVariable Long id) {
    return ResponseEntity.ok(userService.findById(id));
}
```

`@ControllerAdvice` catches it regardless of whether it's declared — that's why unchecked is the standard choice for custom business exceptions.

---

## try-with-resources

When working with resources that need closing (file streams, DB connections), use try-with-resources instead of manual try-finally.

```
// old way — verbose, error prone if you forget finally
FileReader reader = null;
try {
    reader = new FileReader("file.txt");
    // use reader
} finally {
    if (reader != null) {
        reader.close();
    }
}
```

```
// try-with-resources — cleaner, auto-closes
try (FileReader reader = new FileReader("file.txt")) {
    // use reader
} catch (IOException e) {
    // handle exception
}
```

The resource is automatically closed when the try block exits — even if an exception is thrown. Any class implementing `AutoCloseable` can be used this way.

---

## Multi-catch

Catch multiple exception types in one block if you want the same handling:

```
try {
    // some code
} catch (IOException | SQLException e) {
    log.error("Error occurred: {}", e.getMessage());
}
```

Avoids repeating the same catch logic for different exception types.

---

## Custom checked exception (rare, but good to know)

```java
public class InsufficientFundsException extends Exception {
    public InsufficientFundsException(String message) {
        super(message);
    }
}
```

You'd use a checked exception when you want to force the caller to explicitly handle a specific failure scenario — like in banking operations where ignoring the exception could be dangerous. But in typical Spring Boot REST API development, unchecked is almost always preferred for the reasons above.

---

## Stuff I want to remember

**What's the difference between checked and unchecked exceptions?**

Checked exceptions must be caught or declared with throws — the compiler enforces this. Unchecked exceptions extend RuntimeException and don't require explicit handling. IOException is checked, NullPointerException is unchecked.

**Why do Spring Boot custom exceptions extend RuntimeException?**

Because checked exceptions would require adding throws declarations through every layer — controller, service, repository — which is messy. Unchecked exceptions bubble up naturally to @ControllerAdvice without any throws declarations needed anywhere in the call chain.

**What is try-with-resources and why use it?**

It automatically closes resources like file streams or connections when the try block finishes, even if an exception occurs. Cleaner than manual try-finally and less error prone — you can't forget to close the resource.

---

*The checked vs unchecked distinction made a lot more sense once I tried writing a checked custom exception and saw how many places needed a throws declaration just to make it compile. Switched to RuntimeException immediately after that.*