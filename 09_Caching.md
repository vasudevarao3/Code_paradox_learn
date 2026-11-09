# Caching

Starting from what caching physically is, build up to cache-aside design, write strategies, freshness and invalidation, hot-key stampedes, and why a cache going down should be a performance incident, not an outage.

##### 01. What is Caching, At all? - What caching actually is, from sticky-note analogy up

* A **cache** is a fast storage layer that sits in front of something slower (a database, an external API, an expensive computation) and stores the result of work that's already been done, so a repeated request for the same thing can skip straight to the answer.
  * A cache is a performance layer, never the source of truth. If every cache in a system vanished at once, the system should get slower, not wrong. The moment a cache is the only place a piece of data lives, it has stopped being a cache and become an unreliable database.
* “Caching always makes a system more correct or more reliable.”?
  * It doesn't: caching is purely a performance tool. Used carelessly, it introduces a category of bug that didn't exist before: a cached value that no longer matches reality. A system with no cache at all is never stale. A system with a badly-designed cache can be fast and wrong at the same time, which is usually worse than being slow and right.

##### 02. Cache Basics: Hits, Misses and Hit Ratio: Mechanics - Hits, misses and why the cache is never the source of truth.

* Every cache lookup ends in one of exactly two outcomes. A **hit** means the value was found in the cache and can be returned immediately, without touching the slower system behind it. A **miss** means the value wasn't there, so the request has to fall back to the real source of truth, and, usually, store the result for next time.
* A cache only earns its complexity when the same data is requested repeatedly. The fraction of requests that hit is the single most important number for judging whether a cache is doing its job.
* **Cache hit ratio** = cache hits ÷ total requests = 920,000 hits ÷ 1,000,000 requests = 92% hit ratio: a healthy, well-tuned result for most read-heavy workloads
* **Typical hit latency** = one in-memory cache read = ~1 ms for a Redis GET on the same private network
* **Typical miss latency** = cache read + database query = ~1 ms + ~45 ms ≈ 46 ms, roughly 46x slower than a hit
* That 46x gap is why hit ratio matters so much: at a 92% hit ratio and 1,000,000 requests, the average request latency is (0.92 × 1 ms) + (0.08 × 46 ms) ≈ 4.6 ms. Drop the hit ratio to 70% on the same mix, and the average climbs to (0.70 × 1 ms) + (0.30 × 46 ms) ≈ 14.5 ms, a 3x regression driven entirely by more requests taking the slow path.
* **Cache:** A fast storage layer used to avoid repeating expensive database or service work.
* **Hit:** A request that finds its value in the cache and returns it without the slower backend.
* **Miss:** A request that finds nothing in the cache and must fall back to the source of truth.
* **Hit ratio:** The share of requests served from cache: the core signal for whether a cache is worth its complexity.
* **TTL:** Time-to-live: how long a cached item is allowed to stay before it's treated as stale.
* **Invalidation:** Actively removing or updating cached data the moment the source of truth changes.
* **Stampede:** A surge of simultaneous cache misses that sends a thundering herd of traffic at the backend.
* **Eviction:** Removing items from a full cache to make room for newer or more valuable values.
* **Write-Through:** Writing to the cache and the database in the same operation.
* **Write-back:** Writing to the cache first and flushing to the database later, asynchronously.
* **Write-around:** Writing straight to the database and deliberately skipping the cache.
* **Cache warning:** Proactively populating a cache before real traffic hits it, so a cold start doesn't trigger a system-wide stampede.

##### 03. Cache-Aside and Read Through: Patterns - A mechanical, step-by-step walkthrough of a MISS versus a HIT

* **Cache-aside** is the most common pattern in application code: the application itself checks the cache, and on a miss, explicitly queries the database and writes the result back into the cache before returning it. Nothing happens automatically: every step is code the application owns.
* **Read-through** does the identical work, just moved one layer down: the cache client itself knows how to fetch from the database on a miss, so the application only ever talks to the cache and never sees the branch. Either way, the database stays the authority: the cache is only ever a shortcut in front of it.
* So what's actually difference between them?
  * Functionally, nothing the caller can observe: both fall back to the database on a miss and both repopulate the cache. The difference is who owns the fallback code. Cache-aside keeps it in application logic, which is flexible but means every caller has to remember to do it correctly. Read-through pushes it into a shared cache client or library, so the miss-handling logic is written once and can't be forgotten by a new call site.
