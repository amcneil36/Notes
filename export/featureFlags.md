# Feature Flag Best Practices

Feature flags are one of the most powerful tools in modern software delivery. They decouple deployment from release, let you ship incomplete work safely, enable instant kill switches, and make gradual rollouts trivial. They're also one of the most common sources of tech debt, production incidents, and untestable code — because most teams adopt them enthusiastically and manage them terribly.

The goal: **use flags to ship faster and safer, then remove them before they become the problem.**

---

## What Feature Flags Actually Are

A feature flag is a conditional branch in your code that's controlled by external configuration rather than a code change. That's it. Everything else — gradual rollouts, A/B testing, kill switches, entitlements — is a use case built on top of that primitive.

```
if (featureFlags.isEnabled("new-checkout-flow", context)) {
    return newCheckoutFlow(cart);
} else {
    return legacyCheckoutFlow(cart);
}
```

The key insight is that this is **a branch in your code's control flow**, and branches have costs: they multiply the state space you need to test, reason about, and maintain. A flag that lives for two weeks is a release mechanism. A flag that lives for six months is a liability.

---

## Flag Types — Not All Flags Are Equal

Different flag types have different lifetimes, ownership models, and cleanup urgency. Treating them all the same is how you end up with 400 flags and no idea which ones matter.

| Type | Purpose | Expected Lifetime | Owner | Cleanup Urgency |
|---|---|---|---|---|
| **Release flag** | Gate incomplete or risky code during rollout | Days to weeks | Dev team | High — remove after full rollout |
| **Experiment flag** | A/B test a hypothesis with measurable outcomes | Weeks to months | Product/data team | Medium — remove after experiment concludes |
| **Ops flag** | Kill switch or circuit breaker for operational control | Permanent (but rarely toggled) | On-call / SRE | Low — these are infrastructure |
| **Permission flag** | Gate features by user segment, tier, or entitlement | Long-lived | Product team | Low — these are business logic |
| **Migration flag** | Control traffic between old and new implementations during a system migration | Weeks to months | Dev team | High — remove after migration completes |

**The rule:** If you don't know what type a flag is when you create it, you won't know when to delete it. Every flag should be tagged with its type and an expected expiration date at creation time.

---

## Flag Architecture — Where Flags Live

### Naming Conventions

Flag names are the only thing between you and chaos. You'll have dozens, then hundreds. Without naming discipline, you end up grepping Slack threads to figure out what `enable_new_thing_v2_FINAL` controls.

**Recommended format:**
```
<type>.<domain>.<feature>
```

**Examples:**
```
release.checkout.one-click-purchase
experiment.search.ai-suggestions
ops.payments.stripe-circuit-breaker
permission.seller.bulk-upload
migration.catalog.new-pricing-engine
```

| Convention | Why |
|---|---|
| Prefix with flag type | You can filter dashboards, set cleanup policies per type, and instantly know a flag's purpose |
| Include the domain/team | Ownership is clear — no "who owns this flag?" Slack messages |
| Use kebab-case for the feature | Readable, grep-friendly, no ambiguity |
| Never include version numbers | `v2`, `v3`, `final` — these are symptoms of flags that should have been cleaned up |

### Where to Evaluate Flags

Flags can be evaluated server-side, client-side, or both. Where you evaluate determines your latency, consistency, and security characteristics.

