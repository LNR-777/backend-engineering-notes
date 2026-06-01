# Pagination in REST APIs

When your database has thousands of records you don't want to return all of them in one response. That's where pagination comes in — you return data in chunks.

Two main approaches — offset based and cursor based.

---

## Why pagination matters

Without it —

// json

GET /api/products

// returns 50,000 records in one shot
```

That kills your server, kills the network, and the frontend has no idea what to do with 50k records. Pagination solves this.

---

## Offset Based Pagination

Most common approach. You specify which page you want and how many records per page.

```
GET /api/products?page=0&size=10
```

Page 0 → first 10 records
Page 1 → next 10 records
Page 2 → next 10 after that

In Spring Boot with JPA this is straightforward:

```java
@GetMapping
public ResponseEntity<Page<Product>> getProducts(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size) {

    Pageable pageable = PageRequest.of(page, size);
    Page<Product> products = productRepository.findAll(pageable);
    return ResponseEntity.ok(products);
}
```

Spring's `Page` object gives you a lot out of the box:

```
{
  "content": [...],
  "totalElements": 500,
  "totalPages": 50,
  "currentPage": 0,
  "size": 10,
  "first": true,
  "last": false
}
```

Frontend knows exactly how many pages exist and where it is.

---

## Adding sorting

```java
Pageable pageable = PageRequest.of(page, size, Sort.by("createdAt").descending());
```

Or via query params:

```
GET /api/products?page=0&size=10&sort=createdAt,desc
```

Spring handles this automatically if you pass `Pageable` directly as a method param:

```java
@GetMapping
public ResponseEntity<Page<Product>> getProducts(Pageable pageable) {
    return ResponseEntity.ok(productRepository.findAll(pageable));
}
```

---

## Problem with offset pagination

Offset pagination has one known issue — if data changes between page requests you can get duplicate or missing records.

Say you're on page 1, someone adds a new record at the top. When you fetch page 2, the records shift and you either see a duplicate from page 1 or miss one entirely.

For most apps this is acceptable. For real time feeds or large datasets it's a problem — that's where cursor based pagination comes in.

---

## Cursor Based Pagination

Instead of page numbers you use a cursor — usually the id or timestamp of the last record you saw.

```
GET /api/products?cursor=102&size=10
```

"Give me 10 products after id 102."

```java
@GetMapping
public ResponseEntity<List<Product>> getProducts(
        @RequestParam Long cursor,
        @RequestParam(defaultValue = "10") int size) {

    List<Product> products = productRepository
            .findByIdGreaterThanOrderByIdAsc(cursor, PageRequest.of(0, size));
    return ResponseEntity.ok(products);
}
```

No matter how many records get added or deleted, the cursor always points to a stable position. No duplicates, no missing records.

**Downside** — you can't jump to page 5 directly. You have to go through pages 1, 2, 3, 4 first. No random access.

---

## Which one to use

| | Offset | Cursor |
|---|---|---|
| Simple to implement | Yes | No |
| Jump to any page | Yes | No |
| Stable with changing data | No | Yes |
| Good for large datasets | No | Yes |

For most CRUD apps — offset pagination with Spring's `Pageable` is fine. For feeds, timelines, or large datasets — cursor based.

---

## Stuff I want to remember

**What is the difference between offset and cursor based pagination?**

Offset uses page numbers — skip this many records, take this many. Simple but unstable when data changes between requests. Cursor uses a pointer to the last seen record — always stable, no duplicates, but you can't jump to an arbitrary page. I use offset for most projects since Spring JPA makes it very easy with Pageable.

**What does Spring's Page object return?**

It returns the content array plus metadata — total elements, total pages, current page, whether it's the first or last page. The frontend can use this to build pagination controls without making extra requests.

**Why is offset pagination bad for large datasets?**

Because `OFFSET 10000 LIMIT 10` in SQL still scans through 10,000 rows before returning 10. The bigger the offset, the slower the query. Cursor based pagination avoids this because it uses `WHERE id > cursor` which can use an index directly.

---

*Used Spring's Pageable in a project — didn't realise how much it gives you out of the box until I saw the response. Total pages, first/last flags, everything. Saved a lot of manual work.*