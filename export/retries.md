# Retry Best Practices

Retries are the most intuitive resilience mechanism in distributed systems — something failed, so try again. And that simplicity is exactly what makes them dangerous. A naive retry strategy can turn a minor blip into a cascading outage that takes down your entire service mesh. A well-designed retry strategy can make transient failures invisible to users.

The goal: **retry when it's safe, back off when it's not, and never make a bad situation worse.**

---

## Why Retries Exist — The Transient Failure Problem

In a distributed system, failures are not binary. They're not just "works" or "broken." There's a third state: **transiently broken** — the request failed, but the same request sent a moment later would succeed.

Common transient failures:

| Failure | Cause | Retryable? |
|---|---|---|
| `503 Service Unavailable` | Upstream is temporarily overloaded or deploying | Yes — it'll likely recover |
| `504 Gateway Timeout` | Upstream is slow, proxy timed out | Maybe — depends on whether the request completed |
| Connection reset / refused | Pod restarting, network blip, load balancer cycling | Yes |
| DNS resolution failure | Transient DNS propagation issue | Yes — with a brief delay |
| `429 Too Many Requests` | Rate limited | Yes — after the `Retry-After` period |
| `500 Internal Server Error` | Bug or transient crash | Maybe — depends on the root cause |
| `400 Bad Request` | Client sent malformed input | **No** — retrying won't fix the input |
| `401 Unauthorized` | Invalid/expired credentials | **No** — retrying won't generate a valid token |
| `404 Not Found` | Resource doesn't exist | **No** — the resource won't appear on retry |
| `409 Conflict` | State conflict (e.g., concurrent update) | **Maybe** — if you re-read state and retry with updated data |

The fundamental rule: **only retry on failures that have a reasonable chance of succeeding on the next attempt.** Retrying a `400 Bad Request` fifty times is not resilience — it's insanity.

---

## Retry Strategies — Choosing the Right Backoff

### 1. Immediate Retry (No Backoff)

Retry instantly, zero delay.

```
Attempt 1: fail → Attempt 2: fail → Attempt 3: success
```

**When to use:** Almost never. Only appropriate for in-process operations (local cache miss, lock contention) where the failure is expected to resolve within microseconds.

**When NOT to use:** Network calls. Immediate retries against a struggling upstream service just pile more load on it at exactly the wrong moment.

### 2. Fixed Delay

Wait a constant duration between retries.

```
Attempt 1: fail → wait 1s → Attempt 2: fail → wait 1s → Attempt 3: success
```

**When to use:** Simple scenarios where the delay is well-matched to the expected recovery time (e.g., retrying after a known deployment window).

**Problem:** If 200 clients all retry at 1-second intervals, they all retry at the same time, creating synchronized load spikes (the "thundering herd"). This is why fixed delay alone is almost always wrong for service-to-service calls.

### 3. Exponential Backoff (Recommended Baseline)

Each retry waits exponentially longer: 1s, 2s, 4s, 8s...

```
Attempt 1: fail → wait 1s → Attempt 2: fail → wait 2s → Attempt 3: fail → wait 4s → Attempt 4: success
```

**When to use:** Most service-to-service HTTP calls. Gives the upstream time to recover while still retrying.

**Formula:**
```
delay = min(base_delay * 2^(attempt - 1), max_delay)
```

**Problem:** Still causes synchronized retries if many clients start at the same time. All of them back off on the same schedule.

### 4. Exponential Backoff with Jitter (Recommended)

Add randomness to break synchronization. This is the correct default for almost all retry scenarios.

```
delay = random(0, min(base_delay * 2^(attempt - 1), max_delay))
```

Or the "decorrelated jitter" variant (slightly better distribution):

```
delay = random(base_delay, previous_delay * 3)
delay = min(delay, max_delay)
```

**Java (pseudocode):**
```java
long calculateDelay(int attempt, long baseDelayMs, long maxDelayMs) {
    long exponentialDelay = baseDelayMs * (1L << (attempt - 1));
    long cappedDelay = Math.min(exponentialDelay, maxDelayMs);
    return ThreadLocalRandom.current().nextLong(0, cappedDelay);
}
```

