# Database Scaling and Replication

Starting from what replication physically is, build up to WAL streaming, the synchronous-vs-async trade-off, read scaling, lag, failover, and write amplification, and where each one quietly runs out.

##### 01. What is Replication? (From Scratch): Foundation - What replication really is, from a master-ledger analogy up.

* Picture a company with one master ledger kept at headquarters, and branch offices that need to answer questions like “what's this customer's balance” without phoning headquarters for every single question. The company's fix: after every new ledger entry, headquarters mails a photocopy of just that entry to each branch. A branch clerk keeps a growing stack of these copied pages and answers questions straight from their own copy: fast, local, no call to HQ required.
* That is replication. A **primary** database is the ledger of record: every write lands there first. A **replica** is a branch office: it never invents its own entries, it only receives and applies copies of entries that already happened on the primary, in the same order they happened. The moment you introduce a copy, you also introduce a delay: the photocopy takes a moment to arrive and get filed. That delay is where most of the hard problems in this guide come from.
* Once you have replicas, you get two benefits almost for free: extra machines that can answer read queries, and a standby copy that can take over if the primary dies. This guide covers both, and exactly where each one quietly runs out.
* **Replication:** Copying changes from one database to one or more other databases so they can serve reads or survive failover.
* **Primary:** The leader database that accepts writes and is the canonical source of truth.
* **Replica:** A follower that copies a primary and serves reads or takes over during failover.
* **Lag:** The delay between a write reaching the primary and that same write appearing on a replica.
* **WAL:** Write-ahead log (binlog in MySQL): an ordered, durable record of every committed change, streamed to replicas so they can replay it in the same order.
* **Synchronous replication:** The leader waits for a follower to acknowledge before committing; stronger safety, higher latency.
* **Asynchronous replication:** The leader commits quickly and pushes changes later; higher throughput, more lag exposure.
* **Semi-synchronous replication:** The leader waits for just enough replicas to satisfy a quorum, not every follower: bounded latency, stronger durability than fully async.
* **Quorum:** The minimum number of replicas that must acknowledge a write before it counts as durable, e.g. 2 of 3 copies.
* **Read-after-write:** A consistency rule where a user's immediate read is sent to the primary or a fresh replica.
* **Failover:** Promoting a healthy replica to primary after the leader is unavailable.
* **Write Amplification:** The cost of fan-out writes to many replicas or multiple storage paths.

##### 02. How Changes Actually Ship: WAL & Binlogs - What actually ships from primary to replica: a log of changes, not fresh snapshots.

* It is tempting to picture replication as “the replica periodically downloads a fresh copy of the whole database.” That would be enormously wasteful: for a database of any real size, re-copying everything for every small change would saturate the network before it saturated anything else. Real replication ships something much smaller: an ordered log of the changes themselves.
* Every write a primary commits is first appended to a durable, sequential log: Postgres calls it the  **write-ahead log (WAL)** , MySQL calls it the  **binlog** . A replica does not ask “what does the whole table look like now?” It asks “what has changed since the last entry I applied?”, and the primary streams the next entries in that log. Each replica applies those entries in the exact order they were written, which is what keeps a replica's copy an accurate (if slightly delayed) history of the primary.

  * What ships between primary and replica is a log of changes, not a fresh copy of the database. That single fact is why replication is cheap enough to run continuously: a 2 TB database might only ship a few megabytes of WAL per second under normal write load, nowhere close to re-sending 2 TB on every change.
* What if a replica falls too far behind to catch up from the log?

  * Primaries cannot keep an infinite log around; old WAL/binlog segments eventually get purged to reclaim disk. If a replica has been disconnected long enough that the entry it needs has already been purged, incremental streaming cannot recover it. The fix is a full resync: rebuild that replica from a fresh base backup of the primary and resume streaming from there, effectively starting over. Chapter 09 puts a concrete number on how much retention is enough.

##### 03. Synchronous vs Asynchronous Replication: Trade-off - The real latency vs durability trade off behind every replication topology

