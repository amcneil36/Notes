# Latency Best Practices

Latency is the one metric that users feel directly. They don't feel CPU utilization. They don't feel memory pressure. They feel the spinner. And the threshold between "fast" and "slow" is brutally tight — studies consistently show that anything over 200ms feels sluggish for interactive operations, and anything over 1 second feels broken.

But latency is also the most misunderstood metric. Teams obsess over averages (which hide problems), ignore tail latency (which catches real users), measure in the wrong places (which gives false confidence), and optimize the wrong things (which wastes engineering time while the actual bottleneck sits untouched).

The goal: **measure latency where it matters, understand what the numbers actually mean, and reduce it by fixing the right things in the right order.**

---

## What Latency Actually Means — And Why Your Measurement Is Probably Wrong

Latency is the time elapsed between a request being sent and the response being fully received. Simple in concept, deceptively complex in practice — because *where* you measure determines *what* you see.

### The Measurement Points

```
Client               Network        Load Balancer      Service              Database
  │                    │                │                 │                    │
  ├── Client sends ────┤                │                 │                    │
  │                    ├── In flight ───┤                 │                    │
  │                    │                ├── Queued ───────┤                    │
  │                    │                │                 ├── Processing ──────┤
  │                    │                │                 │                    ├── Query ──┤
  │                    │                │                 │                    │           │
  │                    │                │                 ├── Response built ──┤           │
  │                    │                ├── Forwarded ────┤                    │           │
  │                    ├── In flight ───┤                 │                    │           │
  ├── Client receives ─┤                │                 │                    │           │
  │                    │                │                 │                    │           │
  └────────────── User-perceived latency ──────────────────────────────────────┘
                       └───── Network latency ─────┘
                                        └── LB queueing ┘
                                                         └── Server processing time ─────┘
                                                                              └─ DB time ┘
```

### Where Teams Typically Measure vs. Where They Should

| Measurement Point | What It Captures | What It Misses |
|---|---|---|
| **Server-side handler timer** | Processing time within your code | Network latency, LB queueing, serialization, TLS handshake |
| **Load balancer metrics** | Time from LB to response | Client-side network latency, DNS, connection setup |
| **Client-side SDK timer** | Full round-trip from caller | Nothing — this is the closest to reality for service-to-service |
| **Real User Monitoring (RUM)** | True end-user experience including rendering | Only available for browser/mobile clients |
| **Synthetic monitoring** | Controlled, repeatable measurement | Doesn't reflect real user conditions (geography, device, network) |

**The rule:** measure at the outermost point you control. Server-side timers are useful for debugging, but they'll tell you the service took 15ms while the user experienced 800ms because of network, DNS, TLS, and serialization overhead.

For service-to-service calls, the *caller's* measurement is the truth. Your service thinks it responded in 20ms. The caller measured 200ms. The difference is real, and it's your problem to investigate (serialization? large payloads? connection pool exhaustion on the caller side? network?).

---

## Percentiles — The Only Way to Talk About Latency

### Why Averages Lie

Average latency is the single most misleading metric in distributed systems. Here's why:

```
100 requests:
  95 complete in 10ms
  4 complete in 200ms
  1 completes in 5,000ms

Average: 57ms  ← looks fine
Median (P50): 10ms ← most users are happy
P95: 200ms ← 1 in 20 users waits 200ms
P99: 5,000ms ← 1 in 100 users waits 5 SECONDS
```

The average tells you everything is fine. The P99 tells you a real human is staring at a spinner for 5 seconds on every hundredth request. At 10,000 requests per minute, that's 100 users per minute having a terrible experience — and your dashboard says "57ms average, looks good."

### Which Percentiles to Track

| Percentile | What It Tells You | When to Alert | SLO Relevance |
|---|---|---|---|
| **P50 (median)** | The typical experience for most users | Rarely — shifts here indicate systemic changes | Baseline understanding |
| **P90** | Where things start getting slow | Useful for capacity planning | Some SLOs target this |
| **P95** | The experience for 1 in 20 users | Good alert threshold for degradation | Common SLO target |
| **P99** | The experience for 1 in 100 users | Alert on sustained spikes | Most meaningful SLO target |
| **P99.9** | Extreme tail — often infrastructure artifacts | Generally too noisy for alerting | Useful for debugging, not SLOs |
| **Max** | Single worst request | Never alert on this | Debugging only — often GC pauses or cold starts |