**JavaScript:**
```js
function calculateDelay(attempt, baseDelayMs, maxDelayMs) {
  const exponentialDelay = baseDelayMs * Math.pow(2, attempt - 1);
  const cappedDelay = Math.min(exponentialDelay, maxDelayMs);
  return Math.floor(Math.random() * cappedDelay);
}
```

Jitter spreads retries across time, preventing thundering herds. The AWS Architecture Blog published the math on this in 2015 and it's still the gold standard.

### 5. Linear Backoff

Wait proportionally longer: 1s, 2s, 3s, 4s...

```
delay = base_delay * attempt
```

**When to use:** When you want a gentler ramp than exponential but still increasing. Useful for polling operations where you don't want to wait 32 seconds on the 6th attempt.

### Strategy Comparison

| Strategy | Thundering Herd Risk | Recovery Speed | Complexity | Best For |
|---|---|---|---|---|
| Immediate | Extreme | Fastest | None | In-process only |
| Fixed delay | High | Medium | Low | Simple, known delays |
| Exponential | Medium | Slow at high attempts | Low | General purpose |
| Exponential + jitter | **Low** | Moderate | Low | **Default for HTTP** |
| Linear | Medium | Moderate | Low | Polling |

---

## Retry Budgets and Limits

### Max Retry Count

Every retry loop **must** have a maximum attempt count. An unbounded retry is an infinite loop waiting to happen.

| Scenario | Recommended Max Retries |
|---|---|
| Synchronous HTTP (user-facing) | 2–3 (total 3–4 attempts) |
| Synchronous HTTP (service-to-service) | 3–5 |
| Async message processing | 3–5 before DLQ |
| Batch / scheduled job | 5–10 (with longer backoff) |
| Database connection | 3 |
| DNS resolution | 2 |

For user-facing requests, 3 total attempts is usually the upper limit. A user watching a spinner for 30 seconds while you retry 10 times is not a good experience — fail fast and show a clear error.

### Total Timeout (Retry Budget)

Max retries alone aren't enough. You also need a **total timeout** — the maximum wall-clock time you're willing to spend on all attempts combined.

```
Total timeout = initial_timeout + retry_1_delay + retry_1_timeout + retry_2_delay + retry_2_timeout + ...
```

**Example:**
- Per-attempt timeout: 5s
- Max retries: 3
- Backoff: 1s, 2s, 4s
- Total worst case: 5s + 1s + 5s + 2s + 5s + 4s + 5s = 27 seconds

If that 27 seconds exceeds your SLA for the endpoint, reduce the per-attempt timeout or the retry count.

**Java:**
```java
RetryConfig config = RetryConfig.custom()
    .maxAttempts(3)
    .waitDuration(Duration.ofSeconds(1))
    .retryOnResult(response -> response.getStatusCode() >= 500)
    .retryOnException(e -> e instanceof ConnectException)
    .build();
```

**JavaScript:**
```js
async function withRetry(fn, { maxAttempts = 3, baseDelayMs = 1000, maxDelayMs = 10000 } = {}) {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (attempt === maxAttempts || !isRetryable(err)) throw err;
      const delay = calculateDelay(attempt, baseDelayMs, maxDelayMs);
      await new Promise(r => setTimeout(r, delay));
    }
  }
}
```

### Retry Budget (Percentage-Based)

Instead of per-request retry limits, some systems use a global retry budget: "retries should be at most 20% of total requests." If your service sends 100 real requests per second, it can send at most 20 retries per second. This prevents retry amplification at the system level.

Google SRE uses this approach. It's more sophisticated than per-request limits but requires centralized tracking (e.g., a shared counter in Redis or an in-process sliding window).

---

## Idempotency — The Prerequisite for Safe Retries

A retry is only safe if the operation produces the same result when executed multiple times. This property is called **idempotency**.

### The Idempotency Problem

```
Client → POST /orders → Server (processes, creates order, response lost in transit)
Client → POST /orders → Server (processes again → DUPLICATE ORDER)
```

