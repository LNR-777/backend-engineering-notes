# @Transactional in Spring Boot

`@Transactional` is one of those annotations you add without thinking much about it early on. But understanding what it actually does under the hood matters — especially when things go wrong.

---

## What it does

When you mark a method with `@Transactional`, Spring wraps it in a database transaction. If everything inside succeeds — changes are committed. If an exception is thrown — everything is rolled back. All or nothing.

```java
@Transactional
public void transferMoney(Long fromId, Long toId, Double amount) {
    accountService.debit(fromId, amount);   // deduct from sender
    accountService.credit(toId, amount);    // add to receiver
}
```

If `credit()` fails after `debit()` already ran — without `@Transactional` the money is gone from one account and never reached the other. With `@Transactional` the whole thing rolls back. Neither account changes.

---

## Where to put it

On the service layer, not the controller or repository.

```java
@Service
public class PaymentService {

    @Transactional
    public void processPayment(PaymentRequest request) {
        // multiple DB operations here
    }
}
```

Repository methods already have transactions by default in Spring Data JPA. The service layer is where you coordinate multiple operations that should be treated as one unit.

---

## Rollback behavior

By default `@Transactional` only rolls back on unchecked exceptions (RuntimeException and its subclasses). Checked exceptions don't trigger a rollback by default.

```java
@Transactional
public void saveUser(User user) throws IOException {
    userRepository.save(user);
    // if IOException is thrown here — NO rollback by default
}
```

To rollback on checked exceptions:

```java
@Transactional(rollbackFor = Exception.class)
public void saveUser(User user) throws IOException {
    userRepository.save(user);
    // now IOException triggers rollback too
}
```

Or rollback on a specific exception:

```
@Transactional(rollbackFor = CustomException.class)
```

---

## Propagation

What happens when a `@Transactional` method calls another `@Transactional` method?

```java
@Transactional
public void methodA() {
    methodB(); // methodB is also @Transactional
}
```

This is controlled by propagation. Default is `REQUIRED` — if a transaction already exists, join it. If not, create a new one.

Common propagation types:

```
// joins existing transaction or creates new one (default)
@Transactional(propagation = Propagation.REQUIRED)

// always creates a new transaction, suspends existing one
@Transactional(propagation = Propagation.REQUIRES_NEW)

// runs without a transaction even if one exists
@Transactional(propagation = Propagation.NOT_SUPPORTED)
```

`REQUIRES_NEW` is useful when you want a method to commit independently — like logging. Even if the main transaction rolls back, the log entry should stay.

---

## Read-only transactions

For queries that don't modify data, mark the transaction as read-only:

```java
@Transactional(readOnly = true)
public List<User> getAllUsers() {
    return userRepository.findAll();
}
```

This gives the database a hint to optimize the query. No dirty checking, no write locks. Small performance improvement, especially for heavy read operations.

---

## Common mistake — @Transactional on private methods

```java
@Service
public class UserService {

    @Transactional
    private void saveUser(User user) { // won't work
        userRepository.save(user);
    }
}
```

Spring uses a proxy to wrap the method in a transaction. Proxies can only intercept public methods. `@Transactional` on a private method is silently ignored — no error, no transaction either. Always put it on public methods.

---

## Another common mistake — calling @Transactional method from within the same class

```java
@Service
public class UserService {

    public void doSomething() {
        this.saveUser(user); // calling own method directly
    }

    @Transactional
    public void saveUser(User user) {
        userRepository.save(user);
    }
}
```

When `doSomething()` calls `saveUser()` directly on `this`, it bypasses the Spring proxy. The `@Transactional` on `saveUser()` is ignored. This is one of the trickiest bugs to spot — no error, just no transaction.

Fix — inject the service into itself or restructure the code so the `@Transactional` method is called from outside the class.

---

## Stuff I want to remember

**What does @Transactional actually do under the hood?**

Spring creates a proxy around the class. When the method is called, the proxy opens a database transaction, runs the method, and commits if successful or rolls back if an exception is thrown. The actual class code doesn't change — the proxy handles the transaction management around it.

**Why doesn't @Transactional work on private methods?**

Spring's proxy can only intercept public method calls from outside the class. Private methods are called directly, not through the proxy, so the transaction logic never runs. No error is thrown — it just silently does nothing.

**When would you use REQUIRES_NEW propagation?**

When you need a method to commit independently of the calling transaction. Most common use case is audit logging or error logging — you want the log entry saved even if the main operation fails and rolls back.

---

*The private method issue got me once — spent an hour wondering why my transaction wasn't rolling back. Turned out the method was private. Adding public fixed it immediately.*