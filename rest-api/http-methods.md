# HTTP Methods

So basically HTTP methods tell the server what action you want to perform on a resource.
URL = which resource. Method = what to do with it.

Simple enough, but PUT vs PATCH is where most people get confused (including me initially).

---

## GET

Used to fetch data. Nothing gets modified on the server side.
No request body needed.

```http
GET /api/users/42
```

That's it. You're asking — give me the user with id 42.

---

## POST

Used to create a new resource. You send the data in the request body, server creates it and usually returns the created object with a generated ID.

```http
POST /api/users
Content-Type: application/json

{
  "name": "Rohit",
  "email": "rohit@gmail.com"
}
```

Server responds with `201 Created` and the new user object.

---

## PUT

This one — you're replacing the entire resource. You have to send the full object, not just what changed.

```http
PUT /api/users/42

{
  "id": 42,
  "name": "Rohit Kumar",
  "email": "rohit@gmail.com"
}
```

If you forget to send `email` here, depending on your implementation it might get nulled out. So PUT is all or nothing.

---

## PATCH

Only send what you want to update. Rest stays as it is.

```http
PATCH /api/users/42

{
  "name": "Rohit Kumar"
}
```

Just the name changes. Email, phone, everything else — untouched.

This is what I use most of the time honestly, because in real apps you rarely want to replace the whole object just to change one field.

---

## DELETE

Remove a resource.

```http
DELETE /api/users/42
```

Returns `204 No Content` usually — meaning it worked, nothing to return.

---

## PUT vs PATCH — the part that always comes up in interviews

Think of it like this — say you have a user with 10 fields. You only want to update the phone number.

With **PUT** you'd have to send all 10 fields just to change one. Wasteful, and risky if you accidentally leave something out.

With **PATCH** you just send `{ "phone": "9876543210" }` and you're done.

---

## Idempotency (they will ask this)

Idempotent means — call it once or call it 10 times, the result is the same.

- GET → idempotent (just reading)
- PUT → idempotent (keeps replacing with the same data)
- DELETE → idempotent (after first call, resource is gone, further calls change nothing)
- POST → NOT idempotent (each call creates a new record)
- PATCH → depends on implementation, generally not considered idempotent

Why does this matter? If a network request fails and you don't know if it went through — with idempotent methods you can safely retry. With POST, retrying might create duplicate entries.

---

## In Spring Boot

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }

    @PostMapping
    public ResponseEntity<User> createUser(@RequestBody UserRequest request) {
        User saved = userService.save(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(saved);
    }

    @PutMapping("/{id}")
    public ResponseEntity<User> updateUser(@PathVariable Long id, @RequestBody UserRequest request) {
        return ResponseEntity.ok(userService.replace(id, request));
    }

    @PatchMapping("/{id}")
    public ResponseEntity<User> partialUpdate(@PathVariable Long id, @RequestBody Map<String, Object> fields) {
        return ResponseEntity.ok(userService.partialUpdate(id, fields));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

Nothing fancy here. Just mapping each HTTP method to the right annotation.

---

## Stuff I want to remember

**What's the difference between PUT and PATCH?**

PUT replaces the whole resource — you send the complete object. PATCH is partial, only what changed. Say a user has 10 fields and I just want to update the phone number. With PUT I'd have to send all 10 fields. With PATCH I just send the phone number. I use PATCH for most updates because it's safer and less wasteful.

**Why is POST not idempotent?**

Every POST creates something new. Call it twice with the same data, you get two records. That's why you can't blindly retry a failed POST — you might end up with duplicates.

**Is DELETE idempotent?**

Yes. First call removes the resource. Second call — resource is already gone, state doesn't change. Server might return 404 the second time but the outcome is still the same — resource doesn't exist.

---

*Still exploring how PATCH is handled differently in different backends — will update this as I learn more.*