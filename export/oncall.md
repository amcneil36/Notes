# On-Call Best Practices

Being on-call is one of the highest-leverage activities a software engineer does — and one of the most mismanaged. A well-run on-call rotation catches problems early, distributes knowledge, and builds system resilience. A poorly-run one burns people out, hides systemic issues, and trains everyone to ignore alerts. This guide covers the full lifecycle: preparing for on-call, responding to incidents, handing off cleanly, and learning from what went wrong.

---

## Quick Decision Guide

| Situation | What to Do |
|-----------|-----------|
| You're paged and don't recognize the alert | Check the runbook first, escalate if no runbook exists |
| Alert fires but looks like a false positive | Acknowledge, verify, then fix the alert — never just silence it |
| You're unsure if it's a real incident | Treat it as real until proven otherwise — false negatives cost more than false positives |
| Incident is beyond your ability to resolve | Escalate immediately — escalation is not failure, it's protocol |
| You're handing off mid-incident | Written summary in the incident channel, verbal handoff, explicit "you now own this" |
| You just finished a rough on-call week | Write down what hurt and file tickets — pain without follow-up is wasted suffering |

---

## 1. Rotation Design

A rotation is the foundation. Get it wrong and everything downstream suffers.

### Rotation Length

| Length | When It Works | When It Doesn't |
|--------|--------------|-----------------|
| **1 week** | Most teams, most of the time | Very small teams (< 3 engineers) — too frequent |
| **2 weeks** | Small teams that need longer gaps between rotations | Large teams — too long to stay sharp |
| **24-hour shifts** | Follow-the-sun with multiple time zones | Single-timezone teams — unsustainable |

**Recommendation:** One week, Monday-to-Monday, with handoff during business hours. Never hand off on a Friday afternoon.

### Rotation Equity

Every engineer on the team should be in the rotation. Exempting senior engineers creates knowledge silos. Exempting junior engineers delays their growth. The only valid exemption is someone who genuinely cannot be reached (e.g., international travel with no connectivity).

Track on-call load over time:
- Number of pages per rotation
- Pages outside business hours specifically
- Number of incidents requiring escalation
- Time spent on incident response vs. total on-call hours

If one person consistently gets hit harder than others (bad luck, certain failure-prone days), rebalance manually. "Fair" means equal burden, not equal calendar slots.

### Primary and Secondary

| Role | Responsibility |
|------|---------------|
| **Primary** | First responder. Acknowledges all pages within the SLA. Owns triage and initial response. |
| **Secondary** | Backup. Engaged if primary doesn't acknowledge within the escalation window, or if the incident needs a second pair of hands. |
| **Escalation manager** | Team lead or engineering manager. Engaged for severity-1 incidents or when the on-call engineer needs organizational support (cross-team coordination, customer communication). |

Secondary is not optional for production services. A single point of failure in your on-call rotation is the same anti-pattern you'd reject in your architecture.

---

## 2. Before Your Rotation Starts

The best incident response happens before the incident.

### Pre-Rotation Checklist

- [ ] Review open incidents and ongoing issues from the previous rotation
- [ ] Read the handoff notes from the outgoing on-call engineer
- [ ] Verify your alerting and paging tools are configured and reachable (phone, laptop, app notifications)
- [ ] Test that you can VPN in and access production dashboards from wherever you'll be
- [ ] Review any recent deployments, feature flags, or config changes that landed in the last week
- [ ] Skim the runbook index — you don't need to memorize them, but know what exists
- [ ] Confirm the secondary on-call knows they're secondary

### Know Your System's Failure Modes

You should be able to answer these questions before your rotation starts:

1. What are the top 3 most common alerts, and what do they mean?
2. What are the critical dependencies, and what happens when each one goes down?
3. Where are the dashboards for key business metrics (not just infra metrics)?
4. What was the last major incident, and what was the root cause?
5. Is anything currently degraded or operating on a workaround?

If you can't answer these, your handoff process is broken — fix it.

---

## 3. Alert Design for On-Call

Alerts are the interface between your system and the human who has to respond. Bad alerts are a UX problem.

### The Alert Pyramid