### The Percentile Rules

1. **Report P50, P95, and P99 at minimum.** P50 for baseline, P95 for degradation, P99 for tail.
2. **Set SLOs on P99, not P50.** "P50 < 100ms" is easy to hit and tells you nothing useful. "P99 < 500ms" is meaningful.
3. **Never average percentiles across instances.** The P99 across 10 instances is NOT the average of their individual P99s. Aggregate the raw histograms, then compute percentiles.
4. **Never average percentiles across time windows.** The P99 for the hour is not the average of the 12 five-minute P99s. This is mathematically wrong and will undercount your worst latencies.

### Histogram vs. Summary — How Percentiles Are Computed

There are two approaches, and the distinction matters:

| Approach | How It Works | Aggregatable? | Accuracy |
|---|---|---|---|
| **Histogram** (bucketed) | Counts requests falling into predefined latency buckets (0-10ms, 10-50ms, 50-100ms, ...) | Yes — you can sum histograms across instances | Approximate — limited by bucket boundaries |
| **Summary** (client-side) | Computes exact percentiles in the client process | No — you cannot aggregate percentile values | Exact for that instance |

**Use histograms.** They're aggregatable across instances and time windows. Summaries give exact values for individual instances but cannot be combined — and in a multi-instance deployment, per-instance percentiles are nearly useless.

Choose bucket boundaries that match your SLO thresholds. If your SLO is "P99 < 500ms," have buckets at 100ms, 250ms, 500ms, 1000ms, 2500ms, 5000ms. If your smallest bucket is 1 second, you can't tell the difference between a 50ms response and a 950ms response.

---

## HTTP API Latency — What to Track

### The Essential Metrics

| Metric | Granularity | Why |
|---|---|---|
| **Request duration** (P50/P95/P99) | Per endpoint, per method, per status code class | The core metric — how long requests take |
| **Time to first byte (TTFB)** | Per endpoint | How long until the response starts — important for streaming and large payloads |
| **Upstream dependency latency** | Per dependency, per operation | How much of your latency is your code vs. things you call |
| **Connection pool wait time** | Per connection pool (DB, HTTP client) | Hidden latency from waiting for a connection — often the real bottleneck |
| **Queue/thread pool wait time** | Per pool | Time spent waiting for a thread to become available before processing even starts |
| **Serialization/deserialization time** | Per endpoint (if significant) | Large payloads can make ser/deser a meaningful chunk of total latency |
| **GC pause time** | Per instance | Garbage collection can inject latency spikes that look like application slowness |

### Dimensions / Labels

Tag your latency metrics with dimensions that let you slice the data:

| Dimension | Example Values | Why |
|---|---|---|
| `endpoint` / `route` | `/api/v1/orders`, `/api/v1/users/{id}` | Different endpoints have wildly different latency profiles |
| `method` | `GET`, `POST`, `PUT` | Reads and writes often differ by 10x |
| `status_class` | `2xx`, `4xx`, `5xx` | Error responses are often faster (short-circuit) or slower (timeouts) — mixing them distorts the picture |
| `region` / `datacenter` | `us-east`, `us-west` | Cross-region latency differences are significant |
| `version` / `canary` | `v2.3.1`, `canary` | Compare latency between deployments |
| `tenant` / `customer_tier` | `enterprise`, `free` | Multi-tenant services often have different profiles per tenant |

**Cardinality warning:** don't use unbounded values as dimensions (user ID, order ID, request ID). High-cardinality labels explode your metrics storage and make aggregation meaningless. If you need per-request latency data, that's what traces are for — not metrics.

### What "Good" Looks Like

These are general guidelines, not absolutes — they depend heavily on what the endpoint does:

| Operation Type | Target P50 | Target P99 | Notes |
|---|---|---|---|
| Simple CRUD read (cached) | < 10ms | < 50ms | If it's slower, check your cache |
| Simple CRUD read (DB) | < 20ms | < 100ms | Single-row lookups should be fast |
| Complex query / aggregation | < 100ms | < 500ms | Consider async or pagination if consistently slow |
| Write with validation | < 50ms | < 200ms | Synchronous validation should be fast; offload heavy processing |
| Batch operation | < 500ms | < 2s | Consider async for anything larger |
| External API call | < 200ms | < 1s | You're at the mercy of the external service; cache aggressively |
| File upload/download | Varies | Varies | Measure TTFB separately from total transfer time |

---

## Kafka Consumer Latency — What to Track

Kafka consumer latency is fundamentally different from HTTP latency. It's not request-response — it's a pipeline, and "latency" encompasses everything from when the message was produced to when your consumer has finished processing it.

### The Kafka Latency Timeline

```
Producer publishes
    │
    ├── Producer batching delay (linger.ms, batch.size)
    │
    ├── Network: producer → broker
    │
    ├── Broker writes to partition (acks=all waits for replicas)
    │
    ├── Message sits in partition waiting to be consumed (CONSUMER LAG)
    │
    ├── Consumer polls and fetches the message
    │
    ├── Deserialization
    │
    ├── Consumer processing (your business logic)
    │
    ├── Offset commit
    │
    └── End-to-end latency = all of the above
```

### Essential Kafka Latency Metrics

| Metric | What It Measures | Why It Matters |
|---|---|---|
| **Consumer lag** (offsets behind) | How many messages are waiting to be consumed | The single most important Kafka latency metric — if lag is growing, you're falling behind |
| **Consumer lag in time** | How old the oldest unconsumed message is | More intuitive than offset lag — "we're 45 seconds behind" vs. "we're 12,000 offsets behind" |
| **Poll-to-process duration** | Time from `poll()` return to processing complete | Your consumer's actual processing speed |
| **End-to-end latency** | Time from producer `send()` to consumer processing complete | The true latency of your pipeline |
| **Commit latency** | Time to commit offsets | High commit latency can indicate broker issues |
| **Rebalance frequency and duration** | How often consumers rebalance and how long it takes | Rebalances pause all consumption — frequent rebalances kill latency |
| **Fetch latency** | Time for the consumer to fetch a batch from the broker | Network or broker performance indicator |
| **Records per poll** | How many records each `poll()` returns | Too few means polling overhead dominates; too many means processing batches are too large |

### Consumer Lag — The Metric That Matters Most

Consumer lag is the Kafka equivalent of queue depth. It tells you whether your consumers can keep up with the producers. Everything else is secondary.

**Healthy lag behavior:**
- Lag hovers near zero during normal traffic
- Lag spikes briefly during traffic bursts, then recovers
- Recovery time after a spike is predictable and within SLA

**Unhealthy lag behavior:**
- Lag grows monotonically — consumers cannot keep up
- Lag never recovers after spikes — processing is slower than production rate
- Lag oscillates wildly — frequent rebalances or unstable processing

**How to measure lag:**

Consumer lag is the difference between the latest offset in the partition and the consumer group's committed offset. Most monitoring tools compute this automatically. If you're tracking it yourself:

```
lag = (latest_offset_in_partition) - (consumer_group_committed_offset)
```

Convert offset lag to time lag by estimating the production rate:

```
time_lag ≈ offset_lag / messages_per_second
```

Or better, embed a timestamp in each message and compute the difference between the message timestamp and the current wall clock when you process it.

### Tuning Consumer Performance

The following parameters directly affect consumer latency. There's no single "right" configuration — it depends on your message volume, processing cost, and latency requirements.

