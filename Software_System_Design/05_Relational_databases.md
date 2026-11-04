# Relational Databases

Starting from what a database physically is, build up to modeling relational data, designing indexes, bounding database connections, isolating transactional work from analytics, and enforcing integrity in the database itself.

##### What is a Database At all?: Foundations - What a database actually is, before why relational: from a plan-English analogy up.

* A **database** is a software program whose entire job is storing data durably, letting many callers read and write it safely at the same time, and answering structured questions about it quickly. You don't open it like a file: you send it a query, and it decides how to serve every caller correctly, even when hundreds of them show up in the same second.
* A **relational** database adds structure on top of that. Instead of storing everything as one large blob per record, it separates data into tables with clear types, keys, and relationships. That matters because real systems care about entities, their links, and integrity rules: a customer has many orders, an order has many items, a payment belongs to one order. Those links aren't just business labels: they are the basis for joins, consistency checks, and safe concurrent updates.
* **Database:**  Software whose entire job is storing data durably and letting many callers read and write it safely at the same time.
* **Relational Database:** A database that stores structured rows in tables and connects tables through keys.
* **Row:** One record in a table, such as one order or one customer.
* **Column:** A named attribute with a data type and optional constraints.
* **Primary Key:** A value that uniquely identifies one row.
* **Foreign Key:** A column constrained to reference a row in another table.
* **Normalization:** Storing each fact in exactly one place, so updates can't leave copies disagreeing.
* **Index:** A separate search structure that speeds selected access patterns.
* **B-Tree:** The sorted, balanced tree structure most database indexes use, built to be searched in a handful of steps.
* **Query Plan:** The database's chosen operations for executing a query, such as a scan, seek, join, or sort.
* **Transaction:** A unit of work that commits completely or rolls back completely.
* **Connection:** A database session that consumes memory and server resources.
* **Read replica:** A database copy that applies changes from a primary and can serve selected reads.

##### 02. Rows, Tables and Columns: Mechanics - The physical shape of a table, with real sample row so row and column stop being jargon.

* A **row** is one complete record: one customer, one order, one payment.
* A **column** is one named, typed attribute shared by every row in the table: every customer row has the same four columns, just different values in them.

  * A row is the atomic unit the database can lock, update, or delete independently of every other row. That's the real, structural difference between “one row per order” and one giant nested JSON document holding many orders together: the database can touch one order without ever touching, or blocking, any of the others.

##### 03. Relational Modeling: Entities and Relationships - Entities, keys and relationships and how a foreign key encodes them.

* Start from entities, relationships, and access patterns. A customer has many orders, an order has many line items, and an order may have one payment. Separate tables avoid repeating parent data and let constraints protect each relationship individually.
* The **customers.id** column is the table's  **primary key** : it uniquely identifies one row and nothing else can share its value. **orders.customer_id** is a  **foreign key** : a column that must match some existing primary key value in **customers**, or the database refuses the write.
* What's a junction table?

  * Some relationships are many-to-many, not one-to-many: a product can appear in many orders, and an order can contain many products. Neither table can hold a single foreign key to the other, so a third “junction” table sits between them: **order_line_items** with its own **order_id** and **product_id** foreign keys, one row per product-in-an-order pairing.

##### 04. Normalization and Denormalization: Design - Storing each fact once, and when deliberately breaking that rule is the right call.

* **Normalization** is the discipline of storing each fact in exactly one place. If a customer's email lives only in the **customers** table, updating it is a single-row write. If it's copied onto every order that customer has ever placed, an update has to find and fix every copy: miss one, and the data now silently disagrees with itself.
  * "A fully normalized schema is always the faster one.”?
    * Normalization is about eliminating duplicated facts and the update anomalies they cause; it isn't primarily a speed optimization. A normalized schema often needs more joins to answer a common read, and each join costs something. Denormalizing one specific, measured hot path is a legitimate trade; the mistake is denormalizing everywhere by default instead of only where an access pattern actually justifies it.
* **Denormalization** means deliberately reintroducing some of that duplication, usually to avoid a join on a hot read path. It can be the right call, but it should be justified by a measured access pattern, not chosen by default because it's convenient to write.

##### 05. Transactions and ACID: Correctness - The Unit of "do the whole business action, or do nothing at all"

