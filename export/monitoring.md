# Monitoring Best Practices

Good monitoring tells you the health of your system before your users do. It's the difference between proactively catching a slow database query at 2 AM and finding out from a Slack message at 9 AM that checkout has been broken for seven hours. But monitoring done poorly — vanity dashboards, uncurated metrics, no baselines — creates a false sense of security that's worse than having nothing at all, because your team *thinks* they're covered.

This guide covers what to monitor, how to structure it, and how to avoid the traps that make monitoring useless.

> **Note:** This guide focuses on *monitoring* — what you observe and how you observe it. For guidance on *alerting* — when to page, threshold strategies, and escalation — see alerting.md.

---

## Quick Decision Guide

| Question | Answer |
|----------|--------|
| "What metrics should I start with?" | The four golden signals: latency, traffic, errors, saturation |
| "How many dashboards should my service have?" | One primary operational dashboard per service, plus optional deep-dives |
| "Should I monitor the app or the infra?" | Both — app metrics tell you *what's* broken, infra metrics tell you *why* |
| "How long should I retain metrics?" | High-resolution (10s–1m) for 7–14 days, downsampled (5m–1h) for 6–12 months |
| "When should I add a new metric?" | When it would change a decision or action you'd take during an incident |
| "What's the biggest monitoring mistake?" | Collecting metrics nobody looks at and missing the ones that matter |

---

## 1. The Four Golden Signals

Google's SRE book nailed this: if you can only monitor four things, monitor these.

| Signal | What It Measures | Example |
|--------|-----------------|---------|
| **Latency** | Time to serve a request | p50, p95, p99 response times |
| **Traffic** | Demand on your system | Requests per second, messages consumed per second |
| **Errors** | Rate of failed requests | HTTP 5xx rate, exception count, failed queue messages |
| **Saturation** | How "full" your system is | CPU utilization, memory pressure, thread pool usage, queue depth |

### The key insight

These four signals cover the vast majority of production incidents. A latency spike with rising saturation points to a resource bottleneck. A traffic drop with no error spike suggests an upstream issue. Errors climbing with flat latency means you're failing fast (which might be correct behavior).

**Don't start with 50 metrics.** Start with these four, get them right, then add more when you have a specific question they can't answer.

### How to measure latency correctly

Always track latency using percentiles (P50, P95, P99), not averages. Averages hide outliers and give false confidence. Use histograms (not summaries) so percentiles can be aggregated across instances and time windows. See Latency for percentile measurement details, including why averages lie, histogram vs. summary tradeoffs, and which percentiles to use for SLOs.

---

## 2. The USE Method for Infrastructure

For every infrastructure resource (CPU, memory, disk, network), track three things:

| Letter | Stands For | What to Check |
|--------|-----------|---------------|
| **U** | Utilization | Percentage of resource capacity in use |
| **S** | Saturation | Work queued because the resource is full |
| **E** | Errors | Error events on the resource |

### Applying USE to common resources

| Resource | Utilization | Saturation | Errors |
|----------|-------------|------------|--------|
| CPU | CPU % (per core and aggregate) | Run queue length, load average | Machine check exceptions (rare) |
| Memory | Used memory / total memory | Swap usage, OOM kill count | ECC errors, allocation failures |
| Disk I/O | Disk busy % | I/O queue depth, await time | Read/write errors, SMART alerts |
| Network | Bandwidth utilization | TCP retransmits, socket backlog | Dropped packets, CRC errors |
| Thread pool | Active threads / max threads | Queue depth, rejected tasks | Thread pool rejections |
| Connection pool | Active connections / max pool | Connection wait time, pending requests | Connection timeouts, refused connections |

### The rule of 70%

If utilization sits above 70% during normal traffic, you're one traffic spike away from saturation. This isn't a universal law, but it's a useful heuristic: investigate *before* you hit the wall, not after.

---

## 3. Application vs. Infrastructure Monitoring

A common mistake is monitoring only one layer. You need both, and you need to understand what each layer tells you.

| Layer | Tells You | Blind Spots |
|-------|-----------|-------------|
| **Application metrics** | What's broken from the user's perspective (latency, errors, throughput) | Why it's broken at the system level |
| **Infrastructure metrics** | Resource constraints, capacity issues, hardware failures | Whether users are actually affected |
| **Business metrics** | Whether the system is doing what it's supposed to (orders placed, payments processed) | Technical root cause |

### The three-layer monitoring model

```
┌─────────────────────────────────────┐
│        Business Metrics             │  ← Orders/min, revenue, conversion rate
│   "Is the business outcome healthy?"│
├─────────────────────────────────────┤
│       Application Metrics           │  ← Latency, error rate, throughput
│    "Is the service behaving?"       │
├─────────────────────────────────────┤
│      Infrastructure Metrics         │  ← CPU, memory, disk, network
│    "Are the resources healthy?"     │
└─────────────────────────────────────┘
```

