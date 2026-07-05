# PRD (Product Requirements Document) Best Practices

A PRD is the contract between "what we want" and "what we build." When written well, it eliminates ambiguity, kills scope creep before it starts, and gives engineers enough detail to make real technical decisions. When written badly, it becomes a vague wish list that spawns endless Slack threads, mid-sprint pivots, and features that ship but don't actually solve the problem.

This guide is written for **software engineers** who need to author or contribute to PRDs for feature-level work. It's not about the ceremony — it's about writing something useful enough that your team can build from it without guessing.

---

## The Two Failure Modes of PRDs

Every bad PRD falls into one of two traps:

| Failure Mode | What It Looks Like | Consequence |
|---|---|---|
| **Too vague** | "The system should handle errors gracefully." | Engineers interpret this 5 different ways. QA doesn't know what to test. You ship something, but nobody agrees it's right. |
| **Too prescriptive** | "Use a Redis sorted set with TTL of 3600s to rank items by score, exposing a GET /v1/rankings endpoint returning..." | The PRD is now a tech spec. Product can't review it. When requirements change, the whole thing needs a rewrite because it's coupled to an implementation. |

**The sweet spot:** A PRD should be specific enough that two engineers reading it independently would build the same *behavior*, even if they chose different *implementations*.

---

## What a PRD Is (and Isn't)

| A PRD IS | A PRD IS NOT |
|---|---|
| A description of **what** the system should do and **why** | A technical design document (how) |
| A definition of **done** — acceptance criteria, success metrics | A project plan with timelines and assignees |
| A forcing function to **think through edge cases** before writing code | A spec for database schemas, API contracts, or class hierarchies |
| A **communication tool** between product, engineering, and stakeholders | A ticket description (those come from the PRD, not the other way around) |

**The hand-off:** A PRD feeds into a tech spec. The PRD says "users should be able to cancel an in-progress order within 5 minutes of placement." The tech spec says "we'll add a `cancelOrder` mutation that checks `order.createdAt` against a 5-minute window and publishes a `ORDER_CANCELLED` event to Kafka."

If your PRD contains class names, you've gone too far. If your PRD doesn't contain acceptance criteria, you haven't gone far enough.

---

## Anatomy of a Good PRD

### The Essential Sections

Every feature-level PRD needs these sections. Skip one and you're inviting ambiguity.

| Section | Purpose | Common Mistake |
|---|---|---|
| **Problem Statement** | Why does this feature exist? What pain point does it solve? | Writing a solution description disguised as a problem statement |
| **Goals & Non-Goals** | What's in scope and — critically — what's explicitly out of scope | Listing goals but no non-goals, leaving scope unbounded |
| **User Stories / Use Cases** | Concrete scenarios: "As a [user], I want to [action] so that [outcome]" | Vague stories like "As a user, I want a better experience" |
| **Functional Requirements** | Observable behaviors the system must exhibit | Mixing in implementation details ("use Redis", "add a cron job") |
| **Edge Cases & Error Handling** | What happens when things go wrong or inputs are unexpected | Pretending the happy path is the only path |
| **Acceptance Criteria** | Testable conditions that define "done" | Criteria so vague they can't be turned into test cases |
| **Success Metrics** | How you'll measure whether the feature actually solved the problem | No metrics, or vanity metrics that don't connect to the problem |
| **Dependencies & Risks** | External teams, services, or unknowns that could block or derail | Discovering dependencies mid-sprint |
| **Open Questions** | Things you don't know yet — and who owns answering them | Burying unknowns in prose where they get overlooked |

### Optional But Valuable Sections

| Section | When to Include |
|---|---|
| **User Flow / Wireframes** | When the feature has a UI component and behavior isn't obvious from text |
| **Data Requirements** | When the feature introduces new data models, storage needs, or data flows |
| **Migration / Rollout Plan** | When the feature affects existing data or requires a phased rollout |
| **Security & Privacy Considerations** | When the feature handles PII, auth, or changes access patterns |
| **Internationalization / Localization** | When the feature surfaces user-facing text or date/currency formatting |

---

## Writing Each Section Well

### Problem Statement

**The rule:** If you can't explain the problem without mentioning your solution, you don't understand the problem yet.

Bad:
> We need to add a caching layer to reduce latency on the product detail page.

Good:
> Product detail pages take 2.3 seconds to load on average (P95: 4.1s). This exceeds our 1-second target and is correlated with a 12% bounce rate increase over the past quarter. Users in high-latency regions (LATAM, SEA) are disproportionately affected.

The bad version has already decided the solution (caching). The good version states the problem with data, leaving room for engineers to propose the right fix — which might be caching, might be query optimization, might be a CDN change.

