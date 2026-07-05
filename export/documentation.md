# Documentation Best Practices

Good documentation is the difference between a team that scales and a team that drowns in chat questions. Bad documentation — stale, scattered, unfindable — is arguably worse than no documentation at all, because it actively misleads. The goal is documentation that is **accurate, discoverable, and owned** — and that means being deliberate about what gets documented, where it lives, and who keeps it alive.

---

## The Four Failure Modes of Documentation

Before discussing what to do, it helps to name what goes wrong. Nearly every documentation problem falls into one of these buckets:

| Failure Mode | Symptoms | Root Cause |
|---|---|---|
| **Stale docs** | Instructions don't match reality; new hires follow docs and break things | No ownership, no review cadence, no coupling to code changes |
| **Undiscoverable docs** | "I know we documented this somewhere…" followed by 20 minutes of searching | No consistent structure, no naming conventions, no index |
| **Missing docs** | Tribal knowledge locked in one person's head; onboarding takes weeks | No culture of documentation; no clear expectations of what to document |
| **Wrong-level docs** | Team-specific decisions buried in org-wide pages, or org standards scattered across team spaces | No hierarchy model; no guidance on where things belong |

A good documentation strategy addresses all four simultaneously. Fixing one without the others just moves the pain around.

---

## What Deserves Documentation (and What Doesn't)

Not everything needs a doc. Over-documenting creates noise that makes the important stuff harder to find. Under-documenting creates tribal knowledge silos. The key is identifying what has **lasting value to someone other than the author**.

### Document This

| Category | Why | Example |
|---|---|---|
| **Architecture decisions** | They outlive sprints and explain *why* the system looks the way it does | ADRs, tech design docs, system diagrams |
| **Onboarding procedures** | Every new hire hits the same friction; documenting it once saves weeks per person | Local dev setup, environment access, team norms |
| **Runbooks and operational procedures** | On-call engineers need answers at 2 AM, not a chat thread from 6 months ago. See On-Call for runbook best practices. | Incident response, deployment rollback, common alert remediation |
| **API contracts and integration points** | Consumers of your service shouldn't need to read your source code | REST/gRPC contracts, event schemas, SLA expectations |
| **Cross-team agreements** | Verbal agreements rot; written ones survive reorgs | SLAs, data ownership, shared library responsibilities |
| **Non-obvious decisions** | "Why did we do it this way?" is the most expensive question in engineering | Trade-off analysis, rejected alternatives, constraint explanations |

### Don't Document This

| Category | Why Not | What to Do Instead |
|---|---|---|
| **Things the code already says** | Duplicating what's readable in source creates drift | Write readable code; use meaningful names |
| **Temporary workarounds** | They become permanent if documented formally | Track in your issue tracker with an expiration date |
| **Step-by-step for trivial tasks** | Over-documenting insults the reader and creates maintenance burden | Trust your teammates' competence |
| **Meeting notes that aren't decisions** | Nobody re-reads "we discussed X"; only decisions matter | Extract decisions into their proper doc; discard the rest |
| **Anything with a shelf life under 2 weeks** | It'll be wrong before anyone reads it | Use chat or issue tracker comments for ephemeral context |

### The Litmus Test

Before writing a doc, ask:

1. **Will someone other than me need this information?** If no, it's a personal note, not documentation.
2. **Will this still be true in 3 months?** If no, it probably belongs in a ticket or chat thread.
3. **Is this answering a question that's been asked more than once?** If yes, document it immediately.
4. **Would a new team member need this on their first week?** If yes, it's onboarding material and belongs in a discoverable place.

---

## Where Documentation Lives: The Hierarchy Model

One of the most common documentation problems is everything living in the same flat space. A team runbook sits next to a company-wide security policy. An org architecture diagram is mixed in with sprint retrospective notes. The fix is a clear hierarchy with explicit rules about what goes where.

### The Three Tiers