**Work top-down during incidents.** Business metrics tell you *what's* impacted. Application metrics narrow it to *which service*. Infrastructure metrics reveal *why*.

---

## 4. What to Monitor (and What Not To)

### Every service should monitor

| Category | Metrics | Why |
|----------|---------|-----|
| Request rate | Requests/sec by endpoint | Detect traffic anomalies, capacity plan |
| Error rate | 4xx and 5xx rates, exception counts | Detect failures before users report them |
| Latency | p50, p95, p99 by endpoint | Catch performance regressions |
| Dependencies | Latency and error rate for every downstream call | Pinpoint which dependency is causing issues |
| Resource usage | CPU, memory, thread pools, connection pools | Detect saturation before it causes failures |
| JVM/Runtime | GC pause time, heap usage, thread count (if applicable) | Catch memory leaks and GC storms early |

### Stop monitoring these (or at least stop putting them on dashboards)

| Vanity Metric | Why It's Useless | What to Track Instead |
|---------------|------------------|----------------------|
| Total request count (cumulative) | Number only goes up — tells you nothing about current state | Requests per second (rate) |
| Average response time | Hides outliers and bimodal distributions | Percentile latencies (p50, p95, p99) |
| Uptime percentage (calculated live) | Changes too slowly to be actionable | Error rate and availability over rolling windows |
| Server count | Doesn't tell you if servers are healthy | Healthy instance count vs. desired count |
| "Green/red" status without context | Binary status hides partial failures | Percentage-based health with thresholds |

### The litmus test for adding a new metric

Ask yourself: **"If this metric changed significantly, what action would I take?"**

- If the answer is specific ("I'd scale up the pool", "I'd check the downstream service"), add the metric.
- If the answer is "I'd investigate more" with no clear next step, you probably need a different metric.
- If the answer is "nothing", don't add it.

---

## 5. Dashboard Design

Dashboards are the user interface of your monitoring system. Bad dashboards are like bad UIs — they have all the data but none of the answers.

### The one-dashboard rule

Every service should have exactly **one primary operational dashboard**. This is the dashboard you pull up at 2 AM when something is wrong. It should answer three questions in under 30 seconds:

1. **Is the service healthy right now?** (error rate, latency)
2. **What changed?** (recent deployments, traffic shifts)
3. **Where should I look next?** (dependency health, resource utilization)

### Primary dashboard layout

| Row | Content | Purpose |
|-----|---------|---------|
| Top | Traffic rate + error rate (overlaid) | Instant health signal |
| Second | Latency percentiles (p50, p95, p99) | Performance at a glance |
| Third | Dependency health (latency + error rate per downstream) | Isolate external causes |
| Fourth | Resource saturation (CPU, memory, thread pools) | Identify resource constraints |
| Bottom | Recent deployments / config changes (annotations) | Correlate changes with behavior shifts |

### Deep-dive dashboards

After your primary dashboard, you may need specialized dashboards for:

- **Database performance**: query latency, connection pool utilization, slow query counts
- **Cache behavior**: hit rate, eviction rate, memory usage
- **Queue processing**: consumer lag, processing rate, dead letter queue depth
- **JVM internals**: GC pauses, heap generations, class loading

**The rule: link, don't merge.** Keep deep-dives as separate dashboards linked from the primary one. Cramming everything into one dashboard defeats the purpose.

### Dashboard anti-patterns

| Anti-Pattern | The Problem | The Fix |
|--------------|-------------|---------|
| The "wall of graphs" | 40 panels, no hierarchy, no one reads it | Ruthlessly cut to the metrics that drive decisions |
| The "single-number" dashboard | One big green number that turns red when it's too late | Show trends, not just current state |
| The "deploy-and-forget" dashboard | Created during initial setup, never updated | Review dashboards quarterly; delete unused panels |
| The "snowflake" dashboard | Every team's dashboard looks completely different | Standardize layout for primary operational dashboards |
| The "no-context" dashboard | Graphs without units, labels, or thresholds | Every panel needs a title, unit, and reference line for "normal" |

---

## 6. Baselines and Anomaly Detection

You can't detect abnormal behavior if you don't know what normal looks like.

### Establishing baselines

A baseline is the expected range for a metric during normal operation. It's not a single number — it varies by:

| Factor | Example |
|--------|---------|
| Time of day | Traffic peaks at 10 AM and 7 PM, drops at 3 AM |
| Day of week | Monday traffic is 20% higher than Sunday |
| Seasonality | Holiday shopping season traffic is 3–10x normal |
| Recent deployments | A new feature may shift baseline latency |

