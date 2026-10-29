# [Servers and Compute](https://codeparadox.io/tracks/compute-traffic-foundations/sections/software-system-architecture-section-1#server)

* what is a server and builds toward capacity planning, scaling, load balancing, and cost-aware trade-offs?

##### 01. What is a Server?: Before we scale anything - Clients, servers and physical vs. virtual machines from scratch.

* A server is a computer that provides something to other computers over a network.
* “Server” can mean the physical or virtual machine itself, or it can mean the software running on that machine that actually answers requests
* The computer asking for something is usually called the client. The client sends a request; the server receives it, does some work, and sends back a response.
  * A server is not inherently a giant, special computer. Your laptop can technically run server software. Production servers are simply machines configured to reliably serve real workloads, at scale, all day, every day.
* **Server:** A computer that accepts requests or jobs and provides a service to other programs. It may be a physical machine, virtual machine, container host, or cloud instance.
* **Instance**: One running copy of a server process or virtual machine. Two instances can run the same application while using separate CPU and memory.
* **Application server:** A server instance that runs business logic: validating requests, applying rules, calling databases, and producing responses.
* **Worker:** A compute process that handles background jobs outside the interactive request path. Workers commonly process images, reports, emails, and scheduled jobs.
* **Fleet:** A group of server instances performing the same role. A fleet can be uniform, with identical sizes, or heterogeneous, with different capacities.
* **Load balancer:** The network entry point that distributes incoming requests across healthy application-server instances.
* **CPU / vCPU:** The processing resource that executes instructions. CPU-bound work slows down when computation consumes all available processor time.
* **Memory / RAM:** Fast temporary storage used by running processes and in-flight work. Memory-bound systems fail when active work requires more RAM than a host owns.
* **Throughput:** The amount of work completed per unit of time, such as requests per second (RPS) or jobs per second.
* **Latency:** The time one request or job takes from start to finish. Throughput describes volume; latency describes waiting time experienced by one item. Think of throughput as “How much work can you finish?” and latency as “How long does one piece of work take?”. The server handles 100 requests per second (Throghput), and a typical request takes 50 milliseconds(Latency).
* **Availability:** The ability to continue serving requests when components fail. Multiple servers can improve availability only when traffic can avoid unhealthy instances.
* **Stateless service:** A service that does not keep request-specific state only in one server's local memory, allowing later requests to run on another instance.

##### 02. From Browser to server and Back - What happens when you open a website: The full request/response journey behind one page load

* The browser sends a request, that request travels across the network, a server receives it, application code executes, that code may query a database, and finally a response travels all the way back.
  * Every request that reaches your application costs real compute time: CPU cycles to run code, and RAM to hold the request's data while it's being processed. That cost is exactly what this whole section teaches you to measure and provision for.
* Later concepts: DNS lookups, TCP connection setup, TLS/HTTPS negotiation, CDNs, and reverse proxies all sit inside this journey too. They matter, but they're networking concerns, not compute-capacity concerns, so this section deliberately keeps them out of view. Later material can go deeper on that networking layer.

##### 03. Web Servers vs Application servers: What each role does, plus static vs dynamic requests - Two roles, not always two machines

* A **web server** primarily handles HTTP connections: it can serve static files directly, or act as a proxy that forwards requests onward. Nginx and Apache are common examples.
* An **application server** runs your actual business logic: authentication, validation, calling a database, building a response. A Node.js app, a Java/Spring service, or a Python/FastAPI app are all application servers.
  * They describe two different *responsibilities* that traffic passes through, not two mandatory pieces of hardware. Many production systems collapse them into one process; others deliberately split them. Both are valid, and this course's exercises focus on the request path's shape, not on forcing every role onto its own box.
* Modern systems handle this split in different ways. Sometimes the two roles run as genuinely separate machines. Sometimes an application framework serves HTTP directly, folding both roles into one process. Sometimes a load balancer or reverse proxy sits in front and takes on some of the web server's job. What matters for system design is which *role* a component performs, not whether that role happens to live on its own physical machine.
* Static vs Dynamic requests:
  * Not every request costs the same. A static request asks for a file that already exists, like an image; the server just hands it over. A dynamic request runs real application code: it authenticates you, applies business rules, and often talks to a database. That code execution is exactly what shows up as CPU and memory usage on your application servers.

##### 04. CPU, RAM and Server Resource: Utilization, memory and CPU/memory/IO - bound workloads. - The two resources that matter most

* **CPU** performs computation: executing application code, serializing JSON, encrypting or decrypting data, validating input, transforming data, running calculations. Every one of those actions takes measurable processor time.
* CPU saturation doesn't just slow down the current request: it increases queueing, so every subsequent request waits longer too, and overall latency climbs even before anything technically “fails.”
* **RAM (memory)** stores data actively needed by running processes: the application runtime itself, in-flight request data, in-memory caches, active background jobs, and buffers. Unlike CPU, memory isn't reclaimed the instant work finishes; it stays occupied for as long as that work is in progress.

  * CPU and RAM are different bottlenecks. A server can have plenty of spare CPU and still crash from running out of memory, and vice versa. Always check both, independently.