| Parameter | What It Controls | Latency Impact |
|---|---|---|
| `max.poll.records` | Max messages returned per `poll()` | Higher = better throughput but higher per-batch latency. Lower = lower latency per message but more polling overhead |
| `fetch.min.bytes` | Minimum data the broker should accumulate before responding | Higher = fewer fetches, better throughput, higher latency. Set to 1 for lowest latency |
| `fetch.max.wait.ms` | Max time the broker waits to accumulate `fetch.min.bytes` | Caps the worst-case fetch delay. Lower = lower latency, more broker load |
| `max.poll.interval.ms` | Max time between `poll()` calls before the consumer is considered dead | If processing takes longer than this, you get a rebalance — set it higher than your worst-case batch processing time |
| `session.timeout.ms` | Heartbeat-based liveness timeout | Too low = false rebalances. Too high = slow detection of dead consumers |
| `enable.auto.commit` / commit interval | How frequently offsets are committed | Auto-commit at long intervals can cause reprocessing after a crash — but committing too frequently adds overhead |

**The throughput vs. latency tradeoff:**

- For **lowest latency**: `fetch.min.bytes=1`, `fetch.max.wait.ms=0-100`, `max.poll.records` tuned to what you can process in <100ms
- For **highest throughput**: `fetch.min.bytes=65536+`, `fetch.max.wait.ms=500`, `max.poll.records` set high, process in batches
- For **balanced**: `fetch.min.bytes=1`, `fetch.max.wait.ms=100-500`, `max.poll.records` tuned to 100-500, batch database writes

### Partition Count and Parallelism

The maximum consumer parallelism in Kafka equals the number of partitions. If you have 8 partitions, you can have at most 8 consumers in a consumer group. Adding a 9th consumer does nothing — it sits idle.

**When to increase partitions:**
- Consumer lag is growing and you've already optimized processing speed
- Each consumer is CPU/IO-bound processing its current load
- You have headroom for more consumer instances

**When NOT to increase partitions:**
- The bottleneck is a downstream dependency (DB, API), not consumer processing — more partitions just increases load on the bottleneck
- Message ordering within a key matters and you have more partitions than distinct keys
- You haven't tried optimizing the consumer itself first (batching writes, async processing, caching)

### Rebalance Storms — The Silent Latency Killer

Every consumer group rebalance pauses ALL consumers in the group while partitions are reassigned. During a rebalance, latency goes to infinity — no messages are processed.

Common causes of excessive rebalances:
- Consumers taking too long to process a batch (exceeding `max.poll.interval.ms`)
- Frequent deployments without graceful shutdown
- Unstable network causing heartbeat failures
- Consumer instances scaling up and down too aggressively

Mitigation:
- Use static group membership (`group.instance.id`) to survive brief restarts without triggering rebalance
- Implement graceful shutdown — stop polling, process remaining records, commit offsets, then leave the group
- Set `max.poll.interval.ms` high enough to accommodate your worst-case batch
- Use cooperative rebalancing (`CooperativeStickyAssignor`) instead of eager rebalancing — only the affected partitions pause, not the entire group

---

## Latency Budgeting — Decomposing an SLA Across a Call Chain

When your endpoint has a 3-second SLA and it calls four downstream services, each of which calls two more — how do you ensure the SLA is met? You create a latency budget.

### The Concept

A latency budget works like a financial budget: you have a total allocation, and every operation along the call chain spends from that budget. When the budget is spent, you stop spending (fail fast) rather than blowing past the SLA.

```
Total budget: 3000ms (user-facing SLA)

├── Deserialization + validation:   50ms
├── Cache lookup:                   10ms
├── Database query:                200ms
├── Downstream Service A:          500ms
├── Downstream Service B:          300ms
├── Response serialization:         40ms
├── Buffer for overhead/variance:  400ms
└── Remaining:                    1500ms  ← headroom is healthy
```

### Budget Allocation Strategies

| Strategy | How It Works | Best For |
|---|---|---|
| **Static allocation** | Each operation gets a fixed timeout (e.g., Service A gets 500ms) | Predictable call chains with stable latency |
| **Proportional allocation** | Each operation gets a percentage of remaining budget | Variable-depth call chains |
| **Deadline propagation** | Pass an absolute deadline timestamp; each service computes its own remaining budget | Deep call chains across many services |

### Deadline Propagation — The Scalable Approach

