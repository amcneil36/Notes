# Releasing Best Practices

A release is the most dangerous thing your code does. Everything before it — writing, reviewing, testing — is rehearsal. The release is the performance. And yet most teams treat it as an afterthought: merge to main, kick off the pipeline, hope for the best. Good release practices don't just prevent outages — they make shipping faster because you stop fearing the deploy button.

---

## Quick Decision Guide

| Strategy | Best For | Avoid When |
|---|---|---|
| **Rolling** | Stateless services, routine updates | Breaking API changes, DB migrations |
| **Blue-Green** | Zero-downtime cutover, easy rollback | Large infrastructure cost is a concern |
| **Canary** | High-traffic services, risk-sensitive changes | Changes that can't be meaningfully validated with partial traffic |
| **Feature Flags** | Decoupling deploy from release, gradual rollouts | Short-lived changes where flag overhead isn't worth it |
| **Dark Launch** | Performance validation of new paths | User-facing behavior changes |
| **Big Bang** | Coordinated multi-system cutover (last resort) | Literally everything else — avoid this if you can |

---

## Release Strategies

### Rolling Deployment

The default for most container orchestrators. Old instances are replaced with new ones incrementally. At any point during the deploy, both old and new versions are serving traffic.

**Use this when:**
- The change is backward-compatible
- You have health checks that catch startup failures
- The service is stateless or handles in-flight requests gracefully

**Don't use this when:**
- Old and new versions can't coexist (breaking schema changes, incompatible API contracts)
- You need instant rollback — rolling back means rolling forward through the old version again

**The rule:** If your change requires the old version to stop running before the new version starts, rolling deployments will hurt you.

### Blue-Green Deployment

Two identical environments: one live (blue), one staged (green). You deploy to green, validate, then switch traffic. If something breaks, switch back.

**Use this when:**
- You need instant rollback (flip the switch)
- You want to validate the full deployment before any user sees it
- You can afford running two environments simultaneously

**Don't use this when:**
- Your infrastructure budget is tight — you're paying for double capacity during the switch
- Your service has sticky sessions or long-lived connections that make the cutover messy
- Database state diverges between blue and green (this is the classic trap)

**The rule:** Blue-green is the safest strategy for stateless services, but the moment persistent state enters the picture, you need to think much harder.

### Canary Deployment

Route a small percentage of traffic (1-5%) to the new version. Monitor error rates, latency, and business metrics. If everything looks good, gradually increase traffic. If not, pull the canary.

**Use this when:**
- You're deploying to a high-traffic service where a bad release has outsized impact
- You have good observability — dashboards, alerts, SLOs — to compare canary vs. baseline
- You want production validation without full exposure

**Don't use this when:**
- Traffic volume is too low to produce statistically meaningful results at 1-5%
- The change is all-or-nothing (e.g., a database migration that affects all requests regardless of which version handles them)
- You don't have automated analysis or someone watching the canary — an unmonitored canary is just a slow rollout

**Canary success criteria (define these before deploying):**

| Metric | Threshold |
|---|---|
| Error rate (5xx) | No more than baseline + 0.1% |
| p99 latency | No more than baseline + 10% |
| Business metric (conversions, cart adds, etc.) | No statistically significant regression |

**The rule:** A canary without success criteria and someone (or something) watching it is just a rolling deployment with extra steps.

### Feature Flags

Deploy the code but keep the new behavior behind a flag. Enable the flag for internal users, then a percentage of production traffic, then everyone. The deploy and the release become two separate events. See Feature Flags for full details on flag lifecycle and management.

**Use this when:**
- You want to ship code continuously but release features on a business timeline
- You need to kill a feature instantly without a redeploy
- You're running A/B experiments
- Multiple teams contribute to the same service and need to release independently

**Don't use this when:**
- The change is infrastructure-only (config, dependency updates, performance tuning) — a flag adds complexity for no benefit
- You won't clean up the flag within 2-4 weeks — stale flags are tech debt with compound interest
- The flagged code path has deep tentacles across the system, making the "off" state hard to reason about

**The rule:** Feature flags decouple deploy from release. That's powerful. But every flag you add is a branch in your code that doubles the state space you need to test. Ship the flag, prove the feature, kill the flag.

### Dark Launch

Deploy the new code path alongside the old one. Both execute, but only the old path's result is returned to the user. The new path runs in shadow mode — you compare its output and performance against the old path.

**Use this when:**
- You're replacing a critical system (rewriting a pricing engine, migrating a data source)
- You need to validate correctness and performance under real production load
- The new path has no side effects (or side effects are safely isolated)

**Don't use this when:**
- The new path produces side effects (sends emails, charges cards, writes to shared state) — shadow mode doesn't mean invisible
- You can't afford the extra compute cost of running both paths

**The rule:** Dark launches answer the question "will this work in production?" without risking "what happens if it doesn't."

---

## Release Cadence and Branching

How often you release and how you manage branches are two sides of the same coin. The right cadence depends on your team's maturity, test coverage, and deployment confidence. Trunk-based development suits teams deploying continuously, GitFlow suits versioned products, and release trains suit organizations coordinating across multiple teams. See Version Control for branching strategy details.

### Choosing Your Cadence

| Signal | Points Toward | Points Away From |
|---|---|---|
| High test coverage, fast CI | Continuous deployment | Scheduled releases |
| Multiple teams sharing a codebase | Release trains | Continuous per-team deploys |
| External compliance requirements | Scheduled releases with sign-off | Continuous deployment |
| Microservice architecture, team owns the service | Continuous deployment | Release trains |
| Low confidence in test suite | Scheduled releases with manual QA | Continuous deployment |

