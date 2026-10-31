# Application State and Sessions

Starting from what a cookie physically is, build up to session affinity, shared storage, Redis, and WebSocket connection state, and where each one actually lives when a server restarts.

##### 01. What is Application State? : Foundations - State, Statelessness and why they matter

* Start with plainest version of this: imagine a coffee shop with one barista who remembers every regular's usual order by heart. That works great, until a second barista opens a register and a regular orders from them instead. The second barista has no idea what “the usual” means, because that memory lived in one person's head, not somewhere both baristas could check.
* That's exactly the problem with a web server that keeps things in its own process memory. **State** is any information that has to survive beyond a single request: a shopping cart, a login, who owns a live chat connection. Local process memory is fast, but another server can't see it, and a restart erases it completely.
  * The question to always ask isn't “can this server remember it?” A single process can remember anything, briefly. The real question is **where does the system keep the truth, and how does every other server find it?**
* **State:** Information that must be remembered across more than one request or operation
* **Stateless server:** A server that can handle any request without relying on private data left behind by an earlier request on that same process.
* **Durable:** Able to survive a process restart, a server dying, or the machine being replaced entirely.
* **Horizontal scaling:** Adding more servers, rather than making one server bigger. This is why statelessness matters so much.

##### 02. Cookies, from Scratch: Mechanics - what a cookie actually is on the wire, and the flags that keep it safe