```
        ┌─────────────────┐
        │   PAGE (wake up) │  ← Customer-impacting, requires immediate action
        ├─────────────────┤
        │  TICKET (next day)│  ← Degraded but not broken, fix during business hours
        ├─────────────────┤
        │  DASHBOARD (FYI)  │  ← Informational, visible but no notification
        └─────────────────┘
```

**Every alert must answer three questions:**

1. **What is broken?** — Not "disk usage high" but "disk usage on payment-service-prod-3 at 92%, will fill in ~4 hours at current write rate"
2. **Why does it matter?** — "Payment processing will stop when disk is full"
3. **What should I do?** — Link to the runbook, or inline the first three steps

### Alert Quality Rules

| Rule | Why |
|------|-----|
| Every page must be actionable | If the on-call can't do anything about it, it shouldn't page |
| Every page must require a human | If it can be auto-remediated, automate it and downgrade to a ticket |
| Alert on symptoms, not causes | "Error rate > 5%" beats "CPU > 80%" — CPU might be high and everything's fine |
| Set thresholds from data, not gut | Use p95/p99 of normal operating range plus a margin |
| Alerts should have owners | Every alert maps to a team. Orphan alerts rot. |
| Review alert volume monthly | If you're averaging > 2 pages per on-call day, your alerts need tuning |

### The "2 AM Rule"

Before creating any paging alert, ask: *"Would I wake someone up at 2 AM for this?"*

If the answer is no, it's a ticket or a dashboard metric — not a page. Paging someone unnecessarily at 2 AM erodes trust in the entire alerting system and trains people to ignore pages.

---

## 4. Incident Response

When you get paged, follow this order: **Acknowledge → Assess → Act → Communicate**.

### Step 1: Acknowledge

Acknowledge the page within your SLA (typically 5-15 minutes). Acknowledgment means "I see this and I'm looking at it," not "I've fixed it." If you can't respond, let it escalate — that's what the escalation chain is for.

### Step 2: Assess Severity

| Severity | Definition | Response |
|----------|-----------|----------|
| **SEV-1** | Customer-facing outage, revenue impact, data loss risk | All hands. Open a war room. Notify stakeholders immediately. |
| **SEV-2** | Significant degradation, partial outage, workaround exists | Primary on-call drives. Pull in help as needed. Stakeholder update within 30 min. |
| **SEV-3** | Minor impact, non-critical feature affected | Primary on-call handles. Fix during business hours if possible. |
| **SEV-4** | Cosmetic, no user impact | Log it. Fix in normal sprint work. |

**When in doubt, round up.** A SEV-2 that turns out to be a SEV-3 is fine. A SEV-3 that was actually a SEV-1 is a disaster.

### Step 3: Act

```
1. Check the runbook for this alert
2. Look at recent changes (deployments, config changes, feature flags)
3. Check dependency health (downstream services, databases, caches, queues)
4. Look at the metrics dashboard — is this a spike, a trend, or a step change?
5. If you know the fix → apply it
6. If you don't → escalate and start gathering data for whoever you escalate to
```

**The 15-Minute Rule:** If you've been looking at a problem for 15 minutes and you're not making progress, escalate. Don't spend 90 minutes proving you can't fix it alone. Escalation is a tool, not an admission of defeat.

### Step 4: Communicate

Communication during an incident is not optional — it's part of the response.

| Audience | What They Need | When |
|----------|---------------|------|
| **Your team** | Technical details, what you've tried, what you need | Real-time in incident channel |
| **Stakeholders** (PMs, leadership) | Impact summary, ETA if known, next update time | Every 15-30 min for SEV-1/2 |
| **Customers** (via status page) | Plain-language impact, that you're working on it | As soon as impact is confirmed |

**Template for status updates:**

```
INCIDENT UPDATE — [Service Name] — [SEV level]
Status: [Investigating / Identified / Mitigating / Resolved]
Impact: [What users are experiencing]
Current action: [What we're doing right now]
Next update: [Time]
```

Never say "we're looking into it" without saying when you'll update next. Silence during an incident is worse than bad news.

### The Incident Commander Role

For SEV-1 and complex SEV-2 incidents, one person should own coordination while others debug. This is the Incident Commander (IC). The IC does NOT fix the problem — they run the response.

