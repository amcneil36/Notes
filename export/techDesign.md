# Tech Design Best Practices

A tech design document is the cheapest place to find mistakes. It's far easier to delete a paragraph than to revert a deployment. But too many designs are either so vague they don't prevent anything, or so over-engineered they never get built as written. This guide covers how to write tech designs that are useful, reviewable, and actually followed — whether you're designing a new system or a single feature.

---

## When Do You Need a Tech Design?

Not every change needs a design doc. Writing one when you don't need it wastes time. Skipping one when you do need it wastes more.

| Signal | Design Doc? | Why |
|--------|-------------|-----|
| New service or system | **Yes** | Multiple teams will depend on your decisions |
| Cross-team feature | **Yes** | Alignment is the whole point |
| Feature touching 3+ services | **Yes** | Coordination cost is high enough to warrant upfront planning |
| Data model changes with migration | **Yes** | Irreversible or expensive to fix — think it through |
| Performance-critical path changes | **Yes** | You need to prove the approach before building it |
| Bug fix or small refactor | **No** | A PR description is sufficient |
| Adding a CRUD endpoint | **No** | Unless it introduces a new pattern or data model |
| Spiking / prototyping | **No** | Write the design *after* the spike, not before |

**Rule of thumb:** If you'd be nervous merging the PR without talking to someone first, write it down instead of Slacking it.

---

## The Two Levels of Tech Design

### System Design

Covers architecture, service boundaries, data flow, and infrastructure. Written when creating something new or making fundamental changes to how systems interact.

**Audience:** Staff+ engineers, architects, partner teams, future you.

**Typical lifespan:** Referenced for months or years. Worth maintaining.

### Feature Design

Covers how a specific feature will be implemented within an existing system. Focused on API contracts, data model changes, edge cases, and rollout.

**Audience:** Your team, your reviewers, your on-call rotation.

**Typical lifespan:** Referenced during implementation, then rarely again. Doesn't need to be a living document.

| Aspect | System Design | Feature Design |
|--------|--------------|----------------|
| **Length** | 5-15 pages | 1-5 pages |
| **Alternatives section** | Required — this is where the value lives | Required but can be brief |
| **Diagrams** | Architecture, data flow, sequence diagrams | Sequence diagrams, state machines |
| **Review audience** | Cross-team, architecture review | Your team + directly affected teams |
| **Approval needed** | Usually formal | Lightweight — 1-2 reviewers |

---

## Recommended Document Structure

This template works for both levels. Scale sections up or down based on complexity.

### 1. Title & Metadata

```
# [Feature/System Name] Tech Design

Author: [Name]
Date: [Date]
Status: [Draft | In Review | Approved | Superseded]
Reviewers: [Names]
```

**Status matters.** Readers need to know if this is a proposal, an agreed-upon plan, or a historical artifact. A design with no status is a design nobody trusts.

### 2. Problem Statement

What problem are you solving? Why does it matter? What happens if you do nothing? A strong problem statement includes concrete numbers, describes the impact, and does not prescribe a solution. See PRD for detailed guidance and examples on writing effective problem statements.

### 3. Goals and Non-Goals

Explicitly state what's in scope and what isn't. Non-goals are just as important — they prevent scope creep and set expectations.

```
## Goals
- Reduce product service p99 latency to <200ms at 15k RPM
- Ensure cache staleness does not exceed 60 seconds
- No changes to the public API contract

## Non-Goals
- Caching for the search service (separate design)
- Real-time cache invalidation (eventual consistency is acceptable)
- Migrating off the current database (out of scope)
```

### 4. Proposed Solution

The meat of the document. Describe *what* you're building and *how* it works. Include:

- **Architecture changes** — new components, modified interactions, removed dependencies
- **API contracts** — request/response shapes, not just "an endpoint"
- **Data model changes** — schema diffs, migration strategy
- **Sequence diagrams** — for any flow involving 3+ components
- **Error handling** — what fails, how it fails, what the user sees
- **Rollout plan** — feature flags, phased rollout, rollback strategy

**Bad: Hand-wavy diagram**
```
Client -> Service -> Cache -> DB
```

**Good: Enough detail to implement from**
```
┌────────┐     ┌──────────────┐     ┌───────────┐     ┌────────┐
│ Client │────>│ Product API  │────>│ Redis     │────>│ CatDB  │
│        │<────│ (read-through│<────│ (TTL: 60s)│<────│        │
└────────┘     │  + fallback) │     └───────────┘     └────────┘
               └──────────────┘
                     │
                     │ cache miss or Redis down
                     │ → query DB directly
                     │ → circuit breaker after 5 failures in 10s
```