* **Asynchronous:** the primary commits locally, replies to the client immediately, and pushes the change to replicas afterward. Writes are fast, but there is a window (sometimes milliseconds, sometimes longer under load) where a committed write exists only on the primary. If the primary dies in that window, that write can be lost even though the client was already told it succeeded.
* **Synchronous:** the primary waits for at least one replica to confirm it has durably received the write before replying to the client. Nothing is acknowledged as done until a copy exists somewhere else. That closes the data-loss window, at the direct cost of added latency on every write: now every commit pays for a network round trip to a follower.
* “Synchronous replication eliminates replication lag.”?
  * It doesn't: it eliminates the data-loss window, not the lag itself. A synchronous replica still takes time to receive and apply the WAL entry, it just makes the primary wait for that confirmation before replying to the client. Asynchronous and synchronous replicas can end up with nearly identical lag under normal load; the difference only shows up the moment the primary crashes mid-write.
* Most production systems land in between: semi-synchronous replication waits for acknowledgment from just enough replicas to satisfy a **quorum** (say, 2 of 3) instead of every follower. That bounds the added latency to whichever replica in the quorum responds fastest, while still guaranteeing the write survives on more than one node.
  * Synchronous replication trades latency for a durability guarantee; it does not remove replication lag, it just refuses to acknowledge a write until enough copies exist to survive losing the primary.

##### 04. Read Scaling with Replicas: Capacity - Turning idle replicas into extra read capacity

