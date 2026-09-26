# RestTemplate vs Feign Client

When your Spring Boot service needs to call another service or external API, you have a few options. RestTemplate and Feign are the two most common. This note covers both and when to use which.

---

## RestTemplate

The classic way to make HTTP calls from Spring Boot. It's been around since Spring 3 and works well for simple use cases.

### Basic GET request

```java
@Service
public class ProductService {

    private final RestTemplate restTemplate;

    public ProductService(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    public Product getProduct(Long id) {
        String url = "http://product-service/api/products/" + id;
        return restTemplate.getForObject(url, Product.class);
    }
}
```

Register as a bean:

```java
@Bean
public RestTemplate restTemplate() {
    return new RestTemplate();
}
```

### POST request

```java
public Order createOrder(OrderRequest request) {
    String url = "http://order-service/api/orders";
    return restTemplate.postForObject(url, request, Order.class);
}
```

### With headers

```java
public User getUserWithAuth(Long id, String token) {
    HttpHeaders headers = new HttpHeaders();
    headers.setBearerAuth(token);
    headers.setContentType(MediaType.APPLICATION_JSON);

    HttpEntity<Void> entity = new HttpEntity<>(headers);

    ResponseEntity<User> response = restTemplate.exchange(
        "http://user-service/api/users/" + id,
        HttpMethod.GET,
        entity,
        User.class
    );

    return response.getBody();
}
```

---

## Feign Client

Feign is a declarative HTTP client — you define an interface and annotate it, Feign generates the implementation. Much cleaner than RestTemplate for complex APIs.

### Setup

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

Enable in main class:

```java
@SpringBootApplication
@EnableFeignClients
public class MyApplication { }
```

### Define the client

```java
@FeignClient(name = "product-service", url = "http://localhost:8081")
public interface ProductClient {

    @GetMapping("/api/products/{id}")
    Product getProduct(@PathVariable Long id);

    @PostMapping("/api/products")
    Product createProduct(@RequestBody ProductRequest request);

    @GetMapping("/api/products")
    List<Product> getAllProducts(@RequestParam String category);
}
```

### Use it

```java
@Service
public class OrderService {

    @Autowired
    private ProductClient productClient;

    public void processOrder(Long productId) {
        Product product = productClient.getProduct(productId);
        // use product
    }
}
```

That's it. No URL building, no response entity handling, no manual header setting for basic cases. Just call the method.

---

## Feign with headers

```java
@FeignClient(name = "user-service", url = "http://localhost:8082",
             configuration = FeignConfig.class)
public interface UserClient {

    @GetMapping("/api/users/{id}")
    User getUser(@PathVariable Long id);
}
```

```java
@Configuration
public class FeignConfig {

    @Bean
    public RequestInterceptor requestInterceptor() {
        return requestTemplate -> {
            requestTemplate.header("Authorization", "Bearer " + getToken());
        };
    }
}
```

---

## Error handling with Feign

```java
@Component
public class FeignErrorDecoder implements ErrorDecoder {

    @Override
    public Exception decode(String methodKey, Response response) {
        return switch (response.status()) {
            case 404 -> new ResourceNotFoundException("Resource not found");
            case 400 -> new BadRequestException("Bad request to external service");
            default -> new Exception("External service error: " + response.status());
        };
    }
}
```

---

## RestTemplate vs Feign — comparison

| | RestTemplate | Feign |
|---|---|---|
| Style | Imperative — write the call yourself | Declarative — define interface, Feign does the rest |
| Code amount | More verbose | Much less code |
| Readability | Lower for complex calls | High — reads like a local method call |
| Error handling | Manual | Configurable via ErrorDecoder |
| Spring Cloud integration | Needs extra config | First-class support |
| Status | Being deprecated (use WebClient) | Actively maintained |

---

## WebClient — the modern alternative

RestTemplate is in maintenance mode — Spring recommends `WebClient` (from Spring WebFlux) for new projects. It supports both sync and async calls.

```java
WebClient webClient = WebClient.create("http://product-service");

Product product = webClient.get()
    .uri("/api/products/{id}", id)
    .retrieve()
    .bodyToMono(Product.class)
    .block(); // .block() makes it synchronous
```

For microservices with high throughput, WebClient's async/reactive nature gives better performance. For simple services, Feign is still cleaner to write.

---

## Stuff I want to remember

**What is the difference between RestTemplate and Feign?**

RestTemplate is imperative — you write the HTTP call yourself, build the URL, set headers, handle the response. Feign is declarative — you define an interface with annotations describing the API you want to call, and Feign generates the HTTP client automatically. Feign results in much cleaner, more readable code especially when calling multiple endpoints.

**Why is RestTemplate being deprecated?**

Spring is moving towards reactive programming with WebFlux. RestTemplate is blocking — it holds a thread for the duration of each HTTP call. WebClient is non-blocking and can handle more concurrent requests with fewer threads. For new projects Spring recommends WebClient over RestTemplate.

**When would you use Feign over WebClient?**

Feign is simpler and more readable — great when you're in a synchronous Spring MVC app and don't need reactive capabilities. WebClient is better when you need async/non-blocking calls or are building a reactive application. In a standard Spring Boot REST API with moderate load, Feign is usually the pragmatic choice.

---

*Used RestTemplate during my internship before learning about Feign. After switching to Feign for service-to-service calls, the code went from 15 lines per call to 1 method on an interface. That was a clear win.*