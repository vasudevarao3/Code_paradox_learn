##### 09. Load balancing & Traffic Distribution: Turning many servers into one fleet - Routing algorithms, with an interactive animated demo.

* A load balancer is one ingress point that distributes incoming traffic across a set of backend servers.
* It converts several individual machines into one usable fleet, routes only to healthy backends, and is what makes horizontal scaling actually work.
* Methods

  * **Round Robin:** Sends the same number of requests to each server, in turn. A strong default for a uniform fleet where every server has the same capacity.
  * **Weighted Round Robin:** Sends work in configured proportions. Use it when servers have different capacities; weights should represent whichever resource actually limits the workload.
  * **Least Connections:** Prefers the server with fewer active connections. Useful when request duration varies significantly and connection count approximates current load.
* **Uniform vs. mixed fleets:**

  * Equal routing is only safe when equal work produces roughly equal utilization. In a mixed fleet, one request may consume the same CPU or memory regardless of host size, so a smaller server needs a proportionally smaller traffic share. Always evaluate the hottest server after routing, never the average server.
* Why not just always use Least Connections?

  * It looks like the safest universal default, but connection count is only a good proxy for load when every connection costs about the same amount of work. A fleet mixing a handful of long-lived streaming connections with thousands of short API calls will route badly under Least Connections, since one “connection” on the streaming server is not comparable to one on the API server. Round Robin or Weighted Round Robin is often the more predictable choice once request cost varies by endpoint, not just by server size.

##### 10. Local bottlenecks & fleet averages: What averages hide - Why a healthy fleet average can hide one over

* A fleet average is computed across every server. A local bottleneck lives on exactly one of them. The two numbers can tell completely different stories.
* If total fleet CPU is below 80%, every server is healthy?

  * Averages can conceal a badly overloaded individual machine sitting behind two idle ones. Always inspect the busiest server specifically, not the mean across the fleet.
* Fleet totals can hide overloaded individual machines. Teach yourself to check the hottest server or resource first, and only then look at the fleet-wide picture.

##### 11. Little's Law & Memory Sizing: Concurrency Determines Memory Demand - Little's law and Why memory is local to each server.

* Throughput measures how quickly work arrives or completes. Memory is often held for a request or job's entire lifetime.
* A modest arrival rate can therefore still create large memory demand, if each item stays active for a long time.
  * Little's Law:

    * concurrency = arrival rate × average duration = 6 jobs/s × 30 s = 180 concurrent jobs
  * Memory Demand:

    * memory = concurrency× memory per job =180 × 900 MB ÷ 1,024 = 158.2 GB
* **RAM is local, not a shared bucket:**
  * Three servers may collectively own enough memory while one of them still crashes. Each job runs on exactly one host, and that host alone must hold the job's working set. Load distribution, not fleet totals, determines whether memory placement actually succeeds.

##### 12. Workload isolation: Not all work is the same - Seperate interactive traffic from background batch work

* Interactive APIs optimize for latency and immediate availability. Batch systems optimize for throughput and completion deadlines. Running both on one finite pool lets a scheduled batch run consume capacity that customers need right now.
  * Required workload throughpu:

    * backlog ÷ deadline seconds = 240,000 ÷ (15 × 60) = 266.7 jobs/s
  * Backlog Drain Time:

    * backlog ÷ worker throughput ÷ 60 = 240,000 ÷ 300 ÷ 60 = 13.3 minutes
* The worker pool must sit outside the API traffic path, with enough of its own throughput to meet its own deadline. The API fleet must independently satisfy its own request target. If either pool relies on borrowed capacity from the other, the two workloads are not actually isolated, they're just drawn as if they were.

##### 13. Baseline, peaks & Burst traffic: Sizing Across time, not just load - Sizing one fixed fleet across both quiet and busy hours

* When autoscaling isn't available, the same fleet stays online all day. A good design covers the highest specified demand while avoiding excessive idle capacity during ordinary traffic.
  * Peak condition:

    * fleet capacity ≥ full peak demand ÷ utilization target = 1,700 ÷ 80% = 2,125 RPS of fleet capacity, minimum
  * Baseline Efficiency:

    * baseline demand ÷ fleet capacity × 100 = utilization at rest = 900 ÷ 2,125 ≈ 42.4% most of the day
* Applying the same 80% headroom target from the capacity-planning chapter, a fleet that must absorb the 1,700 RPS peak without exceeding 80% utilization needs at least 2,125 RPS of raw capacity. That same fleet only runs at about 42.4% during the other 21 hours of the day, when demand sits at the 900 RPS baseline: comfortable, but far from fully used. Ask three questions before sizing a fixed fleet: must the full peak be handled, and for how long? Can temporary utilization run hotter than the sustained target? And what does the extra idle capacity cost during every hour that isn't the peak? A baseline utilization floor exists precisely to reject a technically valid but wasteful design, one that buys a huge fleet for a short daily spike and then sits mostly idle.
* Why not just build for the peak and accept the idle cost?
  * You usually can, and sometimes should: it's a valid design, not a mistake. The trade-off is purely economic. A fleet sized at 2,125 RPS to cover the peak is carrying about 136% more raw capacity than the 900 RPS baseline actually uses, 21 hours out of 24. Whether that's acceptable depends on whether autoscaling is available (so the fleet can shrink outside the peak window) and how expensive the chosen server shape is per hour; it's a cost decision made explicit, not a universal rule against building for the peak.

##### 14. Cost vs Capacity: Choosing among valid designs - Choosing the cheapest design that still meets every constraint.

* Cost should never be the first filter. A valid architecture has to satisfy its technical constraints, capacity, utilization, redundancy, first. Only once several designs are all valid does cost help you choose between them.
  * Capacity planning is constraint solving, not simply buying bigger servers. The cheapest option that satisfies every requirement is a good choice; the cheapest option overall, ignoring requirements, is not.

##### 15. The Compute-sizing workflow: Putting it all together - A repeatable 12-step process for sizing any computer tier

* Before touching a design canvas, find the actual bottleneck. Scaling a resource that isn't constrained does nothing for end-to-end throughput.
* **A repeatable process:**
  * Do the arithmetic before placing components. This keeps the design canvas from becoming a guessing exercise, and gives you a short, defensible explanation for every decision.
    1. Understand Demand: Record baseline rate, peak rate, and the time unit for every value.
    2. Identify the workload: Decide if it's request-driven, job-driven, CPU-bound, or memory-bound.
    3. Find per-server capacity: Look up (or measure) how much one instance of a given size can do
    4. Apply Headroom: Multiply rated capacity by the target utilization ceiling before sizing anything.
    5. Calculated server count: Divide required demand by safe per-server capacity, then round up.
    6. Check local CPU / RAM: Verify no single server, not just the fleet average, exceeds its limit.
    7. Check traffic distribution: Confirm the routing policy actually matches the fleet, uniform or mixed.
    8. Check peak behaviour: Re-run the same checks against the peak, not only the baseline.
    9. Check failure & redundancy: Confirm the design still holds if one instance is unavailable.
    10. Check cost: Compare viable designs by monthly cost only after they all satisfy the constraints above.
    11. Simulate / load test: Validate the design against realistic traffic before trusting the arithmetic.
    12. Iterate: Revisit any step once traffic, workload shape, or budget changes.

16. Practice labs: Apply what you just learned - Four hands-on design problems that build on every chapter above.