* A cache only helps when the same data is requested repeatedly. A one-time read produces exactly one guarantee miss and adds a write to the cache that nothing will ever read again: pure overhead, no benefit.

##### 04. Write Strategies: Patterns - Write0through, write-back, and write-around, compared side by side.

* Everything so far has been about reads. Writes need their own decision: when a value changes, does the cache get updated immediately, later or not at all? There are three standard answers, and they trade latency against durability in opposite directions.
* When would anyone actually choose write-around?
  * Write-around makes sense when data is written far more often than it's re-read soon after: bulk imports, logging, analytics ingestion. Populating the cache on every such write would just evict genuinely hot data to make room for values nobody is about to ask for again.
* | Strategy      | Write Path                                                                   | Latency Felt by Caller                        | Durability Risk                                              |
  | ------------- | ---------------------------------------------------------------------------- | --------------------------------------------- | ------------------------------------------------------------ |
  | Write-through | Write to the cache and the database in the same operation                    | As slow as the database write                 | Low: the database is updated before the write "succeeds"     |
  | Write back    | Write to the cache immediately, flush to the database later, asynchronously. | Fast: the caller doesn't wait on the database | Higher: data can be lost if the cache dies before the flush. |
  | Write-around  | Write straight to the database; skip the cache entirely.                     | Unaffected by the cache                       | Low, but the next read of that key is a guaranteed miss.     |
* Safer: Write-through and cache-aside the database us always caught up
* Riskier: Write-back; faster writes, but data can vansish if the cache dies first.

##### 05. Freshness, TTL and Invalidation: Correctness - How long a cached value is allowed to be wrong before it's refreshed.

* Freshness is the central challenge of caching. A cached value is only ever a snapshot of the truth at the moment it was written: the instant the real data changes underneath it, that snapshot becomes stale. Two mechanisms keep that staleness bounded: a **TTL** sets a maximum age after which the value expires on its own, and **invalidation** actively removes or refreshes the cached value the moment the source of truth changes, instead of waiting for the clock.
* **Rule of thumb:** the freshness window should be short enough that stale reads stay rare, but long enough that the cache still absorbs meaningful repeated traffic. Too short, and the cache barely helps; too long, and staleness becomes a real user-facing bug.
* “A longer TTL is always safer.”?
  * Backwards. A longer TTL means longer exposure to stale data every single time the underlying value changes: it's safer for load, not for correctness. A shorter TTL reduces staleness risk at the cost of more cache misses and more backend traffic. Neither direction is free; the right TTL is a deliberate trade-off against how often the data actually changes and how bad a stale read would be.
* There's no universal correct TTL: size it to how fast the real data actually changes. A live inventory count that oversells the moment it's wrong might get 5-15 seconds. A user profile that changes rarely might get 5-15 minutes. A static asset that almost never changes might get hours or days, with invalidation reserved for the rare write that can't wait that long. Invalidation complements a TTL, it doesn't replace it: a TTL bounds the worst case even if an invalidation event is ever missed or delayed.

##### 06. Hot keys and Stampedes: Failure Modes - When one key gets so much traffic that the cache stops helping

* A **hot key** is a single cache entry that receives a wildly disproportionate share of traffic: a trending product, a celebrity's profile, a homepage banner. Caches handle that traffic beautifully right up until the key expires. The instant it does, every one of those requests misses at exactly the same moment and all of them ask the database for the same row at once: a **cache stampede** (also called a thundering herd).

  * A hot key isn't solved by adding more cache nodes: every node would still forward the same simultaneous miss to the database at once. It's a concurrency problem: the fix is making sure only one in-flight request refreshes the value while everyone else waits on that result.
