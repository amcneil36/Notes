# Caching Best Practices

Caching is the most impactful performance optimization you can make — and the most dangerous one to get wrong. A well-placed cache turns a 200ms database query into a 2ms memory lookup. A poorly managed cache serves stale data, creates consistency nightmares, and gives you a system that's fast but wrong. The hard part of caching is never the cache itself — it's knowing when the cached data is no longer valid.

**The goal: serve data fast without lying to the user.**

---

## When to Cache (and When Not To)

### Cache When:

| Signal | Example |
|--------|---------|
| **Read-heavy, write-light** | Product catalog pages viewed 10,000x/min, updated 2x/day |
| **Expensive to compute** | Aggregation query that joins 5 tables and takes 800ms |
| **Data changes infrequently** | Country list, tax rates, feature flag configurations |
| **Tolerance for slight staleness** | Dashboard metrics, recommendation lists, search suggestions |
| **Upstream is slow or rate-limited** | Third-party API with 500ms latency and 100 req/min cap |

### Don't Cache When:

| Signal | Why |
|--------|-----|
| **Data must be real-time accurate** | Account balances, inventory counts during checkout, auth tokens |
| **Writes are as frequent as reads** | Chat messages, live bidding systems |
| **Data is unique per request** | Personalized search results with dozens of filter combinations |
| **Cache hit rate would be low** | Long-tail queries where most keys are accessed once |
| **The origin is already fast** | If your database returns in 5ms, adding a cache adds complexity for marginal gain |

The simplest cache decision: **if you wouldn't show yesterday's data to a user, think twice before caching it.**

---

## Cache Strategies

### 1. Cache-Aside (Lazy Loading)

The application manages the cache explicitly. On read, check the cache first. On miss, read from the source, then populate the cache.

```
Read:
  1. Check cache for key
  2. Cache hit → return cached value
  3. Cache miss → read from database → write to cache → return value

Write:
  1. Write to database
  2. Invalidate (delete) the cache key
```

**When to use:** Most applications. It's the simplest pattern and gives you full control.

**Trade-offs:**
- First request after expiry/invalidation is always slow (cache miss)
- Risk of stale data if invalidation is missed
- Application code must handle both cache and database

### 2. Read-Through

The cache itself is responsible for loading data on a miss. The application only talks to the cache, never directly to the database.

```
Read:
  1. Request data from cache
  2. Cache hit → return cached value
  3. Cache miss → cache loads from database → stores it → returns value
```

**When to use:** When you want to keep caching logic out of your application code. Common with caching libraries that support loaders.

**Trade-offs:**
- Cleaner application code
- Cache must understand how to load from the source
- Still has cold-start latency on first access

### 3. Write-Through

Every write goes through the cache to the database. The cache is always up-to-date because it's in the write path.

```
Write:
  1. Write to cache
  2. Cache synchronously writes to database
  3. Return success
```

**When to use:** When you need strong consistency between cache and database and can tolerate slightly slower writes.

**Trade-offs:**
- Cache is always fresh — no stale reads
- Writes are slower (two writes: cache + database)
- Caches data that may never be read (wasted memory for write-heavy keys)

### 4. Write-Behind (Write-Back)

Write to the cache immediately, then asynchronously flush to the database in batches.

```
Write:
  1. Write to cache → return success immediately
  2. Background: batch-flush dirty entries to database
```

**When to use:** Write-heavy workloads where you can tolerate the risk of data loss between the cache write and the async database write.

**Trade-offs:**
- Fastest writes — user gets immediate confirmation
- Risk of data loss if the cache node crashes before flushing
- Complex to implement correctly (ordering, deduplication, failure handling)
- Not appropriate for financial or transactional data

### 5. Refresh-Ahead

Proactively refresh cache entries before they expire, based on predicted access patterns.

```
Background:
  1. Monitor access frequency for cached keys
  2. Before TTL expires on hot keys, reload from database
  3. Cache is always warm for popular data
```

**When to use:** When cache misses are expensive and you have predictable access patterns (e.g., product pages, homepage content).

**Trade-offs:**
- Eliminates cache-miss latency for hot data
- Wastes resources refreshing data that may not be accessed again
- More complex to implement and tune

### Strategy Comparison

