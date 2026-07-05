# Rate Limiting Best Practices

Rate limiting is the practice of controlling how many times a user (or system) can perform an action within a given time window. It's the complement to spam-click defense — where spam-click defense handles rapid burst clicks on a single action, rate limiting handles sustained or repeated abuse over time.

---

## Frontend vs. Backend — The Short Answer

**Frontend rate limiting is UX. Backend rate limiting is correctness.**

You need both, but for completely different reasons:

| | Frontend | Backend |
|---|---|---|
| Purpose | Prevent accidental rapid actions | Enforce hard business rules |
| Bypassable? | Yes — always | No (if implemented correctly) |
| Scope | Single browser session | All clients, all sessions |
| Response time | Instant (no network) | Adds latency |
| Protects against | Confused users | Malicious users, automation, scrapers |

---

## Frontend Rate Limiting

Frontend limiting is about **feedback and prevention of accidents**, not security.

### 1. Cooldown State

After an action fires, put the button in a "cooldown" state for N seconds before it can be used again.

```tsx
const COOLDOWN_MS = 5000;
const [cooldownUntil, setCooldownUntil] = useState<number | null>(null);

const isCoolingDown = cooldownUntil !== null && Date.now() < cooldownUntil;

async function handleAction() {
  if (isCoolingDown) return;
  setCooldownUntil(Date.now() + COOLDOWN_MS);
  await performAction();
}

<Button onClick={handleAction} disabled={isCoolingDown}>
  {isCoolingDown ? "Please wait..." : "Submit"}
</Button>
```

### 2. Countdown Timer Feedback

Show the user exactly how long they need to wait. Reduces frustration and support tickets.

```tsx
const [secondsLeft, setSecondsLeft] = useState(0);

// Decrement every second while cooling down
useEffect(() => {
  if (secondsLeft <= 0) return;
  const timer = setTimeout(() => setSecondsLeft(s => s - 1), 1000);
  return () => clearTimeout(timer);
}, [secondsLeft]);

<Button disabled={secondsLeft > 0}>
  {secondsLeft > 0 ? `Try again in ${secondsLeft}s` : "Resend Code"}
</Button>
```

### 3. Client-Side Token Bucket (for power users)

For more granular control — allow bursts but cap sustained rate:

```ts
class TokenBucket {
  private tokens: number;
  private lastRefill: number;

  constructor(private capacity: number, private refillRatePerSec: number) {
    this.tokens = capacity;
    this.lastRefill = Date.now();
  }

  consume(): boolean {
    const now = Date.now();
    const elapsed = (now - this.lastRefill) / 1000;
    this.tokens = Math.min(this.capacity, this.tokens + elapsed * this.refillRatePerSec);
    this.lastRefill = now;

    if (this.tokens >= 1) {
      this.tokens -= 1;
      return true; // allowed
    }
    return false; // rate limited
  }
}

const bucket = new TokenBucket(5, 1); // 5 burst, 1/sec sustained

function handleAction() {
  if (!bucket.consume()) {
    toast.warning("Slow down — you're doing that too fast.");
    return;
  }
  performAction();
}
```

### What Frontend Rate Limiting Does NOT Do

- It does not stop automation (scripts, curl, Postman)
- It does not protect against multi-tab abuse
- It does not protect against a user clearing browser state and reloading
- It resets on page refresh

---

## Backend Rate Limiting — The Real Defense

Backend rate limiting is the enforcer. It runs server-side and cannot be bypassed by the client.

### Core Concepts: Algorithms

#### Fixed Window

Count requests in a fixed time bucket (e.g., per minute). Simple but has a burst problem at window boundaries.

```
Window: 00:00 - 01:00 → 100 requests allowed
Window: 01:00 - 02:00 → 100 requests allowed

Problem: User sends 100 at 00:59 and 100 at 01:01 → 200 requests in 2 seconds
```

#### Sliding Window Log

Track the exact timestamp of each request. Check how many fall within the last N seconds. Accurate but memory-intensive.

#### Sliding Window Counter (Recommended)

A hybrid — approximate sliding window using two fixed window counters. Accurate enough for most use cases, very memory efficient.

```
current_count = prev_window_count * overlap_ratio + current_window_count
```

#### Token Bucket (Recommended for bursts)

A bucket fills at a fixed rate and holds up to N tokens. Each request costs a token. Allows controlled bursts without a hard cliff.

```
Capacity: 10 tokens
Refill: 1 token/second

Burst of 10 → allowed
11th immediate request → denied
After 1 second → 1 new token, 1 more request allowed
```

#### Leaky Bucket

Requests enter a queue and are processed at a fixed rate. Smooths traffic but adds latency.

---

### Where to Enforce Backend Rate Limits

#### Layer 1: API Gateway (Preferred for most cases)

The API gateway (Kong, Apigee, Istio service mesh) is the best place for broad rate limiting. No custom code, handles it before your service even sees the request.

```yaml
# Kong rate-limit plugin example
plugins:
  - name: rate-limiting
    config:
      minute: 60        # 60 req/min per user
      hour: 1000
      policy: redis     # distributed, shared across instances
      limit_by: consumer
```