* How do single-flight locking and TTL jitter actually work?

  * Single-flight locking: on a miss, the first request acquires a short lock (commonly held for a few hundred milliseconds, roughly as long as the database query it's protecting takes) and goes to the database. The other 49,999 requests see the lock, wait, and get served the result it fetches instead of each independently querying the database.
  * TTL jitter: instead of every related key sharing one exact expiry (say, a flat 300-second TTL), each key gets a small random offset added, for example 300s ± 30s. That spreads thousands of simultaneous expirations across a 60-second window instead of one single instant, so misses trickle in rather than arriving all at once.
* **Hot keys vs. fanout:**

  * A stampede is many clients hitting *one* key at the same moment. **Fanout** is the opposite shape: *one* request that needs many different keys at once. A celebrity profile page that has to show “posts from the 500 accounts you follow” isn't one cache lookup, it's 500 parallel lookups fired from a single page load. A celebrity with 10 million followers posting once can also fan *out* a single write into millions of downstream feed-cache updates.
  * Both failure modes can hit the same celebrity profile at once: the profile key itself can stampede on expiry, while rendering the page around it fans out into hundreds of other lookups. Fixing the stampede (single-flight, jitter) does nothing for the fanout cost, and vice versa; a design has to name which one it's actually solving.

##### 07. Eviction and Memory Pressure: Capacity - What happens when the cache is full and something has to go.

* A cache's memory is finite, even when the data behind it isn't. Once it fills up, adding a new value means something else has to leave first. Which entry gets picked is the  **eviction policy** : the most common, **LRU** (least recently used), removes whichever entry hasn't been read in the longest time, on the assumption that recent access is the best predictor of near-future access.
* **LRU (Least Recently Used):** Evicts whichever entry hasn't been read in the longest time. Assumes recent access predicts near-future access; weak against a one-time bulk scan that evicts genuinely hot keys.
* **LFU (Least Frequently Used):** Evicts the entry with the lowest total access count. Protects consistently popular keys better than LRU, but can keep a once-viral key that's gone cold for too long.
* **FIFO (First In, First Out):**  Evicts whichever entry was inserted first, regardless of how often it's read. Cheap to implement, but ignores access patterns entirely.
* **TinyLFU:** An approximate, memory-efficient frequency tracker (used by caches like Caffeine) that gets most of LFU's benefit without storing a full per-key counter for every entry. A common modern default in high-throughput caches.
* A cache sized too small for its working set doesn't just run a little slower: it can start evicting a value right before it would have been read again, collapsing the hit ratio into a churn cycle where the cache never settles into steady state.

##### 08. Multi-Layer Caching: Architecture - Browser, CDN and application caches, each absorbing a different layer of load.

* Real systems rarely rely on a single cache. A browser cache, a CDN edge cache, and an application-level cache like Redis can all sit in front of the same database, each absorbing a different slice of repeated load before it ever reaches the layer behind it.
* “More cache layers always means more performance.”?
  * Only if each layer is actually absorbing traffic that would otherwise reach the layer behind it. A CDN cache that's misconfigured to bypass on every request, or an application cache keyed in a way that almost never matches twice, adds latency and operational surface without removing any load from the database. Each layer earns its place by proving it actually has a meaningful hit ratio, the same standard chapter 02 applied to a single cache.
  * **Design idea:** cache the expensive thing, not the whole page. Cache the hot API result, the rendered fragment, or the static asset, not an entire system's worth of state behind one key, which makes invalidation nearly impossible to reason about

##### 09. Check Reliability and Failure Modes: Reliability - What breaks (and what shouldn't) when the cache itself goes down

* Every cache eventually goes down: a node restarts, a cluster fails over, a deploy flushes it empty. What the application does at that moment is a deliberate design decision, not an accident.**Fail-closed** treats a cache error as a hard failure and rejects the request. **Fail-open** treats it exactly like a miss and falls through to the database instead.
* A cache that comes back up empty creates the same problem as an expiring hot key, just at the scale of the entire dataset at once: every key is a guaranteed miss, and if enough traffic arrives before the cache refills naturally, the database sees a stampede it was never sized for. **Cache warming** (proactively loading known-hot keys back in before traffic is allowed to hit the cache) turns that cold start into a controlled, gradual ramp-up instead of a system-wide thundering herd.

##### 10. Cache Design Workflow: Method - A repeatable method for any cache-design interview question.

* A repeatable way to work through any cache-design question, whether it's an interview or a real incident: start from the access pattern, and let the read/write/staleness answers decide the rest.
  1. Identify which reads are repeated often enough to be worth caching
  2. Choose key and value shape, and what the cache actually stores
  3. Estimate hit rate and the real cost of a miss
  4. Choose a write strategy: write-through, write-back, or write-around
  5. Define freshness: TTL length and an invalidation rule
  6. Model thundering-herd risk for any hot keys
  7. Pick an eviction policy and a realistic memory budget
  8. Decide fail-open vs. fail-closed, and whether the database can absorb 100% of traffic
  9. Plan cache warm-up behavior after a restart or deploy

##### 11. Practice labs: Apply it - Four hands-on problems that build on every chapter above.
