# When to Use Message Queues

## What Is a Message Queue?

A message queue is a middleman that sits between two pieces of software. One side (the **producer**) drops a message into the queue. The other side (the **consumer**) picks it up and processes it later. The producer doesn't wait — it moves on immediately.

```
┌──────────┐         ┌───────────────┐         ┌──────────┐
│ Producer │  ────▶  │ Message Queue │  ────▶   │ Consumer │
└──────────┘         └───────────────┘         └──────────┘
    writes                 holds                  reads
   and leaves           until ready            when it can
```

Think of it like a mailbox. You drop a letter in, walk away, and the recipient checks it on their own schedule. You don't stand at their door waiting for them to read it.

### Key vocabulary

- **Producer** — the service that sends the message. It writes to the queue and moves on.
- **Consumer** — the service that reads and processes the message. It works at its own pace.
- **Broker** — the software that runs the queue (e.g., Kafka, RabbitMQ, SQS). It stores messages, manages delivery, and handles retries.
- **Queue (point-to-point)** — each message goes to exactly one consumer. Good for distributing tasks across workers.
- **Topic (pub/sub)** — each message goes to every subscriber. Good for broadcasting events.
- **Acknowledgment (ack)** — after processing a message, the consumer tells the broker "I'm done." If the consumer crashes before acking, the broker redelivers the message.
- **Dead Letter Queue (DLQ)** — a special queue where messages go after failing too many times. It prevents one bad message from blocking everything behind it.

---

## Quick Decision Guide

Each row links to a deep dive below.

| Factor | Use a Queue | Don't Use a Queue |
|--------|-------------|-------------------|
| Response dependency | Caller doesn't need the result right away | Caller needs an answer before it can continue |
| Service coupling | Services should evolve independently | Services are tightly related and change together |
| Traffic patterns | Bursty or unpredictable load | Steady, predictable traffic |
| Failure handling | Downstream failures shouldn't lose work | Failures are rare and can be retried inline |
| Number of consumers | Multiple services react to the same event | One producer talks to one consumer |
| Latency requirements | Seconds of delay are acceptable | Sub-second response is required |
| Ordering & sequencing | Events must be processed in order per entity | No ordering constraints |
| System complexity | Team can operate a broker and monitor queues | Small team, simple system, few moving parts |

---

## 1. Response Dependency

### Use a queue: the caller doesn't need the result right now

This is the single most important question. If the person or service that triggered the work doesn't need the answer before it can continue, a queue is a natural fit.

**Example — placing an order:**

Without a queue (synchronous):
```
User clicks "Place Order"
  → validate payment          (200ms)
  → send confirmation email   (500ms)
  → update inventory          (100ms)
  → notify warehouse          (300ms)
  → update analytics          (150ms)
  → return 200 to user       (total: 1,250ms)
```

The user waits 1.25 seconds because every downstream step runs inline. If the email server is slow, the user waits longer. If analytics is down, the entire order fails — even though the user doesn't care about analytics.

With a queue (asynchronous):
```
User clicks "Place Order"
  → validate payment          (200ms)
  → publish "OrderPlaced" event to queue
  → return 200 to user       (total: ~220ms)

Meanwhile, in the background:
  → email service picks up event → sends confirmation
  → inventory service picks up event → decrements stock
  → warehouse service picks up event → starts fulfillment
  → analytics service picks up event → logs metrics
```

The user gets a response in 220ms. The downstream work happens asynchronously. If the email server is slow or analytics is down, the user doesn't notice — the queue holds the messages until those services are ready.

**Why this matters for junior engineers:** The instinct is to do everything in one request-response cycle because that's how you learn to code — call a function, get a result. But in distributed systems, not every downstream action needs to complete before you can respond. Identifying which work is "must happen now" vs. "must happen eventually" is the key insight.

### Don't use a queue: the caller needs an answer immediately

If the calling service can't proceed without the result, a synchronous call is simpler and correct.

**Example — checking if a username is available:**
```
User types "cooldev99" → frontend calls POST /check-username → backend checks DB → returns "available" or "taken"
```

