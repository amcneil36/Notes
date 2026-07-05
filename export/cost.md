# Software Cost Best Practices

Every line of code you ship has a price tag — you just can't see it. Infrastructure costs are the silent second codebase: they grow with every feature, every dependency, every "just add another instance" decision. And unlike bugs, cost problems compound quietly until someone gets a bill that makes them question the entire architecture.

Most teams don't ignore cost intentionally. They ignore it because it's invisible. By the time the monthly report arrives, the damage is done and the root cause is buried under six weeks of deploys. The fix isn't austerity — it's awareness.

The goal: **know what every service costs, know why, and make cost a first-class engineering decision — not a finance team's problem.**

---

## The Two Types of Software Cost

Software cost breaks into two categories, and teams that only track one are flying half-blind.

### 1. Cost to Host (Fixed / Semi-Fixed)

The cost of keeping your software running, regardless of traffic. This is your baseline — the floor of your infrastructure bill even at zero requests per second.

| Cost Driver | Examples |
|---|---|
| **Compute baseline** | Minimum pod replicas, reserved instances, always-on VMs |
| **Storage** | Persistent volumes, database storage, object storage (S3/GCS/Azure Blob) |
| **Networking** | Load balancers, static IPs, VPN tunnels, DNS zones |
| **Managed services** | Database instances, cache clusters, message broker clusters (always-on) |
| **Licenses** | Commercial software, monitoring platforms, APM tools |
| **Environments** | Dev, staging, QA, perf — each one is a multiplier on your hosting cost |

Hosting cost is the one that creeps. Nobody notices an extra 2 pods here, a dev environment that nobody uses there. Over a year, those add up to real money.

### 2. Cost to Run (Variable / Usage-Based)

The cost that scales with traffic, data volume, and feature usage. This is where cost optimization has the biggest ROI because it's directly tied to your code and architecture decisions.

| Cost Driver | Examples |
|---|---|
| **Compute scaling** | Autoscaled pods/instances beyond the baseline |
| **Data transfer** | Egress between regions, zones, or to the internet |
| **API calls** | Third-party API usage (payment gateways, geocoding, ML inference) |
| **Log ingestion** | Volume of logs shipped to your observability platform |
| **Message throughput** | Kafka/SQS/RabbitMQ message volume and retention |
| **Database I/O** | Read/write IOPS, query compute (BigQuery, Snowflake) |
| **CDN bandwidth** | Edge cache misses, origin pulls |
| **Batch compute** | Scheduled jobs, ETL pipelines, ML training runs |

Variable cost is where bad code decisions directly translate to dollars. An N+1 query that fires 100 database calls instead of 1 doesn't just slow things down — it multiplies your database I/O cost by 100x for that operation.

---

## What to Track — The Cost Pyramid

Not all resources are equally expensive or equally actionable. Focus your tracking in priority order:

```
          /   Compute   \           ← Usually 40-60% of total cost
         /   Data/Storage \         ← Grows forever if unmanaged
        /    Networking     \       ← Sneaky — often ignored until shocking
       /   Managed Services   \     ← Databases, caches, queues
      /  Observability & Tooling \  ← Logging, APM, monitoring platforms
     /   Environments & Sprawl     \ ← Dev/staging/QA multipliers
```

### Compute — The Big One

Compute is almost always the largest line item. Track these metrics per service:

| Metric | Why It Matters | What to Watch For |
|---|---|---|
| **CPU requests vs. actual usage** | Over-requesting wastes money; under-requesting risks throttling | Services requesting 4 cores but averaging 0.3 cores |
| **Memory requests vs. actual usage** | Same as CPU — over-provisioned memory is wasted capacity | 2 GB requested, 400 MB used |
| **Pod/instance count** | More replicas = more cost | Minimum replicas set higher than needed "just in case" |
| **Autoscaler efficiency** | How well does scaling match demand? | Scaling too aggressively on minor traffic spikes |
| **GPU utilization** | GPUs are 10-50x more expensive than CPUs per hour | GPU instances sitting idle between inference batches |
| **CPU architecture** | ARM instances are 20-40% cheaper for compatible workloads | Running everything on x86 without evaluating ARM |

#### The Request vs. Usage Gap

This is the single most common source of compute waste. In Kubernetes, you pay for what you **request**, not what you **use**. If a pod requests 2 CPU cores but uses 0.2, you're paying for 2 and using 0.2. That's 90% waste.

