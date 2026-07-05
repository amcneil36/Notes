# Exception Handling Best Practices

## HTTP APIs — Never Expose Raw Exceptions

### Why It Matters
Returning raw exceptions or stack traces from HTTP endpoints is a **security risk and an API design smell**:
- Leaks internal implementation details (class names, library versions, DB schema hints)
- Gives attackers a roadmap to your stack and potential vulnerabilities
- Tightly couples your API contract to internal code structure
- Exceptions are developer-facing; HTTP responses are consumer-facing

### The Right Pattern
Always translate exceptions into a **stable, structured error response**. For the full error response contract (fields, format, status code mapping table), see API Design — Error Responses. The key implementation points:

- Map exceptions → HTTP status codes centrally (e.g., a global exception handler / `@ControllerAdvice` in Spring)
- Include a `traceId` / `correlationId` so logs can be correlated without exposing internals
- Never expose stack traces in prod responses — not even in `4xx` responses

### Nuance: Internal/Dev APIs
In strictly internal APIs (never public-facing), surfacing exception *messages* (not traces) can speed up debugging. Still avoid stack traces. Use environment-gated behavior if needed.

---

## Browser / Frontend — Source Maps and Console Errors

### Network Tab (XHR/Fetch returning stack traces)
Same issue as above — **high severity**. Server is leaking backend internals to the browser. Fix: centralized server-side exception handling.

### Sources Tab (JS stack traces visible)
Two root causes:
- **Unminified JS in prod** — always minify/bundle for production
- **Source maps served publicly** — source maps should be uploaded *only* to your error monitoring tool (Sentry, Datadog, Splunk), never served from your CDN or app server

### Console Errors
Can't be fully prevented (client-side code is public), but:
- Never log sensitive data (tokens, PII, internal URLs) to the console
- Strip verbose logging in production builds via build flags

| Issue | Fix |
|---|---|
| XHR returns stack trace | Server-side global exception handler |
| Source maps public in prod | Upload to error tracker only; exclude from CDN output |
| Console leaks sensitive data | Environment-aware logging; strip in prod builds |

---

## General Exception Handling Best Practices

### 1. Catch Specific Exceptions at Each Layer — Let the Global Handler Be the Catch-All

The rule: **don't scatter broad `catch (Exception e)` blocks throughout your services and controllers.** That's the anti-pattern — it hides bugs and prevents you from responding appropriately to different failure modes.

```java
// Bad — in a service or controller
try { ... } catch (Exception e) { log(e); }

// Good — specific handlers at each layer
try { ... } catch (ResourceNotFoundException e) { ... }
              catch (ValidationException e) { ... }
```

The **one legitimate place** to catch `Exception` broadly is in your global error handler — and even then, it's intentional:

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<?> handleNotFound(ResourceNotFoundException e) { ... }

    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<?> handleValidation(ValidationException e) { ... }

    // Intentional safety net — catches anything that slipped through
    @ExceptionHandler(Exception.class)
    public ResponseEntity<?> handleUnexpected(Exception e) {
        log.error("Unhandled exception", e);
        return ResponseEntity.status(500).body(buildErrorResponse("INTERNAL_ERROR", traceId));
    }
}
```

The global handler's catch-all is **architecturally deliberate** — its only job is to ensure no exception ever reaches the client unhandled. The distinction is intent:

| Where | Catching `Exception` broadly | Verdict |
|---|---|---|
| Service / business logic | "I don't know what might throw" | Bad |
| Individual controller | "I'll handle it here to be safe" | Bad |
| Global error handler | "Nothing should escape unhandled" | Good |

### 2. Don't Swallow Exceptions Silently
```java
// Bad — exception disappears
try { riskyOp(); } catch (Exception e) { }

// Good
try { riskyOp(); } catch (Exception e) {
    log.error("Failed to execute riskyOp", e);
    throw new ServiceException("Operation failed", e);
}
```

### 3. Preserve the Cause (Exception Chaining)
Always pass the original exception as the cause when wrapping:
```java
throw new ServiceException("Order processing failed", originalException);
```
This keeps the full diagnostic chain intact in your logs.

### 4. Use Custom Exception Hierarchy
Define a hierarchy that maps to your domain:
```
AppException (base)
├── ValidationException       → 400
├── AuthenticationException   → 401
├── AuthorizationException    → 403
├── ResourceNotFoundException → 404
├── ConflictException         → 409
└── InternalServiceException  → 500
```
This makes centralized HTTP mapping clean and predictable.

### 5. Centralize HTTP Exception Mapping
Don't map exceptions to HTTP responses in every controller. Use a single handler:
- **Spring**: `@ControllerAdvice` / `@ExceptionHandler`
- **Node/Express**: error middleware
- **Go**: middleware wrapper pattern

### 6. Log at the Right Level
- `ERROR` — unexpected failures requiring investigation
- `WARN` — expected failure paths (e.g., not found, bad input) — often better logged at this level or not at all
- `DEBUG` — verbose diagnostic info (off in prod)

Avoid logging the same exception multiple times at different layers — it clutters logs and inflates noise.

### 7. Don't Use Exceptions for Control Flow
```java
// Bad — using exception as a conditional
try {
    return cache.get(key);
} catch (CacheMissException e) {
    return db.get(key); // this is just an if/else
}

// Good
if (cache.has(key)) return cache.get(key);
return db.get(key);
```
Exceptions have overhead and obscure intent when used as flow control.

### 8. Fail Fast at Boundaries
Validate inputs at system entry points (API layer, queue consumers, scheduled jobs) and throw early. Don't let bad data propagate deep into business logic where it's harder to trace.

### 9. Distinguish Retryable vs. Non-Retryable Errors
When calling downstream services, classify exceptions:
- **Retryable**: transient network errors, 503s, timeouts → retry with backoff
- **Non-retryable**: 400 Bad Request, 404, auth failures → fail immediately

Mark custom exceptions accordingly (e.g., `@Retryable` annotation, or an `isRetryable()` flag).

### 10. Alert on the Right Things
Not every exception deserves a PagerDuty alert:
- `ResourceNotFoundException` → not an alert
- Spike in `InternalServiceException` → alert
- Any `Error` (OOM, StackOverflow) → alert immediately

Use error rate thresholds, not raw exception counts.

---

## Quick Reference Checklist

- [ ] Global exception handler in place (no per-controller try/catch for HTTP mapping)
- [ ] API never returns raw stack traces in any environment exposed externally
- [ ] All exceptions are logged with `traceId` / `correlationId`
- [ ] Custom exception hierarchy exists and maps to HTTP status codes
- [ ] Source maps not publicly served in production
- [ ] Exceptions are not swallowed silently anywhere
- [ ] Retryable vs. non-retryable exceptions are classified for downstream calls
- [ ] Alerts are tuned to actionable exception types, not all exceptions
