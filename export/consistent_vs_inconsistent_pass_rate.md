# Consistent vs Inconsistent Pass Rate

When gradually rolling out a binary deployment or enabling a feature flag for a subset of users (i.e., canarying), you need a strategy for deciding *who* gets the feature. Two fundamental approaches exist: **consistent** and **inconsistent** pass rates.

---

## Consistent Pass Rate (Sticky Assignment)

A fixed subset of users always receives the feature. For example, with a 10% rollout, the same 10% of users get the feature on every request — the remaining 90% never see it.

This is typically implemented by hashing a stable identifier (e.g., user ID or device ID) and checking whether it falls within the target bucket. The result is deterministic and repeatable.

### Tradeoffs

**Advantages:**
- **Reproducible behavior** — a user's experience is stable across sessions. If they see the feature today, they'll see it tomorrow.
- **Easier debugging** — when a bug is reported, you know whether the reporting user was in the canary group. You can reproduce their exact environment.
- **Cleaner metrics** — user-level analysis is straightforward. Each user is either in the treatment group or the control group, so you get clean A/B cohorts with no bleed-over.
- **Safer rollbacks** — if you reduce the rollout percentage, the same users lose access; you're not randomly re-assigning the population.
- **User experience coherence** — users don't experience the feature flickering in and out, which can cause confusion and noise in support tickets.

**Disadvantages:**
- **Concentrated blast radius** — if the new binary has a bug, the same set of users is repeatedly affected. They bear the full brunt of the issue for the duration of the canary.
- **Potential sampling bias** — if your hash bucketing isn't well-distributed, or if the "always-on" cohort happens to share a demographic or usage pattern, your canary metrics may not generalize to the full population.

---

## Inconsistent Pass Rate (Non-Sticky / RNG-Based Assignment)

Each request independently rolls the dice. A user might get the feature on one request and not on the next. With a 10% pass rate, any given user has a 10% chance on each individual request.

### Tradeoffs

**Advantages:**
- **Distributed blast radius** — no single user is consistently on the receiving end of a bug. The pain is spread more evenly across the population over time.
- **More representative exposure** — over many requests, a broader cross-section of the user base touches the new code, which can surface edge cases faster.
- **Simple implementation** — just a random number check, no user ID hashing or state needed.

**Disadvantages:**
- **Terrible user experience** — users see the feature intermittently. A new UI, a changed API response, or a different behavior appearing and disappearing is confusing and often generates noise (support tickets, bug reports that can't be reproduced).
- **Dirty metrics** — users are in both treatment and control at different times. You can't cleanly attribute outcomes to the feature since the same user contributes to both groups. This makes any A/B analysis unreliable.
- **Debugging is a nightmare** — a user reports an issue, you try to reproduce it, and you can't because their next request hit the control path. You lose the ability to confidently say "this user was on the new binary."
- **Stateful operations break** — if the feature involves any state (session data, caches, database writes, in-flight transactions), inconsistent assignment can corrupt state or cause failures when a user flips between code paths mid-flow.
- **Hard to rollback meaningfully** — reducing the pass rate doesn't cleanly remove any group from exposure; it just lowers the probability for everyone, which makes it harder to reason about who has seen what.

---

## Summary

| | Consistent | Inconsistent |
|---|---|---|
| User experience | Stable | Unpredictable |
| Debugging | Easy | Hard |
| Metrics / A/B analysis | Clean | Dirty |
| Blast radius | Concentrated | Distributed |
| Stateful safety | Safe | Risky |
| Implementation complexity | Moderate (hashing) | Simple (RNG) |

## Recommendation for Canarying

**~95% of the time: use consistent.** The debugging and metrics advantages far outweigh the concentrated blast radius concern, particularly since the whole point of a low-percentage canary is to limit exposure while you validate.

**~5% of the time: inconsistent** is acceptable — only for purely infrastructure-level changes that are completely invisible to users. Even then, the benefit is marginal.

### When to use consistent (nearly always)

- **UI changes** — new checkout flow, redesigned nav, feature toggle that changes what a user sees. A user who sees the feature one visit and not the next will file a bug report or think they imagined it.
- **A/B experiments** — any change you intend to measure. User-level attribution is required for valid metrics.
- **API response format changes** — if a client caches or builds state off the response, flipping between formats mid-session corrupts things.
- **Authentication, session, or personalization logic** — these are inherently stateful. Inconsistent assignment here causes hard-to-trace failures.
- **Pricing, promotions, or entitlements** — a user seeing a price one way then another is a trust and compliance problem, not just a UX one.
- **Anything you will need to debug** — which is essentially everything.

### When inconsistent is acceptable (rarely)

Inconsistent only makes sense when the change is **completely imperceptible to the user** — i.e., the output or behavior looks identical regardless of which path they hit:

- Switching between two equivalent data sources or read replicas (same data, different infrastructure).
- Swapping an internal ranking or scoring algorithm where the results are close enough that users won't notice the difference between requests.
- Infrastructure-level routing changes (e.g., testing a new load balancer path or cache layer) where the response is functionally identical.
- Logging, tracing, or observability instrumentation with no effect on the response.

### Is inconsistent ever actually better UX?

No. Inconsistent is **at best equal UX** — and only in the narrow cases above where the change is truly invisible. The moment a user can perceive any difference between the two paths, inconsistent is strictly worse: they get a flickering, non-reproducible experience that erodes trust and generates noise.

The "distributed blast radius" advantage of inconsistent is about operational risk spreading, not user experience. Don't conflate the two.

---

## Related Documents

- Feature Flags — flag lifecycle, naming conventions, and rollout best practices
- A/B Testing — experiment design and statistical methodology (consistent assignment is critical for valid A/B results)
