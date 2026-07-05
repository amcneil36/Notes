# Load Testing & Capacity Planning Best Practices

Before a feature goes to production, you need confidence it can survive real traffic. There are two distinct approaches — **empirical** (run actual load tests) and **analytical** (calculate whether the numbers work). Best practice is to use both, choosing based on what you're validating and the risk profile of the change.

---

## Two Approaches, Different Purposes

### Empirical: Load Testing
Run actual traffic against the system and observe behavior. The system either holds up or it doesn't.

**Best for:**
- Validating a new service or major architectural change
- Finding unknown bottlenecks you didn't model
- Confirming system behavior under sustained load (memory leaks, connection pool exhaustion, GC pressure)
- Validating concurrency — how does the system behave when 500 requests arrive simultaneously?

### Analytical: Capacity Modeling
Use math and known limits to calculate whether your projected usage fits within existing headroom.

**Best for:**
- Small, well-understood features on stable infrastructure
- Quick go/no-go decisions before committing to load test infrastructure
- Identifying *which* component is the bottleneck before testing
- Features where load testing isn't practical (prod-only external dependencies, hard-to-simulate traffic patterns)

**The key insight:** You don't always need to run a load test. If a feature adds 50 read QPS and your database supports 10,000 QPS with current utilization at 2,000, the math is sufficient. But if you're adding 8,000 QPS, the math tells you you're too close — and now you need a test to understand behavior at the margin.

---

## Analytical Capacity Planning

### Step 1: Know Your Limits

Before you can calculate, you need to know the ceilings. Document these per component:

| Component | Limit to Know |
|---|---|
| Database (reads) | Max read QPS, connection pool size |
| Database (writes) | Max write QPS, replication lag threshold |
| Cache (Redis) | Max ops/sec, max memory, eviction policy |
| API service | Max concurrent connections, thread pool size |
| Kafka | Max throughput per partition, consumer lag tolerance |
| External APIs | Rate limits (often in contract SLA) |
| CPU | Core count, target utilization ceiling (typically 60-70%) |
| Memory | Heap size, GC overhead budget |
| Network | Bandwidth per node, inter-service latency budget |

If you don't know these numbers, find them before writing the feature. Ask the platform team, check your runtime configuration, read the vendor docs, or run a baseline load test on the empty system.

### Step 2: Project Your Load

For a given feature, calculate what it adds to each component:

```
Projected Load = (requests per user action) × (active users) × (peak multiplier)
```

**Example: New "View Order History" feature**

- Expected users: 500 ops associates using the app
- Usage pattern: each user views history ~10 times/day, concentrated in a 2-hour morning window
- Raw QPS: (500 users × 10 views/day) / (8 hours × 3600 sec) = 0.17 QPS
- Peak window QPS: (500 × 10) / (2 hours × 3600 sec) = 0.69 QPS
- Each view triggers 3 DB reads (order list + status + user metadata)
- **DB read QPS added: 0.69 × 3 = ~2.1 QPS**

Current DB read utilization: 800 QPS. DB limit: 5,000 QPS. Headroom: 4,200 QPS.
**Conclusion: Ship it. No load test required.**

### Step 3: Apply a Safety Multiplier

Never assume average load is peak load. Production traffic is bursty and unpredictable.

- **1.5x–2x** for internal tools with predictable usage patterns
- **3x–5x** for consumer-facing features with viral potential
- **10x** for anything that could be triggered by external events (sales events, news, batch jobs)

If your projected load × safety multiplier still fits within headroom, you're analytically safe. If it's within 80% of a limit at the multiplied number, treat it as a risk and run a test.

### Step 4: Identify the Bottleneck Component

Rarely does everything fail at once. One component will hit its limit first — find it.

Common bottleneck candidates:
- **Database connection pool** — often smaller than people think (e.g., 100 connections shared across 10 service instances = 10 per instance)
- **Thread pool / event loop** — services block on downstream calls and exhaust threads
- **External API rate limits** — third-party limits you didn't account for
- **Network bandwidth** — relevant for large payloads or high-frequency polling
- **Cache miss rate** — feature bypasses cache more than expected, amplifying DB load

Once you know the bottleneck, you know where to focus your test or your optimization.

---

## Load Testing (Empirical)

### When You Must Run a Load Test

- A new service being deployed for the first time
- Migrating to new infrastructure (new DB engine, new cache cluster, new region)
- A feature that multiplies existing load significantly (e.g., adding polling, webhooks, or a new batch job)
- When analytical modeling shows headroom < 50% at projected peak
- Regulatory or compliance requirement
- After a production incident — to validate the fix holds under load
- Before a known traffic event (Black Friday, product launch, batch processing window)

### Types of Load Tests

| Test Type | What It Does | When to Use |
|---|---|---|
| **Smoke test** | 1–5% of target load for a short burst | Sanity check — does it respond at all? |
| **Load test** | Steady target load for 10–30 minutes | Normal operating conditions |
| **Stress test** | Ramp up beyond expected peak until failure | Find the breaking point |
| **Spike test** | Sudden jump to peak, then drop | Simulate flash traffic (marketing push, batch job) |
| **Soak test** | Sustained moderate load for hours | Find memory leaks, connection pool exhaustion, GC drift |
| **Breakpoint test** | Slow ramp until something breaks | Discover the system's theoretical ceiling |

Don't just run a load test and call it done. Run a **smoke → load → stress** sequence. The smoke test confirms basic functionality, the load test validates normal operation, and the stress test tells you how much headroom you actually have.

### What to Measure

Don't just watch "does it work" — instrument the right signals:

**Latency:**
- P50 (median) — what a typical user experiences
- P95 — what the 95th percentile user experiences (important for SLAs)
- P99 — tail latency (often caused by GC pauses, lock contention, cache misses)
- P999 — the worst 0.1% (database connection timeouts, retry storms)

