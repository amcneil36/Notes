# Prioritization

Everything is important. Nothing is urgent. And yet somehow the team is always firefighting, the backlog is 400 items long, and nobody can explain why Feature X shipped before Bug Y got fixed. Prioritization isn't about making a perfect ordered list — it's about making defensible trade-off decisions fast enough that the team can actually execute.

The goal is simple: at any given moment, the team should be working on the thing that delivers the most value relative to the cost and risk of doing it (or not doing it). Everything else is just mechanics.

---

## How Far Ahead to Plan

Planning too far ahead is waste. Planning too little is chaos. The right horizon depends on the decision you're making.

| Planning Horizon | What You're Deciding | Fidelity |
|---|---|---|
| **Quarterly** (8–12 weeks out) | Themes, bets, and capacity allocation. "What problems are we solving this quarter?" | Low — directional, not task-level. Expect 30–40% of this to shift. |
| **Sprint** (1–2 weeks out) | Specific work items. "What ships this sprint?" | High — committed work, clear acceptance criteria. |
| **Daily** (today/tomorrow) | Execution focus. "What's blocked? What changed?" | Tactical — unblock, adjust, stay on track. |

**The trap**: teams that plan quarterly at sprint-level fidelity burn hours on estimates that are wrong by the time the work starts. Conversely, teams that only plan sprint-to-sprint drift without direction.

**Rule of thumb**: plan at the *coarsest* fidelity that still lets you make the decision. Quarterly planning should fit on one page. Sprint planning should fit in one meeting.

---

## How to Determine Priority

Three forces compete for every item in your backlog:

### 1. Business Value

What's the impact if we do this? Revenue, retention, user experience, operational efficiency, compliance — whatever metric your team is accountable for.

Ask: *"If we ship this, who benefits, and how much?"*

Not everything has a dollar sign. Internal developer tooling that saves 20 engineers 30 minutes a day is high value even if it never touches a customer. Platform work that prevents a 2 AM page every week is high value even if nobody outside the team notices.

### 2. Cost and Effort

How long will this take? How many people? What's the opportunity cost — what *won't* we do while we're doing this?

| Effort Bucket | Duration | Team Impact |
|---|---|---|
| **Small** | < 1 day | One person, no coordination |
| **Medium** | 1–3 days | One person, minor coordination |
| **Large** | 1–2 weeks | Multiple people or cross-team dependency |
| **Extra Large** | > 2 weeks | Needs to be broken down before it's plannable |

If something is XL, don't prioritize it — break it down first. You can't prioritize what you can't estimate, and you can't estimate what you don't understand.

### 3. Risk and Cost of Delay

What happens if we *don't* do this? This is the most underweighted factor in most backlogs.

| Risk Signal | What It Means | Urgency |
|---|---|---|
| Data loss or corruption | Users are losing data right now | Do it now |
| Security vulnerability | Exposure window is open | Do it now |
| Revenue impact | Measurable money being lost per day/week | This sprint |
| Degraded experience | Users can work around it but shouldn't have to | Next sprint |
| Tech debt accumulating | Gets worse the longer you wait, but slowly | This quarter |
| Nice to have | No one suffers if this waits | When there's room |

**Cost of delay is the tiebreaker.** Two items with similar value and effort? The one that gets more expensive to delay wins.

---

## The Priority Matrix

Stop debating priority in the abstract. Use a simple 2x2 to sort quickly:

```
                    HIGH VALUE
                        |
         DO FIRST       |      DO NEXT
        (high value,    |    (high value,
         low effort)    |     high effort)
                        |
   ─────────────────────┼─────────────────────
                        |
         FILL GAPS      |      DROP OR DEFER
        (low value,     |    (low value,
         low effort)    |     high effort)
                        |
                    LOW VALUE
       LOW EFFORT ──────────────── HIGH EFFORT
```

- **Do First**: High value, low effort. These are your quick wins. Ship them.
- **Do Next**: High value, high effort. These are your bets. Plan them, break them down, commit to them.
- **Fill Gaps**: Low value, low effort. Use these as buffer work between bigger items. Good for onboarding or context-switching days.
- **Drop or Defer**: Low value, high effort. Say no. If someone keeps asking for it, make them articulate the value. If they can't, it stays off the board.

---

## Bugs vs. Enhancements

No, they should not be prioritized the same way. They have fundamentally different risk profiles.

### Bugs

Bugs represent **broken promises**. The system was supposed to do X. It doesn't. Someone is affected *right now*.

| Bug Severity | Definition | Response |
|---|---|---|
| **P1 — Critical** | System down, data loss, security breach, or major revenue impact | Drop everything. Fix now. |
| **P2 — High** | Major feature broken, no workaround, significant user impact | Fix this sprint. Displace lower-priority planned work if needed. |
| **P3 — Medium** | Feature impaired but workaround exists, moderate user impact | Prioritize alongside enhancements using the normal matrix. |
| **P4 — Low** | Cosmetic, edge case, or minor inconvenience | Backlog. Fix when convenient or batch with related work. |