The client retried because it didn't get a response. But the server already processed the first request. Now there are two orders.

### Which Operations Are Naturally Idempotent?

| HTTP Method | Idempotent? | Why |
|---|---|---|
| `GET` | Yes | Read-only, no side effects |
| `PUT` | Yes (if implemented correctly) | Sets state to a specific value, same result on repeat |
| `DELETE` | Yes (if implemented correctly) | Deleting an already-deleted resource is a no-op |
| `POST` | **No** | Creates new resources — each call may create a duplicate |
| `PATCH` | **No** (usually) | Depends on implementation — "increment by 1" is not idempotent |

### Making Non-Idempotent Operations Safe

#### Strategy 1: Idempotency Keys

The client generates a unique key (UUID) and sends it with every attempt. The server deduplicates.

```
POST /orders
X-Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

Server:
1. Check if key exists in idempotency store
2. If yes → return cached response (don't process again)
3. If no → process, store key + response, return response
```

**Java:**
```java
@PostMapping("/orders")
public ResponseEntity<?> createOrder(
    @RequestHeader("X-Idempotency-Key") String idempotencyKey,
    @RequestBody OrderRequest request) {

    // Check for existing result
    Optional<OrderResponse> existing = idempotencyStore.get(idempotencyKey);
    if (existing.isPresent()) {
        return ResponseEntity.ok(existing.get());
    }

    // Process and store
    OrderResponse response = orderService.create(request);
    idempotencyStore.put(idempotencyKey, response, Duration.ofHours(24));
    return ResponseEntity.status(HttpStatus.CREATED).body(response);
}
```

**JavaScript:**
```js
app.post('/orders', async (req, res) => {
  const idempotencyKey = req.headers['x-idempotency-key'];
  if (!idempotencyKey) return res.status(400).json({ error: 'Missing idempotency key' });

  const cached = await redis.get(`idempotency:${idempotencyKey}`);
  if (cached) return res.status(200).json(JSON.parse(cached));

  const result = await orderService.create(req.body);
  await redis.set(`idempotency:${idempotencyKey}`, JSON.stringify(result), 'EX', 86400);
  return res.status(201).json(result);
});
```

#### Strategy 2: Conditional Requests (ETags / Versioning)

For updates, use `If-Match` headers with version numbers or ETags:

```
PUT /orders/123
If-Match: "v5"

Server: only apply the update if the current version is v5. Return 412 Precondition Failed if not.
```

This prevents stale retries from overwriting newer data.

#### Strategy 3: Natural Idempotency Through Design

Design operations to be naturally idempotent where possible:

```
// Not idempotent — each call adds $10
POST /accounts/123/credit { "amount": 10.00 }

// Idempotent — sets balance to a specific value
PUT /accounts/123/balance { "balance": 110.00 }

// Idempotent — each call with this transactionId is a no-op after the first
POST /accounts/123/credit { "amount": 10.00, "transactionId": "txn-abc-123" }
```

---

## Retries in Async / Event-Driven Systems

Retrying HTTP calls is straightforward. Retrying message processing is a different beast with its own patterns and pitfalls.

### Kafka Retry Patterns

#### Pattern 1: In-Process Retry (Simple)

Retry within the consumer before acknowledging. The message stays on the partition until committed.

```java
@KafkaListener(topics = "orders")
public void consume(ConsumerRecord<String, Order> record) {
    for (int attempt = 1; attempt <= 3; attempt++) {
        try {
            orderService.process(record.value());
            return; // success — offset will be committed
        } catch (TransientException e) {
            if (attempt == 3) throw e; // will trigger error handler
            sleep(1000 * attempt);
        }
    }
}
```

**Problem:** Blocks the consumer while retrying. If backoff is long, consumer lag grows. If the partition has many messages, one poison pill can stall everything behind it.

#### Pattern 2: Retry Topics (Recommended for Kafka)

Route failed messages to dedicated retry topics with increasing delays, keeping the main topic flowing.

```
main-topic → consumer fails → retry-topic-1 (1 min delay)
                             → retry-topic-2 (5 min delay)
                             → retry-topic-3 (30 min delay)
                             → dead-letter-topic (give up)
```