| Responsibility | What It Looks Like |
|---------------|-------------------|
| **Own the timeline** | "We started investigating at 02:34. We identified root cause at 03:15. We're rolling back now." |
| **Coordinate responders** | "Alice, check the database. Bob, check the payment provider. Report back in 10 minutes." |
| **Manage communication** | Send stakeholder updates on schedule. Update the status page. |
| **Make decisions** | "We're rolling back. We're not going to keep debugging in production." |
| **Prevent chaos** | Keep the war room focused. Shut down tangents. Make sure people aren't duplicating effort. |

**IC is not the most senior engineer.** It's whoever is best at organizing under pressure. Rotate the role so more people develop the skill.

**IC anti-pattern:** The IC starts debugging. Now nobody is coordinating, stakeholders aren't getting updates, and three engineers are investigating the same thing independently. If the IC starts looking at dashboards, someone else needs to take over coordination.

### War Room Management

A war room (physical or virtual) is where the incident gets resolved. Without structure, it devolves into a group chat of people sharing theories simultaneously.

**War room rules:**

1. **One channel, one thread.** All incident communication in a single, dedicated channel. Not DMs, not multiple threads, not "I'll share in standup tomorrow."

2. **Announce what you're doing.** Before investigating anything, post what you're checking. "I'm looking at database connection pool metrics." This prevents duplication and creates a searchable timeline.

3. **Report findings, not theories.** "Connection pool is at 100% utilization, up from 40% baseline" is useful. "I think it might be a connection leak" is noise until backed by data.

4. **IC controls the room.** Only the IC assigns tasks and makes decisions. Others surface information. This isn't about hierarchy — it's about preventing thrash.

5. **Timebox investigations.** "Check the payment provider logs — report back in 10 minutes whether you find anything or not." No-finding reports are just as valuable as positive findings.

6. **Document in real time.** Paste timestamps, commands run, screenshots of dashboards, decisions made and why. The postmortem timeline will thank you.

---

## 5. Runbooks

A runbook is a checklist for incident response. It turns tribal knowledge into repeatable process.

### What Makes a Good Runbook

| Element | Example |
|---------|---------|
| **Title** | "Payment Service — High Error Rate" |
| **Alert this maps to** | `payment-svc-error-rate-high` |
| **Symptoms** | Error rate > 5% on /checkout endpoint, users seeing 500 errors |
| **Likely causes** | (1) Database connection pool exhaustion, (2) Downstream payment provider outage, (3) Bad deployment |
| **Diagnostic steps** | Step-by-step with exact commands, dashboard links, and what to look for |
| **Remediation steps** | Step-by-step fix for each likely cause |
| **Escalation path** | Who to call if this doesn't work |
| **Last updated** | Date and author — stale runbooks are dangerous |

### Runbook Anti-Patterns

- **"Ask Bob"** — If the runbook says "ask Bob," it's not a runbook, it's a contact card. Bob's knowledge needs to be in the document.
- **Outdated commands** — A runbook with wrong CLI commands is worse than no runbook. Review quarterly.
- **Novel-length prose** — A runbook is a checklist, not a design doc. Use numbered steps, not paragraphs.
- **Missing "verify" steps** — Every remediation should end with "verify the fix by checking [metric/endpoint/dashboard]."

### When to Write a New Runbook

After every incident where the on-call engineer had to figure something out from scratch, a runbook should be created or updated. This is the single highest-ROI post-incident action. Make it a required output of every postmortem.

---

## 6. Escalation

Escalation is the most important skill an on-call engineer has — and the most underused.

### When to Escalate

| Trigger | Action |
|---------|--------|
| 15 minutes with no progress | Escalate to secondary or subject matter expert |
| SEV-1 incident | Escalate immediately, regardless of your confidence level |
| Impact is growing | Escalate before it gets worse, not after |
| You need cross-team coordination | Escalate to engineering management for organizational support |
| You're unsure of the blast radius | Escalate — uncertainty about scope is itself a signal |

### How to Escalate Well

A bad escalation: *"Hey, payment service is broken, can you look?"*

A good escalation:

```
What's happening: Payment service error rate spiked to 15% at 02:34 UTC
Impact: ~200 failed transactions/min, customers seeing checkout failures
What I've checked:
  - Recent deploys: none in last 6 hours
  - Database: connections normal, query latency normal
  - Payment provider status page: no reported issues
What I haven't checked:
  - Network layer between us and payment provider
  - Whether the error pattern matches a specific payment type
What I need: Someone with network/infra access or payment provider expertise
```

The goal is to hand off context, not just hand off the problem. Every minute the next person spends re-discovering what you already know is wasted.

---

## 7. Handoffs

A handoff is where institutional knowledge goes to die — unless you're deliberate about it.

### End-of-Rotation Handoff Template

```
## On-Call Handoff — [Date Range]

### Active Issues
- [Issue 1]: Status, what's been done, what's left
- [Issue 2]: Status, link to incident channel

### Resolved Incidents
- [Incident 1]: Brief summary, root cause, link to postmortem
- [Incident 2]: Brief summary, action items still open

### Things to Watch
- [Deployment X landing Tuesday — has had issues in staging]
- [Database migration running — monitor replication lag]
- [Vendor Y maintenance window Thursday 2-4 AM UTC]

### Alert Noise
- [Alert A fired 12 times, all false positives — ticket filed to fix threshold]
- [Alert B is new, seems misconfigured — needs tuning]

### Action Items from This Rotation
- [ ] Fix Alert A threshold (ticket: LINK)
- [ ] Write runbook for [scenario] (ticket: LINK)
- [ ] Follow up on [vendor issue] (ticket: LINK)
```

### Handoff Rules

1. **Written first, verbal second.** The document is the source of truth. The verbal handoff adds color and answers questions.
2. **Handoff during business hours.** Never at 5 PM Friday. Monday morning or early afternoon is ideal.
3. **Both people present.** Async handoffs (just dropping a doc in Slack) lose nuance. 15 minutes of face-time is worth it.
4. **Outgoing on-call owns the handoff.** It's the outgoing person's job to make the incoming person successful — not the incoming person's job to extract information.

---

## 8. Postmortems

Postmortems are how you turn incidents into improvements. Skip them and you'll keep having the same incidents.

### When to Write a Postmortem

| Trigger | Required? |
|---------|-----------|
| SEV-1 incident | Always |
| SEV-2 incident | Always |
| SEV-3 with interesting root cause | Recommended |
| Near-miss (almost became an incident) | Recommended — these are gold |
| Same alert fired 3+ times in one rotation | Required — this is a systemic problem |

### Postmortem Structure

```
## Postmortem — [Incident Title]
Date: [Date]
Duration: [Start → End]
Severity: [SEV-X]
Authors: [Names]

### Summary
[2-3 sentences: what happened, impact, resolution]

### Timeline
[Chronological events with timestamps]
- 02:34 — Alert fired: payment-error-rate-high
- 02:37 — On-call acknowledged, began investigation
- 02:52 — Escalated to payments team
- 03:15 — Root cause identified: connection pool leak after deploy v2.4.1
- 03:22 — Rolled back to v2.4.0
- 03:28 — Error rate returned to normal

### Root Cause
[Technical explanation of what went wrong and why]

### Impact
[Numbers: users affected, transactions failed, revenue impact, duration]

### What Went Well
[Things that worked — detection speed, response, tooling]

### What Went Poorly
[Things that didn't — detection gaps, communication, missing runbooks]

### Action Items
| Action | Owner | Priority | Ticket |
|--------|-------|----------|--------|
| Fix connection pool leak | @engineer | P1 | LINK |
| Add runbook for payment rollback | @on-call | P2 | LINK |
| Add alert for connection pool exhaustion | @infra | P2 | LINK |
```

### Postmortem Rules

1. **Blameless.** Focus on systems and processes, not individuals. "The deploy process allowed a broken build to reach production" — not "Bob deployed broken code."
2. **Action items have owners and deadlines.** An action item without an owner is a wish. Track them to completion.
3. **Share broadly.** Postmortems only create value if people read them. Post to a shared channel, present at team meetings.
4. **Follow up.** Review action items 30 days later. If they're not done, either do them or explicitly decide they're not worth doing.

