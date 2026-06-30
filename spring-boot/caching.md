# Caching in Spring Boot

If your app keeps fetching the same data from the database repeatedly, caching can save a lot of unnecessary DB hits. Spring Boot makes this easy with annotation based caching.

---

## Why cache

Say you have an endpoint that fetches a product by ID. If the same product gets requested 1000 times in a minute, you're hitting the database 1000 times for data that hasn't changed. Caching stores the result after the first call and serves it from memory for subsequent calls.

---

## Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

Enable caching in your main application class or a config class:

```java
@SpringBootApplication
@EnableCaching
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

---

## @Cacheable — cache the result of a method

```java
@Service
public class ProductService {

    @Cacheable("products")
    public Product getProductById(Long id) {
        System.out.println("Fetching from DB for id: " + id);
        return productRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Product not found"));
    }
}
```

First call — prints "Fetching from DB", actually hits the database.
Second call with the same id — no print, returns directly from cache.

The cache key is based on the method parameters by default. Different `id` values get cached separately.

---

## @CachePut — update the cache without skipping the method

```java
@CachePut(value = "products", key = "#product.id")
public Product updateProduct(Product product) {
    return productRepository.save(product);
}
```

Unlike `@Cacheable`, this always runs the method — but it also updates the cache with the new result. Use this for update operations where you need the cache to reflect the latest data.

---

## @CacheEvict — remove from cache

```java
@CacheEvict(value = "products", key = "#id")
public void deleteProduct(Long id) {
    productRepository.deleteById(id);
}
```

When a product is deleted, remove it from cache too — otherwise stale data stays cached and gets served on the next read.

You can also clear the entire cache:

```java
@CacheEvict(value = "products", allEntries = true)
public void clearAllProducts() {
    // clears the whole "products" cache
}
```

---

## Custom cache key

By default Spring uses the method parameters as the key. You can customize it:

```java
@Cacheable(value = "products", key = "#category + '-' + #page")
public List<Product> getProductsByCategory(String category, int page) {
    return productRepository.findByCategory(category, page);
}
```

---

## Conditional caching

Only cache under certain conditions:

```java
@Cacheable(value = "products", condition = "#id > 0")
public Product getProductById(Long id) {
    return productRepository.findById(id).orElseThrow();
}
```

Or skip caching the result based on what's returned:

```java
@Cacheable(value = "products", unless = "#result == null")
public Product getProductById(Long id) {
    return productRepository.findById(id).orElse(null);
}
```

---

## Cache providers

By default Spring Boot uses a simple in-memory `ConcurrentHashMap` based cache — fine for development, not great for production.

For production, use something like Redis:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

```properties
spring.cache.type=redis
spring.redis.host=localhost
spring.redis.port=6379
```

With Redis, cache survives application restarts and can be shared across multiple instances of your app — which the default in-memory cache can't do.

---

## When NOT to cache

- Data that changes frequently — cache becomes stale quickly, defeats the purpose
- Data specific to a single user that's rarely re-requested
- Small datasets where the DB query is already fast — caching adds complexity for no real benefit

---

## Stuff I want to remember

**What's the difference between @Cacheable and @CachePut?**

`@Cacheable` skips the method entirely if the value is already in cache — returns cached result directly. `@CachePut` always executes the method and updates the cache with the new result. Use `@Cacheable` for reads, `@CachePut` for updates where you need the cache refreshed.

**Why use Redis instead of the default in-memory cache?**

The default cache is just a HashMap inside the application — it's lost on restart and isn't shared if you run multiple instances of your app (common in production with load balancers). Redis is external, persists across restarts, and all instances of your app can share the same cache.

**What happens if you forget @CacheEvict on a delete operation?**

The deleted record stays in cache. Next read for that id returns the cached (stale) data instead of a proper "not found" response. That's a subtle bug — the API would appear to work but return wrong data.

---

*Added caching to a product listing endpoint once and the response time dropped from 200ms to under 5ms on cached requests. That's when caching actually clicked for me — it's not just theory, it's a real performance lever.*