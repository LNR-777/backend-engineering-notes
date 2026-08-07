# Testing REST APIs with Postman

Postman is the tool I use most when building and testing APIs. It's way faster than writing test code just to check if an endpoint works — you can hit your API, see the full request/response, and debug issues in seconds.

---

## Basic setup

After downloading Postman, create a Collection to group all your API requests together.

```
backend-engineering-notes/
└── My API Collection/
    ├── Auth/
    │   ├── Login
    │   └── Refresh Token
    ├── Users/
    │   ├── Get All Users
    │   ├── Get User by ID
    │   ├── Create User
    │   └── Delete User
    └── Products/
        └── ...
```

Keeping requests in folders makes it easy to find them later.

---

## Making requests

**GET request:**

```
Method: GET
URL: http://localhost:8080/api/users/42
```

**POST with JSON body:**

```
Method: POST
URL: http://localhost:8080/api/users
Headers: Content-Type: application/json
Body (raw, JSON):
{
  "name": "Rohit",
  "email": "rohit@gmail.com",
  "password": "password123"
}
```

**With Authorization header:**

```
Headers:
  Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

---

## Environment Variables — stop hardcoding URLs

Instead of typing `http://localhost:8080` in every request, use environments.

Create a `Local` environment with:

```
baseUrl = http://localhost:8080
token = (empty, will fill after login)
```

Then use it in your requests:

```
URL: {{baseUrl}}/api/users
Authorization: Bearer {{token}}
```

When you switch from local to dev server, just change the `baseUrl` value in the environment — all requests update automatically.

---

## Auto-setting the token after login

Write a test script in Postman that automatically saves the token after login — so you don't have to copy-paste it manually every time.

In your Login request, go to the **Tests** tab and add:

```javascript
const response = pm.response.json();
pm.environment.set("token", response.token);
```

Now after running the login request, `{{token}}` is automatically set in the environment. All subsequent requests that use `{{token}}` will have the right value.

---

## Writing Tests in Postman

Postman has a built-in test runner. In the **Tests** tab of any request:

```javascript
// check status code
pm.test("Status is 200", function() {
    pm.response.to.have.status(200);
});

// check response has a field
pm.test("Response has user id", function() {
    const body = pm.response.json();
    pm.expect(body.id).to.exist;
});

// check specific value
pm.test("User name is correct", function() {
    const body = pm.response.json();
    pm.expect(body.name).to.equal("Rohit");
});
```

Green = pass, Red = fail. Run all tests together using **Collection Runner**.

---

## Viewing the full request and response

Postman shows you everything — which is super useful for debugging:

- **Response Body** — the actual JSON/text returned
- **Response Headers** — Content-Type, status, Location, etc.
- **Status Code** — right at the top
- **Response Time** — how long the API took
- **Console (bottom left)** — shows the actual HTTP request that went out

When something is wrong, check the console — it shows exactly what was sent.

---

## Common things to test for every endpoint

For every endpoint I build, I test:

**Happy path:**
- Correct data → expected response and status code
- Response body has the right fields

**Error cases:**
- Missing required fields → 400 with validation message
- Invalid ID → 404
- No auth token → 401
- Wrong role → 403
- Duplicate data → 409

---

## Postman for JWT flow

Step by step for testing a JWT secured API:

```
1. POST /auth/login → get access token
   (test script auto-saves token to environment)

2. GET /api/users → Authorization: Bearer {{token}}
   → should return 200

3. GET /api/users with no token
   → should return 401

4. GET /api/admin/dashboard with user token (non-admin)
   → should return 403
```

Testing all three — valid token, no token, wrong role — covers the main auth scenarios.

---

## Stuff I want to remember

**Why use environment variables in Postman?**

So you don't hardcode URLs and tokens in every request. When your base URL or token changes, you update it in one place — the environment — and all requests automatically use the new value. Also makes switching between local and production environments easy.

**What is the difference between Params and Body in Postman?**

Params adds query parameters to the URL (`?key=value`). Body is the request body — used for POST, PUT, PATCH. You'd use Params for filtering/pagination and Body for sending data to create or update something.

**How do you test that an endpoint properly returns 401 for unauthenticated requests?**

Remove the Authorization header from the request in Postman and send it. If it still returns 200, your security config isn't protecting that endpoint. Should return 401 with no token, 403 with a valid token but wrong role.

---

*Postman saved me hours of debugging. Being able to see the exact request that goes out and the full response — headers included — makes it so much easier to spot what's wrong compared to reading logs alone.*