```
┌─────────────────────────────────────────────┐
│              COMPANY / PLATFORM             │
│  Standards, policies, shared tooling docs   │
│  Owner: Platform teams, Staff+ engineers    │
│  Cadence: Quarterly review                  │
├─────────────────────────────────────────────┤
│              ORG / DOMAIN                   │
│  Architecture, cross-team contracts, ADRs   │
│  Owner: Org leads, principal engineers      │
│  Cadence: Monthly review                    │
├─────────────────────────────────────────────┤
│              TEAM                           │
│  Runbooks, onboarding, local processes      │
│  Owner: Individual teams                    │
│  Cadence: Sprint-level review               │
└─────────────────────────────────────────────┘
```

### What Goes Where

| Document Type | Tier | Platform | Rationale |
|---|---|---|---|
| Coding standards, style guides | Company | Wiki | Applies to everyone; changes rarely |
| Security policies, compliance | Company | Wiki | Auditable, centralized, version-controlled |
| Shared library usage guides | Company | Repo README + Wiki | Code-adjacent docs in repo, overview in wiki |
| Domain architecture diagrams | Org | Wiki | Cross-team visibility; not tied to a single repo |
| ADRs (Architecture Decision Records) | Org or Team | Repo (`docs/adr/`) | Tied to the codebase they affect; versioned with code |
| Cross-team API contracts | Org | Repo (contract repo or producer repo) | Machine-readable, versioned, testable |
| Service-specific runbooks | Team | Repo (`docs/runbooks/`) or Wiki | On-call needs fast access; keep close to the service. See On-Call for runbook best practices. |
| Onboarding guide | Team | Wiki | Frequently updated; benefits from rich formatting |
| Local dev setup | Team | Repo (`README.md` or `docs/setup.md`) | Must live with the code it describes |
| Incident postmortems | Team or Org | Wiki | Searchable, cross-referenced, accessible to leadership. See On-Call for postmortem practices. |
| Sprint processes, team norms | Team | Wiki | Team-specific; doesn't belong in code |

### The Golden Rule of Placement

**Code-coupled documentation lives in the repo. Everything else lives in the team wiki.**

If a doc describes how to build, run, test, deploy, or configure a specific service — it belongs in that service's repository. If it describes processes, decisions, architecture, or agreements that span multiple services or involve non-engineers — it belongs in the wiki.

This rule alone eliminates 80% of "where should I put this?" questions.

---

## Wiki vs. Repo: Playing to Each Platform's Strengths

Both platforms have strengths. Using them wrong is why documentation rots.

| Dimension | Wiki | Repo (Markdown) |
|---|---|---|
| **Best for** | Process docs, architecture overviews, onboarding, postmortems | Setup guides, ADRs, runbooks, API docs, inline code docs |
| **Versioning** | Page history (clunky, rarely used) | Git history (precise, reviewable, diffable) |
| **Review process** | None built-in; changes go live immediately | PR-based review; changes are gated |
| **Discoverability** | Search + page tree + labels | File browser + grep; harder for non-engineers |
| **Staleness risk** | High — no forcing function to update | Lower — changes can be coupled to code PRs |
| **Audience** | Engineers, PMs, leadership, stakeholders | Engineers primarily |
| **Rich formatting** | Tables, diagrams, macros, embedded content | Markdown (simpler, but portable) |

### Linking the Two

The worst outcome is having the same information in both places. Instead, create **pointers**:

- Repo README should link to the wiki space for the team/service
- Wiki service pages should link to the repo's `docs/` directory
- ADRs in the repo should be indexed on a wiki page with a summary table
- Runbooks can live in either place but must be linked from the other

---

## Wiki Space Structure

A consistent wiki structure across teams makes it possible for anyone in the org to find anything in any team's space. The structure below works well for most engineering teams:

```
Team Space Root
├── 🏠 Home (dashboard with links to key pages)
├── 📋 Onboarding
│   ├── New Hire Checklist
│   ├── Environment Setup (→ link to repo)
│   ├── Key Contacts & Escalation Paths
│   └── Team Norms & Working Agreements
├── 🏗️ Architecture
│   ├── System Overview & Diagrams
│   ├── ADR Index (→ links to repo ADRs)
│   └── Integration Points
├── 📖 Runbooks
│   ├── Incident Response
│   ├── Deployment Procedures (→ link to repo)
│   └── Common Alert Remediation
├── 📊 Processes
│   ├── Sprint Ceremonies
│   ├── On-Call Rotation
│   └── Release Process
└── 📝 Postmortems
    └── [Chronological list]
```

### Naming Conventions

Consistent naming is the cheapest discoverability improvement you can make:

| Convention | Example | Why |
|---|---|---|
| Prefix with service name | `[order-service] Deployment Runbook` | Disambiguates in search results |
| Date-prefix postmortems | `2024-03-15 — Cart Timeout Incident` | Chronological ordering without manual sorting |
| Use labels consistently | `runbook`, `adr`, `onboarding`, `architecture` | Enables cross-space search and filtering |
| Avoid generic titles | ~~"Notes"~~, ~~"Docs"~~, ~~"Info"~~ → `Order Service Architecture Overview` | Generic titles are unsearchable and undiscoverable |

---

## Repository Documentation Structure

For in-repo documentation, consistency across repos matters just as much as it does in the wiki:

```
repo-root/
├── README.md              # What this service does, how to run it, links out
├── CONTRIBUTING.md        # How to contribute, PR process, code standards
├── docs/
│   ├── setup.md           # Detailed local dev setup
│   ├── architecture.md    # Service-level architecture & design
│   ├── runbooks/
│   │   ├── deployment.md
│   │   └── rollback.md
│   └── adr/
│       ├── 0001-use-kafka-for-events.md
│       └── 0002-switch-to-grpc.md
└── api/
    └── openapi.yaml       # Or protobuf definitions
```

### The README Is Your Front Door

The README is the single most important document in any repository. It should answer five questions in under 2 minutes of reading:

1. **What is this?** — One paragraph, no jargon.
2. **How do I run it locally?** — Commands, not prose.
3. **How do I test it?** — `mvn test`, `npm test`, whatever applies.
4. **How do I deploy it?** — Link to the pipeline or deployment doc.
5. **Where do I go for more?** — Links to wiki, runbooks, architecture docs.

If your README doesn't answer these five questions, it's incomplete.

---

## Keeping Documentation Fresh: The Staleness Problem

Stale documentation is the #1 reason teams stop trusting (and stop writing) docs. Every strategy for freshness boils down to one principle: **couple documentation updates to the workflows that change the thing being documented**.

### Strategies That Work

| Strategy | How It Works | Effort | Effectiveness |
|---|---|---|---|
| **PR-coupled updates** | Require doc updates in PRs that change behavior | Low | High |
| **Onboarding-driven audits** | New hires flag every doc that's wrong during onboarding | Free | High |
| **Quarterly doc review** | Schedule a recurring task to review key docs | Medium | Medium |
| **Ownership labels** | Every wiki page has an explicit owner who reviews it | Low | High |
| **Expiration dates** | Mark docs with a "review by" date; stale ones get flagged | Low | Medium |
| **Doc-as-code in PRs** | Docs live in the repo; changes are reviewed like code | Low | High |

### Strategies That Don't Work

| Strategy | Why It Fails |
|---|---|
| "Everyone is responsible" | Nobody is responsible |
| Annual documentation sprints | You can't fix 12 months of drift in a week |
| Wiki gardening volunteers | Goodwill burns out; it's unrecognized work |
| Automated "this page is old" banners | People learn to ignore them |

### The PR Checklist Approach

The highest-ROI freshness strategy is adding a single line to your PR template:

```markdown
## PR Checklist
- [ ] If this PR changes behavior, relevant docs have been updated
```