```
Requested:  ████████████████████  2.0 CPU
Actually used: ██                 0.2 CPU
Wasted:     ██████████████████    1.8 CPU (90% waste)
```

**How to right-size:**
1. Run the service under realistic load for at least 2 weeks
2. Measure P95 CPU and memory usage (not average — you need headroom for spikes)
3. Set requests to 1.2–1.5x the P95 value (buffer for bursts)
4. Set limits to 2–3x the request (or no limit for CPU if your platform allows)
5. Re-evaluate after major feature changes or traffic pattern shifts

### Storage — The One That Never Shrinks

Storage costs compound because data accumulates. Nobody ever deletes anything.

| What to Track | Why |
|---|---|
| **Total volume size vs. used space** | Provisioned-but-empty storage costs the same as full storage on many platforms |
| **Data growth rate** | A 5% monthly growth rate means 80% more storage cost in a year |
| **Retention policies** | Are you keeping 2 years of logs you'll never query? |
| **Storage tier** | Hot storage for cold data is expensive; tiered storage (hot → warm → cold → archive) saves 60-90% |
| **Snapshot/backup volume** | Snapshots accumulate — old ones are rarely cleaned up |
| **Database storage** | Table bloat, unused indexes, orphaned data |

#### Retention — The Conversation Nobody Wants to Have

Every data store needs a retention policy. Not "we'll figure it out later." A policy. Written down.

| Data Type | Suggested Retention | Why |
|---|---|---|
| Application logs | 7–30 days hot, 90 days warm, archive or delete | You almost never look at logs older than a week |
| Metrics | 30 days high-resolution, 1 year downsampled | Detailed metrics are huge; downsampled is 95% as useful |
| Traces | 7–14 days | Traces are large; you only need them for active investigations |
| Business data | Per regulatory requirement | Compliance dictates — no more, no less |
| Backups | 30 days rolling | Older backups are almost never restored |
| Dev/staging data | Ephemeral — spin up, tear down | Persistent dev databases are money pits |

### Networking — The Stealth Cost

Networking costs are invisible until they're not. Most platforms charge for data leaving a boundary (egress) but not for data entering (ingress).

| Cost Driver | How to Reduce |
|---|---|
| **Cross-region egress** | Keep services that talk frequently in the same region |
| **Cross-zone egress** | Many platforms charge for inter-AZ traffic; topology-aware routing helps |
| **Internet egress** | CDN caching, response compression, pagination |
| **NAT gateway traffic** | Route high-volume internal traffic through private endpoints, not NAT |
| **Load balancer idle cost** | Consolidate load balancers where possible; shared ingress controllers |

### Managed Services & Dependencies

Every managed service has a cost, and it's often more than you think when you factor in I/O, storage, and data transfer on top of the base instance price.

| Service | What Drives Cost | How to Track |
|---|---|---|
| **Databases (RDS, CloudSQL, etc.)** | Instance size, storage, IOPS, read replicas, backups | Monitor query cost, connection count, replica necessity |
| **Cache (Redis, Memcached)** | Instance size, data volume | Monitor hit rate — a low hit rate means you're paying for a cache that isn't helping |
| **Message brokers (Kafka, SQS)** | Message volume, retention period, partition count | Monitor message throughput and consumer lag |
| **Observability platforms** | Log volume (GB ingested), metric cardinality, trace volume | Set volume budgets per service; alert on spikes |
| **Third-party APIs** | Request count, data volume | Track per-endpoint usage; cache responses where possible |
| **ML inference** | GPU hours, request count, model size | Batch requests; use smaller models where accuracy permits |
| **CDN** | Bandwidth, cache miss rate, origin pulls | Optimize cache keys; set appropriate TTLs |

---

## Code Decisions That Cost Money

Your architecture and code directly affect your infrastructure bill. These are the patterns that silently inflate cost.

### The Expensive Patterns

