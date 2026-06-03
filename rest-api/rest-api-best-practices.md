# REST API Design Best Practices

Knowing REST concepts is one thing. Designing a clean API that others can actually work with is another. These are the things I've picked up that separate a well designed API from a messy one.

---

## Use nouns for URLs, not verbs

URLs represent resources. HTTP methods represent actions. Don't mix them.

```
// bad
GET  /getUsers
POST /createUser
PUT  /updateUser/42
DELETE /deleteUser/42

// good
GET    /api/users
POST   /api/users
PUT    /api/users/42
DELETE /api/users/42
```

The URL says what, the method says how. Keep it that way.

---

## Use plural nouns

Consistency matters. Always plural, even for single resource endpoints.

```
// consistent — always plural
GET /api/users
GET /api/users/42
GET /api/products
GET /api/products/5

// inconsistent — confusing
GET /api/user
GET /api/users/42
```

---

## Nested resources for relationships

When one resource belongs to another, nest the URL.

```
GET /api/users/42/orders        // all orders for user 42
GET /api/users/42/orders/7      // specific order 7 of user 42
POST /api/users/42/orders       // create order for user 42
```

Don't go more than 2-3 levels deep though — it gets messy.

```
// too deep, hard to read
GET /api/users/42/orders/7/items/3/reviews
```

---

## Return correct status codes

Already covered in the status codes note but worth repeating — don't return 200 for everything.

```
// creating something
return ResponseEntity.status(HttpStatus.CREATED).body(savedUser);   // 201

// deleting something
return ResponseEntity.noContent().build();                           // 204

// resource not found
throw new ResourceNotFoundException("User not found");              // 404
```

---

## Use query params for filtering, sorting, pagination

```
GET /api/products?category=electronics&sort=price,asc&page=0&size=10
```

Not a new endpoint for every filter combination. One endpoint, query params handle the rest.

---

## Version your API

Already covered in the versioning note — but always have a version in your URL from day one. Even if you're on v1.

```
GET /api/v1/users
```

Saves you from breaking clients later when you need to make changes.

---

## Use consistent response structure

Every success response should follow the same format. Makes frontend integration predictable.

```java
public class ApiResponse<T> {

    private boolean success;
    private String message;
    private T data;

    public static <T> ApiResponse<T> success(T data) {
        return new ApiResponse<>(true, "Success", data);
    }

    public static <T> ApiResponse<T> success(String message, T data) {
        return new ApiResponse<>(true, message, data);
    }
}
```

```
{
  "success": true,
  "message": "User created successfully",
  "data": { "id": 42, "name": "Rohit" }
}
```

---

## Validate input early

Don't let invalid data reach your service layer. Use Bean Validation at the controller level.

```java
@PostMapping
public ResponseEntity<ApiResponse<User>> createUser(@Valid @RequestBody UserRequest request) {
    // if we reach here, input is already validated
}
```

---

## Don't expose internal details

- Don't return stack traces in error responses
- Don't expose database IDs in a way that reveals your DB structure
- Don't return fields that contain sensitive data (passwords, tokens, internal flags)

```
// bad — returning the whole entity with sensitive fields
return ResponseEntity.ok(userRepository.findById(id));

// good — return a DTO with only what the client needs
return ResponseEntity.ok(new UserResponse(user));
```

---

## Stuff I want to remember

**Why use nouns instead of verbs in URLs?**

Because HTTP methods already express the action — GET, POST, PUT, DELETE. Adding verbs to URLs like `/getUser` is redundant and inconsistent. URLs should identify resources, not describe operations on them.

**How deep should nested URLs go?**

Two levels is usually enough. `/users/42/orders` makes sense. Going deeper like `/users/42/orders/7/items/3` gets hard to read and maintain. If it gets that complex, consider restructuring the resource design.

**Should you always wrap responses in a standard structure?**

It depends on the team but having a consistent wrapper makes life easier for frontend developers. They always know where the data is, whether the request succeeded, and what the message is — without parsing different response formats for different endpoints.

---

*Most of these I learned by looking at well designed public APIs like Stripe and GitHub and noticing the patterns they follow consistently.*