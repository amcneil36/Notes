# Testing Best Practices

Bad tests are worse than no tests. They give you false confidence, slow your team down, and make every refactor feel like defusing a bomb. Good tests are the opposite — they're a safety net that lets you move fast, ship confidently, and sleep through the night. This guide covers how to write tests that actually protect you, when to automate vs. test manually, and how to build a test pipeline that doesn't make you want to quit engineering.

---

## The Testing Pyramid (And Why Most Teams Get It Wrong)

The testing pyramid isn't new, but most teams invert it. They write a handful of unit tests, skip integration tests entirely, and pile everything into slow, flaky end-to-end tests. Then they wonder why CI takes 45 minutes and nobody trusts the results.

```
         /  E2E  \          ← Few, slow, expensive
        /----------\
       / Integration \      ← Moderate count, moderate speed
      /----------------\
     /    Unit Tests     \  ← Many, fast, cheap
    /____________________\
```

### What Each Layer Should Do

| Layer | Scope | Speed | Count | Purpose |
|-------|-------|-------|-------|---------|
| **Unit** | Single function/class, no I/O | < 10ms each | 70-80% of tests | Verify logic in isolation |
| **Integration** | Multiple components, real I/O (DB, HTTP, queues) | < 5s each | 15-25% of tests | Verify components work together |
| **E2E** | Full system, user-facing flows | 10-60s each | 5-10% of tests | Verify critical user journeys |

### The Distribution That Actually Works

Aim for this ratio as a starting point, then adjust based on your system's architecture:

| Metric | Target | Why |
|--------|--------|-----|
| Unit test coverage | **80%+ line coverage** | Catches logic bugs early, runs in seconds |
| Integration test coverage | **60%+ of service boundaries** | Every API endpoint, DB query path, and queue consumer should have at least one integration test |
| E2E test count | **10-20 critical paths** | Login, checkout, payment, core CRUD — the flows that generate revenue or page you at 3am |
| Full suite runtime | **< 10 minutes** | Longer than this and developers stop running tests before pushing |

If your E2E tests outnumber your integration tests, you've got the pyramid upside down. Fix it.

---

## Unit Testing: The Foundation

Unit tests are your first line of defense. They should be fast, deterministic, and test behavior — not implementation.

### What to Unit Test

- Pure business logic (calculations, transformations, validations)
- State machines and conditional branching
- Edge cases: nulls, empty collections, boundary values, overflow
- Error handling paths (what happens when the input is garbage?)
- Serialization/deserialization logic

### What NOT to Unit Test

- Trivial getters/setters with no logic
- Framework glue code (controllers that just delegate to a service)
- Third-party library internals
- Configuration files

### The Three Laws of a Good Unit Test

1. **Test behavior, not implementation.** If you rename a private method and 15 tests break, those tests are coupled to implementation. Test the public contract.
2. **One assertion per logical concept.** It's fine to have multiple `assert` calls if they all verify the same behavior. It's not fine to test three unrelated things in one test.
3. **No shared mutable state.** Each test sets up its own world, runs, and tears it down. If test order matters, your tests are broken.

### The Anatomy of a Good Test

Follow the **Arrange → Act → Assert** pattern (also called Given-When-Then):

```
test "expired coupon returns zero discount" {
    // Arrange
    coupon = createCoupon(expiresAt: yesterday)
    cart = createCart(items: [itemA, itemB], total: 100.00)

    // Act
    discount = applyDiscount(cart, coupon)

    // Assert
    assertEqual(discount, 0.00)
    assertEqual(cart.total, 100.00)
}
```

Name tests like sentences: `"returns zero discount when coupon is expired"`, not `"testApplyDiscount3"`. When a test fails in CI, the name should tell you what broke without reading the code.

---

## Integration Testing: Where the Real Bugs Live

Unit tests prove your code works in isolation. Integration tests prove it works with the outside world. Most production incidents aren't logic bugs — they're integration bugs: wrong SQL, mismatched serialization, broken contract with a downstream service.

### What to Integration Test

