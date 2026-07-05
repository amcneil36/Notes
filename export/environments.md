# Environment Best Practices

An "environment" is an isolated deployment of your application — its own infrastructure, config, data, and URL. The question of how many to have is one of the most underrated architectural decisions on a team.

---

## Do You Even Need Multiple Environments?

Not every app does. The cost of multiple environments is real: more infra to maintain, more configs to sync, more pipelines to run, more bugs that only reproduce "in staging." Be honest about whether the complexity pays for itself.

### Stick with just Production when:

- The app is internal-only with a small, trusted user base (e.g., a 5-person ops tool)
- There is no customer-facing risk — mistakes are quickly reversible
- The team is 1-2 developers and the velocity is slow
- The app has no external integrations that could cause real-world side effects
- Deploys are infrequent and manually reviewed before going out

**The trap:** Teams add environments out of habit or policy, not need. A `staging` environment that nobody uses — or that is perpetually broken — provides zero value and 100% of the cost.

### Add environments when:

- Bugs in production affect real customers or revenue
- The app has external integrations (payment processors, email, SMS, Kafka, partner APIs) where test traffic must be isolated
- You have regulatory or compliance requirements (SOX, PCI, HIPAA)
- Multiple developers are merging code simultaneously — you need a shared integration point
- You need to validate infrastructure changes (migrations, config changes) before they hit prod
- QA or product owners need a stable place to verify features before release
- You run load or performance tests that would impact real users

---

## How Many Environments?

The standard answer is **3** for most teams. Some teams need 4. Rarely more than that.

### The "why not more" argument

Each additional environment:
- Multiplies your infrastructure cost
- Adds a pipeline stage that can block deployments
- Drifts from production over time (the longer it exists, the less it resembles prod)
- Requires someone to maintain, monitor, and keep stable
- Creates a "works in staging, fails in prod" bug surface

More environments ≠ more safety. A well-maintained 3-environment setup beats a neglected 5-environment one every time.

---

## The Standard Environment Stack

### 1. Development (Dev)

**Purpose:** Active development. Fast, unstable, throwaway.

- Developers deploy here constantly — sometimes multiple times per hour
- It is expected to be broken. Nobody should rely on it for anything stable.
- Often runs locally or in a personal cloud namespace (e.g., a Kubernetes dev namespace per developer)
- Uses mocked or sanitized data, never real customer data
- External integrations point to sandbox/mock endpoints
- No SLA. If it goes down, nobody cares.

**Who uses it:** Developers only.

**Key rule:** Dev should be cheap to reset. If it's hard to recreate, something went wrong with your infrastructure-as-code.

---

### 2. Staging (Pre-Production)

**Purpose:** Production mirror. Validation before release.

This is the most important non-prod environment. It should be as close to production as humanly possible.

- Same infrastructure size class as prod (or a scaled-down but structurally identical version)
- Same config management approach (centralized configuration service, environment variables, secrets manager)
- Same external integrations — but pointed at vendor sandbox/test accounts, not live
- Same deployment pipeline as prod (if it can't deploy to staging, it can't deploy to prod)
- Stable enough for QA, product, and stakeholders to use for validation
- Seeded with realistic but anonymized/synthetic data

**Who uses it:** QA engineers, product owners, developers validating pre-release, automated integration/E2E tests.

**Key rule:** Staging should never be treated as a dumping ground. A broken staging environment that nobody fixes is worse than no staging at all — it erodes trust in the validation process and developers start skipping it.

---

### 3. Production (Prod)

**Purpose:** Real users. Real data. Real consequences.

- All changes must pass dev → staging before reaching here
- Deployments are gated (automated tests must pass, sometimes manual approval)
- Monitored 24/7 with alerts
- No manual config changes directly in prod — all changes come through the pipeline
- Real external integrations (live payment processors, real Kafka topics, real partner APIs)
- Backups, disaster recovery, and runbooks in place

**Who uses it:** Real users (and oncall engineers debugging incidents).

**Key rule:** Prod is not a testing environment. If you find yourself saying "let's just test it in prod," that's a signal your staging environment isn't trustworthy.

---

## The 4-Environment Stack (When You Need It)

Some teams add a fourth environment. The most common additions:

### Option A: Local / Personal Dev Namespace

Separate from a shared "dev" — each developer has their own isolated instance. Useful when:
- Multiple developers are working on conflicting features simultaneously
- You don't want one developer's broken deploy to block another's

