# Partitioning and Sharding

Starting from why one node can't hold everything, build up to shard keys, hot-key skew, rebalancing, consistent hashing, and cross-shard queries, without creating hot shards, expensive scatter-gather queries, or brittle rebalance logic.

##### 01. One Node Can't Hold Everything: Foundations - Why partitioning exisits at all, from a plain-English analogy up

* Picture a single filing cabinet holding every customer record a company has, A through Z. One clerk works the drawer: every lookup, every new form, every correction goes through that one clerk and that one drawer. It works fine at first. Then the company grows, and the line to reach that one drawer never stops.
* That is exactly the problem with one database node holding an entire table. A bigger machine buys some headroom (more RAM, a faster disk, more CPU cores) but there is a ceiling, and every request still funnels through one process, one disk, and one lock at a time.**Partitioning** is the fix: split the records into smaller, independent groups, and give each group its own cabinet, its own clerk, its own line.
  * Partitioning isn't primarily about running out of disk space. It's about running out of **throughput** : the number of reads and writes one node's CPU, memory, and disk can push through per second. Splitting the data is what lets multiple nodes push through work in parallel.
* **Partitioning:** Splitting a table or database into smaller subsets so no single node has to handle all of the data.
* **Sharding:** A form of horizontal partitioning where rows are assigned to shards based on a key.
* **Shard Key:** The field or function used to decide which shard owns a row.
* **Hot key:** A key that receives a disproportionate share of traffic and overwhelms the one shard it lives on.
* **Range Sharding:** Rows are partitioned by contiguous value ranges, such as user ID ranges or time buckets.
* **Hash Sharding:** Rows are partitioned by a hash of the key so distribution is roughly even across shards.
* **Consistent hashing:** A hashing scheme where adding or removing a shard only remaps the keys nearest to it, not the whole dataset.
* **Cross-shard join:** A query that touches data on multiple shards, generally much harder and slower than a single-shard query.
* **Rebalancing:** Moving data between shards as capacity, throughput, or shard count changes.
* **Read fan-out:** A single request touching many shards to collect results, which can multiply tail latency.
* **Scatter-gather:** The full pattern of fanning a query out to every shard, waiting for all of them, then merging the partial results into one answer.

##### 02. Sharding and Hash Keys: Mechanics - how a shared key gives row exactly one logical home

* Sharding chooses a **shard key** (usually a column already present on every row, like **user_id**) and assigns each row to exactly one shard based on that key. A row for user 42 should consistently land on the same shard every time. That consistency is what makes routing a stateless, one-step calculation instead of a lookup table that itself has to be kept in sync.
  * Total cluster capacity = per-shard capacity × number of shards = 4 shards × 5,000 writes/s each = 20,000 writes/s, on paper
* The formula above is only true when work is spread evenly. The real design decision is choosing a shard key that matches how the system actually reads and writes data. If a disproportionate share of traffic targets one key, the system still has one hot shard, regardless of how many shards exist in total.

##### 03. Range Sharding vs Hash Sharding: Strategy - The same dataset, two boundary strategies, two different failure modes.

* There are two fundamentally different ways to decide the boundary between shards, and they trade off against each other directly.
  * **Range sharding** assigns contiguous ranges of the key to each shard: IDs 1 through 1,000,000 on shard 1, the next million on shard 2, and so on.
  * **Hash sharding** instead runs the key through a hash function first, so which shard a row lands on looks essentially random.
  * Range sharding wins at cheap range scans and time-ordered queries, at the risk that the newest data always lands on one shard. Hash sharding wins at even write distribution across all shards, at the cost of losing cheap range queries entirely: “users 5 through 8” now means asking every shard and discarding most of what comes back.
* Neither one is universally “correct.” A time-series analytics table that's almost always queried by date range benefits from range sharding despite the hot-shard risk on recent data. A high-throughput write path with no natural range-query pattern is usually better off hashed.

