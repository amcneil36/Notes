# Monolith vs Microservices: When to Use Each

> **Reference:** Martin Fowler — MonolithFirst (2015)
> Sam Newman — *Building Microservices* (O'Reilly)

---

## The Core Tradeoff

Microservices solve real problems — but only problems you actually have. The cost of distributed systems is real and immediate; the benefits are conditional on scale and team size. Starting with microservices means paying the distributed systems tax before you know where your actual seams are.

**The guiding principle:** Architecture complexity should be proportional to the problems it solves. Don't import microservices complexity to solve a problem you don't have yet.

---

## Quick Decision Guide

Each row links to a deep dive below.

| Factor | Lean Monolith | Lean Microservices |
|--------|---------------|---------------------|
| Scaling model | Traffic fits on a few instances | Components have wildly different load profiles |
| Blast radius & fault isolation | Risk of total outage is acceptable | Must contain failures to individual components |
| Deployment & release cadence | Coordinated releases are manageable | Components need independent ship cycles |
| Team coordination | Fewer than ~20 engineers on the codebase | 50+ engineers, coordination costs outweigh distributed costs |
| Debugging & observability | Want simple stack traces in one process | Already invested in distributed tracing |
| Refactoring & cross-cutting changes | Domain boundaries are still emerging | Boundaries are well-understood and stable |
| Operational overhead | Want one pipeline, one deploy, one config | Can absorb per-service infrastructure ceremony |
| Technology flexibility | Uniform stack is fine | Modules have genuinely different runtime needs |

---

## 1. Scaling Model

### Monolith: Scale everything together

A monolith scales as a single unit. When one module gets hot, you must scale the entire application to handle it. You cannot dedicate more compute to a single high-traffic module without also scaling every low-traffic module alongside it. At some traffic level this becomes expensive — if one endpoint needs 40 pods, every other endpoint in the system also runs on 40 pods whether it needs them or not.

For most systems this is fine. If your traffic fits on a handful of reasonably-sized instances, you're not overpaying by much, and the simplicity of a single scaling target is a real operational win.

### Microservices: Scale each component independently

Each service can be scaled to exactly the compute it needs. A message ingestion service handling 3,000 RPS gets 40 pods; a config lookup service handling 1 RPS gets 2. You pay for what you need, and a traffic spike in one service doesn't require provisioning resources for unrelated ones.

The flip side: you now have N scaling configurations to manage, N sets of HPA rules, N capacity plans. Each service's scaling behavior needs to be understood, tuned, and monitored independently. This operational cost is worth paying when the load profiles are genuinely different — orders-of-magnitude differences, not small variations.

### Pick monolith when: traffic fits on a few instances. Pick microservices when: components have orders-of-magnitude different load profiles (e.g., 3,000 RPS ingestion vs. 1 RPS config lookups).

---

## 2. Blast Radius & Fault Isolation

### Monolith: Everything fails together

A bad deploy, uncaught exception, or memory leak in *any* module takes down the entire system. Every team's features become unavailable simultaneously. No team can ship anything until the issue is resolved and a rollback or hotfix is deployed. One engineer's bad commit can halt the entire engineering org.

The mitigation here is discipline — good testing, canary deploys, feature flags, and circuit breakers within the monolith. These help, but they don't change the fundamental blast radius.

### Microservices: Failures are contained

A bug in Service A doesn't take down Service B. A bad deploy or crash is contained to one service — the rest of the platform keeps running. Teams can ship on their own cadence without risking each other's uptime.

But containment isn't free. You need to design for partial failure — what happens when Service A is down but Service B depends on it? Circuit breakers, fallbacks, retry policies, and graceful degradation all become necessary. A monolith either works or it doesn't; a microservices system can be in a thousand partially-degraded states, each of which needs to be handled correctly.

### Pick monolith when: total blast radius is acceptable (small team, not mission-critical). Pick microservices when: an outage in one component must not take down unrelated functionality.

---

## 3. Deployment & Release Cadence

### Monolith: One artifact, coordinated releases

One deployable unit means one deployment config, one release process, one pipeline. Engineers focus on domain logic, not infrastructure ceremony. The trade-off is coordination — when many teams contribute to the same artifact, deploying means shipping everyone's changes together. Feature flags and strict branching strategy become critical to avoid "my feature is ready but I'm blocked by your broken commit."

### Microservices: Ship independently

Teams can deploy their service on their own cadence without coordinating with every other team. Service A ships three times a day while Service B ships once a month. This autonomy is powerful at scale.

The cost shows up in cross-service changes. A refactor that touches 3 services requires 3 coordinated PRs, 3 deploys, and careful sequencing. What was a single method rename in a monolith becomes an API versioning problem. You also need backwards-compatible API evolution (additive changes, deprecation windows) to avoid breaking consumers during rollouts.

### Pick monolith when: one team or a few teams can coordinate releases comfortably. Pick microservices when: teams genuinely need to ship independently at different cadences.

---

## 4. Team Coordination

### Monolith: Shared codebase, shared discipline

Many teams working on one codebase requires discipline — module boundaries enforced by package structure, clear ownership conventions, and code review norms. But the upside is significant: new engineers clone one repo, run one command, and have a working system. There is no mental model of "which service owns what" and no per-service setup ritual. Shared types are just types — no published library versioning across repos.

At some team size (roughly 50+ engineers on the same codebase), the coordination costs of a shared codebase start to outweigh the simplicity benefits. Merge conflicts multiply, CI times grow, and the "everyone touches everything" model breaks down.

### Microservices: Team autonomy via service ownership

Teams own their service end-to-end — code, deploy, on-call. This reduces coordination overhead at large org scale. Each team can make decisions about their service's internals without cross-team negotiation.

The cost is a different kind of coordination problem. Shared contracts (API schemas, event formats) need governance. Cross-team changes require RFC-style proposals instead of atomic PRs. A new engineer joining the org has to learn which services exist, who owns them, and how they interact — a mental model that grows linearly with the number of services.

### Pick monolith when: fewer than ~20 engineers, or domain boundaries are still emerging. Pick microservices when: team size makes shared-codebase coordination more expensive than distributed-system coordination.

---

## 5. Debugging & Observability

### Monolith: Full stack traces, one process

Debugging a monolith means reading stack traces. The entire call chain is in one process — you can set a breakpoint, step through the logic, and see the full picture. Log correlation is trivial because there's only one log stream. When something fails at 2 AM, you look at one set of logs.

### Microservices: Distributed tracing across services

A single user request spans multiple services. A failure in Service C might manifest as a timeout in Service A, which surfaces as a 500 in the API gateway. Understanding what went wrong requires distributed tracing (Jaeger, Zipkin), correlation IDs propagated through every hop, and structured logging across all services.

This infrastructure is powerful once it's working, but it's a significant investment to set up and maintain. Without it, debugging a microservices system is like reading a novel with every third chapter ripped out.

### Pick monolith when: you want simple, zero-infrastructure debugging. Pick microservices when: you've already invested in (or are willing to invest in) distributed tracing and observability.

---

## 6. Refactoring & Cross-Cutting Changes

### Monolith: Atomic refactoring, boundaries can shift

Rename a class, move a module — everything compiles or it doesn't. No versioned API contracts, no multi-repo PRs, no deployment sequencing. This is invaluable when domain boundaries are still emerging, because the cost of getting a boundary wrong is low: you refactor it. The right seams reveal themselves through experience, and a monolith lets you reorganize cheaply.

### Microservices: Stable boundaries, expensive to change

Once a service boundary is drawn, moving logic across it is expensive. What was a method call becomes an API migration — new endpoint, data migration, consumer updates, deprecation of the old endpoint. Premature splits lock in wrong boundaries permanently.

This rigidity is a feature when boundaries are well-understood and stable. It enforces separation that might otherwise erode. But it's a severe liability when you're still learning the domain.

### Pick monolith when: domain boundaries are still evolving. Pick microservices when: boundaries are well-understood and you want to enforce them structurally.

---

## 7. Operational Overhead

### Monolith: One of everything

One deployment configuration, one CI/CD pipeline, one Kubernetes namespace, one config group, one canary/Flagger setup, one set of monitoring dashboards, one alerting config, one on-call runbook. Engineers write business logic instead of managing infrastructure per component.

### Microservices: N of everything

Each service requires its own deployment configuration, CI/CD pipeline, Kubernetes namespace, config group, canary/Flagger setup, monitoring dashboards, alerting rules, and on-call runbook. Every engineer working on a service needs to understand all of this just to make a change and ship it — before writing a single line of business logic, they need to know how to configure, launch, and deploy that specific service.

At 5 services this is manageable. At 30+ services, a significant portion of every engineer's time is spent on infrastructure ceremony rather than domain logic. New engineers face a steep ramp on every service they touch. The overhead can be mitigated with platform teams and templated infrastructure, but it never fully disappears.

### Pick monolith when: you want engineers focused on domain logic. Pick microservices when: you can absorb the per-service operational cost (or have a platform team to centralize it).

---

## 8. Technology Flexibility

### Monolith: One stack

All modules must share the same language, runtime, and major framework versions. A dependency upgrade affects the whole system. This is constraining if different modules have genuinely different runtime requirements, but for most applications a single well-chosen stack covers all needs.

### Microservices: Polyglot by nature

Service A can use Java, Service B can use Python. Useful when different modules have genuinely different runtime requirements — an ML optimizer in Python, a high-throughput ingestion service in Go, a CRUD API in Java.

The cost is fragmented tooling, build systems, and expertise requirements. Every language and runtime is another thing to hire for, maintain, and secure. Polyglot is a powerful tool when it solves a real problem; it's an expensive indulgence when "we thought it would be cool" is the justification.

### Pick monolith when: a single stack covers your needs. Pick microservices when: components have genuinely different runtime requirements.

---

## The Spectrum: Modular Monolith

The false dichotomy is "messy monolith vs. clean microservices." The real options are:

```
Monolith (1 module)
    -> Modular Monolith (N modules, 1 deployable)
        -> Macro-services (3-5 deployables by major domain)
            -> Microservices (N deployables by bounded context)
```

A **modular monolith** gives you most of the organizational benefits of microservices (clear ownership, enforced boundaries, independent testability) without the distributed systems tax. Modules are enforced via package structure, not network calls. This is the right starting point for most systems.

Extract to a separate deployable when you have a **specific, measured reason** — not in anticipation of one.

---

## Signals That You Should Extract a Service

| Signal | Example |
|--------|---------|
| Traffic is orders of magnitude higher than the rest of the system | A message ingestion endpoint at 3,000+ RPS when most other endpoints are under 5 RPS |
| Module uses a fundamentally different technology | Python ML optimizer embedded in a Java monolith |
| Module has radically different deployment risk profile | A stable EDI parser vs. a frequently-changing UI backend |
| External system integration demands isolation | An EDI/B2B integration that uses different network paths, credentials, and compliance scope |
| Module needs to scale to zero independently | A batch job that runs once per night |

---

## Signals That You Should NOT Extract a Service (Yet)

| Signal | What it means |
|--------|---------------|
| "We might need to scale this someday" | Measure first. Premature optimization. |
| "The team is growing" | Add modules, not services. |
| "We want team autonomy" | Code ownership and module structure solve this without distributed complexity. |
| Two services always deploy together | They're not actually independent — you're paying the distributed tax for nothing. |
| You're building complex async choreography to work around service boundaries | The seam is wrong. Consider merging back. |

---

## The Mono-Repo vs. Multi-Repo Axis (Separate from Monolith vs. Microservices)

Repo structure and deployment architecture are independent decisions:

| | Multi-Repo | Mono-Repo |
|---|---|---|
| **Microservices** | Common pattern: 1 repo per service | Google/Meta model: all services in one repo, built with selective CI |
| **Monolith** | Unusual | Natural fit |

A **mono-repo of microservices** gives you atomic cross-service PRs, shared tooling, and unified CI without collapsing deployable boundaries. Teams still own their service; they just share a repo. The tradeoff is CI/CD complexity — you need build graph tooling (e.g., Nx, Gradle multi-project, Bazel) to only build what changed.

The biggest mono-repo benefit at scale: **infrastructure ownership can be centralized**. A platform team owns the deployment templates, CI/CD pipeline patterns, and release process. App engineers declare their service in a high-level config and never touch the infrastructure layer. This is only achievable with a mono-repo (or very strong shared library discipline in multi-repo).

---

## Related

See Message Queues for when to use async messaging for decoupling services.

---

## Recommended Reading

- **Martin Fowler — MonolithFirst**: `martinfowler.com/bliki/MonolithFirst.html` — the foundational argument for starting simple
- **Sam Newman — Building Microservices** (O'Reilly) — comprehensive reference; notably, Newman himself recommends starting with a monolith in most cases
- **Martin Fowler — Microservice Trade-Offs**: `martinfowler.com/articles/microservice-trade-offs.html`
- **Majestic Monolith** (DHH / Basecamp): the argument that most applications never need to leave the monolith
