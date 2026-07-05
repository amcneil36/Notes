# Technical Debt Best Practices

Technical debt is the gap between the code you have and the code you need. Every shortcut, every "we'll fix it later," every framework you outgrew but didn't replace — it accumulates silently until the team spends more time working around problems than solving them. The insidious part is that technical debt doesn't announce itself. It shows up as slower velocity, longer onboarding, more bugs in areas nobody wants to touch, and a growing sense that everything is harder than it should be.

The metaphor is financial debt, and it's apt: some debt is strategic (a mortgage to buy a house), and some is reckless (credit card debt on things you didn't need). The goal isn't zero debt — it's managed debt with a paydown plan.

**The goal: keep debt visible, intentional, and under control — so it's a tool, not a trap.**

---

## Types of Technical Debt

Not all debt is the same. Understanding the type tells you how it got there and how to prioritize paying it down.

### The Quadrant (after Martin Fowler)

|  | **Deliberate** | **Inadvertent** |
|--|---------------|----------------|
| **Prudent** | "We know this isn't ideal, but we're shipping now and will refactor next sprint." | "Now that we've built it, we realize there was a better approach." |
| **Reckless** | "We don't have time for tests." | "What's a design pattern?" |

**Prudent/deliberate debt** is fine — it's a conscious trade-off with a payback plan. **Reckless/inadvertent debt** is what kills teams. It accumulates without awareness and has no plan for resolution.

### Common Categories

