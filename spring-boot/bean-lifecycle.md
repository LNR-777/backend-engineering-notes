# Bean Lifecycle in Spring Boot

Every bean in Spring goes through a lifecycle — from creation to destruction. Understanding this helps when you need to run code at specific points, like setting up a connection after a bean is created or cleaning up resources before it's destroyed.

---

## The lifecycle in order

```
1. Bean instantiation — Spring creates the object
2. Dependency injection — dependencies are injected
3. @PostConstruct — your custom initialization code runs
4. Bean is ready — used by the application
5. @PreDestroy — your custom cleanup code runs
6. Bean is destroyed — application shuts down
```

---

## @PostConstruct — run code after bean is created

```java
@Service
public class CacheService {

    private Map<String, String> cache;

    @PostConstruct
    public void init() {
        // runs automatically after this bean is created and dependencies are injected
        cache = new HashMap<>();
        loadInitialData();
        System.out.println("CacheService initialized");
    }

    private void loadInitialData() {
        // load some data into cache on startup
        cache.put("config_key", "config_value");
    }
}
```

Common use cases — initializing a cache, loading config from DB, setting up a connection pool, validating configuration values.

---

## @PreDestroy — run code before bean is destroyed

```java
@Service
public class CacheService {

    @PreDestroy
    public void cleanup() {
        // runs before the application shuts down
        cache.clear();
        System.out.println("CacheService cleanup done");
    }
}
```

Common use cases — closing connections, flushing caches, releasing resources, saving state before shutdown.

---

## InitializingBean and DisposableBean interfaces

Alternative to `@PostConstruct` and `@PreDestroy`. Less common but you might see it in older codebases.

```java
@Service
public class DatabaseService implements InitializingBean, DisposableBean {

    @Override
    public void afterPropertiesSet() throws Exception {
        // same as @PostConstruct
        System.out.println("DatabaseService initialized");
    }

    @Override
    public void destroy() throws Exception {
        // same as @PreDestroy
        System.out.println("DatabaseService destroyed");
    }
}
```

`@PostConstruct` and `@PreDestroy` are cleaner and more readable — use those unless you have a specific reason not to.

---

## @Bean with initMethod and destroyMethod

When you define beans manually in a `@Configuration` class:

```java
@Configuration
public class AppConfig {

    @Bean(initMethod = "init", destroyMethod = "cleanup")
    public SomeService someService() {
        return new SomeService();
    }
}
```

Spring calls `init()` after creation and `cleanup()` before destruction — same lifecycle, just configured differently.

---

## Bean scope and lifecycle

Lifecycle behaves differently based on scope:

- **Singleton** (default) — one instance for the whole app. Created at startup, destroyed at shutdown. `@PostConstruct` and `@PreDestroy` both work.

- **Prototype** — new instance every time it's requested. `@PostConstruct` runs on each new instance. `@PreDestroy` does NOT run — Spring doesn't track prototype beans after creation. You're responsible for cleanup.

```java
@Service
@Scope("prototype")
public class ReportGenerator {

    @PostConstruct
    public void init() {
        // runs every time a new instance is created
    }

    @PreDestroy
    public void cleanup() {
        // WARNING — this won't be called for prototype beans
    }
}
```

---

## ApplicationListener — hook into application events

If you want to run code when the application fully starts up (after all beans are ready):

```
@Component
public class StartupListener implements ApplicationListener<ApplicationReadyEvent> {

    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        System.out.println("Application is fully started and ready");
        // good place for tasks that need the full context to be ready
    }
}
```

Difference from `@PostConstruct` — `@PostConstruct` runs when that specific bean is initialized. `ApplicationReadyEvent` fires when the entire application context is ready.

---

## Stuff I want to remember

**What is the difference between @PostConstruct and ApplicationReadyEvent?**

`@PostConstruct` runs when that specific bean is created, during application startup — before all beans are ready. `ApplicationReadyEvent` fires after the entire application context is up and ready to serve requests. If your initialization depends on other beans being fully set up, `ApplicationReadyEvent` is safer.

**Why doesn't @PreDestroy work for prototype beans?**

Spring doesn't keep track of prototype beans after handing them out. It creates them and gives them away — after that they're on their own. Since Spring doesn't hold a reference, it can't call `@PreDestroy` when shutting down. Singleton beans Spring tracks from creation to destruction.

**When would you use @PostConstruct?**

Whenever you need to run initialization logic that requires dependencies to be injected first. Constructor runs before injection, so you can't use dependencies there if they're field-injected. `@PostConstruct` runs after injection, so all dependencies are available.

---

*Learned about @PostConstruct when I needed to load some config values from DB into memory when the app starts. Without it I wasn't sure where to put that initialization code.*