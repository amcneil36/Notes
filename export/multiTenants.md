# Multi-Tenancy Best Practices

## Context

You have a software application where ~90% of behavior is identical across all tenants, but ~10% needs to vary. In this discussion, tenants are **countries** (USA, England, Canada, Mexico), though these principles generalize to any tenant dimension (business unit, brand, customer org, etc.).

---

## 1. Fundamental Architecture Decision: Isolation vs. Sharing

There are three broad models. Pick one per layer (you can mix):

| Model | Description | When to Use |
|---|---|---|
| **Silo (fully isolated)** | Separate infrastructure per tenant — dedicated servers, databases, pipelines | Strict regulatory requirements (data residency), vastly different SLAs, or when tenants pay for dedicated capacity |
| **Pool (fully shared)** | Single deployment serves all tenants, distinguished by a `tenant_id` column or request header | Behavior is mostly identical, cost efficiency matters, and you want a single deployment pipeline |
| **Bridge (hybrid)** | Shared compute, isolated databases — or shared database with isolated schemas | Need data isolation without the operational cost of full silos |

**For the 90/10 scenario, Pool or Bridge is almost always the right call.** Full silo only makes sense when regulatory or contractual obligations demand it.

---

## 2. Application Server Layer

### 2.1 Tenant Resolution

Every request must be tagged with a tenant identity as early as possible. Common patterns:

- **HTTP header** — e.g., `X-Tenant-Id: US` injected by the API gateway or load balancer based on domain/path/geo
- **Subdomain** — `us.myapp.com`, `mx.myapp.com`
- **JWT claim** — tenant embedded in the auth token
- **Path prefix** — `/us/api/orders`, `/mx/api/orders`

Best practice: resolve tenant **once at the edge** (gateway/ingress) and propagate it through the call chain via request context, thread-local, or a context object. Don't re-resolve mid-request.

### 2.2 Tenant Context Propagation

```
Request → Gateway (resolve tenant) → Context/ThreadLocal → Service Layer → Repository Layer
```

- Use a `TenantContext` abstraction that every layer can read from without coupling to HTTP
- For async flows (message queues, scheduled jobs), serialize tenant ID into the message/event payload so downstream consumers can restore context
- In Java/Spring, a `HandlerInterceptor` or `Filter` sets the tenant context; in Node, middleware does the same

### 2.3 Deploying Shared vs. Separate Instances

Since 90% is shared, **deploy a single application version** for all tenants. Don't fork the codebase per tenant — that's a maintenance nightmare.

- Use **feature flags** or **configuration** to toggle the 10% that differs (see Section 4)
- If a tenant has extreme traffic or isolation needs, you can run a **dedicated instance of the same artifact** with tenant-specific config — same code, different runtime config

---

## 3. Database Layer

### 3.1 Strategies

| Strategy | Pros | Cons |
|---|---|---|
| **Shared database, shared schema** (`tenant_id` column on every table) | Simple ops, easy cross-tenant queries/reporting | Risk of data leakage if queries miss the `tenant_id` filter; noisy neighbor risk |
| **Shared database, separate schemas** (one schema per tenant) | Better isolation, easier per-tenant backup/restore | Schema migration must run N times; connection pooling more complex |
| **Separate databases** | Strongest isolation, easy data residency compliance | Highest ops cost; cross-tenant analytics requires ETL |

### 3.2 Recommendations for the 90/10 Case

- **Default to shared database + `tenant_id` column.** It's the simplest and works well when the data model is 90% identical.
- **Enforce tenant isolation at the repository/ORM layer**, not by trusting every developer to remember `WHERE tenant_id = ?`. Use:
  - Row-level security (Postgres RLS)
  - Hibernate filters / JPA `@Filter`
  - A base repository class that automatically appends the tenant predicate