### 5. Alternatives Considered

**This is the most important section for reviewers.** If you only considered one approach, you didn't design — you just described what you were already going to do.

For each alternative:
- What is it?
- Why is it reasonable?
- Why did you reject it?

| Approach | Pros | Cons | Verdict |
|----------|------|------|---------|
| Read-through Redis cache | Simple, proven pattern, handles thundering herd | Extra infra (Redis), cache invalidation complexity | **Selected** — best trade-off for our latency target |
| Application-level in-memory cache (Caffeine/Guava) | No extra infra, lowest latency | Cache per instance = inconsistency across pods, memory pressure | Rejected — inconsistency is unacceptable for pricing data |
| CDN caching at edge | Offloads completely, massive scale | 60s TTL too aggressive for CDN, complicates purging | Rejected — doesn't meet staleness requirement |

**The table format forces clarity.** Prose alternatives sections tend to bury the reasoning.

### 6. Risks and Open Questions

Be honest about what you're unsure about. A design that claims zero risk is a design that hasn't been thought through.

```
## Risks
- Redis failure during peak traffic could cause DB overload
  - Mitigation: Circuit breaker with fallback to degraded response (cached stale data)
- Cache stampede on popular products after TTL expiry
  - Mitigation: Probabilistic early expiration (jitter on TTL)

## Open Questions
- Should we use a shared Redis cluster or a dedicated one for this service?
- What is the acceptable cache miss ratio during cold start?
```

### 7. Metrics and Success Criteria

How will you know the design worked? Define measurable outcomes.

```
## Success Criteria
- p99 latency < 200ms at 15k RPM (measured over 7-day window)
- Cache hit ratio > 90% after warm-up
- Zero increase in error rate vs. baseline
- Redis memory usage < 2GB
```

### 8. Timeline and Milestones (Optional)

For larger designs, break the work into phases. This is especially useful when the design spans multiple sprints or teams.

---

## Common Anti-Patterns

### 1. The "Solution Looking for a Problem" Design

You picked the technology first and wrote the design to justify it.

**Symptoms:**
- The problem statement is vague or retrofitted
- Alternatives section is perfunctory — one "real" option and two strawmen
- Heavy focus on implementation details, light on *why*

**Fix:** Write the problem statement and goals *before* thinking about solutions. If you can't articulate the problem clearly, you're not ready to design.

### 2. The "Novel-Length" Design

Fifteen pages of prose, no diagrams, no tables, no code. Reading it feels like a chore. Nobody finishes it. Reviews are superficial.

**Symptoms:**
- Wall-of-text sections with no visual breaks
- Reviewers approve without meaningful feedback
- Implementation diverges because nobody remembered the details

**Fix:** Use tables for comparisons, diagrams for flows, code for contracts. If a section is longer than a page, break it up or cut it. Design docs are not essays — they're reference material.

### 3. The "Too Vague to Be Wrong" Design

```
The service will handle requests efficiently and scale as needed.
We will use appropriate caching strategies.
Error handling will follow best practices.
```

None of these statements can be reviewed because none of them say anything. A design that can't be wrong also can't be right.

**Fix:** Replace every weasel word with a number, a name, or a concrete decision:
- "efficiently" → "p99 < 200ms"
- "appropriate caching" → "Redis read-through with 60s TTL"
- "best practices" → "retry 3x with exponential backoff, circuit-break after 5 failures in 10s"

### 4. The "Over-Engineered for Day One" Design

You designed for 100x scale, event sourcing, CQRS, and a plugin architecture — for a feature that has 50 users.

**Symptoms:**
- The design introduces abstractions that only pay off at a scale you haven't reached
- "Future-proof" appears multiple times
- The estimated timeline keeps growing
- You're building infrastructure, not solving the stated problem

**Fix:** Design for 10x, not 100x. Make it easy to evolve, not easy to extend. Prefer concrete decisions now over abstract flexibility for later. You can always write a v2 design when the requirements actually change.

### 5. The "No Alternatives" Design

The design goes straight from problem to solution with no evidence that other approaches were considered.