| Category | Examples | How It Accumulates |
|----------|---------|-------------------|
| **Code debt** | Duplicated logic, overly complex functions, poor naming, dead code, inconsistent patterns | Copy-paste shortcuts, no refactoring during feature work |
| **Architecture debt** | Monolith that should be services (or vice versa), wrong data store, missing abstraction layers | Original design outgrown, requirements changed |
| **Test debt** | Low coverage, flaky tests, missing integration tests, manual-only QA | "We'll add tests later" (you won't) |
| **Dependency debt** | Outdated frameworks, unpatched libraries, deprecated APIs, end-of-life runtimes | "If it's not broken, don't upgrade" |
| **Infrastructure debt** | Manual deployments, no IaC, inconsistent environments, hand-configured servers | Incremental ad-hoc changes without codification |
| **Documentation debt** | Outdated READMEs, missing architecture diagrams, tribal knowledge in people's heads | "The code is the documentation" |
| **Process debt** | No CI/CD, manual release checklists, unclear ownership, missing runbooks | Team grew but processes didn't |
| **Data debt** | Inconsistent schemas, orphaned tables, missing indexes, no data lifecycle management | Schema changes without cleanup, accumulated one-off migrations |

---

## Identifying Technical Debt

You can't fix what you can't see. Most teams massively underestimate how much debt they carry because nobody is systematically looking for it.

### Signals That Debt Is Accumulating

| Signal | What It Means |
|--------|--------------|
| **Velocity is declining** | Features that used to take 2 days now take a week. The codebase is fighting you. |
| **Bug rate is increasing** | Changes in one area break things in unrelated areas. Coupling and fragility. |
| **Onboarding takes weeks** | New engineers can't be productive because the system is too hard to understand. |
| **"Don't touch that code"** | There are files or modules everyone avoids. That fear is a debt signal. |
| **Workarounds outnumber fixes** | The team adds workarounds instead of fixing root causes. Each workaround is new debt. |
| **Deploys are scary** | Every release feels risky. Rollbacks are common. Confidence is low. |
| **Same bugs keep recurring** | The same class of bug appears repeatedly. The root cause was never fixed. |
| **Manual steps in automated pipelines** | "After the deploy, manually run this script and then restart the cache." |
| **Increasing test flakiness** | Tests fail randomly. Nobody trusts the test suite. People start skipping tests. |
| **Duplicated logic across services** | Three services implement the same business rule slightly differently. |

### Techniques for Finding Debt

**1. Code Hotspot Analysis**

Files that change frequently AND have high complexity are your highest-debt areas. They're where you spend the most time and where changes are most risky.

```
# Files with most commits in the last 6 months
git log --since="6 months ago" --pretty=format: --name-only | sort | uniq -c | sort -rn | head -20

# Cross-reference with complexity (lines of code as a rough proxy)
wc -l $(git log --since="6 months ago" --pretty=format: --name-only | sort -u)
```

Files that are both large AND frequently modified are your top candidates for refactoring.

**2. Dependency Audit**

```
# How many dependencies are outdated?
# (language-specific commands)
npm outdated          # JavaScript
mvn versions:display-dependency-updates  # Java
pip list --outdated   # Python
```

Count the number of major version bumps behind. Each major version is a bigger migration later.

**3. Test Coverage Gaps**

Coverage numbers aren't everything, but areas with zero coverage are a clear signal. Look for:
- Business-critical paths with no tests
- Code that was "too complex to test" (i.e., too complex, period)
- Test files that haven't been updated in months despite code changes

**4. Team Survey**

Ask every engineer on the team:
1. What part of the codebase do you dread working in? Why?
2. What manual task do you do repeatedly that should be automated?
3. What's the thing you'd fix if you had one free week?
4. Where do you see bugs happen most often?

The answers won't surprise you — but having them documented and aggregated makes them actionable.

**5. Architecture Decision Records (ADR) Review**

If you have ADRs, review the "rejected alternatives" sections. Sometimes the original constraints have changed, and the rejected approach is now the right one. If you don't have ADRs, that's documentation debt.

---

## Quantifying Debt

"We have tech debt" isn't actionable. "We spend 30% of sprint capacity working around debt in the payments module" is.

### Metrics That Make Debt Visible

| Metric | What to Measure | What It Reveals |
|--------|----------------|-----------------|
| **Change failure rate** | % of deployments that cause incidents | High = fragile code, insufficient testing |
| **Lead time for changes** | Time from commit to production | Increasing = friction from debt |
| **Bug density per module** | Bugs filed per module per quarter | Hotspots of code quality problems |
| **Time spent on unplanned work** | Hours spent on bug fixes, incidents, workarounds vs. planned features | The "interest payment" on your debt |
| **Deployment frequency** | How often you deploy to production | Decreasing = growing friction |
| **Onboarding time** | Days until a new engineer ships their first meaningful PR | Increasing = complexity/documentation debt |

### Estimating Cost

For each debt item, estimate:
1. **Interest cost:** How much time does this debt cost the team per sprint? (e.g., "The manual deploy process costs 4 hours per sprint")
2. **Paydown cost:** How much effort to fix it? (e.g., "Automating deploys would take 2 sprints")
3. **Risk:** What's the probability and impact of this debt causing an incident?

This gives you a payback period: if the manual deploy costs 4 hours/sprint and automation takes 80 hours, it pays for itself in 20 sprints (~5 months). That's a clear ROI calculation you can present to stakeholders.

---

## Prioritizing Debt Paydown

You can't fix everything at once. You shouldn't try. Prioritize based on impact, not aesthetics.

### The Priority Matrix

| | **High Frequency (touched often)** | **Low Frequency (rarely touched)** |
|--|-----------------------------------|----------------------------------|
| **High Impact (critical path)** | Fix first. This is your biggest risk and your biggest drag on velocity. | Fix when you're nearby. The risk is real but the drag is low. |
| **Low Impact (non-critical)** | Fix opportunistically. It's annoying but not dangerous. | Ignore until it becomes high-impact or high-frequency. |

### Prioritization Criteria

| Factor | Weight | Question |
|--------|--------|----------|
| **Blast radius** | High | If this breaks, how many users/services are affected? |
| **Velocity drag** | High | How much does this slow down feature development? |
| **Incident frequency** | High | Has this caused incidents? How often? |
| **Interest rate** | Medium | Is this debt getting worse over time, or is it stable? |
| **Fix cost** | Medium | Is this a 2-hour fix or a 2-month project? |
| **Opportunity alignment** | Medium | Are we already working in this area? Can we fix it alongside feature work? |

### What NOT to Prioritize

- Cosmetic improvements to code nobody touches
- Rewriting systems that work fine because "we'd do it differently now"
- Framework upgrades with no security or functionality motivation
- "Gold plating" — making already-good code perfect

---

## Strategies for Paying Down Debt

### 1. The Boy Scout Rule ("Leave it better than you found it")

Every PR that touches a file should improve it slightly. Rename a confusing variable. Extract a method. Add a missing test. Fix a TODO.

**This is the most important strategy.** It's not dramatic, but it's compound interest in your favor. Over months, frequently-touched files get incrementally better without dedicated "tech debt sprints."

**Rules:**
- Keep the improvement small — don't mix large refactors with feature work
- The improvement should be in the same file/module you're already changing
- Never block a feature PR to do a large refactoring — that's a separate PR

### 2. Dedicated Allocation (The 20% Rule)

Reserve a fixed percentage of team capacity for debt paydown. Common approaches:

| Approach | How It Works | Trade-offs |
|----------|-------------|-----------|
| **Fixed sprint percentage** | 20% of every sprint is for tech debt | Steady progress. Requires discipline to protect. |
| **Debt sprint** | Every 5th sprint is 100% tech debt | Big chunks get done. Feature work pauses. Stakeholder buy-in needed. |
| **Friday debt day** | One day per week for tech debt | Easy to implement. Easy to let slip. |
| **Embedded in feature work** | Refactoring is part of the feature estimate | Most sustainable. Requires honest estimation. |

**Recommended:** Embed in feature work as the baseline, with 10-20% explicit allocation for debt that isn't adjacent to any feature. The dedicated allocation is for the stuff that will never be "in the neighborhood" of a feature.

### 3. Strangler Fig Pattern (for Large Rewrites)

Don't rewrite large systems from scratch. Build the new system alongside the old one, redirecting traffic piece by piece until the old system can be decommissioned.

```
Phase 1: New system handles 1 endpoint, old system handles 99
Phase 2: New system handles 10 endpoints, old system handles 90
...
Phase N: New system handles everything, old system decommissioned
```

**Why this works:** You deliver value incrementally. There's no "18-month rewrite that might fail." Each phase is a deployable, testable unit.

**Why rewrites fail:** The old system is still accumulating features and fixes. If the rewrite takes too long, it has to play catch-up with a moving target. Strangler fig avoids this by making the new system production-ready from Phase 1.

### 4. Refactoring Before Feature Work

When a feature requires changes in a high-debt area, do the refactoring first in a separate PR.

```
PR 1: Refactor PaymentProcessor — extract interface, add tests, clean up error handling
PR 2: Add new payment method (builds on clean foundation from PR 1)
```

This is more upfront work but results in:
- Easier code review (refactoring and feature changes reviewed separately)
- Lower risk (refactoring can be rolled back independently)
- Better test coverage (tests added during refactoring protect the feature)

### 5. Dependency Update Sprints

Batch dependency updates into focused sessions rather than letting them pile up:

- **Monthly:** Minor version bumps, security patches
- **Quarterly:** Major version upgrades, framework updates
- **As needed:** Emergency security patches (zero-day CVEs)

Each session: update, run full test suite, deploy to staging, verify, deploy to production. Batching reduces context-switching overhead.

---

## Preventing New Debt

Paying down debt while continuously adding more is a treadmill. Prevention is cheaper than remediation.

### At the Code Level

| Practice | What It Prevents |
|----------|-----------------|
| Code review with quality standards | Shortcuts that create code debt |
| Definition of done includes tests | Test debt |
| Linting / static analysis in CI | Style inconsistencies, common bugs |
| Architecture decision records (ADRs) | "Why did we do it this way?" becoming unanswerable |
| Automated dependency scanning | Dependency debt |

### At the Process Level

| Practice | What It Prevents |
|----------|-----------------|
| Sprint retrospectives that track debt | Debt becoming invisible |
| Explicit "tech debt" ticket type | Debt being lumped into feature estimates and never done |
| Tech debt budget in roadmap planning | Leadership treating all capacity as feature capacity |
| Onboarding feedback loops | Documentation debt becoming normalized |

### At the Architecture Level

| Practice | What It Prevents |
|----------|-----------------|
| Clear module boundaries | Coupling that makes debt expensive to fix |
| Defined interfaces between services | Tight coupling across service boundaries |
| Consistent patterns across the codebase | "Every module works differently" debt |
| Regular architecture reviews | Design that's diverged from requirements |

### The "Debt Deposit" Rule

When you take on deliberate debt (a shortcut to hit a deadline), create a ticket immediately:
1. **What** the shortcut is
2. **Why** it was taken
3. **What** the proper implementation looks like
4. **When** it should be fixed (target sprint or quarter)
5. **What happens** if it's not fixed (risk)

Debt without a ticket is invisible debt. Invisible debt doesn't get fixed.

---

## Communicating Debt to Stakeholders

Engineers know debt is a problem. The challenge is explaining it to stakeholders who measure progress in features shipped.

### What Works

| Approach | Example |
|----------|---------|
| **Translate to business impact** | "This tech debt is adding 2 weeks to every feature in the payments module. Fixing it takes 3 weeks and saves 2 weeks per feature going forward." |
| **Show the trend** | "Our deployment frequency dropped from 10/week to 3/week over the last 6 months. This is the tax we're paying on accumulated debt." |
| **Use incident data** | "3 of our last 5 incidents were caused by the same legacy module. The fix is a focused refactoring effort." |
| **Frame as risk** | "This deprecated library has known security vulnerabilities. Every day we don't upgrade is a day we're exposed." |
| **Compare to financial debt** | "We borrowed time to ship Q2 features fast. Now we're paying interest — every sprint, 20% of our time goes to working around the shortcuts we took." |

### What Doesn't Work

| Approach | Why |
|----------|-----|
| "The code is messy" | Not actionable. Leadership can't evaluate "messy." |
| "We need to rewrite everything" | Sounds expensive, risky, and unscoped. |
| "We should use [shiny new technology]" | Sounds like engineering hobby time. |
| "It's just the right thing to do" | "Right" doesn't appear on a roadmap. |

---

## Common Pitfalls

| Pitfall | Problem | Fix |
|---------|---------|-----|
| **All-or-nothing thinking** | "We need 3 months to fix our debt" → never approved → never started | Start with 10-20% allocation. Show results. Earn more capacity. |
| **Invisible debt** | No tickets, no tracking, no visibility. Leadership doesn't know it exists. | Track debt items in your ticket system. Report on them in sprint reviews. |
| **Gold plating** | Refactoring code that works fine and nobody touches. Perfectionism disguised as debt paydown. | Prioritize by frequency and impact, not aesthetics. |
| **Rewrite fever** | "Let's rewrite the whole system in Rust." The rewrite takes 2 years and doesn't ship. | Use strangler fig. Deliver value incrementally. Prove the new approach before committing. |
| **Tech debt as punishment** | Debt work is assigned to junior engineers or the person with the lightest sprint. | Debt work is real work. Senior engineers should do the hardest debt items. |
| **No definition of done for debt** | Debt item "refactor auth module" sits open for 6 months. What does "done" mean? | Define acceptance criteria for debt tickets just like feature tickets. |
| **Ignoring interest rate** | Treating all debt equally. Some debt gets worse over time, some is stable. | Prioritize accelerating debt (where interest compounds) over stable debt. |
| **Confusing debt with complexity** | "This system is complex" ≠ "This system has tech debt." Some things are genuinely complex. | Debt is the gap between what you have and what you need. Complexity is sometimes the correct solution. |

---

## Quick Reference Checklist

- [ ] **Debt is tracked** — every known debt item has a ticket with description, impact, and estimated fix cost
- [ ] **Debt is categorized** — code, architecture, test, dependency, infrastructure, documentation, process
- [ ] **Debt is prioritized** — by frequency of impact, blast radius, and velocity drag — not aesthetics
- [ ] **Capacity is allocated** — 10-20% of sprint capacity for explicit debt work, plus Boy Scout improvements in every PR
- [ ] **Debt is measured** — change failure rate, lead time, bug density, time on unplanned work
- [ ] **Prevention is in place** — code review standards, CI checks, ADRs, definition of done includes tests
- [ ] **New debt is deliberate** — shortcuts are documented with a payback plan, not taken accidentally
- [ ] **Stakeholders are informed** — debt impact translated to business terms (velocity, risk, cost)
- [ ] **Large rewrites use strangler fig** — incremental delivery, not big-bang replacement
- [ ] **Dependencies are updated regularly** — monthly patches, quarterly major upgrades
- [ ] **Debt work is reviewed in retros** — what did we pay down? What's still accumulating?
