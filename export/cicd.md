# CI/CD Best Practices

A broken CI/CD pipeline doesn't just slow you down — it erodes trust. Engineers stop trusting tests that flake. They stop trusting deploys that break. They start doing things manually, and manual means inconsistent. The goal of CI/CD isn't automation for automation's sake. It's to make shipping safe, fast, and boring.

**Goal: Every commit should be a candidate for production. Your pipeline's job is to prove it — or reject it — fast.**

---

## 1. Continuous Integration vs. Continuous Delivery vs. Continuous Deployment

These three terms get thrown around interchangeably, but they mean different things.

| Term | What It Means | Implies |
|------|--------------|---------|
| **Continuous Integration (CI)** | Every developer integrates code into a shared branch frequently (at least daily). Each integration triggers an automated build and test. | Fast feedback, small changes, no long-lived branches |
| **Continuous Delivery (CD)** | Every commit that passes the pipeline *could* go to production. Deployment is a manual decision. | Production-ready artifacts at all times |
| **Continuous Deployment** | Every commit that passes the pipeline *does* go to production automatically. No human gate. | Full pipeline confidence, mature test suites, feature flags |

Most teams should aim for **Continuous Delivery** first. Continuous Deployment is a maturity milestone, not a starting point.

---

## 2. Branching Strategy

Your branching model dictates how code flows into your pipeline. Pick wrong and your CI/CD is fighting your workflow instead of enabling it. For services, trunk-based development is the recommended default — DORA research consistently shows it correlates with elite delivery performance. GitFlow is better suited for versioned products like libraries and SDKs.

See Version Control for branching strategy details.

---

## 3. Pipeline Anatomy — What Runs When

A well-designed pipeline is a funnel: fast, cheap checks first. Slow, expensive checks last. Fail fast, fail early.

### Stage Order

```
┌─────────┐    ┌──────────┐    ┌───────────┐    ┌──────────┐    ┌──────────┐    ┌────────┐
│  Lint /  │───▶│  Build   │───▶│   Unit    │───▶│ Integra- │───▶│ Security │───▶│ Deploy │
│  Format  │    │ Compile  │    │  Tests    │    │  tion    │    │  Scans   │    │        │
└─────────┘    └──────────┘    └───────────┘    └──────────┘    └──────────┘    └────────┘
   ~30s           ~1-3m           ~1-5m            ~3-10m          ~2-5m         varies
```

### What Triggers What

Not every pipeline stage needs to run on every event. Be intentional.

| Trigger | What Should Run | Why |
|---------|----------------|-----|
| **Push to feature branch** | Lint, build, unit tests | Fast feedback for the developer. Don't waste resources on heavy tests for WIP code. |
| **Pull request opened/updated** | Lint, build, unit tests, integration tests, security scans, coverage | This is your quality gate. Everything that proves "this is safe to merge." |
| **Merge to main/trunk** | Full build, all tests, artifact publish, deploy to dev/staging | The code is now shared truth. Build the real artifact and start promoting it. |
| **Tag / release** | Build from tag, publish artifact, deploy to production | This is the ship event. Build from the immutable tag. |
| **Scheduled (nightly)** | Full E2E suite, performance tests, dependency vulnerability scans | Expensive tests that don't need to block every PR but need to run regularly. |

### The Golden Rule of Pipeline Triggers

**Build the artifact once, deploy it many times.** Don't rebuild for each environment. Build on merge to trunk, publish the artifact (container image, JAR, bundle), and promote that same artifact through environments. If you're recompiling for staging and again for production, you're not testing what you're shipping.

---

## 4. Build Stage

### Best Practices

- **Deterministic builds.** Same commit should produce the same artifact every time. Pin dependency versions. Avoid `latest` tags in base images. Use lock files (`package-lock.json`, `gradle.lockfile`, `go.sum`).
- **Cache aggressively.** Cache dependencies, build layers, and compilation outputs between runs. A clean build every time is wasteful.
- **Keep builds under 5 minutes.** If your build takes longer, investigate: are you building things you don't need? Can you parallelize compilation? Are your Docker layers ordered correctly (least-changing layers first)?
- **Version your artifacts.** Every artifact should be traceable to a commit. Use `git SHA`, semver, or both. `my-service:a3f8b2c` is infinitely more useful than `my-service:latest`.
- **Fail the build on warnings.** Compiler warnings, deprecation notices, linting violations — treat them as errors in CI. Warnings that nobody reads are just noise.

