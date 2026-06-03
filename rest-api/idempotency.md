# Idempotency in REST APIs

Idempotency means calling the same operation multiple times gives the same result as calling it once. The state of the server doesn't change after the first call.

This matters more than most people think — especially when networks are unreliable and clients need to retry requests.

---

## Why it matters

Say a client sends a payment request. Network drops. Client doesn't know if the request went through or not. What does it do?

If the endpoint is idempotent — retry safely. Same result.
If the endpoint is not idempotent — retrying might create a duplicate payment. Problem.

---

## Which HTTP methods are idempotent

- **GET** — always idempotent. Just reading data, nothing changes.
- **PUT** — idempotent. Sending the same data replaces the resource with the same thing each time.
- **DELETE** — idempotent. First call deletes the resource. Second call — resource is already gone, state doesn't change.
- **POST** — NOT idempotent. Each call creates a new resource.
- **PATCH** — depends on implementation. Usually not idempotent.

```
DELETE /api/users/42  →  user deleted, 204
DELETE /api/users/42  →  user still gone, 404 (but state is same)
```

The server might return different status codes on repeated calls but the end state is the same. That's what makes it idempotent.

---

## Making POST idempotent with Idempotency Keys

POST creates a new resource every time — but sometimes you need a POST to be safe to retry. Common solution is an idempotency key.

Client generates a unique key (usually a UUID) and sends it with the request:

```http
POST /api/payments
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json

{
  "amount": 5000,
  "to": "acc_123"
}
```

Server stores this key after processing. If the same key comes again — server returns the cached response instead of processing again.

In Spring Boot:

```java
@PostMapping("/payments")
public ResponseEntity<PaymentResponse> createPayment(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody PaymentRequest request) {

    // check if we already processed this key
    Optional<PaymentResponse> cached = idempotencyStore.get(idempotencyKey);
    if (cached.isPresent()) {
        return ResponseEntity.ok(cached.get());
    }

    // process payment
    PaymentResponse response = paymentService.process(request);

    // store result against the key
    idempotencyStore.save(idempotencyKey, response);

    return ResponseEntity.status(HttpStatus.CREATED).body(response);
}
```

Stripe uses this exact pattern for their payment API.

---

## Idempotency vs Safety

Two different concepts that people mix up:

- **Safe** — the operation doesn't modify server state. GET is safe.
- **Idempotent** — repeated calls have the same effect. GET, PUT, DELETE are idempotent.

DELETE is idempotent but not safe — it does modify state (deletes something), just not on repeated calls.
POST is neither safe nor idempotent.

---

## Real world example — why this matters in microservices

In a microservices setup, service A calls service B. Network timeout. Service A doesn't know if B processed the request. Service A retries.

If service B's endpoint is not idempotent — duplicate records, duplicate emails sent, duplicate charges. Real damage.

Good microservice APIs are designed with idempotency in mind from the start — especially for anything involving money, notifications, or state changes.

---

## Stuff I want to remember

**What is idempotency in simple terms?**

Call it once or call it 10 times — the result is the same. The server ends up in the same state regardless of how many times the request was made. GET, PUT and DELETE are idempotent. POST is not.

**How do you make a POST endpoint idempotent?**

Use an idempotency key — client generates a unique ID and sends it in the header. Server processes the request, stores the result against that key. If the same key comes again, server returns the stored result without processing again. Stripe does this for payments.

**Why is DELETE considered idempotent even though it returns 404 on second call?**

Because idempotency is about the state of the server, not the response code. After first DELETE the resource is gone. After second DELETE the resource is still gone — state didn't change. The 404 is just the server telling you it couldn't find what you wanted to delete, but the outcome is the same.

---

*This clicked when I thought about it from the client's perspective — if my request fails and I need to retry, will retrying cause problems? If yes, the endpoint is not idempotent and needs to be designed more carefully.*