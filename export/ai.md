# AI Best Practices for Software Engineers

AI tools are the biggest productivity multiplier since the IDE. But most engineers either underuse them (treating AI as a fancy autocomplete) or overuse them (blindly shipping AI-generated code without understanding it). Both waste the tool's potential and introduce risk.

The goal: **use AI to amplify your judgment, not replace it. Delegate the mechanical, keep the critical thinking.**

---

## The Mental Model — AI as a Peer, Not an Oracle

The single most important thing to internalize: **AI is a fast, tireless, mediocre-to-good engineer that never gets tired but also never truly understands your system.** It doesn't know your business context, your team's conventions, your production topology, or why that weird workaround exists in line 247. You do.

| AI is great at | AI is bad at |
|---|---|
| Boilerplate and scaffolding | Architectural decisions that require business context |
| Translating between formats (JSON ↔ YAML ↔ SQL ↔ code) | Knowing your team's unwritten conventions |
| Explaining unfamiliar code or concepts | Judging whether a change is safe to ship |
| Finding patterns and inconsistencies | Understanding why a workaround exists |
| Drafting documentation from code | Prioritizing what matters most right now |
| Generating test cases (especially edge cases) | Knowing your production traffic patterns |
| Summarizing long threads, PRs, or docs | Political and organizational judgment calls |
| Rapid prototyping and exploration | Security-critical logic (crypto, auth, access control) |

The engineers who get the most from AI treat it like a very fast junior teammate: give it clear instructions, review its output critically, and never let it make decisions you should be making.

---

## Using AI Across the Engineering Workflow

### 1. Planning and Design

AI is surprisingly good at helping you think through a problem — not because it has better ideas than you, but because it forces you to articulate the problem clearly, and it can rapidly enumerate options you might not have considered.

**What works:**
- Describe a problem and ask for 3-4 different approaches with trade-offs
- Paste a tech design and ask for blind spots or missing edge cases
- Ask "what could go wrong with this approach?" — AI is good at pessimistic enumeration
- Generate sequence diagrams or data flow diagrams from a description
- Rubber-duck debugging: explain your thinking to AI and ask it to poke holes

**What doesn't work:**
- Asking AI to make the decision for you ("Should we use Kafka or RabbitMQ?") — it'll give you a reasonable-sounding answer based on generic factors, but it doesn't know your team's operational maturity with either tool, your latency requirements, or your infra budget
- Using AI output as your design doc without significant editing — the structure will be clean but the content will be generic

**Example — good planning prompt:**

```
We need to add multi-tenancy to our order management service.
Current state: single-tenant, PostgreSQL, ~50 tables, Spring Boot.
Traffic: 2000 orders/hour, growing 3x over the next year.
Tenants will share the same deployment but must have strict data isolation.

Give me 3 approaches (shared schema with tenant column, schema-per-tenant,
database-per-tenant) with trade-offs for: query complexity, data isolation,
operational overhead, and migration effort from our current single-tenant schema.
```

This works because it gives the AI **your specific constraints**, not a generic question.

### 2. Writing Code

This is where most engineers start, and where the biggest pitfalls live.

**The spectrum of AI coding tasks (from high-confidence to low-confidence):**

| Confidence | Task Type | Example |
|---|---|---|
| High | Boilerplate / scaffolding | Generate a new REST controller with CRUD endpoints matching an existing pattern |
| High | Format translation | Convert a JSON schema to a TypeScript interface |
| High | Regex / string manipulation | Write a regex to extract version numbers from changelog entries |
| High | Unit test generation | Generate test cases for a pure function with known inputs/outputs |
| Medium | Refactoring | Extract a method, rename across files, convert callback to async/await |
| Medium | Bug fix with clear repro | "This function returns null when the list is empty — fix it" |
| Medium | Integration code | Write a Kafka consumer config that matches our existing producer setup |
| Low | Complex business logic | Implement a pricing engine with discount stacking rules |
| Low | Performance-critical code | Optimize a query that joins 6 tables with window functions |
| Low | Security-sensitive code | Implement JWT validation, RBAC, or encryption |
| Very Low | Architectural changes | Redesign the data model for a new feature spanning 4 services |

