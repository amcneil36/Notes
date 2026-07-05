# Version Control Best Practices

Version control is the foundation of everything else — CI/CD, code review, releasing, collaboration. Get it wrong and you'll spend more time fighting your tools than shipping code. Get it right and it's invisible. The best version control workflow is one where nobody has to think about version control.

The goal: **keep the mainline stable, branches short, history useful, and merges painless.**

---

## Branching Strategies — Choosing the Right Model

There is no universal answer. But there is usually a clear best answer for your situation. Here are the three models worth considering — everything else is a variation of these.

### 1. Trunk-Based Development (Recommended Default)

Everyone works on `main` (or very short-lived branches that merge back within hours to a couple of days). Feature flags gate incomplete work. Releases cut directly from trunk.

```
main: ──●──●──●──●──●──●──●──●──●──●──
              ↑         ↑
          short-lived   short-lived
          branch        branch
          (1-2 days)    (hours)
```

**When to use:**
- Microservices where each team owns a service end-to-end
- Teams with strong CI/CD, automated tests, and deployment confidence
- Services that deploy multiple times per day or week
- When you want to minimize merge conflicts and integration pain

**When NOT to use:**
- Versioned software (libraries, SDKs, mobile apps) that ships numbered releases
- Teams without automated tests — broken trunk blocks everyone
- Heavily regulated environments requiring formal release sign-off against a frozen branch

**The trade-off:** Requires discipline. Main must always be releasable. That means fast tests, good feature flags, and a team culture where breaking trunk is treated seriously. The payoff is that you almost never deal with merge conflicts and integration surprises disappear.

### 2. GitHub Flow

Feature branches off `main`, pull request to `main`, deploy from `main`. No `develop` branch, no release branches. Simpler than GitFlow, more structured than pure trunk-based.

```
main:    ──●──●──────●──────●──●──●──
               \    /  \      /
feature-a:      ●──●    \    /
                         ●──●
                      feature-b
```

**When to use:**
- Most teams. This is the pragmatic middle ground.
- Teams that want code review via pull requests on every change
- When branches live 1-5 days (short enough to avoid drift, long enough for review)

**When NOT to use:**
- When you need to maintain multiple release versions simultaneously
- When your deploy pipeline takes hours and you need a staging branch

**The trade-off:** You get code review on everything (good), but branches that linger more than a few days start accumulating merge pain (bad). The discipline required here is keeping branches short, not keeping trunk stable — the PR process handles that.

### 3. GitFlow

Long-lived `develop` branch, feature branches off `develop`, release branches for stabilization, hotfix branches off `main`. The full ceremony.

```
main:      ──●──────────────●──────●──
              \            / \    /
release:       \      ●──●   ●──●
                \    /        (hotfix)
develop:    ──●──●──●──●──●──●──
               \  /     \  /
feature-a:      ●        ●
                       feature-b
```

**When to use:**
- Versioned software that ships named releases (v2.3.1)
- Mobile apps with app store review cycles
- Libraries and SDKs where multiple major versions are supported simultaneously
- Regulatory environments requiring a formal release branch with sign-offs

**When NOT to use:**
- Microservices deploying continuously — GitFlow adds ceremony that actively slows you down
- Small teams where the overhead of `develop` + `release` + `hotfix` branches exceeds the value
- Any situation where you deploy more than once a week

**The trade-off:** Maximum control over what goes into each release. But also maximum friction, maximum merge conflicts, and the highest chance that "it works on develop but breaks on main." GitFlow was designed for a world of infrequent, versioned releases. If that's not your world, don't use it.

### Strategy Comparison

| Factor | Trunk-Based | GitHub Flow | GitFlow |
|---|---|---|---|
| Branch lifespan | Hours | 1–5 days | Days to weeks |
| Merge conflict risk | Very low | Low | High |
| Release flexibility | Deploy anytime | Deploy anytime | Scheduled releases |
| Required discipline | Keep trunk stable | Keep branches short | Follow the full workflow |
| Code review model | Optional / pair programming | PR-based | PR-based |
| Best for | High-velocity services | Most teams | Versioned products |
| Worst for | No-test environments | Long-lived features | Continuous deployment |

