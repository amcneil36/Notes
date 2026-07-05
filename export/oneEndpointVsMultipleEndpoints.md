# One Endpoint vs Multiple Endpoints

When designing REST APIs that share similar logic but differ by action (e.g., `stop`, `resume`, `restart`), a common question is whether to consolidate into a single generic endpoint (`/operations/{action}/{planId}`) or split into separate dedicated endpoints (`/operations/stop/{planId}`, `/operations/resume/{planId}`, `/operations/restart/{planId}`).

The primary trade-off is **code simplicity vs observability granularity**.

---

## Engineering Effort Comparison

| Artifact | Single endpoint | Split endpoints (per action) |
|---|---|---|
| **Backend interface methods** | 1 | 1 per action |
| **Backend impl methods** | 1 (with routing `if/else`) | 1 per action (no routing logic) |
| **Backend test methods** | Fewer, but each covers more branches | More, but each is simpler and focused |
| **API gateway proxy config files** | 1 (wildcard pattern covers all actions) | 1 per action |
| **UI call sites** | 1 fetch call, action passed as path variable | 1 fetch call per action button |
| **Swagger/OpenAPI entries** | 1 | 1 per action |
| **New action added in future** | Add `else if` branch only — no gateway config change needed | New interface method + impl method + gateway config file + tests |

### Example — Stop/Resume/Restart Sequential Operations

| Artifact | Single endpoint | Split endpoints |
|---|---|---|
| Backend interface methods | 1 (`sequentialAction`) | 3 (`stopSequentialOp`, `resumeSequentialOp`, `restartSequentialOp`) |
| Backend impl methods | 1 | 3 |
| API gateway proxy config files | 1 (wildcard — no changes needed for new actions) | 3 |
| UI changes | Action passed as URL segment | Separate fetch call per button |

The single endpoint approach is meaningfully less code to write and review when the actions are first introduced. The split approach becomes less effort over time because each piece is smaller and independently testable.

---

## Observability Comparison

| Observability Signal | Single endpoint | Split endpoints |
|---|---|---|
| **Call count per action** | Via log search query (no code changes needed) | ✅ Out of the box |
| **Latency per action** | Via log search — requires adding explicit duration logging | ✅ Out of the box |
| **Error rate per action** | Via log search query (no code changes needed) | ✅ Out of the box |
| **400/500 breakdown per action** | Via log search — requires logging the HTTP status code explicitly | ✅ Out of the box |
| **Grafana dashboard per action** | ❌ Not possible with current infra | ✅ Out of the box |
| **Alert per action** | Via log search query (no code changes needed), but requires extra alert setup; not metric-native | ✅ Out of the box |
| **CPU/memory per action** | ❌ Pod-level only | ❌ Pod-level only |
| **Tracing (distributed trace per action)** | ⚠️ All actions share the same span name | ✅ Distinct span name per action |
| **Throughput / RPS per action** | Via log search query (no code changes needed) | ✅ Out of the box |
| **Timeout rate per action** | Via log search — requires explicitly logging timeouts with the action name | ✅ Out of the box |
| **Apdex score per action** | ❌ Not practical — depends on latency data which itself requires added logging | ✅ Out of the box |

## Alerting

With a **single endpoint**, alerts fire on the merged metric — you know something is wrong but not which action is the cause. Surfacing the action in an alert requires additional log search-based alerting that parses log lines.

With **split endpoints**, the action is baked into the metric label. Alerts automatically communicate which action is affected with no extra configuration.

---

## Other Pros and Cons

### In favour of split endpoints

- **Spring routing is explicit** — each endpoint maps to exactly one method; no `if/else` routing logic in the controller.
- **Swagger/OpenAPI docs are cleaner** — each action gets its own entry with its own description, request/response examples, and error codes.
- **Easier to add action-specific validation** — e.g., stop might reject if no carrier is ASSIGNED; resume might reject if `isStopped=false`. With split endpoints, that lives in the method. With a generic endpoint, you are adding more branches to an already-branchy method.
- **Easier to add action-specific request bodies** — if stop, resume, and restart ever need different payloads, split endpoints handle that naturally.
- **Easier to apply security/auth per action** — if resume ever needs a different role than stop, split endpoints make that trivial.
- **Cleaner error messages** — a 400 from `/operations/stop/{planId}` is self-documenting; a 400 from `/operations/{action}/{planId}` requires the caller to parse the response body to understand what went wrong.

### In favour of a single endpoint

- **Less boilerplate** — one method instead of three; shared `InvocationContext` setup is not duplicated.
- **Consistent URL pattern** — callers do not need to know the exact action URL, just the pattern.
- **Easier to add new actions** — adding `pause` is one new `else if` branch vs. a new method, interface entry, impl override, gateway config file, and test class.

### Things that cut both ways

- **Testing** — split endpoints have more test methods but each is simpler and more focused. A generic endpoint has fewer test methods but each needs to cover more branches.
- **Future-proofing** — if there are only ever 3 actions, split is cleaner long-term. If actions might grow to 10+, generic scales better.
- **Code review** — split makes the diff per PR smaller and more reviewable; generic consolidates everything but makes each PR's change harder to isolate.

---

## Key Decision Factors

1. **Traffic volume** — High-frequency APIs need per-endpoint alerting to catch regressions fast. Low-volume operator actions can tolerate merged metrics.

2. **SLA/SLO requirements** — If there is a formal latency or error rate target on a specific action, split endpoints are required to measure it independently.

3. **Incident response** — When an on-call alert fires, do you need to know *which* action is broken immediately, or is "something is broken" sufficient to start investigating?

4. **Future growth** — If the feature is likely to expand (more actions, higher volume), splitting now is cheaper than retrofitting observability later.

5. **Action divergence** — Do the actions share enough structure that combining them saves meaningful duplication, or are they just superficially similar? Actions that diverge significantly in business logic lean toward split.

---

## Summary

If most rows in the observability table matter to your team → **split endpoints**.  
If you are comfortable relying on log search logs to answer per-action questions → **single endpoint** is acceptable.

For low-volume operator features (e.g., manual sequential operations), the merged metric is usually acceptable short-term. The risk is when the team later wants SLO dashboards or automated per-action alerting — that is when the single endpoint becomes painful to retrofit.