| Evaluation Location | Pros | Cons | Best For |
|---|---|---|---|
| **Server-side** | Secure (users can't see flag state), consistent, no exposure of flag logic | Requires a network call (or SDK with caching) | Backend behavior, API changes, pricing logic, anything security-sensitive |
| **Client-side** | Instant evaluation, no server round-trip, enables UI-only experiments | Flag state is visible to users, can be tampered with, requires SDK in the client | UI changes, layout experiments, cosmetic toggles |
| **Edge/CDN** | Fastest possible evaluation, no origin hit | Limited context (no user-specific data without additional lookups), harder to debug | Regional rollouts, traffic shaping, A/B by geography |

**The rule:** If the flag controls anything that affects money, access, or security, evaluate it server-side. Client-side flags are for UI experiments and cosmetic changes. If you're putting a client-side flag around a pricing discount, stop — an enterprising user with browser dev tools will find it.

### Flag Dependencies

Flags should be independent. The moment one flag's behavior depends on another flag's state, you've created a combinatorial explosion that's nearly impossible to test.

```
// Terrible — 4 possible states, only 2 of which are valid
if (flags.isEnabled("new-search") && flags.isEnabled("new-search-ranking")) {
    // new search with new ranking
} else if (flags.isEnabled("new-search") && !flags.isEnabled("new-search-ranking")) {
    // new search with old ranking
} else if (!flags.isEnabled("new-search") && flags.isEnabled("new-search-ranking")) {
    // old search with new ranking — does this even make sense?
}
```

```
// Better — one flag, one behavior
if (flags.isEnabled("new-search-v2")) {
    // new search with new ranking (they're one feature)
}
```

If two flags are always toggled together, they should be one flag. If one flag only makes sense when another is on, they should be one flag. Fight the urge to create granular flags "just in case" — you're creating a testing matrix that grows exponentially.

---

## The Flag Lifecycle — Birth to Death

Every flag should follow a predictable lifecycle. The moment you skip a phase, you're accumulating debt.

### 1. Creation

Before writing code, define:

- **Name** (following your naming convention)
- **Type** (release, experiment, ops, permission, migration)
- **Owner** (team or individual responsible for cleanup)
- **Default state** (off — always off, unless it's an ops kill switch that should default to on)
- **Expiration date** (when should this flag be removed?)
- **Rollout plan** (who gets it first, what's the ramp schedule?)

Register the flag in your flag management system before writing the code that uses it. This creates the audit trail and makes the flag visible to the whole team.

### 2. Development

Write the flagged code path alongside the existing path. Both paths must work. The "flag off" path is your production code today — don't break it while adding the new path.

```
// The flag-off path should always be the safe, known-good path
function getPrice(item, context) {
    if (flags.isEnabled("migration.pricing.new-engine", context)) {
        return newPricingEngine.calculate(item);
    }
    return legacyPricingEngine.calculate(item);
}
```

### 3. Rollout

Gradual. Always gradual for release and migration flags.

**Recommended ramp schedule:**

| Phase | Audience | Duration | Purpose |
|---|---|---|---|
| Internal | Your team only | 1-2 days | Smoke test in production |
| Dogfood | All internal employees | 2-3 days | Broader validation, catch edge cases |
| Canary | 1-5% of production traffic | 1-3 days | Statistical validation of error rates and latency |
| Ramp | 10% → 25% → 50% → 100% | 1-2 days per step | Gradual exposure with monitoring at each step |
| Full | 100% of traffic | 1-2 weeks bake | Confirm stability before cleanup |

At each phase, monitor:
- Error rates (compare flagged vs. unflagged cohorts)
- Latency (p50, p95, p99)
- Business metrics (conversion, revenue, cart size — whatever matters for this feature)
- Resource consumption (CPU, memory, DB queries)

If any metric degrades, stop the rollout and investigate. Don't just push through because the feature "mostly works."

### 4. Cleanup (The Phase Everyone Skips)

Once the flag is at 100% and baked for 1-2 weeks with no issues:

1. **Remove the flag evaluation** from code — keep only the new path
2. **Remove the flag definition** from your flag management system
3. **Delete the old code path** — don't comment it out, delete it
4. **Remove any tests** that specifically test the "flag off" behavior (the old path is gone)
5. **Commit with a message** that references the flag name for traceability

**How to enforce cleanup:**

- Set expiration dates on flags and alert when they pass
- Run a weekly report of flags past their expiration date, visible to the whole team
- Include flag cleanup in sprint planning — it's real work, not busywork
- Some teams block creating new flags if the team has more than N expired flags

---

## Testing Flagged Code

Feature flags make testing harder because every flag doubles the number of states your system can be in. With 5 independent flags, you have 32 possible configurations. With 10, you have 1,024. You can't test all of them. Don't try.

### What to Test

| Test Type | Flag State | Why |
|---|---|---|
| **Unit tests for the new path** | Flag ON | Validate the new behavior works correctly |
| **Unit tests for the old path** | Flag OFF | Confirm you didn't break the existing behavior |
| **Integration tests** | Flag ON and OFF separately | Verify both paths work end-to-end |
| **The transition** | Flag toggled mid-request (if applicable) | Ensure no inconsistency when flag state changes |

### What NOT to Test

- Every combination of every flag. You'll never finish, and most combinations are meaningless.
- Flag interactions unless flags are explicitly designed to interact (and they shouldn't be — see "Flag Dependencies" above).

### Testing Strategy

```
// Test the new path
test("new checkout flow calculates tax correctly", () => {
    flags.override("release.checkout.one-click-purchase", true);
    result = checkout(cart);
    assert(result.tax == expectedTax);
});

// Test the old path still works
test("legacy checkout flow is unaffected", () => {
    flags.override("release.checkout.one-click-purchase", false);
    result = checkout(cart);
    assert(result.tax == expectedTax);
});

// Test the flag evaluation itself
test("flag respects targeting rules", () => {
    flags.override("release.checkout.one-click-purchase", true, { segment: "beta-users" });
    assert(flags.isEnabled("release.checkout.one-click-purchase", betaUser) == true);
    assert(flags.isEnabled("release.checkout.one-click-purchase", regularUser) == false);
});
```

**Staging vs. production testing:** Test flag-off in staging (it's your safety net). Test flag-on in production with your gradual rollout. Don't rely solely on staging for the new path — staging doesn't have real traffic, real data, or real scale.

---

## Operational Risk — When Flags Become Weapons

Feature flags are runtime configuration. That makes them as powerful — and as dangerous — as any production config change. A misconfigured flag can take down a service faster than a bad deploy, because there's no build, no pipeline, and no CI to catch it.

### Kill Switches

Every service should have ops flags that act as kill switches for expensive or risky operations:

```
ops.search.disable-ai-ranking          // Fall back to deterministic ranking
ops.payments.disable-stripe            // Route payments to backup processor
ops.catalog.disable-external-enrichment // Skip third-party API calls
```

Kill switches should:
- **Default to ON** (the feature is enabled) — you toggle them OFF to disable
- **Be evaluable without network calls** — if your flag service is down, the kill switch should fail to the safe state
- **Be documented in your runbook** — on-call engineers need to know what each switch does without reading code
- **Have no targeting rules** — kill switches are all-or-nothing, not "disable for 10% of users"

### The Flag Service Is a Single Point of Failure

If your application evaluates flags via a remote service on every request, that service is on the critical path. When it goes down, every flag evaluation fails.

**Mitigations:**

| Strategy | How It Works |
|---|---|
| **Local cache with TTL** | SDK caches flag state locally, refreshes periodically. If the service is down, stale cache is used. |
| **Default values** | Every flag evaluation includes a default value that's used when evaluation fails. The default should always be the safe/existing behavior. |
| **Bootstrap file** | Ship a static JSON file with flag defaults as part of your deployment artifact. Loaded on startup, used as fallback. |

```
// Always provide a sensible default
enabled = flags.isEnabled("release.checkout.one-click-purchase", context, 
    default=false  // if evaluation fails, use the old checkout flow
);
```

**The rule:** A flag service outage should never change your application's behavior. The default/cached state should be indistinguishable from "flags are working normally with their last known values."

### Flag-Caused Incidents

The most common flag-related incidents:

1. **Wrong targeting** — Flag enabled for all users instead of just internal. Solution: require approval for targeting rule changes that affect >10% of traffic.
2. **Flag dependency conflict** — Flag A assumes Flag B is on, but someone toggled B off. Solution: don't create flag dependencies.
3. **Stale flag pointing at dead code** — The code behind the flag was refactored, but the flag still evaluates and routes to a broken path. Solution: flag cleanup enforcement.
4. **Flag evaluation performance** — Evaluating 50 flags per request, each with complex targeting rules, adds measurable latency. Solution: batch flag evaluation, use local cache, limit the number of active flags per service.

---

## Common Mistakes — What NOT to Do

### 1. Flags Without Expiration Dates

Every release and migration flag should have an expiration date set at creation. "We'll clean it up later" is a lie your future self will resent. Set the date. Alert on it. Hold teams accountable.

### 2. Using Flags for Permanent Business Logic

```
// Bad — this isn't a flag, it's a config value masquerading as a flag
if (flags.isEnabled("enable-premium-features", user)) {
    showPremiumUI();
}
```

If the "flag" is never intended to be removed — it's a permanent entitlement or tier check — don't put it in your feature flag system. Put it in your authorization layer, your user service, or your config. Feature flag systems aren't designed for permanent business rules, and treating them that way bloats your flag count and confuses the purpose of the system.

The exception: **ops flags** (kill switches) are intentionally permanent. But they should be clearly tagged as ops flags, not mixed in with release flags.

### 3. Flag Spaghetti — Too Many Flags in One Code Path

```
// Nightmarish — 3 flags, 8 possible states, nobody can reason about this
if (flags.isEnabled("new-cart") && flags.isEnabled("new-pricing") && !flags.isEnabled("legacy-discounts")) {
    applyNewPricingWithoutLegacyDiscounts(cart);
} else if (flags.isEnabled("new-cart") && flags.isEnabled("new-pricing")) {
    applyNewPricingWithLegacyDiscounts(cart);
} else if (flags.isEnabled("new-cart")) {
    applyOldPricingToNewCart(cart);
} else {
    applyOldPricingToOldCart(cart);
}
```

If you need multiple flags to describe a single behavior, you need one flag. If you need nested flag checks, you have a design problem. Refactor the feature so it's controlled by a single flag, or break it into independent features with independent flags.

### 4. Putting Flags in the Wrong Layer

| Scenario | Wrong Layer | Right Layer | Why |
|---|---|---|---|
| New pricing algorithm | Frontend flag hiding the price display | Backend flag controlling the calculation | A frontend flag doesn't prevent the new price from appearing in APIs, emails, or other channels |
| New UI layout | Backend flag controlling which template to render | Frontend flag controlling component rendering | The backend doesn't need to know about your CSS changes |
| Kill switch for a third-party API | Frontend flag | Backend flag | The backend is the one making the API call — the frontend can't prevent it |
| A/B test on button color | Backend flag | Frontend flag | This is purely visual — no backend logic involved |

**The rule:** The flag should live in the layer that owns the behavior being changed. If the flag is in a different layer, you're either not protecting what you think you're protecting, or you're adding unnecessary coupling.

### 5. No Monitoring on Flag Changes

Toggling a flag in production IS a production change. Treat it like one:

- Log every flag toggle with who, what, when, and why
- Alert when a flag affecting >10% of traffic is toggled
- Correlate flag changes with your monitoring dashboards — if error rates spike 30 seconds after a flag toggle, you know the cause
- Require a brief description for every flag toggle (not just "turning on the new thing")

### 6. Skipping the Gradual Rollout

"It works in staging, let's just turn it on for everyone."

No. Staging doesn't have your production traffic patterns, your production data volume, or your production edge cases. The gradual rollout exists because you can't predict what production will do to your code. Even a 1% canary for 24 hours catches issues that staging never will.

### 7. Leaving Both Code Paths After Full Rollout

```
// The flag has been at 100% for three months. Both paths still exist.
if (flags.isEnabled("new-search")) {
    return newSearch(query);  // always takes this path
} else {
    return oldSearch(query);  // dead code
}
```

Dead code is confusing, untested, and rotting. Once a flag is fully rolled out and baked, delete the flag check and the old code path. If you're afraid to remove it because "we might need to roll back," that's a sign you haven't baked long enough — extend the bake period, don't keep the dead code.

### 8. Flag Sprawl — No Limits on Active Flags

Set a limit on how many active flags a service can have. A reasonable starting point:

| Service Complexity | Max Active Flags |
|---|---|
| Simple service (single responsibility) | 5-10 |
| Medium service | 10-20 |
| Large monolith | 20-30 |

When you hit the limit, you can't create new flags until you clean up old ones. This creates natural pressure to manage flag lifecycle.

---

## Flag Evaluation Performance

Flag evaluation cost is easy to ignore when you have 5 flags. It becomes a real problem at 50.

### Optimization Strategies

| Strategy | How | When to Use |
|---|---|---|
| **Batch evaluation** | Evaluate all flags for a request in a single call, not one-by-one | Always — avoid N+1 flag evaluation calls |
| **Local cache** | Cache flag rules in-process, refresh on an interval or via streaming updates | Any production service |
| **Precompute targeting** | Resolve user segments at login/session start, not on every request | Complex targeting rules with many segments |
| **Limit active flags** | Fewer flags = fewer evaluations = lower overhead | Always |

### What to Monitor

- **Flag evaluation latency** — p50 and p99, per request
- **Cache hit rate** — if your local cache is missing frequently, your TTL is too short or the flag set is changing too often
- **Flag count per service** — trending upward means cleanup isn't happening
- **Stale flag count** — flags past their expiration date, still active

---

## Quick Reference Checklist

### Flag Creation
- [ ] Flag has a clear name following the naming convention (`<type>.<domain>.<feature>`)
- [ ] Flag type is defined (release, experiment, ops, permission, migration)
- [ ] Owner is assigned (team or individual)
- [ ] Expiration date is set (for release and migration flags)
- [ ] Default value is safe (flag off = existing behavior, except ops kill switches)
- [ ] Rollout plan is documented (ramp schedule, success criteria)

### Flag Development
- [ ] Flag is evaluated in the correct layer (backend for logic, frontend for UI)
- [ ] Both flag-on and flag-off paths work correctly
- [ ] No dependencies on other flags
- [ ] Default/fallback value is provided for evaluation failures
- [ ] Flag evaluation is batched, not one-by-one per request

### Flag Rollout
- [ ] Gradual rollout follows the ramp schedule (internal → dogfood → canary → ramp → full)
- [ ] Monitoring is in place for error rates, latency, and business metrics
- [ ] Flag toggles are logged with who, what, when, and why
- [ ] Rollback plan is clear (toggle flag off)

### Flag Testing
- [ ] Unit tests cover both flag-on and flag-off paths
- [ ] Integration tests run for both states
- [ ] No tests depend on multiple flags being in specific states simultaneously
- [ ] Flag targeting rules are tested (correct users get correct state)

### Flag Cleanup
- [ ] Flag has been at 100% for at least 1-2 weeks with no issues
- [ ] Flag evaluation is removed from code
- [ ] Old code path is deleted (not commented out)
- [ ] Flag definition is removed from the flag management system
- [ ] Tests for the old path are removed or updated
- [ ] Commit references the flag name for traceability

### Operational Readiness
- [ ] Kill switches exist for critical dependencies and expensive operations
- [ ] Kill switches are documented in the service runbook
- [ ] Flag service failure falls back to cached/default values gracefully
- [ ] Flag change alerts are configured for high-impact toggles
- [ ] Active flag count is within the service limit

---

## Related Documents

- Consistent vs Inconsistent Pass Rate — assignment strategy details for gradual rollouts (consistent/sticky vs. random)
- A/B Testing — experiment design, statistical rigor, and when to use experiment flags
- Application vs DB vs Config — where feature flags fit in the broader configuration landscape
- Releasing — how feature flags integrate into the release process
