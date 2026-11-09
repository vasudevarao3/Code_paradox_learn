# Object Storage & File Systems

Starting from what a bucket and a key physically are, build up to presigned uploads, multipart transfers, filesystem IOPS ceilings, durability, and CDN delivery, and exactly which payloads belong outside the relational database.

##### 01. What is Object Storage? Foundations - A warehouse with flat addressing vs. a filing cabinet, from a plain-english analogy up

* Picture two ways of storing things at scale. The first is a filing cabinet: to find one document, you open a specific drawer, then a specific folder inside it, then flip through pages until you find the one you want. Every lookup means walking a hierarchy, one level at a time. The second is a giant warehouse with a flat addressing system: you don't walk any aisles yourself; you hand the clerk one exact label, and the item comes back directly, whether it's the first thing ever stored or the ten-billionth.
* That second model is what **object storage** actually is. It's a service that stores binary data (images, videos, backups, PDFs, anything) as  **objects** , each one addressed by a single flat string called a  **key** , inside a named container called a  **bucket** . There's no directory to open and no tree to traverse, which is exactly why it scales to trillions of objects spread across thousands of machines in a way a mounted filesystem never could.
* “Object storage is just a slower, cheaper database.”?
  * It isn't a database at all. There's no query language, no joins, no transactions across objects, and no way to ask “find me every object where X.” It's a durable key-to-blob lookup service reached over HTTP. Comparing its speed to a database's misses the point: it's optimized for durably holding huge binary payloads and serving them at HTTP scale, not for transactional access to small structured rows.
* The shift from “a path you walk” to “an address you name directly” is the single idea underneath everything else in this guide; nearly every chapter that follows is really just a consequence of that one design choice.
* **Object storage:** A service that stores binary data as objects in buckets, each addressed by a key rather than a local filesystem path.
* **Bucket:** A named, top-level container objects live in: think of it as the warehouse itself, not a folder inside it.
* **Blob:** A binary large object, such as an image, audio file, or video
* **Block Storage:** Low-level storage where data is addressed in fixed blocks; useful for virtual machines and databases.
* **File System:** Hierarchical storage mounted on a machine; useful for local, POSIX-like operations.
* **Key layout:** The naming strategy for object paths, often including tenant, ID, version, and content type.
* **Object metadata**: Data about the object, such as size, checksum, content type, and retention policy.
* **Presigned URL:** A normal storage URL with a cryptographic signature in its query string, granting temporary, scoped permission without the app server touching the bytes.
* **Multipart upload:** Splitting a large object into independently-uploaded parts the store stitches together on completion, so a failed part only needs that part retried.
* **IOPS ceiling:** The maximum number of storage operations a node or mount can deliver per second.
* **TTL:** Time to live, a policy that controls how long cached or temporary content remains valid.
* **Versioning:** Keeping multiple versions of the same object to allow safe restore or rollback.

##### 2. Object vs Block vs File Storage: Access Patterns - Three storage models, and which access pattern each one actually fits.

* Relational databases are excellent for rows, keys, transactions, and query semantics. They are not the best place for your largest binary data. Images, videos, documents, and backups are typically stored in an object store and referenced from the database by a key or URL; the database keeps the business record, the object store keeps the file itself.
* Below the object store sits a choice that's easy to conflate: block storage is best for a VM or database volume where a system reads and writes fixed-size blocks at an offset. File storage is great for POSIX-like directories and shared mounts. Object storage is best for large, independently addressed objects that are primarily accessed by URL or key: three different tools for three different access shapes.

##### 03. Buckets, Keys and Object Mechanics: Mechanics - What a real GET and PUT against an object key look like on the wire.

* Concretely: a bucket might be named **codeparadox-user-uploads**, and inside it, one user's avatar might live at the key **tenant-42/user-9001/avatar/original.jpg**. A GET and a PUT against that address are literally just HTTP requests: no special protocol, no client library required to understand the mechanics.

##### 4. Key Layout and Metadata: Design - Why the key string is part of the design, and what metadata rides along with it.

* Object storage makes the key string itself part of the system design. If the key structure is poor, tenants can collide, cache correctness becomes hard, and storage churn gets expensive. A consistent layout such as tenant_id / user_id / object_type / id / version is far easier to debug, audit, and manage at scale than an ad hoc one.
  * tenant=42 / user=9001 / avatar / 2026-09-13 / original.jpg