| Pattern | Cost Impact | Fix |
|---|---|---|
| **N+1 queries** | 100x database I/O for a single page load | Batch fetches, JOINs, DataLoader pattern |
| **Unbounded queries** | Full table scans on millions of rows | Pagination, LIMIT, query filters |
| **Chatty service calls** | 50 HTTP calls where 1 batch call would do | Aggregate APIs, batch endpoints, BFF pattern |
| **Log verbosity** | DEBUG logging in production = 10-100x log volume | Log at INFO baseline; targeted DEBUG only |
| **Large payloads** | Serializing entire objects when the caller needs 3 fields | Field selection, GraphQL, projection |
| **No caching** | Recomputing or re-fetching data that rarely changes | Cache at the appropriate layer with appropriate TTLs |
| **Synchronous fan-out** | 1 request triggers 10 downstream calls in series | Async processing, event-driven architecture |
| **Inefficient serialization** | XML/JSON for high-throughput internal communication | Protobuf, Avro, or MessagePack for service-to-service |
| **Missing indexes** | Full table scans on every query | Index your access patterns; monitor slow query logs |
| **Over-fetching from DB** | `SELECT *` when you need 3 columns | Select only what you need; use projections |

### The Cost of "It's Just a Small Change"

Small decisions compound. Here's what seemingly innocuous changes can cost at scale:

```
"Let's add a DEBUG log in the order processing loop"
→ 500 orders/sec × 10 log lines per order × 500 bytes per line
→ 2.5 MB/sec → 216 GB/day → $430/day in log ingestion (at $2/GB)
→ $157,000/year from one log statement

"Let's add a header with the full user profile to every internal request"
→ 2 KB extra per request × 50,000 req/sec
→ 100 MB/sec additional network traffic → 8.6 TB/day
→ Cross-zone egress at $0.01/GB = $86/day = $31,000/year

"Let's keep 90 days of Kafka retention instead of 7"
→ 12.8x more storage per topic
→ 50 topics × 100 GB average = 5 TB at 7 days → 64 TB at 90 days
→ At $0.10/GB/month = $6,400/month → $76,800/year
```

The point isn't that these changes are always wrong — sometimes 90 days of Kafka retention is a business requirement. The point is that these decisions should be made with the cost visible, not discovered on a bill 3 months later.

---

## GPU — The Expensive Exception

GPUs deserve their own section because they are 10–50x more expensive per hour than CPUs, and GPU waste is the fastest way to blow an infrastructure budget.

| GPU Consideration | Guidance |
|---|---|
| **Utilization threshold** | Target >60% utilization; below 40% is a red flag |
| **Idle GPU instances** | A single idle A100 GPU costs ~$25,000/year; shut down what's not in use |
| **Right-sizing** | Don't use an A100 when a T4 handles the workload — 10x cost difference |
| **Batch vs. real-time** | Batch inference requests to maximize GPU throughput; avoid 1-request-at-a-time patterns |
| **Spot/preemptible instances** | Training workloads can often tolerate interruptions — 60-90% savings |
| **Model optimization** | Quantization, distillation, pruning can reduce GPU requirements by 2-4x |
| **Shared GPU scheduling** | Time-slicing or MIG (Multi-Instance GPU) lets multiple workloads share one GPU |
| **CPU offloading** | Pre/post-processing on CPU; only inference on GPU. Don't waste GPU cycles on data prep |

### The GPU Cost Trap

```
Scenario: ML team provisions 8 A100 GPUs for a training cluster
  - Training runs happen Tuesday and Thursday, ~4 hours each
  - GPUs sit idle the other 160 hours/week
  - Monthly cost: 8 GPUs × $3.00/hr × 730 hrs = $17,520/month
  - Actual usage: 8 GPUs × $3.00/hr × 32 hrs = $768/month
  - Waste: $16,752/month = $201,000/year

Fix: Use spot instances for training, autoscale to zero when idle,
     or use a shared GPU pool with scheduling
```

---

## Team-Level Cost Accountability

Cost optimization doesn't work as a top-down mandate. It works when every team owns the cost of their services the same way they own uptime and latency.

### What Every Team Should Know

| Question | Why It Matters |
|---|---|
| What does my service cost per month? | If you can't answer this, you can't optimize it |
| What's my cost per request/transaction? | This is your unit economics — the true cost of the business operation |
| What changed since last month? | Catch cost drift before it compounds |
| What are my top 3 cost drivers? | Focus optimization where it matters most |
| What would 2x traffic do to my cost? | Understand your cost scaling curve before it happens |

### Cost Per Transaction — The Metric That Matters

