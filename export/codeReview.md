# Code Review Best Practices

Code review is the single highest-leverage quality practice a team can adopt. Done well, it catches bugs before they reach production, spreads knowledge across the team, and raises the bar on code quality over time. Done poorly — rubber-stamp approvals, nitpick wars, week-long review cycles — it becomes a bottleneck that breeds resentment and ships bugs anyway.

This guide covers both sides of the table: how to review code effectively, and how to submit code that's easy to review.

---

## The Goal of Code Review — What You're Actually Optimizing For

Code review is not a gate. It's a conversation. The goal is not to prove the author wrong or to demonstrate your own cleverness. It's to ensure:

1. **Correctness** — Does this code do what it claims to do?
2. **Clarity** — Will the next engineer understand this in 6 months?
3. **Safety** — Does this introduce security, performance, or reliability risks?
4. **Consistency** — Does this follow the team's conventions?
5. **Knowledge sharing** — Does reviewing this make the team collectively smarter?

If your review process doesn't serve at least three of these, it's theater.

---

## How Many Approvals — The Right Number

| Team Size | Recommended Approvals | Why |
|---|---|---|
| 2-3 engineers | 1 approval | Small teams can't afford to block on multiple reviewers |
| 4-8 engineers | 1-2 approvals | One domain expert + one generalist is the sweet spot |
| 8+ engineers | 2 approvals | At scale, you need redundancy — one reviewer will miss things |
| Critical path (auth, payments, data migrations) | 2 approvals minimum | High-blast-radius changes warrant the extra eyes regardless of team size |

**Rules of thumb:**

- One approval is almost always sufficient for everyday changes. Two approvals should be the exception, not the default.
- If your team requires 3+ approvals to merge, you have a trust problem, not a quality problem. Fix the trust problem.
- The person who approves should be someone who could *maintain* this code tomorrow. Don't collect approvals from people who won't feel the consequences.

---

## Reviewer Selection — Who Should Review What

Not all reviewers are equal. A review from someone unfamiliar with the domain is better than no review, but worse than a review from someone who owns that area.

| Reviewer Type | Best For | Watch Out For |
|---|---|---|
| **Domain owner** | Business logic, data model changes, API contracts | May be too close to the code — misses forest for trees |
| **Recent contributor** | Incremental changes to active areas | Knows the context but may share the author's blind spots |
| **Outsider / rotating reviewer** | Architecture changes, new patterns, onboarding | May lack context — pair with a domain owner |
| **Tech lead / senior engineer** | Cross-cutting concerns, design decisions | Don't make them a bottleneck — rotate this responsibility |

**Anti-pattern:** Routing every PR to the same senior engineer. This creates a single point of failure, burns them out, and prevents the team from developing review skills.

---

## The Reviewer's Playbook — How to Review Code

### Step 1: Understand the Intent Before Reading Code

Read the PR description, linked ticket, or design doc *first*. If the PR doesn't have a description, that's your first comment — ask for one. You can't evaluate code without knowing what problem it's solving.

### Step 2: Review in Layers

Don't try to catch everything in one pass. Review in order of importance:

| Pass | Focus | Questions to Ask |
|---|---|---|
| **1. Design** | Architecture, approach, data flow | Is this the right approach? Are there simpler alternatives? Does this belong in this service? |
| **2. Logic** | Correctness, edge cases, error handling | What happens when this input is null? What if the downstream service is down? Is there a race condition? |
| **3. Safety** | Security, performance, observability | Is user input validated? Are queries indexed? Is there logging/metrics for debugging? |
| **4. Style** | Naming, formatting, conventions | Does this follow team conventions? Is the naming clear? |

Most review value comes from passes 1 and 2. If you spend all your time on pass 4, you're wasting everyone's time — that's what linters are for.

### Step 3: Check What's Missing, Not Just What's There

The hardest bugs to catch are the ones hiding in code that *wasn't* written:

- Missing error handling
- Missing input validation
- Missing tests for edge cases
- Missing logging or metrics
- Missing migration or rollback plan
- Missing documentation for public APIs

Train yourself to ask: "What would break this?" and "What's not here that should be?"

### Step 4: Run the Code (When It Matters)

For non-trivial changes — especially UI changes, new APIs, or data migrations — pull the branch and run it locally. Reading a diff is not a substitute for seeing the behavior. You don't need to do this for every PR, but if you're approving a change you can't reason about from the diff alone, you should.

For IDE-specific review tools (diff viewer, blame/annotate, Find Usages, coverage visualization), see IDE.

---

## Giving Feedback — The Art of Not Being a Jerk

### Classify Your Comments

Not every comment carries the same weight. Make this explicit so the author knows what's blocking and what's optional.