##### 04. Hot keys and Skew: Failure Mode - Why an evenly-sized cluster can still have one shard on fire

* Hot keys are the classic shard-killer. A single popular user, product, or region can dominate one shard and create a throughput bottleneck even when the rest of the data is spread perfectly evenly. That is a **skew** problem, not a total-capacity problem: the cluster has plenty of spare capacity, just not where the traffic is landing.
* “Hash sharding eliminates hot keys.”?
  * Hashing spreads *different* keys evenly across shards; it does nothing to protect against *one* key getting an outsized share of traffic. A single viral post, a celebrity account, or a flash-sale product is still exactly one key, and it still hashes to exactly one shard. Hashing distributes the dataset; it can't distribute a single hot row's traffic across multiple machines.
* Put a number on it: a 3-shard cluster each rated for 10,000 writes/s has 30,000 writes/s of total capacity on paper. If one key drives 9,500 of those writes/s onto Shard 2 alone, Shard 2 is already at 95% utilization while Shard 1 and Shard 3 sit under 10%. That is the symptom worth watching for: **one shard pinned above roughly 80% utilization while its neighbors sit idle** , regardless of how healthy the cluster-wide average looks. The actual design goal is simpler to state than to achieve: pick a shard key that spreads work roughly evenly across every shard, not just evenly across the dataset's storage.

##### 05. Rebalancing and Routing: Operations - What actually happens, physically, when a shard has to split.

* As data grows, a shard must eventually be split; as traffic shifts, the system may need to move keys off an overloaded shard entirely. Rebalancing is real operational complexity: rows physically move between machines, routing tables change mid-flight, and some queries may see temporary imbalance or partial unavailability while the move is in progress.
* **Rule of thumb:** a shard key should be durable and stable. If the system needs to remap the key often, that's usually a sign of a design problem with the key choice, not just an operational inconvenience to route around.
* What happens to a write that arrives mid-move?
  * The source shard usually keeps accepting writes the whole time it runs. A common approach is dual-write during the migration window: new writes land on both the old and new location until the copy catches up, then routing flips atomically to the new shard and the old copy is dropped. Skip this and a write that lands between the copy finishing and routing flipping over can silently vanish, applied to a shard nothing is reading from anymore.

##### 06. Consistent Hashing: Mechanics - Why adding shard 17 shouldn't require re-copying shards 1 through 16.

* The naive way to hash-shard is **hash(key) mod N**, where N is the shard count. That works fine until N changes. Adding a single shard changes the divisor for every single key, which means almost every key's answer changes too; the cluster ends up re-copying nearly the entire dataset just to add one machine.
* **Consistent hashing** fixes this by hashing both the keys and the shards onto the same conceptual ring of values, and assigning each key to the next shard found walking clockwise from it. Adding a shard only inserts one new point on that ring; it only takes over the keys between itself and its nearest neighbor, leaving everyone else's assignment untouched.
  * Only the keys nearest the changed shard ever move. Everyone else's assignment stays exactly where it was, which is the entire reason consistent hashing lets a cluster resize without a full-dataset migration.

##### 7. Cross-Shard Queries: Query Patterns - What a scatter-gather query looks like next to a single-shared one.

