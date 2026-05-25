# Dependency Injection in Spring Boot

Dependency Injection is one of the core concepts of Spring. Once you get it, a lot of other Spring stuff starts making sense too.

The idea is simple — instead of a class creating its own dependencies, Spring creates them and hands them over. You just declare what you need.

---

## The problem without DI

```java
public class UserController {

    // creating dependency manually
    private UserService userService = new UserService();
}
```

This works but it's tightly coupled. If `UserService` changes its constructor, `UserController` breaks. Testing is also harder — you can't swap in a mock `UserService` without changing `UserController`.

---

## With Dependency Injection

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

`UserController` doesn't create `UserService` — Spring does. The controller just says "I need a UserService" and Spring provides it.

---

## Three ways to inject

### 1. Constructor Injection (recommended)

```java
@RestController
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }
}
```

This is the preferred way. Dependencies are set at construction time, the field can be `final`, and it makes testing straightforward — you can pass a mock directly through the constructor.

From Spring 4.3+ if there's only one constructor you don't even need `@Autowired` on it.

---

### 2. Field Injection

```java
@RestController
public class UserController {

    @Autowired
    private UserService userService;
}
```

Simpler to write but not recommended. The field can't be `final`, you can't easily write unit tests without Spring context, and it hides dependencies — you don't know what a class needs just by looking at its constructor.

A lot of tutorials use this because it's less code. In real projects constructor injection is better.

---

### 3. Setter Injection

```java
@RestController
public class UserController {

    private UserService userService;

    @Autowired
    public void setUserService(UserService userService) {
        this.userService = userService;
    }
}
```

Used when the dependency is optional or can change after construction. Rarely used in practice.

---

## How Spring knows what to inject

Spring scans for classes annotated with `@Component`, `@Service`, `@Repository`, `@Controller` and registers them as beans in the ApplicationContext.

```java
@Service
public class UserService {
    // Spring creates an instance of this and keeps it ready
}
```

When `UserController` needs a `UserService`, Spring looks in its container, finds the `UserService` bean, and injects it.

---

## What happens when there are multiple implementations

Say you have an interface `NotificationService` with two implementations — `EmailNotificationService` and `SmsNotificationService`.

```java
@Service
public class EmailNotificationService implements NotificationService { }

@Service
public class SmsNotificationService implements NotificationService { }
```

If you inject `NotificationService` Spring doesn't know which one to use and throws `NoUniqueBeanDefinitionException`.

Fix it with `@Primary` or `@Qualifier`:

```java
// option 1 — mark one as primary
@Primary
@Service
public class EmailNotificationService implements NotificationService { }
```

```java
// option 2 — specify which one you want
@Autowired
@Qualifier("smsNotificationService")
private NotificationService notificationService;
```

---

## Bean scope

By default Spring creates one instance of each bean and reuses it — this is called Singleton scope.

```java
@Service
// same as writing @Scope("singleton")
public class UserService { }
```

You can change it:

```java
@Service
@Scope("prototype") // new instance every time it's injected
public class UserService { }
```

Singleton is fine for stateless services which is most of the time. Prototype is for when each injection needs its own fresh instance.

---

## Stuff I want to remember

**Why is constructor injection better than field injection?**

With constructor injection dependencies are explicit — you can see exactly what a class needs just by looking at the constructor. The field can be `final` so it can't be changed accidentally. And unit testing is easy — just pass a mock through the constructor, no Spring context needed. Field injection hides dependencies and makes testing harder.

**What is the Spring IoC container?**

IoC stands for Inversion of Control. Normally your code creates objects — with IoC, the framework creates and manages them. The container holds all beans, handles their lifecycle, and injects them where needed. You give up control of object creation to Spring — that's the inversion.

**What happens if Spring can't find a bean to inject?**

It throws `NoSuchBeanDefinitionException` at startup. The app won't start. That's actually useful — you catch missing dependencies immediately, not at runtime when a specific endpoint is hit.

---

*Field injection was the first thing I learned because all tutorials use it. Switched to constructor injection after understanding why it's better — cleaner code and unit tests are much easier to write.*