| Strategy | Read Latency | Write Latency | Consistency | Complexity | Best For |
|----------|-------------|---------------|-------------|------------|----------|
| Cache-aside | Miss is slow | Fast (invalidate only) | Eventual | Low | **Default choice** |
| Read-through | Miss is slow | Fast | Eventual | Medium | Abstracted cache layer |
| Write-through | Always fast | Slower (dual write) | Strong | Medium | Read-heavy, consistency matters |
| Write-behind | Always fast | Fastest | Weak (risk of loss) | High | Write-heavy, loss-tolerant |
| Refresh-ahead | Always fast | Depends | Eventual | High | Hot data with predictable patterns |

**Default to cache-aside** unless you have a specific reason to choose something else.

---

## Cache Invalidation

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

This is where most caching implementations break. Getting data into the cache is easy. Knowing when to remove it is the real problem.

### Strategy 1: TTL (Time-To-Live)

Every cache entry expires after a fixed duration. Simple, predictable, and usually good enough.

```
cache.set("product:123", productData, ttl=300)  // expires in 5 minutes
```

**TTL Guidelines:**

| Data Type | Recommended TTL | Why |
|-----------|----------------|-----|
| Static reference data (country lists, config) | 1-24 hours | Changes rarely, low risk of staleness |
| Product catalog / content | 5-60 minutes | Balances freshness with read performance |
| User session / profile | 15-30 minutes | Needs to reflect recent changes reasonably quickly |
| Search results / recommendations | 1-5 minutes | Users expect some freshness but tolerate slight staleness |
| Real-time metrics / dashboards | 10-60 seconds | Needs to feel "live" without hammering the source |

**Jitter your TTLs.** If every cache entry for a product catalog has a 5-minute TTL and they were all populated at the same time, they all expire at the same time — stampeding the database. Add randomness: `ttl = base_ttl + random(0, base_ttl * 0.1)`.

### Strategy 2: Event-Driven Invalidation

When the source data changes, publish an event that triggers cache invalidation.

```
1. Product updated in database
2. Publish event: "product:123:updated"
3. Cache subscriber receives event → deletes "product:123"
4. Next read triggers a cache miss → fresh data loaded
```

**When to use:** When you need near-real-time consistency and you already have an event bus (Kafka, pub/sub, change data capture).

**Pitfall:** If the invalidation event is lost or delayed, the cache stays stale. Always pair with a TTL as a safety net — event-driven invalidation handles the common case, TTL handles the failure case.

### Strategy 3: Version-Based Invalidation

Embed a version number in the cache key. When data changes, increment the version, and the old key is effectively abandoned.

```
Key: "product:123:v5" → updated → "product:123:v6"
```

**When to use:** Immutable cache entries (pre-rendered pages, compiled assets, computed results). The old version naturally expires via TTL.

### Invalidation Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| **No TTL at all** | Cache entry lives forever. If invalidation fails, data is stale indefinitely. | Always set a TTL, even a generous one. It's your safety net. |
| **Invalidate-then-write** | Delete cache, then update DB. If DB write fails, cache is empty but data didn't change. | Write to DB first, then invalidate cache. |
| **Update cache instead of invalidate** | Update cache and DB separately. Race conditions between concurrent writes can leave cache with stale data. | Prefer delete-on-write over update-on-write. Let the next read repopulate. |
| **Broad invalidation** | "Something changed, clear everything." Destroys hit rate. | Invalidate only the specific keys affected by the change. |
| **No invalidation at all** | "TTL will handle it." Maybe — but a 5-minute TTL means 5 minutes of stale data after every write. | Invalidate on write, use TTL as backup. |

---

## Cache Stampede (Thundering Herd)

When a popular cache key expires, hundreds of concurrent requests all miss the cache simultaneously and all hit the database at once.

```
t=0:     Cache key "homepage" expires
t=0.01s: 500 requests arrive, all get cache miss
t=0.02s: 500 identical database queries execute concurrently
t=0.5s:  Database buckles under load
```

### Solutions

**1. Locking (Mutex)**

Only one request loads the data. Others wait for the cache to be populated.

```
1. Request A: cache miss → acquire lock → load from DB → populate cache → release lock
2. Requests B-500: cache miss → lock is held → wait → cache is populated → serve from cache
```

**Trade-off:** Adds latency for waiting requests. If the lock holder crashes, you need a lock timeout.

**2. Early Expiry / Probabilistic Refresh**

