# Logging Best Practices

Logging is the one observability signal you always have. Metrics tell you *that* something is wrong, traces tell you *where* in the call chain it happened, and logs tell you *why*. But most teams either log too much (drowning in noise, burning money) or too little (flying blind during incidents). And almost everyone gets structured logging wrong.

The goal: **every log line should be parseable by a machine, readable by a human, and useful during an incident.**

---

## Log Levels — Use Them Correctly

Most teams treat log levels as vibes. They're not. Each level has a specific purpose, a specific audience, and a specific cost.

| Level | Purpose | When to Use | Who Reads It |
|---|---|---|---|
| **TRACE** | Ultra-verbose diagnostic detail | Method entry/exit, variable state during debugging | Developer, locally only |
| **DEBUG** | Detailed operational info | Request payloads, cache decisions, branch logic | Developer during troubleshooting |
| **INFO** | Normal operational events | Service started, request completed, job finished, config loaded | Ops, dashboards, audit |
| **WARN** | Something unexpected, but handled | Retry succeeded, fallback used, deprecated feature invoked, approaching threshold | Ops, alerting (non-paging) |
| **ERROR** | Something failed and needs attention | Unhandled exception, downstream call failed after retries, data inconsistency | On-call, incident response |
| **FATAL** | Service is going down | OOM, unrecoverable state, failed health check | On-call, PagerDuty |

### The Rules

1. **INFO is your baseline in production.** If your service can't run at INFO level without flooding logs, you're logging at the wrong level.
2. **DEBUG should never be on by default in production.** It exists for temporary, targeted troubleshooting. Turn it on for a specific class or module, investigate, turn it off.
3. **WARN means "this is unusual but we handled it."** If you didn't handle it, it's an ERROR. If it's expected behavior (like a 404), it's not a WARN — it's INFO or nothing.
4. **ERROR means "someone should look at this."** If nobody needs to look at it, downgrade it. If you're logging 500 ERRORs per minute and nobody investigates any of them, you've trained your team to ignore errors.
5. **Never log at FATAL and keep running.** If you log FATAL, the process should be shutting down. If it's not shutting down, it's not fatal.

### The Litmus Test

Before choosing a level, ask:

- "If this log line fired 10,000 times in an hour, would I want to know?" — If no, it's TRACE or DEBUG.
- "Does this indicate a problem?" — If no, it's INFO at most.
- "Is the problem handled?" — If yes, WARN. If no, ERROR.
- "Is this a normal, expected code path?" — If yes, it's not WARN or ERROR, regardless of what it "feels" like.

---

## Structured Logging — The Non-Negotiable

Plain text logs are a liability. They look fine in a terminal and completely fall apart when you need to search, filter, aggregate, or alert on them.

### Plain Text vs. Structured

```
# Plain text — good luck parsing this at scale
2025-07-04 12:34:56 INFO OrderService - Order 12345 placed by user 67890 for $42.99

# Structured JSON — every field is queryable
{
  "timestamp": "2025-07-04T12:34:56.789Z",
  "level": "INFO",
  "logger": "OrderService",
  "message": "Order placed",
  "orderId": "12345",
  "userId": "67890",
  "amount": 42.99,
  "currency": "USD",
  "traceId": "abc-123-def-456",
  "spanId": "span-789"
}
```

With structured logging, you can:
- Search for all logs where `orderId = 12345` without regex gymnastics
- Aggregate error counts by `logger` or `endpoint`
- Build dashboards on `amount` or `duration` fields
- Set up alerts on specific `error` code values

### What Fields to Always Include

Every log line should carry a baseline set of fields, injected automatically (not manually typed in each log call):

| Field | Source | Purpose |
|---|---|---|
| `timestamp` | Logging framework (ISO 8601, UTC) | When it happened |
| `level` | Logging framework | Severity |
| `logger` | Logging framework (class/module name) | Where in the code |
| `message` | Developer | What happened (human-readable) |
| `traceId` | Distributed tracing context (MDC/context propagation) | Correlate across services |
| `spanId` | Distributed tracing context | Correlate within a trace |
| `service` | Environment/config | Which service emitted this |
| `environment` | Environment/config | prod, staging, dev |
| `host` / `pod` | Runtime metadata | Which instance |

### What Fields to Add Per Log Statement

Beyond the baseline, add contextual fields that will help you filter during an incident:

```
# Good — structured context fields
log.info("Order placed", { orderId, userId, amount, paymentMethod })

# Bad — interpolated into the message string
log.info("Order " + orderId + " placed by user " + userId + " for $" + amount)
```

Put data in fields, not in the message. The message should describe what happened. The fields should describe the context.

---

## Observability Correlation — Connecting the Dots

Logs in isolation are useful. Logs connected to traces and metrics are powerful. The key is **correlation IDs** — shared identifiers that let you jump between signals.

### Trace ID Propagation

Every inbound request should carry (or generate) a trace ID that flows through every downstream call and appears in every log line.

```
Request arrives → Extract/generate traceId → Store in thread-local/context
  → Every log statement automatically includes traceId (via MDC or equivalent)
  → Every outbound HTTP/gRPC/message call propagates traceId in headers
  → Downstream services extract and continue the chain
```

| Propagation Mechanism | Protocol |
|---|---|
| `X-Trace-Id` / `traceparent` (W3C) | HTTP headers |
| Message metadata / headers | Kafka, RabbitMQ, SQS |
| gRPC metadata | gRPC calls |
| Thread-local / async context | In-process (MDC, AsyncLocalStorage, context.Context) |

### Correlation ID vs. Trace ID

They're often conflated, but they serve different purposes:

| ID | Scope | Purpose |
|---|---|---|
| **Trace ID** | Single request across all services | Links all spans in a distributed trace |
| **Correlation ID** | Business transaction (may span multiple requests) | Links all activity for a business operation (e.g., an order from placement through fulfillment) |
| **Request ID** | Single request within one service | Deduplication, idempotency |

For most teams, trace ID alone is sufficient. Add a correlation ID when a single business operation spans multiple independent requests (e.g., an async workflow triggered by an event).

### From Logs to Traces and Back

The power of correlation is the ability to jump between signals:

1. Alert fires on error rate spike
2. Open the dashboard, see the error count by endpoint
3. Click through to logs filtered by endpoint and time window
4. Find the error log, grab the `traceId`
5. Open the trace viewer, see the full request flow
6. Identify the slow/failing span
7. Click through to that service's logs for that trace

If any link in that chain is missing — if logs don't have trace IDs, if traces don't link to logs — you're back to grep and guesswork during an incident.

---

## Performance and Cost — Logging's Hidden Tax

Logging is not free. Every log line has a cost: CPU to format it, memory to buffer it, network to ship it, storage to keep it, and compute to index it. At scale, bad logging habits can cost more than the infrastructure they're monitoring.

### The Cost Equation

```
Annual log cost = (avg log lines/sec) × (avg bytes/line) × (seconds/year)
                  × (cost per GB ingested + cost per GB stored × retention days / 365)
```

A service logging 1,000 lines/sec at 500 bytes/line generates ~15 TB/year. If your log platform charges $2/GB ingested, that's $30,000/year — for one service.

### What's Actually Expensive

| Cost Driver | Impact | Mitigation |
|---|---|---|
| **High-cardinality fields** | Explodes index size (e.g., logging full request bodies, UUIDs as field names) | Log IDs and summaries, not payloads |
| **DEBUG in production** | 10-100x volume increase | Never enable globally; use targeted, time-limited debug |
| **Logging in hot loops** | Millions of log lines per second | Move to metrics or sample |
| **Large log lines** | > 1 KB per line adds up fast | Truncate large payloads; log references, not content |
| **Long retention** | Storage costs compound | Tier retention: 7 days hot, 30 days warm, archive the rest |

### Sampling — When You Can't Log Everything

For high-throughput services, logging every request is neither feasible nor necessary. Sampling strategies:

**Head-based sampling**: Decide at request entry whether to log (e.g., log 10% of requests).
```
if hash(traceId) % 10 == 0:
    log at DEBUG level
else:
    log at INFO level only
```

**Tail-based sampling**: Log everything in a buffer; only persist logs for requests that were slow, errored, or interesting.
```
on request complete:
    if response.status >= 500 or response.duration > p99_threshold:
        flush all buffered logs for this request
    else:
        discard DEBUG/TRACE logs, keep INFO+
```

**Error-biased sampling**: Always log 100% of errors. Sample successes.

Tail-based sampling is the best approach when your platform supports it — you get full detail for every interesting request and minimal noise for the rest.

### Log Volume Budgets

Set a log volume budget per service and track it like you track CPU or memory:

- Define a target: e.g., < 500 lines/sec at INFO level during normal traffic
- Alert if a service exceeds 2x its budget for > 10 minutes (probably a log spam bug)
- Review volume per service monthly — catch creep before it hits the bill

---

## Log Security — What You Must Never Log

Logging sensitive data is a compliance violation waiting to happen. Once it's in your log pipeline, it's replicated to indexes, dashboards, archives, and potentially third-party tools — all outside the access controls of your primary data store.

### The Never-Log List

| Data Type | Examples | Why |
|---|---|---|
| **Credentials** | Passwords, API keys, tokens, secrets, certificates | Compromise risk — logs are less secured than secret stores |
| **Payment data** | Full card numbers, CVVs, bank accounts | PCI-DSS violation |
| **Personal identifiers** | SSN, driver's license, passport numbers | Regulatory violation (GDPR, CCPA, HIPAA) |
| **Authentication tokens** | JWTs, session IDs, OAuth tokens | Session hijacking if logs are breached |
| **Full request/response bodies** | May contain any of the above | You can't guarantee what's in them |
| **Health/medical data** | Diagnoses, prescriptions, insurance IDs | HIPAA violation |

### Masking Strategies

When you need to log *something* about sensitive data (for debugging or audit), mask it:

```
# Full value — NEVER
{ "creditCard": "4111111111111111" }

# Masked — safe, still useful for debugging
{ "creditCard": "****1111" }

# Tokenized — reference that maps back in a secure system
{ "creditCardToken": "tok_abc123" }

# Hashed — useful for correlation without exposing the value
{ "emailHash": "sha256:a1b2c3..." }
```

### Implementation Approaches

| Approach | How It Works | Trade-offs |
|---|---|---|
| **Field-level masking in code** | Developer explicitly masks before logging | Most control, but relies on developers remembering |
| **Logging framework filters** | Pattern-based scrubbing at the log output layer | Catches accidental leaks, but regex-based masking is fragile |
| **Pipeline-level scrubbing** | Log shipping agent scrubs before ingestion | Defense in depth, but the data still exists briefly in memory/local disk |
| **Structured logging + allowlists** | Only log fields on an approved list; reject unknown fields | Strictest, but requires discipline and breaks flexibility |

The right answer is **defense in depth** — mask in code as the first line, scrub at the pipeline as the safety net, and audit periodically for leaks.

### Audit Logging vs. Application Logging

Don't conflate them. They serve different purposes and have different retention/access requirements:

| Aspect | Application Logs | Audit Logs |
|---|---|---|
| **Purpose** | Debugging, monitoring, troubleshooting | Compliance, forensics, accountability |
| **Content** | Technical events, errors, performance | Who did what, when, to what resource |
| **Retention** | Days to weeks | Months to years (regulatory) |
| **Access** | Engineering teams | Security, compliance, legal |
| **Mutability** | Can be dropped/sampled | Must be immutable and complete |

Keep audit logs in a separate, tamper-evident pipeline with stricter access controls.

---

## Common Mistakes

### 1. Logging and Throwing

```
# Bad — the same error gets logged at every layer as the exception bubbles up
try:
    process_order(order)
except OrderException as e:
    log.error("Failed to process order", e)  # logged here
    raise  # ...and logged again in the caller, and again in the global handler
```

Log at the point where the exception is **handled**, not at every point where it passes through. If you're rethrowing, don't log — let the handler that ultimately deals with it do the logging.

### 2. String Concatenation in Log Statements

```
# Bad — string is built even if DEBUG is disabled
log.debug("Processing order " + order.id + " with " + items.size() + " items")

# Good — parameterized; no string construction if level is disabled
log.debug("Processing order {} with {} items", order.id, items.size())
```

Parameterized logging avoids the CPU cost of string construction when the log level is disabled. At high throughput, this matters.

### 3. Logging Inside Tight Loops

```
# Bad — 1 million log lines for 1 million items
for item in items:
    log.info("Processing item {}", item.id)

# Better — log summary
log.info("Processing {} items", items.size())
# ...process...
log.info("Completed processing {} items, {} succeeded, {} failed", total, success, failure)
```

If you absolutely must log per-item, use DEBUG level and ensure it's off in production.

### 4. Using Log Levels as Categories

```
# Bad — using WARN because "the user should know about this"
log.warn("User {} logged in successfully")  # This is INFO, not WARN

# Bad — using ERROR for expected business outcomes
log.error("Payment declined for user {}")   # Expected path, not an error
```