* Store the object key, content type, a checksum or version, and any retention policy as metadata on the object itself. Keep the database small and searchable while the object store carries the huge payloads and everything needed to describe them.
* **The hidden cost of a tidy, sequential key:**
  * A layout that sorts neatly, like a timestamp prefix (**2026-01-01/…**,**2026-01-02/…**) or a monotonically increasing ID, looks clean in a console listing, but it can be a real performance problem. Many object stores partition storage internally by the lexicographic (alphabetical) order of the key. A stream of sequential, timestamp-prefixed keys all sorts into the same narrow range, which routes almost every write to the same small set of partitions at once instead of spreading load across the store.
* “A clean, sorted key layout is always the better design.”?
  * Sorted and human-readable is a console-browsing convenience, not a storage-performance goal. A workload writing thousands of objects per second with sequential, timestamp-prefixed keys can saturate a single hot partition while the rest of the store sits idle. The fix is to randomize or hash a short prefix onto the key, for example a 4-character hash of the object ID placed before the timestamp, so otherwise-sequential writes land on different partitions instead of the same one.

##### 05. Direct Uploads and Presigned URLs: Upload Patterns - What a presigned URL actually is, and why it keeps bytes off the app server.

* The naive way to accept an upload is to have the client send the file straight to the app server, which then relays it to the object store. That works, but it means the app server holds the entire payload in memory or on local disk for however long the transfer takes: a 200MB video ties up a worker and its memory the whole time, even though the app server isn't doing anything useful with those bytes.
* A **presigned URL** fixes this. It's an ordinary object-store URL with a cryptographic signature embedded in its query string, generated by the app server using credentials it already holds. That signature grants whoever has the URL temporary, narrowly scoped permission (“PUT to this exact key, expiring in 15 minutes”) without the app server ever touching the file's bytes. The client uploads directly to the object store; the app server's only job becomes deciding who is allowed to write which key, and for how long.

  * Direct upload: client → app server → object store → success callback → metadata write. The app server sits in the middle of every byte.
  * Presigned URL upload: app grants short-lived permission → client uploads directly to the object store → client notifies the app to write metadata. The app server never sees a single byte of the file, only a small JSON request and response.
* The difference shows up fastest in memory. A 200MB upload held in an app-server worker for the roughly 90 seconds a slow mobile connection might take ties up that worker and its memory the entire time. At 50 concurrent uploads, that's 50 workers and close to 10GB of RAM spent on bytes the application never actually needed to touch, capacity that direct, app-mediated uploads simply can't give back to anything else until every transfer finishes.

##### 06. Multipart Uploads for Large Files: Large Files - Splitting a huge upload into parts so one failed part doesn't cost the whole transfer.

* A presigned URL solves who's allowed to upload. It doesn't solve what happens when a single large upload fails at 95% complete on a flaky connection. For objects past roughly 100MB, most object stores support  **multipart upload** : the client splits the file into independent parts, uploads them (often in parallel) and each part returns its own ETag. A final “complete” call hands the store the list of part ETags, and the store stitches them together into one object server-side.

  * A 100MB upload split into four 25MB parts that fails on part 3 at 95% complete only has to retry that one 25MB part, not the other 75MB already safely stored. On a connection that can sustain 5MB/s, that's a roughly 5-second retry instead of redoing the full 20-second upload from zero, and the saving grows with file size: a failed single-shot 2GB upload at 99% complete means redoing essentially all 2GB, while a multipart upload in 25MB parts only loses the one part in flight.

##### 07. Filesystem Ceilings and IOPS: Capacity - What a mount that's 'big enough' in gigabytes can still saturate under load.

* Local filesystems and shared mounts can become bottlenecks when many workers read or write concurrently. IOPS limits, disk throughput ceilings, and network saturation can make a filesystem slower than expected even when the storage is “large enough” in raw capacity.
* Storage Ceiling:
  * concurrent workers × IOPS per worker
  * 40 workers × 230 IOPS each ≈ 9,200 IOPS against a 10,000 IOPS mount: 92% utilized, matching the figure below
  * At 92% of a mount's IOPS ceiling, the symptom usually isn't an error, it's latency. Every read or write past that ceiling queues behind whatever else is already contending for the same disk, so a request that normally completes in 2ms can start taking 50-100ms under load, even though every individual operation eventually succeeds. Adding one more worker past the ceiling doesn't add throughput; it just lengthens the queue everyone else is waiting behind.
* This is why many systems avoid putting large or bursty file workloads on a shared database host or a single mount. Object storage separates the workload from any one node's local limits entirely.