**P1 and P2 bugs bypass normal prioritization.** They go to the top of the queue because the cost of delay is immediate and compounding. A P1 bug at 2 AM is not competing with Feature Y for sprint capacity — it's a fire.

**P3 and P4 bugs compete with enhancements.** Treat them like any other backlog item: value vs. effort vs. cost of delay. A P4 bug that's been open for six months and nobody cares about is less important than an enhancement that unblocks a sales deal.

### Enhancements

Enhancements represent **new promises**. They expand what the system can do. Their urgency comes from business opportunity, not system failure.

The key difference: **bugs have a known cost of inaction** (users are suffering). **Enhancements have a speculative cost of inaction** (we *might* lose a deal, we *might* fall behind competitors). Weight accordingly.

**Don't let the bug backlog become a graveyard.** If a bug has been open for 90+ days and nobody has escalated it, either fix it, downgrade it, or close it. A backlog full of ancient P4s that nobody looks at is just noise.

---

## Handling Interrupts

Unplanned work will happen. Production breaks. Leadership asks for something urgent. A partner team needs help. The question isn't whether interrupts will come — it's how you absorb them without derailing everything.

### Reserve Capacity

Don't plan sprints at 100% capacity. Leave a buffer.

| Team Type | Planned Capacity | Buffer for Interrupts |
|---|---|---|
| Product team (stable product) | 80–85% | 15–20% |
| Platform/infra team | 70–75% | 25–30% |
| On-call-heavy team | 60–70% | 30–40% |

If you're consistently using more than your buffer, you don't have an interrupt problem — you have a reliability or process problem. Fix the root cause.

### Classify Before Reacting

Not every interrupt deserves an immediate response.

| Interrupt Type | Response | Timeline |
|---|---|---|
| **Production down** | All hands. Drop planned work. | Now |
| **Leadership escalation** | Assess real urgency. Push back if it's not truly urgent. | Within hours |
| **Partner team request** | Negotiate scope and timeline. Don't just absorb it. | This sprint or next |
| **"Quick question" / "small ask"** | Timebox it. If it's > 2 hours, it goes through normal prioritization. | Backlog if not quick |

**The biggest trap**: treating everything from leadership as P1. Most "urgent" leadership requests are actually "I'd like an update" or "can we explore this?" — not "drop everything." Ask clarifying questions before disrupting the sprint.

### Track Interrupt Load

If you don't measure interrupts, you can't manage them. Track:
- How many interrupts per sprint
- How many hours consumed
- What percentage of planned work got displaced

If interrupt load is consistently > 30% of capacity, something structural is wrong. Raise it. Use the data.

---

## Common Mistakes

### Prioritizing by who shouts loudest
The squeaky wheel gets the grease — and the team gets whiplash. Priority should come from data and trade-off analysis, not volume. If a stakeholder can't articulate the business impact, it's not urgent.

### Treating all bugs as higher priority than all enhancements
A P4 cosmetic bug is not more important than a revenue-driving feature. Bug vs. enhancement is a *type*, not a *priority*. Severity determines how a bug enters the queue, not a blanket rule.

### Never saying no
A backlog that only grows is not a backlog — it's a junk drawer. Regularly prune. Close items that haven't been touched in 90 days. If they matter, they'll come back.

### Planning too granularly too early
Estimating story points for work that's three months out is theater. You don't have enough information, and the estimate will be wrong. Save detailed planning for the sprint horizon.

### Ignoring tech debt until it's a crisis
Tech debt doesn't announce itself with a production alert — it announces itself when a "simple change" takes two weeks. Allocate 15–20% of capacity to tech debt every quarter. Don't wait for it to become a bug. See Technical Debt for tech debt management strategies.

### Context-switching as a prioritization strategy
If the team is working on five things simultaneously, you haven't prioritized — you've just started everything. Limit work in progress. Two to three active initiatives per team is the sweet spot. More than that and everything slows down.

---

## Related Guides

- See Estimates for estimation methods, decomposition, and communicating estimates.
- See Technical Debt for tech debt classification, tracking, and paydown strategies.

---

## Quick Reference Checklist

- [ ] Quarterly plan exists and fits on one page (themes and bets, not task lists)
- [ ] Sprint backlog is prioritized by value, effort, and cost of delay — not gut feel
- [ ] Every item in the sprint has clear acceptance criteria before it's committed
- [ ] Bugs are classified by severity (P1–P4), not treated as a monolith
- [ ] P1/P2 bugs bypass normal prioritization and get fixed immediately
- [ ] P3/P4 bugs compete with enhancements on equal footing
- [ ] Sprint capacity includes a buffer for interrupts (15–30% depending on team type)
- [ ] Interrupts are classified before reacting — not everything is a fire
- [ ] Interrupt load is tracked and reviewed each sprint
- [ ] Backlog is pruned regularly — items untouched for 90+ days are closed or re-evaluated
- [ ] Work in progress is limited to 2–3 active initiatives per team
- [ ] Tech debt gets 15–20% of quarterly capacity, not zero (see Technical Debt)
- [ ] "No" is a valid and regularly used prioritization outcome
- [ ] XL items are broken down before they're prioritized
- [ ] Cost of delay is explicitly discussed when two items seem equally important
