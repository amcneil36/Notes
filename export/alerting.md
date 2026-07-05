# Alerting Best Practices

Good alerting is the difference between catching a problem at 2% error rate and catching it after 500 users have filed support tickets. But bad alerting — too many alerts, wrong thresholds, vague messages — is arguably worse than no alerting at all, because it trains your team to ignore the system entirely.

The goal is simple: **every alert should be actionable, and every actionable problem should trigger an alert.**

---

## The Two Failure Modes of Alerting

Most teams suffer from one (or both) of these:

### 1. Alert Fatigue — Too Many Alerts

Symptoms:
- On-call engineer receives 50+ alerts per shift
- Team has a shared understanding that "most alerts can be ignored"
- Alerts are acknowledged without investigation
- Alert channels are muted or filtered to a folder
- New engineers ask "which alerts actually matter?" and nobody has a clear answer

Root causes:
- Alerting on metrics that aren't tied to user impact
- Thresholds set too tight (alerting on normal variance)
- No distinction between "something is degraded" and "something is broken"
- Alerting on symptoms AND causes simultaneously (getting 5 alerts for 1 problem)
- Alerts left in place after the condition they monitor is no longer relevant

### 2. Missing Critical Alerts — Not Enough Coverage

Symptoms:
- Users or stakeholders report problems before any alert fires
- Incidents are discovered during morning standups, not at 3 AM when they started
- Post-mortems repeatedly cite "we didn't have an alert for this"
- Business-impacting issues go unnoticed because only infrastructure metrics are monitored