* **CPU Bound, memory-bound and I/O-bound:**

  * “The server is overloaded” doesn't always mean CPU is high. Every workload has a resource that actually limits it first.
  * **CPU-bound:** Computation itself is the limit. The server is busy calculating, encrypting, or transforming data.
  * **Memory-bound:** Working data cannot comfortably fit in RAM. Concurrency, not raw request rate, is usually the driver.
  * **I/O-bound:** The application spends most of its time waiting on the network, disk, or a database call to return.

##### 05. Throughput, latency and Concurrency: Measuring work - RPS, latency vs throughput, and why concurrency isn't rate

* **RPS** stands for  **requests per second** : simply how many requests complete in a one-second window.
* A simple `GET /health` endpoint and an expensive image-processing endpoint do not cost the same amount of compute per request. The same server might handle 800 RPS of the first and only 40 RPS of the second. Always attach a capacity number to the specific workload it was measured against.
* **Latency is not throughput:** Throughput asks “how much work can we complete?” Latency asks “how long does one request take?” A system can process a huge volume of requests per second while each individual request still takes a noticeable amount of time to finish; both are true at once.
* **Concurrency: how many, not how fast**

  * Throughput counts completions per second. **Concurrency** counts how many requests are active at the same moment, in flight, holding resources, regardless of the rate they arrive at.
  * This distinction matters more than it looks: a request that stays open for a long time (say, generating a report) ties up memory for its entire duration, even at a modest arrival rate. You'll use this idea directly later, when sizing memory with Little's Law.

##### 06. Server capacity and headroom: Capacity is not safe capacity - Tested maximum vs safe sustained operating capacity

* A server's tested maximum, say 800 RPS, is the ceiling measured under ideal conditions. Running there continuously leaves zero margin for anything unexpected. An operational target, commonly 80% of the tested maximum in this course's exercises, deliberately reserves the rest as headroom.
* Why headroom exisit:
  * Real traffic is never perfectly uniform. Headroom absorbs:
    * Traffic variance: request rates fluctuate second to second.
    * Uneven distribution: a load balancer rarely splits traffic perfectly.
    * Slow requests: a handful of expensive requests can dominate a server's time.
    * Background work: deploys, health checks, and log shipping all use some capacity too.
    * Unexpected spikes: nothing in production behaves exactly like a spec sheet.
* “80% utilization is a universal industry law.”?
  * It isn't. 80% is a common, reasonable operational target, and it's the specific constraint used in many of this course's simulations, but real systems use different ceilings depending on workload volatility, failure tolerance, and cost sensitivity. Treat the number in each problem's requirements as the actual contract, not a universal constant.

##### 07. Capacity planning: Turning Demand into a Server count - The core sizing formulas, plus an interactive calculator

* Capacity planning is arithmetic before it is architecture. Three formulas cover almost every sizing decision you'll make in this course.

  **FLEET CAPACITY:**

  * fleet capacity = server capacity × number of servers = 3 × 600 RPS = 1,800 RPS

  **SAFE SERVER CAPACITY:**

  * safe capacity = maximum capacity × utilization target = 600 × 80% = 480 safe sustained RPS

  **SERVERS REQUIRED:**

  * servers required = ceil(required traffic ÷ safe capacity per server) = 1,700 ÷ 480 = 3.54 → round up to 4 servers
* **Demand:** The work arriving at the system: requests per second, jobs per second, or concurrent sessions.
* **Capacity:** The maximum work a component can complete per unit of time under the model being used.
* **Utilization:** The fraction of available capacity currently demanded. At 80%, one-fifth remains as headroom.
* **Headroom:** Unused capacity intentionally reserved for variability, measurement error, slow requests, and short bursts.
* **Concurrency:** The number of requests or jobs in progress at the same time, not the number arriving each second.
* **Bottleneck:** The first constrained resource that prevents the system from meeting its requirement.

  * A server's maximum capacity is not automatically its safe sustained capacity. Every sizing calculation in this course starts by applying a utilization target, not the raw tested maximum.

##### 08. Vertical vs Horizontal scaling: Two ways to add capacity - Scaling up vs scaling out, and when each one wins

* Scaling up (vertical) replaces a server with a larger shape.

  * Move to a larger machine. Operationally simple, but every size has a hard ceiling and the largest shapes can have a poor price-to-capacity ratio.
* Scaling out (horizontal) adds more servers of the same or a similar shape. Each solves a different constraint.

  * Add machines and distribute work. Capacity grows incrementally, but every instance must actually be reachable, and the routing policy has to match the fleet.
* Prefer scaling up when:

  * The workload can't be partitioned safely.
  * Operational simplicity matters more than redundancy.
  * A larger shape fits both the capacity and budget limits.
  * Coordination cost grows with server count.
* Prefer scaling out when:

  * One machine can't meet aggregate demand.
  * Work can be distributed across stateless instances.
  * Capacity should grow in smaller increments.
  * No single compute failure should take everything down.
* More servers always means more usable capacity?

  * Only if traffic can actually reach them. A server sitting on the canvas with no connection from the load balancer contributes zero usable capacity, no matter how large it is.
* Adding a node is not the same as adding a reachable path. Every application server needs a real connection from the entry point before it counts toward fleet capacity.