**Spring Kafka makes this straightforward:**
```java
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<String, Object> template) {
    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(template,
        (record, ex) -> new TopicPartition(record.topic() + ".dlt", record.partition()));

    DefaultErrorHandler handler = new DefaultErrorHandler(recoverer,
        new FixedBackOff(1000L, 3L)); // 1s delay, 3 retries

    handler.addNotRetryableExceptions(
        ValidationException.class,     // bad data — don't retry
        DeserializationException.class  // corrupt message — don't retry
    );
    return handler;
}
```

#### Pattern 3: Dead Letter Queue (DLQ)

After all retry attempts are exhausted, the message lands in a DLQ. This is not optional — it's the safety net.

**DLQ requirements:**
- Preserve the original message, headers, and metadata
- Add failure context: error message, stack trace, attempt count, timestamp of each failure
- Set up alerting on DLQ depth — messages sitting in DLQ means data is not being processed
- Build tooling to inspect, replay, and purge DLQ messages
- Monitor DLQ growth rate — a spike means something systemic is wrong

**What NOT to do with DLQs:**
- Don't ignore them. A growing DLQ is lost data.
- Don't auto-replay without fixing the root cause. You'll just DLQ them again.
- Don't use the same DLQ for all topics. Separate by topic or domain so failures in one stream don't obscure others.

### Poison Pill Messages

A poison pill is a message that will never process successfully, no matter how many times you retry. It's invalid data, a schema mismatch, or a bug in the consumer that triggers on specific input.

**Detection:**
```java
try {
    process(message);
} catch (Exception e) {
    if (isPoisonPill(e)) {
        log.error("Poison pill detected, routing to DLQ", { topic, partition, offset, error: e.getMessage() });
        sendToDlq(message, e);
        return; // commit offset, move on
    }
    throw e; // transient — retry
}

boolean isPoisonPill(Exception e) {
    return e instanceof DeserializationException
        || e instanceof ValidationException
        || e instanceof SchemaException
        || e instanceof NullPointerException; // almost always a data issue
}
```

The key insight: **non-retryable errors should skip the retry loop entirely** and go straight to the DLQ. Retrying a message that fails on deserialization 3 times with exponential backoff is wasted time and compute.

---

## Circuit Breakers — When to Stop Retrying Entirely

Retries handle transient failures. Circuit breakers handle sustained failures. They're complementary, not competing.

When an upstream service is truly down — not a blip, but a sustained outage — retrying makes things worse. Every retry adds load to an already-struggling service, consumes your thread pool, and delays your response. A circuit breaker detects this pattern and stops sending requests entirely for a cooldown period.

See Graceful Degradation for circuit breaker fundamentals (states, configuration, fallback strategies).

### Retry + Circuit Breaker Interaction

**The correct order: Retry wraps Circuit Breaker.**

```
Request → Retry(attempt 1) → CircuitBreaker → HTTP call (fails)
                             → CircuitBreaker → HTTP call (fails)
        → Retry(attempt 2) → CircuitBreaker → HTTP call (succeeds)
```

If the circuit is open, the retry gets an immediate rejection (no network call) and can decide whether to wait and try again or fail fast. If you put circuit breaker outside retry, the circuit breaker sees every retry as a separate failure and opens prematurely.

**Java (Resilience4j composition):**
```java
Supplier<Response> supplier = () -> httpClient.call(request);

// Correct: retry decorates circuit breaker
supplier = CircuitBreaker.decorateSupplier(circuitBreaker, supplier);
supplier = Retry.decorateSupplier(retry, supplier);
```

---

## Retry Amplification — The Cascade That Kills Systems

This is the single most dangerous anti-pattern in retry design, and it's shockingly common.

### The Scenario

```
Service A → retries 3x → Service B → retries 3x → Service C → retries 3x → Database

If Database is slow:
- Service C sends 3 requests to Database
- Service B sees C fail, sends 3 requests to C → 3 × 3 = 9 DB requests
- Service A sees B fail, sends 3 requests to B → 3 × 9 = 27 DB requests

1 user request → 27 database hits
```

