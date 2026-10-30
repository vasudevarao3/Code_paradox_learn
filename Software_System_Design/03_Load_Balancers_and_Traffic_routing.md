### Load Balancers & Traffic Routing

* Starting from a plain-English analogy for what a load balancer even is, build up to Layer 4 vs. Layer 7 routing, TLS termination, every major routing algorithm, health checks, graceful failover, and isolated backend pools.

##### 01. What is Load balancer:

* A Load Balancer is a component that distributes incoming requests across multiple servers so that no single server becomes overloaded.

##### 02: What a Load balancer Does: Load Balancer Fundamentals - Listeners, backends, pools, and the forwarding path, from first principles.

* A load balancer is a network service placed in front of backend servers. Clients connect to the load balancer rather than choosing an application server directly.  The load balancer than selects an eligible backend and forwards the work.
* This indirection creates one stable entry point while allowing the backend fleet to grow, shrink, or lose unhealthy members. It does not create backend capacity by itself. It makes existing capacity reachable and controllable.

  * **Load balancer:** A service that accepts incoming connections or requests and forwards each one to an eligible backend.
  * **Listener:** The load balancer endpoint that accepts traffic on a protocol and port, such as HTTPS on port 443.
  * **Backend:** A server instance that can receive forwarded work from the load balancer.
  * **Backend pool:** A named or logical group of backends eligible for a particular class of traffic.
  * **Routing rule:** A condition and action that decide which backend pool receives traffic.
  * **Health check:** A periodic probe used to decide whether a backend should remain eligible for new traffic.
  * **Failover:** Continued service after a component fails, using healthy capacity that remains available.
  * **Active connection:** A connection currently open on a backend. Duration determines how long it contributes to current work.
  * **Traffic Weight:** A relative value that controls how much traffic one backend receives compared with others.
  * **Upstream:** A common proxy term for the backend service or pool to which traffic is forwarded.

**The Forwarding path:**

* A valid path has three stages: traffic arrives, the load balancer makes a routing decision, and a healthy backend handles the work.

##### 03. Choose what the Load balancer needs to understand: Layer 4 vs layer 7 Load Balancer - Exactly what information is visible to the load balancer at each layer

* Layer 4 and layer 7 refer to layers in the network model.
* The distinction is entirely mechanical: a **Layer 4(Application Load Balancer - ALB)** load balancer forwards a **transport connection using addresses and ports and never opens it**, so it cannot see anything about the HTTP request inside. A **Layer 7(Network Load Balancer - NLB)** load balancer terminates the connection and **reads the actual HTTP request** (method, path, headers, cookies) before it decides where to send it.
* ALB:
  * Can inspect IP, port, TCP or UDP connection
  * Can't route to /api
  * Typically used in General TCP services, very high throughput
* NLB:
  * Can inspect HTTP host, path, method, headers
  * Can route to /api
  * Typically used in HTTP APIs, host and path routing, request policies.

##### 04. TLS Termination & the Reverse Proxy Role: Mechanics - Where the certificate lives, and why layer 7 routing needs it there

* A load balancer sitting in front of HTTPS traffic almost always plays a second role: it terminates TLS. The client's encrypted connection ends at the load balancer, not at the backend. The load balancer holds the TLS certificate and private key, decrypts the incoming request, and only then can a Layer 7 routing rule actually read the path, headers, or cookies described in the previous chapter: encrypted bytes can't be inspected for a path match.
* Traffic between the load balancer and the backend is commonly plain HTTP again, since it stays inside a private network. Some deployments re-encrypt that hop for defense in depth, but either way the certificate lives in exactly one place (the load balancer) instead of being copied onto every backend instance.
* Why keep the certificate in one place?
  * A certificate copied onto every backend instance is a certificate that has to be rotated, secured, and audited on every backend instance. Centralizing TLS termination at the load balancer means renewing one certificate updates the whole fleet at once, and it removes the encryption/decryption CPU cost from every individual backend.

##### 5. Choose the signal that represents backend work: Load balancing algorithms - Round Robin, Weighted Round Robin, Least connections, compared.

* A routing algorithm is the rule used to select a backend. The correct algorithm depends on whether servers are equal, whether request cost varies, and whether current connection count is a useful signal.
* **Round Robin:** Cycles through eligible backends in turn. It assumes each request costs roughly the same and each backend has roughly the same capacity. Equal request count. Best for similar servers and similar request cost.
* **Weighted Round Robin:** Cycles according to configured weights. Deterministic, and appropriate when backend capacities differ in known proportions. Configured traffic ratio. Best when backend capacities differ predictably.
* **Least Connections:** Chooses the backend with the fewest active connections. It reacts to long-lived requests that remain open while short requests finish.
* A load balancer just splits traffic evenly, no matter what?
  * Round Robin does, but it's a deliberate default for a uniform fleet, not the definition of load balancing itself. Weighted Round Robin and Least Connections exist specifically because backends are often not identical and requests are often not equal-cost. Picking “even” when the fleet is heterogeneous overloads the smaller or slower-clearing backends while the rest sit under-used.

##### 06. Route in Proportion to Backend Capacity: Weighted load balancing - Routing traffic in proportion to backend capacity, with the math

* Round Robin sends equal request counts. That is unsafe for a mixed fleet because a Small server and a Large server receive the same traffic despite having different limits. Weighted Round Robin lets the routing share mirror capacity.
* Capacity-Proportional Weight:

  * server weight ∝ server rated capacity = 220:430:800:800 becomes 22:43:80:80