Before the TTL actually expires, randomly decide to refresh the entry. The probability increases as expiry approaches.

```
remaining_ttl = expiry_time - now
should_refresh = random() < (1 / remaining_ttl_seconds)

// As TTL gets close to 0, the probability of refreshing approaches 100%
// One lucky request refreshes early, the rest keep serving the existing value
```

**Trade-off:** Doesn't guarantee prevention, but dramatically reduces the probability of a stampede.

**3. Stale-While-Revalidate**

Serve the stale cached value immediately while refreshing in the background. The user gets a fast (slightly stale) response, and the cache is updated for the next request.

```
1. Cache key expired
2. Return stale value to caller immediately
3. Background: reload from database, update cache
4. Next request gets fresh data
```

**Trade-off:** Users may see slightly stale data for one request cycle. Excellent for data where "1 minute stale" is acceptable.

---

## Local vs. Distributed Cache

### Local (In-Process) Cache

Data lives in the application's memory space. Fastest possible access — no network hop.

| Pros | Cons |
|------|------|
| Sub-microsecond reads | Data duplicated across instances |
| No network latency | Invalidation across instances is hard |
| No external dependency | Limited by instance memory |
| Survives network partitions | Lost on restart/deploy |

**Best for:** Small, static datasets (config, feature flags, lookup tables), hot-path data where even 1ms network latency matters, and fallback caching when the distributed cache is down.

### Distributed Cache (Redis, Memcached, etc.)

Data lives in a shared external cache cluster. All application instances share the same cached data.

| Pros | Cons |
|------|------|
| Single source of truth across instances | Network hop on every access (~0.5-2ms) |
| Survives instance restarts | External dependency that can fail |
| Scales independently from application | Operational overhead (cluster management) |
| Easy invalidation (one place to delete) | Serialization/deserialization cost |

**Best for:** Shared state, session data, any cache where consistency across instances matters.

### Two-Tier Caching (Recommended for High-Traffic Systems)

Use both. Local cache for hot data with short TTLs, distributed cache as the shared backing layer.

```
Request → Local cache (hit?) → Distributed cache (hit?) → Database
```

- Local cache TTL: 10-60 seconds (very short, tolerates slight staleness)
- Distributed cache TTL: 5-30 minutes (longer, shared invalidation)
- Database: source of truth

**Invalidation flow:** Write to DB → invalidate distributed cache → local caches expire naturally via short TTL (or broadcast invalidation event for immediate consistency).

---

## Cache Key Design

Bad cache keys cause collisions, wasted memory, and debugging nightmares.

### Rules

1. **Include all inputs that affect the output.** If the response varies by user role, the role must be in the key.

    ```
    Bad:  "products"         (which products? for which locale?)
    Good: "products:electronics:en-US:page=2:sort=price"
    ```

2. **Use a consistent delimiter.** Colons are conventional. Be consistent across the codebase.

3. **Prefix with the entity type.** Makes it easy to find and invalidate related keys.

    ```
    "user:123:profile"
    "user:123:orders"
    "product:456:details"
    ```

4. **Keep keys deterministic.** Same inputs must always produce the same key. Sort query parameters, normalize casing.

    ```
    Bad:  Cache key depends on HashMap iteration order (non-deterministic)
    Good: Sort parameters alphabetically before building the key
    ```

5. **Avoid unbounded keys.** If the key includes a full query string or request body, hash it. Long keys waste memory and slow lookups.

    ```
    "search:" + sha256(normalized_query_params)
    ```

6. **Include a version prefix.** When the cached data structure changes (e.g., new fields), bump the version to avoid deserialization errors on old entries.

    ```
    "v2:product:456:details"
    ```

---

## What to Cache — And Where

| What | Where | TTL | Notes |
|------|-------|-----|-------|
| Database query results | Distributed cache | 5-30 min | Most common use case |
| Rendered HTML fragments | Local or distributed | 1-15 min | Great for expensive templates |
| API responses from third parties | Distributed cache | Depends on API's rate limits | Respect upstream cache headers |
| Computed aggregations | Distributed cache | Match computation frequency | Pre-compute on a schedule if possible |
| Configuration / feature flags | Local cache | 30-300 sec | Refresh frequently, tolerate brief staleness |
| Session data | Distributed cache | 15-30 min (sliding) | Must be shared across instances |
| Static assets (CSS, JS, images) | CDN / browser cache | Long (hours/days) | Use content hashing for cache busting |
| DNS lookups | Local cache | Follow DNS TTL | Most runtimes handle this automatically |

