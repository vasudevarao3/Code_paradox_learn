# Queues & Background Processing

Starting from what a message queue physically is, build up to delivery guarantees, worker sizing, backpressure, retry storms, ordering, and dead-letter handling; and exactly what a queue promises versus what it just happens to usually do.

##### 01. What is a Message Queue?: Foundations - A message queue, from a plain-English kitchen analogy up

* Picture a busy restaurant kitchen. A waiter takes an order, then has two choices. They could walk to the kitchen and stand at the pass until the cook finishes the dish, but then they can't take any other table's order while they wait, and if the kitchen is backed up, so is every table they serve. Or they can clip the ticket to a rail by the kitchen door and immediately go back to the floor. The cook pulls tickets off that rail whenever they're free, in whatever order makes sense, and nothing about the order gets lost even if one cook steps away for a minute: the ticket just sits there until someone is available.
* That rail is a  **message queue** . The waiter is a  **producer** : something that creates work. The cook is a **consumer** (or  **worker** ): something that processes it. The ticket itself is a  **message** . And the queue's entire job is to let the producer stop thinking about the work the instant it's handed off, while guaranteeing the work still gets done.
* A queue's value isn't speed: pulling a ticket off a rail doesn't cook the dish any faster. Its value is  **decoupling** : the producer and the consumer no longer have to be available, healthy, or even running at the same moment for work to eventually get done.

##### 02. Queues and Decoupling: Mechanics - Why moving work off the request path changes what can fail

* In a web request, decoupling means the code that answers the client and the code that does the expensive work no longer have to be the same code, running at the same time. The request handler writes a message to the queue (a cheap, fast, local operation) and responds immediately. A separate pool of workers reads from that queue on its own schedule and does the slow part.
* **Producer:** The system that places a message into the queue.
* **Consumer / Worker:** The process that reads a message off the queue and does the actual work
* **Broker:** The queueing system itself: the software that stores, orders and hands out messages.
* **Backlogs / Queue depth:** The number of messages currently waiting to be processed.
* **Backpressure:** The system deliberately slowing or rejecting new work because downstream can't keep up.
* **Poison pill:** A message that always fails processing, and blocks real work behind it if not isolated.
* Put a number on the trade: writing a message to the queue typically costs a few milliseconds (1-5 ms for a local broker write), versus the 200 ms-2 s a synchronous email send or card-network round trip can take. A checkout that used to wait on all three steps inline can respond in well under 500 ms once only the critical part (the charge itself) runs inline and everything else is enqueued.

##### 03. Durable Messages and Acknowledgment: Reliability - A queue is a safety mechanism, not just a buffer.

* A queue that only holds messages in memory is really just a to-do list that vanishes the moment the process restarts: that's not meaningfully safer than doing the work inline. A**durable** queue writes each message to disk (and usually replicates it) before it's considered accepted, so a broker crash doesn't erase work that was already promised.
* The other half of durability is  **acknowledgment** . A worker doesn't just quietly finish a message: it has to explicitly tell the broker “I'm done, you can delete this.” Until that ack arrives, the broker keeps the message around, because the worker might have crashed mid-job.
* Put a number on it: a broker replicated across 3 nodes that requires 2 of 3 to confirm a write before acknowledging it can survive losing any single node without losing a message. A broker that only writes to memory on 1 node survives exactly zero node failures: a crash or restart during a traffic spike can erase every message that hadn't been flushed yet, which could easily be thousands of jobs at even a modest 500 messages/sec.

##### 04. Delivery Guarantees: Semantics - At-most-once, at-least-once, and exactly-once, compared side by side.