Root causes:
- Only alerting on infrastructure metrics (CPU, memory) without application-level signals
- No business metric monitoring (order volume, conversion rate, revenue flow)
- Alerting on absolutes instead of anomalies (a 50% drop in traffic doesn't fire if traffic is still above the static threshold)
- Missing dependency monitoring (your service is fine, but the downstream it depends on is returning garbage)

---

## What to Alert On — The Alert Pyramid

Think of alerting in layers. Each layer catches different classes of problems:

```
        /  Business  \          ← "Are users getting value?"
       / Application  \         ← "Is the service behaving correctly?"
      / Infrastructure  \       ← "Are the resources healthy?"
     /  Dependencies      \     ← "Are the things we depend on healthy?"
```

### Layer 1: Business Metrics (Highest Signal)

These are the alerts that matter most because they directly reflect user impact. They're also the most commonly missing.

| Metric | Alert Condition | Why It Matters |
|---|---|---|
| Order/transaction volume | Drops > 30% vs. same hour last week | Revenue impact — if orders stop, nothing else matters |
| Conversion rate | Drops below historical baseline | Feature may be broken even if the service is "up" |
| Revenue flow | Falls below expected range for time of day | Catches payment processing failures |
| Active users / sessions | Sudden drop > 40% | May indicate login failures, DNS issues, or CDN outage |
| Business SLA metrics | Approaching SLA breach threshold | Contractual obligations at risk |

Business alerts are powerful because they catch problems that infrastructure alerts miss entirely. A subtle data corruption bug might cause zero errors, zero latency spikes, and zero CPU anomalies — but orders drop 20% because prices are wrong.

### Layer 2: Application Metrics

These are the bread and butter of service alerting. They tell you whether your code is behaving correctly.

| Metric | Alert Condition | Why It Matters |
|---|---|---|
| Error rate (5xx) | > 1% of requests over 5 minutes | Service is failing for real users |
| Error rate (4xx) | Spike > 3x baseline | May indicate a bad deploy or client-side breakage |
| P95 latency | > 2x baseline for 5+ minutes | Users are experiencing slowness |
| P99 latency | > 5x baseline | Tail latency affecting a meaningful number of users |
| Request throughput | Drops > 50% vs. expected | Service may be unreachable or upstream is broken |
| Queue depth / consumer lag | Growing for > 10 minutes | Consumers can't keep up — data is getting stale |
| Circuit breaker state | Open | Downstream dependency is failing |

### Layer 3: Infrastructure Metrics

These tell you whether the underlying compute, storage, and network are healthy.

| Metric | Alert Condition | Why It Matters |
|---|---|---|
| CPU utilization | > 80% sustained for 10+ min | Approaching saturation; autoscaling may not keep up |
| Memory utilization | > 85% or steady climb over hours | OOM kill risk or memory leak |
| Disk usage | > 80% | Logs or data filling disk; service will crash at 100% |
| Pod restarts | > 3 in 10 minutes | CrashLoopBackOff or OOM kills |
| DB connection pool | > 80% utilized | Connection exhaustion imminent |
| DB replication lag | > 5 seconds | Read replicas serving stale data |
| Cache hit rate | Drops below 70% | Cache is thrashing; DB is absorbing the load |
| JVM GC pause time | > 500ms or > 10% of wall time | GC pressure causing latency spikes |

### Layer 4: Dependency Metrics

Your service is only as reliable as its weakest dependency.

| Metric | Alert Condition | Why It Matters |
|---|---|---|
| Downstream error rate | > 5% to a specific dependency | That dependency is degraded |
| Downstream latency | > 3x baseline | Calls are timing out, backing up your thread pool |
| Certificate expiry | < 30 days | SSL cert expiry causes hard outages with no graceful degradation |
| External API rate limit | > 80% of quota consumed | About to hit the wall on a third-party API |
| DNS resolution failures | Any sustained failure | Complete service unreachability |

---

## Alert Severity Levels

Not every alert deserves the same response. Define clear severity levels and stick to them.

| Severity | Meaning | Response | Notification |
|---|---|---|---|
| **P1 / Critical** | User-facing outage or data loss in progress | Immediate response required (wake someone up) | Page on-call + incident channel |
| **P2 / High** | Significant degradation, not full outage | Respond within 15 minutes during business hours | Page on-call during business hours, notify off-hours |
| **P3 / Warning** | Something is trending toward a problem | Investigate within the current shift | Alert channel (no page) |
| **P4 / Info** | Informational — may need attention eventually | Review during next business day | Dashboard only or low-priority channel |

### The Key Rule: Only P1 and P2 Should Page

If your P3 alerts are paging people, you've mislabeled them. If your P1 alerts aren't waking people up, you've mislabeled those too.

A useful litmus test for severity:
- **P1**: "If this happened at 3 AM on a Saturday, would I wake someone up?" If yes, it's P1.
- **P2**: "If this happened at 3 AM, would it be okay to wait until 7 AM?" If yes, it's P2.
- **P3**: "Can this wait until Monday?" If yes, it's P3.
- **P4**: "Is this just nice to know?" Then it's P4 or shouldn't be an alert at all.

---

## Thresholds — The Art of Not Being Annoying

Setting thresholds is where most teams go wrong. Too tight and you get alert fatigue. Too loose and you miss real problems.

### Static Thresholds

Fixed numeric values. Simple and predictable, but brittle.

```
# Good for metrics with stable baselines
alert: HighErrorRate
  condition: error_rate > 1%
  for: 5 minutes
```

**When to use:** Metrics with well-known, stable limits (CPU > 90%, disk > 85%, error rate > 1%).

**When NOT to use:** Metrics that vary by time of day, day of week, or season. A static threshold of "requests < 1000/sec" will fire every night when traffic naturally drops.

### Dynamic / Anomaly-Based Thresholds

Compare current values to historical baselines (same hour last week, rolling average).

```
# Alert when metric deviates significantly from its own baseline
alert: TrafficAnomaly
  condition: current_value < 0.5 * avg(same_hour_last_4_weeks)
  for: 15 minutes
```

**When to use:** Business metrics, traffic volume, conversion rates — anything with natural variability.

**When NOT to use:** Metrics that should have hard limits regardless of baseline (error rates, resource exhaustion).

### Rate-of-Change Thresholds

Alert on how fast a metric is changing, not its absolute value.

```
# Memory is climbing too fast — probably a leak
alert: MemoryLeak
  condition: rate_of_increase(memory_usage, 1h) > 5%
  for: 30 minutes
```

**When to use:** Memory leaks, queue depth growth, disk usage growth, connection pool drain.

### The "For" Duration — Don't Alert on Blips

Almost every alert should have a minimum duration before firing. A 2-second CPU spike to 95% is normal. A 10-minute sustained spike is a problem.

| Metric Type | Recommended Minimum Duration |
|---|---|
| Error rate | 3–5 minutes |
| Latency | 5 minutes |
| CPU / Memory | 10 minutes |
| Disk usage | 15 minutes |
| Business metrics | 10–15 minutes |
| Queue depth growth | 10 minutes |
| Dependency errors | 3 minutes |

Shorter durations catch problems faster but generate more noise. Longer durations reduce noise but delay response. Tune based on how fast the metric can cause real damage.

---

## Writing Actionable Alert Messages

A bad alert message:

```
ALERT: High CPU on prod-service-7
```

What is the on-call engineer supposed to do with this? Which service? What's the CPU at? What's normal? Where do I look?

A good alert message includes:

```
[P2] CPU at 87% on order-api (pod order-api-5b8f9-xk2wj)
  Current: 87% (threshold: 80%)
  Baseline: 45% at this time of day
  Duration: 12 minutes
  Dashboard: <link>
  Runbook: <link>
  Possible causes: Recent deploy, traffic spike, memory leak triggering GC
```

### Alert Message Checklist

Every alert should answer these questions:

1. **What** is happening? (metric name, current value, threshold)
2. **Where** is it happening? (service, pod, region, endpoint)
3. **How bad** is it? (severity, how far from normal)
4. **How long** has it been happening? (duration)
5. **Where do I look?** (dashboard link, log query link)
6. **What do I do?** (runbook link, or inline first-response steps)

If your alert can't answer at least questions 1–4, it's not ready for production.

---

## Alert Hygiene — Keeping Your Alerts Healthy

Alerts are not "set and forget." They require maintenance, just like code.

### Regular Alert Review

Schedule a monthly or quarterly alert review. For each alert, ask:

1. **Has this alert fired in the last 90 days?**
   - If no: Is the condition still possible? If not, delete it. If yes, is the threshold too loose?
2. **When it fired, was it actionable?**
   - If the response was always "acknowledge and ignore," the alert is noise. Fix the threshold or delete it.
3. **When it fired, did someone take a meaningful action?**
   - If yes, the alert is healthy. If no, it's a candidate for downgrade or deletion.
4. **Has the underlying system changed?**
   - New infrastructure, new dependencies, new traffic patterns — thresholds may need adjustment.

### The Alert-to-Incident Ratio

Track how many alerts result in actual incidents or meaningful actions. A healthy ratio:
- **> 50% of pages result in action** — your alerting is well-tuned
- **20–50%** — review your thresholds and severities
- **< 20%** — your alerting is actively harming your team's effectiveness

### Deduplication and Grouping

When a database goes down, you don't want 15 alerts — one for each service that depends on it. Group related alerts:

- **By root cause**: If the DB is down, fire one "database unavailable" alert, not N "service X can't reach DB" alerts
- **By service**: Group all alerts for a single service instance into one notification with a summary
- **By time window**: If the same alert fires 10 times in 5 minutes, send one notification with a count

Most alerting platforms support grouping rules. Use them aggressively.

### Suppress During Known Events

Maintenance windows, deployments, and planned failovers will trigger alerts. Suppress alerts during these windows to avoid noise and false escalations:

- **Deployment windows**: Suppress latency and error rate alerts for 5–10 minutes after a deploy (but NOT for longer — if errors persist post-deploy, that's a real problem)
- **Maintenance windows**: Suppress infrastructure alerts for the affected components
- **Planned failovers**: Suppress dependency alerts during failover testing

Never suppress P1 alerts. If something is truly critical, you want to know even during maintenance.

---

## Common Anti-Patterns

### 1. Alerting on Causes Instead of Symptoms

**Bad**: Alert on "GC pause > 200ms"
**Better**: Alert on "P95 latency > 500ms"

The GC pause is a *cause*. The latency increase is the *symptom* that users feel. Alert on what users experience, then use dashboards and logs to diagnose the cause.

Exception: alert on causes when the symptom is catastrophic and you want early warning (e.g., memory leak trending toward OOM — don't wait for the OOM kill).

### 2. Duplicating Alerts Across Layers

If you alert on "DB connection pool > 90%" AND "service error rate > 5%" AND "P95 latency > 2s" — and all three fire simultaneously because of a DB issue — your on-call gets three pages for one problem.

Fix: use dependent/hierarchical alerts. If the DB connection pool alert fires, suppress the downstream symptom alerts for a grace period.

### 3. Percentage Thresholds on Low-Volume Endpoints

An endpoint that serves 10 requests per hour has a 10% error rate if a single request fails. That's noise, not a real problem.

```
# Bad — fires on 1 error out of 10 requests
alert: HighErrorRate
  condition: error_rate > 5%

# Good — requires both rate AND volume
alert: HighErrorRate
  condition: error_rate > 5% AND request_count > 100
  for: 5 minutes
```

Always pair percentage thresholds with a minimum volume floor.

### 4. Alerting on Every Exception

Not every exception is alert-worthy. A `404 Not Found` is expected behavior. A `NullPointerException` in your payment processing path is not.

Classify exceptions by severity (see the exception handling best practices doc) and only alert on the ones that indicate real problems:
- `ResourceNotFoundException` — not an alert
- Spike in `InternalServiceException` — alert
- Any `Error` (OOM, StackOverflow) — alert immediately

### 5. No Alert Ownership

Every alert must have an owner — a team or individual responsible for responding to it. Unowned alerts are ignored alerts.

If nobody knows who should respond to an alert, the alert will bounce between teams during an incident, wasting the time that matters most.

### 6. Copy-Pasting Alerts Between Services

Each service has different traffic patterns, different SLOs, and different failure modes. An error rate threshold of 1% might be perfect for a high-traffic API but absurdly tight for a batch job that processes 50 items per day.

Treat alert configuration as part of the service's operational contract, not a boilerplate template.

---

## Setting Up Alerts for a New Service — Checklist

When launching a new service, start with these baseline alerts and tune from there:

### Day 1 (Before First Deploy)

- [ ] Error rate (5xx) > 1% for 5 minutes → P2
- [ ] P95 latency > target SLO for 5 minutes → P2
- [ ] Pod restart count > 3 in 10 minutes → P2
- [ ] CPU > 80% sustained for 10 minutes → P3
- [ ] Memory > 85% sustained for 10 minutes → P3
- [ ] Health check endpoint failing → P1
- [ ] Zero request throughput (service appears dead) → P1

### Week 1 (After Baseline Established)

- [ ] Throughput drops > 50% vs. baseline → P2
- [ ] DB connection pool > 80% → P3
- [ ] Downstream dependency error rate > 5% → P3
- [ ] Queue consumer lag growing for > 10 minutes → P3

### Month 1 (After Traffic Patterns Are Known)

- [ ] Business metric anomaly detection (vs. historical baseline) → P2
- [ ] Dynamic latency thresholds based on observed percentiles → P2/P3
- [ ] Cache hit rate drops below established baseline → P3
- [ ] Certificate expiry < 30 days → P3
- [ ] Review and remove any alerts that fired but were never actioned

---

## SLO-Based Alerting

The most mature approach to alerting is to derive alerts directly from your Service Level Objectives.

### The Concept

Instead of alerting on arbitrary thresholds, define what "good" looks like for your users and alert when you're burning through your error budget too fast.

```
SLO: 99.9% of requests complete successfully within 500ms
Error budget: 0.1% of requests can fail per month
Monthly request volume: 10,000,000
Error budget: 10,000 failed requests per month
```

### Burn Rate Alerts

Alert based on how fast you're consuming your error budget:

| Burn Rate | Meaning | Budget Exhaustion | Alert |
|---|---|---|---|
| 1x | Normal consumption | ~30 days | No alert |
| 2x | Slightly elevated | ~15 days | P4 (dashboard) |
| 10x | Significant issue | ~3 days | P3 (warning) |
| 30x | Major incident | ~24 hours | P2 (page during hours) |
| 100x | Outage in progress | ~7 hours | P1 (page immediately) |

This approach naturally handles the "percentage on low volume" problem — a single error on a low-volume endpoint barely dents the budget. And it naturally distinguishes between "slightly elevated error rate for a week" (slow burn, investigate Monday) and "massive spike right now" (page immediately).

### Multi-Window Burn Rate

Use two time windows to avoid alerting on brief spikes:

```
# Alert only when BOTH conditions are true:
# 1. Short window shows high burn rate (confirms it's happening NOW)
# 2. Long window shows sustained burn (confirms it's not a 30-second blip)

alert: SLOBurnRate
  condition:
    short_window (5min): burn_rate > 14x
    long_window (1hr): burn_rate > 14x
  severity: P1
```

---

## Quick Reference — Alert Design Checklist

For every alert you create, verify:

- [ ] **Actionable**: Someone can do something about it when it fires
- [ ] **Owned**: A specific team is responsible for responding
- [ ] **Contextual**: Alert message includes current value, threshold, dashboard link, and runbook
- [ ] **Deduplicated**: Won't fire alongside 5 other alerts for the same root cause
- [ ] **Tuned**: Has a minimum duration ("for" clause) to avoid blips
- [ ] **Volume-aware**: Percentage thresholds are paired with minimum request counts
- [ ] **Severity-appropriate**: P1/P2 page, P3/P4 don't
- [ ] **Reviewed regularly**: Scheduled for quarterly review
- [ ] **Tested**: You've verified the alert actually fires by simulating the condition

---

## Related Guides

- See Monitoring for the four golden signals framework, dashboard design, and the three pillars of observability.
- See Latency for percentile measurement details (P50/P95/P99, why averages lie, histograms vs. summaries).
