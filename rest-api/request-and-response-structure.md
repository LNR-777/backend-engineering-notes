# Request and Response Structure

Every API call has two sides — the request (what the client sends) and the response (what the server sends back). Understanding the structure of both is fundamental.

---

## HTTP Request Structure

A request has four parts:

```
1. Request Line  — method + URL + HTTP version
2. Headers       — metadata about the request
3. Blank Line    — separates headers from body
4. Body          — actual data (only for POST, PUT, PATCH)
```

Full example:

```
POST /api/users HTTP/1.1
Host: api.myapp.com
Content-Type: application/json
Authorization: Bearer eyJhbGci...
Accept: application/json

{
  "name": "Rohit",
  "email": "rohit@gmail.com"
}
```

---

## Common Request Headers

| Header | What it does |
|---|---|
| `Content-Type` | Tells server what format the body is in |
| `Accept` | Tells server what format the client wants back |
| `Authorization` | Carries the auth token |
| `Content-Length` | Size of the request body in bytes |

`Content-Type: application/json` — most common for REST APIs.
`Content-Type: multipart/form-data` — for file uploads.

---

## HTTP Response Structure

```
1. Status Line   — HTTP version + status code + reason phrase
2. Headers       — metadata about the response
3. Blank Line    — separates headers from body
4. Body          — actual data being returned
```

Full example:

```
HTTP/1.1 201 Created
Content-Type: application/json
Location: /api/users/42
Date: Mon, 15 Jan 2024 10:30:00 GMT

{
  "id": 42,
  "name": "Rohit",
  "email": "rohit@gmail.com"
}
```

---

## Common Response Headers

| Header | What it does |
|---|---|
| `Content-Type` | Format of the response body |
| `Location` | URL of newly created resource (used with 201) |
| `Cache-Control` | Caching instructions |
| `WWW-Authenticate` | Tells client how to authenticate (with 401) |

---

## How this looks in Spring Boot

``java
@PostMapping("/users")
public ResponseEntity<User> createUser(@RequestBody UserRequest request) {
    User saved = userService.save(request);
    
    URI location = ServletUriComponentsBuilder
        .fromCurrentRequest()
        .path("/{id}")
        .buildAndExpand(saved.getId())
        .toUri();
    
    return ResponseEntity.created(location).body(saved);
}
```

This sets the status to 201 Created and automatically adds the `Location` header pointing to the new resource.

---

## @RequestBody vs @RequestParam vs @PathVariable

Three ways to read data from a request in Spring Boot:

```java
// reads from the URL path
// GET /api/users/42
@GetMapping("/users/{id}")
public ResponseEntity<User> getUser(@PathVariable Long id) { }

// reads from query params
// GET /api/users?role=admin
@GetMapping("/users")
public ResponseEntity<List<User>> getUsers(@RequestParam String role) { }

// reads from the request body
// POST /api/users with JSON body
@PostMapping("/users")
public ResponseEntity<User> createUser(@RequestBody UserRequest request) { }
```

---

## ResponseEntity — full control over the response

`ResponseEntity` lets you control everything — status code, headers, body.

```java
// just status and body
return ResponseEntity.ok(user);                          // 200
return ResponseEntity.status(HttpStatus.CREATED).body(user); // 201
return ResponseEntity.noContent().build();               // 204

// with custom headers
HttpHeaders headers = new HttpHeaders();
headers.add("X-Custom-Header", "some-value");
return ResponseEntity.ok().headers(headers).body(user);
```

---

## What good JSON responses look like

Keep response structure consistent. Wrap data in a standard envelope:

```
{
  "success": true,
  "data": {
    "id": 42,
    "name": "Rohit",
    "email": "rohit@gmail.com"
  }
}
```

Error response:

```
{
  "success": false,
  "status": 404,
  "message": "User not found with id: 42",
  "timestamp": "2024-01-15T10:30:00"
}
```

Frontend developers will love you for consistency.

---

## Stuff I want to remember

**What is the difference between Content-Type and Accept headers?**

`Content-Type` describes what format you're sending in the request body. `Accept` describes what format you want back in the response. So `Content-Type: application/json` means "I'm sending JSON", `Accept: application/json` means "I want JSON back."

**What is the Location header used for?**

It's returned in 201 Created responses to tell the client where the newly created resource lives — the URL to GET it. Spring's `ResponseEntity.created(uri)` sets this automatically.

**When should you use ResponseEntity vs just returning the object?**

Return the object directly when 200 is always the right status and you don't need custom headers. Use `ResponseEntity` when you need to control the status code (like 201 for create, 204 for delete) or add custom response headers.

---

*Understanding request/response structure helped a lot when debugging API issues in Postman — knowing exactly where to look for the status, headers and body instead of just staring at the response.*