Instead of passing relative timeouts, propagate an absolute deadline through the call chain. Each service can compute its remaining budget without knowing anything about the upstream chain.

```
User request arrives at 14:00:00.000, SLA is 3 seconds
→ Deadline: 14:00:03.000

Service A receives request at 14:00:00.050
→ Remaining budget: 2950ms
→ Calls Service B with deadline: 14:00:03.000 (same deadline, just forwarded)

Service B receives request at 14:00:00.120
→ Remaining budget: 2880ms
→ Calls Database with timeout: min(default_db_timeout, 2880ms)
```

**How to propagate deadlines:**

Pass the deadline as a header (commonly `X-Request-Deadline` or `grpc-timeout` in gRPC, which does this natively). Each service:

1. Extracts the deadline from the incoming request
2. Checks if the deadline has already passed — if so, reject immediately (the caller already timed out)
3. Sets its downstream call timeouts to `min(default_timeout, remaining_budget)`
4. Propagates the same deadline to all downstream calls

### Parallel vs. Sequential Calls

How you arrange downstream calls dramatically affects your latency budget:

```
Sequential: Total latency = A + B + C
  A (200ms) → B (300ms) → C (100ms) = 600ms

Parallel: Total latency = max(A, B, C)
  A (200ms) ↘
  B (300ms) → max = 300ms
  C (100ms) ↗
```

If Service A's response is not needed to call Service B, call them in parallel. This is one of the highest-leverage latency optimizations in distributed systems and is often overlooked because sequential code is simpler to write.

### Budget Accounting for Retries

Retries consume budget. If your timeout per attempt is 500ms and you retry 3 times, the worst case is 1500ms — just from one downstream call. Factor this into your budget:

```
Downstream call budget = (per_attempt_timeout × max_attempts) + (total_backoff_time)

Example:
  Per attempt: 500ms
  Max retries: 2 (3 total attempts)
  Backoff: 100ms, 200ms
  Worst case: 500ms + 100ms + 500ms + 200ms + 500ms = 1800ms
```

If 1800ms blows your budget, either reduce the per-attempt timeout, reduce the retry count, or accept that retries may occasionally breach the SLA (and measure how often).

---

## General Strategies for Reducing Latency

These are ordered roughly by ease and impact — start at the top.

### 1. Cache What You Can

The fastest request is the one you don't make. Caching is the single most effective latency reduction technique — in-process caches add less than 1ms, distributed caches (Redis, Memcached) add 1-5ms, and CDN/edge caches serve from the nearest location. Cache effectiveness depends on hit rate: a 95% hit rate is transformational.

See Caching for comprehensive caching strategies, including cache layer selection, invalidation approaches, and cache-aside vs. write-through patterns.

### 2. Reduce Payload Size

Serialization, network transfer, and deserialization all scale with payload size. Smaller payloads = lower latency at every step.

| Technique | Impact | Effort |
|---|---|---|
| **Return only requested fields** (field selection / sparse fieldsets) | High — can eliminate 80%+ of payload | Medium |
| **Paginate list endpoints** | High — prevents unbounded responses | Low |
| **Compress responses** (gzip, brotli) | Medium — 60-90% size reduction for JSON/text | Low (usually framework-level config) |
| **Use a binary serialization format** (Protobuf, Avro, MessagePack) | High for large payloads — 50-80% smaller than JSON | High (requires schema management) |
| **Remove redundant or denormalized data** | Varies | Low |
| **Use ETags / conditional requests** | High for unchanged data — returns 304 with zero body | Low |

### 3. Optimize Database Access

Database calls are the most common source of latency in backend services.

| Technique | When to Use |
|---|---|
| **Add appropriate indexes** | Queries doing full table scans — check `EXPLAIN` output |
| **Use connection pooling** | Always — creating a new connection per request adds 5-50ms |
| **Batch multiple queries into one round trip** | Multiple sequential queries to the same database |
| **Use read replicas for read-heavy workloads** | When write volume is low relative to reads |
| **Denormalize for read performance** | When joins across many tables dominate latency |
| **Avoid N+1 query patterns** | Loading a list, then querying details for each item individually |
| **Use prepared statements** | Repeated queries — saves query parsing and plan generation |
| **Right-size your connection pool** | Too small = requests waiting for connections. Too large = connection overhead on the DB |