### Docker Build Optimization

```dockerfile
# Good: ordered from least to most frequently changing
FROM eclipse-temurin:21-jre-alpine

# OS-level deps change rarely
RUN apk add --no-cache curl

# Dependencies change sometimes
COPY build/libs/dependencies/ /app/libs/

# Application code changes often
COPY build/libs/my-service.jar /app/

ENTRYPOINT ["java", "-jar", "/app/my-service.jar"]
```

---

## 5. Testing in the Pipeline

The key insight for testing in CI/CD is that not all tests belong at every stage. Structure your pipeline as a funnel: fast, cheap checks (linting, unit tests) run on every push, while heavier checks (integration, E2E, performance) run at later gates. Be opinionated about what blocks a merge versus what is informational.

See Testing for testing strategy details.

---

## 6. Artifact Management

### What Counts as an Artifact

Anything your pipeline produces that gets deployed or consumed downstream:

- Container images
- JARs / WARs / fat JARs
- npm packages
- Compiled binaries
- Terraform plans
- Database migration bundles

### Rules

1. **Immutable artifacts.** Once published, an artifact version is never overwritten. `v1.2.3` means the same thing forever.
2. **Signed artifacts.** Know who built it and that it hasn't been tampered with. Container image signing (cosign, Notation) should be table stakes.
3. **Retention policy.** Don't keep everything forever. Keep the last N production versions and delete the rest. Set this up day one — cleaning up 50,000 stale images later is painful.
4. **Metadata.** Tag artifacts with: git SHA, branch, build timestamp, build number. You'll need this at 2 AM when debugging production.
5. **Separate registries for snapshots vs. releases.** Snapshot/dev artifacts are disposable. Release artifacts are not. Different retention, different access controls.

---

## 7. Environment Promotion Strategy

Artifacts should flow through a defined promotion path (e.g., dev → staging → pre-prod → production), with explicit pass/fail gates at each transition. The closer your lower environments are to production, the more trustworthy your validation is — differences between environments are where bugs hide.

See Environments for environment strategy details.

---

## 8. Deployment Strategies

No single deployment strategy fits every situation. The main options — rolling, blue-green, and canary — each have different trade-offs around rollback speed, infrastructure cost, and risk exposure. Canary is the recommended default for most production services due to its minimal blast radius and data-driven promotion.

See Releasing for deployment strategy details.

---

## 9. Deployment Frequency and Release Cadence

### How Often Should You Deploy?

As often as your pipeline lets you, safely.

| Frequency | Maturity Signal | Prerequisites |
|-----------|----------------|---------------|
| **Multiple times per day** | Elite | Trunk-based dev, feature flags, automated canary, comprehensive tests, observability |
| **Daily** | High | Good test coverage, automated deploys, canary or blue-green |
| **Weekly** | Medium | Reasonable test coverage, semi-automated deploys |
| **Biweekly / Monthly** | Low | Manual QA cycles, change approval boards, batched releases |

**The paradox of deployment frequency:** Deploying more often is *safer*, not riskier. Small changes are easier to understand, easier to test, easier to roll back, and easier to debug when they break. A deploy with 3 commits is a focused change. A deploy with 300 commits is a mystery box.

### Release Cadence Rules

1. **Deploy on business days.** Friday deploys are fine *if* you have automated canary analysis and instant rollback. If you're manually watching dashboards, don't deploy on Friday.
2. **Avoid deploy freezes longer than 2 weeks.** Freeze periods create a backlog of changes that all deploy at once — the opposite of safe, incremental delivery.
3. **Decouple deploy from release.** Deploy code frequently. Release features with feature flags. "Deploy" is a technical event. "Release" is a business event. They don't have to be the same.
4. **Have a deploy calendar for shared environments.** If 5 teams share staging, don't let them stomp on each other's deploys. Coordinate or give each team their own namespace.

---

## 10. Rollbacks