**The bottom line:** If you're deploying a service (not shipping a versioned product), start with GitHub Flow. Graduate to trunk-based when your CI/CD and test coverage are mature enough. Only reach for GitFlow when you genuinely need to maintain multiple release versions.

---

## Branch Lifecycle — How Long Is Too Long?

Short-lived branches are not a suggestion — they're a survival strategy. The longer a branch lives, the more it diverges from `main`, and the more painful the merge becomes. This isn't linear — it's exponential. A branch that's 2 days old is easy to merge. A branch that's 2 weeks old is a therapy session.

| Branch Age | Risk Level | What Happens |
|---|---|---|
| < 1 day | Low | Merges cleanly almost every time |
| 1–3 days | Low-Medium | Occasional minor conflicts, easy to resolve |
| 4–7 days | Medium | Conflicts likely, especially in active codebases |
| 1–2 weeks | High | Significant merge effort, integration bugs probable |
| 2+ weeks | Critical | You're not branching, you're forking. Expect pain. |

### Rules for Branch Hygiene

1. **Target 1-3 days for feature branches.** If your feature takes longer, break it into smaller increments. Ship the data model first, then the API, then the UI. Use feature flags to hide incomplete work.

2. **Rebase or merge from main daily.** If you must have a longer branch, pull changes from `main` into your branch every day. Don't let the gap widen.

3. **Delete branches after merge.** Stale branches are clutter. They confuse people ("is this still active?"), pollute autocomplete, and occasionally get accidentally worked on. Merge, delete, move on.

4. **One branch per logical change.** Don't bundle unrelated work into a single branch. "Add payment endpoint + fix logging + update README" is three branches, not one. This also makes code review tractable and rollback surgical.

---

## Release Branching — Main vs. Release Branch

This question comes up constantly: "Should we release from `main` or create release branches?"

### Release from Main (Recommended for Services)

Every commit to `main` is a release candidate. Deploy what's on `main`.

```
main:  ──●──●──●──●──●──●──  (deploy any of these)
```

**Advantages:**
- Simple mental model — `main` is truth
- No branch synchronization issues
- Hotfixes go to `main` and are automatically in the next deploy
- Works naturally with continuous deployment

**When this breaks down:** When you need to ship a fix for the version currently in production while `main` has moved ahead with unreleased features. In that case, cherry-pick the fix to a tag or short-lived hotfix branch.

### Release Branches (For Versioned Products)

Cut a branch when you're ready to stabilize. Only bug fixes go into the release branch. New features continue on `main` or `develop`.

```
main:       ──●──●──●──●──●──●──●──
                  \
release/2.3: ──●──●──●  (v2.3.0, v2.3.1)
                       \
                        ●  (v2.3.2 hotfix)
```

**Advantages:**
- Can stabilize a release without freezing development
- Can maintain and patch multiple versions simultaneously
- Clear audit trail of what went into each release

**When this breaks down:** When hotfixes need to go to both the release branch and `main` — you now have two places to apply the fix, and they can drift. Always forward-port fixes: apply to the release branch, then merge or cherry-pick to `main`.

### Decision Guide

| Situation | Strategy |
|---|---|
| SaaS / microservices / one version in production | Release from main |
| Mobile app with store review cycles | Release branches |
| Library / SDK with semver | Release branches |
| Internal tool with one consumer | Release from main |
| Multiple customers on different versions | Release branches (per major version) |

---

## Merge vs. Rebase — The Eternal Debate

Both exist for a reason. The debate isn't which is "better" — it's which is appropriate for the situation.

### Merge

Creates a merge commit that preserves the full branch history. Both parent commits are visible in the graph.

```bash
git checkout main
git merge feature-branch
```

```
main:    ──●──●──────●──  (merge commit)
               \    /
feature:        ●──●
```