### Post-Incident Follow-Up Tracking

The postmortem is the easy part. The hard part is making sure the action items actually get done. Most teams write great postmortems and then let the action items rot.

**Follow-up process:**

1. **Action items become tickets within 24 hours.** Not "we should probably file a ticket." File it in the postmortem meeting, link it in the postmortem doc.

2. **Weekly review for the first 30 days.** Someone (engineering manager, IC, or a rotating role) reviews all open postmortem action items weekly. This should take 5 minutes in standup.

3. **30-day closure review.** If an action item isn't done after 30 days, it gets escalated or explicitly deprioritized. "We decided this isn't worth doing" is a valid outcome — "it fell off the radar" is not.

4. **Track completion rate as a team metric.** If your postmortem action item completion rate is below 80%, your postmortems are theater. You're going through the motions without improving.

| Action Item State | What It Means |
|-------------------|---------------|
| Filed (with ticket) | Owned and tracked |
| In progress | Actively being worked on |
| Completed | Fix deployed and verified |
| Won't fix | Explicitly deprioritized with documented reasoning |
| Stale (> 30 days, no update) | **Problem.** Escalate or close. |

### Incident Metrics

Track these across all incidents to spot systemic patterns. Individual incidents teach you about specific failures. Trends teach you about your system's reliability posture.

| Metric | What It Tells You | Act When |
|--------|-------------------|----------|
| **MTTD** (Mean Time to Detect) | How long between the problem starting and someone knowing about it | > 10 minutes — your monitoring/alerting has gaps |
| **MTTA** (Mean Time to Acknowledge) | How long between the alert firing and the on-call responding | > 15 minutes — paging configuration or response discipline issue |
| **MTTR** (Mean Time to Resolve) | How long between detection and resolution | Trending upward — system is getting more complex or runbooks are lacking |
| **Incident frequency** | How many incidents per week/month | Increasing — reliability is degrading, invest in prevention |
| **Incidents per service/module** | Which parts of the system generate the most incidents | Top 3 repeat offenders need focused engineering investment |
| **Incidents caused by deploys** | What % of incidents follow a deployment | > 20% — your testing, canary, or rollback process is insufficient |
| **Escalation rate** | What % of incidents require escalation beyond primary on-call | > 50% — training gap, runbook gap, or system complexity issue |
| **Postmortem action item completion rate** | What % of action items are completed within 30 days | < 80% — you're learning but not improving |
| **Repeat incidents** | Same root cause appearing in multiple incidents | Any repeat — the previous fix was incomplete or the action item wasn't done |

**Review cadence:** Monthly for the team, quarterly for leadership. Present trends, not just raw numbers. "MTTR increased 40% this quarter because we took on three new services without updating runbooks" is actionable. "We had 12 incidents" is not.

---

## 9. Toil Reduction

On-call toil is any repetitive, manual, automatable work that scales with system size. Left unchecked, it consumes all your on-call hours and crowds out real incident response.

### Identifying Toil

| Signal | Example | Fix |
|--------|---------|-----|
| Same alert fires every week | Disk cleanup alert on log volume | Automate log rotation, raise threshold |
| Manual remediation every time | Restart service X when it leaks memory | Fix the memory leak; interim: auto-restart with health checks |
| Routine tasks during on-call | "Every Monday, manually run the batch reconciliation" | Automate via cron/scheduler |
| Alerts with known-safe resolutions | Cache miss spike after deploy (self-resolves in 10 min) | Add delay/auto-resolve to the alert |

### The 50% Rule

Google's SRE book recommends that on-call engineers spend no more than 50% of their on-call time on toil. The other 50% should go to improving the on-call experience — writing runbooks, tuning alerts, automating remediation, filing bugs against noisy systems.

If your on-call engineers are spending 100% of their time firefighting, you don't have an on-call problem — you have a reliability problem. Escalate to leadership.

---

## Common Pitfalls