**The rule:** the more context AI would need to get it right, the less you should trust its output without careful review. Boilerplate is safe because the pattern is well-established. Business logic is risky because the AI doesn't know your business.

**Practical workflow for AI-assisted coding:**

1. **Start with context.** Point the AI at the relevant files, interfaces, and patterns before asking it to generate code. The more context it has, the more consistent its output will be with your codebase.
2. **Generate in small chunks.** Don't ask for an entire feature at once. Generate one class, one method, one test at a time. Review each piece before moving on.
3. **Read every line.** If you can't explain what a line does and why it's there, you don't understand it well enough to ship it. AI-generated code that you don't understand is technical debt with your name on it.
4. **Test it.** AI-generated code compiles and looks right more often than it actually is right. Run the tests. Write new tests if the AI didn't. Check edge cases.
5. **Check for hallucinated APIs.** AI will confidently use methods, classes, and libraries that don't exist. Always verify that imports resolve and APIs match your dependency versions.

### 3. Testing

AI is genuinely excellent at test generation — often better than the developer who wrote the code, because it mechanically considers edge cases that humans skip out of familiarity bias.

**What to use AI for:**

| Test Type | AI Effectiveness | Notes |
|---|---|---|
| Unit tests for pure functions | Excellent | Give it the function signature and contract, get comprehensive cases |
| Edge case enumeration | Excellent | "What edge cases should I test for this date parser?" — AI will list 15 things you forgot |
| Test data generation | Excellent | Generate realistic fixtures, boundary values, invalid inputs |
| Integration test scaffolding | Good | Generate the boilerplate setup/teardown, you fill in the assertions |
| Contract / API tests | Good | Generate request/response pairs from an OpenAPI spec |
| Property-based test generators | Good | Define the invariant, let AI write the property test |
| E2E test scripts | Medium | Good starting point but often needs adjustment for timing, selectors, and real-world flow |
| Performance test scenarios | Medium | Can generate load profiles and JMeter/Gatling configs, but you need to validate the scenario matches real traffic |

**Example — test generation prompt:**

```
Here's a function that calculates shipping cost based on weight, distance,
and shipping tier (standard, express, overnight):

[paste function]

Generate JUnit 5 tests covering:
- Normal cases for each tier
- Boundary values (0 weight, max weight, 0 distance)
- Invalid inputs (negative weight, null tier)
- Rounding behavior for fractional costs

Follow the arrange-act-assert pattern. Use @ParameterizedTest where it makes the tests more readable.
```

**What to watch for:** AI-generated tests sometimes test the implementation rather than the behavior. If the test would still pass after you introduce a bug, it's a bad test. Review the assertions critically — are they actually checking the right thing?

### 4. Code Review

AI can be a useful first-pass reviewer, but it's not a replacement for a human reviewer who understands the system and the change's intent.

**Good uses:**
- Scan a diff for common issues: null checks, resource leaks, missing error handling, inconsistent naming
- Check whether a change follows your team's coding standards
- Identify missing test coverage for the changed code paths
- Summarize a large PR — "what does this PR actually do?" — before deep-diving

**Bad uses:**
- Sole reviewer on a PR (no human review)
- Architectural review — AI doesn't know if this change conflicts with work on another branch
- "Is this production-ready?" — AI can't assess operational risk

**Workflow:** Use AI as your pre-review check. Before requesting human review, run the diff through AI and fix the obvious issues. This raises the quality floor and lets human reviewers focus on design and correctness, not style nits.

### 5. Documentation

Documentation is one of the highest-ROI uses of AI because it takes something developers hate doing (writing docs) and makes it fast.

**What works well:**
- Generate API documentation from code (endpoint signatures, request/response examples)
- Convert inline comments and README bullet points into proper documentation
- Write first-draft runbooks: "Given this service's architecture, write an incident runbook for database connection exhaustion"
- Summarize a codebase or module for onboarding docs
- Generate changelog entries from git commit history
- Translate documentation between formats (wiki → Markdown, Swagger → human-readable)