| Prefix | Meaning | Author Should |
|---|---|---|
| **`blocker:`** | This must be fixed before merging | Address it |
| **`suggestion:`** | I think this is better, but it's your call | Consider it, respond either way |
| **`nit:`** | Style/preference, totally optional | Fix if easy, ignore if not |
| **`question:`** | I don't understand this — help me learn | Explain (in the code or the comment) |
| **`praise:`** | This is genuinely good work | Keep doing it |

If you don't classify your comments, authors will treat everything as a blocker, or nothing as a blocker. Both are bad.

### Feedback That Helps vs. Feedback That Hurts

| Bad Feedback | Why It's Bad | Better Alternative |
|---|---|---|
| "This is wrong." | No explanation, no direction | "This will return null when X is empty — consider adding a guard clause here." |
| "Why did you do it this way?" | Sounds accusatory even if you're curious | "What drove the decision to use X over Y here? Wondering if Y would simplify the error handling." |
| "I would have done it differently." | Irrelevant unless you explain why | "An alternative approach: [code]. This avoids the allocation on every call." |
| "LGTM" (on a 500-line PR) | Rubber stamp, adds no value | If you truly reviewed it and it's clean, say what you checked: "Reviewed the error paths and concurrency — looks solid." |
| 15 nitpicks, 0 design comments | Misallocated attention | Lead with the important feedback. Save nits for the end or skip them entirely. |

### The Praise Ratio

If you only ever leave critical comments, authors will dread your reviews. Call out good patterns when you see them. "Nice use of the builder pattern here — much cleaner than the previous approach" costs you 5 seconds and makes the author's day. You don't need to be fake about it — just notice when something is genuinely well done.

### Tone Rules

- **Use "we" not "you."** "We should add validation here" lands differently than "You forgot validation."
- **Ask questions instead of making demands.** "Have you considered X?" invites dialogue. "Do X." shuts it down.
- **Assume competence.** The author probably had a reason for their choice. Ask about it before assuming they were wrong.
- **Be specific.** "This is confusing" is useless. "This method does three things — consider splitting it into `parseInput()`, `validate()`, and `execute()`" is actionable.

---

## The Author's Playbook — How to Submit Reviewable Code

The best way to get a fast, useful review is to make the reviewer's job easy. Most slow review cycles are caused by the author, not the reviewer.

### Keep PRs Small

This is the single most impactful thing you can do.

| PR Size (lines changed) | Typical Review Quality | Typical Review Time |
|---|---|---|
| < 100 lines | High — reviewer can hold the full context | Minutes to an hour |
| 100-300 lines | Good — reviewer can follow with effort | Hours |
| 300-500 lines | Declining — reviewer starts skimming | A day or more |
| 500+ lines | Poor — rubber-stamp territory | Days, or never |

Research consistently shows that review quality drops off a cliff past ~300 lines. If your PR is over 500 lines, it will get rubber-stamped. Not because your reviewers are lazy — because human attention is finite.

**How to keep PRs small:**

- **Stack PRs** — break a feature into a chain of dependent PRs. Each one is reviewable on its own.
- **Separate refactoring from behavior changes.** A PR that renames 40 files AND changes business logic is unreviable. Do the rename first, then the logic change.
- **Extract infrastructure changes.** New database tables, config changes, and dependency upgrades should be separate PRs.
- **Ship behind feature flags.** You can merge incomplete features safely if they're gated.

### Write a Good PR Description

A PR without a description is a PR that will be reviewed slowly and poorly.

**What to include:**

```
## What

One-sentence summary of what this PR does.

## Why

What problem does this solve? Link to the ticket/issue.

## How

Brief explanation of the approach. Call out any non-obvious decisions.

## Testing

How was this tested? Manual steps, automated tests, screenshots for UI changes.

## Rollback

How do we undo this if it goes wrong? (For non-trivial changes)
```

You don't need all five sections for a 20-line bug fix. But for anything that touches business logic, data, or infrastructure — write the description. Your future self will thank you during an incident.

### Make the Diff Reviewable

- **Commit history matters.** Clean, logical commits that tell a story are easier to review than a single squashed blob. Consider organizing commits as: "Add data model → Add service layer → Add API endpoint → Add tests."
- **Don't mix formatting changes with logic changes.** If you ran a formatter, do that in a separate commit.
- **Highlight the important parts.** If your PR has 200 lines but only 30 are interesting, say so in the description. "The key change is in `OrderService.java:45-80` — everything else is generated code / test fixtures."

### Respond to Feedback Gracefully

- **Respond to every comment.** Even if it's just "Done" or "Good point, fixed." Leaving comments unresolved signals you didn't read them.
- **Push back when you disagree** — but explain why. "I considered that, but X because Y" is a perfectly valid response. Silent compliance on everything suggests you're not thinking critically about the feedback.
- **Don't take it personally.** The review is about the code, not about you. If a reviewer's tone is off, address it privately — don't escalate in the PR comments.