With 4 layers, it's 81. With 5, it's 243. This geometric amplification turns a slow database into a full outage across your entire service mesh.

### How to Prevent It

**1. Only retry at the edge (preferred)**

Only the outermost caller retries. Intermediate services fail fast and propagate the error.

```
Service A (retries 3x) → Service B (no retry) → Service C (no retry) → Database
```

**2. Retry budgets**

Cap total retries as a percentage of normal traffic. If retries exceed 20% of total requests, stop retrying system-wide.

**3. Propagate retry context**

Pass a `Retry-Count` or `X-Remaining-Budget` header so downstream services know they're in a retry chain and can adjust behavior:

```
X-Retry-Attempt: 2
X-Max-Retries: 3
```

Downstream services can choose to skip their own retries when they see they're already inside a retry.

**4. Use circuit breakers at every layer**

When a service consistently fails, the circuit breaker stops sending requests entirely, breaking the amplification chain.

**5. Set per-request deadlines**

Propagate a deadline (absolute timestamp) instead of a timeout (relative duration). If the deadline has passed by the time a downstream receives the request, it can reject it immediately instead of doing work that will be wasted.

```
X-Request-Deadline: 2026-07-05T14:30:00.000Z
```

```java
Instant deadline = Instant.parse(request.getHeader("X-Request-Deadline"));
if (Instant.now().isAfter(deadline)) {
    return ResponseEntity.status(504).body("Request deadline exceeded");
}
```

---

## Timeouts — The Other Half of the Equation

Retries without timeouts are dangerous. A retry that hangs for 60 seconds before failing burns a thread, a connection, and a user's patience.

### Timeout Types

| Timeout | What It Controls | Recommended Default |
|---|---|---|
| **Connection timeout** | Time to establish TCP connection | 1–3 seconds |
| **Read/response timeout** | Time to receive the full response after connecting | 5–15 seconds (depends on operation) |
| **Total request timeout** | Wall-clock time for the entire attempt including retries | Based on SLA |
| **Idle timeout** | How long to keep an idle connection alive | 30–60 seconds |

### Timeout Best Practices

1. **Always set both connection and read timeouts.** Never rely on defaults — many HTTP clients default to infinity.
2. **Set the timeout shorter than the caller's timeout.** If Service A gives you 10 seconds, your downstream call should timeout at 8 seconds — leaving 2 seconds for processing the response.
3. **Budget timeouts across the call chain.** If the user-facing SLA is 3 seconds, and you have 3 downstream calls, each one gets ~800ms, not 3 seconds each.
4. **Shorter timeouts for retryable calls.** If you're going to retry 3 times, each attempt should have a tight timeout so the total stays within budget.

```java
// Tight timeout per attempt, total budget enforced separately
HttpClient client = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(2))
    .build();

HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("
    .timeout(Duration.ofSeconds(5))
    .build();
```

```js
const controller = new AbortController();
const timeoutId = setTimeout(() => controller.abort(), 5000);

try {
  const response = await fetch(' {
    signal: controller.signal
  });
} finally {
  clearTimeout(timeoutId);
}
```

---

## Hedged Requests — The Latency Trick

Instead of waiting for a timeout and then retrying, send a duplicate request to a different instance after a short delay. Use the first response that comes back.

```
t=0ms:   Send request to Instance A
t=50ms:  No response yet → send same request to Instance B (hedge)
t=80ms:  Instance B responds → use this response, cancel A
```

**When to use:** Read-only operations against stateful stores where tail latency is high but median latency is low. Google uses this extensively for Bigtable and Spanner reads.

**When NOT to use:**
- Write operations (unless idempotent)
- When extra load is a concern (hedging doubles request volume in the worst case)
- Against rate-limited APIs

---

## Anti-Patterns — What Will Bite You

### 1. Retrying Everything

```java
// Bad — retries 400 Bad Request, 401 Unauthorized, and even NPEs
catch (Exception e) {
    retry();
}
```