Returns `429 Too Many Requests` automatically. Add a `Retry-After` header for good UX.

#### Layer 2: Service Layer (For business-rule limits)

For limits that are specific to your domain logic (e.g., "a user can only perform 10 actions per hour on a resource"), implement in the service layer with Redis.

```java
@Service
public class RateLimiterService {

    private final RedisTemplate<String, String> redis;

    // Sliding window counter using Redis
    public boolean isAllowed(String userId, String action, int maxRequests, Duration window) {
        String key = "ratelimit:" + action + ":" + userId;
        long now = System.currentTimeMillis();
        long windowStart = now - window.toMillis();

        // Remove entries outside the window
        redis.opsForZSet().removeRangeByScore(key, 0, windowStart);

        // Count remaining
        Long count = redis.opsForZSet().zCard(key);
        if (count != null && count >= maxRequests) {
            return false; // rate limited
        }

        // Add current request
        redis.opsForZSet().add(key, String.valueOf(now), now);
        redis.expire(key, window);
        return true;
    }
}
```

Usage in a controller:
```java
@PostMapping("/orders/{id}/cancel")
public ResponseEntity<?> cancelOrder(@PathVariable String id, @RequestHeader("X-User-Id") String userId) {
    if (!rateLimiter.isAllowed(userId, "order:cancel", 10, Duration.ofHours(1))) {
        return ResponseEntity
            .status(HttpStatus.TOO_MANY_REQUESTS)
            .header("Retry-After", "3600")
            .body("Rate limit exceeded. Max 10 cancellations per hour.");
    }
    return orderService.cancel(id, userId);
}
```

#### Layer 3: Database (Last resort)

For very coarse limits you can enforce at the DB level with a rate-tracking table, but this is slow and rarely preferred over Redis.

---

### Rate Limit Keys — What to Limit By

Choosing the right key is critical. Too broad = useless. Too narrow = easy to bypass.

| Key | Use Case | Bypass Risk |
|---|---|---|
| IP address | Unauthenticated endpoints | VPNs, shared IPs |
| User ID | Authenticated actions | Low — tied to account |
| User ID + action | Per-action limits | Very low |
| User ID + resource ID | "Max N actions on resource X" | Very low |
| API key | Partner/service accounts | Low |

For authenticated APIs: **always key by `userId + action`** at minimum. IP-based limits alone are too coarse and penalize shared networks (office Wi-Fi, corporate proxies).

---

### HTTP Response Standards

Always return proper HTTP semantics when rate limiting:

```
HTTP/1.1 429 Too Many Requests
Retry-After: 60
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1719792000

{
  "error": "rate_limit_exceeded",
  "message": "You've exceeded the limit of 100 requests per minute.",
  "retryAfter": 60
}
```

| Header | Meaning |
|---|---|
| `Retry-After` | Seconds until the client may retry |
| `X-RateLimit-Limit` | Max requests allowed in window |
| `X-RateLimit-Remaining` | Requests remaining in current window |
| `X-RateLimit-Reset` | Unix timestamp when the window resets |

---

### Graduated Response (Recommended)

Instead of a hard block, consider escalating consequences:

| Violation Level | Response |
|---|---|
| 1st breach | `429` + short cooldown (30s) |
| 2nd breach | `429` + longer cooldown (5min) + warning message |
| 3rd breach | `429` + account flag for review |
| Sustained abuse | Temporary account suspension |

This is more user-friendly for accidental abuse and still blocks intentional abuse.

---

## Recommended Strategy by Endpoint Type

| Endpoint Type | Frontend | Gateway | Service Layer |
|---|---|---|---|
| Auth (login, OTP) | Cooldown UI | 5 req/min per IP | Lockout after N failures |
| Write actions (reject, submit) | Disable button | 60 req/min per user | 10 per hour per resource |
| Search / read | None needed | 200 req/min per user | Usually not needed |
| File upload | Progress indicator | 10 req/min per user | Size + count limits |
| Webhooks / external APIs | N/A | Per-key limits | Circuit breaker |

---

## What NOT to Do

- **Frontend-only rate limiting** — bypassed by any HTTP client
- **IP-only limits on authenticated endpoints** — punishes innocent users behind shared IPs
- **Silent rate limiting** — always tell the client they're limited and when to retry
- **No `Retry-After` header** — clients will hammer you on retry without it
- **Same limit for all users** — consider tiered limits (admins vs. regular users vs. service accounts)
- **In-memory rate limiting on multi-instance services** — state is per-instance, limits are ineffective; always use Redis for distributed state

---

## Related Topics

- Spam Click Defense — UI-specific protections (button disable, optimistic UI, debounce)
- Retries — idempotency keys, safe retry design, `Retry-After` handling from the client side

---

## Quick Reference

1. **UI**: Cooldown button state after write actions (5s minimum), show countdown
2. **API Gateway**: 60 req/min per user on write endpoints
3. **Service layer**: Redis sliding window — max 10 actions per user per hour per resource
4. **Key**: `ratelimit:{action}:{userId}` — not IP-based
5. **Response**: `429` with `Retry-After` + `X-RateLimit-*` headers
6. **Auth endpoints**: Stricter — 5 req/min per IP, lockout after 5 failures