**Use when:**
- Merging a PR into `main` — the merge commit is a clear record of "this PR landed here"
- You want to preserve the exact history of what happened on the branch
- The branch had multiple meaningful commits you want to keep

### Rebase

Replays your commits on top of the target branch, creating a linear history. No merge commit.

```bash
git checkout feature-branch
git rebase main
```

```
Before:  main: ──●──●──
                  \
         feature:  ●──●

After:   main: ──●──●──
                        \
         feature:        ●──●  (replayed on top)
```

**Use when:**
- Updating your feature branch with the latest from `main` — `git rebase main` gives you a clean, linear history
- Cleaning up commit history before opening a PR (interactive rebase to squash fixup commits)
- You want a clean, readable `git log` without merge commit noise

**Never use when:**
- The branch is shared with others who have already pulled it — rebase rewrites history, and their copies will diverge. Rebase only branches that are yours alone.
- After the PR is opened and others are reviewing — changing commit hashes mid-review makes the diff confusing

### The Pragmatic Default

1. **Rebase your feature branch onto main** before opening the PR — keeps your branch up to date with a clean history
2. **Merge (or squash-merge) the PR into main** — creates a clear record in `main`'s history

```bash
# Before opening PR, update your branch
git checkout feature-branch
git rebase main

# Merge PR (via GitHub UI or CLI — squash merge for a clean single commit)
git checkout main
git merge --squash feature-branch
git commit -m "Add payment processing endpoint"
```

### Squash Merge — The Third Option

Collapses all commits on the branch into a single commit on `main`. The branch history is lost (but still visible in the PR if you use GitHub).

**Use when:** Your branch has messy commits ("WIP", "fix typo", "actually fix it this time") and you want a single clean commit on `main`.

**Don't use when:** The branch has meaningful, distinct commits that tell a story (e.g., "add data model", "add API handler", "add validation").

| Strategy | History | Best For |
|---|---|---|
| Merge commit | Full branch history preserved | Feature branches with meaningful commits |
| Squash merge | Single clean commit | Messy branches, atomic changes |
| Rebase + fast-forward | Linear, no merge commit | Clean branches, linear history purists |

---

## Branching Off Something That Isn't Main

Sometimes you need to build on work that hasn't landed in `main` yet. This is one of the trickiest version control situations, and there's no perfect answer — only trade-offs.

### Scenario: Your Task Depends on an Unmerged Branch

Your teammate is working on `feature-auth` which introduces a new auth module. Your task requires that module. `feature-auth` hasn't been merged to `main` yet.

#### Option 1: Branch Off the Dependency Branch (Stacking)

```bash
git checkout feature-auth
git checkout -b feature-dashboard  # branches off feature-auth
```

```
main:            ──●──●──
                      \
feature-auth:          ●──●──●
                              \
feature-dashboard:             ●──●
```

**Pros:** You can start immediately. You have access to all the code you need.

**Cons:** If `feature-auth` changes (rebased, squashed, amended), your branch's base has shifted. When `feature-auth` merges to `main`, you need to rebase `feature-dashboard` onto `main` and resolve any conflicts from the rebase.

**Recovery after `feature-auth` merges:**
```bash
git checkout feature-dashboard
git rebase --onto main feature-auth  # replay your commits onto main, discarding feature-auth base
```

#### Option 2: Wait for the Dependency to Merge

The cleanest option. Don't start until `feature-auth` is in `main`.

**Pros:** No branch dependency issues. Clean history.

**Cons:** You're blocked. If `feature-auth` takes a week to review, you lose a week.

**When this is the right call:** When the dependency is close to merging (hours, not days), or when you have other work to do in the meantime.

#### Option 3: Cherry-Pick What You Need

If you only need a small part of `feature-auth` (say, a type definition or a utility function), cherry-pick those specific commits.

```bash
git checkout main
git checkout -b feature-dashboard
git cherry-pick <commit-hash-from-feature-auth>
```