* ACID stands for Atomicity, Consistency, Isolation, and Durability. A transaction groups multiple statements into one commit decision. An order and a payment can live in separate tables and still succeed or fail together.
* In plain English: a transaction is the unit of “do this whole business action, or do nothing at all.” If a customer places an order and the payment fails, the system should not leave the order looking as if it succeeded. The database gives the application a consistent view even while many requests happen at the same time.
* **Atomicity:** All statements inside a transaction commit together, or every one of them rolls back. There is no partial result where the order exists but the payment doesn't.
* **Consistency:** The database's own rules stay true before and after every commit: foreign keys match real rows, NOT NULL columns have values, and every UNIQUE constraint still holds.
* **Isolation:** A transaction never observes another transaction's half-finished state, even when thousands run at the same time. Each one sees a consistent snapshot, not a partial write in progress.
* **Durability:** Once a transaction commits, the change survives a crash or reboot. The database only reports success after the write is on durable storage, not just in memory.
* Durability isn't a promise about speed, it's a promise about survival: a Postgres commit doesn't return to the caller until the write-ahead log entry for that transaction has been flushed (fsynced) to disk, which is why a crash one millisecond after a commit returns still can't lose that write, while a crash one millisecond before commit loses nothing the caller was ever told succeeded.

##### 06. Indexes and Query Plans: Performance - What a B-tree actually is, and why a sorted structure makes a seek fast

* An **index** is a separate, ordered structure the database maintains alongside the table. It can replace a full scan with a targeted seek, but every write to the table must also update every index built on it.
* “Adding an index only helps; it can't make anything worse.”?
  * Every index the database maintains has to be updated on every INSERT, UPDATE, and DELETE that touches its column, so more indexes means more write work and more storage, even on rows a given read-only query never touches. An index is a trade: faster reads for one specific access pattern, paid for by slower writes and more disk space, all the time, not just when that pattern runs.
* Most relational indexes are a  **B-tree** : a balanced, sorted structure, conceptually a lot like a dictionary. To find a word in a dictionary, you don't read every page: you open roughly to the right letter, then narrow in from there. A B-tree does the same thing with row keys: it keeps them sorted in a shallow tree, and each level of the tree lets the database discard a large fraction of the remaining rows with a single comparison, rather than reading them one by one.
  * A sorted structure turns “search everything” into “discard half the remaining rows at every step.” That's exactly what makes B-tree lookups scale with the tree's depth (logarithmically) instead of with the table's row count (linearly).
  * Query Target: p95 latency <= 100 ms
  * Write cost: Physical writes per INSERT = 1 row write + 1 write per index on that table
    * orders has 3 indexes (customer_id, created_at, status): one INSERT costs 4 physical writes, not 1
  * Storage cost: row * bytes per index entry
    * 10,000,000 orders × 16 bytes per B-tree entry ≈ 160 MB, per index, before the table's own storage

##### 07. Database Connection Pooling: Capacity - Bounding real database concurrency below the connection limit

* Each PostgreSQL connection consumes memory and backend resources. An application fleet can open more connections than the database can safely accept, even when the underlying query traffic itself is perfectly reasonable.
* Direct Connections:

  * server instances × connections per instance
  * 20 app instances opening 15 connections each = 300 direct connections, 3x Postgres's 100 limit
* Pooled Connections:

  * pool size ≤ database limit
  * one pooler (e.g. PgBouncer) holding 90 real connections multiplexes all 20 instances' traffic, matching Figure 8
* Queue depth at saturation

  * concurrent queries − pool size = requests waiting
  * 150 concurrent queries against a 90-connection pool leaves 60 requests queued
* A pool trades unbounded connection growth for bounded queueing. Size it below the database limit, but high enough that peak concurrent queries don't wait beyond the latency target. The 300 direct connections above aren't a hard failure by themselves, Postgres just refuses the 101st simultaneous one, which is exactly the failure a pool is designed to prevent by making every instance share from one bounded set.

##### 08. OLTP vs OLAP: Two Different workload shapes - Why analytics and checkout compete for the same capacity budget

* Online transaction processing, or  **OLTP** , favors short, indexed reads and writes with low latency: a checkout, a login, adding one item to a cart.
* Online analytical processing, or  **OLAP** , favors large scans and aggregations: a monthly revenue report scanning millions of rows.
* Sharing one capacity budget between them lets analytical work disrupt customer-facing transactions: a single report query can hold locks, saturate disk I/O, or evict hot data from cache, and checkout latency spikes for reasons that have nothing to do with checkout traffic itself.
* Put numbers on it: a checkout write that normally commits in 8 ms can jump past 500 ms if it's stuck behind a report that's mid-scan and holding a lock the write needs. That's not a checkout bug, a slower disk, or bad luck: it's two unrelated workloads sharing one database's fixed I/O and lock budget, and it's exactly what read replicas (next chapter) exist to separate.