---

## Review Turnaround — How Fast Is Fast Enough

| Situation | Target Turnaround | Why |
|---|---|---|
| Normal PR | < 4 business hours | Keeps the team unblocked without requiring instant responses |
| Small/trivial PR (< 50 lines) | < 2 hours | These are easy — don't let them languish |
| Urgent fix (production issue) | < 1 hour | Prioritize over current work |
| Large PR (500+ lines) | Same day, but push back on size | Review it, but also comment that it should be broken up next time |

**The 24-hour rule:** If a PR has been open for more than 24 business hours without a review, something is broken. Either the team is overloaded, the reviewer wasn't pinged, or the PR is too big to face. Diagnose and fix the root cause.

**Context switching cost:** Reviewing a PR takes less time than you think. Most engineers overestimate the cost of switching context to do a review. In practice, a well-written 200-line PR takes 15-30 minutes. The cost of *not* reviewing — blocked teammates, stale branches, merge conflicts — is higher.

---

## What to Automate — Stop Arguing About Formatting

If humans are debating it in code review, automate it. Every minute spent on style arguments is a minute not spent on logic, design, or security.

| Concern | Automate With | Stop Doing in Review |
|---|---|---|
| Code formatting | Prettier, google-java-format, spotless | Arguing about bracket placement |
| Import ordering | IDE settings, isort, organize-imports | Manually reordering imports |
| Linting rules | ESLint, Checkstyle, PMD, Pylint | Pointing out unused variables |
| Type safety | TypeScript strict mode, compiler warnings | Catching type mismatches by eye |
| Test coverage thresholds | CI coverage gates | Manually counting test cases |
| Dependency vulnerabilities | Snyk, Dependabot, Renovate | Googling CVEs during review |
| Commit message format | commitlint, git hooks | Rewriting commit messages in comments |

**The rule:** If a computer can catch it, a human shouldn't be spending review time on it. Invest in CI checks and pre-commit hooks so reviewers can focus on what machines can't evaluate — design, logic, and intent.

---

## Common Anti-Patterns — What NOT to Do

### 1. Rubber-Stamping

**Symptoms:**
- "LGTM" on every PR regardless of size or complexity
- Approvals within 60 seconds of PR creation
- Zero comments on non-trivial changes

**Why it's dangerous:** It creates the illusion of review without the substance. Bugs ship, and the team falsely believes they have a safety net.

**Fix:** Track review comment rates. If a reviewer consistently approves without comments on 300+ line PRs, have a conversation. Consider requiring at least one inline comment on non-trivial PRs.

### 2. Nitpick Culture

**Symptoms:**
- 20+ comments on a PR, all about naming and formatting
- Zero comments about logic, design, or edge cases
- Authors dread opening PRs because of the barrage

**Why it's dangerous:** It drives attention away from high-value feedback, slows down the team, and creates adversarial dynamics. Engineers start avoiding code changes to avoid the review gauntlet.

**Fix:** Adopt comment prefixes (`blocker:`, `nit:`, `suggestion:`). Agree as a team that nits are optional and should never block a merge. Automate formatting and linting so there's nothing to nitpick.

### 3. The Gatekeeper

**Symptoms:**
- One senior engineer must approve every PR
- PRs sit for days waiting for this person
- Other reviewers' approvals "don't count"

**Why it's dangerous:** Single point of failure. When the gatekeeper is on vacation, sick, or leaves the team, everything stops. It also prevents other engineers from developing review skills.

**Fix:** Rotate review responsibilities. Use CODEOWNERS files to distribute ownership. Trust your team — if you can't trust them to review code, you have a hiring or mentoring problem, not a review process problem.

### 4. The Mega-PR

