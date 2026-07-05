# A/B Testing

## What Is A/B Testing?

A/B testing (also called split testing) is a controlled experiment where two or more variants of a page, feature, or experience are shown to different segments of users simultaneously. Traffic is randomly split between a **control** (the current version, "A") and one or more **treatments** (the new version(s), "B", "C", etc.). Statistical analysis is then used to determine which variant performs better against a predefined metric.

At its core, A/B testing replaces opinion-driven decisions with data-driven evidence. Instead of debating whether a green or blue button will convert better, you measure it.

## When to Use A/B Testing

A/B testing is the right tool when:

- **You have a clear hypothesis** — "Changing X will improve metric Y by Z%."
- **You can measure the outcome** — The target metric is quantifiable (conversion rate, click-through rate, revenue per session, latency, error rate, etc.).
- **You have sufficient traffic** — The experiment needs enough users to reach statistical significance in a reasonable timeframe.
- **The change is reversible** — If the treatment performs poorly, you can roll back without lasting damage.
- **You want to isolate causation** — Observational data can show correlation, but only a randomized experiment proves that your change *caused* the improvement.

### Common Use Cases

| Scenario | Example |
|---|---|
| UI/UX changes | Button color, layout, copy, checkout flow |
| Feature launches | Rolling out a new recommendation algorithm |
| Backend changes | Comparing two ranking models or search relevance strategies |
| Pricing & promotions | Testing discount structures or free-shipping thresholds |
| Performance tuning | Measuring the business impact of a latency reduction |

### When NOT to Use A/B Testing

- **Extremely low traffic** — You won't reach significance. Consider qualitative research instead.
- **Irreversible changes** — Database migrations or breaking API changes aren't candidates for split testing.
- **Ethical concerns** — Don't experiment on experiences that could cause harm to a user segment (e.g., withholding critical safety information).
- **No clear metric** — If you can't define what "better" means, you can't measure it.

## Best Practices

### 1. Start with a Strong Hypothesis

Never run a test "just to see what happens." Frame it as:

> "If we [change], then [metric] will [improve/decrease] because [reasoning]."

A good hypothesis is specific, measurable, and grounded in user research or data.

### 2. Define Your Primary Metric Before You Start

Pick **one** primary metric (the "Overall Evaluation Criterion") before the experiment launches. You can track secondary metrics, but your go/no-go decision should hinge on one metric. Choosing the metric after seeing results is p-hacking.

### 3. Calculate Sample Size in Advance

Use a power analysis to determine how many users you need. Key inputs:

- **Baseline conversion rate** — What is the current metric value?
- **Minimum detectable effect (MDE)** — What is the smallest improvement worth detecting?
- **Statistical significance level (alpha)** — Typically 0.05 (5% false-positive rate).
- **Statistical power (1 - beta)** — Typically 0.80 (80% chance of detecting a real effect).

Running a test without a sample size calculation often leads to either stopping too early (false positives) or running too long (wasted opportunity cost).

### 4. Randomize Properly

Randomization must be **consistent** (same user sees the same variant across sessions), **uniform** (traffic split matches your intended allocation), and **independent** (one experiment's assignment doesn't leak into another). Use a hash of user ID, not random per-request assignment. See Consistent vs Inconsistent Pass Rate for assignment strategy details.

Most experimentation platforms (GrowthBook, LaunchDarkly, Optimizely, etc.) handle this, but verify it during setup.

### 5. Avoid Peeking and Early Stopping

Looking at results daily and stopping as soon as you see significance inflates your false-positive rate dramatically. Instead:

- Commit to a **fixed sample size** or **fixed duration** before launch.
- If you must monitor continuously, use **sequential testing** methods (e.g., always-valid p-values or Bayesian approaches) that are designed for it.

### 6. Run the Test Long Enough

Even if you hit your sample size quickly, run the test for at least **one full business cycle** (typically 1-2 weeks) to account for:

- Day-of-week effects (weekday vs. weekend behavior)
- Novelty effects (users reacting to change, not the change itself)
- External events (promotions, holidays, competitor actions)

### 7. Test One Change at a Time

Changing the button color *and* the headline *and* the layout simultaneously makes it impossible to attribute the result. If you need to test multiple changes together, use **multivariate testing** (MVT) with proper factorial design — but be aware this requires significantly more traffic.

### 8. Segment, but Don't Fish

Pre-define segments you want to analyze (e.g., mobile vs. desktop, new vs. returning users). Slicing results by every possible dimension after the fact will produce false positives. If you do exploratory segmentation, treat findings as hypotheses for future tests, not conclusions.

### 9. Watch for Guardrail Metrics

Beyond your primary metric, monitor guardrail metrics that should *not* degrade:

- Page load time
- Error rates
- Cart abandonment
- Customer support contacts

A treatment that boosts conversions but doubles error rates is not a win.

### 10. Document Everything

For every experiment, record:

- **Hypothesis** — What you expected and why.
- **Design** — Traffic split, audience, duration, variants.
- **Results** — Primary metric, secondary metrics, confidence intervals.
- **Decision** — Ship, iterate, or kill — and the reasoning.
- **Learnings** — What did you learn even if the test was inconclusive?

This institutional knowledge compounds over time and prevents teams from re-running the same failed experiments.

### 11. Account for Multiple Comparisons

If you test more than two variants (A/B/C/D), the probability of a false positive increases. Apply corrections such as:

- **Bonferroni correction** — Divide alpha by the number of comparisons.
- **False Discovery Rate (FDR)** — Controls the expected proportion of false discoveries.

### 12. Beware of Common Pitfalls

| Pitfall | Why It's Dangerous |
|---|---|
| **Sample Ratio Mismatch (SRM)** | If your 50/50 split is actually 52/48, something is wrong with assignment. Investigate before trusting results. |
| **Survivorship bias** | Only analyzing users who completed a flow ignores those who dropped off. |
| **Interaction effects** | Running multiple experiments on the same user pool can cause interference. Use mutual exclusion or layered experiments. |
| **HiPPO effect** | The "Highest Paid Person's Opinion" overriding data. Let the numbers speak. |

## Key Terminology

| Term | Definition |
|---|---|
| **Control** | The existing experience (variant A) |
| **Treatment** | The new experience being tested (variant B) |
| **Statistical significance** | The probability that the observed difference is not due to chance (typically p < 0.05) |
| **Confidence interval** | The range within which the true effect likely falls |
| **Power** | The probability of detecting a real effect when one exists |
| **MDE** | Minimum Detectable Effect — the smallest change worth detecting |
| **p-value** | The probability of observing results at least as extreme as the actual results, assuming the null hypothesis is true |
| **Novelty effect** | A temporary spike in engagement simply because something is new |

## Summary

A/B testing is one of the most powerful tools for making evidence-based product decisions. The methodology is straightforward — split traffic, measure outcomes, pick the winner — but the discipline required to do it correctly is often underestimated. Define your hypothesis, calculate your sample size, resist the urge to peek, and document your learnings. Done well, it transforms product development from guesswork into science.

---

## Related Documents

- Consistent vs Inconsistent Pass Rate — why consistent (sticky) assignment is essential for valid A/B experiments
- Feature Flags — experiment flags as a flag type, lifecycle management, and rollout best practices