### How to build useful baselines

1. **Collect at least two weeks of data** before defining what's "normal"
2. **Segment by time** — a weekday baseline and a weekend baseline at minimum
3. **Use percentile bands** instead of fixed thresholds — "p95 latency is normally between 180ms and 250ms during business hours"
4. **Update baselines after significant changes** — new deployments, new features, and traffic pattern shifts all invalidate old baselines

### The "compared to yesterday" technique

The simplest baseline: overlay today's metrics against the same time window yesterday (or last week). Most monitoring tools support this natively. If today's error rate is 3x yesterday's at the same hour, something changed.

```
// Pseudo-query: compare current error rate to same time last week
current:  rate(http_errors_total[5m])
baseline: rate(http_errors_total[5m] offset 7d)
```

---

## 7. Monitoring Dependencies

Your service is only as reliable as its least-monitored dependency.

### What to track for every dependency

| Metric | Why |
|--------|-----|
| Latency (p50, p95, p99) | Detect slowdowns before they cascade |
| Error rate | Catch failures in downstream services |
| Timeout rate | Distinguish between slow and dead |
| Circuit breaker state | Know when your service has stopped calling a dependency |
| Connection pool utilization | Detect pool exhaustion before it becomes an outage |

### Inbound vs. outbound monitoring

| Direction | What You See | Blind Spots |
|-----------|--------------|-------------|
| **Outbound** (you calling them) | Latency/errors from your perspective | Their internal health, other consumers' experience |
| **Inbound** (them calling you) | Your behavior from their perspective | Why they're calling less or differently |

**Monitor both.** If your outbound latency to a database is fine but their CPU is at 95%, you're about to have a problem. If your inbound traffic dropped 50% but your service looks healthy, the problem is upstream.

### The dependency map

Every service should maintain a dependency map — a simple list of what it calls and what calls it. This sounds obvious, but most teams can't produce one on demand during an incident.

```
┌──────────┐     ┌──────────────┐     ┌──────────┐
│ Frontend │────→│ Order Service│────→│ Database │
└──────────┘     │              │────→│ Cache    │
                 │              │────→│ Payment  │
                 │              │────→│ Inventory│
                 └──────────────┘     └──────────┘
                       ↑
                 ┌──────────┐
                 │ Kafka    │ (async)
                 └──────────┘
```

For each arrow, you should have latency and error rate metrics. If you don't, that's a monitoring gap.

---

## 8. Logging vs. Metrics vs. Traces

These three pillars of observability serve different purposes. Using the wrong one for the job wastes resources and hides signal.

| Pillar | Best For | Not Good For | Cardinality |
|--------|----------|--------------|-------------|
| **Metrics** | Aggregated system behavior, trends, alerting | Individual request debugging | Low (pre-aggregated) |
| **Logs** | Detailed context for specific events, audit trails | Real-time trends, dashboarding | High (per-event) |
| **Traces** | Request flow across services, latency breakdown | Aggregate trends, long-term storage | Medium (sampled) |

### When to use which

| Scenario | Use |
|----------|-----|
| "Is the service healthy right now?" | Metrics |
| "Why did this specific request fail?" | Logs + Traces |
| "Where is the latency coming from in this request flow?" | Traces |
| "How many 500 errors did we throw in the last hour?" | Metrics |
| "What was the payload of the failed request?" | Logs |
| "Which service in the chain is the bottleneck?" | Traces |
| "Is error rate trending up over the past week?" | Metrics |

### The cardinality trap

Metrics are cheap when cardinality is low (a handful of label values). They get expensive fast when cardinality explodes.

```
// Low cardinality — good for metrics
labels: { endpoint: "/api/orders", method: "GET", status: "200" }

// High cardinality — use logs or traces instead
labels: { endpoint: "/api/orders", method: "GET", user_id: "abc123", order_id: "ord-456" }
// This creates a unique time series for every user/order combination.
// Your metrics storage will not thank you.
```

**Rule of thumb:** If a label can take more than ~100 distinct values, it belongs in logs or traces, not metric labels.

---

## 9. Metric Naming and Hygiene

Consistent metric names make dashboards, alerts, and queries dramatically easier to build and maintain.

### Naming conventions

| Convention | Example | Why |
|------------|---------|-----|
| Use dots or underscores, pick one | `http_request_duration_seconds` | Consistency across the codebase |
| Include the unit | `_seconds`, `_bytes`, `_total` | Avoids "is this milliseconds or seconds?" confusion |
| Use `_total` suffix for counters | `http_requests_total` | Distinguishes counters from gauges |
| Prefix with the service or domain | `order_service_http_requests_total` | Avoids collisions in shared metric stores |
| Use snake_case | `queue_message_processing_seconds` | Most metric systems prefer this over camelCase |