| Integration Point | What to Verify |
|-------------------|----------------|
| **Database** | Queries return expected results, migrations work, constraints enforce rules |
| **HTTP APIs** | Request/response serialization, status codes, error responses, auth headers |
| **Message queues** | Messages serialize correctly, consumers handle retries and dead letters |
| **Caches** | Cache hits return correct data, TTL expiration works, cache misses fall through |
| **File systems / object stores** | Reads, writes, permissions, handling of missing files |

### Best Practices

- **Use real dependencies when possible.** Testcontainers (or equivalent) for databases and queues. In-memory fakes for caches. The closer to production, the more bugs you catch.
- **Don't mock what you don't own.** Mocking a third-party HTTP client means you're testing your assumptions, not reality. Use a lightweight test server or recorded responses instead.
- **Test the sad paths.** What happens when the database is down? When the API returns a 500? When the queue message is malformed? These are the scenarios that page you.
- **Isolate test data.** Use transactions that roll back, unique test prefixes, or ephemeral databases. Never rely on pre-existing data in a shared environment.

---

## End-to-End Testing: Less Is More

E2E tests are the most expensive tests you can write — slow to run, hard to maintain, and prone to flakiness. Use them surgically.

### The E2E Litmus Test

Before writing an E2E test, ask yourself:

> *"If this flow breaks in production, does someone get paged?"*

If yes, write the E2E test. If no, cover it with integration and unit tests instead.

### Rules for E2E Tests That Don't Suck

1. **Cap it at 10-20 critical user journeys.** If you have 200 E2E tests, you don't have a test suite — you have a liability.
2. **Use stable selectors.** Test IDs (`data-testid`), not CSS classes or XPath. CSS changes shouldn't break your tests.
3. **Build in resilience.** Retry on transient failures. Use explicit waits, not `sleep(3)`. Accept that some flakiness is inherent and quarantine tests that flake more than 2% of the time.
4. **Own the test data.** Seed the data you need at the start of each test. Never depend on data created by another test.
5. **Run E2E tests in a dedicated environment.** Not in your unit test suite. Not blocking every PR. Run them on merge to main or on a schedule.

---

## Manual Testing: Not Dead, Just Misunderstood

Automation doesn't replace manual testing — it replaces *repetitive* testing. Some things are still better done by a human.

### When Manual Testing Wins

| Scenario | Why Automation Falls Short |
|----------|---------------------------|
| **Exploratory testing** | Humans find bugs that no one thought to write a test for. Automated tests only catch what you anticipated. |
| **UX/visual review** | Does it *feel* right? Is the animation smooth? Is the error message helpful? Machines can't judge this. |
| **New feature validation** | Before you know what "correct" looks like, automation is premature. Explore first, automate later. |
| **Edge cases in complex workflows** | Multi-step business processes with unusual combinations are expensive to automate and cheap to walk through once. |
| **Security testing** | Automated scanners catch known vulnerabilities. Humans find novel attack vectors through creative abuse. |

### How to Do Manual Testing Well

- **Use checklists, not scripts.** A checklist gives structure without rigidity. "Verify login works" is a script. "Test login with valid creds, invalid creds, expired session, locked account, SSO redirect" is a checklist.
- **Time-box exploratory sessions.** 30-60 minutes, focused on one area. Take notes. File bugs immediately. If you find a bug manually, write an automated regression test so it never comes back.
- **Pair test complex features.** Two people testing together find more bugs than two people testing separately. One drives, one questions.
- **Don't skip manual testing on "small" changes.** The changes you think are safe are the ones that bite you.

---

## Killing Flaky Tests

Flaky tests erode trust. After enough false failures, developers stop investigating test failures and start re-running the pipeline. That's when real bugs slip through.

### Root Causes and Fixes

| Root Cause | Symptom | Fix |
|-----------|---------|-----|
| **Shared mutable state** | Tests pass alone, fail together | Isolate test data. Each test creates and cleans up its own state. |
| **Timing dependencies** | Tests fail under load or on slow machines | Replace `sleep()` with explicit waits or polling. Use deterministic clocks in tests. |
| **Order dependency** | Tests pass in one order, fail in another | Randomize test execution order. If that breaks things, fix the coupling. |
| **External service dependency** | Tests fail when a staging API is down | Mock or stub external services in unit/integration tests. Only hit real services in E2E. |
| **Race conditions** | Tests fail intermittently with no pattern | Use synchronization primitives. If testing async code, wait for completion signals — not arbitrary timeouts. |
| **Non-deterministic data** | Tests fail on certain dates, locales, or timezones | Pin dates and locales in test fixtures. Never use `now()` directly — inject a clock. |