**What to edit heavily:**
- AI documentation tends to be verbose and generic. Cut ruthlessly. If a sentence doesn't add information the reader needs, delete it.
- AI doesn't know what your team actually struggles with. Add the "here's what usually goes wrong" sections yourself — those are the most valuable parts of any doc.
- Check all code examples in generated docs. AI will generate plausible-looking examples that don't compile or use deprecated APIs.

### 6. Debugging and Incident Response

AI can accelerate debugging dramatically if you feed it the right context.

**Effective patterns:**
- Paste a stack trace and ask for the most likely root cause
- Paste error logs with timestamps and ask for pattern analysis
- "This query is slow — here's the explain plan — what's wrong?" — AI is good at reading explain plans
- "This test fails intermittently — here are the last 5 failure logs — what could cause flakiness?"

**What to provide for best results:**
1. The error or symptom (exact message, not paraphrased)
2. What you expected to happen
3. What actually happened
4. Relevant code (the function that failed, not the entire service)
5. Environment context (Java version, framework version, OS, etc.)

The more precise your description of the problem, the more precise the AI's analysis will be. "It doesn't work" gets you generic troubleshooting steps. "This Spring Boot 3.2 service throws `NoSuchBeanDefinitionException` for `OrderRepository` on startup after upgrading from 3.1, but only when the `test` profile is active" gets you a targeted answer.

### 7. Prioritization and Decision-Making

AI can help you think through priorities — not by making the decision, but by structuring the problem.

**Useful patterns:**
- "Here are 8 tech debt items. Help me rank them by: blast radius if left unfixed, effort to fix, and dependency on other work."
- "We have 3 weeks until the deadline and these 12 stories are left. Help me identify which are must-have vs. nice-to-have based on these acceptance criteria."
- Paste a list of bugs and ask AI to categorize by likely root cause — often reveals that 5 "different" bugs are the same underlying issue
- "What's the minimum viable scope to deliver this feature?" — AI can help you identify what to cut

**What AI cannot do:** Prioritize based on political context, team dynamics, stakeholder preferences, or organizational strategy. Those are human judgment calls.

### 8. Learning and Upskilling

AI is an exceptional learning tool — arguably better than Stack Overflow for conceptual understanding because you can have a conversation.

**Effective patterns:**
- "Explain how connection pooling works in HikariCP like I'm a senior engineer who's never configured it before"
- "What's the difference between optimistic and pessimistic locking? When would I use each?"
- "Walk me through this Kubernetes deployment YAML line by line — explain what each field does and what the defaults are"
- "I just read about the saga pattern. Give me a concrete example of where it would be better than a distributed transaction in a microservices architecture"
- Paste unfamiliar code and ask for an explanation — faster and more contextual than documentation

**The trap:** Don't use AI as a substitute for building real understanding. If you ask AI to explain something, make sure you can explain it back in your own words. If you can't, you didn't learn it — you just read an explanation.

---

## Prompt Engineering — Getting Better Output

The difference between a mediocre AI response and a great one is almost always the prompt. Prompt engineering isn't a gimmick — it's the skill of communicating clearly with a system that takes you literally.

### The Core Principles

#### 1. Be Specific About What You Want

```
# Bad — vague, gets a generic answer
Write a function to process orders.

# Good — specific constraints produce specific output
Write a Java method that takes a List<Order> and returns a Map<String, BigDecimal>
where the key is the customer ID and the value is the total order amount.
Orders with status CANCELLED should be excluded.
Use streams. Follow Google Java style.
```

#### 2. Provide Context and Constraints

```
# Bad — no context
How should I handle errors in my API?

# Good — grounded in your specific situation
I'm building a REST API in Spring Boot 3. We use a global @ControllerAdvice
for exception handling. Our current pattern maps domain exceptions to HTTP
status codes with a standard error response body (code, message, traceId).

I need to add handling for a new case: when an upstream service returns a
5xx error during order creation. Should I retry and then return 502 to the
caller, or return 503? What should the error response body include?
```