### 4. Parallelize Independent Operations

If you need data from three sources and none depends on the other, fetch all three concurrently:

```
Sequential:
  fetch_user(id)         → 50ms
  fetch_orders(userId)   → 80ms
  fetch_preferences(id)  → 30ms
  Total: 160ms

Parallel:
  fetch_user(id)         → 50ms ↘
  fetch_orders(userId)   → 80ms → max = 80ms
  fetch_preferences(id)  → 30ms ↗
  Total: 80ms (50% reduction)
```

This works for HTTP calls, database queries, cache lookups — any I/O operation. The reduction is free in the sense that no individual operation gets faster; you're just not wasting time waiting sequentially.

### 5. Move Work Out of the Request Path

If the user doesn't need to wait for it, don't make them.

| Work | Move To | User Impact |
|---|---|---|
| Sending emails / notifications | Async queue (Kafka, SQS) | User gets response immediately |
| Generating reports / PDFs | Background job | Return a job ID, poll or push when done |
| Audit logging | Async event | No latency contribution |
| Analytics / tracking events | Fire-and-forget or async | No latency contribution |
| Complex validation that can be deferred | Async pipeline with compensation | Optimistic response, correct asynchronously |
| Image/file processing | Background worker | Accept upload, process later |

**The key principle:** every millisecond of work on the request path is a millisecond the user waits. Ruthlessly move anything that isn't required for the response.

### 6. Reduce Network Round Trips

Every network round trip adds at least the network RTT (often 1-5ms within a datacenter, 20-100ms cross-region, 100-300ms cross-continent).

| Technique | Saves |
|---|---|
| **Connection keep-alive / pooling** | TCP handshake (1 RTT) and TLS handshake (1-2 RTTs) per request |
| **HTTP/2 multiplexing** | Head-of-line blocking — multiple requests over one connection |
| **Batch APIs** | Multiple logical operations in one HTTP request |
| **GraphQL or composite endpoints** | Client gets exactly the data it needs in one call instead of N calls |
| **Colocate services** | Cross-region calls become same-datacenter calls |
| **Prefetch / push** | Send data before it's requested (HTTP/2 server push, preloading) |

### 7. Use Appropriate Timeouts

Slow dependencies will make your service slow unless you bound the damage with timeouts. See the retries best practices doc for detailed timeout guidance.

Key latency-related timeout principles:
- **Set timeouts shorter than your SLA.** If your SLA is 3 seconds, a 10-second timeout on a downstream call guarantees SLA breach.
- **Differentiate connection timeout from read timeout.** Connection timeout should be tight (1-3 seconds) — if you can't connect in 3 seconds, something is wrong. Read timeout depends on the operation.
- **Use circuit breakers for sustained slowness.** A dependency that responds in 4999ms (just under your 5-second timeout) is almost as bad as one that doesn't respond at all. Circuit breakers catch this pattern.

### 8. Warm Up Cold Paths

Cold starts and cold caches cause latency spikes that disproportionately hit the first requests after a deployment, scale-up, or restart.

| Cold Path | Warmup Strategy |
|---|---|
| **JVM JIT compilation** | JVM warm-up period with synthetic traffic before receiving real traffic |
| **Empty caches** | Pre-populate cache from a snapshot or dedicated warmup job before routing traffic |
| **Database connection pools** | Initialize connections at startup, not on first request |
| **DNS resolution** | Pre-resolve and cache DNS entries at startup |
| **Class loading / dependency injection** | Eager initialization instead of lazy (trade startup time for first-request latency) |
| **Container cold start** | Keep minimum instances warm; use provisioned concurrency (serverless) |

### 9. Optimize Serialization

For high-throughput services, serialization and deserialization can consume a meaningful percentage of total latency.