* Before sessions make sense, it helps to know exactly what a cookie is: not the metaphor, the actual bytes. A cookie is a small piece of text. That's it. The server sends it once in a response header, the browser stores it, and the browser automatically attaches it to every later request to that same site. No magic, no special protocol: just a header the browser is contractually obligated to remember and resend.
* “A cookie is where the session data itself lives.”?
  * It almost never is. A well-designed cookie holds only an opaque, meaningless-looking ID. The actual session data (who this user is, what's in their cart) stays on the server. If the cookie held real data directly, anyone could edit it in their browser and hand themselves admin access.
* Because the browser sends the cookie back automatically on every request, including ones triggered by other sites embedding a link to yours, cookies need explicit flags to stay safe:
  * **Secure:** Only ever sent over HTTPS, never plain HTTP
  * **HTTP ONLY:** Invisible to JavaScript: malicious page scripts can't read it
  * **Same Site:** Controls whether other sites can trigger it being sent
  * **Max-Age / Expires:** How long the browser keeps it before discarding it

##### 03. Sessions and Session IDs: Foundations - How a session ID lets the server recognize the same visitor request after request

* A **session** is server-side state tied to one visitor across many requests. Logging in creates a session; the server generates a random **session ID** , stores the session data keyed by that ID, and sends the ID back in a cookie. Every later request presents that ID, and the application looks up the matching session to know who's asking.
* Why does the ID need to be unpredictable?
  * If session IDs were sequential (session 1, session 2, session 3...), anyone could guess someone else's ID and impersonate them. Session IDs are generated using a cryptographically random source specifically so guessing one is computationally infeasible.

##### 04. Session Affinity and Sticky Sessions: Routing - Sticky routing, what it fixes, and what it silently doesn't

* If session data lives only in one server's local memory, that server is the only one who can answer that client's requests correctly. **Session affinity** (also called sticky sessions) is a load-balancer policy that pins one client to one backend so local-memory sessions keep working, at the cost of routing flexibility.
* Random session lookup without affinity
  * success rate ≈ 1 ÷ number of servers
  * A 4-server fleet: only 1 ÷ 4 = 25% of a client's follow-up requests land back on the server holding its session
  * That 25% figure is a routing problem: the session is sitting safely on Server 1, just unreachable 75% of the time. Chapter 07 covers the sharper version of the same constraint: what happens to that session when Server 1 doesn't just become unreachable, but fails outright.

##### 05. Shared Session Storage: Design - Externalizing state so any server can serve any request

* Externalizing session state into a shared store gives every server the same authoritative lookup. Any healthy server can then handle any request, a deployment can safely replace instances one at a time, and session affinity becomes an optimization instead of a requirement.
* “Moving sessions into a shared store is free: it just removes the affinity problem.”?

  * It removes a routing problem and introduces two new ones: every request now pays a network round trip it didn't pay before, and the store itself becomes a dependency every single server shares. Externalizing state is almost always the right trade, but it is a trade, not a strict improvement.
* **The cost: one extra network hop, every request**

  * A local in-process memory lookup costs a fraction of a millisecond: effectively free. A lookup against a shared store adds a real network round trip on top of it.
  * Local memory lookup - read from the same process's RAM

    * ~0.01-0.05 ms: effectively free
  * Shared-store lookup - network round trip to Redis on the same private network

    * ~1-2 ms added per request
  * 1-2 ms sounds negligible next to a 100ms request, and usually is. It stops being negligible once a single request fans out into several session checks (an API gateway, then a downstream service, then a background job), or once the store itself is under load and that 1-2 ms becomes 20-30 ms during a spike.
* **Sizing the store: every request is now a store request**

  * Once session state moves off the application server, the shared store has to sustain roughly one operation per request across the entire fleet, since nearly every request checks the session at least once.
  * Store throughput requirement
    * store ops/sec ≈ total fleet request rate
    * A fleet handling 100,000 RPS needs a session store that can sustain roughly 100,000 reads/sec, before counting writes
* **What happens when the store itself goes down**

  * A local-memory design fails one server at a time: the rest of the fleet is unaffected. A shared store is a single dependency every server now relies on, so when it goes down, every server loses session access at once, not just one-N'th of the fleet. Whether that means every user gets logged out (fail-closed) or every request is treated as a fresh, anonymous visitor (fail-open) is a deliberate design decision, not an accident, the same choice Caching makes for a cache outage.

##### 06. What is Redis? Mechanics - In-memory key-vaue storage, TTLs, and why it's the default choice here

* Redis is an  **in-memory key-value store** : a database that keeps everything in RAM instead of on disk, which is what makes it fast enough to sit on the critical path of every request. You give it a key (a string like **session:a91f**) and a value, and later ask for that same key back. That's the entire mental model.
* The other reason it fits sessions so well: Redis supports a**TTL** (time-to-live) on any key. Set a session with a 3,600-second TTL, and Redis deletes it automatically an hour later, exactly the expiry behavior a session needs, with no cleanup job required.

##### 07. Session Durability and Failover: Reliability - Correct routing now vs surviving a server actually dying

* Correctness asks whether a request can find its session  *right now*. Durability asks whether that session still exists *after* the process that owned it fails. Sticky routing only answers the first question.
* “Sticky sessions and durable sessions are the same thing.”?

  * Sticky sessions solve routing: the same client always reaches the server holding its data.
  * Durability is a separate property: does that data survive the server crashing? Affinity alone gives you zero durability, since the crashed server takes the session with it.
* **Local state: one failure, one server's share gone**

  * Chapter 04 showed that without affinity, only 1 ÷ N of a client's requests happen to land on the server actually holding its session. A crash makes that same constraint permanent: whichever server dies takes its share of sessions with it, and they are never coming back because no other copy ever existed.
  * **Local-state loss on server failure**
    * session loss ≈ 1 ÷ server count
    * On the same 4-server fleet from chapter 04: one crash permanently destroys 1 ÷ 4 = 25% of all active sessions
* **Shared storage doesn't automatically mean durable**

  * Moving sessions into Redis fixes the routing problem from chapter 04 and chapter 05 completely: every server shares one lookup, so an application-server crash loses 0% of sessions. That is a real, useful fact, and it's easy to stop there. But it only moves the single point of failure, it doesn't remove it:  **Redis itself is not durable by default** . A default Redis instance keeps everything in RAM and writes nothing to disk; if that one process crashes or the host reboots, every session it held is gone.
    * A naive, single-node Redis session store has simply relocated the single point of failure from “one application server” to “one Redis process.” Durability has to be engineered in deliberately: it doesn't come free just because the state is now “shared.”
* **RDB snapshotting:** Periodically writes the full dataset to disk. Fast to restore from, but any writes since the last snapshot are lost on a crash.
* **AOF (append-only file):** Logs every write operation to disk as it happens. Far less data loss on a crash than RDB alone, at the cost of extra disk I/O per write.
* **Replication:** One or more replicas hold a live copy of the primary's data, so a replica can be promoted if the primary dies, instead of losing the dataset outright.
* **Single-node reality:** RDB and AOF protect against a restart losing data, not against the one node being unreachable. Only replication (plus failover) removes that node as a single point of failure.
* Traffic failover is a third, separate concern again: health checks have to remove the dead server from rotation, and the remaining fleet needs enough spare CPU capacity to absorb its share of the load. For the store itself, that means a replica promotion has to happen, and clients need to reconnect to whichever node is now primary.

##### 08. Redis Read and Write Scaling: Capacity - Why Replicas help reads but never help writes.

* Session workloads are usually read-heavy: a session is checked on nearly every request, but written far less often. Read replicas add read capacity in parallel. Writes still flow through one primary, so read and write utilization must always be calculated separately.
* Read utilization = reads per second ÷ total read capacity × 100 = 110,000 ÷ 135,000 ≈ 81.5% across the primary and both replicas
* Write utilization = writes per second ÷ primary write capacity × 100 = 20,000 ÷ 18,000 ≈ 111.1%: over capacity, even though reads have headroom

##### 09. WebSocket Connection State: Real-Time - Long lived connections need a registry, not just a cookie.

* Every connection discussed so far is a normal HTTP request: open, respond, close, repeat.  **A WebSocket is different** : it's a single connection the client and server keep open for minutes or hours, so either side can push a message at any moment without the other having to ask first. That's what makes live chat and real-time notifications possible.
* The catch: that open connection is pinned to one specific server process the whole time it's alive. If another service needs to push a message to that user, it has to know which gateway process currently holds their connection: a lookup no cookie or database row alone can answer, because the connection itself lives in one process's memory.
* Gateway fleet capacity = instances × connections per instance = 4 gateways × 50,000 connections each = 200,000 total
* N-1 capacity check = total capacity − largest gateway's capacity ≥ active connections = 200,000 − 50,000 = 150,000 ≥ 130,000 active connections: survives losing one gateway
* Why does the registry matter more than the connection count math?
  * The capacity math only tells you the fleet has room for a lost gateway. It doesn't tell you what happens to the 50,000 clients that gateway was holding: those connections drop, and every one of those clients has to reconnect and get a fresh entry written into the registry pointing at whichever gateway they land on next. The registry isn't just a lookup table, it's the thing that makes a dropped connection recoverable instead of silently stuck.

##### 10. State Design Workflow: Method - A repeatable method for any state-design interview question

* A repeatable way to work through any state-design question, whether it's an interview or a real incident: start by naming every piece of state, who owns it, how long it lives, and what happens the moment its owner disappears.
  1. Identify every piece of state and its expected lifetime
  2. Separate the client-carried identifier from the trusted server-side data
  3. Decide whether servers must be interchangeable
  4. Choose affinity, shared storage, or both
  5. Size read and write throughput separately
  6. Test what happens on process failure and node failure
  7. Calculate N-1 capacity for the remaining fleet
  8. Check topology, replication, and cost

##### 11. Practise Labs: Apply it - Four hands-on problems that build on every chapter above
