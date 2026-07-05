# API Design Best Practices

## 1. URL & Resource Design

### Use Nouns, Not Verbs
URLs should represent resources (things), not actions. HTTP methods express the action.

```
# Bad
GET  /getUser/123
POST /createOrder
POST /deleteItem/456

# Good
GET    /users/123
POST   /orders
DELETE /items/456
```

### Use Plural Nouns for Collections
```
/users          # collection
/users/123      # single resource
/users/123/orders   # sub-resource (orders belonging to user 123)
```

### Use Kebab-Case for Multi-Word Path Segments
```
# Bad
/productCategories
/product_categories

# Good
/product-categories
```

### Keep Hierarchy Shallow
Nest resources only when the sub-resource cannot meaningfully exist without the parent. Beyond 2 levels, prefer query parameters.

```
# Okay
GET /orders/123/items

# Getting unwieldy
GET /users/123/orders/456/items/789/reviews

# Better — flatten with query params
GET /reviews?orderId=456&itemId=789
```

### Don't Put Actions in URLs — Unless Truly Necessary
For operations that don't map cleanly to CRUD, use a verb *as a sub-resource* as a last resort:

```
# Tolerable (action on a resource)
POST /orders/123/cancel
POST /accounts/456/verify-email

# Avoid — looks like an RPC endpoint
POST /cancelOrder?id=123
```

---

## 2. HTTP Methods

| Method | Semantics | Idempotent | Safe |
|--------|-----------|------------|------|
| GET | Fetch resource(s) | Yes | Yes |
| POST | Create a new resource | No | No |
| PUT | Replace a resource entirely | Yes | No |
| PATCH | Partially update a resource | No* | No |
| DELETE | Remove a resource | Yes | No |

*PATCH _can_ be idempotent if designed carefully (e.g., `set field X to value Y`), but it's not guaranteed.

### Never Use GET for Mutating Operations
GET requests may be cached, retried, or prefetched. Side effects will be triggered unexpectedly.

```
# Bad
GET /users/123/deactivate

# Good
POST /users/123/deactivate
```

### Use PUT for Full Replacement, PATCH for Partial Updates
```
# PUT — replaces the whole user; missing fields are unset/defaulted
PUT /users/123
{ "name": "Alice", "email": "alice@example.com", "role": "admin" }

# PATCH — only updates the fields provided
PATCH /users/123
{ "email": "alice@new.com" }
```

---

## 3. HTTP Status Codes

Return the most specific, correct code. Never return `200 OK` for an error.

### Common Status Codes

| Code | When to Use |
|------|-------------|
| 200 OK | Successful GET, PUT, PATCH |
| 201 Created | Successful POST that created a resource |
| 204 No Content | Successful DELETE or action with no response body |
| 400 Bad Request | Client sent malformed or invalid data |
| 401 Unauthorized | Not authenticated (no or invalid credentials) |
| 403 Forbidden | Authenticated but not authorized |
| 404 Not Found | Resource doesn't exist |
| 409 Conflict | State conflict (e.g., duplicate, optimistic lock failure) |
| 422 Unprocessable Entity | Syntactically valid but semantically invalid request |
| 429 Too Many Requests | Rate limit exceeded |
| 500 Internal Server Error | Unexpected server-side failure |
| 503 Service Unavailable | Downstream dependency unavailable |

### Don't Abuse 200
```json
// Bad — 200 with error body
HTTP/1.1 200 OK
{ "success": false, "error": "User not found" }

// Good
HTTP/1.1 404 Not Found
{ "error": "USER_NOT_FOUND", "message": "No user with id 123" }
```

### 401 vs. 403
- `401` — "I don't know who you are." (missing/invalid auth token)
- `403` — "I know who you are, but you can't do this."

---

## 4. Error Responses

Use a **consistent, structured error format** across all endpoints.

```json
{
  "error": "VALIDATION_FAILED",
  "message": "Request body is invalid",
  "traceId": "abc-123-def-456",
  "details": [
    { "field": "email", "issue": "must be a valid email address" },
    { "field": "age",   "issue": "must be a positive integer" }
  ]
}
```

- `error` — machine-readable, stable code (treat like an enum)
- `message` — human-readable description (can change without breaking clients)
- `traceId` — correlates the error to logs without exposing internals
- `details` — optional array for field-level validation errors

Never expose stack traces, internal class names, or SQL errors in responses.

For how to implement centralized exception-to-HTTP-status mapping (global exception handlers, custom exception hierarchies), see Exception Handling.

---

## 5. Versioning

### Version at the URL Path (Preferred)
```
/v1/orders
/v2/orders
```
Easy to see in logs, easy to route at the gateway, easy to test manually.

### Alternative: Version via Accept Header
```
Accept: application/vnd.myapi.v2+json
```
Cleaner URLs, but harder to test and route. Prefer URL versioning for simplicity.

### When to Version
- Breaking changes require a new version. Breaking changes include:
  - Removing or renaming fields
  - Changing field types
  - Removing endpoints
  - Changing status code semantics
- Adding new optional fields is **not** breaking — don't bump the version for additive changes.