#### 3. Show Examples of What You Want (Few-Shot)

```
I need to write error messages for our API. Here's the style we use:

Example 1: "Order ORD-12345 not found. Verify the order ID and try again."
Example 2: "Cannot cancel order ORD-12345 — status is SHIPPED. Only orders
in PENDING or CONFIRMED status can be cancelled."

Now write error messages for:
- Duplicate order submission
- Payment method expired
- Shipping address outside service area
```

Few-shot examples are the single most effective prompting technique. They calibrate tone, format, length, and style better than any amount of description.

#### 4. Ask for the Format You Want

```
# Gets a wall of text
Explain the pros and cons of caching strategies.

# Gets a scannable, structured answer
Compare these caching strategies in a table with columns for:
Strategy | Best For | Consistency Risk | Complexity | When to Avoid

Strategies: cache-aside, read-through, write-through, write-behind, refresh-ahead
```

#### 5. Use Iterative Refinement

Don't expect the first response to be perfect. Use follow-ups to narrow:

```
Prompt 1: "Generate a Kafka consumer config for Spring Boot."
→ Gets a generic config

Prompt 2: "Good start. Now adjust for: 3 partitions, consumer group 'order-processor',
max.poll.records=50, manual offset commit, and a custom deserializer for our
OrderEvent class."
→ Gets a tailored config

Prompt 3: "Add error handling with retry topics (1 min, 5 min, 30 min) and a DLQ."
→ Gets the full production-ready config
```

Building up in layers is more reliable than asking for everything at once.

### Anti-Patterns in Prompting

| Anti-Pattern | Why It Fails | Fix |
|---|---|---|
| "Write me a microservice" | Way too broad — gets generic boilerplate | Break into small, specific tasks |
| "Make it better" | Better how? Faster? Cleaner? More readable? | Be specific: "reduce duplication" or "add null checks" |
| "Fix the bug" (with no context) | AI doesn't know what the bug is | Include the error, expected behavior, and actual behavior |
| Pasting 2000 lines of code | Exceeds useful context — AI loses focus | Paste only the relevant function and its dependencies |
| "Are you sure?" / "Think harder" | Doesn't actually improve output quality | Reframe the question with more constraints instead |
| Accepting first response | First response is often 70% right | Iterate: "This is close, but change X and Y" |

---

## When NOT to Use AI

AI is not universally appropriate. Know when to close the chat and think for yourself.

| Scenario | Why AI Is Wrong Here |
|---|---|
| **Security-critical implementations** | Crypto, auth flows, access control, input sanitization — AI can produce code that looks correct but has subtle vulnerabilities. These require expert review regardless of who (or what) wrote the code. |
| **Production incidents requiring fast judgment** | AI can help analyze logs, but the decision to roll back, failover, or page another team is yours. Don't waste incident time crafting prompts. |
| **Sensitive data handling** | Don't paste PII, credentials, or production data into AI tools. Even internal tools have logging and retention policies. |
| **Decisions that need accountability** | "The AI told me to" is not an acceptable post-mortem root cause. If you can't defend the decision independently, don't delegate it. |
| **When you don't understand the output** | If AI generates code you can't review, you can't ship it. Either learn enough to review it or ask a teammate who can. |
| **When exploring is the point** | Sometimes the value is in the struggle — debugging a tricky issue yourself builds understanding that skipping to the AI answer doesn't. Use judgment on when learning matters more than speed. |

---

## The Verification Habit

The single highest-value habit you can build: **always verify AI output before acting on it.**

### Verification Checklist by Output Type

| Output Type | How to Verify |
|---|---|
| **Generated code** | Does it compile? Do imports resolve? Do tests pass? Does it handle edge cases? Did it hallucinate an API? |
| **Factual claims** | Can you find a primary source (official docs, RFC, source code) that confirms this? |
| **Configuration** | Do the config keys exist in your version? Are the values within valid ranges? |
| **SQL queries** | Run EXPLAIN first. Check the results against known data. Verify it doesn't do a full table scan. |
| **Shell commands** | Read the command before running it. Especially anything with `rm`, `chmod`, `curl | bash`, or `>`. |
| **Dependency suggestions** | Does the library exist? Is it maintained? Is it approved for use in your organization? Check the actual npm/Maven registry. |
| **Explanations** | Does the explanation match the actual code/docs? AI confidently explains things that aren't true. |