This doesn't require tooling. It doesn't require a process change. It just makes documentation a visible part of the definition of done. Reviewers who see an unchecked box will ask about it.

### Ownership Model

Every document should have an explicit owner. This doesn't mean one person writes everything — it means one person (or team) is accountable for accuracy.

| Document Type | Owner | Review Cadence |
|---|---|---|
| Team runbooks | On-call rotation lead | Every sprint |
| Onboarding guide | Most recent new hire + manager | Every new hire (they update as they go) |
| Architecture docs | Tech lead | Quarterly |
| ADRs | Original author | Never (they're immutable point-in-time records) |
| API contracts | Producing team | Every breaking change |
| Org-level standards | Staff/Principal engineers | Quarterly |

---

## Discoverability: Making Docs Findable

Writing good docs is useless if nobody can find them. Discoverability requires intentional structure, not just good search.

### The Three Ways People Find Docs

| Method | When It's Used | How to Optimize |
|---|---|---|
| **Browse** | "I'm new, show me everything" | Consistent space structure, clear page tree, index pages |
| **Search** | "I know what I'm looking for" | Good titles, labels, keywords in the first paragraph |
| **Ask someone** | "I give up looking" (this is a failure state) | Reduce this by making browse and search actually work |

### Discoverability Checklist

1. **Index pages** — Every wiki space and every repo `docs/` folder should have an index page listing all documents with one-line descriptions.
2. **Consistent labels** — Use a standard set of labels across all team spaces: `runbook`, `adr`, `onboarding`, `architecture`, `api`, `postmortem`.
3. **Descriptive titles** — The title should be a complete thought. `[order-service] Kafka Consumer Runbook` beats `Kafka Stuff`.
4. **First-paragraph summary** — The first paragraph of every doc should answer "what is this about and who is it for?" Search results show the first ~150 characters.
5. **Cross-linking** — Every doc should link to related docs. An architecture doc should link to ADRs. A runbook should link to the alert it addresses. Isolated docs are lost docs.
6. **Single source of truth** — Never duplicate content across pages. Link to the canonical source. If you find yourself copying, you're creating a future inconsistency.

---

## Building a Documentation Culture

You can't process your way to good documentation. If the team doesn't value writing, no amount of templates or checklists will fix it. But culture can be nudged.

### What Works

| Tactic | Why It Works |
|---|---|
| **Leaders write docs** | If the tech lead writes docs, the team writes docs. Culture flows downhill. |
| **Celebrate good docs** | Call out great documentation in retros and reviews. Make it visible work. |
| **Make docs part of the definition of done** | A feature isn't done until it's documented. No exceptions. |
| **Onboarding exposes gaps** | New hires are the best auditors. Make "update the doc" part of onboarding. |
| **Low friction tooling** | If writing a doc requires 6 clicks and a template, people won't do it. |
| **Blameless correction** | When someone finds a stale doc, the response should be "thanks for fixing it" not "who let this get stale?" |

### What Doesn't Work

| Tactic | Why It Fails |
|---|---|
| Mandating doc word counts | People pad with filler; quality drops |
| Requiring docs for every PR | Most PRs don't change behavior; this creates checkbox fatigue |
| Centralizing all writing to one person | Single point of failure; that person burns out or leaves |
| Treating docs as a separate workstream | Documentation should be part of the work, not something done after it |

---

## Source Code Documentation

Code documentation serves a different purpose than wiki documentation. It answers "what does this specific piece of code do and why?" rather than "how does the system work?"

### When to Comment Code

| Situation | Comment? | Example |
|---|---|---|
| The *why* behind a non-obvious decision | Yes | `// Using insertion sort here because n < 20 in practice and it avoids allocation` |
| Complex business logic | Yes | `// Tax exemption applies only to reseller accounts with valid certificates` |
| Workarounds for known bugs | Yes | `// Workaround for KAFKA-12345: consumer rebalance during shutdown` |
| Public API methods | Yes | Javadoc/JSDoc with param descriptions and return values |
| What the code literally does | No | ~~`// increment counter`~~ `counter++` |
| Obvious control flow | No | ~~`// check if null`~~ `if (user == null)` |

### Code Doc Standards (Brief)

- **Public APIs**: Always document with Javadoc/JSDoc. Include parameter descriptions, return values, thrown exceptions, and a one-line summary.
- **Internal methods**: Document only when the *why* isn't obvious from the code.
- **Classes/modules**: A one-paragraph header explaining the responsibility and key collaborators.
- **Constants and config values**: Document the meaning and valid ranges, not just the name.
- **README over comments**: If you need more than 5 lines of comments to explain something, it belongs in a `docs/` file, not inline.

---

## Common Mistakes

1. **Writing docs nobody asked for.** If nobody has ever asked the question your doc answers, you're writing for an audience that doesn't exist. Wait for the question to be asked twice, then document.

2. **Documenting how, not why.** Code shows *how*. Docs should explain *why* — the context, constraints, trade-offs, and rejected alternatives that led to the current state.

3. **Treating all docs as permanent.** Some docs are point-in-time (postmortems, ADRs). Others are living documents (runbooks, setup guides). Handle them differently. ADRs are never updated — they're superseded by new ADRs. Runbooks are updated constantly.

4. **No ownership = no maintenance.** A doc without an owner is a doc that will be wrong within 6 months. If you can't assign an owner, question whether the doc should exist.

5. **Duplicating content across locations.** The moment the same information lives in two places, they will diverge. One will be wrong. Link, don't copy.

6. **Burying the lede.** The most important information should be in the first two paragraphs. If someone has to scroll past a table of contents, a revision history, and three paragraphs of context before getting to the actual content — the doc has failed.

7. **Writing for the expert audience only.** The people who need docs the most are the ones who know the least. Write for the person who joined the team last week, not the person who built the system.

8. **Conflating documentation with communication.** A chat message is communication. An issue tracker comment is communication. Documentation is a persistent, discoverable artifact that outlives the conversation. Don't treat chat threads as documentation, and don't use docs when a chat message would suffice.

9. **Ignoring the search experience.** If your doc title is "Notes" and the first paragraph is "This page contains various information about our service" — search will never surface it. Write titles and intros that contain the keywords someone would actually search for.

10. **Perfectionism as procrastination.** A rough doc that exists is infinitely more valuable than a polished doc that doesn't. Write it ugly, publish it, and refine later. The biggest enemy of documentation isn't bad writing — it's no writing.

---

## Related Guides

- See On-Call for runbook best practices and postmortem processes.

---

## Quick Reference Checklist

### Before Writing a Doc
- [ ] Has this question been asked more than once?
- [ ] Will this information be valid in 3+ months?
- [ ] Is there an existing doc I should update instead of creating a new one?
- [ ] Do I know which tier (Company / Org / Team) this belongs to?
- [ ] Do I know which platform (wiki / repo) this belongs on?

### While Writing
- [ ] Title is descriptive and searchable (includes service name if applicable)
- [ ] First paragraph summarizes what the doc covers and who it's for
- [ ] Content explains *why*, not just *how*
- [ ] Links to related docs instead of duplicating content
- [ ] Code examples are included where they'd clarify
- [ ] Written for someone who joined the team last week

### After Publishing
- [ ] Explicit owner is assigned
- [ ] Appropriate labels are applied (wiki) or the doc is in the right directory (repo)
- [ ] Index page is updated to include the new doc
- [ ] Cross-links are added from related docs
- [ ] Review cadence is set (if it's a living document)

### Ongoing Maintenance
- [ ] PR template includes "docs updated?" checkbox
- [ ] New hires flag stale docs during onboarding
- [ ] Quarterly review is scheduled for architecture and org-level docs
- [ ] Ownership is transferred when people change teams