* Sharding helps throughput, but it makes some queries strictly harder. If a request only needs data for one shard key (“this user's cart”) it stays a fast, single-shard lookup. If a request needs data spread across every shard (“total revenue this week”) the application has to run a **scatter-gather** query: fan the request out to every shard, wait for all of them to respond, and merge the partial results into one answer.
* scatter-gather query: `latency ≈ slowest of N shards, not the average`
* A single-shard query stays fast, local, and one predictable round trip, exactly as if sharding never happened. A scatter-gather query across 12 shards where 11 answer in 8 ms and one is briefly slow and answers in 400 ms doesn't average those out to something reasonable: the coordinator waits for every shard, so the whole query takes 400 ms, bounded by the single slowest participant. This is one reason many systems are designed around single-shard access patterns from the start: users read their own data, products are read by product_id, or events are partitioned by tenant so a tenant's dashboard never has to fan out at all.

##### 8. Capacity and Shard counts: sizing - Why average utilization across shards can hide the real bottleneck

* Capacity planning has to include both storage and concurrency. If each shard can handle a known amount of throughput, total system capacity is roughly per-shard capacity multiplied by shard count. That number is easy to calculate, and easy to present as reassuring when it hides a skewed distribution underneath it.
* “More shards always means more capacity.”?
  * Only if the shard key spreads work evenly. If one key is hot, adding shards just adds idle machines: the hot shard still bottlenecks the whole system at the same ceiling it had before, because that one key's traffic never had anywhere else to go. Doubling the shard count doesn't double capacity when the actual constraint is skew, not raw node count.
* Total write capacity: shards × per-shard write capacity
  * 3 shards × 10,000 writes/s = 30,000 writes/s on paper
  * That Multiplication only describes the ceiling if work is spread evenly. A hot key can collapse the effective capacity straight down to one shard's ceiling: Figure 9's cluster still has 30,000 writes/s on paper, but Shard 3 alone caps the system's real throughput at its own 10,000 writes/s the moment it hits 97% utilization, no matter how idle Shards 1 and 2 are.
* It isn't enough to say “we have 16 shards.” A real design has to explain why the key distribution is even and why no single tenant, key, or traffic pattern is quietly dominating one shard.

##### 9. Sharding Design Workflow: Method - A repeatable method for any shard-design interview question.

* A repeatable way to work through any shard-design question, whether it's an interview or a real capacity incident: start from how the system actually reads and writes data, then define the shard key and estimate skew before ever picking a shard count.
  1. Name the hottest access patterns first
  2. Choose a stable, high-cardinality shard key
  3. Check for hot-key and time-clustering skew
  4. Estimate per-shard capacity and total shard count
  5. Model which queries become cross-shard scatter-gathers
  6. Plan rebalancing, splitting, and routing behavior
  7. Define failure and partial-availability behavior
  8. Document operational limits and rebalance triggers

##### 10. Trade-offs and Pitfalls: Reality Check - What sharding buys you, and what it quietly costs.

* Sharding solves exactly one problem (a single node running out of throughput) and adds several more: coordination overhead, skew risk, and real operational work. None of that makes sharding a bad idea, it makes it a trade, and every trade here has a specific cost sitting directly across from its specific benefit.
* | Dimension    | What sharding Buys                                                                                                           | What it costs                                                                                                                                            |
  | ------------ | ---------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | Throughput   | Near-linear write/read capacity growth as shards are added, since each shard handles its own slice of traffic independently. | Only if the shard key actually spreads load evenly. A hot key still caps the whole system at one shard's ceiling regardless of shard count (chapter 04). |
  | Query shape  | Single-shard reads stay exactly as fast as an unsharded table.                                                               | Any query that doesn't know the shard key becomes a scatter-gather, bounded by the slowest of N shards, not the average (chapter 07).                    |
  | Growth       | A stable, high-cardinality key lets capacity grow smoothly over time.                                                        | Moving data between shards is real migration work: copying rows, tracking in-flight writes, and flipping routing atomically (chapter 05).                |
  | Blast radius | Shards fail independently; losing one doesn't take the whole dataset down.                                                   | A transaction or join spanning two shards no longer gets the single-node guarantees it used to get for free.                                             |
* Put a number on the whole trade at once: sharding a single 1,000 writes/s node into 10 shards of 1,000 writes/s each buys 10,000 writes/s of total capacity (chapter 02's formula). But a dashboard query that used to be one 40 ms lookup is now a 10-way scatter-gather bounded by whichever shard answers slowest, plus whatever rebalancing work keeps those 10 shards from drifting unevenly in the first place (chapter 05). The write win is real; it was never free.

##### 11. Practice Labs: Apply IT - Four hands-on problems that build on every chapter above