### Hallucination Patterns to Watch For

AI doesn't "lie" — it generates plausible-sounding output based on patterns. But plausible isn't correct. Common hallucinations:

- **Invented methods or classes:** `StringUtils.isNotBlank()` exists. `StringUtils.containsOnlyAlphanumeric()` does not. AI will use both with equal confidence.
- **Wrong library versions:** "In Spring Boot 3, use `@EnableWebMvc`" — this was deprecated. The AI trained on older data.
- **Fabricated configuration keys:** `spring.kafka.consumer.auto-offset-reset=latest` is real. `spring.kafka.consumer.retry.max-attempts` is not (it's a Spring Retry config, not a Kafka consumer property).
- **Plausible but wrong logic:** The code compiles, the tests pass, but the business logic has a subtle error — an off-by-one in a boundary check, a missing timezone conversion, a race condition in concurrent access.
- **Confident wrong answers:** "This is thread-safe because `HashMap` handles concurrent access" — no, it does not. AI states this confidently because plenty of wrong code on the internet uses `HashMap` in concurrent contexts.

---

## Governance and Compliance — The Brief Version

AI doesn't exempt you from your existing responsibilities:

- **Code ownership:** You own what you commit. AI-generated code goes through the same review process as human-written code. "AI wrote it" is not a defense for a bug, a security hole, or a compliance violation.
- **IP and licensing:** Be cautious about AI reproducing verbatim code from open-source projects with incompatible licenses. If output looks suspiciously specific (complete function with original variable names and comments), verify it isn't copied from a GPL/AGPL project.
- **Data handling:** Don't paste production data, customer data, PII, credentials, or proprietary business logic into external AI tools. Internal AI coding tools may have different policies — know the boundary.
- **Audit trail:** For regulated systems, you may need to document that AI was used in development. Check your team's policy.
- **Security review:** AI-generated code should go through the same security scanning (Snyk, SonarQube) as human code. Don't assume AI output is safe.

---

## Productivity Multipliers — Patterns That Compound

These are the AI usage patterns that save the most time over weeks and months:

### 1. The Translator

Use AI to translate between formats you understand conceptually but are tedious to write by hand:

- SQL → ORM entity classes
- OpenAPI spec → TypeScript client SDK
- Protobuf → documentation
- Figma design → component scaffolding
- English requirements → Gherkin scenarios
- Error log → issue tracker ticket with repro steps

### 2. The Reviewer

Before submitting anything for human review, run it through AI:

- PR diff → first-pass review
- Tech design → blind spot check
- Documentation → clarity and completeness review
- Config change → sanity check ("Does this look right for a production Kafka cluster?")

This catches the easy stuff so human reviewers can focus on the hard stuff.

### 3. The Enumerator

Humans are bad at exhaustive enumeration. AI is good at it:

- "What are all the ways this API call can fail?"
- "List every state transition for an order in our system"
- "What edge cases should I consider for this date range query?"
- "What environment variables does this service need?"

### 4. The Explainer

Use AI to accelerate onboarding and unfamiliar-codebase exploration:

- "What does this module do? Summarize in 3 sentences."
- "Trace the flow from when a user clicks Submit to when the order is saved."
- "What's the relationship between these 4 classes?"
- Paste a complex regex or SQL query and ask for a plain-English explanation

### 5. The Automator

Use AI to write the automation you've been putting off:

- Shell scripts for repetitive tasks
- Git hooks for pre-commit validation
- CI/CD pipeline configs
- Database migration scripts
- Cron jobs with proper error handling and logging

---

## Common Mistakes

### 1. Copy-Paste Without Reading

The most dangerous pattern. AI generates 40 lines of code, you paste it in, it compiles, the test passes, you ship it. Two weeks later, it breaks in production because of an edge case the AI didn't handle and you didn't read closely enough to catch.

**Fix:** Read every line. If a section isn't clear, ask the AI to explain it before you commit it.

### 2. Using AI as a Crutch Instead of a Tool

If you use AI for every task — including ones you should be able to do yourself — your skills atrophy. AI should make you faster at things you understand, not replace understanding.

**Fix:** If you couldn't solve the problem without AI (even slowly), take the time to understand the AI's solution. Use it as a learning opportunity.

### 3. Over-Engineering Because AI Makes It Easy

AI will happily generate a Strategy pattern with a factory, a builder, and three levels of abstraction for what should be an `if` statement. Just because AI can generate complex code quickly doesn't mean complex code is the right answer.

**Fix:** Always ask "is this the simplest thing that works?" AI tends toward over-engineering because training data is full of over-engineered examples.

### 4. Trusting AI on Your Dependencies' APIs

AI frequently hallucinates methods, parameters, and configuration keys — especially for libraries that have changed between major versions. It trained on Stack Overflow answers from 2019, but you're running the 2025 version.

**Fix:** Always verify API calls against the official documentation for your specific version. `cmd+click` into the source. Check the method signature.

### 5. Providing Too Little Context

"Write a service" or "fix this bug" with no context produces generic, unhelpful output. The AI fills in the blanks with assumptions, and its assumptions are almost never your specific situation.

**Fix:** Include your tech stack, constraints, existing patterns, error messages, and what you've already tried. More context = better output.

### 6. Providing Too Much Context

Pasting your entire codebase into a prompt is just as bad as too little. The AI loses focus, misses what's important, and may fixate on irrelevant details.

**Fix:** Paste only the relevant code. If the AI needs more, it'll ask (or you'll notice the output is wrong because it's missing context — that's your cue to add the right file).