**Symptoms:**
- PRs with 1000+ lines changed
- "I'll review it this weekend" (they won't)
- Reviewers skim the first 200 lines and approve

**Why it's dangerous:** Large PRs don't get reviewed — they get approved. The probability of catching a bug in a 1000-line PR is nearly zero because the reviewer can't hold all the context in their head.

**Fix:** Set a soft limit (e.g., 400 lines). When a PR exceeds it, the first review comment should be: "Can we break this up?" Not as punishment — as a genuine request to make the review useful.

### 5. Review Ping-Pong

**Symptoms:**
- Reviewer leaves 3 comments → author fixes → reviewer finds 3 new things → repeat
- PR goes through 5+ review cycles over several days
- Both parties are frustrated

**Why it's dangerous:** It signals the reviewer isn't doing thorough passes. Each cycle adds context-switching cost and delay.

**Fix:** Do a complete review in one pass. Leave all your comments at once. If you find yourself adding new comments on the second round that you should have caught the first time, you're reviewing too quickly.

### 6. Bikeshedding

**Symptoms:**
- Long comment threads about variable names on a PR that changes critical business logic
- The most complex part of the PR has zero comments
- The team can debate `userId` vs `user_id` for three days

**Why it's dangerous:** It's the path of least resistance — anyone can have an opinion on naming, but evaluating the correctness of a retry algorithm requires effort. Bikeshedding is a sign that reviewers are avoiding the hard work.

**Fix:** Agree on conventions once (in a style guide), automate what you can, and move on. If a naming debate lasts more than two comments, take it offline or defer to the author.

---

## Code Review for Different Change Types

Not every PR deserves the same level of scrutiny. Calibrate your effort:

| Change Type | Review Depth | Focus On |
|---|---|---|
| **Bug fix** | Medium — verify the fix + verify no regression | Root cause addressed? Test added for the failing case? |
| **New feature** | High — full design + logic + test review | Right abstraction? Edge cases? API contract correct? |
| **Refactoring** | Medium — behavior should not change | Behavior preserved? Tests still pass? No accidental changes? |
| **Dependency upgrade** | Low-Medium — changelog + breaking changes | Breaking changes? CVE fixes? Lock file updated? |
| **Config change** | Low but careful — small changes, big blast radius | Correct environment? Validated values? Rollback plan? |
| **Test-only change** | Low — mostly a sanity check | Tests actually test something meaningful? Not just asserting `true == true`? |
| **Documentation** | Low — quick read for accuracy | Technically correct? Will it confuse more than it helps? |

---

## Code Review Etiquette — The Unwritten Rules

1. **Don't approve and then leave blockers.** If you leave a blocker comment, don't also approve the PR. It sends mixed signals.
2. **Don't request changes on something you haven't fully reviewed.** If you've only looked at one file, say so — don't block the whole PR.
3. **Don't re-review unchanged code.** If the author addressed your comments, verify the fix. Don't use it as an opportunity to find new things in code you already approved.
4. **Don't leave comments and disappear.** If you start a review, finish it. Coming back two days later with "oh, one more thing" is demoralizing.
5. **Don't use reviews to teach at the wrong time.** If something is correct but you'd do it differently, save the lesson for a pairing session or team discussion — not a PR that's blocking a release.
6. **Do acknowledge when you were wrong.** If the author pushes back and they're right, say so. "Good point, I hadn't considered that" builds trust.

---

## Measuring Review Health — What to Track

You can't improve what you don't measure. Track these metrics at the team level (not individual):

| Metric | Healthy Range | Red Flag |
|---|---|---|
| **Time to first review** | < 4 hours | > 24 hours consistently |
| **Review cycles per PR** | 1-2 rounds | 4+ rounds regularly |
| **PR size (median lines)** | 100-300 lines | > 500 lines consistently |
| **Comment-to-approval ratio** | At least 1 comment per 200 lines on non-trivial PRs | Zero comments on large PRs |
| **Time from PR open to merge** | < 1 business day | > 3 business days consistently |
| **Review participation** | Distributed across team | 1-2 people doing 80%+ of reviews |

**Don't optimize for speed alone.** A team that merges everything in 30 minutes with no comments isn't fast — they're not reviewing.

---

## Quick Reference Checklist

### For Reviewers

- [ ] Read the PR description and linked ticket before looking at code
- [ ] Review in layers: design → logic → safety → style
- [ ] Check what's missing, not just what's there
- [ ] Classify comments: `blocker:`, `suggestion:`, `nit:`, `question:`, `praise:`
- [ ] Leave all comments in one pass — avoid review ping-pong
- [ ] Use "we" language and assume the author had reasons for their choices
- [ ] Actually praise good code when you see it
- [ ] Respond within 4 business hours for normal PRs
- [ ] Pull the branch and run it for non-trivial UI or behavior changes
- [ ] Don't block on nits — save formatting opinions for the linter config

### For Authors

- [ ] Keep PRs under 300 lines whenever possible
- [ ] Write a PR description with what, why, how, and testing
- [ ] Separate refactoring from behavior changes
- [ ] Self-review your own diff before requesting review
- [ ] Highlight the important parts of large PRs in the description
- [ ] Respond to every comment, even if it's just "Done"
- [ ] Push back when you disagree — with an explanation
- [ ] Don't take feedback personally — it's about the code

### For the Team

- [ ] Automate formatting, linting, and coverage checks in CI
- [ ] Agree on comment prefixes so everyone knows what's blocking
- [ ] Set a soft PR size limit and hold each other to it
- [ ] Rotate review responsibilities — no single gatekeeper
- [ ] Track review health metrics monthly
- [ ] Require at least 1 approval for normal changes, 2 for high-risk changes
- [ ] Treat review turnaround as a team commitment, not a personal favor