**The rule:** Release cadence is a function of confidence. The more confident you are that a change is safe (tests, observability, rollback), the more frequently you can release. If you're afraid to deploy, the answer isn't "deploy less" — it's "invest in the things that make deploying safe."

---

## The Release Checklist

Every release, no matter how small, should pass through a mental (or actual) checklist. This isn't bureaucracy — it's the difference between "that deploy went fine" and "that deploy went fine because we checked."

### Before You Deploy

- **Is the change backward-compatible?** If old and new versions coexist during rollout, will they conflict? Think API contracts, database schemas, cache formats, message queue payloads.
- **Are feature flags in the right state?** If the new code is behind a flag, is the flag off by default? Have you tested the "flag off" path?
- **Is the rollback plan clear?** "Redeploy the old version" is an acceptable plan. "I don't know" is not.
- **Are alerts and dashboards ready?** If this release breaks something, will you know? Can you distinguish release-related issues from background noise?
- **Is anyone watching?** Don't deploy and walk away. Someone should be watching metrics for at least the first few minutes after traffic starts hitting the new version.

### After You Deploy

- **Verify in production.** Don't trust the pipeline green check alone. Hit the service. Check logs. Confirm the change is live and behaving as expected.
- **Monitor for the bake period.** How long depends on traffic patterns — a service with daily traffic spikes needs to bake through at least one spike.
- **Communicate.** If the release affects other teams, downstream services, or user-facing behavior, let people know it's live. A Slack message takes 10 seconds and prevents 10 confused pings.

---

## Rollback

Rollback doesn't need a dedicated playbook for every release, but you should always know the answer to: **"If this goes wrong, how do I undo it, and how long will that take?"**

**Three questions to answer before every release:**

1. **Can I roll back by redeploying the previous version?** If yes, this is the simplest path. Make sure the previous artifact is still available and deployable.
2. **Are there state changes that make rollback non-trivial?** Database migrations, cache format changes, message schema changes — these can't be undone by just redeploying old code.
3. **What's my blast radius?** If the release is bad, does it affect one endpoint or the entire service? One region or all regions? Knowing this shapes how urgently you respond.

**Keep rollback fast.** If rolling back takes 45 minutes, you'll hesitate to do it, and hesitation during an incident is expensive. Aim for rollback in under 5 minutes for most services.

---

## Database Changes and Releases

This is where releases get tricky. Code is stateless and replaceable — databases are not.

**The golden rule:** Make database changes backward-compatible with the currently running code. Deploy the schema change first, then deploy the code that uses it.

| Change Type | Safe Approach |
|---|---|
| Add a column | Add with a default value or as nullable. Deploy code that writes to it after. |
| Remove a column | Deploy code that stops reading/writing the column first. Drop the column later. |
| Rename a column | Don't. Add the new column, migrate data, update code, drop the old column. |
| Add an index | Add concurrently (if your DB supports it). Monitor lock contention. |
| Change a column type | Add a new column with the new type, dual-write, migrate, swap, drop old. |

**The rule:** Never make a database change that would break the currently running version of your code. The deploy and the migration are two steps, not one.

---

## Common Mistakes

1. **Deploying on Friday afternoon.** You know why. Your monitoring is weaker on weekends, your team is unavailable, and "quick fix" turns into "Monday morning war room." If you must deploy on Friday, make it early and small.

2. **Treating every release the same.** A config change and a database migration are not the same risk. Scale your process to the risk — don't skip the checklist for big changes, and don't make small changes jump through unnecessary hoops.

3. **No bake time.** Deploying and immediately moving on. Some bugs only surface under specific traffic patterns, time-of-day effects, or after caches expire. Let the release bake.

4. **"We'll roll back if something goes wrong" without ever testing rollback.** If you've never rolled back your service, you don't know if rollback works. Test it. Ideally in a non-prod environment, but test it.

5. **Coupling deploy and release without feature flags.** You merge a half-finished feature into main, deploy it, and now users see broken UI. Feature flags exist for this exact reason.

6. **Not communicating releases.** Other teams depend on your service. If you change behavior — even if the API contract is the same — tell downstream consumers. Silent releases create silent failures.

7. **Deploying multiple unrelated changes in one release.** When something breaks, you now have to figure out which of the five changes caused it. Ship small, ship often. Each release should be one logical change.

8. **Skipping staging because "it's a small change."** The size of the change has no correlation with the size of the blast radius. A one-line typo in a config can take down a service.

---

## Quick Reference Checklist

### Pre-Release

- [ ] Change is backward-compatible with the currently running version
- [ ] Database migrations (if any) are deployed separately and are backward-compatible
- [ ] Feature flags are in the correct default state
- [ ] Rollback plan is clear and tested (or at least understood)
- [ ] Alerts and dashboards are in place to detect regressions
- [ ] Downstream teams are notified if behavior changes
- [ ] Release is happening during business hours with the team available

### During Release

- [ ] Monitor error rates, latency, and business metrics during rollout
- [ ] Verify the change is live (don't just trust the pipeline)
- [ ] If using canary: confirm canary success criteria before promoting

### Post-Release

- [ ] Bake the release through at least one traffic cycle
- [ ] Clean up feature flags within 2-4 weeks
- [ ] Update runbooks if operational behavior changed
- [ ] Close the ticket / update the changelog