Total cost is meaningless without context. A service that costs $50,000/month and processes 100 million transactions is efficient. One that costs $5,000/month and processes 1,000 transactions is not.

```
Cost per transaction = Monthly infrastructure cost / Monthly transaction count

Example:
  Service A: $12,000/month ÷ 50M requests = $0.00024/request
  Service B: $3,000/month ÷ 200K requests = $0.015/request

Service B costs 62x more per transaction despite being 4x cheaper total.
```

Track cost per transaction over time. If it's increasing without a corresponding feature justification, something is drifting.

### Team Cost Dashboard

Every team should have visibility into their service costs, refreshed at least weekly. At minimum, show:

- **Total cost** by service, broken down by resource type (compute, storage, networking, dependencies)
- **Cost trend** — week-over-week and month-over-month
- **Cost per transaction** — the efficiency metric
- **Top cost anomalies** — spikes or step changes that need explanation
- **Resource utilization** — CPU/memory requests vs. actual usage (the waste indicator)
- **Environment costs** — how much are dev/staging/QA costing relative to production?

### The Cost Review Ritual

Add a 5-minute cost check to your existing operational review (sprint retro, ops review, whatever cadence your team uses):

1. **What changed?** — Any cost spikes or step changes since last review?
2. **Why?** — New feature, traffic growth, configuration change, or waste?
3. **Action needed?** — Right-size, clean up, or accept and budget for it?

This doesn't need to be a separate meeting. Five minutes of cost awareness every two weeks prevents six-figure surprises.

---

## Org-Level Cost Governance

Team-level awareness is necessary but not sufficient. Without organizational governance, teams optimize locally while costs grow globally.

### Chargeback and Showback

| Model | How It Works | When to Use |
|---|---|---|
| **Showback** | Show teams their cost, but don't charge them | Starting out; building awareness before accountability |
| **Chargeback** | Bill teams for their actual usage against their budget | Mature organizations with reliable cost attribution |
| **Shared pool** | Infrastructure is a shared cost allocated by headcount or revenue | Small orgs or when attribution is too complex |

**Recommendation:** Start with showback. Forcing chargeback before teams have visibility and tooling just creates resentment and gaming. Once teams can see and understand their costs, transition to chargeback for direct resources (compute, storage) while keeping shared services (networking, platform tooling) as a shared pool.

### Tagging and Attribution

You can't manage cost you can't attribute. Every resource needs tags that map it to a team, service, and environment.

**Minimum required tags:**

| Tag | Purpose | Example |
|---|---|---|
| `team` | Cost ownership | `team:order-management` |
| `service` | Which service uses it | `service:order-api` |
| `environment` | Prod vs. non-prod | `environment:production` |
| `cost-center` | Financial attribution | `cost-center:CC-4521` |

**The untagged resource problem:** Every cloud platform has resources that slip through without tags — someone spun up a test instance manually, a Terraform module doesn't propagate tags, a script creates resources without metadata. Track untagged resources as a metric. Target < 5% untagged by cost.

### Budget and Forecast

| Practice | Cadence | Who |
|---|---|---|
| **Monthly cost report** | Monthly | Finance + engineering leadership |
| **Budget vs. actual comparison** | Monthly | Service owners + finance |
| **Quarterly forecast** | Quarterly | Engineering leadership |
| **Annual capacity planning** | Annually | Architecture + finance + product |
| **Cost anomaly alerts** | Real-time | Service owners (automated) |

#### Cost Anomaly Detection

Set up automated alerts for cost anomalies, just like you alert on error rates:

| Condition | Alert |
|---|---|
| Daily spend > 130% of 30-day average | Investigate same day |
| Weekly spend > 120% of 4-week average | Review in next team sync |
| New resource type appears (GPU, large instance) | Notify immediately — someone may be experimenting without awareness of cost |
| Untagged resources > 10% of total spend | Tag cleanup sprint |

### Environment Cost Discipline

Non-production environments are a stealth cost multiplier. Every environment is a fraction of your production cost, and most teams have too many.

| Environment Practice | Savings |
|---|---|
| **Scale non-prod to zero overnight and weekends** | 65-75% reduction in non-prod compute |
| **Use smaller instance types for dev/staging** | 50-80% per environment |
| **Ephemeral environments** — spin up for testing, tear down after | Eliminates persistent idle environments |
| **Shared staging** instead of per-team staging | N environments → 1 environment |
| **Audit quarterly** — which environments exist, who uses them, when were they last accessed? | Catches forgotten environments |

