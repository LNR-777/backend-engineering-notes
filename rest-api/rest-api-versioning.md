# REST API Versioning

When you build an API and clients start using it, you can't just change endpoints or response structures whenever you want — that breaks existing clients. Versioning is how you handle changes without breaking things.

There are a few approaches. Each has tradeoffs.

---

## Why versioning matters

Say your current API returns:

```json
{
  "name": "Rohit Sharma",
  "phone": "9876543210"
}
```

Now you want to split `name` into `firstName` and `lastName`. If you just change the response, every client that reads `name` breaks.

With versioning you introduce `/v2/` with the new structure while `/v1/` stays as it is. Old clients keep working, new clients use the updated version.

---

## 1. URL Versioning

Version is part of the URL. Most common approach.

```
GET /api/v1/users
GET /api/v2/users
```

In Spring Boot:

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserV1Controller {

    @GetMapping("/{id}")
    public ResponseEntity<UserV1Response> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(userService.getUserV1(id));
    }
}

@RestController
@RequestMapping("/api/v2/users")
public class UserV2Controller {

    @GetMapping("/{id}")
    public ResponseEntity<UserV2Response> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(userService.getUserV2(id));
    }
}
```

**Good:** Easy to understand, easy to test, visible in browser and Postman.
**Bad:** URL should ideally represent a resource, not a version. Purists don't like this.

In practice — this is what most teams use because it's simple.

---

## 2. Header Versioning

Version is passed in a custom request header.

```
GET /api/users
API-Version: 2
```

In Spring Boot:

```java
@GetMapping(value = "/users/{id}", headers = "API-Version=1")
public ResponseEntity<UserV1Response> getUserV1(@PathVariable Long id) {
    return ResponseEntity.ok(userService.getUserV1(id));
}

@GetMapping(value = "/users/{id}", headers = "API-Version=2")
public ResponseEntity<UserV2Response> getUserV2(@PathVariable Long id) {
    return ResponseEntity.ok(userService.getUserV2(id));
}
```

**Good:** URL stays clean, more "RESTful" technically.
**Bad:** Can't test directly in a browser, harder to cache, less visible.

---

## 3. Request Param Versioning

Version as a query parameter.

```
GET /api/users/42?version=1
GET /api/users/42?version=2
```

In Spring Boot:

```java
@GetMapping(value = "/users/{id}", params = "version=1")
public ResponseEntity<UserV1Response> getUserV1(@PathVariable Long id) {
    return ResponseEntity.ok(userService.getUserV1(id));
}

@GetMapping(value = "/users/{id}", params = "version=2")
public ResponseEntity<UserV2Response> getUserV2(@PathVariable Long id) {
    return ResponseEntity.ok(userService.getUserV2(id));
}
```

**Good:** Easy to test in browser.
**Bad:** Mixing versioning with query params that should be for filtering — feels off.

---

## 4. Accept Header Versioning (Media Type)

Version is part of the `Accept` header.

```
GET /api/users/42
Accept: application/vnd.myapp.v1+json
```

Most "correct" according to REST principles but rarely used in real projects because it's complex to implement and hard to test.

---

## Which one to use

Honestly for most projects — **URL versioning**. It's simple, everyone understands it, easy to test in Postman. The other approaches are more "correct" on paper but add complexity without much benefit for most teams.

GitHub API uses URL versioning. Stripe uses it too. That's good enough validation.

---

## Stuff I want to remember

**Why do we need API versioning?**

Once an API is live and clients are using it, you can't change the contract without breaking them. Versioning lets you introduce breaking changes in a new version while keeping the old version running for existing clients.

**Which versioning approach is best?**

Depends on the team but URL versioning is the most practical. It's visible, easy to test, easy to document. Header versioning is cleaner from a REST perspective but harder to work with day to day.

**When should you version an API?**

When you're making a breaking change — removing a field, renaming a field, changing a data type, changing the URL structure. Non-breaking changes like adding a new optional field don't need a new version.

---

*Learned this the hard way — changed a response structure once and broke the frontend. After that started taking versioning seriously.*