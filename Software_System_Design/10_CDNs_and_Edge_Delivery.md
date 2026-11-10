# CDNs & Edge Delivery

Starting from why physical distance costs a request anything at all, build up to cache keys, TTL policy, origin shielding, purge propagation, and the privacy boundary every edge cache has to respect.

##### 01. Why Distance is Latency: Foundations - Why physical distance actually costs a request.

* Every byte your browser sends and receives has to physically travel through fiber-optic cable, and that travel takes time, roughly bounded by the speed of light, no matter how fast the server on the other end is. A user in São Paulo talking to a server in Virginia pays that distance cost on every single request, even if the server itself responds instantly the moment the request arrives.
* A **CDN** (content delivery network) is the “open a local warehouse” move, applied to web content. Instead of every user reaching all the way back to one origin server, the CDN stocks copies of popular content in small facilities (called  **points of presence** , or POPs) scattered near where users actually are.
  * A CDN doesn't make your origin server faster. It makes most requests never have to reach your origin server at all: the response comes from a warehouse a few miles away instead of one halfway around the world.

##### 02. What a CDN Is: Mechanics - Points of presence, origins, and why caching copies near users changes everything

* A CDN is a distributed layer of caching servers placed near users. Instead of every user hitting the same origin app or storage service, the CDN stores copies of content at nearby edge locations. That reduces network distance, lowers latency, and protects backend capacity. All three follow directly from the same move: put a copy closer to the reader.
* It's most valuable when the same asset is requested repeatedly from many locations: product images, JS bundles, media files, rendered HTML pages, static assets. It isn't a full replacement for the application tier, but for the fraction of traffic that's cacheable, it can offload the overwhelming majority of request volume the origin would otherwise have to serve directly.
* It's most valuable when the same asset is requested repeatedly from many locations: product images, JS bundles, media files, rendered HTML pages, static assets. It isn't a full replacement for the application tier, but for the fraction of traffic that's cacheable, it can offload the overwhelming majority of request volume the origin would otherwise have to serve directly.
* **CDN (Content Delivery Network):** A network of edge locations that cache and serve content closer to end users.
* **POP (Point of presence):** an edge location where CDN servers are deployed, close to a cluster of users.
* **Origin:** The source application or storage system that sits behind the CDN.
* **TTL:** Time-to-live; the maximum time a cached response can be served before it must be revalidated.
* **Cache-Control:** The response header from origin that tells the edge how long, and how publicly, it may cache a response.
* **ETag:** An opaque fingerprint of a resource's current content, used to check freshness without re-sending the body.
* **Cache Key:** The combination of URL, query parameters, and selected headers a CDN uses to decide if two requests are “the same.”
* **Purging:** Explicitly removing stale cached content, rather than waiting for its TTL to naturally expire.
* **Revalidation:** Asking the origin whether a cached response is still current, typically via an ETag, before serving it again.
* **Shield:** A protected edge or origin-proxied layer that absorbs traffic before it reaches the origin.
* **Cache miss:** A request the CDN can't serve from cache, so it must fetch (or re-fetch) from origin.
* **Static content:** Content that is stable and suitable for long-lived caching, such as images or versioned JS bundles.

##### 03. Anatomy of a CDN Request: Mechanics - The literal headers that decides whether the edge serves from cache or asks origin

* “The edge caches things” is true but vague. Concretely: when a request arrives at a POP, the edge computes a **cache key** from it, checks whether it already has a stored response under that exact key, and if so, whether that response is still inside its allowed freshness window. Two response headers from origin drive almost this entire decision.
* Cache-Control: public, max-age=86400 tells the edge two things at once: **public** means shared caches (not just the requesting browser) are allowed to store this, and **max-age=86400** is the TTL (86,400 seconds, or one day) that the edge may serve this response without checking back with origin at all. Once that window elapses, the cached copy is considered stale, and the edge has to do something before serving it again.
* Why bother with a fingerprint instead of just re-fetching the whole file?
  * If the edge always re-downloaded the full response to check whether it changed, revalidation would cost almost as much bandwidth as never caching at all. An ETag lets the edge send a tiny conditional request instead (“is the resource still fingerprint a1b2c3d4?”) and origin can answer with an empty **304 Not Modified** when nothing changed, or the full body only when it actually has.