### 7. Not Iterating

Accepting the first response as final. AI conversations are iterative — the first answer is a rough draft. The value comes from refining.

**Fix:** Treat AI output as a starting point. "This is close, but adjust X" is a perfectly valid follow-up. Two or three rounds of refinement typically produces significantly better output than one shot.

### 8. Asking AI to Make Decisions It Shouldn't

"Should we use microservices or a monolith?" AI will give you a confident answer. It will be a reasonable generic answer. It will not account for your team size, operational maturity, deployment infrastructure, or the fact that your last microservices migration took 18 months and burnt out half the team.

**Fix:** Use AI to enumerate options and trade-offs. Make the decision yourself.

---

## Quick Reference Checklist

- [ ] **AI is a tool, not an authority** — verify every output before acting on it
- [ ] **Read every line** of generated code before committing. If you can't explain it, don't ship it
- [ ] **Provide specific context** in prompts: tech stack, constraints, existing patterns, error messages
- [ ] **Use few-shot examples** when you want output in a specific style or format
- [ ] **Iterate on responses** — first draft is rarely the final answer, refine in 2-3 rounds
- [ ] **Check for hallucinated APIs** — verify imports resolve and methods exist in your dependency version
- [ ] **Don't paste sensitive data** — no PII, credentials, production data, or proprietary logic into external tools
- [ ] **AI-generated code gets the same review process** as human-written code — PR review, security scan, tests
- [ ] **Use AI for enumeration** — edge cases, failure modes, test scenarios, state transitions
- [ ] **Break large tasks into small prompts** — one function, one test, one config at a time
- [ ] **Test AI-generated code** — compile, run, check edge cases, verify behavior matches intent
- [ ] **Don't over-engineer** — "AI can generate it quickly" doesn't mean complex code is the right answer
- [ ] **Use AI to translate** formats you understand but are tedious to write (SQL → ORM, spec → types, logs → tickets)
- [ ] **Pre-review with AI** before requesting human review — catch the easy stuff early
- [ ] **Build understanding, not dependency** — if you can't solve it without AI, use the solution as a learning opportunity
- [ ] **Know when to close the chat** — security-critical code, production incidents, and accountability decisions are yours
