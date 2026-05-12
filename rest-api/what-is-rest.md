# What is REST?

REST stands for Representational State Transfer. It's not a library or a framework — it's just a set of rules (constraints) for designing APIs.

Roy Fielding defined these rules in his PhD thesis back in 2000. A lot of APIs today call themselves "RESTful" but honestly most of them only follow 3-4 of the 6 constraints. Still, knowing all 6 is important — especially for interviews.

---

## The 6 Constraints

### 1. Client-Server

Client and server are separate. Client handles the UI, server handles the data. They talk to each other only through the API.

The benefit — you can completely change your frontend (say switch from a web app to a mobile app) without touching the backend at all, as long as the API stays the same.

---

### 2. Stateless

Every request must carry all the info the server needs. The server doesn't remember anything about you between requests.

This is why we send a JWT token with every request. The server doesn't have a session stored somewhere saying "oh this is user 42 who logged in 20 minutes ago." It just validates the token fresh each time.

```http
GET /api/orders/5
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

Remove that Authorization header and the server has no idea who you are.

---

### 3. Cacheable

Responses should indicate whether they can be cached or not. If something can be cached, the client doesn't need to hit the server again for the same data.

```http
HTTP/1.1 200 OK
Cache-Control: max-age=3600
```

This tells the client — this response is good for 1 hour, don't bother me again until then. Good for performance.

---

### 4. Uniform Interface

This is the main one. All REST APIs follow the same consistent style —

- Resources are identified by URIs (`/users/42` not `/getUser?id=42`)
- You use standard HTTP methods (GET, POST, PUT, PATCH, DELETE)
- Responses are representations of the resource (usually JSON)

```
// RESTful
GET /api/users/42

// Not RESTful — this is RPC style
GET /api/getUser?id=42
```

---

### 5. Layered System

The client doesn't know (and doesn't care) what's sitting between it and the actual server. Could be a load balancer, an API gateway, a cache layer — doesn't matter.

```
Client → API Gateway → Load Balancer → Spring Boot App → MySQL
```

Client just hits the API URL. Everything behind it is invisible.

---

### 6. Code on Demand (optional)

Server can send executable code to the client, like JavaScript. This one is optional and most REST APIs don't use it at all.

---

## Quick Recap

| Constraint | What it means |
|---|---|
| Client-Server | UI and backend are separate |
| Stateless | Every request is self-contained |
| Cacheable | Responses say if they can be stored |
| Uniform Interface | Consistent URLs and HTTP methods |
| Layered System | Client doesn't know about backend layers |
| Code on Demand | Server can send executable code (optional) |

---

## Stuff I want to remember

**What is REST in simple words?**

It's an architectural style — not a protocol, not a library. Just rules for how APIs should behave. The big ones are: stateless communication, resource-based URLs, and standard HTTP methods. That's what most people mean when they say RESTful.

---

**What does stateless actually mean?**

The server doesn't remember you between requests. Every time you call an API you have to carry your own context — that's why JWT exists. The server just reads the token, figures out who you are, and processes the request. No stored sessions anywhere. Makes scaling much easier because any server can handle any request.

---

**Do all REST APIs really follow all 6 constraints?**

No, most don't. HATEOAS is almost never implemented in real projects. Code on Demand is optional anyway. In practice when someone says their API is RESTful they usually just mean — stateless, resource URLs, proper HTTP methods. Technically they should call it an HTTP API but nobody really does that.

---

*Notes from my backend learning journey — written to understand, not to copy.*