**Throughput:**
- Requests per second achieved vs. target
- Error rate (should be < 0.1% under normal load)
- Saturated throughput — the point where adding more load stops increasing processed requests

**Resource utilization:**
- CPU utilization per instance (target: < 70% at peak)
- Memory usage and growth over time (flat line = good, slow climb = memory leak)
- JVM heap and GC pause frequency/duration
- DB connection pool utilization (> 80% = danger zone)
- Thread pool queue depth

**Downstream:**
- DB query latency under load
- Cache hit rate under load (drops under load = cache thrashing)
- External API call latency and error rate

### Load Test Tools

| Tool | Best For |
|---|---|
| **k6** | Modern, scriptable, great developer UX, CI-friendly |
| **Gatling** | JVM-based, good for Java teams, detailed HTML reports |
| **JMeter** | Mature, GUI-based, widely supported in enterprise |
| **Locust** | Python-based, easy to write realistic user scenarios |
| **Artillery** | Good for HTTP + WebSocket, YAML-based config |

On most Kubernetes platforms, k6 integrates well with existing CI pipelines and produces clean metrics for Grafana dashboards.

### Writing a Good Load Test

A load test is only as good as how well it simulates real traffic. Common mistakes:

**Don't test a single endpoint in isolation.** Real users don't hit one endpoint 1,000 times. They follow a workflow — login → search → view → act. Model that workflow.

```js
// k6 example — realistic workflow simulation
import http from 'k6/http';
import { sleep } from 'k6';

export let options = {
  stages: [
    { duration: '2m', target: 100 },   // ramp up to 100 users
    { duration: '10m', target: 100 },  // hold steady
    { duration: '2m', target: 0 },     // ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% of requests under 500ms
    http_req_failed: ['rate<0.01'],    // error rate under 1%
  },
};

export default function () {
  // Simulate a realistic user flow
  http.get('/api/orders');
  sleep(1);
  http.get('/api/orders/123');
  sleep(2);
  http.post('/api/orders/123/reject', JSON.stringify({ reason: 'price' }));
  sleep(Math.random() * 3); // randomize think time
}
```

**Use realistic think time.** Real users pause between actions. Requests with zero think time create artificial concurrency spikes that don't reflect real usage.

**Use production-shaped data.** Testing with 10 records in the database when prod has 10 million will give you incorrect query performance results. Seed with realistic data volume before testing.

**Run from a dedicated environment.** Load test results from your laptop are meaningless. Run from a dedicated load test environment that mirrors the network path to staging.

### Interpreting Results — Pass/Fail Criteria

Define pass criteria *before* running the test, not after:

```yaml
# Example success criteria
thresholds:
  p95_latency: < 300ms          # at target load
  p99_latency: < 1000ms         # at target load
  error_rate: < 0.1%            # at target load
  peak_throughput: >= 500 rps   # system can handle this
  cpu_utilization: < 70%        # at target load
  memory_growth: < 5%           # over 30-min soak
```

A load test without pre-defined pass criteria is just watching numbers. You'll convince yourself everything is fine because you don't know what "fine" looks like.

---

## Combining Both Approaches

The most effective process is sequential:

```
1. Analytical model first
   → Calculate projected QPS per component
   → Identify the bottleneck
   → Determine headroom

2. Decision point
   → Headroom > 50% at 3x safety multiplier? → Analytical is sufficient, document and ship
   → Headroom 20-50%? → Run a targeted load test on the bottleneck component
   → Headroom < 20%? → Mandatory load test + consider architectural changes

3. Load test if required
   → Smoke → Load → Stress
   → Validate pass criteria
   → Capture baseline metrics for future comparison

4. Document findings
   → Record projected vs. actual QPS
   → Record system limits discovered
   → Set up monitoring alerts at 80% of discovered limits
```

---

## Monitoring and Alerting as a Continuous Load Test

A load test gives you a snapshot. Monitoring gives you continuous validation.

After launch, set alerts at meaningful thresholds — not at the limit itself:

| Metric | Alert at | Page at |
|---|---|---|
| DB read QPS | 70% of limit | 90% of limit |
| API error rate | > 0.5% | > 2% |
| P95 latency | > 2× baseline | > 5× baseline |
| CPU utilization | > 70% | > 90% |
| DB connection pool | > 75% utilized | > 90% utilized |
| Cache hit rate | < 80% | < 60% |

This turns your production system into an ongoing load test at real scale, with real data, validating real user patterns. The monitoring baseline established in load testing becomes your production SLO.

---

## Common Mistakes

- **Only testing the happy path** — add error scenarios (timeouts, DB unavailability, cache misses)
- **Not resetting state between runs** — leftover data from run 1 affects run 2 results
- **Testing against prod** — a load test in production will impact real users; always use a dedicated environment
- **Not accounting for retries** — if your client retries on failure, a failing system gets 3x the load it would otherwise
- **Ignoring downstream services** — your service might handle the load fine but overwhelm a dependency
- **One-time testing only** — load test again after significant dependency upgrades, schema changes, or traffic growth milestones (2×, 5×, 10× user growth)

---

## Quick Reference

1. **Before any new feature:** Do the QPS math. Document it in the PR description.
2. **If headroom > 50%:** Analytical sign-off is sufficient. No load test needed.
3. **If headroom 20–50%:** Run a k6 load test against staging, targeting the bottleneck endpoint.
4. **If headroom < 20%:** Mandatory load test + architectural review before shipping.
5. **New services or infra changes:** Always run smoke → load → stress before first prod deploy.
6. **Post-launch:** Set Grafana alerts at 70% of known limits; revisit capacity at each 2× traffic milestone.