### Metric hygiene checklist

- **Prune unused metrics quarterly.** If nobody queries it and no alert fires on it, remove it.
- **Document every custom metric.** A metric without context is a metric nobody trusts.
- **Standardize labels across services.** If one team uses `status_code` and another uses `http_status`, cross-service dashboards become painful.
- **Don't change metric semantics silently.** If you change what a metric measures, rename it. Someone's alert depends on the old meaning.

---

## 10. Common Anti-Patterns

### Dashboard sprawl

**The symptom:** Your team has 15 dashboards. Nobody knows which one to check during an incident. Each was created for a specific investigation and never cleaned up.

**The fix:** Audit dashboards quarterly. Archive anything that hasn't been opened in 90 days. Consolidate overlapping dashboards. Appoint a dashboard owner per service.

### Vanity metrics

**The symptom:** Your dashboard shows impressive numbers (total requests served: 4.2 billion!) that don't help anyone make decisions.

**The fix:** Apply the litmus test from Section 4. Every metric on a dashboard should answer the question "what would I do differently if this number changed?"

### Missing baselines

**The symptom:** An engineer stares at a graph showing 340ms p95 latency and says, "Is that... normal?" Nobody knows, because nobody recorded what normal looks like.

**The fix:** For every key metric on your primary dashboard, annotate the expected range. Most tools support threshold lines or bands. "p95 is normally 200–300ms" turns a mystery into an answer.

### Alert-driven monitoring

**The symptom:** The team only looks at monitoring when an alert fires. Dashboards are never proactively reviewed.

**The fix:** Schedule a weekly 15-minute "service health review" where the on-call engineer walks through the primary dashboard. Catches slow degradation that doesn't trigger alerts.

### Copy-paste monitoring

**The symptom:** Every service uses the same dashboard template, including panels for components the service doesn't use (cache hit rate for a service with no cache, queue lag for a service with no consumers).

**The fix:** Start from a template, but customize it for each service's actual architecture. Remove panels that will always be empty or irrelevant.

### The "more data is better" fallacy

**The symptom:** The service emits 500 custom metrics. Storage costs are climbing. Nobody can find the metric they need because there are too many.

**The fix:** Start with the four golden signals. Add metrics only when you have a concrete question they'd answer. Treat every new metric like a feature — it has a maintenance cost.

---

## 11. Monitoring in Practice: A Checklist by Phase

### Before launch (new service)

- [ ] Four golden signals instrumented (latency, traffic, errors, saturation)
- [ ] Primary operational dashboard created with standard layout
- [ ] Dependency monitoring in place for every downstream service
- [ ] Connection pools, thread pools, and queues have utilization metrics
- [ ] Metric names follow the team's naming conventions
- [ ] Health check endpoint returns meaningful status (not just 200 OK)
- [ ] Baselines documented (even if estimated — refine after launch)

### After launch (first two weeks)

- [ ] Baselines validated against real traffic patterns
- [ ] Dashboard reviewed and irrelevant panels removed
- [ ] Metrics confirmed to populate correctly (no silent gaps)
- [ ] Dependency latency and error rates match expectations
- [ ] Key alerts configured (see alerting.md)

### Ongoing (quarterly)

- [ ] Dashboard audit — remove unused panels and dashboards
- [ ] Metric audit — prune metrics nobody queries
- [ ] Baseline review — update thresholds for changed traffic patterns
- [ ] Dependency map review — new dependencies added since last review?
- [ ] Cost review — are storage costs growing faster than the service is?

---

## Quick Reference

| Principle | Summary |
|-----------|---------|
| Start with the four golden signals | Latency, traffic, errors, saturation cover most incidents |
| Monitor both app and infra | App tells you *what*, infra tells you *why* |
| One primary dashboard per service | The 2 AM dashboard — answers health questions in 30 seconds |
| Establish baselines | You can't detect anomalies without knowing what normal looks like |
| Monitor every dependency | Your service is as reliable as its least-monitored dependency |
| Right pillar for the job | Metrics for trends, logs for details, traces for request flow |
| Watch cardinality | High-cardinality labels belong in logs, not metrics |
| Prune ruthlessly | Unused metrics and dashboards are debt, not assets |
| Apply the litmus test | If a metric change wouldn't change your action, don't track it |
| Review regularly | Monitoring is a living system — treat it like code, not furniture |

---

## Related Guides

- See Alerting for alert design details — thresholds, severity levels, SLO-based alerting, and alert hygiene.
- See Latency for percentile measurement details and latency optimization strategies.
- See Logging for structured logging, log levels, and the logging pillar of observability in depth.