| Optimization | Impact |
|---|---|
| Use streaming parsers for large payloads instead of loading into memory | Lower memory, lower latency |
| Avoid serializing fields the caller doesn't need | Less work, smaller payload |
| Pre-compute serialized representations for hot data | Eliminates repeated serialization |
| Consider binary formats (Protobuf, Avro) for internal service-to-service calls | 3-10x faster serialization, 50-80% smaller payloads |
| Cache serialized responses when the underlying data doesn't change | Eliminates serialization entirely |

### 10. Profile Before Optimizing

Never guess where the latency is. Measure first.

Distributed tracing (Jaeger, Zipkin, OpenTelemetry) shows you exactly where time is spent across a call chain. Flame graphs show you where time is spent within a single process. Without these, you're optimizing blind.

Common surprise findings when teams actually profile:
- "The database is slow" → actually, connection pool wait time is 10x the query time
- "The downstream service is slow" → actually, serializing the 2MB response body takes longer than the network transfer
- "We need to optimize the algorithm" → actually, 80% of latency is in the logging framework doing synchronous I/O
- "We need more instances" → actually, one endpoint is single-threaded due to a synchronized block

---

## Tracking Upstream Latency — Measuring Your Dependencies

Your service is only as fast as its slowest dependency on the critical path. Tracking upstream latency lets you distinguish between "our code is slow" and "the thing we call is slow."

### What to Track for Each Dependency

| Metric | Why |
|---|---|
| **Request latency** (P50/P95/P99) per operation | Baseline and trend analysis |
| **Timeout rate** | What percentage of calls exceed your timeout |
| **Error rate** | Errors are often faster than successes (short-circuit) — mixing them distorts latency metrics |
| **Connection pool utilization** | High utilization → requests waiting for connections → hidden latency |
| **Circuit breaker state** | When the circuit opens, you know the dependency is in trouble |
| **Request volume** | Latency often correlates with load — track both to see the relationship |

### Separating Your Latency from Dependency Latency

In your latency metrics, break out time spent waiting on dependencies:

```
Total request latency: 450ms
  ├── Your code (processing, validation): 30ms
  ├── Database call: 200ms
  ├── Cache lookup: 5ms
  ├── Downstream Service X: 180ms
  └── Serialization + overhead: 35ms
```

This decomposition is critical during incidents. "Our P99 spiked to 2 seconds" is a starting point. "Our P99 spiked to 2 seconds, of which 1.8 seconds is Service X" is an answer.

Distributed tracing gives you this automatically. If you don't have tracing, instrument your HTTP client and database client to record latency per call and log or emit it as metrics.

---

## Common Mistakes

### 1. Measuring Average Latency and Calling It Good

The average is dominated by the majority of fast requests and hides the tail. If your average is 50ms but your P99 is 3 seconds, 1% of your users are having a terrible experience. Track and alert on percentiles.

### 2. Measuring Only Server-Side Latency

Your server-side timer says 20ms. The client measured 500ms. The difference is TLS handshake, DNS, connection establishment, load balancer queueing, serialization, and network transfer. Measure at the caller, not just the callee.

### 3. Mixing Error and Success Latency

Errors are often extremely fast (immediate rejection = 2ms) or extremely slow (timeout = 30 seconds). When mixed with success latency, they distort both the median and the tail. Filter latency metrics by status code class: track `2xx` latency, `4xx` latency, and `5xx` latency separately.

### 4. Ignoring Coordinated Omission

If your load test sends a request, waits for the response, then sends the next one — you're measuring the system under artificially low load. When requests take longer, the load generator slows down, giving the system time to recover. Real traffic doesn't wait politely.

Use load testing tools that account for coordinated omission (like wrk2 or Gatling with open workload models) or inject requests at a fixed rate regardless of response time.

### 5. Optimizing Without Profiling

"The database is probably slow, let's add caching." Maybe. Or maybe the serialization of the 5MB response body takes 200ms and caching wouldn't help at all because the cache hit itself still requires serialization.

Profile first. Trace the request. Identify the actual bottleneck. Then optimize.

### 6. Setting the Same Timeout for Every Operation