- **Index the `tenant_id` column** — it appears in nearly every query
- If a specific country has **data residency laws** (e.g., Mexico's LFPDPPP, EU GDPR for England), consider a separate database in the required region for that tenant only — bridge model

### 3.3 Migrations

- Schema changes must be tenant-aware. A migration that adds a column needed only by Mexico should still be safe to apply globally (nullable column with a default, or ignored by other tenants via config)
- Never branch your schema per tenant unless absolutely forced — divergent schemas are the #1 cause of multi-tenant tech debt

---

## 4. Configuration & Feature Toggles

This is how you handle the 10% that differs. This is arguably the most important section.

### 4.1 Layered Configuration Model

```
Base Config (all tenants)
  └── Tenant Override (per-country)
       └── Environment Override (per-env, per-tenant)
```

Example:
```yaml
# base config
order.maxItems: 100
shipping.provider: "default-carrier"
tax.enabled: true

# tenant: MX override
order.maxItems: 50
shipping.provider: "mx-carrier"
tax.vatRate: 0.16
```

The application reads base config first, then merges tenant-specific overrides on top. Unset keys fall through to the base.

### 4.2 Where to Store Tenant Config

The general principles for choosing between application code, database, and externalized configuration apply here -- the key addition for multi-tenancy is that each tool must support tenant-scoped overrides on top of base values. See Application vs DB vs Config for the general configuration placement framework.

| Tool | Good For (Tenant-Specific) |
|---|---|
| **Centralized configuration service / Spring Cloud Config / Consul KV** | Runtime-mutable tenant overrides; ideal for operational knobs (rate limits, feature flags, provider URLs) |
| **Environment variables** | Immutable per-deployment config (DB connection strings, secrets) |
| **Feature flag service (LaunchDarkly, Flagr, etc.)** | Conditional behavior toggling with targeting rules (by tenant, by percentage, by user segment) |
| **Database config table** | Tenant-specific business rules that change frequently and are managed by non-engineers |

### 4.3 Feature Flag Patterns for Tenant-Specific Behavior

Feature flags are the preferred mechanism for toggling tenant-specific behavior without hardcoded tenant checks. See Feature Flags for flag lifecycle and management. The key multi-tenant pattern: rather than `if (tenant == "MX") { ... }` scattered through code:

```java
// Good: externalized, configurable
if (featureFlags.isEnabled("customShippingLabel", tenantContext)) {
    // Mexico-specific shipping label logic
}

// Bad: hardcoded tenant check
if (tenant.equals("MX")) {
    // Mexico-specific shipping label logic
}
```

Why this matters:
- When Canada later needs the same behavior, you flip a flag instead of changing code
- You can test tenant-specific behavior in any environment by toggling flags
- You avoid a codebase full of country-specific `if` statements that nobody can reason about

### 4.4 The Strategy Pattern for Variant Behavior

When the 10% difference is a **whole algorithm** (tax calculation, address validation, shipping rules), use a strategy pattern keyed by tenant:

```java
public interface TaxCalculator {
    TaxResult calculate(Order order);
}

@Component("US")
public class UsTaxCalculator implements TaxCalculator { ... }

@Component("MX")
public class MxTaxCalculator implements TaxCalculator { ... }

// At runtime:
TaxCalculator calc = taxCalculatorRegistry.get(tenantContext.getTenantId());
```

This keeps tenant-specific logic **isolated, testable, and discoverable** instead of buried in conditionals.

---

## 5. Message Queues & Async Processing

- **Always include `tenant_id` in every message payload.** Consumers must restore tenant context before processing.
- **Shared topics vs. per-tenant topics:**
  - Default to shared topics with `tenant_id` as a message attribute. Simpler to manage.
  - Use per-tenant topics only when throughput isolation is critical (one tenant's volume shouldn't backpressure another)
- **Dead letter queues** should preserve tenant context so failed messages can be reprocessed correctly

---

## 6. Caching

- **Namespace your cache keys by tenant:** `cache:US:product:12345` not `cache:product:12345`
- Without tenant-scoped keys, you **will** serve USA data to Mexico users. This is a when-not-if bug.
- Consider per-tenant cache eviction policies if data freshness requirements differ

---

## 7. Observability & Monitoring

- **Tag every log line, metric, and trace span with `tenant_id`.** Non-negotiable.
- This lets you:
  - Alert on tenant-specific error spikes
  - Debug issues scoped to a single country
  - Track per-tenant SLA compliance
  - Identify noisy neighbor problems
- Dashboards should support filtering by tenant

---

## 8. Testing

- **Unit tests** for shared logic don't need tenant awareness
- **Unit tests** for tenant-specific strategies should parameterize across tenants
- **Integration/E2E tests** should run the critical paths for **each tenant** — don't assume USA tests cover Mexico
- Use a test matrix:

```
[US, GB, CA, MX] × [order flow, returns, tax calculation, shipping]
```

- Tenant-specific config in test environments should mirror production structure

---

## 9. Common Pitfalls

| Pitfall | Why It Hurts | Fix |
|---|---|---|
| Hardcoded `if (tenant == "XX")` everywhere | Unscalable, untestable, invisible | Feature flags + strategy pattern |
| Forgetting `tenant_id` in a query | Data leakage across countries | ORM-level enforcement, RLS |
| Per-tenant code branches or forks | Merge hell, divergent behavior | Single codebase, config-driven |
| Tenant config in code (constants) | Requires deploy to change | Externalized configuration service |
| No tenant in async messages | Jobs process data in wrong tenant context | Mandate tenant in message schema |
| Shared cache keys without tenant prefix | Cross-tenant data contamination | Enforce `{tenant}:{key}` pattern |

---

## 10. Decision Checklist for Adding a New Tenant

When onboarding a new country (e.g., adding Brazil):

1. **Config** — Add tenant-specific config overrides (tax rates, currency, locale, provider URLs)
2. **Feature flags** — Enable/disable features for the new tenant
3. **Strategies** — Implement any new strategy variants (tax calc, address validation, etc.)
4. **Database** — If using separate schemas/databases, provision and migrate
5. **Cache** — Verify cache key namespacing handles the new tenant
6. **Messages** — Confirm async consumers handle the new tenant ID gracefully
7. **Observability** — Add tenant to dashboards and alerting
8. **Testing** — Add new tenant to test matrix
9. **Documentation** — Document tenant-specific behaviors and their config keys

The goal: onboarding a new tenant should be a **configuration exercise**, not a code change.

---

## Related Documents

- Application vs DB vs Config — the general configuration placement framework (code vs. database vs. externalized config)
- Feature Flags — flag lifecycle, naming conventions, and rollout best practices for feature toggles used in tenant-specific behavior