**Pros:** Minimal dependency. Your branch is based on `main`.

**Cons:** If the cherry-picked code changes in `feature-auth`, you have two divergent copies. May cause conflicts when both branches merge.

**When this is the right call:** When you need 1-2 small, stable pieces — not the entire branch.

#### Option 4: Extract the Shared Code First

If both branches need the same foundational code, extract it into its own PR. Merge that first, then both branches can build on it independently.

```
PR 1: Add auth module interfaces (merge first)
PR 2: Implement auth module (feature-auth, depends on PR 1)
PR 3: Build dashboard using auth interfaces (feature-dashboard, depends on PR 1)
```

**Pros:** Clean dependency graph. No branch-off-branch issues.

**Cons:** Requires upfront planning and decomposition. Not always possible if the dependency is deeply intertwined.

### Decision Guide for Dependencies

| Situation | Recommended Approach |
|---|---|
| Dependency merges within hours | Wait |
| Dependency merges within days, you need all of it | Branch off it (stacking) |
| You only need a tiny piece of the dependency | Cherry-pick |
| Multiple branches need the same foundation | Extract shared code into its own PR first |
| Dependency timeline is unknown or long | Talk to your teammate — pair review to unblock, or restructure the work |

---

## Reverting — Undoing What Shouldn't Have Shipped

Things go wrong. The question isn't whether you'll need to revert — it's how cleanly you can do it when the time comes.

### `git revert` — The Safe Undo

Creates a new commit that undoes the changes from a previous commit. History is preserved. This is what you use in 90% of cases.

```bash
# Revert a single commit
git revert <commit-hash>

# Revert a merge commit (specify which parent to keep — almost always parent 1)
git revert -m 1 <merge-commit-hash>

# Revert a range of commits
git revert --no-commit <oldest-hash>^..<newest-hash>
git commit -m "Revert commits X through Y: broke payment processing"
```

**Why `git revert` and not `git reset`:** Revert adds history — it's a new commit that says "we undid this." Reset erases history. On shared branches (especially `main`), erasing history breaks everyone else's checkout. Use `revert`.

### When to Revert

| Situation | Action |
|---|---|
| Deployed commit causes production issues | Revert immediately, investigate later |
| Merged PR introduces a bug caught in staging | Revert the merge commit |
| Feature is functionally correct but causes performance regression | Revert, optimize, re-land |
| Commit has a typo or minor issue | Fix forward (new commit) — don't revert for trivial things |

### The Revert-and-Reland Pattern

You revert a PR because it caused issues. You fix the issue. Now you need to re-apply the changes. But `git revert` created an "anti-commit" — merging the same branch again won't work because Git thinks those changes are already applied (and then undone).

**Solution: Revert the revert.**

```bash
# Original merge
git merge feature-payments  # commit A

# Something breaks — revert the merge
git revert -m 1 A  # commit B (undoes A)

# Fix the issue on the feature branch
git checkout feature-payments
# ... make fixes ...
git commit -m "Fix payment validation edge case"

# Revert the revert, then merge the fixed branch
git checkout main
git revert B  # commit C (undoes the undo — re-applies original changes)
git merge feature-payments  # brings in the fix
```

This is confusing the first time. It's completely normal. The key insight: reverting a merge doesn't "un-merge" the branch in Git's eyes — it just applies the inverse diff. To re-merge, you have to undo the inverse first.

---

## Essential Commands — The Ones That Matter

You don't need to know 200 Git commands. You need to know 20 well.

### Daily Workflow

| Command | What It Does | When to Use |
|---|---|---|
| `git status` | Shows working tree state | Before every commit — know what you're committing |
| `git diff` | Shows unstaged changes | Review changes before staging |
| `git diff --staged` | Shows staged changes | Review changes before committing |
| `git add -p` | Stage changes interactively, hunk by hunk | When you want to commit only part of a file |
| `git commit -m "message"` | Create a commit | After staging changes |
| `git push -u origin branch` | Push branch and set upstream | First push of a new branch |
| `git pull --rebase` | Pull and rebase local commits on top | Updating your branch (avoids unnecessary merge commits) |