The average team has at least one environment that nobody has touched in 6 months but is still running 24/7. Find it. Kill it.

---

## The Cost Optimization Playbook

When you need to reduce cost, work through these levers in order — highest ROI first.

### Tier 1: Eliminate Waste (No behavior change, just cleanup)

| Action | Typical Savings | Effort |
|---|---|---|
| Right-size CPU/memory requests to match actual usage | 30-50% compute | Low |
| Delete unused resources (orphaned volumes, old snapshots, idle instances) | 5-15% total | Low |
| Shut down unused environments | 10-30% non-prod | Low |
| Remove unused indexes (they cost write I/O) | Varies | Low |
| Scale non-prod to zero outside business hours | 65-75% non-prod | Medium |
| Clean up old container images from registries | 5-10% storage | Low |

### Tier 2: Optimize Usage (Small code/config changes)

| Action | Typical Savings | Effort |
|---|---|---|
| Reduce log verbosity and add sampling | 30-70% logging cost | Low |
| Add caching for repeated lookups | 20-50% on cached paths | Medium |
| Fix N+1 queries and unbounded fetches | 50-90% on affected queries | Medium |
| Compress payloads for internal communication | 20-40% network | Low |
| Implement data retention and lifecycle policies | 40-80% storage over time | Medium |
| Tune autoscaler settings (scale-down delay, threshold) | 10-30% compute | Low |
| Use pagination and field selection for API responses | 20-50% on affected endpoints | Medium |

### Tier 3: Architecture Changes (Larger investment, larger payoff)

| Action | Typical Savings | Effort |
|---|---|---|
| Move from synchronous fan-out to event-driven | 30-60% compute for affected flows | High |
| Replace polling with push/webhooks | 80-95% for polling workloads | Medium |
| Introduce tiered storage (hot → warm → cold) | 60-90% storage | Medium |
| Consolidate microservices that always deploy together | 20-40% on overhead | High |
| Switch to ARM-based instances for compatible workloads | 20-40% compute | Medium |
| Use spot/preemptible instances for fault-tolerant workloads | 60-90% for eligible workloads | Medium |
| Adopt reserved instances or savings plans for stable baselines | 30-60% on committed usage | Low (commitment) |

### Tier 4: Strategic Decisions (Product/business level)

| Action | Typical Savings | Effort |
|---|---|---|
| Deprecate low-usage features that carry high infrastructure cost | Varies widely | High |
| Renegotiate vendor contracts with usage data | 10-30% on third-party services | Medium |
| Build vs. buy re-evaluation with current costs | Varies | High |
| Multi-region strategy review — do you need all those regions? | 30-50% if regions can be consolidated | High |

---

## Common Mistakes

### 1. Optimizing Before Measuring

"Let's move to ARM instances to save money." How much? "We think a lot." How much do you spend on compute now? "Not sure."

Always measure first. Know your baseline, know your breakdown, know your top 3 cost drivers. Then optimize the thing that's actually expensive, not the thing that feels expensive.

### 2. Over-Provisioning "Just in Case"

```
# The fear-driven resource request
resources:
  requests:
    cpu: "4"        # Actual P95 usage: 0.5 cores
    memory: "8Gi"   # Actual P95 usage: 1.2 Gi
```

Teams over-provision because they've been burned by resource exhaustion and nobody gets fired for having headroom. But 8x over-provisioning across 200 services is a massive waste.

Fix: Right-size based on observed P95 usage + 20-50% buffer. Use autoscaling for spikes instead of static over-provisioning.

### 3. Treating Non-Prod Like Prod

Running dev and staging with the same instance sizes, replica counts, and retention policies as production is throwing money away. Non-prod doesn't need 3 replicas, 30 days of log retention, or production-grade database instances.

### 4. Ignoring Data Transfer Costs

Teams obsess over compute and ignore the $40,000/month cross-region data transfer bill. Data transfer costs are especially brutal for:
- Services that replicate data across regions
- Fan-out architectures that broadcast to many consumers
- Services behind NAT gateways with high outbound traffic
- Large API responses without compression

### 5. "We'll Optimize Later"

