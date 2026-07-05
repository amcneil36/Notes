# Graceful Degradation Best Practices

Every system has failure modes. The question isn't whether your dependencies will go down — it's what your users experience when they do. A system that returns a 500 error because a recommendation engine is slow is poorly designed. A system that shows the page without recommendations and loads them later is gracefully degraded. The difference isn't complexity — it's intentional design.

**The goal: when something breaks, break less. Preserve core functionality even when supporting systems fail.**

---

## The Degradation Hierarchy

Not all functionality is equally important. The first step in graceful degradation is knowing what to sacrifice.

```
Priority 1 — Core functionality (NEVER sacrifice)
  └── Checkout can process a payment
  └── Users can log in
  └── Critical data is not lost or corrupted

Priority 2 — Important functionality (Degrade with notification)
  └── Search returns results (maybe slower, maybe fewer)
  └── Order history loads
  └── Notifications are delivered

Priority 3 — Enhancement functionality (Silently degrade)
  └── Recommendations are shown
  └── Analytics events are tracked
  └── Personalization is applied

Priority 4 — Nice-to-have (Drop entirely under pressure)
  └── A/B test variations
  └── Real-time chat widget
  └── Animated transitions
```

**Exercise for every service:** List your features in these tiers. If you can't agree on what's Priority 1 vs. Priority 3, you're not ready for your next outage.

---

## Circuit Breakers