### Branching and Navigation

| Command | What It Does | When to Use |
|---|---|---|
| `git checkout -b name` | Create and switch to a new branch | Starting new work |
| `git switch name` | Switch branches (modern alternative) | Navigating between branches |
| `git branch -d name` | Delete a merged branch | After PR is merged |
| `git stash` / `git stash pop` | Temporarily shelve changes | Switching branches with uncommitted work |

### History and Investigation

| Command | What It Does | When to Use |
|---|---|---|
| `git log --oneline --graph` | Visual commit history | Understanding branch topology |
| `git log --follow path/file` | History of a specific file (including renames) | Tracing a file's evolution |
| `git blame path/file` | Who changed each line and when | Finding who to ask about code |
| `git bisect start` / `good` / `bad` | Binary search for the commit that introduced a bug | Debugging regressions |
| `git reflog` | History of all HEAD movements | Recovering from mistakes ("I lost my commit!") |

### Recovery

| Command | What It Does | When to Use |
|---|---|---|
| `git revert <hash>` | Undo a commit with a new commit | Undoing changes on shared branches |
| `git reset --soft HEAD~1` | Undo last commit, keep changes staged | "I committed too early" |
| `git reset HEAD~1` | Undo last commit, keep changes unstaged | "I committed the wrong thing" |
| `git checkout -- file` | Discard unstaged changes to a file | "I don't want these edits" |
| `git cherry-pick <hash>` | Apply a specific commit to current branch | Grabbing a fix from another branch |

### The Commands You Should Avoid on Shared Branches

| Command | Why It's Dangerous |
|---|---|
| `git push --force` | Rewrites remote history — breaks everyone else's checkout |
| `git push --force-with-lease` | Safer force push (checks for upstream changes first), but still use sparingly |
| `git reset --hard` | Destroys uncommitted changes permanently — no recovery |
| `git rebase` on a shared branch | Rewrites commits others have already based work on |
| `git commit --amend` after pushing | Changes a commit that's already on the remote |

---

## Commit Messages — Making History Readable

Your commit history is documentation. Six months from now, someone (probably you) will run `git log` trying to understand why a change was made. Bad commit messages make this archaeology impossible.

### Format

```
<type>: <short summary in imperative mood> (< 72 chars)

<optional body — explain WHY, not WHAT>

<optional footer — ticket refs, breaking changes>
```

### Types (Conventional Commits)

| Type | Use For |
|---|---|
| `feat` | New functionality |
| `fix` | Bug fix |
| `refactor` | Code change that neither fixes a bug nor adds a feature |
| `docs` | Documentation only |
| `test` | Adding or updating tests |
| `chore` | Build, CI, tooling, dependencies |
| `perf` | Performance improvement |

### Good vs. Bad

| Bad | Good |
|---|---|
| "Fix bug" | "fix: prevent duplicate order creation on retry timeout" |
| "Update code" | "refactor: extract payment validation into dedicated service" |
| "WIP" | Don't commit WIP to `main`. Squash before merge. |
| "Changes" | "feat: add rate limiting to /api/orders endpoint" |
| "Address PR feedback" | Squash this into the relevant commit before merge |

**The litmus test:** If someone reads only your commit messages (not the diffs), can they understand what changed and why? If not, your messages need work.

---

## Common Mistakes

### 1. Long-Lived Feature Branches

A branch that's been open for 3 weeks is not a feature branch — it's a parallel universe. When you finally merge it, you'll spend more time resolving conflicts than you spent writing the feature. Break work into smaller increments. Ship behind feature flags. Merge daily if you can.

### 2. Committing Directly to Main

Unless your team has explicitly adopted trunk-based development with the discipline it requires (fast tests, feature flags, pair programming), committing directly to `main` bypasses code review and skips CI validation on the branch. Use pull requests.

### 3. Using `git push --force` on Shared Branches