### Goals & Non-Goals

**Non-goals are more important than goals.** Everyone roughly agrees on what they want to build. The fights happen over what's *not* in scope.

Bad:
> **Goals:** Improve checkout experience.

Good:
> **Goals:**
> - Reduce checkout abandonment rate from 34% to below 25% by simplifying the payment step
> - Support Apple Pay and Google Pay as payment methods
> - Maintain PCI compliance for all new payment flows
>
> **Non-Goals (for this iteration):**
> - Redesigning the cart page (separate initiative, Q3)
> - Supporting cryptocurrency payments
> - Changing the existing credit card flow — only adding new methods alongside it

Non-goals aren't "things we'll never do." They're "things we're explicitly not doing *right now*" — and saying so prevents scope creep mid-build.

### User Stories

**Make them specific and testable.** A user story you can't write a test case from is too vague.

Bad:
> As a user, I want to manage my orders easily.

Good:
> As a logged-in customer, I want to cancel an order that hasn't shipped yet, so that I'm not charged for items I no longer want.
>
> **Conditions:**
> - Order status must be "Processing" (not "Shipped" or "Delivered")
> - Order must have been placed within the last 30 minutes
> - Cancellation triggers a full refund to original payment method within 5 business days
> - User receives a confirmation email within 60 seconds of cancellation

The conditions section is where engineers get the detail they need. Without it, you'll get five Slack messages asking "what does 'cancel' mean exactly?"

### Functional Requirements

Write requirements as **observable behaviors**, not implementation instructions. Use the format: "When [trigger], the system should [behavior], resulting in [outcome]."

Bad:
> The system should use a message queue to process refunds asynchronously.

Good:
> When a customer cancels an order, the system should initiate a refund within 30 seconds. The customer should see a "Refund Pending" status immediately. The refund should complete within 5 business days. If the refund fails, the system should retry up to 3 times and alert the support team if all retries fail.

The bad version tells engineers *how* to build it. The good version tells them *what it should do* — they might use a queue, they might not.

### Edge Cases & Error Handling

**This is the section that separates useful PRDs from decorative ones.** If you only describe the happy path, you've described maybe 30% of the actual work.

For every feature, ask:

| Question | Example for Order Cancellation |
|---|---|
| What if the input is invalid? | User tries to cancel an order that doesn't exist, or belongs to another user |
| What if a downstream service is down? | Payment service is unreachable when processing the refund |
| What if the operation partially succeeds? | Order is marked cancelled but refund fails |
| What if there's a race condition? | User clicks cancel while the order is transitioning to "Shipped" |
| What if the user does this twice? | User double-clicks the cancel button, or retries after a timeout |
| What happens at boundaries? | Order placed exactly 30 minutes ago — is it cancellable or not? |
| What about concurrent users? | Two support agents try to cancel the same order simultaneously |

**Document the expected behavior for each case.** Don't leave it to engineers to guess what the product wants. A table works well:

| Scenario | Expected Behavior |
|---|---|
| Order already shipped | Show error: "This order has already shipped and cannot be cancelled." Offer "Return" flow instead. |
| Payment refund fails | Retry 3 times with exponential backoff. If all fail, mark order as "Cancellation Pending" and create a support ticket automatically. |
| Double-click / duplicate request | Idempotent — second request returns the same result as the first without re-processing. |
| Boundary: exactly 30 min | Inclusive — order placed at T is cancellable until T+30m (inclusive). |

### Acceptance Criteria

**Every acceptance criterion should be directly translatable into a test case.** If a QA engineer can't read it and write a test, it's too vague.

Bad:
> - The feature should work correctly
> - The UI should be responsive
> - Errors should be handled properly

Good:
> - [ ] Given an order in "Processing" status placed < 30 minutes ago, when the customer clicks "Cancel Order", then the order status changes to "Cancelled" and a confirmation email is sent within 60 seconds
> - [ ] Given an order in "Shipped" status, when the customer clicks "Cancel Order", then an error message is displayed: "This order has already shipped" and the order status does not change
> - [ ] Given a cancellation request where the payment refund fails 3 times, then the order status changes to "Cancellation Pending" and a support ticket is created with the order ID, customer ID, and failure reason
> - [ ] Given two simultaneous cancellation requests for the same order, then only one cancellation is processed and both requests return a success response

**The pattern:** Given [precondition], when [action], then [observable result].

### Success Metrics

**Connect metrics to the problem, not the feature.** If your problem statement says "checkout abandonment is too high," your success metric should measure abandonment — not "number of users who clicked the new button."

