# Defending Against Spam Clicks

When a user hammers a button — liking a photo, submitting a form, cancelling an order — without proper defenses you can end up with duplicate records, race conditions, or inconsistent state. Here's the full playbook.

---

## The Core Problem

A single UI action maps to a network request. If that request can be fired multiple times before the first one resolves, you get:

- **Duplicate mutations** (two likes, two rejections, two charges)
- **Race conditions** (last write wins, inconsistent final state)
- **Database constraint violations** (or silent duplicates if no constraints exist)
- **Wasted server resources** (N identical inflight requests)

---

## Frontend Defenses

These are your first line of defense. They improve UX and reduce noise, but **they are not sufficient alone** — a motivated user (or Postman) bypasses them entirely.

### 1. Disable the Button After Click

The most basic defense. Immediately disable the button on click and only re-enable it once you have a confirmed response.

```tsx
const [loading, setLoading] = useState(false);

async function handleCancel() {
  if (loading) return; // guard clause
  setLoading(true);
  try {
    await cancelOrder(orderId);
  } finally {
    setLoading(false);
  }
}

<Button onClick={handleCancel} disabled={loading}>
  {loading ? <Spinner /> : "Cancel Order"}
</Button>
```

**Why the guard clause matters:** React batching doesn't guarantee `disabled` propagates before a second click event fires. The explicit `if (loading) return` is the real lock.

### 2. Optimistic UI

Update the local state immediately (before the server responds), then reconcile on success/failure. This removes any perceived incentive to click again.

```tsx
// Mark as liked instantly
setLiked(true);
try {
  await likePhoto(photoId);
} catch {
  setLiked(false); // revert on failure
  toast.error("Failed to like. Try again.");
}
```

### 3. Debounce / Throttle

For actions where partial clicks are expected (search-as-you-type, scroll events), debounce collapses rapid calls into one:

```ts
import { debounce } from 'lodash';

const handleLike = debounce(async (id) => {
  await likePhoto(id);
}, 300);
```

For one-shot actions (reject, submit), **prefer the disable pattern over debounce** — debouncing introduces a delay that feels broken.

### 4. Loading State as Visual Feedback

Always show a loading indicator. Users spam-click when the UI feels unresponsive. Instant visual feedback kills the urge.

---

## Backend Defenses

These are **mandatory**. Frontend guards are UX polish; backend guards are correctness.

### 1. Idempotency Keys — Most Important

The client generates a unique key per logical operation and sends it with the request. The server uses it to deduplicate — if the same key arrives twice, return the original response instead of processing again. This is the single most important backend defense against spam clicks, handling both accidental double-clicks and network retries.

See Retries for idempotency key design and patterns (implementation examples, conditional requests, natural idempotency through API design).

### 2. Database-Level Unique Constraints

The absolute last line of defense. If every other layer fails, the database should make duplicate writes impossible.

```sql
-- Prevent a user from liking the same photo twice
ALTER TABLE photo_likes
  ADD CONSTRAINT uq_user_photo UNIQUE (user_id, photo_id);

-- Prevent an order from being cancelled more than once
ALTER TABLE order_actions
  ADD CONSTRAINT uq_order_action UNIQUE (order_id, action_type);
```

Catch the unique constraint violation in your service layer and return a `409 Conflict` (not a `500`).

### 3. Optimistic Locking

Use a version column to prevent stale writes from racing each other.

```java
// JPA example
@Entity
public class Order {
    @Version
    private Long version;
    private OrderStatus status;
}

// Only updates if version matches what you read
UPDATE orders SET status = 'CANCELLED', version = 2
WHERE id = ? AND version = 1;
-- 0 rows updated → someone else got there first → throw OptimisticLockException
```

Return a `409` when the lock fails. The client can refresh and retry intentionally.

### 4. Distributed Locking (Redis)

For critical operations where you need to serialize access across multiple service instances:

```java
String lockKey = "order:cancel:" + orderId;
boolean acquired = redis.set(lockKey, userId, SET_IF_ABSENT, 30, SECONDS);

if (!acquired) {
    throw new ConflictException("Action already in progress");
}

try {
    cancelOrder(orderId);
} finally {
    redis.del(lockKey);
}
```

Use this for actions that are expensive, non-idempotent by nature, or must be serialized (e.g., financial operations).

### 5. Rate Limiting

Limit how many times a given user can hit a specific endpoint in a time window. Return `429 Too Many Requests` with a `Retry-After` header. Most API gateways (Kong, Apigee, Istio service mesh) support this natively without custom code.

See Rate Limiting for algorithm details (token bucket, sliding window), enforcement layers, and HTTP 429 response standards.

### 6. Check-Then-Act (Stateful Guard)

Before processing, verify the resource is in a valid state for the action:

```java
Order order = orderRepository.findById(orderId);

if (order.getStatus() != OrderStatus.ACTIVE) {
    throw new ConflictException("Order already actioned: " + order.getStatus());
}

order.setStatus(OrderStatus.CANCELLED);
orderRepository.save(order);
```

Combine with a unique constraint or optimistic lock — `check-then-act` alone has a TOCTOU race condition under concurrent load.

---

## Recommended Layered Strategy

| Layer | Mechanism | What It Catches |
|---|---|---|
| Frontend | Disable + loading state | Accidental double-clicks |
| Frontend | Optimistic UI | Removes UX motivation to re-click |
| Network/Gateway | Rate limiting | Sustained hammering |
| Service | Idempotency keys | Network retries + spam |
| Service | Check-Then-Act + optimistic lock | Race conditions |
| Database | Unique constraints | Everything that slips through |

**You don't pick one — you stack them.** Frontend guards improve UX. Backend guards ensure correctness. Database constraints are the safety net.

---

## What NOT to Do

- **Frontend-only defense** — trivially bypassed with dev tools or curl
- **Debounce on one-shot actions** — introduces artificial delay, still doesn't prevent duplicates
- **Catch-and-swallow DB constraint errors as 500s** — the client can't distinguish "your request worked (duplicate)" from "server is broken"
- **Using request timestamp as a dedup key** — clock skew and millisecond precision make this unreliable at scale

---

## Related Topics

- Retries — idempotency key design, conditional requests, safe retry patterns
- Rate Limiting — rate limit algorithms, enforcement layers, HTTP 429 standards

---

## Quick Reference

1. **UI**: Disable the action button on click, show spinner, re-enable only on API error
2. **API Gateway**: Rate limit write endpoints to 1 req/5s per user per resource
3. **Service layer**: Accept an idempotency key from the client, deduplicate in Redis with 1h TTL
4. **DB**: `UNIQUE(resource_id, action_type)` constraint on the actions table
5. **JPA entity**: `@Version` on the entity for optimistic locking
