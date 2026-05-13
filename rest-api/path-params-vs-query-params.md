# Path Params vs Query Params vs Request Body

Three ways to send data to a REST API. Knowing when to use which one is something that comes up a lot — both in real projects and interviews.

---

## Path Parameters

Part of the URL itself. Used to identify a specific resource.

```
GET /api/users/42
```

Here `42` is the path param. You're saying — give me the user whose id is 42.

In Spring Boot:

```java
@GetMapping("/users/{id}")
public ResponseEntity<User> getUser(@PathVariable Long id) {
    return ResponseEntity.ok(userService.findById(id));
}
```

Use path params when you're targeting one specific resource.

---

## Query Parameters

Appended to the URL after `?`. Used for filtering, sorting, searching, pagination — anything optional.

```
GET /api/users?role=admin&city=pune
```

Here `role` and `city` are query params. You're saying — give me users, but filter by role and city.

In Spring Boot:

```java
@GetMapping("/users")
public ResponseEntity<List<User>> getUsers(
    @RequestParam(required = false) String role,
    @RequestParam(required = false) String city) {
    return ResponseEntity.ok(userService.findByFilters(role, city));
}
```

`required = false` means these are optional. If not sent, just ignore them.

---

## Request Body

Data sent inside the HTTP request body. Used when you're creating or updating something — usually with POST, PUT, PATCH.

```
POST /api/users
Content-Type: application/json

{
  "name": "Rohit",
  "email": "rohit@gmail.com",
  "role": "admin"
}
```

In Spring Boot:

```java
@PostMapping("/users")
public ResponseEntity<User> createUser(@RequestBody UserRequest request) {
    return ResponseEntity.status(HttpStatus.CREATED).body(userService.save(request));
}
```

You can't send a request body with GET — only POST, PUT, PATCH, DELETE.

---

## When to use what

| Situation | Use |
|---|---|
| Fetch a specific user by id | Path param — `/users/42` |
| Search users by name or filter by role | Query param — `/users?role=admin` |
| Create a new user with multiple fields | Request body |
| Update a user's details | Request body |
| Paginate results | Query param — `/users?page=1&size=10` |

---

## A real example putting it together

```
GET /api/orders/42/items?status=pending&page=1
```

- `42` → path param, identifies which order
- `status=pending` → query param, filter items by status
- `page=1` → query param, pagination

In Spring Boot:

```java
@GetMapping("/orders/{orderId}/items")
public ResponseEntity<List<Item>> getOrderItems(
    @PathVariable Long orderId,
    @RequestParam(required = false) String status,
    @RequestParam(defaultValue = "1") int page) {
    return ResponseEntity.ok(orderService.getItems(orderId, status, page));
}
```

---

## Stuff I want to remember

**When do you use path params vs query params?**

Path params are for identifying a specific resource — like a user id or order id. Query params are for optional stuff like filters, search, sorting, pagination. Simple rule — if removing it makes the URL point to a different resource, it's a path param. If removing it just changes the results, it's a query param.

**Can you send a request body with GET?**

Technically the HTTP spec doesn't forbid it but in practice no — GET requests don't have a body. For filtering or searching you use query params. If you have complex filter requirements that don't fit in a URL, that's when some APIs use POST for search endpoints, though that's debatable.

**What happens if a required path param is missing?**

Spring Boot won't even match the route — you'll get a 404. Query params you can mark as `required = false` so they're optional, but path params are always required since they're part of the URL structure.

---

*Understood this better while building a leave management API — figuring out what goes in the URL vs the body made the endpoints much cleaner.*