* Every queue has to answer one question precisely: If something goes wrong mid-delivery, does the message get lost, delivered twice, or delivery twice or delivery exactly once? These are three genuinely different contracts, and mixing them up is once of the most, and most expensive, mistakes in queue design.
* "A queue guarantees exactly-once processing by default.”
  * Almost no queue does, out of the box. Most popular systems (SQS, RabbitMQ, Kafka's default consumer mode) are **at-least-once** by default: they will redeliver a message rather than risk losing it, which means your worker code will occasionally see the same message twice. Exactly-once isn't a broker setting you flip on; it's a design you build, usually by making your handler idempotent (chapter 07) or by tracking processed-message IDs yourself.
* Why does at-least-once end up being the practical default? Because the broker can't actually see inside a worker's process to know if it's “still working” or “silently died.” It has to guess, using a timer called a  **visibility timeout** .
* Visibility timeout, as a rule of thumb = visibility timeout ≥ p99 job processing time
  * a job with a 12s p99 processing time should get a visibility timeout of at least 15s, not a default 30s accepted blindly, and not 5s, which would redeliver healthy jobs still mid-flight

##### 05. Worker Throughput and Sizing: Capacity - Why a backlog can grow forever from a gap that closes seconds later.

* Workers exist to consume the backlog faster than it arrives. That sentence hides a sharper point: “faster than” is the whole game. If processing rate is even *momentarily* below arrival rate (not permanently, just for a few minutes during a traffic spike or a slow downstream call), the backlog doesn't stay flat. It grows every second the gap is open, and it only starts shrinking again once processing rate exceeds arrival rate, not merely matches it.
* This is the same intuition behind queueing theory's classic result: as utilization (arrival rate ÷ processing rate) approaches 100%, average wait time doesn't creep up gently: it climbs sharply, because there's no slack left to absorb any variance in arrivals. A system running at 95% utilization isn't “almost at capacity” in a comfortable sense; it's one small burst away from a backlog that takes a long time to work back down.
* Backlog growth
  * arrival rate − processing rate = backlog change per unit time
  * 1,200 arrivals/min − 1,000 jobs/min capacity = +200/min backlog growth (Figure 7's overload case)
* Required workers
  * ⌈arrival rate ÷ throughput per worker⌉, with headroom for spikes
  * 1,200 jobs/min ÷ 50 jobs/min per worker = 24 workers at the raw ratio; most teams size above that, say 30, for headroom
* Sizing a worker pool for the *average* arrival rate guarantees the backlog will grow during every burst above average, and because growth compounds, it can take far longer than the burst itself to recover. Size for a realistic peak, not the average.

##### 06. Backpressure and Latency: Reliability - Queue depth is a symptom, not a fix by itself.

* Backpressure is the system's way of saying “we are overloaded”, and doing something about it, rather than just absorbing more work. That might mean slowing producers down, rejecting low-priority jobs outright, or routing overflow into a separate, lower-priority queue. A queue with no backpressure doesn't fail loudly; it just becomes a latency amplifier that quietly stores everything while telling no one that anything is wrong.
* Put concrete thresholds on “healthy” versus “unhealthy”: a backlog age under 5 seconds is usually invisible to users; one that climbs past 60 seconds and keeps rising, even if queue depth itself looks modest, means the system is actively falling behind, not just momentarily busy. In an interview, a growing queue depth is usually a signal, not the problem itself: it's pointing at an under-provisioned worker fleet, a downstream dependency that got slow, or a piece of work that shouldn't have been queued for real-time handling in the first place.

##### 07. Retry storms, Poison Pills and Idempotency: Reliability - When retrying a failed job become the next outage

Retrying failed work sounds like a purely defensive move, but done carelessly it can become the outage.

| Failure Mode               | Common symptom                                  | Typical fix                                 |
| -------------------------- | ----------------------------------------------- | ------------------------------------------- |
| Transient dependency error | Slow retries, queue spike                       | Retry with exponential backoff and jitter   |
| Poison message             | Same failing job repeats forever                | Cap retries, move to DLQ for manual triage  |
| Worker crash               | Backlog grows while uptime stays okay           | Autoscale workers, keep handlers idempotent |
| Retry storm                | One dependency failure multiplies queue traffic | Circuit breakers, retry caps, rate limits   |

* Put a number on how a storm actually forms: a dependency hiccups and 1,000 in-flight jobs fail within the same second. With no backoff, all 1,000 retry almost immediately, more than doubling the load on a dependency that was already struggling to keep up with its normal traffic. With jittered exponential backoff (roughly 1s, 2s, 4s, 8s, each with random jitter so retries don't land in lockstep), those same 1,000 retries spread out over tens of seconds instead of arriving in one spike, giving the dependency room to actually recover.
* A **poison pill** is the individual-message version of the same problem: a message whose payload always causes processing to fail (malformed data, an edge case the handler never accounts for), so it fails, gets retried, fails again, forever. Left alone, it can block every well-formed message queued up behind it.
* “Retrying a failed job is always safe.”
  * It's only safe if the job is  **idempotent** : running it twice produces the same result as running it once. Sending a confirmation email twice is mildly annoying. Charging a credit card twice for a single order is a real financial bug. Because most queues are at-least-once by default (chapter 04), any handler that isn't explicitly written to be idempotent (usually via an idempotency key that lets it recognize “I already did this exact operation”) is quietly unsafe to retry.

##### 08. Ordering and Partitioning: Design - Not every queue preserves the order work arrived in.

* Queue ordering is a product decision as much as a technical one. Some work genuinely must execute in sequence: you can't apply a “refund” event before the “charge” event it refers to. Other work is completely independent and only *looks* like it needs ordering because it's sitting in the same queue.
* A single, strictly ordered queue caps throughput at whatever one consumer can process: you can't safely parallelize it without risking out-of-order delivery. Partitioning solves this by routing messages that share a key (like a user ID or order ID) to the same partition, so order is preserved *within* that key, while unrelated keys process fully in parallel across many consumers.
* Put a number on the throughput difference: a single ordered queue caps out at whatever one consumer can process, say 200 jobs/sec. Partition the same traffic by **order_id** across 16 partitions and 16 consumers, and total throughput rises to roughly 3,200 jobs/sec, provided no single key gets hot enough to dominate its own partition (chapter 07's poison-pill problem, one level up).
* What if a message needs to be ordered relative to two different keys at once?
  * Partitioning only gives you one ordering guarantee: within a single key. If a message genuinely depends on ordering across two different keys (say, both a specific order and a specific warehouse), no partition scheme assigns it to both lanes at once. The usual fix is to pick the one relationship that actually matters for correctness and accept weaker guarantees on the other, or to serialize that specific cross-key case through a separate mechanism (a lock, a saga step) rather than trying to force the queue itself to solve it.

##### 09. Dead-Letter Queues and Replays: Operations - Quarantining bad work so it can't block good work.

* A **dead-letter queue** (DLQ) is a quarantine area for messages that failed too many times or can't be parsed at all. Once a message exceeds its retry limit, the broker moves it out of the main queue and into the DLQ instead of retrying forever, which is what stops a single poison pill from blocking everything behind it.
* Critically, the DLQ isn't a trash can. The message is preserved so an operator can inspect it, understand why it failed, fix the underlying bug, and then deliberately **replay** it back into the main queue for reprocessing.
* A concrete policy puts a number on “too many times”: 5 retries with backoff of 1s, 5s, 30s, 2min, then 10min, after which the job dead-letters instead of retrying a 6th time. That caps the worst case at roughly 18 minutes of retrying a single job before it stops consuming a worker slot, a deliberate, bounded number instead of an open-ended “keep trying.”
* Doesn't a DLQ risk hiding failures nobody ever looks at?
  * Yes: a DLQ with no alerting is worse than no DLQ at all, because failures go silent instead of loudly retrying in front of an on-call engineer. A real DLQ setup always pairs the quarantine with an alert on non-zero DLQ depth, so a human actually goes and looks.

##### 10. Queue Design Workflow: Method - A repeatable method for any queue-design interview question.

* A repeatable way to work through any queue-design question, whether it's an interview or a real incident: start with what business criticality actually demands, then work outward to durability, delivery guarantees, capacity, and failure isolation.
  1. Separate request-path work from background work
  2. Decide what can fail fast, what must retry, and what must never be lost
  3. Choose a delivery guarantee: at-least-once plus idempotency, or a harder exactly-once design
  4. Set the visibility timeout around realistic p99 job duration
  5. Define retry policy: backoff, jitter, and a maximum retry count
  6. Add DLQ handling and alerting for poison-message isolation
  7. Size the worker fleet for peak arrival rate, not the average
  8. Implement backpressure before backlog age becomes a user-facing problem
  9. Decide whether ordering is required, and partition by key if it is
  10. Monitor backlog depth, backlog age, and retry volume continuously

##### 11. Practice Labs: Apply It - Four hands-on problems that build on every chapter above.