A circuit breaker detects when a downstream dependency is failing and stops sending requests to it — protecting both your service (from wasting resources on doomed calls) and the downstream service (from being overwhelmed while it's trying to recover).

### Circuit Breaker States

```
CLOSED (normal) ─── failures exceed threshold ───→ OPEN (failing fast)
     ↑                                                    │
     │                                          wait timeout expires
     │                                                    ↓
     └────── probe request succeeds ────── HALF-OPEN (testing)
                                                    │
                                          probe fails → back to OPEN
```

| State | Behavior |
|---|---|
| **Closed** | Requests flow normally. Failures are counted. |
| **Open** | All requests fail immediately without attempting. Returns a fallback or error. |
| **Half-Open** | One probe request is allowed through. If it succeeds → Closed. If it fails → Open. |

### Configuration Guidelines

| Parameter | Recommended Starting Point | Notes |
|---|---|---|
| Failure rate threshold | 50% | Open circuit if half of recent requests fail |
| Sliding window size | 10–20 requests | Too small = flappy. Too large = slow to open. |
| Wait duration (open → half-open) | 30–60 seconds | How long to wait before probing |
| Permitted calls in half-open | 3–5 | How many probe requests to test with |
| Slow call threshold | 3–5 seconds | Calls exceeding this count as failures |

**Java (Resilience4j):**
```java
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)
    .slidingWindowSize(10)
    .waitDurationInOpenState(Duration.ofSeconds(30))
    .permittedNumberOfCallsInHalfOpenState(3)
    .slowCallDurationThreshold(Duration.ofSeconds(5))
    .slowCallRateThreshold(80)
    .build();

CircuitBreaker breaker = CircuitBreaker.of("orderService", config);

Supplier<OrderResponse> decorated = CircuitBreaker
    .decorateSupplier(breaker, () -> orderClient.getOrder(orderId));
```

See Retries for how retries interact with circuit breakers (composition order, retry amplification prevention).

### Fallback Strategies When the Circuit Opens

| Strategy | How It Works | When to Use |
|----------|-------------|-------------|
| **Cached response** | Return the last known good response from cache | Read operations where slightly stale data is acceptable |
| **Default value** | Return a hardcoded or configured default | Feature flags, configuration, non-critical display data |
| **Reduced functionality** | Skip the failed component, serve the page without it | Recommendations, personalization, analytics |
| **Alternative service** | Route to a backup or secondary provider | Payment processors, CDNs, DNS providers |
| **Queued for later** | Accept the request, queue it, process when dependency recovers | Write operations that don't need immediate confirmation |
| **Honest error** | Tell the user something is temporarily unavailable | When no fallback exists and pretending would be worse |

```
// Pseudocode — recommendation service with circuit breaker + fallback
function getProductPage(productId) {
    product = productService.get(productId)  // Priority 1 — no fallback, must succeed

    try {
        recommendations = recommendationCircuitBreaker.call(
            () -> recommendationService.get(productId)
        )
    } catch (CircuitOpenException e) {
        recommendations = cache.get("recommendations:" + productId)  // stale is fine
        if (recommendations == null) {
            recommendations = []  // show nothing rather than error
        }
    }

    try {
        reviews = reviewCircuitBreaker.call(
            () -> reviewService.get(productId)
        )
    } catch (CircuitOpenException e) {
        reviews = { available: false, message: "Reviews temporarily unavailable" }
    }

    return renderPage(product, recommendations, reviews)
}
```

The key insight: **different dependencies get different fallback strategies** based on their priority tier.

---

## Bulkheads

A bulkhead isolates failures so that a problem in one part of the system doesn't sink the whole ship. The metaphor comes from ship design — watertight compartments prevent a single hull breach from flooding the entire vessel.

### Thread Pool Bulkheads

Without bulkheads, all outbound calls share the same thread pool. If one downstream service hangs, it exhausts the entire pool and every other service call starves.

```
Without bulkhead:
  Shared thread pool (200 threads)
  ├── Payment service calls (normally 20 threads)
  ├── Inventory service calls (normally 20 threads)
  ├── Recommendation calls (normally 10 threads)
  └── Available for other work (150 threads)

  Payment service hangs → 200 threads stuck waiting → ALL services blocked

With bulkhead:
  Payment pool (30 threads max)      ← hangs, only 30 threads affected
  Inventory pool (30 threads max)    ← still working fine
  Recommendation pool (15 threads max) ← still working fine
  General pool (125 threads)         ← still available
```

### Connection Pool Bulkheads

Same principle applied to database and HTTP connections. A runaway query pattern in one module shouldn't exhaust connections for the entire application.

```
// Separate connection pools per downstream
payment-db-pool:     max=20, timeout=5s
inventory-db-pool:   max=20, timeout=5s
analytics-db-pool:   max=10, timeout=10s
```

### Bulkhead Sizing

| Factor | Guideline |
|--------|-----------|
| Normal concurrency | Size the bulkhead for peak normal usage + 50% headroom |
| Timeout per call | Shorter timeouts on lower-priority bulkheads |
| Queue depth | Small or zero — queuing behind a stuck service just delays the inevitable |
| Monitoring | Alert when a bulkhead hits capacity — it means traffic or latency changed |

The trade-off: bulkheads waste resources during normal operation (some threads/connections sit idle in each pool). This is intentional — you're trading peak efficiency for fault isolation.

---

## Load Shedding

When your service is overwhelmed, it's better to reject some requests cleanly than to serve all of them poorly. Load shedding is the deliberate decision to drop low-priority work so that high-priority work can still succeed.

### When to Shed Load

| Signal | Action |
|--------|--------|
| CPU consistently above 85% | Start rejecting non-critical requests |
| Request queue depth exceeds threshold | Reject new requests rather than queueing indefinitely |
| Response latency exceeds SLA | Shed lower-priority traffic to recover latency for higher-priority |
| Upstream is sending more than agreed rate | Reject excess with 429 and Retry-After |

### What to Shed First

```
Shed order (first to drop → last to drop):
  1. Health check polling from non-critical monitors
  2. Analytics and telemetry collection
  3. Recommendation and personalization requests
  4. Search queries (return cached/limited results instead)
  5. Read operations (serve from cache if possible)
  6. Write operations (last resort — these represent real user actions)
```

### How to Shed

**Return 503 with Retry-After**, not a timeout. A 503 tells the client "I'm alive but overloaded — try later." A timeout wastes the client's time waiting.

```
HTTP/1.1 503 Service Unavailable
Retry-After: 30
Content-Type: application/json

{ "error": "Service temporarily overloaded", "retry_after_seconds": 30 }
```

**Shed early, not late.** If you know you're going to reject a request, do it at the edge (load balancer, API gateway) before it consumes resources deeper in the stack.

### Priority-Based Request Classification

Tag requests with priority at the edge so downstream services can make shedding decisions:

| Priority | Examples | Shedding Behavior |
|----------|---------|-------------------|
| Critical | Payment processing, authentication | Never shed |
| High | Core reads/writes (order placement, search) | Shed only under extreme load |
| Medium | Secondary features (recommendations, reviews) | Shed under moderate load |
| Low | Analytics, prefetching, background sync | Shed first |

---

## Timeouts as Degradation

Timeouts aren't just error handling — they're a degradation tool. A well-tuned timeout lets you fail fast and fall back, rather than hanging indefinitely.

### Cascading Timeout Budgets

In a call chain A → B → C, the timeout at each layer should be shorter than the caller's timeout.

```
User-facing request: 3s total budget
├── Service A timeout: 2.5s (leaves 500ms for processing)
│   ├── Service B call: 1.5s timeout
│   │   └── Service C call: 800ms timeout
│   └── Cache fallback: 100ms timeout
└── Response to user
```

If Service C is slow, B times out at 1.5s, falls back to a degraded response, and A still returns to the user within 3s. Without budget discipline, C hanging for 30s cascades up to the user.

### Aggressive Timeouts for Non-Critical Calls

Lower-priority dependencies should have **shorter** timeouts than higher-priority ones. If the recommendation service doesn't respond in 200ms, skip it — don't let it jeopardize the page load.

```
// Critical dependency — generous timeout
product = productService.get(id, timeout=2000ms)

// Enhancement — aggressive timeout, with fallback
try {
    recommendations = recommendationService.get(id, timeout=200ms)
} catch (TimeoutException e) {
    recommendations = fallbackRecommendations()
}
```

---

## Feature Degradation Modes

Pre-define degradation modes for your service so you can switch quickly during incidents.

### Mode Definitions

| Mode | Description | Trigger |
|------|-------------|---------|
| **Normal** | All features enabled, all dependencies healthy | Default state |
| **Degraded** | Non-critical features disabled, cached/default fallbacks active | One or more non-critical dependencies down |
| **Emergency** | Only core functionality active, all enhancements stripped | Critical dependency degraded, high load |
| **Maintenance** | Read-only mode or static fallback page | Planned maintenance, data migration |

### Implementation

Use feature flags or configuration to toggle degradation modes without deploying code:

```
degradation_mode = config.get("degradation_mode", default="normal")

if (degradation_mode == "emergency") {
    // Skip non-critical features
    recommendations = []
    reviews = { available: false }
    analytics.disable()
    personalization.disable()
}
```

**Pre-build the degradation path.** If you've never tested your system in degraded mode, you don't have a degraded mode — you have a hope.

---

## Fallback Design Patterns

### Pattern 1: Cache Fallback

When the primary source is unavailable, serve from cache.

```
try {
    data = primaryService.get(key)
    cache.set(key, data, ttl=1h)
    return data
} catch (ServiceUnavailableException e) {
    cached = cache.get(key)
    if (cached != null) {
        return cached  // stale but better than nothing
    }
    throw e  // no cache available, propagate the error
}
```

**Important:** The cache TTL for fallback purposes should be longer than the cache TTL for freshness. You might normally cache for 5 minutes, but keep a "fallback copy" for 1 hour.

### Pattern 2: Static Defaults

When a dynamic service is down, return a sensible static default.

```
try {
    config = configService.getFeatureFlags()
} catch (ServiceUnavailableException e) {
    config = HARDCODED_SAFE_DEFAULTS  // conservative defaults that disable risky features
}
```

**Design safe defaults:** When in doubt, the default should disable the feature, not enable it. A missing feature is better than a misbehaving feature.

### Pattern 3: Queue and Process Later

For write operations, accept the request and process it asynchronously when the dependency recovers.

```
try {
    notificationService.send(notification)
} catch (ServiceUnavailableException e) {
    retryQueue.enqueue(notification)  // process when service is back
    log.warn("Notification queued for retry")
}
```

**Only works when:** The user doesn't need immediate confirmation of the downstream action, and you have durability guarantees on the queue.

### Pattern 4: Reduced Fidelity

Serve a simpler version of the response.

```
try {
    searchResults = searchService.fullSearch(query)  // ranked, personalized, filtered
} catch (ServiceDegradedException e) {
    searchResults = searchService.basicSearch(query)  // simple keyword match, no ranking
}
```

This works for search, recommendations, dashboards, reports — anywhere a "good enough" answer is better than no answer.

### Pattern 5: Client-Side Resilience

Return enough information for the client to degrade gracefully on its own.

```
{
    "product": { ... },
    "recommendations": null,
    "recommendations_status": "unavailable",
    "reviews": { "count": 42, "details_available": false }
}
```

The client decides how to render based on what data is available, rather than failing the entire page.

---

## Testing Degradation

Degradation logic that hasn't been tested is decoration, not protection.

### What to Test

| Test | What It Validates |
|------|------------------|
| Kill a non-critical dependency | Service continues with degraded response, no errors to user |
| Kill a critical dependency | Service returns appropriate error, doesn't cascade |
| Slow down a dependency (add latency) | Timeouts fire, fallbacks activate, response time stays within SLA |
| Overwhelm the service with traffic | Load shedding activates, critical requests still served |
| Fill a bulkhead (exhaust thread pool for one dependency) | Other dependencies are unaffected |
| Flip to emergency degradation mode | Only core features active, response times improve |
| Bring a dependency back after failure | Circuit breaker transitions to half-open, then closed. System recovers. |

### Chaos Engineering (Lightweight Version)

You don't need a full Chaos Monkey setup to test degradation. Start simple:

1. **Dependency kill test:** During a staging load test, kill one dependency at a time. Verify the degradation behavior matches your design.
2. **Latency injection:** Add artificial latency to one dependency. Verify timeouts and fallbacks activate.
3. **Gradual load increase:** Ramp traffic until load shedding activates. Verify that priority-based shedding works as expected.
4. **Game days:** Schedule a team exercise where you simulate an outage in production-like conditions. Practice the response, not just the automation.

---

## Common Pitfalls

| Pitfall | Problem | Fix |
|---------|---------|-----|
| **No fallback defined** | Dependency goes down, service returns 500. | Define a fallback for every external call. Even "return empty" is a fallback. |
| **Fallback that calls the same failing dependency** | Circuit opens for Service A, fallback also calls Service A. | Fallbacks must use a different path — cache, default, alternative service. |
| **Fallback that's never tested** | You think you have degradation, but the fallback code has a bug. | Test fallback paths as part of regular testing. Run dependency-down tests in CI. |
| **Treating all errors the same** | Timeout and 400 Bad Request get the same circuit breaker treatment. | Only count server-side failures (5xx, timeouts, connection errors) toward the circuit breaker. Client errors (4xx) are not the server's fault. |
| **No monitoring of degraded state** | Service is running in degraded mode for weeks without anyone noticing. | Alert when a circuit opens. Dashboard showing current degradation state. |
| **Cascading failures through shared resources** | Shared thread pool, shared DB connection pool, shared cache — one failure drains them all. | Bulkheads. Isolate resources per dependency or per priority tier. |
| **Load shedding without priority** | All requests equally likely to be rejected. Critical operations fail alongside analytics. | Classify requests by priority. Shed low-priority first. |
| **Degradation mode that's never turned off** | Emergency mode activated during an incident, forgotten, runs for months. | Auto-recovery through circuit breakers. Manual overrides should have TTLs and alerts. |
| **Optimistic fallback data** | Fallback shows "everything is fine" when it isn't. Users make decisions based on stale data. | Make degradation visible when it matters. "Prices as of 10 minutes ago" is honest. "Price: $0.00" is a lie. |

---

## Related Topics

- Retries — retry strategies, idempotency, retry amplification prevention
- Caching — cache strategies, TTL design, cache key patterns

---

## Quick Reference Checklist

- [ ] **Features classified by priority tier** — know what to sacrifice before the incident
- [ ] **Circuit breakers** on every external dependency with defined fallback behavior
- [ ] **Bulkheads** isolate thread pools and connection pools per dependency
- [ ] **Timeout budgets** cascade correctly through the call chain — each layer shorter than the caller
- [ ] **Load shedding** activated by CPU, queue depth, or latency — sheds low-priority first
- [ ] **Fallback defined** for every external call — cache, default, queue, or reduced fidelity
- [ ] **Degradation modes** (normal, degraded, emergency) switchable via config, not code deploy
- [ ] **Safe defaults** — when in doubt, disable the feature, don't enable it
- [ ] **Degradation is visible** — monitoring and alerts show when a circuit is open or a fallback is active
- [ ] **Tested** — dependency failures, latency injection, and load shedding validated in staging
- [ ] **Auto-recovery** — circuit breakers close automatically, manual overrides have TTLs
- [ ] **503 with Retry-After** — shed cleanly, don't let clients hang on a timeout