**Symptoms:**
- No "Alternatives Considered" section, or it's empty
- Reviewers ask "did you consider X?" and the answer is always "no"
- The solution happens to use the author's favorite technology

**Fix:** Consider at least two genuine alternatives. "Do nothing" counts as one. If you truly can't think of alternatives, you don't understand the problem space well enough.

### 6. The "Museum Piece" Design

Written once, approved, never updated. The actual implementation drifted significantly, but the design doc still says v1. New team members read it and get confused.

**Symptoms:**
- Implementation doesn't match the doc
- The doc has no "Status" field or it still says "Approved" from 18 months ago
- Nobody knows whether the doc is current

**Fix:** Mark designs with a status. When implementation diverges significantly, either update the doc or mark it `Superseded` with a link to what replaced it. A wrong document is worse than no document.

---

## Writing Tips for Better Designs

### Lead with Decisions, Not Discovery

Your doc should present conclusions, not narrate your research process. Readers don't need to follow your journey — they need to evaluate your destination.

**Bad:** "First we looked at Redis, then we considered Memcached, then we talked to the platform team about Hazelcast..."

**Good:** "We chose Redis with read-through caching. See Alternatives Considered for why we rejected Memcached and Hazelcast."

### Use Concrete Numbers

Vague designs produce vague reviews. Concrete numbers give reviewers something to push back on.

| Vague | Concrete |
|-------|----------|
| "High throughput" | "12k RPM sustained, 25k RPM peak" |
| "Low latency" | "p50 < 50ms, p99 < 200ms" |
| "Large dataset" | "~500M rows, ~200GB, growing 2% monthly" |
| "Eventually consistent" | "Consistency window < 60 seconds under normal operation" |
| "Scalable" | "Horizontal scale to 20 pods, tested to 50k RPM" |

### Diagrams Are Not Optional

For any design involving more than two components, include at least one diagram. Sequence diagrams are the highest-value diagram type — they expose timing, ordering, and failure modes that architecture boxes-and-arrows diagrams miss.

You don't need a diagramming tool. ASCII diagrams, Mermaid, or even a photo of a whiteboard are all fine. The goal is clarity, not aesthetics.

### Write the Rollback Plan

Every design should answer: "What happens if this doesn't work and we need to undo it?"

- Can you feature-flag it off?
- Is the data migration reversible?
- Can you run old and new in parallel?
- What's the blast radius of a rollback?

If the answer to all of these is "no" or "I don't know," your design is riskier than you think.

---

## The Review Process

### As the Author

- **Share early.** A half-finished design gets better feedback than a polished one. Reviewers feel obligated to approve something you clearly spent weeks on.
- **Name your reviewers.** "Sending to the team" means nobody owns the review. Pick 2-3 people who will actually read it.
- **Call out what you're unsure about.** Highlight the sections where you want pushback. Reviewers will focus there instead of bikeshedding your naming choices.
- **Set a deadline.** "Please review by Friday" gets responses. "Please review when you can" doesn't.

### As a Reviewer

- **Read the whole thing before commenting.** Context from later sections often answers questions from earlier ones.
- **Focus on the alternatives section.** This is where the real design decisions live. Push back on rejected alternatives if the reasoning is weak.
- **Ask "what happens when X fails?"** The failure modes section reveals how deeply the author has thought about the problem.
- **Don't wordsmith.** Design reviews are about correctness and trade-offs, not prose quality.
- **Approve with conditions if appropriate.** "Approved, but please add a rollback plan for the migration" is more useful than blocking indefinitely.

---

## Related Guides

- See PRD for writing effective problem statements and product requirements.

---

## Quick Reference Checklist

- [ ] Problem statement includes concrete numbers and success criteria
- [ ] Goals and non-goals are explicitly stated
- [ ] At least two genuine alternatives were considered (including "do nothing")
- [ ] Alternatives comparison uses a table with pros, cons, and verdict
- [ ] Proposed solution includes API contracts or data model changes (not just prose)
- [ ] At least one diagram exists for multi-component flows
- [ ] Error handling and failure modes are addressed
- [ ] Rollout strategy is defined (feature flags, phased rollout, canary)
- [ ] Rollback plan exists and is realistic
- [ ] Risks and open questions are honestly listed
- [ ] Success metrics are measurable and time-bound
- [ ] Document has a status field (Draft / In Review / Approved / Superseded)
- [ ] Named reviewers are assigned with a review deadline
- [ ] Design is concise enough that reviewers will actually read it