* Read scaling is the most common reason to add replicas. If a primary is saturated by read-heavy traffic, you can fan out selected queries to follower nodes that are streaming the primary's WAL/binlog. This raises the total number of read requests the database tier can satisfy without making the primary do every read.
* Read capacity = primary capacity + replica capacity 1 + replica capacity 2 + ...  = 3k (primary's leftover) + 20k (replica 1) + 20k (replica 2) = 43k reads/s total

##### 05. Replication Lag and Read-After-Write Consistency: Consistency - Why a replica is only ever eventually consistent and how to fix it for a user's own writes.

* Replication lag is the gap between “the write was accepted on the primary” and “the same write is visible on a follower.” A system that sends all reads to replicas can return stale data. That is acceptable for analytics dashboards or cache warming, but not for order history, account balance, or “my last action just happened.”
* That last case is specific and common enough to name: users expect their own next read to show the action they just triggered. That is a product requirement, not a database default, and it is not free: a **read-after-write** policy has to deliberately route that one read to the primary, or to a replica proven caught up, instead of to whichever replica happens to be least loaded.
* Lag budget
  * acceptable staleness depends entirely on what's being read
  * ~50 ms: invisible to users · ~500 ms-2 s: visible staleness risk on 'did my action just happen' flows · >30 s: usually a broken replica, not normal lag
* The usual fix is a short read-after-write window: after a user sends a write, their next reads go to the primary (or a replica confirmed fresh enough) until the normal replicas have caught up. Route sessions to that consistent source for a brief period after their own write, then allow replica reads again once the change is safely replicated. This keeps the experience correct without throwing away the benefit of read replicas for everything else.
  * There are really only two consistency levels a replicated system forces a choice between: **strong local consistency** , where a user's own write is guaranteed visible on their own very next read because that read is pinned to the primary (or a replica proven caught up), and  **eventual consistency** , where any replica may be stale for some bounded time. Most production systems mix both: eventually consistent by default, with a short strong-consistency window immediately after a user's own write.

##### 6. Failover and Active-Passive Design: Reliability - Promoting a replica safety, and knowing what you might lose

* Failover means promoting a replica to primary after the leader is down or unhealthy. But failover is not just a database switch. The load balancer must route writes to the new primary, clients need new connection targets, and the system must know whether the promoted node has enough data to accept writes safely.
* "Failover means zero data loss, automatically.”?
  * Only if the replication behind it was synchronous (or at least semi-synchronous with a satisfied quorum). With plain asynchronous replication, the promoted replica may be missing the last few writes that had not shipped yet when the primary died; those writes are gone. Zero-loss failover is a deliberate design choice you pay write latency for, not a free property of “having a replica.”
* Put a number on it. A primary doing 2,000 writes/sec crashes with an asynchronous replica that was 150 ms behind at the moment of death. That 150 ms window holds roughly 300 writes (2,000/sec × 0.15 s) that were acknowledged to clients but never shipped: those are gone the instant that replica gets promoted. Availability and safety are two separate questions here, not one:**availability** asks whether some node is willing to accept writes right now (a healthy primary, or a promoted replica), while**safety** asks whether that node has enough of the primary's data to promote without losing writes like those 300.

##### 07. Write Amplification and Bottlenecks: Bottlenecks - Why more replicas never raises the write ceiling

* All writes still hit the primary. A replica is not a free second writer. It must receive, validate, and store the same changes, which uses network, CPU, and disk. In other words, replication can scale reads, but it does not make the leader write path disappear.

  * Replicas scale reads, never writes. More replicas means more parallel read capacity, not a higher write ceiling: the primary is still the only node accepting new writes, and every replica still has to apply that exact same stream in full just to keep up.
* “Adding more read replicas also scales how many writes the system can handle.”?

  * Every replica still has to apply the exact same stream of writes the primary produced; replicas parallelize reads, not writes. The write ceiling is set by the primary's own capacity, and in synchronous setups, by however long the slowest replica in the quorum takes to acknowledge.

##### 08. Replication Design Workflow: Method -A repeatable method for any replication-design interview question

* Use a repeatable workflow. First decide which workloads need fresh data and which can tolerate lag. Then decide whether reads need scale and which writes are mission critical. After that, size the replication topology and the failover path.

  1. Identify write-heavy and read-heavy paths
  2. Decide which reads need fresh data
  3. Pick leader and follower topology
  4. Set lag tolerance per workload
  5. Define failover trigger and promotion rules
  6. Choose synchronous, semi-synchronous, or asynchronous replication
  7. Budget network and disk for replica writes
  8. Confirm WAL/binlog retention covers your worst-case replica downtime
  9. Document read-after-write exceptions
* What if step 4 (lag tolerance) and step 2 (reads needing scale) pull in opposite directions?

  * They often do. A reporting workload wants cheap, heavily-cached replica reads and can tolerate minutes of lag; the same system's checkout confirmation screen needs that lag near zero. Resolve this per read path, not once for the whole system: route the report to any caught-up replica, and pin checkout confirmation to the primary or a replica proven fresh, exactly as chapter 05 lays out. Trying to pick one lag tolerance for the entire database usually means over-paying for the reporting path or under-serving the checkout one

##### 09. Read Replicas in Production: Operations - The Operational controls real replicated systems still need, with concrete threshold.

* Replicas are powerful, but they also add operational complexity: lag, failover drift, stale read exceptions, and a higher chance of user-visible inconsistency if the load balancer is misconfigured. The safest systems write these thresholds down as numbers, not vibes.
* Lag alerting thresholds
  * pick concrete bands, don't just watch the raw number
  * < 5s: normal, no action · 5-60s: page on-call, investigate · > 60s: treat as a failed replica, pull it from read routing
* WAL / binlog retention
  * retention window ≥ worst-case replica downtime you plan to tolerate
  * 4 hours of retained WAL covers a routine maintenance restart; a replica down longer than that needs a full resync instead of catching up
* Failover timeout budget
  * detection + decision + promotion + reconnect = total user-visible downtime
  * 3 failed health checks at 5s apart (15s) + promotion (~10s) + client reconnect (~5s) ≈ 30s budgeted outage
* Replica Lag: Reads may be stale; use read-after-write windows and dedicated freshness checks
* Failover safety: Validate replication health before promoting a replica
* Read Routing: Not all read requests should go to replicas
* Monitoring: Track lag, CPU, IOPS, and promotion readiness continuously

##### 10: Practice Labs: Apply It - Four hands-on problems that build on every chapter above