Deploys will fail. Plan for it. Every deploy should have a documented rollback plan, and rollback should be practiced regularly — not discovered for the first time during an incident. Options include redeploying the previous version, traffic switching (blue-green), feature flag toggles, or forward fixes.

See Releasing for full rollback strategy details.

---

## 11. Pipeline Performance

Slow pipelines kill productivity and encourage bad habits (skipping CI, pushing directly, batching changes).

### Speed Targets

| Stage | Target | Unacceptable |
|-------|--------|-------------|
| Lint + static analysis | < 1 min | > 3 min |
| Build + compile | < 3 min | > 10 min |
| Unit tests | < 5 min | > 10 min |
| Integration tests | < 10 min | > 20 min |
| **Total PR pipeline** | **< 15 min** | **> 30 min** |
| Full pipeline (to deployable artifact) | < 20 min | > 45 min |

### Optimization Strategies

1. **Parallelism.** Run independent stages concurrently. Unit tests and linting don't depend on each other — run them in parallel.
2. **Caching.** Cache dependencies, Docker layers, compiled outputs. A cache miss should be the exception, not the rule.
3. **Test parallelism.** Split your test suite across multiple runners. Most CI systems support this natively.
4. **Skip what hasn't changed.** In a monorepo, only build and test the packages affected by the changeset. Use dependency-aware build tools.
5. **Fail fast ordering.** Put the fastest checks first. If linting fails in 15 seconds, don't wait 8 minutes for the build to also tell you about a formatting issue.
6. **Right-size your runners.** If your build is CPU-bound, give it more cores. If it's I/O-bound, give it faster disks. Don't guess — profile.
7. **Measure and track.** Pipeline duration should be a metric you monitor. Set alerts for regressions. "The build got 3 minutes slower" should trigger investigation, not acceptance.

---

## 12. Pipeline Security

Security isn't a separate phase — it's woven into every stage.

### Security in the Pipeline

| Stage | Security Check | Tools (Examples) |
|-------|---------------|-----------------|
| **Pre-commit** | Secrets detection | gitleaks, detect-secrets |
| **Build** | Dependency vulnerability scan (SCA) | Snyk, OWASP Dependency-Check, Trivy |
| **Build** | Static analysis (SAST) | SonarQube, Semgrep, CodeQL |
| **Build** | Container image scan | Trivy, Snyk Container |
| **Deploy** | Infrastructure-as-code scan | Checkov, tfsec |
| **Post-deploy** | Dynamic analysis (DAST) | OWASP ZAP, Burp Suite |

### Security Gating Rules

- **Block on:** Known critical/high CVEs in direct dependencies, hardcoded secrets, SQL injection patterns
- **Warn on:** Medium CVEs, transitive dependency vulnerabilities, informational findings
- **Don't block on:** Low/informational findings (fix them, but don't break the pipeline)

---

## 13. Configuration Management in CI/CD

Configuration is the #1 source of "it works in staging but not in production" bugs.

### Rules

1. **Config is not code.** Don't bake environment-specific config into your artifact. Inject it at deploy time.
2. **Same artifact, different config.** The container image deployed to staging and production should be byte-for-byte identical. Only configuration differs.
3. **Validate config in the pipeline.** If your service reads from a centralized configuration service, validate the schema in CI. Don't find out about a typo in production.
4. **Secrets are never in source control.** Use a secrets manager. Rotate credentials regularly. Audit access.
5. **Feature flags are config.** Treat them with the same rigor — version them, audit changes, have a process for cleanup.

---

## 14. Common Mistakes

### Mistake 1: Rebuilding Artifacts Per Environment

**Bad:** Building a JAR for dev, a different JAR for staging, and another for production.

**Why it's bad:** You're not testing what you're shipping. The production build could have different behavior due to build-time differences.

**Fix:** Build once on merge to trunk. Promote the same artifact through every environment. Inject config at runtime.

### Mistake 2: Treating CI as "Run Tests on a Branch"

**Bad:** CI only runs tests. No linting, no security scans, no artifact publishing, no enforcement.

**Why it's bad:** You're using 10% of what a pipeline can do. Tests alone don't prove deployability.

**Fix:** A complete pipeline includes: lint → build → test (multiple levels) → scan → publish → deploy.

