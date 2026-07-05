# Application Code vs. Database vs. Configuration: Where Does It Belong?

One of the most consequential decisions in software design is choosing *where* to put a piece of logic or data. The wrong choice creates tight coupling, fragile deployments, or unnecessary database churn. This guide lays out a clear mental model for making that call.

---

## The Three Buckets

| Layer | Examples | Characteristics |
|---|---|---|
| **Application Code** | Java, JavaScript, Python | Versioned, deployed, compiled/interpreted at runtime |
| **Database** | PostgreSQL, Cassandra, Cosmos DB | Persistent, queryable, shared across instances |
| **Configuration** | Centralized configuration service, CMS, environment variables, feature flags | Externalized, often changeable without a redeploy |

---

## Application Code

### Put it here when...

- The logic is **algorithmic** — it computes, transforms, validates, or orchestrates.
- The behavior is **tightly coupled to the code** — changing it without a deployment would break something.
- It represents a **business rule that rarely changes** and is central to the service contract.
- It's **type-safe and testable** — you want the compiler or test suite to catch regressions.
- It has **no business-meaningful variance** across environments (dev vs. prod behave identically).

### Examples

```java
// Business logic: discount calculation — belongs in code
public BigDecimal applyDiscount(BigDecimal price, DiscountType type) {
    return switch (type) {
        case PERCENTAGE -> price.multiply(BigDecimal.valueOf(0.9));
        case FLAT -> price.subtract(BigDecimal.TEN);
    };
}
```

```js
// Validation logic — belongs in code
function isValidEmail(email) {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
}
```

### Warning signs you're putting too much in code

- You need a deployment to tweak a threshold (e.g., max retry count, timeout value).
- Different environments need different values for the same constant.
- Business stakeholders need to change it frequently and can't touch code.

---

## Database

### Put it here when...

- The data is **user-generated or operational** — it varies per record, user, or transaction.
- You need to **query, filter, join, or aggregate** it.
- It needs to **persist across restarts** and be shared across service instances.
- It has a **lifecycle** — created, updated, deleted independently of a deployment.
- Multiple services or teams need **read access** to the same source of truth.

### Examples

- User profiles, orders, inventory counts, product catalog entries.
- Audit logs, event history, transaction records.
- Lookup tables that change based on business operations (e.g., store hours per location).
- Relationship data (which user belongs to which org).

### Warning signs you're using the DB wrong

- You're storing a single row that never changes per environment — that's a config.
- You're writing code that reads the DB just to get a "constant" on every request — cache it or externalize it.
- You're storing logic in stored procedures that could be in the application layer with proper testing.
- You're abusing a DB as a message queue or pub/sub system.

---

## Configuration (Centralized Config Service, CMS, Environment Variables, Feature Flags)

### Put it here when...

- The value **varies by environment** (dev, staging, prod) without code changes.
- You need to **change it without a redeploy** — operational tuning, kill switches, rate limits.
- It's a **secret or credential** — never hardcode these in source (use a secrets manager/Vault/env vars).
- It controls **feature rollout** — you want to enable/disable something for a subset of users or traffic. See Feature Flags for feature flag lifecycle and best practices.
- It represents an **operational knob** that ops or a product manager might need to turn during an incident.

### Centralized Config vs. CMS vs. Environment Variables

| Tool | Best For |
|---|---|
| **Centralized configuration service** | Service-level runtime config that may vary by environment/region; supports dynamic refresh |
| **CMS** (Content Management) | Externalized content strings, marketing copy, localized text |
| **Environment variables** | Secrets, infrastructure endpoints, immutable per-pod config set at deploy time |
| **Feature flags** | Progressive rollout, A/B experiments, kill switches tied to user segments |

### Examples

```yaml
# Centralized config — operational tuning, not code
maxRetries: 3
timeoutMs: 5000
circuitBreakerThreshold: 0.5
```

```
# Environment variable — secrets and infra endpoints
DATABASE_URL=jdbc:postgresql://prod-db:5432/orders
SECRET_PATH=/prod/stripe/api-key
```

### Warning signs you're over-configuring

- Business logic is encoded in config values that only make sense if you read the code alongside them — that logic belongs in code.
- You have so many flags that turning them all on/off is a deployment-level event anyway.
- Config values are never actually different between environments — they're constants, so put them in code.

---

## Decision Flowchart

```
Does it change per user/record/transaction?
  └─ YES → Database

Does it change per environment or need to change without a redeploy?
  └─ YES → Configuration (config service / env var / feature flag)

Is it a secret or credential?
  └─ YES → Configuration (env var / secrets manager)

Does it need to be queried, joined, or aggregated across records?
  └─ YES → Database

Is it algorithmic logic, validation, or transformation?
  └─ YES → Application Code

Is it user-facing content or copy?
  └─ YES → CMS / Configuration
```

---

## Real-World Scenarios

| Scenario | Wrong choice | Right choice |
|---|---|---|
| Max items per cart | Hardcoded `50` in Java | Config service — ops may need to change it during peak |
| Tax calculation formula | Stored proc in DB | Application code — testable, versionable |
| Stripe API key | Hardcoded string | Secrets manager / environment variable |
| Product descriptions | DB column updated by code only | CMS — content team can edit independently |
| Whether a feature is live | `if (true)` in code | Feature flag via config service — toggle without deploy |
| User's shipping address | In-memory only | Database — must persist across sessions |
| Retry logic | Config value `"retryLogic": "exponential"` | Application code — logic isn't config |
| Promo banner text | Hardcoded in React component | CMS — marketing owns this |

---

## Summary

- **Code** owns logic and behavior that must be tested and versioned.
- **Database** owns data that persists, is user-specific, or needs querying.
- **Config** owns operational knobs, secrets, environment-specific values, and feature toggles.

When in doubt: *if changing it requires a developer and a deploy, it's code. If it can safely change at runtime, it's config. If it needs to live beyond a process restart and vary per record, it's database.*

---

## Related Documents

- Feature Flags — feature flag lifecycle, naming conventions, rollout strategies, and cleanup best practices
- Multi-Tenancy — layered configuration patterns for tenant-specific overrides