Log levels indicate severity, not importance to the business. A declined payment is a normal outcome, not an error. An unexpected `NullPointerException` in the payment flow is an error.

### 5. Inconsistent Log Messages for the Same Event

```
# Different developers, different messages for the same event
log.info("Order created: " + orderId)
log.info("New order placed, id=" + orderId)
log.info("Created order {} for user {}", orderId, userId)
```

Standardize log messages for key events. Use constants or a shared logging utility. Inconsistent messages make searching and alerting unreliable.

### 6. No Context on Errors

```
# Bad — useless during an incident
log.error("Operation failed")

# Good — actionable
log.error("Failed to charge payment", {
    orderId: "12345",
    userId: "67890",
    paymentMethod: "CREDIT_CARD",
    errorCode: "GATEWAY_TIMEOUT",
    retryCount: 3,
    gatewayResponseMs: 30000
})
```

Every error log should include enough context to understand what was being attempted, what input was involved, and what specifically went wrong — without needing to reproduce the scenario.

### 7. Logging Sensitive Data "Just for Debugging"

```
# "I'll remove it before the PR" — no you won't
log.debug("Auth request: {}", requestBody)  # contains password
log.info("User details: {}", user.toString())  # toString() dumps everything including email, phone
```

Never log request/response bodies without a scrubbing layer. Override `toString()` on domain objects to exclude sensitive fields, or use a dedicated serializer for logging that strips PII.

---

## Logging in Different Contexts

### HTTP Request/Response Logging

Log at the boundary — when the request arrives and when the response is sent. Don't log inside every middleware or filter.

```
# Request entry (INFO)
{ "message": "Request received", "method": "POST", "path": "/orders", "traceId": "abc-123" }

# Request completion (INFO)
{ "message": "Request completed", "method": "POST", "path": "/orders", "status": 201,
  "durationMs": 45, "traceId": "abc-123" }

# Request failure (ERROR)
{ "message": "Request failed", "method": "POST", "path": "/orders", "status": 500,
  "durationMs": 120, "error": "PAYMENT_GATEWAY_TIMEOUT", "traceId": "abc-123" }
```

### Async / Event-Driven Logging

Messages and events don't have HTTP context, but they still need correlation:

- Propagate trace ID in message headers/metadata
- Log at consumption: message received, message type, partition/offset
- Log at completion: processing result, duration
- Log failures with the full message metadata (topic, partition, offset, key) — you'll need it to replay

### Scheduled Jobs / Batch Processing

```
# Job start
{ "message": "Job started", "job": "daily-reconciliation", "runId": "run-456" }

# Progress (for long jobs)
{ "message": "Job progress", "job": "daily-reconciliation", "runId": "run-456",
  "processed": 50000, "total": 120000, "elapsedMs": 45000 }

# Job completion
{ "message": "Job completed", "job": "daily-reconciliation", "runId": "run-456",
  "processed": 120000, "succeeded": 119500, "failed": 500, "durationMs": 98000 }
```

Always include a `runId` so you can isolate logs for a specific execution.

---

## Quick Reference Checklist

- [ ] All logs are structured JSON — no plain-text log lines in production
- [ ] Log levels are used correctly: INFO is the production baseline, DEBUG is off by default
- [ ] Every log line includes `timestamp`, `level`, `logger`, `message`, `traceId`, `service`, `environment`
- [ ] Context data goes in structured fields, not interpolated into the message string
- [ ] Trace IDs propagate across all service boundaries (HTTP, messaging, gRPC)
- [ ] No sensitive data in logs: no credentials, PII, tokens, payment data, or full request bodies
- [ ] Masking/scrubbing is implemented in code AND at the pipeline level (defense in depth)
- [ ] Errors are logged at the handling point, not at every layer the exception passes through
- [ ] No logging inside tight loops — use summaries or metrics instead
- [ ] Parameterized logging is used everywhere (no string concatenation)
- [ ] Log volume budget is defined per service and monitored
- [ ] Audit logs are separated from application logs with appropriate retention
- [ ] Key events use standardized, consistent log messages
- [ ] Sampling strategy is in place for high-throughput services

---

## Related Guides

- See Monitoring for the three pillars of observability (logging vs. metrics vs. traces) and when to use each pillar.
- See Cost for broader infrastructure cost management, including how log volume fits into overall service cost.