### Deprecation Strategy
- Announce deprecation in advance with a header: `Deprecation: true` / `Sunset: Sat, 01 Jan 2026 00:00:00 GMT`
- Keep the old version alive for a reasonable migration window

---

## 6. Request & Response Body Design

### Use camelCase for JSON Fields
Consistent with JavaScript conventions, which most API consumers use.

```json
{ "firstName": "Alice", "createdAt": "2025-01-01T00:00:00Z" }
```

### Use ISO 8601 for Dates and Times
```json
{ "createdAt": "2025-07-04T12:00:00Z" }   // UTC preferred
{ "startDate": "2025-07-04" }               // Date-only when time is irrelevant
```

Never use Unix timestamps as the primary format — they're not human-readable and are error-prone (seconds vs. milliseconds confusion).

### Avoid Envelopes Unless Necessary
Don't wrap every response in a generic `data` field just for the sake of it.

```json
// Unnecessary envelope
{ "status": "success", "data": { "id": 1, "name": "Alice" } }

// Cleaner — the HTTP status code already communicates success
{ "id": 1, "name": "Alice" }
```

Envelopes are justified when you need top-level metadata alongside the payload (e.g., pagination info).

### Pagination — Always for Collections
Never return unbounded collections. Prefer **cursor-based** pagination for large or real-time datasets; **offset-based** is simpler for static data.

**Cursor-based (preferred for large data):**
```json
{
  "items": [...],
  "nextCursor": "eyJpZCI6MTAwfQ==",
  "hasMore": true
}
```

**Offset-based:**
```json
{
  "items": [...],
  "total": 2847,
  "page": 3,
  "pageSize": 25
}
```

Use consistent query params: `?limit=25&cursor=...` or `?page=3&pageSize=25`.

### Filtering and Sorting via Query Parameters
```
GET /orders?status=pending&customerId=123&sort=createdAt&order=desc
```

Document supported filter fields explicitly — don't silently ignore unknown parameters.

---

## 7. Idempotency

### GET, PUT, DELETE Must Be Idempotent
Calling them multiple times should produce the same result. Design PUT and DELETE accordingly.

### Use Idempotency Keys for POST
For critical mutations (payments, order creation), accept an `Idempotency-Key` header. Store the result and return the same response on replay.

```
POST /payments
Idempotency-Key: a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

This lets clients safely retry on network failure without double-charging.

---

## 8. Authentication & Authorization

- Always use HTTPS — never transmit credentials or tokens over plain HTTP
- Prefer **OAuth 2.0 / JWT bearer tokens** for API authentication
- **Never** put credentials, tokens, or API keys in the URL path or query string — they end up in server logs and browser history
  ```
  # Bad
  GET /data?api_key=supersecret

  # Good
  GET /data
  Authorization: Bearer <token>
  ```
- Validate permissions at the API layer before any business logic runs
- Apply the principle of least privilege — tokens should only have the scopes they need

---

## 9. Request Validation

Validate early and fail fast at the API boundary. Don't let bad input reach business logic or the database.

- Validate required fields, types, ranges, and formats
- Return **all** validation errors in one response (not one at a time) — saves round trips for the client
- Use `400 Bad Request` for malformed requests, `422 Unprocessable Entity` for semantically invalid ones
- Document constraints clearly in your API spec

For security-focused validation (injection prevention, allowlist vs. denylist strategies, XSS, SQL injection), see Security.

---

## 10. Caching

### Use Standard HTTP Cache Headers
```
Cache-Control: max-age=3600, public      // cache for 1 hour
Cache-Control: no-store                 // never cache (sensitive data)
ETag: "abc123"                          // for conditional requests
Last-Modified: Wed, 01 Jan 2025 00:00:00 GMT
```

- `GET` endpoints should declare their cacheability explicitly
- Use `ETag` / `If-None-Match` to avoid re-sending unchanged payloads (304 Not Modified)
- Responses containing user-specific data should use `Cache-Control: private`

---

## 11. Documentation

- Maintain an **OpenAPI (Swagger)** spec — it serves as the contract between producer and consumer
- Keep the spec in source control alongside the code
- Document every field (description, type, example, constraints)
- Include example request/response payloads for every endpoint
- Document all possible error codes per endpoint, not just the happy path

---

## Quick Reference Checklist

- [ ] URLs use nouns (resources), not verbs — HTTP methods express the action
- [ ] Resource paths use plural nouns and kebab-case
- [ ] Correct HTTP status codes returned (no `200` for errors)
- [ ] Consistent structured error response with `error`, `message`, and `traceId`
- [ ] API is versioned; breaking changes bump the version
- [ ] All date/time fields use ISO 8601 format
- [ ] Collections are paginated — no unbounded list endpoints
- [ ] POST mutations that must be retryable support `Idempotency-Key`
- [ ] Credentials/tokens are never in URLs
- [ ] Input validation happens at the API boundary; all validation errors returned together
- [ ] Cache headers set appropriately on GET endpoints
- [ ] OpenAPI spec exists, is in source control, and is up to date