---

## Monitoring and Observability

You can't manage what you can't measure. A cache without metrics is a black box.

### Key Metrics

| Metric | What It Tells You | Target |
|--------|-------------------|--------|
| **Hit rate** | % of requests served from cache | > 90% for most use cases |
| **Miss rate** | % of requests that fall through to the source | < 10% — investigate if higher |
| **Eviction rate** | How often entries are removed to make room | Should be low if cache is properly sized |
| **Latency (cache reads)** | Time to read from cache | < 1ms local, < 5ms distributed |
| **Latency (cache misses)** | Time to load from source on miss | Shows the cost of not caching |
| **Memory usage** | How full is the cache | Alert at 80% — evictions spike at capacity |
| **Key count** | Number of entries in cache | Track growth over time |
| **TTL distribution** | How entries are distributed by remaining TTL | Detect mass-expiry patterns |

### Alerts

- **Hit rate drops below threshold:** Something changed — new traffic pattern, invalidation bug, or cache is too small.
- **Eviction rate spikes:** Cache is full. Either increase capacity or reduce what you're caching.
- **Cache latency increases:** Network issue, cache overloaded, or serialization cost growing.
- **Cache unavailable:** Your fallback behavior kicks in. Make sure you have one.

---

## Common Pitfalls

| Pitfall | Problem | Fix |
|---------|---------|-----|
| **Caching errors** | A 500 response gets cached. Every subsequent request gets the cached error instead of retrying. | Never cache error responses. Only cache successful results. |
| **Caching nulls without TTL** | Database returns no result, you cache `null` forever. The data is created later but the cache still says it doesn't exist. | Cache negative results with a short TTL (30-60 seconds). |
| **Cache as primary datastore** | Treating the cache as the source of truth. Cache eviction = data loss. | The cache is always a copy. The source of truth is the database. |
| **Ignoring serialization cost** | Caching a complex object graph. Serialization/deserialization takes longer than the database query. | Cache the minimal data needed. Profile serialization time. |
| **No fallback when cache is down** | Distributed cache goes down, application errors instead of falling through to the database. | Always code the "cache miss" path. The cache should be an optimization, not a dependency. See Graceful Degradation for cache fallback patterns during outages. |
| **Over-caching** | Caching everything "just in case." Memory bloat, low hit rates, complex invalidation. | Cache deliberately. Measure hit rates. Remove caches with < 80% hit rate. |
| **Same TTL for everything** | All keys expire at the same time, causing periodic load spikes. | Tune TTLs per data type. Add jitter. |
| **Cache warming on deploy** | Application starts cold after deploy, all traffic hits the database until cache fills. | Pre-warm critical keys on startup, or use rolling deploys so not all instances go cold simultaneously. |
| **Not accounting for cache in capacity planning** | Database is sized assuming cache absorbs 90% of reads. Cache goes down, database gets 10x the expected load. | Size your database to survive a full cache failure, at least temporarily. |

---

## Related Topics

- Graceful Degradation — cache fallback patterns during outages, circuit breaker + cache interaction
- Latency — performance context, where caching fits in the broader latency optimization picture

---

## Quick Reference Checklist

- [ ] **Identified what to cache** — high-read, low-write, expensive-to-compute data
- [ ] **Chosen a cache strategy** — cache-aside is the default; use others with justification
- [ ] **Set appropriate TTLs** per data type — not one TTL for everything
- [ ] **Added jitter to TTLs** to prevent synchronized expiry
- [ ] **Implemented invalidation** on writes — delete from cache when source data changes
- [ ] **Handled cache stampede** — locking, early refresh, or stale-while-revalidate
- [ ] **Designed deterministic cache keys** with entity prefix, all relevant inputs, and version
- [ ] **Built a fallback path** — application works (slower) when cache is unavailable
- [ ] **Never caching errors** — only cache successful responses
- [ ] **Monitoring hit rate, miss rate, eviction rate, and latency**
- [ ] **Sized the cache** based on working set, not total dataset
- [ ] **Cache warming strategy** for cold starts after deploys
- [ ] **Tested cache behavior** — miss path, invalidation, expiry, fallback, stampede under load