### The Flaky Test Protocol

1. **Quarantine immediately.** Move the flaky test to a separate suite that doesn't block CI. Don't delete it — quarantine it.
2. **Track flake rate.** If a test fails > 2% of runs without code changes, it's flaky.
3. **Fix within 1 sprint.** Quarantined tests that sit for months are dead tests. Fix them or delete them.
4. **Never ignore a flaky test.** "Oh that one always fails" is the sentence that precedes every missed production bug.

---

## Taming Slow Test Suites

If your test suite takes more than 10 minutes, developers will stop running it locally. If it takes more than 20 minutes in CI, it's blocking your deploy velocity.

### Speed Targets

| Suite | Target Duration | Acceptable Max |
|-------|----------------|----------------|
| Unit tests | < 30 seconds | 2 minutes |
| Integration tests | < 5 minutes | 8 minutes |
| E2E tests | < 10 minutes | 15 minutes |
| Full pipeline (all suites) | < 10 minutes | 20 minutes |

### How to Speed Things Up

1. **Parallelize.** Run test files in parallel across multiple cores or CI workers. Most test runners support this natively.
2. **Don't start what you don't need.** If a test doesn't need the full application context, don't boot it. Slice your context to load only the components under test.
3. **Use in-memory databases for unit tests.** Only use real databases for integration tests.
4. **Profile your slowest tests.** The slowest 5% of tests often account for 50% of runtime. Fix those first.
5. **Cache aggressively in CI.** Dependencies, compiled artifacts, container images — cache anything that doesn't change between runs.
6. **Run the right tests at the right time.** Unit tests on every push. Integration tests on PR. E2E tests on merge to main. Not everything needs to run every time.

---

## The Over-Mocking Trap

Mocking is a tool, not a strategy. When you mock everything, your tests prove that your mocks work — not that your code works.

### Signs You're Over-Mocking

- You mock more than 3 dependencies in a single test
- Your mock setup is longer than the actual test
- A refactor breaks 50 tests but zero production behavior
- You mock the class you're testing (yes, people do this)
- Tests pass with mocks but the feature is broken in production

### When to Mock vs. Use Real Dependencies

| Scenario | Approach |
|----------|----------|
| External HTTP API you don't control | **Mock** — use recorded responses or a contract test |
| Your own database | **Real** — use testcontainers or an in-memory DB |
| A slow third-party service | **Mock** — but validate the contract separately |
| A class in your own codebase | **Don't mock** — if it's hard to use in tests, it's hard to use in production. Fix the design. |
| Time/clock | **Mock** — inject a clock, never call `now()` directly |
| File system | **Real** — use temp directories. Mock only if I/O is genuinely slow. |

### The Mocking Rule of Thumb

> Mock at the boundaries of your system, not inside it. If you're mocking your own code to test your own code, your architecture has a problem.

---

## Testing in CI/CD Pipelines

Your test suite is only as good as your pipeline. A well-designed pipeline runs the right tests at the right time, fails fast, and gives clear feedback.

### Pipeline Stage Design

| Stage | Trigger | Tests Run | Timeout | Gate? |
|-------|---------|-----------|---------|-------|
| **Pre-push (local)** | Developer runs manually or via git hook | Unit tests | 2 min | Optional |
| **PR opened / updated** | Every push to a PR branch | Unit + Integration | 10 min | **Yes — block merge** |
| **Merge to main** | After PR merge | Unit + Integration + E2E | 20 min | **Yes — block deploy** |
| **Nightly / scheduled** | Cron (daily) | Full regression + performance + security scans | 60 min | Alerts on failure |

### Pipeline Best Practices