##### 04. Cache Rules and TTL: Policy - Choosing how long each kind of content is allowed to be served stale.

Not everything should be cached forever, and not everything should be cached at all. Some content is static and safe to cache for days. Some content changes by the minute and needs constant revalidation. Some content is specific to one signed-in user and must never be served to anyone else. The cache policy, which is really just a TTL choice per content type, determines whether the CDN makes the system fast and safe, or stale and dangerous.

* A static asset, like a versioned JS bundle or a product photo, can safely take a long TTL, often 7 to 30 days, because a new deploy ships under a new hashed filename instead of overwriting the old one in place. A dynamic page with live pricing or inventory needs a short TTL, often 30 to 120 seconds, or no edge caching at all: long enough to absorb a meaningful share of repeated reads, short enough that a price change is never visibly stale for more than a couple of minutes.

| Content Type       | Typical Cache Policy         | Why                                      |
| ------------------ | ---------------------------- | ---------------------------------------- |
| Static Assests     | Long TTL, aggressive caching | Rerely change and are read frequently    |
| Product HTML       | Medium TTL with revalidation | Content changes, but not every second    |
| User specific data | No CDN cache or short TTL    | Private and highly personalized          |
| API responses      | Short TTL or bypass cache    | Often dynamic and sensitive to freshness |

##### 05.

| Cache Keys and Regionality: Mechanics - What actually makes two requests "the same" for caching purposes |
| -------------------------------------------------------------------------------------------------------- |

CDN routing isn't just “all requests go to one place.” It can distinguish by URL path, host header, geography, or request type: some objects should be served from the nearest global cache, some API calls should bypass caching entirely, some should route to the closest regional origin. Underneath all of that routing sits one genuinely confusing but important idea: the  **cache key** .

A cache key is whatever the CDN decides makes two requests count as “the same” request for caching purposes: by default, usually the full URL including its query string. Two requests that don't match on the cache key are treated as entirely unrelated, even if they hit the same path.

This cuts both ways. A query parameter that's functionally irrelevant to the response (like a marketing tracking parameter) can still fragment the cache into many near-duplicate entries if it's left in the key, tanking the hit rate. And a header that's functionally load-bearing (like **Accept-Encoding**, which determines whether the response body is gzip-compressed or not) has to be part of the key, or a client that can't decompress gzip can end up served a gzip-compressed cached copy meant for someone else.

| Request Type     | Recommended Behaviour               | Reason                                           |
| ---------------- | ----------------------------------- | ------------------------------------------------ |
| Hero Image       | Cache aggressively at edge          | Reads are repetitive and mostly static           |
| Search API       | Bypass, or very short TTL           | Freshness is critical and results vary per query |
| Static JS bundle | Cache long, version by path or hash | Better latency and near-perfect cache reuse      |

* Before asking “what TTL should this have,” ask “what's the cache key.” A perfect TTL on the wrong key either serves stale content to the wrong audience or barely caches anything at all.
* How much does an unnecessary cache-key field actually cost?
  * More than it looks like. A **/deals** page that would otherwise get a 95% hit rate can see that collapse below 5% if a marketing tracking parameter that doesn't change the response, like an unstripped **?utm_source=**, is left in the cache key: 200 different campaign links to the same content become 200 separate cache entries, each one a guaranteed first-time miss. Before shipping a route, list which query parameters and headers actually change the response, and build the cache key from exactly those, nothing more.

##### 06. Origin Protection and Shield: Reliability - Keeping a viral spike from ever reaching the app and database tier

