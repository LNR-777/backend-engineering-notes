# HATEOAS

HATEOAS stands for Hypermedia As The Engine Of Application State. It's part of the Uniform Interface constraint we covered in the REST basics note — but it deserves its own note because almost nobody implements it, and that itself is a common interview point.

---

## The idea

A truly RESTful response shouldn't just return data — it should also tell the client what actions are possible next, through links.

Normal REST response:

```
{
  "id": 42,
  "name": "Rohit",
  "status": "ACTIVE"
}
```

HATEOAS response:

```
{
  "id": 42,
  "name": "Rohit",
  "status": "ACTIVE",
  "_links": {
    "self": { "href": "/api/users/42" },
    "orders": { "href": "/api/users/42/orders" },
    "deactivate": { "href": "/api/users/42/deactivate", "method": "POST" }
  }
}
```

The client doesn't need to hardcode URLs — it just follows the links the server gives it. Like clicking links on a website instead of typing URLs manually.

---

## Why it's rarely used

In theory it makes APIs more discoverable and decouples the client from hardcoded URL knowledge. In practice —

- Frontend teams already know the API structure, they don't need discovery
- Adds complexity to every response
- Most teams use API documentation (Swagger/OpenAPI) instead of relying on response links
- Mobile and SPA frontends are tightly coupled to the backend anyway, defeating the purpose

So while it's technically part of "true REST", almost no real-world API implements it fully. GitHub's API does it partially. Most others skip it.

---

## Implementing it in Spring Boot (if you ever need to)

Spring has a module for this — Spring HATEOAS.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-hateoas</artifactId>
</dependency>
```

```java
@GetMapping("/{id}")
public EntityModel<User> getUser(@PathVariable Long id) {
    User user = userService.findById(id);

    EntityModel<User> resource = EntityModel.of(user);
    resource.add(linkTo(methodOn(UserController.class).getUser(id)).withSelfRel());
    resource.add(linkTo(methodOn(OrderController.class).getOrdersByUser(id)).withRel("orders"));

    return resource;
}
```

This automatically generates the `_links` section in the response.

---

## Stuff I want to remember

**What is HATEOAS?**

It's a REST constraint where API responses include links to related actions, so the client doesn't need to hardcode URLs. It's part of the Uniform Interface principle in true REST.

**Why isn't HATEOAS commonly used?**

Most teams rely on API documentation like Swagger instead of runtime discoverability. It adds complexity to every response for a benefit most teams don't actually need — frontend developers already know the API structure from docs.

**Is an API still RESTful without HATEOAS?**

Technically no, by Roy Fielding's original definition. But in practice almost everyone calls their API RESTful even without HATEOAS — what people usually mean is stateless, resource based URLs, proper HTTP methods. Purists would call those APIs "REST-like" rather than fully RESTful.

---

*Learned about this mostly because interviewers ask "is your API truly RESTful" and HATEOAS is usually the gap. Good to know it exists even if I don't implement it in most projects.*