##### 08. Durability and Retrieval: Reliability - What actually has to be true for data to survive years, not just this one request.

* Storage durability is not just “the data exists on disk.” It includes replication, checksum validation, restore paths, versioning, and how quickly the service can detect a failed or corrupted object. A delete or overwrite bug in a file layer is often far more visible than the same bug landing on a single database row.

  * Integrity: Checksums, versioning, and recovery policies catch silent corruption
  * Retrieval: The object key plus its metadata must stay consistent, or a GET returns the wrong thing
* **What "11 nines" durability actually means?**

  * Object stores commonly advertise **99.999999999% annual durability**(11 nines). That is not a claim that nothing ever fails; disks fail constantly, at scale. It is a claim about the *expected loss rate* across the whole system, after replication and integrity checks have already done their job.

    * Expected annual object loss = objects stored × (1 − durability) = 10,000,000 objects × (1 − 0.99999999999) = 0.0001 objects/year, on average
  * In practice that means a store holding 10 million objects would expect to lose roughly one object every 10,000 years on average, not that every object is individually guaranteed forever. That number only holds because of what's underneath it: redundancy across multiple copies or shards, and continuous checksum verification that can detect and repair corruption before it's ever read back wrong.
* **Erasure coding: redundancy without full copies:**

  * The simplest way to survive a lost disk is full replication: keep 3 complete copies of every object, which costs 3x the raw storage (200% overhead) to survive 2 simultaneous failures.**Erasure coding** gets similar protection for far less overhead: an object is split into data shards plus parity shards (for example, 6 data shards and 3 parity shards, written across different disks or racks), and the original object can be fully reconstructed from any 6 of those 9 shards. That survives any 3 simultaneous shard failures, similar protection to triple replication, for roughly 1.5x storage (50% overhead) instead of 3x.
* **3x replication:** 3 full copies. Survives 2 lost copies. ~200% storage overhead. Simple, and fast to read from any copy.
* **6+3 erasure coding:** 6 data shards + 3 parity shards; any 6 of 9 reconstruct the object. Survives 3 lost shards. ~50% overhead. Reconstruction costs CPU time the simple copy doesn't.
* **why it's cheaper:** Parity math (similar to RAID) needs far less redundant data than a full extra copy to tolerate the same number of failures.
* **The trade-off:** Reading a shard-reconstructed object costs more CPU than reading one of several complete copies directly, so erasure coding favors large, infrequently-reconstructed objects over small, latency-sensitive ones.
* **Cross-region replication: surviving a lost region:**

  * Replication and erasure coding both protect against losing a disk, a rack, or even a whole data center, but they're usually confined to one geographic region. **Cross-region replication** copies objects to a second, physically distant region, so a region-wide event (a natural disaster, a regional power or network outage) doesn't take the only copies of the data down with it. It's a separate decision from in-region redundancy: a design can have excellent 11-nines durability within one region and still lose availability entirely if that one region goes dark.

##### 9. CDN and Edge Delivery: Delivery - Caching popular objects close to users, without mistaking the cache for the store.

* Content delivery networks (CDNs) are a good match for static or semi-static objects read frequently from many geographic regions. They reduce origin load and improve latency by caching popular content closer to the user, at edge points of presence around the world.
* “A CDN keeps data durable, so you don't need real storage durability.”?

  * A CDN is a cache, not a store of record. Its edge copies are disposable: evicted on a TTL, purged on invalidation, or simply never populated for a rarely-requested object. If the origin object store loses the data, there is nothing left for the CDN to serve or refresh from. Durability is the object store's job; the CDN's job is only to make repeat reads of already-durable data faster.

##### 10. Storage Design Workflow: Method - A repeatable method for deciding what belongs in the database vs object storage.

* Start with data size and access shape. If you need strong transactional semantics and small, queryable rows, keep it in the database. If you need large uploads, low-cost retention, HTTP-style scaling, or geo-distributed access, move it to object storage instead.
  1. List the payload sizes and access patterns
  2. Separate metadata from binary data
  3. Choose the bucket and key layout, including tenant and versioning
  4. Decide whether uploads are app-mediated or go through a presigned URL
  5. Decide whether large uploads need to be multipart
  6. Set durability, versioning, and lifecycle rules
  7. Check filesystem and IOPS ceilings
  8. Decide whether a CDN is appropriate
  9. Document the operational restore path

##### 11. Practice Labs: Apply it - Found hands-on problem that build on every chapter above