* One of the biggest wins of a CDN is reducing origin traffic. A viral spike should be absorbed by edge caches, not cause a crushing surge on the application servers and database. But hundreds of independent POPs, each experiencing its own first cache miss for a suddenly popular resource, can still collectively fan out into a stampede on origin at the exact same moment.
* A **shield** is a protected layer between the wide edge network and origin that collapses that fan-out: many POPs route their misses through one shield location first, which makes a single request to origin and then serves every waiting POP from that one response.
* Not all traffic should hit the origin, and not all content should bypass the CDN. The right policy balances freshness, correctness, and cost.

##### 07. Purge, Revalidate and Freshness: Operations - Why purging is not instant, and what revalidation buys you instead.

* When content changes, a stale edge copy can keep old data alive for as long as its TTL allows. **Purging** removes a cache entry before its TTL naturally expires. **Revalidation** asks the origin whether the cached version still matches before serving it again, typically via the ETag mechanism from Chapter 03. Both need real operational controls and clear cache-invalidation playbooks.
* "Purging a CDN removes the content everywhere, instantly.”?
  * A purge call is a single request, but it fans out asynchronously to every POP in the network: potentially hundreds of independent locations around the world. Nearby, well-connected POPs typically clear within a second or two; distant or heavily loaded ones can lag for tens of seconds. For a few moments after you issue a purge, different users in different regions can legitimately see different content. Designs that require instant, atomic global freshness need a short TTL or forced revalidation, not just a purge button.
* Purging asks every POP to drop a cached object outright, which (per Figure 8 above) typically clears nearby locations in 1 to 5 seconds and distant or heavily loaded ones in 30 to 60 seconds. Revalidation is cheaper per request and has no fan-out delay at all: one edge sends a small conditional request (Chapter 03's If-None-Match) and gets back either a 304 in a few milliseconds or the fresh body. A route that changes often is usually better served by a short TTL plus revalidation than by a purge call on every single update.
* A CDN does not magically solve stale content by itself. It solves latency and origin load; the freshness policy, and the operational discipline of purging correctly, still has to be designed deliberately.

##### 08. Security and Privacy at the Edge: Trust Boundary - TLS, DDoS, absorption and the danger of caching a private response

* CDNs often provide TLS termination, DDoS absorption, bot filtering, and edge access controls: the edge is genuinely a security asset, not just a performance one. But it's also a trust boundary: it can serve private content incorrectly if configuration is wrong, so it's critical to separate public, cacheable assets from authenticated or user-specific responses.
* The DDoS-absorption claim in real numbers: a CDN's edge network, spread across hundreds of POPs, can often absorb a multi-gigabit-per-second volumetric attack, sometimes spiking into the hundreds of Gbps, by spreading it across that entire footprint. A single origin with a few Gbps of uplink capacity would be saturated by that same traffic in seconds if it ever reached the origin directly.
* “A CDN makes dynamic, personalized content faster too, automatically.”?
  * A CDN only speeds up content it can safely cache and reuse across requesters. Personalized responses (an order history, a dashboard, anything gated behind a login) are, by definition, different for every user. Caching them under a cache key that doesn't account for identity doesn't make them faster; it risks serving one user's private response to a completely different user. Genuinely personalized content either bypasses the shared edge cache, or needs a deliberately identity-aware cache key design. It never gets edge speed for free.

##### 09. CDN Design Workflow: Method - A repeatable method for any edge-delivery design question

* A good CDN design begins by classifying content: what can be cached, what must stay fresh, and what must never be cached at all. Then it defines TTLs, cache keys, and invalidation rules. Only after that does it decide the protection layer and origin-shielding model.
  1. List content classes and access frequency
  2. Decide the cache key for each route: URL, query params, and headers that matter
  3. Choose TTL and revalidation policy per content class
  4. Decide which paths bypass the CDN entirely
  5. Protect the origin with shielding or rate limits
  6. Set purge and rollback procedures, and expect propagation delay
  7. Define edge security and TLS boundaries
  8. Measure cache hit rate and latency by region
  9. Verify privacy and sensitive-data handling under the chosen cache key

##### 10. Practice Labs: Apply it - Four hands-on problems that build on every chapter above