You can't queue this. The user is staring at the screen waiting for an answer. A synchronous HTTP call takes 50ms and gives them the result immediately. Putting a queue in the middle would mean... what? "We'll get back to you about whether your username is available"? That makes no sense.

**The rule:** If the caller is blocked until the work is done, use a synchronous call. Queues are for work that can happen later without blocking the caller.

---

## 2. Service Coupling

### Use a queue: services should evolve independently

Coupling means one service has to know about another service's existence, API, location, and availability. Queues break this coupling by introducing an intermediary — the producer publishes an event without knowing (or caring) who consumes it.

**Without a queue (tightly coupled):**
```
Order Service knows about:
  → Email Service     (HTTP call to email-service:8080/send)
  → Inventory Service (HTTP call to inventory-service:8080/decrement)
  → Analytics Service (HTTP call to analytics-service:8080/track)
```

Now imagine the fraud team wants to check every order. Without a queue, you have to:
1. Write code in the Order Service to call the Fraud Service
2. Add the Fraud Service's URL to the Order Service's config
3. Handle what happens if the Fraud Service is down
4. Redeploy the Order Service

Every new consumer means changing and redeploying the producer. The Order Service has become a God Service that knows about every downstream system.

**With a queue (decoupled):**
```
Order Service publishes "OrderPlaced" event → done.

Consumers (each subscribes independently):
  → Email Service
  → Inventory Service
  → Analytics Service
  → Fraud Service (new — added without touching Order Service)
```

Adding the Fraud Service means deploying one new consumer that subscribes to the "OrderPlaced" topic. The Order Service doesn't change at all. It doesn't even know the Fraud Service exists.

**Why this matters:** In a real system, teams own different services. If adding a fraud check requires the orders team to change their code, you've created an organizational bottleneck — the fraud team is blocked until the orders team has capacity to make the change. Queues let each team move independently.

### Don't use a queue: services are tightly related and change together

If two services are owned by the same team, deployed together, and always change in lockstep — a direct call is simpler. The decoupling benefit of a queue only matters when services need to evolve independently.

**Example:** A service and its internal helper (like a validation library or a formatting utility). These aren't independent services that need decoupling — they're parts of the same thing. A direct function call or HTTP request is appropriate.

**The smell test:** If you'd never deploy service A without also deploying service B, they probably don't need a queue between them.

---

## 3. Traffic Patterns

### Use a queue: bursty or unpredictable load

Queues act as a buffer. They absorb spikes in traffic and let consumers process at a steady, sustainable rate. Without a queue, a traffic spike hits your downstream services directly and they either slow down, drop requests, or crash.

**Example — Black Friday flash sale:**
```
Normal traffic:  100 orders/sec
Flash sale:      10,000 orders/sec (for 5 minutes)

Without a queue:
  → Inventory service gets 10,000 requests/sec
  → Database connection pool exhausted
  → Requests start timing out
  → Users see errors
  → Orders are lost

With a queue:
  → 10,000 orders/sec written to queue (queues are built for this)
  → Inventory service consumes at 500/sec (its sustainable rate)
  → Queue depth grows during the spike
  → Queue drains over the next ~90 seconds after the spike
  → Zero requests lost, zero errors
```

Visually:
```
Incoming rate:   ████████████████░░░░░░░░░░░░░░░░
Queue depth:     ░░░░████████████████████░░░░░░░░░
Consumer rate:   ████████████████████████████████░░
                 ↑ spike starts    ↑ spike ends  ↑ queue drained
```

The queue converts an unpredictable, bursty input into a smooth, predictable output. Your downstream services don't need to scale to peak traffic — they only need to handle the average.