| Metric Type | Good Example | Bad Example |
|---|---|---|
| **Primary** (solves the problem) | Checkout abandonment rate drops from 34% to below 25% | New payment methods are available |
| **Secondary** (health check) | Average checkout completion time decreases by 15% | Page loads in under 2 seconds |
| **Guardrail** (don't break things) | Failed payment rate stays below 1% | No increase in support tickets |
| **Counter-metric** (unintended harm) | Fraud rate doesn't increase by more than 0.1% | *(not measured)* |

**Guardrail and counter-metrics are easy to forget and expensive to skip.** You don't want to ship a feature that improves checkout conversion but doubles fraud.

### Open Questions

**Don't bury unknowns.** Give them a dedicated section with clear ownership and deadlines.

| Question | Owner | Needed By | Status |
|---|---|---|---|
| What's the refund SLA with the payment processor? | @payments-team | Before sprint starts | Pending |
| Do we need to support partial cancellations (individual items)? | @product-manager | Before tech spec | Answered: No, full order only for V1 |
| What's the legal requirement for cancellation confirmation records? | @legal | Before launch | Pending |

An open question without an owner is just noise. An open question without a deadline is a question that never gets answered.

---

## The Five Deadly Sins of PRDs

### 1. The Solution Masquerading as a Problem

> "We need to build a microservice that..."

No. You need to *solve a problem*. The microservice is one possible solution. If your problem statement contains the word "build," "implement," "add," or "create," you've skipped the problem and jumped to the answer.

### 2. The Unbounded Scope

PRDs without non-goals grow like kudzu. Stakeholders add "one more thing" because nothing explicitly says it's out of scope. Then engineers are halfway through a sprint building a feature that's now twice as large as what was estimated.

**Fix it:** Write non-goals first. Be aggressive. You can always promote a non-goal to a goal in the next iteration.

### 3. The Happy Path Fantasy

If your PRD only describes what happens when everything works perfectly, you've described the easy 30% and left the hard 70% to engineer judgment. Every PRD should answer: "What happens when this fails?"

### 4. The Untestable Acceptance Criteria

"The system should be fast." Fast compared to what? Measured how? Under what load?

"The UI should be intuitive." According to whom? Measured by what — task completion rate? Time on task? Support ticket volume?

**If you can't write a test for it, it's not a requirement — it's a wish.**

### 5. The PRD That Never Locks Down

Some PRDs stay in "draft" forever, accumulating edits, comments, and contradictions. Engineers start building from a moving target.

**Fix it:** Set a review deadline. After that date, changes go through a change request process — not inline edits. Version the PRD. If the scope changes materially, that's a conversation, not a silent update.

---

## Scope Management

Scope creep is the #1 killer of feature timelines. A good PRD is your best defense.

### The MoSCoW Method

Prioritize every requirement explicitly:

| Priority | Meaning | Rule |
|---|---|---|
| **Must Have** | The feature is broken without this | If any Must Have is cut, the feature doesn't ship |
| **Should Have** | Important, but the feature works without it | Can be deferred to a fast-follow if timeline is tight |
| **Could Have** | Nice to have, included if there's time | First thing to cut when scope pressure hits |
| **Won't Have** | Explicitly out of scope for this iteration | This is your non-goals list — write it down |

**The trap:** Everything is a "Must Have." If more than 60% of your requirements are Must Have, you haven't prioritized — you've just labeled everything important.

### Version Your Scope

For any non-trivial feature, define what ships in V1 vs. V2:

| V1 (MVP) | V2 (Fast-Follow) | Future |
|---|---|---|
| Cancel full order | Cancel individual items | Auto-cancel stale orders |
| Email confirmation | Push notification + email | Self-service refund tracking |
| Manual refund trigger | Automated refund processing | Partial refunds |

This makes trade-off conversations concrete. When someone says "can we also add partial cancellations?" you can point to the table and say "that's V2."

---

## Writing for Your Audience

A PRD has multiple readers with different needs:

| Reader | What They Care About | How to Write for Them |
|---|---|---|
| **Engineers** | Exact behavior, edge cases, constraints, acceptance criteria | Be specific. Use Given/When/Then. Include boundary conditions. |
| **Product Managers** | Does this solve the user problem? Are the goals right? | Lead with the problem statement and success metrics. |
| **Designers** | User flows, interaction patterns, error states | Include wireframes or flow descriptions for UI features. |
| **QA Engineers** | Testable scenarios, expected vs. actual behavior | Make acceptance criteria directly translatable to test cases. |
| **Stakeholders / Leadership** | Why are we doing this? What's the impact? | Keep the summary and goals section crisp and outcome-focused. |

**Don't write one section for one audience.** Write every section clearly enough that any reader can understand it, but make sure each audience finds what they need without digging.

---

## PRD Review Checklist

Before you mark a PRD as "ready for review," run through this:

### Completeness
- [ ] Problem statement describes the *problem*, not the solution
- [ ] Goals are measurable (include numbers or clear thresholds)
- [ ] Non-goals are explicitly listed
- [ ] Every user story has testable conditions
- [ ] Functional requirements describe *behaviors*, not implementations
- [ ] Edge cases and error scenarios are documented with expected behavior
- [ ] Acceptance criteria follow Given/When/Then format
- [ ] Success metrics include at least one guardrail or counter-metric

### Clarity
- [ ] Two engineers reading this independently would build the same behavior
- [ ] No ambiguous terms — "fast," "intuitive," "seamless," "robust" are banned without quantification
- [ ] Boundary conditions are explicit (inclusive vs. exclusive, exact thresholds)
- [ ] Acronyms and domain terms are defined on first use

### Scope Control
- [ ] Requirements are prioritized (Must/Should/Could/Won't)
- [ ] V1 vs. V2 scope is clearly separated
- [ ] No implementation details in the requirements (no class names, no "use X technology")
- [ ] Open questions have owners and deadlines

### Readiness
- [ ] All "Must Have" open questions are answered
- [ ] Stakeholders have reviewed and signed off
- [ ] Dependencies on other teams are identified and communicated
- [ ] PRD has a version number and a "locked" date

---

## PRD Template

Copy this template and fill it in. Delete sections that don't apply, but think twice before deleting "Edge Cases" or "Non-Goals" — those are the ones people skip and later regret.

```markdown
# [Feature Name] — PRD

**Author:** [Your Name]
**Last Updated:** [Date]
**Version:** [1.0]
**Status:** [Draft | In Review | Approved | Locked]

---

## Problem Statement

[2-3 sentences describing the problem. Include data if available. Do NOT mention the solution here.]

## Goals

- [Measurable goal 1]
- [Measurable goal 2]

## Non-Goals (This Iteration)

- [Explicit exclusion 1 — and brief reason why]
- [Explicit exclusion 2]

## User Stories

### Story 1: [Short Title]
As a [user type], I want to [action] so that [outcome].

**Conditions:**
- [Specific condition 1]
- [Specific condition 2]

### Story 2: [Short Title]
...

## Functional Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-1 | When [trigger], the system should [behavior], resulting in [outcome] | Must Have |
| FR-2 | ... | Should Have |

## Edge Cases & Error Handling

| Scenario | Expected Behavior |
|----------|-------------------|
| [What if X fails?] | [Expected system response] |
| [What if input is invalid?] | [Expected system response] |
| [What about race conditions?] | [Expected system response] |

## Acceptance Criteria

- [ ] Given [precondition], when [action], then [result]
- [ ] Given [precondition], when [action], then [result]

## Success Metrics

| Metric | Current | Target | Type |
|--------|---------|--------|------|
| [Primary metric] | [Baseline] | [Target] | Primary |
| [Health metric] | [Baseline] | [No regression] | Guardrail |

## Dependencies & Risks

| Dependency / Risk | Owner | Mitigation |
|-------------------|-------|------------|
| [External team or service] | [Team] | [Fallback plan] |

## Open Questions

| Question | Owner | Needed By | Status |
|----------|-------|-----------|--------|
| [Question] | [Person/Team] | [Date] | [Pending/Answered] |

## V1 vs. Future Scope

| V1 (This Release) | V2 (Fast-Follow) | Future |
|--------------------|-------------------|--------|
| [Feature A] | [Feature B] | [Feature C] |
```

---

## Related Guides

- See Tech Design for translating PRD requirements into technical specifications.

---

## Quick Reference

| Principle | One-Liner |
|---|---|
| **Problem first** | If your problem statement contains "build" or "implement," you've skipped the problem |
| **Specific > vague** | "Handle errors gracefully" is not a requirement. "Return a 409 with a retry-after header" is. |
| **Non-goals are mandatory** | Unbounded scope is the default. You have to actively constrain it. |
| **Test it or it's not real** | If QA can't write a test case from your acceptance criteria, rewrite them |
| **Edge cases are the work** | The happy path is 30% of the effort. The other 70% is what your PRD should force you to think about. |
| **Lock it down** | A PRD that changes daily is a conversation, not a spec. Version it. Set a lock date. |
| **Metrics prove impact** | "We shipped the feature" is not success. "Abandonment dropped 9 points" is. |