* Assigned Traffic:

  * server RPS = total RPS × server weight ÷ total weight = 1,800 × 22 ÷ 225 = 176 RPS for Small

##### 07. Stop Routing Before Spare Capacity Can Help: Health checks and failover - Detecting a dead backend, and calculating capacity after it's gone

* A failed backend must be removed from the eligible pool. Health checks perform this control-loop function by probing each backend and changing its routing state when failures cross a threshold.
* “Health checks alone guarantee zero failed requests during a backend failure.” ?

  * There is always a detection window: the time between a backend actually failing and the health check crossing its failure threshold and removing it from rotation. Any request routed to that backend during the window still fails. Faster checks shrink the window; they cannot eliminate it.
* Health detection and failover capacity are seperate:

  * Health checks prevent requests from reaching a failed server. They do not create replacement capacity. After removal, the remaining servers must still handle the specified failure traffic without crossing their utilization limit.
* What makes a useful health check:

  * Representative: Probe a path that proves the service can perform meaningful work, not only that a process exisits.
  * Cheap: The probe run frequently, so it must not create significant application or database load.
  * Fast enough: Detection must be quick enough for the recovery objective without reacting to one harmless transient error.
  * Independent: Avoid a probe that declares every backend unhealthy because one optional shared dependency is degraded.

##### 08. Graceful Deregustration & Connection Draining: Removing a Backend safely - What happens to in-flight requests when a backend is removed.

* Health checks answer whether a backend is eligible for new traffic. They don't answer what happens to the requests already in flight on that backend the moment it's marked unhealthy, or simply scheduled for planned removal: a deploy, a scale-down, a maintenance window.
* A **hard removal** stops sending requests instantly and drops any connection already in progress mid-response. **Connection draining** (also called graceful deregistration) stops sending *new* requests immediately, but lets requests already in flight finish normally, up to a configured drain timeout. Only once every in-flight request completes, or the timeout expires, is the backend fully removed.
* Drain timeout heuristic
  * drain timeout ≥ slowest expected request duration
  * Too short and draining degrades into a hard removal anyway

##### 09. Request Count Is Not Current Work: Connection-Aware Routing - Why request count and active work are not the same signal

* Round Robin equalizes arrivals, not active work. When all requests finish in similar time, that distinction may not matter. When some requests remain open far longer, one backend can accumulate more active work than another even after receiving the same number of requests.
  * Round Robin distributes arrivals evenly. It cannot distribute active work evenly, because arrivals and active work are only the same thing when every request takes the same amount of time.
    * Fast-request concurrency	= 	1,800 RPS × 0.05 s = 90 active requests
    * Export concurrency			= 	15 RPS × 8 s = 120 active exports
    * Total average concurrency	=	90 + 120 = 210 open requests
    * Only 15 of 1,815 arrivals per second are exports, yet exports account for 120 of 210 active requests.
* Least Connections uses open connection count as a live approximation of backend work. It does not replace capacity planning. The fleet still needs enough RPS capacity, and connection count is only useful when it correlates with resource usage.

##### 10. Route Each Workload to Dedicated Capacity: Path-Based Routing and Backend Pools - Routing different URL paths to dedicated, isolated backend pools.

* A Layer 7 routing rule can inspect the URL path and choose a backend pool. This allows one domain to serve multiple workloads while keeping their compute capacity independent.
* **Pool Isolation:** A backend is dedicated only when it belongs to one pool. Connecting the same application server to two route pools creates shared capacity and allows one route to affect the other. Isolation requires separate membership, separate sizing, and separate utilization checks.

##### 11. A Repeatable interview Method: Routing Design Workflow - A Repeatable method for any traffic-routing design question

* Separate topology, algorithm, health, and capacity decisions. A correct answer should explain what traffic is being routed, which signal drives selection, which backends are eligible, and what happens when one fails.

  1. Identify the traffic unit: state whether the input is requests, connections, jobs, bytes, or sessions, and keep units attached to every calculation.
  2. Choose Layer 4 or Layer 7: use Layer 7 when routing depends on HTTP host, path, method, or headers, and Layer 4 when transport-level forwarding is sufficient.
  3. Decide where TLS terminates: if the routing rule needs to read the request, TLS has to terminate before that decision is made.
  4. Define backend pools: list which servers may handle each traffic class, and decide whether capacity is shared or isolated.
  5. Choose the routing algorithm: Round Robin for a uniform fleet, Weighted Round Robin for known capacity differences, Least Connections for varying connection duration.
  6. Calculate normal capacity: verify aggregate throughput and every backend's utilization under the selected traffic split.
  7. Define health and eligibility: specify how failed backends are detected and removed from rotation, and how in-flight requests are drained on planned removals.
  8. Calculate failure capacity: remove the failed backend from the math and verify the remaining fleet serves the required failover load.
  9. Check isolation and cost: verify no backend is unintentionally shared, and that every load balancer and server fits the monthly budget.
* Do not trust aggregate capacity: A bad traffic split can overload one backend while total fleet capacity looks sufficient.
* Do not ignore duration: Equal arrival counts do not imply equal active work when requests have different lifetimes.
* Do not fake isolation: A backend shared by two pools is still a shared failure and capacity domain.

##### 12. Prctice labs: Apply Each Traffic-Routing Concept: Four hands-on problems that build on every chapter above