A health check, a user lookup, and a batch report generation do not have the same latency profile. Setting a global 30-second timeout means your health check won't fail fast, and your batch operation will fail prematurely.

Set timeouts per operation based on expected latency plus headroom.

### 7. Not Tracking Latency by Endpoint

A single service-level P99 hides per-endpoint differences. Your `/health` endpoint at 2ms and your `/reports/generate` endpoint at 8 seconds average out to "looks fine." Track latency per endpoint.

### 8. Adding More Instances to Fix a Latency Problem

More instances help with throughput. They don't help with latency unless the bottleneck is resource contention (thread pool exhaustion, CPU saturation, connection pool exhaustion). If a single request is slow because it makes 5 sequential database calls, 100 instances won't make that individual request faster.

Scaling out fixes throughput. Optimizing the request path fixes latency.

### 9. Ignoring Warmup Latency

The first requests after a deploy or scale-up event are often 10-100x slower due to cold caches, JIT compilation, connection pool initialization, and class loading. If you route production traffic to a new instance immediately, those first users get punished.

Use readiness probes that only pass after warmup is complete, pre-warm caches, and gradually shift traffic to new instances.

### 10. Not Accounting for Tail Latency Amplification

If your service makes 10 parallel calls to a backend, and each call has P99 of 100ms, the P99 of the fan-out is NOT 100ms. The probability that at least one of the 10 calls hits P99 is much higher:

```
P(at least one slow) = 1 - (0.99)^10 = 9.6%

Your P90 for the fan-out ≈ the backend's P99
```

With 100 parallel calls, the probability that at least one is slow approaches certainty. Tail latency in fan-out architectures is dominated by the slowest call. Hedged requests and aggressive timeouts are the primary mitigations.

---

## Quick Reference Checklist

- [ ] **Latency is measured in percentiles** — P50, P95, P99 at minimum. Never report averages as the primary metric
- [ ] **Percentiles are computed from aggregatable histograms**, not summaries, with bucket boundaries matching SLO thresholds
- [ ] **Measurement happens at the caller**, not just the server — client-side latency is the truth
- [ ] **Error and success latency are tracked separately** — never mix status codes in the same latency distribution
- [ ] **Latency is tracked per endpoint**, not just per service — aggregate service-level metrics hide per-endpoint problems
- [ ] **SLOs are set on P99**, not P50 or average — the tail is where users suffer
- [ ] **Every downstream dependency has its own latency metrics** — P50/P95/P99, timeout rate, and connection pool utilization
- [ ] **Latency budget is defined for the call chain** — each operation has an allocation, and deadline propagation prevents SLA breaches
- [ ] **Independent calls are parallelized** — sequential calls to independent services are a free latency win
- [ ] **Work that doesn't need to be synchronous is moved async** — email, notifications, analytics, audit logging
- [ ] **Caching is in place for hot-path reads** — in-process for small/stable data, distributed for shared data
- [ ] **Payload sizes are controlled** — pagination, field selection, compression, binary formats where appropriate
- [ ] **Database access is optimized** — indexes, connection pooling, batching, no N+1 queries
- [ ] **Timeouts are set per-operation**, not globally — and they're shorter than the upstream SLA
- [ ] **Cold start / warmup is handled** — readiness probes, cache pre-warming, connection pool initialization
- [ ] **Consumer lag is the primary Kafka latency metric** — tracked in both offsets and time, with alerting on sustained growth
- [ ] **Kafka consumer tuning parameters are set intentionally** — `fetch.min.bytes`, `max.poll.records`, and `fetch.max.wait.ms` are configured for your latency vs. throughput needs
- [ ] **Rebalances are minimized** — static group membership, cooperative rebalancing, graceful shutdown
- [ ] **Latency is profiled before optimizing** — distributed tracing and flame graphs identify the actual bottleneck
- [ ] **Tail latency amplification is understood** — fan-out architectures account for the probability that at least one call hits P99

---

## Related Guides

- See Caching for comprehensive caching strategies and invalidation patterns.
- See Monitoring for the four golden signals framework, dashboard design, and dependency monitoring.