Force-pushing rewrites history that other people have already pulled. Their local branches now diverge from the remote in ways that are painful to reconcile. If you must force-push (after a rebase on your own branch), use `--force-with-lease` to at least check that nobody has pushed in the meantime.

### 4. Not Pulling Before Pushing

```bash
# This creates an unnecessary merge commit
git push  # rejected — remote has new commits
git pull  # creates a merge commit
git push  # now it works, but history has a pointless merge bubble
```

Use `git pull --rebase` to keep history clean. Or better yet, set it as the default:
```bash
git config --global pull.rebase true
```

### 5. Committing Generated Files

`node_modules/`, `target/`, `.class` files, compiled assets, IDE settings — none of these belong in version control. Use `.gitignore` aggressively. If it can be generated from source, it shouldn't be tracked.

### 6. Giant Commits

A commit that touches 47 files across 6 modules is unreviewable and unrevertable. If something in that commit needs to be rolled back, you can't surgically revert just the problematic part. Keep commits focused: one logical change per commit.

### 7. Treating Branches as Backups

Creating branches like `feature-payments-v2-final-FINAL-working` is a sign that something has gone wrong. Branches are for parallel lines of development, not for saving checkpoints. Use commits for checkpoints. Use branches for divergent work.

### 8. Ignoring Merge Conflicts

"Accept theirs" on every conflict just to make the merge go through. You're discarding changes. Merge conflicts exist because two people changed the same code — at least one of those changes matters. Read the conflict. Understand both sides. Resolve intentionally.

### 9. Not Using `.gitignore` From Day One

Adding `.gitignore` after you've already committed `node_modules` doesn't remove it from history. The repo is now permanently bloated. Start every project with a proper `.gitignore`. Use templates from github/gitignore.

### 10. Rebasing Public History

Rebasing commits that have been pushed to a shared branch rewrites those commits' hashes. Everyone who pulled those commits now has orphaned references. Only rebase local, unpushed commits — or branches that are exclusively yours.

---

## Quick Reference Checklist

### Branching

- [ ] Using a branching strategy appropriate for your deployment model (trunk-based or GitHub Flow for services, GitFlow for versioned products)
- [ ] Feature branches live less than 5 days — ideally 1-3
- [ ] Branches are deleted after merge
- [ ] One logical change per branch — no bundling unrelated work
- [ ] Branch names are descriptive: `feature/add-payment-endpoint`, `fix/order-duplicate`, `chore/upgrade-spring-boot`

### Merging and Rebasing

- [ ] Feature branches are rebased onto `main` before opening a PR
- [ ] PRs are merged (or squash-merged) into `main` — not rebased
- [ ] `git pull --rebase` is the default pull strategy
- [ ] Merge conflicts are resolved by understanding both sides, not blindly accepting one
- [ ] Force push is never used on shared branches — `--force-with-lease` only on personal branches

### Commits

- [ ] Commit messages follow conventional format: `type: short description`
- [ ] Messages explain why, not just what
- [ ] No WIP, fixup, or "address feedback" commits on `main` — squash before merge
- [ ] Commits are atomic — one logical change each
- [ ] Generated files, secrets, and IDE configs are in `.gitignore`

### Releases and Reverts

- [ ] Release strategy matches the deployment model (from main for services, release branches for versioned products)
- [ ] Hotfixes are forward-ported from release branches to `main`
- [ ] `git revert` is used (not `git reset`) to undo changes on shared branches
- [ ] Revert-and-reland pattern is understood for re-applying reverted merges
- [ ] Team knows how to use `git bisect` to find regression-introducing commits
- [ ] Deployment rollback strategy is defined and tested — see Releasing for rollback details

### Dependencies and Stacking

- [ ] When branching off another branch, the recovery plan (rebase --onto) is understood
- [ ] Shared foundational code is extracted into its own PR when multiple branches need it
- [ ] Cherry-pick is used sparingly and only for small, stable pieces
- [ ] Blocked work is communicated early — not discovered at merge time