**Why this matters for junior engineers:** Your first instinct might be "just auto-scale the downstream services." Auto-scaling works but has lag time (minutes to spin up new instances) and costs money (you're paying for peak capacity even during the spike). A queue lets you handle spikes with fixed capacity.

### Don't use a queue: steady, predictable traffic

If your traffic is consistent and your downstream services can always keep up, a queue adds complexity without benefit. The buffering advantage only matters when there's something to buffer.

**Example:** An internal admin tool used by 5 people. Traffic is 10 requests/minute, always. Adding a queue here is over-engineering — a direct HTTP call is simpler to build, debug, and maintain.

---

## 4. Failure Handling

### Use a queue: downstream failures shouldn't lose work

When a consumer crashes or a downstream service is unavailable, the queue holds the message until it can be delivered. Without a queue, if the downstream service is down when you call it, the request is lost unless you build retry logic yourself.

**Without a queue:**
```
Order Service calls Inventory Service
  → Inventory Service is down (deploying, crashed, network issue)
  → HTTP call fails with timeout/5xx
  → Now what?
    Option A: Return error to user ("sorry, try again") — bad UX, user might not retry
    Option B: Retry in the Order Service — how many times? How long between retries?
             What if the Order Service restarts mid-retry? The work is lost.
    Option C: Write to a "retry table" in your database — congratulations,
             you just built a worse message queue
```

**With a queue:**
```
Order Service publishes "OrderPlaced" event
  → Inventory Service is down
  → Message sits in the queue
  → Inventory Service comes back up
  → Picks up the message and processes it
  → Acks it
  → No data lost, no user-facing error, no retry logic in your code
```

**The DLQ safety net:** What if the message itself is bad (malformed data, a bug in the consumer's processing logic)? Without a DLQ, the consumer retries the same broken message forever — a "poison message" that blocks the entire queue. With a DLQ configured, after N failed attempts (usually 3-5), the message moves to the DLQ. The main queue keeps flowing. An engineer can inspect the DLQ later, fix the bug, and replay the messages.

```
Main Queue → Consumer tries to process → fails 3 times → Dead Letter Queue
                                                              ↓
                                                    Engineer investigates
                                                    Fixes the bug
                                                    Replays the messages
```

### Don't use a queue: failures are rare and retryable inline

If the downstream service has 99.99% uptime and your use case tolerates the occasional failure (e.g., a best-effort analytics ping), a simple HTTP call with a basic retry (1-2 attempts) is fine. Don't add a queue just to handle a failure scenario that happens once a month.

**The trade-off question:** How bad is it if this message is lost? If the answer is "nobody notices" — you probably don't need a queue. If the answer is "we lose money or data" — you do.

---

## 5. Number of Consumers

### Use a queue: multiple services react to the same event

This is the pub/sub pattern — one event triggers many independent reactions. Without a queue, the producer has to know about and call every consumer individually.

**Example — user signs up:**

Without a queue:
```
Signup Service calls:
  → Welcome Email Service
  → CRM Service (add to customer database)
  → Analytics Service (track signup)
  → Trial Provisioning Service (create free trial)
  → Referral Service (check if referred by another user)
```

Five HTTP calls. If any of them is slow, the signup is slow. If any of them is down, you have to decide: fail the signup (bad) or silently drop that step (also bad). Adding a sixth consumer means changing the Signup Service.

With a queue:
```
Signup Service publishes "UserSignedUp" event → done.

Each consumer subscribes to the topic independently:
  → Welcome Email Service
  → CRM Service
  → Analytics Service
  → Trial Provisioning Service
  → Referral Service
  → [future: any new consumer, zero changes to Signup Service]
```

Each consumer processes the event independently. If one is slow or down, it doesn't affect the others. The Signup Service returns immediately after publishing.

### Don't use a queue: one producer talks to one consumer

If service A always and only talks to service B, and no one else cares about the event, a direct call is simpler. A queue adds a middleman (the broker) that you now have to deploy, monitor, and maintain — and for one-to-one communication, the decoupling benefit is minimal.

**When to reconsider:** Even in a one-to-one scenario, a queue might still be worth it if factors #1 (response dependency), #3 (traffic patterns), or #4 (failure handling) apply. The number-of-consumers factor is just one input to the decision.

---

## 6. Latency Requirements

### Use a queue: seconds of delay are acceptable

Queues add latency. A message must be serialized, sent to the broker, written to disk (for durability), read by the consumer, deserialized, and processed. Even in the best case, this adds milliseconds to seconds of delay compared to a direct call.

For many use cases, this delay is invisible to the user:
- Sending a confirmation email — 5 seconds late? Nobody notices.
- Updating a search index — 2 seconds behind the database? Completely fine.
- Syncing data to a reporting system — minutes behind? Expected.

### Don't use a queue: sub-second response is required

If the user is waiting on the screen for a result, or if the calling service needs the answer to proceed, the queue's inherent latency is a problem.

**Examples where queues hurt latency:**
- **Autocomplete / typeahead** — the user is typing and expects suggestions in <100ms. A queue round-trip would make every keystroke feel laggy.
- **Auth / permission checks** — "can this user access this resource?" needs an answer in milliseconds, not seconds. A synchronous call to the auth service (or even better, a local cache) is the right choice.
- **Real-time gaming** — player actions need sub-10ms processing. Queues are orders of magnitude too slow.
- **Health checks** — an upstream load balancer pinging `/health` every second needs an immediate response to make routing decisions.

**A common mistake junior engineers make:** Adding a queue to be "more scalable" or "more resilient" without realizing they've added 50-500ms of latency to a path where the user is waiting. Always measure the latency impact and ask: "Is the user waiting for this?"

---

## 7. Ordering & Sequencing

### Use a queue: events must be processed in order per entity

Some events only make sense if processed in order. A bank account can't process "withdraw $100" before "deposit $500" — the order matters for correctness.

Most message brokers support **partition-based ordering**: messages with the same key (e.g., user ID, account number) go to the same partition and are consumed in order.

```
Messages:
  { key: "user-123", event: "AccountCreated" }
  { key: "user-123", event: "DepositMade", amount: 500 }
  { key: "user-123", event: "WithdrawalMade", amount: 100 }
  { key: "user-456", event: "AccountCreated" }

Partition 0 (user-123):  AccountCreated → DepositMade → WithdrawalMade  ✅ in order
Partition 1 (user-456):  AccountCreated                                 ✅ in order
```

Messages for different users can be processed in parallel (across partitions), but messages for the same user are always processed sequentially (within a partition).

**Important caveat:** Ordering comes at a cost. If you need strict ordering for a key, only one consumer can read from that partition at a time. This limits parallelism. If you have a "hot key" (one user generating thousands of events), that partition becomes a bottleneck.

### Don't use a queue (for ordering): no ordering constraints

If your events are independent and can be processed in any order — metrics, logs, notifications — you don't need partition-based ordering. You can use a simple work queue where any consumer picks up any message, maximizing parallelism.

**The nuance:** Even without a queue, you might still need ordering — but you'd handle it with timestamps, sequence numbers, or database transactions. Queues with partition ordering give you this guarantee at the infrastructure level instead of building it in application code.

---

## 8. System Complexity

### Use a queue: when the benefits outweigh the operational cost

Queues are not free. Running a message broker means:
- **Deployment & maintenance** — the broker is another system to deploy, configure, upgrade, and patch.
- **Monitoring** — you need dashboards for queue depth (how many messages are waiting), consumer lag (how far behind consumers are), error rates, and DLQ size.
- **Debugging** — when something goes wrong, the message flow is now: producer → broker → consumer. You can't just look at one service's logs. You need to trace a message across three systems.
- **Data serialization** — producers and consumers need to agree on message format (JSON, Avro, Protobuf). Schema changes require coordination.
- **Exactly-once is hard** — most brokers provide at-least-once delivery, meaning consumers can see duplicates. Your consumer code must be **idempotent** (processing the same message twice produces the same result). This is extra design work that doesn't exist with synchronous calls.

**What "idempotent" means in practice:**
```
# Non-idempotent (dangerous with at-least-once delivery):
def process_order(order):
    charge_credit_card(order.amount)  # if called twice, charges twice!

# Idempotent (safe):
def process_order(order):
    if already_processed(order.id):   # check if we've seen this before
        return                        # skip duplicate
    charge_credit_card(order.amount)
    mark_as_processed(order.id)       # record that we handled it
```

### Don't use a queue: small team, simple system, few moving parts

If you have a monolith or 2-3 services, a small team, and predictable traffic — a queue adds operational complexity that you don't need. The team now has to learn broker administration, message serialization, consumer group management, and distributed tracing. That's a lot of overhead for a system that could use HTTP calls.

**The progression in practice:**
1. **Start simple** — direct HTTP calls between services.
2. **Add a queue when you feel the pain** — a downstream service can't keep up, you're losing work when services go down, or you keep adding more consumers to the same event.
3. **Don't start with a queue** — premature optimization. You'll over-invest in infrastructure before you understand your actual access patterns.

---

## Delivery Guarantees

Understanding delivery semantics is critical to choosing and using queues correctly.

| Guarantee | What it means | When messages are lost or duplicated | Real-world analogy |
|---|---|---|---|
| **At-most-once** | Message is delivered zero or one times | Message may be lost if the broker or consumer crashes before processing | Regular mail — the post office tries once, no tracking |
| **At-least-once** | Message is delivered one or more times | Message is never lost, but may be delivered twice if the consumer crashes after processing but before acking | Certified mail with retry — if delivery fails, they try again |
| **Exactly-once** | Message is processed exactly once | No loss, no duplication | Doesn't truly exist in distributed systems — it's faked with at-least-once + idempotent consumers + transactional coordination |

**In practice, at-least-once + idempotent consumers is the standard approach.** It's the safest and most pragmatic choice. You accept that duplicates will happen and design your consumers to handle them gracefully.

---

## Common Patterns

### Work Queue (Competing Consumers)
Multiple consumers pull from the same queue. Each message goes to one consumer. This distributes load across workers — the queue acts as a task distributor.

```
              ┌──────────┐
         ┌──▶ │ Worker 1 │  processes message A
┌───────┐│    └──────────┘
│ Queue ├┤    ┌──────────┐
└───────┘│──▶ │ Worker 2 │  processes message B
         │    └──────────┘
         │    ┌──────────┐
         └──▶ │ Worker 3 │  processes message C
              └──────────┘
```

**Use for:** image processing, report generation, batch jobs — any work that can be parallelized.

### Fan-Out (Pub/Sub)
One message goes to all subscribers. Each subscriber processes independently.

```
                       ┌───────────────┐
                  ┌──▶ │ Email Service  │  sends confirmation
┌───────────┐    │    └───────────────┘
│  "Order   │────┤    ┌───────────────┐
│  Placed"  │    ├──▶ │ Inventory Svc │  decrements stock
│   Topic   │    │    └───────────────┘
└───────────┘    │    ┌───────────────┐
                  └──▶ │ Analytics Svc │  logs metrics
                       └───────────────┘
```

**Use for:** event-driven architectures where multiple systems care about the same event.

### Saga (Choreography)
A multi-step business process coordinated through events. Each service does its part and publishes the next event. If any step fails, compensating events undo previous steps.

```
Order Created → Payment Charged → Inventory Reserved → Shipment Scheduled
                                        ↓ (fails)
                              Payment Refunded ← Inventory Release Failed
```

**Use for:** distributed transactions that span multiple services (checkout, booking, account provisioning).

---

## Common Mistakes

1. **Using a queue when a direct call would work** — If you have two services, one consumer, predictable traffic, and you need a response — just use HTTP. A queue here adds complexity without benefit.
2. **Not making consumers idempotent** — With at-least-once delivery, duplicates will happen. If your consumer charges a credit card without checking if it already charged for that order, you will double-charge customers.
3. **Ignoring the DLQ** — Messages that fail processing need to go somewhere. Without a DLQ, one poison message blocks every message behind it. With a DLQ that nobody monitors, bad messages pile up silently until someone notices data is missing.
4. **Putting too much data in the message** — Messages should be small (IDs, metadata, event type). If you need to send a 10MB file, put it in blob storage and send a reference (the URL or storage key) in the message.
5. **Assuming ordering is global** — Most brokers guarantee ordering within a partition, not across the entire topic. If you need all events in total order, you're limited to a single partition (and a single consumer), which kills throughput.
6. **Forgetting about schema evolution** — Your message format is a contract between producer and consumer. If the producer adds a field, removes a field, or changes a type, consumers that expect the old format will break. Use a schema registry or versioning strategy from day one.
7. **Starting with a queue instead of growing into one** — Build the simple thing first (direct calls). When you hit a real problem (downstream can't keep up, losing work on failures, too many consumers), add a queue to solve that specific problem. Don't add infrastructure in anticipation of problems you might never have.

---

## Related

See Monolith vs Microservices for broader architecture decisions around service boundaries and coupling.