##### 09. Read Replicas and Replication Lag: How a replica stays in sync, and how far behind "in sync" really means

* A **read replica** gives analytics its own capacity domain. Writes still go only to the primary. The primary records every change in a write-ahead log, and the replica continuously streams and replays that log to keep its own copy of the data current.
* That streaming isn't instantaneous: there is always some gap between a write landing on the primary and that same write becoming visible on the replica. That gap is **replication lag** , and it has to be acceptable for whatever the replica is being read for.
* Synchronous vs Asynchronous replication:
  * Asynchronous replication (the default for most read replicas) commits a write on the primary without waiting for the replica to confirm it, so replicas can lag by anywhere from milliseconds to seconds under load. Synchronous replication waits for the replica to confirm before the primary's commit returns: zero lag, but every write now pays the latency cost of a network round trip to the replica, and a slow or unreachable replica can stall writes entirely.

##### 10. Keys and Integrity Constraints: Integrity - Making invalid states impossible for every writer, not just your own code.

* Application-level validation improves error messages, but only database constraints protect data across every writer: every service, retry, one-off script, and future deployment that ever touches this table.
* **Primary Key:** Uniquely identifies each row.
* **Foreign Key:** Requires the referenced parent row to exist.
* **Unique:** Prevents duplicate values, such as two payments for one order.
* **Not Null:** Requires a value for a mandatory attribute.
* **Check:** Enforces a predicate, such as quantity greater than zero.
* **Transaction:** Enforces atomicity across related row changes.

##### 11. Database Design Workflow: Method - A relatable method for any relational-database design question

* Begin with correctness and access patterns, then size performance mechanisms.

  1. List entities and business rules
  2. Write the read and write access patterns
  3. Choose primary and foreign keys
  4. Decide how far to normalize, and where denormalization is justified
  5. Define transaction boundaries
  6. Add required integrity constraints
  7. Design the minimum useful indexes
  8. Budget connections and pool waits
  9. Separate incompatible workloads
  10. Plan replica read paths and an acceptable replication lag
  11. Check cost and failure behavior.
* A checklist stays abstract until it's run against one real schema.
* Here's every step above applied to a single, concrete example: an e-commerce order pipeline built from exactly three tables.
* Applied to one schema: orders · order_line_items · inventory

  * Entities & business rules - orders, order_line_items, inventory. Rule: an order can't be confirmed for more line-item quantity than inventory.quantity_on_hand allows.
  * Access Patterns - Hot read: fetch one order with its line items (checkout confirmation page). Hot write: confirm an order (insert order + line items, decrement inventory). Cold scan: nightly restock report across all inventory.
  * Keys - orders.id PK · order_line_items.id PK with order_id and product_id FKs · inventory.product_id PK.
  * Normalize / Denormalize - Normalize customer name and email into customers, referenced by orders.customer_id. Deliberately denormalize: snapshot unit_price onto order_line_items at order time, so a later price change never rewrites historical orders.
  * Transaction boundaries - “Confirm order” is one transaction: insert the order row, insert its line items, and decrement inventory.quantity_on_hand, all commit together or all roll back.
  * Integrity constraints - CHECK (quantity > 0) on order_line_items · FOREIGN KEY order_line_items.order_id → orders.id · CHECK (quantity_on_hand >= 0) on inventory, so a confirm that would oversell is rejected by the database, not just by application code.
  * Indexes - Index order_line_items.order_id (join target for “get order with its items”) and orders.customer_id (join target for a customer's order history).
  * Connection Budget - 20 checkout-service instances × 15 connections = 300 direct connections against Postgres's 100 limit; route through a 90-connection pool instead, per chapter 07.
  * Workload seperation - Run the nightly restock report (a full scan of inventory and order_line_items) against a read replica, not the primary serving live checkouts.
  * Replica lag budget - The restock report can tolerate up to 2 minutes of replication lag; the checkout confirmation path never reads from the replica at all.
  * Cost & failure check - If the pool is exhausted during a flash sale, confirm-order requests queue (chapter 07's queue math) rather than the database rejecting connections outright; if inventory is already at 0 when the transaction runs, the CHECK constraint rejects it instead of silently overselling.

##### 12. Practice Labs: Apply it - Four hands on problems that build on every chaper above.