### Mistake 3: Long-Lived Feature Branches with Stale CI

**Bad:** A feature branch that's been open for 3 weeks. CI passed when it was created. It hasn't been rebased since.

**Why it's bad:** The CI result is meaningless. Trunk has moved on. The merge will be painful and risky.

**Fix:** Rebase or merge trunk into your branch daily. Better yet, use trunk-based development with feature flags.

### Mistake 4: No Pipeline for Infrastructure Changes

**Bad:** Application code has a full CI/CD pipeline. Terraform, Kubernetes manifests, and config changes are applied manually.

**Why it's bad:** Infrastructure drift, unreproducible environments, no audit trail, no rollback path.

**Fix:** Infrastructure-as-code with its own pipeline: lint → plan → review → apply. Same rigor as application code.

### Mistake 5: Manual Deployments "Because It's Faster"

**Bad:** SSH into a server and deploying manually because "the pipeline is slow" or "it's just a quick fix."

**Why it's bad:** No audit trail, no tests, no rollback, no reproducibility. It's faster until it breaks, and then it's a nightmare.

**Fix:** Fix the pipeline speed. If deploys are slow, invest in making them fast. Never optimize by removing safety.

### Mistake 6: Deploying Without Health Checks

**Bad:** The deploy pipeline finishes and reports success without verifying the new version is actually serving traffic correctly.

**Why it's bad:** "Deployed successfully" means the binary is running, not that it's working. A service can start up and immediately fail on every request.

**Fix:** Post-deploy smoke tests and health checks are mandatory. The pipeline isn't done until the new version is healthy.

### Mistake 7: Skipping CI "Just This Once"

**Bad:** Force-pushing to main, bypassing branch protection, or marking failing checks as passed to unblock a deploy.

**Why it's bad:** "Just this once" happens every sprint. Every shortcut normalizes the next one.

**Fix:** If CI is in the way of a legitimate urgent fix, fix CI. Hotfix branches with a reduced (but non-zero) test suite are fine. Bypassing everything is not.

### Mistake 8: No Rollback Strategy

**Bad:** "We'll figure out rollback if we need it."

**Why it's bad:** You will need it. At 2 AM. Under pressure. With stakeholders watching.

**Fix:** Define and test rollback before the first deploy. Automate it. Practice it.

---

## 15. Quick Reference Checklist

### Pipeline Design
- [ ] Pipeline stages are ordered from fast/cheap to slow/expensive
- [ ] PR pipeline completes in under 15 minutes
- [ ] Artifacts are built once and promoted through environments
- [ ] Every artifact is traceable to a specific commit
- [ ] Independent stages run in parallel

### Testing
- [ ] Unit tests run on every push
- [ ] Integration tests run on PRs and merge to trunk
- [ ] E2E smoke tests run post-deploy
- [ ] Full E2E and performance tests run nightly
- [ ] Flaky tests are tracked and fixed (not just retried)
- [ ] Diff coverage is measured on PRs

### Security
- [ ] Secrets scanning runs pre-commit or in CI
- [ ] Dependency vulnerability scanning (SCA) runs on every PR
- [ ] Static analysis (SAST) runs on every PR
- [ ] Container images are scanned before promotion
- [ ] Critical/high findings block the pipeline

### Deployment
- [ ] Deployment strategy is documented (canary / blue-green / rolling)
- [ ] Health checks verify the deploy actually works
- [ ] Rollback is automated or one-click
- [ ] Rollback has been tested (not just theoretically possible)
- [ ] Database migrations are backward-compatible
- [ ] Deploy and release are decoupled (feature flags)

### Environments
- [ ] Environment promotion path is defined (dev → staging → pre-prod → prod)
- [ ] Promotion gates have explicit pass/fail criteria
- [ ] All environments are provisioned from code (IaC)
- [ ] Config is injected at deploy time, not baked into artifacts
- [ ] Environment parity is actively maintained

### Process
- [ ] Branching strategy is documented and followed
- [ ] Deploy frequency is tracked as a team metric
- [ ] Pipeline duration is monitored with alerts for regressions
- [ ] Rollback drills happen at least quarterly
- [ ] No manual steps exist between merge and production