Later never comes. Cost debt compounds just like tech debt, except cost debt has a monthly invoice. Build cost awareness into the development process from day one — not as a quarterly panic exercise.

### 6. Log Everything, Question Nothing

Logging is often the second or third largest line item, and a single verbose log statement in a hot path can cost more per year than a junior engineer's salary. See Logging for log cost management details, including the cost equation, log volume budgets, and sampling strategies.

### 7. Snapshot and Backup Hoarding

Every automated backup and snapshot policy creates data — and almost none of them have a corresponding cleanup policy. Check your snapshot inventory. You probably have 18 months of daily snapshots for a service that only needs 30 days.

### 8. Forgetting About Idle Resources

Resources that exist but do nothing:
- Load balancers with no targets
- Elastic IPs not attached to anything
- Provisioned database read replicas with zero queries
- Persistent volumes detached from any pod
- Container images in registries for services that were decommissioned years ago

Set up a monthly sweep for idle resources. Most cloud platforms provide tools for this.

### 9. Per-Service Optimization Without System Thinking

Optimizing Service A's database queries might shift load to Service B's cache, which shifts load to Service C's message queue. Cost optimization should consider the system, not just individual services.

### 10. Confusing Cost Reduction with Value Reduction

Cutting cost should not mean degrading the product. Removing a feature to save $500/month when it drives $50,000/month in revenue is not optimization — it's destruction. Always pair cost discussions with value discussions.

---

## Physical Resource Tracking Summary

Quick reference for which resources to track, why, and what "healthy" looks like:

| Resource | Track These Metrics | Healthy Range | Red Flag |
|---|---|---|---|
| **CPU** | Request vs. usage ratio, utilization % | 40-70% utilization, <2x over-provision | <20% utilization or >5x over-provision |
| **Memory** | Request vs. usage ratio, growth trend | 50-80% utilization, stable over time | Steady growth (leak), <30% utilization |
| **GPU** | Utilization %, idle hours, instance type vs. workload | >60% utilization when active | <40% utilization, idle >50% of the time |
| **Storage** | Used vs. provisioned, growth rate, tier distribution | <80% used, growth rate matches business growth | Unbounded growth, no retention policy |
| **Network** | Egress volume, cross-region/zone traffic, compression ratio | Minimal cross-region, responses compressed | Uncompressed large payloads, unnecessary cross-region calls |
| **Database I/O** | IOPS, query count, slow queries, connection pool usage | Queries are indexed, pool <70% | Full table scans, pool exhaustion, unused read replicas |
| **Log volume** | GB/day ingested, lines/sec per service | Within defined budget per service | Sudden spikes, DEBUG in prod, logging in hot loops |
| **Message throughput** | Messages/sec, consumer lag, retention volume | Lag near zero, retention matches policy | Growing lag, 90-day retention on ephemeral data |

---

## Quick Reference Checklist

- [ ] **Every service has a known monthly cost** — if you can't answer "what does this cost?", that's problem #1
- [ ] **Cost per transaction is tracked** for key business operations
- [ ] **CPU and memory requests are right-sized** to within 1.5x of P95 actual usage
- [ ] **GPU instances are monitored for utilization** and shut down or autoscaled to zero when idle
- [ ] **All resources are tagged** with team, service, environment, and cost center
- [ ] **Retention policies exist** for logs, metrics, traces, backups, and snapshots
- [ ] **Non-prod environments scale to zero** outside business hours (or are ephemeral)
- [ ] **Log volume budgets** are set per service with alerts on spikes
- [ ] **Data transfer costs are visible** — cross-region and cross-zone traffic is tracked
- [ ] **Cost anomaly alerts** fire when daily spend exceeds 130% of the 30-day average
- [ ] **Monthly cost review** happens — even if it's 5 minutes in an existing meeting
- [ ] **Quarterly budget vs. actual** comparison is shared with engineering leadership
- [ ] **No idle resources** — monthly sweep for orphaned volumes, detached IPs, unused load balancers, empty registries
- [ ] **Code changes that affect cost** (new logging, new dependencies, new data stores) are called out in PR reviews
- [ ] **Cost optimization follows the tier order** — eliminate waste before changing architecture
- [ ] **Cost and value are discussed together** — never cut cost without understanding the value impact

---

## Related Guides

- See Logging for log cost management details — the cost equation, volume budgets, sampling strategies, and retention tiering.