- **Fail fast.** Run unit tests first. If they fail, skip integration and E2E — don't waste 15 minutes to tell the developer what you could have told them in 30 seconds.
- **Parallelize across stages.** Run linting, unit tests, and static analysis concurrently. Run integration tests in parallel shards.
- **Show the failing test, not the log dump.** Configure your CI to surface test names, failure messages, and diffs — not 500 lines of stack trace.
- **Make test results visible.** Post test summaries as PR comments. Use CI dashboards. If developers have to dig through logs to find failures, they won't.
- **Track test metrics over time.** Test count, pass rate, flake rate, duration — trend these weekly. A declining pass rate is an early warning signal.
- **Never skip tests to unblock a deploy.** If the tests are failing, either the tests are wrong or the code is wrong. Both need fixing before you ship.

### Test Parallelization Strategy

| Approach | Pros | Cons | Best For |
|----------|------|------|----------|
| **File-level parallelism** | Simple to implement, no test changes needed | Uneven distribution if test files vary in size | Unit tests |
| **Test-level parallelism** | Maximum distribution | Requires thread-safe tests, harder to debug | Large integration suites |
| **Sharded CI workers** | Scales horizontally, reduces wall time | More CI cost, need a splitting strategy | E2E suites > 10 minutes |
| **Changed-file-based selection** | Only runs affected tests, very fast | Risk of missing transitive dependencies | PR-level fast feedback |

---

## Common Mistakes

### 1. Testing Implementation Instead of Behavior
Your tests should describe *what* the system does, not *how* it does it. If you refactor internals and tests break despite identical behavior, the tests are wrong.

### 2. The "100% Coverage" Delusion
100% line coverage is a vanity metric. You can have 100% coverage and zero confidence if your assertions are weak. Aim for **80% coverage with strong assertions** over 100% coverage with `assertNotNull`.

### 3. Copy-Paste Test Factories
When test setup becomes repetitive, developers copy-paste and tweak. This creates dozens of nearly-identical setup blocks that are impossible to maintain. Extract test builders, factories, or fixtures.

### 4. No Tests for Error Paths
Most teams test the happy path thoroughly and ignore errors entirely. In production, the happy path works fine — it's the error paths that cause outages. **Test what happens when things go wrong.**

### 5. Writing Tests After the Bug Fix
You found a bug, you fixed it, you moved on. Six months later, the same bug is back. Every bug fix should come with a regression test that fails before the fix and passes after.

### 6. Testing in Production Without a Plan
"We'll just test in prod" isn't a strategy — it's an abdication. If you test in production (and sometimes you should), do it deliberately: feature flags, canary deploys, synthetic monitoring. Not by yolo-deploying and hoping.

### 7. Ignoring Test Maintenance
Tests are code. They need refactoring, dependency updates, and cleanup just like production code. If your test suite is a 5,000-line file that nobody dares touch, it's rotting.

---

## Related Guides

- See CI/CD for pipeline integration and deployment practices.

---

## Quick Reference

| Principle | Rule |
|-----------|------|
| Test ratio | 70-80% unit, 15-25% integration, 5-10% E2E |
| Unit coverage target | 80%+ line coverage with strong assertions |
| Integration coverage | Every service boundary has at least one test |
| E2E scope | 10-20 critical user journeys, no more |
| Suite speed | Full pipeline < 10 min target, < 20 min max |
| Flaky threshold | Quarantine any test that fails > 2% without code changes |
| Mock boundary | Mock at system edges, not inside your own code |
| Test naming | Sentence-style: describes behavior and expected outcome |
| Bug fixes | Every fix ships with a regression test |
| Manual testing | Exploratory sessions, UX review, security probing — always time-boxed |

---

## Checklist

- [ ] Test pyramid ratio is roughly correct (not inverted)
- [ ] Unit tests cover business logic, edge cases, and error paths
- [ ] Integration tests exist for every DB, API, and queue interaction
- [ ] E2E tests cover critical revenue and pager-worthy flows only
- [ ] No test depends on another test's data or execution order
- [ ] Flaky tests are quarantined and tracked — not ignored
- [ ] Full test suite runs in under 20 minutes
- [ ] CI pipeline fails fast (unit tests run first)
- [ ] Test results are visible in PR comments or dashboards
- [ ] Mocks are used at system boundaries, not inside your own code
- [ ] Every bug fix includes a regression test
- [ ] Manual exploratory testing is scheduled for new features
- [ ] Test coverage metrics are tracked and trending upward
- [ ] Tests are named as readable sentences
- [ ] Test code is maintained and refactored like production code
