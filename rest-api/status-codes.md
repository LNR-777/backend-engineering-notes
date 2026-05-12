# HTTP Status Codes

HTTP status codes are returned by the server to indicate the result of an API request.

In backend development, status codes help the frontend understand whether a request was successful or failed.

---

## 200 OK

Indicates that the request was successful.

Example:
- Fetching products successfully
- Getting leave request records

```http
200 OK
```

---

## 201 Created

Indicates that new data was created successfully.

Example:
- New user registration
- Leave request submission

```http
201 Created
```

---

## 400 Bad Request

Indicates that the client sent invalid request data.

Example:
- Missing required fields
- Invalid input values

```http
400 Bad Request
```

---

## 401 Unauthorized

Indicates that authentication is required.

Example:
- Accessing secured API without login

```http
401 Unauthorized
```

---

## 403 Forbidden

Indicates that the user does not have permission to access the resource.

Example:
- Employee trying to access admin APIs

```http
403 Forbidden
```

---

## 404 Not Found

Indicates that the requested resource does not exist.

Example:
- Product ID not found
- User not found

```http
404 Not Found
```

---

## 500 Internal Server Error

Indicates that something failed on the server side.

Example:
- Database connection failure
- Unexpected backend exception

```http
500 Internal Server Error
```

---

## My Learning

While building Spring Boot projects, understanding status codes helped me create better API responses and improve frontend-backend communication.