Only retry on transient, retryable errors. Build an explicit allowlist of retryable status codes and exception types.

### 2. No Backoff

```java
// Bad — hammers the server as fast as possible
while (!success && attempts < 5) {
    success = callService();
}
```

Always add delay between retries. Exponential backoff with jitter is the default.

### 3. No Max Retries

```java
// Bad — retries forever
while (!success) {
    success = callService();
    Thread.sleep(1000);
}
```

Always cap retry attempts. Always have a total timeout.

### 4. Retrying Non-Idempotent Operations Without Protection

```java
// Bad — if the first call succeeded but the response was lost, this creates a duplicate order
try {
    createOrder(order);
} catch (TimeoutException e) {
    createOrder(order); // retry → duplicate order
}
```

Use idempotency keys for any write operation you plan to retry.

### 5. Retrying at Every Layer

The retry amplification problem described above. If Service A retries 3x, Service B retries 3x, and Service C retries 3x, a single failed request generates up to 27 attempts at the leaf service.

### 6. Same Timeout for All Operations

```java
// Bad — a health check and a batch export share the same 30s timeout
HttpClient.builder().readTimeout(Duration.ofSeconds(30)).build();
```

Tune timeouts per operation. A health check should timeout in 2 seconds. A report generation call might legitimately need 30.

### 7. Ignoring `Retry-After` Headers

```java
// Bad — got a 429, retrying immediately
if (response.status == 429) {
    retry(); // the server told you to wait!
}
```

When the server returns `Retry-After`, respect it. The server knows its own capacity better than your client does.

### 8. Retrying on Circuit Breaker Open

```java
// Bad — circuit is open, but we retry 3 times anyway (3 instant rejections)
retry(3, () -> circuitBreaker.call(() -> httpClient.call(request)));
```

When the circuit is open, fail fast. Don't waste retry attempts on a service you already know is down.

### 9. Logging Every Retry as an Error

```java
// Bad — 3 retries × 1000 concurrent requests = 3000 ERROR logs per second
catch (Exception e) {
    log.error("Request failed, retrying", e);
    retry();
}
```

Log the first attempt failure at WARN (it's transient, you're handling it). Log the final failure (all retries exhausted) at ERROR. Don't log intermediate retries at ERROR — it generates noise and triggers false alerts.

### 10. Retrying Database Writes on Deadlocks Without Understanding Isolation

```java
// Dangerous — may produce incorrect results depending on isolation level
catch (DeadlockLoserDataAccessException e) {
    retry(); // what state was the transaction in? what did we read before the deadlock?
}
```

Deadlock retries require re-reading all data from scratch inside a new transaction. Re-executing the same SQL statements with stale in-memory state is a data corruption bug.

### 11. Thread Pool Exhaustion from Blocking Retries

```java
// Bad — synchronous retries with sleep block the thread pool
@PostMapping("/orders")
public Response createOrder(OrderRequest req) {
    // 3 retries × 4s backoff each = this thread is blocked for ~12 seconds
    // With 200 threads and 100 concurrent requests doing this, pool is starved
    return retryTemplate.execute(ctx -> orderClient.create(req));
}
```

For blocking retries on a shared thread pool, use async retry mechanisms or a separate thread pool to avoid starving the web server.

### 12. Retrying Partial Successes

```
Step 1: Deduct inventory ✅ (succeeded)
Step 2: Charge payment ❌ (failed)
→ Retry the whole operation → deducts inventory AGAIN
```

Break multi-step operations into individually idempotent steps, or use the saga pattern with compensating transactions. Never blindly retry a partially-completed workflow.

### 13. Hardcoding Retry Configuration

```java
// Bad — changing retry count requires a code change, build, and deploy
private static final int MAX_RETRIES = 3;
private static final long BACKOFF_MS = 1000;
```

Make retry parameters configurable — via config files, feature flags, or centralized configuration. During an incident, you may need to reduce retries or increase backoff without deploying code.

### 14. Not Testing Retry Behavior

Retries are control flow. Untested control flow is a liability. Test:

- That retryable errors are retried
- That non-retryable errors are NOT retried
- That the max retry count is respected
- That backoff delays are applied (use a mock clock)
- That idempotency keys prevent duplicates
- That the circuit breaker opens after sustained failures
- That the total timeout is enforced

```java
@Test
void shouldRetryOnTransientError() {
    when(client.call(any()))
        .thenThrow(new ServiceUnavailableException())  // attempt 1
        .thenThrow(new ServiceUnavailableException())  // attempt 2
        .thenReturn(successResponse());                // attempt 3

    Response result = retryableClient.call(request);

    assertThat(result.isSuccess()).isTrue();
    verify(client, times(3)).call(any());
}

@Test
void shouldNotRetryOnClientError() {
    when(client.call(any())).thenThrow(new BadRequestException());

    assertThrows(BadRequestException.class, () -> retryableClient.call(request));
    verify(client, times(1)).call(any());
}
```

### 15. Silently Swallowing the Final Failure

```java
// Bad — all retries failed, but the error is swallowed
try {
    retryTemplate.execute(ctx -> service.call());
} catch (Exception e) {
    log.warn("All retries failed, returning default");
    return DEFAULT_VALUE;
}
```

When all retries are exhausted, the failure should be visible — logged at ERROR with full context, surfaced to the caller with an appropriate status code, and ideally tracked as a metric. Silently returning a default value hides systemic issues.

---

## Retry Decision Flowchart

```
Request failed
    │
    ├─ Is the error retryable? (5xx, timeout, connection error)
    │   ├─ No → Fail immediately, return error to caller
    │   └─ Yes ↓
    │
    ├─ Is the operation idempotent (or protected by an idempotency key)?
    │   ├─ No → Fail immediately (unsafe to retry)
    │   └─ Yes ↓
    │
    ├─ Have we exceeded max retry attempts?
    │   ├─ Yes → Fail, log at ERROR, return error
    │   └─ No ↓
    │
    ├─ Has the total timeout / deadline expired?
    │   ├─ Yes → Fail, log at ERROR, return error
    │   └─ No ↓
    │
    ├─ Is the circuit breaker open?
    │   ├─ Yes → Fail fast, don't waste a retry
    │   └─ No ↓
    │
    ├─ Does the response include Retry-After?
    │   ├─ Yes → Wait that long
    │   └─ No → Calculate exponential backoff + jitter
    │
    └─ Retry
```

---

## Related Topics

- Graceful Degradation — circuit breaker fundamentals, timeout budgets, fallback strategies
- Rate Limiting — handling `429 Too Many Requests`, rate limit algorithms, `Retry-After` headers

---

## Quick Reference Checklist

- [ ] **Retryable errors are explicitly defined** — allowlist of status codes and exception types, not a blanket catch-all
- [ ] **Non-retryable errors fail immediately** — 400, 401, 403, 404, 409, validation errors, deserialization errors
- [ ] **Exponential backoff with jitter** is the default backoff strategy for HTTP calls
- [ ] **Max retry count** is set for every retry loop (2–3 for user-facing, 3–5 for service-to-service)
- [ ] **Total timeout** caps the wall-clock time across all attempts
- [ ] **Idempotency keys** protect all retried write operations
- [ ] **`Retry-After` headers** are respected when returned by upstream services
- [ ] **Circuit breakers** wrap downstream calls to prevent retry amplification during sustained outages
- [ ] **Retries only happen at one layer** — edge retry preferred, intermediate services fail fast
- [ ] **Retry parameters are configurable** — not hardcoded constants
- [ ] **DLQ exists** for async/message retry flows with alerting on depth
- [ ] **Poison pill detection** skips retries for messages that will never succeed
- [ ] **Retry behavior is tested** — retryable errors, non-retryable errors, max attempts, backoff, idempotency
- [ ] **Intermediate retries log at WARN**, final failure logs at ERROR with full context
- [ ] **Timeouts are set per-operation** — connection timeout AND read timeout, both explicit
- [ ] **Timeout budget** accounts for all downstream calls within the SLA
- [ ] **No retry amplification** — verified that the retry chain doesn't multiply geometrically across service layers