On a Kubernetes platform this often looks like a personal namespace (`dev-jsmith`) alongside a shared `dev` namespace.

### Option B: QA / Test

A stable environment owned by the QA team, separate from staging. Useful when:
- QA runs long regression cycles that shouldn't be interrupted by developer deploys
- Staging is used for ongoing product demos and needs to stay stable
- You have a dedicated QA team with their own release cadence

### Option C: Performance / Load Test

A dedicated environment that mirrors prod sizing, used exclusively for load testing. Useful when:
- Load tests would affect shared staging stability
- You need accurate performance benchmarks (requires prod-equivalent infra)
- Compliance requires documented load test results before release

### Option D: UAT (User Acceptance Testing)

A business-facing environment where business stakeholders or clients validate features before release. Common in enterprise or B2B software. Useful when:
- External clients must sign off on features before they go live
- There are contractual SLAs around testing windows

---

## Environment Parity — The Most Important Principle

**The closer your non-prod environments are to production, the more trustworthy they are.**

Every difference between staging and prod is a potential "works in staging, fails in prod" bug. Common parity failures:

| What differs | What goes wrong |
|---|---|
| Staging uses SQLite, prod uses PostgreSQL | SQL dialect bugs go undetected |
| Staging has 1 instance, prod has 10 | Race conditions only appear in prod |
| Staging skips auth middleware | Auth bugs go to prod |
| Staging uses stub data, prod has 10M records | Performance issues only appear in prod |
| Staging config managed manually, prod via centralized config | Config drift causes prod-only failures |
| Staging on old infra version | Infrastructure-level bugs missed |

**Rule of thumb:** If you wouldn't run it in prod, don't run it in staging. If staging can't run what prod runs, staging can't validate what prod will do.

---

## Configuration Management Across Environments

Each environment needs its own config, but the structure should be identical — only the values differ.

### Do

- Use a centralized config system (runtime configuration service, AWS Parameter Store, Vault) per environment
- Store config in the same format across all environments — only values differ
- Version your config alongside your code where possible
- Use environment-specific secrets, never shared credentials across environments
- Promote config changes through the same pipeline as code changes

### Don't

- Hardcode environment names or URLs in application code (`if (env == "staging")`)
- Share API keys, database credentials, or service accounts between environments
- Manually edit prod config directly — always go through the pipeline
- Let non-prod environments have access to prod data stores

---

## Data Strategy Per Environment

| Environment | Data Approach |
|---|---|
| Local/Dev | Synthetic seed data, local DB, reset freely |
| Staging | Anonymized/synthetic copy of prod-shaped data, periodic refresh |
| Perf/Load | Prod-volume synthetic data to simulate real load |
| Production | Real customer data, backups required, access controlled |

**Never copy real production data to non-prod environments without anonymization.** This is a compliance issue (GDPR, CCPA, internal data classification policies) and a security issue.

---

## Environment Promotion Flow

Code should flow in one direction only: dev → staging → prod. Never backwards.

```
Feature branch
      │
      ▼
   Dev/Local  ◄── developer testing, fast iteration
      │
      ▼
   Staging    ◄── QA validation, integration tests, product sign-off
      │
      ▼
  Production  ◄── real users, monitored, gated deploy
```

**No hotfix shortcuts** that bypass staging unless you have a documented break-glass procedure with mandatory post-incident review. Every skip of staging is a debt against your trust in the pipeline.

---

## Signs Your Environment Strategy Has Gone Wrong

- "It works in staging but not prod" — parity problem
- "Nobody uses staging anymore" — it's broken or untrustworthy
- "We have 6 environments and I'm not sure what they're all for" — sprawl
- "We test in prod because staging is too slow to deploy to" — pipeline problem
- "Staging has the production database password" — security problem
- Developers are maintaining personal long-lived environments instead of ephemeral ones — cost and drift problem

---

## Quick Reference

| Environment | Namespace Example | Purpose | Who Uses It |
|---|---|---|---|
| Local Dev | `dev-{userid}` Kubernetes namespace | Active feature development | Individual developer |
| Shared Dev | `dev` Kubernetes namespace | Integration testing, shared demos | Dev team |
| Staging | `staging` Kubernetes namespace | Pre-release validation, QA, E2E tests | QA, product, developers |
| Production | `prod` Kubernetes namespace | Real users | Users + oncall |

Keep Kafka topics, configuration groups, and database schemas namespaced per environment. Never let a staging Kafka consumer group read from a prod topic.