| Pitfall | Why It Hurts | Fix |
|---------|-------------|-----|
| **Alert fatigue** | Too many pages desensitize on-call to real problems. People start ignoring alerts. | Enforce the 2 AM rule. Every page must be actionable and require a human. Target < 2 pages/day. |
| **Hero culture** | One person handles everything, others never learn. Single point of failure for knowledge. | Enforce rotation equity. Pair junior and senior on-call. Document tribal knowledge in runbooks. |
| **Poor handoffs** | Context lost between rotations. Incoming on-call starts from zero. | Written handoff template, verbal walkthrough, overlap during business hours. |
| **No runbooks** | Every incident is solved from scratch. Response time balloons. Knowledge lives in people's heads. | Require a runbook as an output of every postmortem. Review quarterly. |
| **Escalation shame** | Engineers sit on problems too long because escalating "feels like failure." | Normalize escalation. Praise fast escalation in retros. The 15-minute rule. |
| **Postmortem theater** | Postmortems happen but action items never get done. Same incidents recur. | Track action items in your ticketing system. Review at 30 days. |
| **Siloed on-call** | Only the "ops team" or "senior engineers" go on-call. Rest of team detached from production. | Everyone who writes code for production should be in the rotation. |
| **No SLA for acknowledgment** | Pages go unacknowledged for hours. Escalation chains don't trigger. | Set explicit ack SLA (5-15 min). Auto-escalate on miss. |
| **On-call as punishment** | On-call is seen as a burden to endure, not a responsibility to own. | Invest in tooling, reduce toil, give comp time. Make on-call sustainable. |

---

## On-Call Maturity Model

Use this to assess where your team is and what to work on next.

| Level | Rotation | Alerts | Runbooks | Incidents | Postmortems |
|-------|----------|--------|----------|-----------|-------------|
| **1 — Reactive** | Ad-hoc, whoever's around | No tuning, high noise | None or outdated | Heroic individual effort | None |
| **2 — Structured** | Defined schedule, primary on-call | Some tuning, mostly actionable | Exist for top alerts | Defined severity levels, escalation path | Written for SEV-1 |
| **3 — Proactive** | Primary + secondary, fair distribution | All pages actionable, reviewed monthly | Cover all paging alerts, reviewed quarterly | Structured response with communication templates | Written for SEV-1/2, action items tracked |
| **4 — Optimized** | Data-driven load balancing, comp time | Auto-remediation for known issues, near-zero noise | Living documents, updated after every incident | Incident commander rotation, cross-team coordination playbooks | Blameless culture, action items completed within 30 days |

Most teams should aim for Level 3. Level 4 is aspirational and worth pursuing for critical-path services.

---

## Quick Reference Checklist

### Setting Up On-Call for a New Service

- [ ] Define the rotation (primary + secondary, 1-week cycles)
- [ ] Set up paging with escalation chains and ack SLAs
- [ ] Create dashboards for key business and system metrics
- [ ] Write runbooks for every paging alert
- [ ] Define severity levels and response expectations
- [ ] Create a handoff template and schedule handoffs during business hours
- [ ] Establish a postmortem process with action item tracking
- [ ] Baseline your alert volume — set a target of < 2 pages/day

### During Your Rotation

- [ ] Acknowledge pages within SLA
- [ ] Follow runbooks — update them if they're wrong
- [ ] Escalate at 15 minutes if stuck
- [ ] Communicate during incidents (status updates, stakeholder notifications)
- [ ] Log every page and its resolution for handoff notes
- [ ] File tickets for alert noise, missing runbooks, and toil

### End of Rotation

- [ ] Write handoff notes using the template
- [ ] Do a verbal handoff with the incoming on-call
- [ ] File postmortems for any SEV-1/2 incidents
- [ ] File tickets for toil reduction and alert tuning
- [ ] Update runbooks based on anything you learned

---

## Related Guides

- See Alerting for alert design principles and monitoring strategy.
- See Documentation for broader documentation strategy and where to store runbooks and postmortems.

---

## Further Reading

- *Site Reliability Engineering* (Google) — Chapters on On-Call, Incident Response, and Postmortems
- *The Practice of Cloud System Administration* (Limoncelli, Chalup, Hogan) — Chapter on On-Call
- *Incident Management for Operations* (O'Reilly) — End-to-end incident lifecycle
- PagerDuty Incident Response Documentation — Open-source incident response process guide
