# How Spring Boot Works — What Happens When You Run the App

When you hit run on a Spring Boot app, a lot happens in a few seconds. Most people just wait for "Started Application in X seconds" and move on. But understanding what's actually happening helps a lot — especially when things break or when an interviewer asks.

---

## The entry point

Every Spring Boot app starts here:

```java
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

`SpringApplication.run()` kicks everything off.

---

## What @SpringBootApplication actually is

It's a combination of three annotations:

```java
@SpringBootConfiguration   // marks this as a configuration class
@EnableAutoConfiguration   // turns on auto-configuration
@ComponentScan             // scans for components in this package and sub-packages
```

You could write all three separately — `@SpringBootApplication` is just a shortcut.

---

## Startup sequence — what happens step by step

### 1. JVM starts, main() runs
Spring Boot creates a `SpringApplication` instance and calls `run()`.

### 2. ApplicationContext is created
This is the heart of Spring — the IoC container. It manages all your beans (objects). Spring Boot decides which type of context to create based on what's on the classpath. If it finds Spring MVC, it creates a `AnnotationConfigServletWebServerApplicationContext`.

### 3. @ComponentScan runs
Spring scans your package and all sub-packages looking for classes annotated with `@Component`, `@Service`, `@Repository`, `@Controller`, `@RestController`. Every class it finds becomes a bean.

```java
@Service
public class UserService { }  // Spring finds this and creates a bean
```

### 4. Auto-configuration kicks in
This is the magic part. Spring Boot looks at what's on your classpath and automatically configures things.

- Found `spring-boot-starter-web`? Automatically sets up an embedded Tomcat server, Spring MVC, etc.
- Found `spring-boot-starter-data-jpa`? Automatically configures a datasource, EntityManager, transaction management.
- Found `spring-boot-starter-security`? Automatically adds security filter chain.

You can see what got auto-configured by adding `--debug` when running the app. It prints a conditions report.

### 5. Dependencies are injected
Spring wires up all the beans. If `UserController` needs `UserService`, Spring injects it automatically.

```java
@RestController
public class UserController {

    private final UserService userService;

    // Spring injects UserService here
    public UserController(UserService userService) {
        this.userService = userService;
    }
}
```

### 6. Embedded server starts
Spring Boot starts an embedded Tomcat (or Jetty/Undertow) on port 8080 by default. No need to deploy a WAR file — the server is inside the app.

### 7. Application is ready
You see `Started MyApplication in 2.345 seconds` — app is ready to handle requests.

---

## Embedded server — why it matters

Traditional Java web apps needed an external Tomcat server. You'd build a WAR file and deploy it. Spring Boot embeds the server inside the JAR — so you just run `java -jar myapp.jar` and it works anywhere. This is why Spring Boot apps are easy to containerize with Docker.

---

## application.properties

Spring Boot reads this file during startup to configure things:

```properties
server.port=8081
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=root
spring.datasource.password=password
spring.jpa.hibernate.ddl-auto=update
```

You can override any auto-configuration value here.

---

## Stuff I want to remember

**What does @SpringBootApplication do?**

It's three annotations in one — `@SpringBootConfiguration`, `@EnableAutoConfiguration`, and `@ComponentScan`. The important ones are EnableAutoConfiguration which sets up everything automatically based on classpath, and ComponentScan which tells Spring where to look for beans.

**What is the Spring IoC container?**

IoC stands for Inversion of Control. Instead of you creating objects with `new`, Spring creates and manages them for you. The container holds all these objects (beans) and injects them wherever needed. You just declare what you need, Spring figures out how to provide it.

**What's the difference between @Component, @Service, and @Repository?**

All three register a class as a Spring bean — functionally they do the same thing. The difference is semantic. `@Component` is generic. `@Service` marks business logic layer. `@Repository` marks data access layer and also adds some persistence-specific exception translation. Using the right one makes the code easier to read and understand.

---

*The auto-configuration part took me a while to understand. Once I realized Spring Boot is just looking at your classpath and saying "oh you have JPA? let me set that up for you" — it all made sense.*