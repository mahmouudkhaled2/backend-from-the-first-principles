# Caching for Backend Engineers: Why It's Everywhere and How It Actually Works

Caching is the mechanism of storing a fast-access subset of data so that repeated or expensive operations can be served instantly instead of recomputed or refetched every time.

## Key Takeaways

- Caching exists to solve exactly two recurring problems: avoiding **repeated expensive computation** and avoiding **repeatedly transferring large amounts of data** — and it shows up at every layer of a system, from CPU hardware to DNS to application-level databases.
- As backend engineers, the most relevant caching layer is **in-memory key-value stores** like **Redis** or **Memcached**, which trade RAM's limited, volatile capacity for dramatically faster read/write speed compared to disk-based databases.
- Real backend caching decisions come down to a handful of recurring patterns: **lazy (cache-aside) vs. write-through** caching strategies, and **eviction policies** (No Eviction, LRU, LFU, TTL-based) for deciding what to discard once the cache fills up.

## 1. What Is Caching?

In simple terms, caching is a mechanism that **decreases the time and effort needed to perform some piece of work**. More technically, it means taking a **subset** of some primary dataset — chosen based on factors like frequency of use, recency of use, and probability of future use — and storing that subset in a location that is faster and easier to access than the primary storage.

Caching is a foundational technique behind nearly every high-performance application, especially those that need to keep latency in the tens of milliseconds or lower.

## 2. Technical Flow (The "Hops")

The general lifecycle of a cached request follows the same shape across nearly every caching context (CDNs, DNS, application caches):

1. A client makes a request for some resource or triggers some computation.
2. The system first checks whether the result already exists in the cache.
3. If found — a **cache hit** — the result is returned immediately, without repeating the expensive work.
4. If not found — a **cache miss** — the system falls back to the original, slower source (a database query, an expensive computation, an origin server, etc.).
5. The freshly computed or fetched result is then written into the cache (often with a **TTL**, or time-to-live) before being returned to the client.
6. Subsequent requests for the same data are served directly from the cache until the entry expires or is evicted.
7. When the cache reaches its capacity limit, an **eviction policy** determines which existing entries are removed to make room for new ones.

## 3. Why Do We Need Caching?

- **Avoiding repeated expensive computation** — operations involving heavy algorithms (e.g., search ranking, trend detection, machine-learning-driven analysis) are far too costly to rerun for every single request.
- **Avoiding repeatedly transferring large amounts of data** — serving large files (video, images) from the origin for every user across the globe would be slow and would overload origin servers.
- **Reducing database load** — offloading frequent, repeated reads to a faster storage layer keeps the primary database free for the operations that actually need it.
- **Reducing latency for end users** — reading from memory-based caches is dramatically faster than disk-based databases or remote computation.
- **Preventing server crashes under load** — without caching, popular endpoints hit by large numbers of concurrent users (e.g., a trending-topics page) could overwhelm backend infrastructure.
- **Controlling cost and rate limits** — caching responses from external, metered, or rate-limited APIs avoids unnecessary billing and throttling.

## 4. Key Comparisons: Caching vs. Always Hitting the Primary Source

### Why can't we just always query the primary database or recompute the result directly?

1. **Compute cost** — many operations (search ranking, trend analysis, complex joins/aggregations across millions of rows) are computationally expensive; repeating them per request doesn't scale.
2. **Latency** — disk-based databases are inherently slower to read from than in-memory storage, since disk access (even SSD) is mechanically and structurally slower than RAM access.
3. **Server load and stability** — high-traffic endpoints (a celebrity's social profile, a trending page, a flash sale product page) can generate enough concurrent requests to overwhelm a database if every request hits it directly.
4. **Cost and rate limits** — external APIs are often metered or rate-limited; recalling them on every request needlessly increases billing and risks throttling.
5. **Diminishing returns on freshness** — a lot of frequently-read data (product details, user profiles, weather data) changes rarely enough that serving slightly-stale cached data is an acceptable trade-off for the performance gained.

The counterpoint the video is careful to note: caching isn't free — cache storage (especially RAM-based) is more expensive and more limited in capacity than disk storage, and write-heavy or highly dynamic data isn't a good caching candidate, so not everything should be cached.

## 5. Deep Dive: Caching Layers and Redis-Style In-Memory Databases

### 5.1 Three practical caching layers for backend engineers

The video frames caching into three layers most relevant to backend work:

- **Network-level caching** — CDNs and DNS.
- **Hardware-level caching** — CPU L1/L2/L3 caches and RAM, which underpin why in-memory databases are fast.
- **Software-level caching** — application-facing tools like Redis or Memcached, which interact with hardware-level caching (RAM) through a library/software interface.

### 5.2 CDNs (Content Delivery Networks)

A CDN caches content on **Edge servers** — servers geographically close to end users — grouped into regional clusters called **PoPs (points of presence)**. The flow:

1. A user requests a resource (image, video, web page) via a URL.
2. The browser issues a **DNS query** to resolve the domain.
3. The CDN's DNS system routes the request to the nearest PoP, factoring in the user's geographic location and network conditions (e.g., routing to a lower-quality video variant on a poor connection).
4. The edge server checks whether the requested content is cached (**cache hit**) or not (**cache miss**).
5. On a cache hit, content is served directly from the edge. On a cache miss, the edge server fetches the content from the **origin server**, serves it to the user, and caches it for future requests.
6. Cached content is held for a configured **TTL**, after which a new request triggers a fresh fetch from the origin, keeping content from going permanently stale.

This is how platforms like Netflix deliver terabytes of pre-encoded, multi-resolution video content globally with minimal buffering, and how platforms like Vercel serve frontend assets from the region closest to the requesting user rather than a single origin.

### 5.3 DNS caching

DNS resolution is cache-heavy at nearly every layer, because fully resolving a domain name can require several hops:

1. The user's device sends a DNS query, typically to a **recursive resolver** provided by their ISP or a public DNS provider (e.g., Google Public DNS, Cloudflare).
2. The recursive resolver first checks its **local cache**. A hit returns the IP immediately.
3. On a miss, it queries a **root server** (there are 13 root server addresses, operated by a small number of organizations, though each address is backed by many physical/anycast servers globally — the video's "13 or 14" estimate lines up with this).
4. The root server doesn't hold the final answer but refers the resolver to the appropriate **TLD (top-level domain) server** (e.g., for `.com`).
5. The TLD server refers the resolver further to the **authoritative name server** for the specific domain.
6. The authoritative name server returns the actual IP address, and the recursive resolver returns it to the user — hence "recursive," since it may traverse multiple servers before resolving.

Because this chain is expensive to repeat, caching happens at nearly every layer: the **operating system** checks its own local DNS cache first, then the **browser's** own DNS cache, then the **recursive resolver's** cache, and in some cases even **authoritative name servers** cache results to reduce repeated lookups.

### 5.4 Hardware-level caching and why RAM-based databases are fast

CPUs maintain **L1, L2, and L3 caches** to speed up repeated or predictable memory access (e.g., sequential array traversal benefits from predictive prefetching into cache). Below that sits **RAM (main memory)**, and below that, secondary storage (disk).

RAM is called "random access" memory because, unlike a mechanical hard disk (which must physically seek to a location), RAM can access any memory address directly via an electrical signal, making access time effectively constant regardless of location. This is what makes RAM dramatically faster than disk — but RAM is also **volatile** (data is lost on power-off) and **limited in capacity**, whereas disk storage is cheaper, larger, and persistent.

This is exactly why technologies like **Redis** and **Memcached** exist: they store data primarily in RAM for speed, while relying on secondary storage behind the scenes for persistence (reloading data into memory on startup). They're commonly described with two defining traits:

- **In-memory** — data lives in RAM rather than on disk, unlike traditional relational databases like PostgreSQL or MySQL.
- **Key-value based** — instead of enforcing a strict relational schema (tables, rows, columns), these databases use a simple key-to-value model, where a value can be a string, number, list, JSON object, etc., depending on the specific technology.

### 5.5 Caching strategies: Lazy (Cache-Aside) vs. Write-Through

- **Lazy caching / cache-aside** — the cache is only populated reactively: a request checks the cache, and only on a miss does the system fetch from the primary source and populate the cache for next time. This is the more common default pattern.
- **Write-through caching** — every write (create/update) operation updates both the primary database and the cache **at the same time**, within the same request. This keeps the cache always fresh (never serving stale data) at the cost of added overhead on every write operation, since two storage systems must be updated instead of one.

### 5.6 Eviction policies

Because in-memory cache capacity is limited, a policy is needed to decide what to discard once the cache is full:

- **No eviction** — no policy configured; new writes fail with an out-of-memory error once the cache is full.
- **LRU (Least Recently Used)** — evicts whichever cached entry was accessed longest ago.
- **LFU (Least Frequently Used)** — evicts whichever cached entry has been accessed the fewest total times.
- **TTL-based eviction** — evicts (or simply expires) entries based on which have the least time remaining before their configured time-to-live runs out.

### 5.7 Common backend use cases for in-memory caching

- **Database query caching** — caching the result of compute-intensive, frequently-hit queries (e.g., joins/aggregations powering a dashboard or landing page) with a TTL, to reduce both latency and database load. E-commerce platforms similarly cache relatively static data like product details, prices, and inventory to avoid re-querying the database for every page view — especially useful during high-traffic events like sales.
- **Session storage** — after authentication, session tokens are typically stored in Redis or a similar in-memory store rather than a relational database, since session lookups happen on nearly every API call and need to be as fast as possible.
- **External API caching** — responses from third-party APIs (e.g., a weather API) are cached with a TTL to avoid re-calling the external service on every request, reducing both latency and the risk of hitting rate limits or increasing billing costs.
- **Rate limiting** — rate-limiting middleware typically stores a per-client request counter (keyed by an IP address extracted from a header such as `X-Forwarded-For`) in an in-memory store, incrementing it per request within a time window and rejecting requests that exceed the configured limit with an **HTTP 429 (Too Many Requests)** response. Redis-style storage is preferred over a relational database here specifically to minimize per-request latency and avoid flooding the primary database with counter updates.
- **Social media / user profile data** — relatively static, infrequently-changing data like user profile information is cached to handle read-heavy traffic (e.g., a celebrity profile or trending topics page) without repeatedly hitting the database for data that rarely changes.

## 6. Interview Questions & Key Concepts

### Fundamentals

**Q: What is caching, and what two general problems does it solve?**
*Caching stores a fast-access subset of data so repeated work doesn't need to be redone. It primarily solves two problems: avoiding repeated expensive computation (e.g., search ranking, aggregations) and avoiding repeatedly transferring large volumes of data (e.g., video content, API payloads).*

**Q: What's the difference between a cache hit and a cache miss, and why does it matter for system design?**
*A cache hit means the requested data was already present in the cache and can be returned immediately; a cache miss means it wasn't, requiring a fallback to the primary source (database, computation, origin server) before the result can be cached and returned. Cache hit ratio is a key metric for evaluating whether a caching strategy is actually effective — a low hit ratio suggests the wrong data is being cached or the TTL is misconfigured.*

**Q: Why is Redis faster than a traditional relational database like PostgreSQL or MySQL for the same lookup?**
*Redis stores data primarily in RAM, which allows direct, constant-time access to any memory address via electrical signals, whereas relational databases are disk-based and must contend with slower, often mechanical or structurally slower storage access. The trade-off is that RAM is volatile and far more limited in capacity, which is why Redis is used as a fast layer in front of a database rather than a full replacement for it.*

### Architecture & Design Decisions

**Q: What's the difference between lazy (cache-aside) and write-through caching, and when would you choose one over the other?**
*Lazy caching only populates the cache reactively on a miss, keeping writes simple but risking a cold cache after invalidation or restart. Write-through caching updates the cache and the database together on every write, keeping the cache always fresh at the cost of extra latency and complexity on write paths. Cache-aside/lazy loading is the more common default in modern systems, with write-through reserved for cases where staleness is unacceptable; a related pattern not covered in the video, write-behind (writing to the cache first and asynchronously flushing to the database), is also common in current system-design discussions as a way to reduce write latency further, at the cost of a small durability risk if the cache fails before the flush completes.*

**Q: How do cache eviction policies like LRU, LFU, and TTL differ, and how would you choose between them?**
*LRU evicts the entry that hasn't been accessed in the longest time, which suits workloads where recent access predicts future access. LFU evicts the entry accessed the fewest total times, which suits workloads where popularity is more stable over time than recency. TTL-based eviction removes entries after a fixed lifespan regardless of access pattern, which suits data with a known freshness window (like weather data or product prices). In practice, Redis and similar systems let you combine an eviction policy (e.g., `allkeys-lru`) with per-key TTLs, so the choice is often "both," not "either."*

**Q: How would you design a rate limiter using an in-memory cache, and why not use a relational database for it?**
*A typical approach keys a counter by client identifier (commonly an IP address pulled from a header like `X-Forwarded-For`), incrementing it on each request within a fixed or sliding time window and rejecting requests once a threshold is exceeded (HTTP 429). An in-memory store is preferred over a relational database because the counter check happens on nearly every request, so its latency directly impacts every API call, and routing that volume of small, frequent writes through a relational database would both slow every request and add unnecessary load to the primary datastore. Note that a fixed-window counter (as described in the video) is simple but can allow bursts at window boundaries; production rate limiters often use a sliding-window or token-bucket algorithm instead for smoother enforcement — worth mentioning as a refinement if asked to go deeper.*

### Real-World Scenarios & Trade-offs

**Q: What kind of data is a good candidate for caching, and what kind isn't?**
*Good candidates are read-heavy, infrequently-changing, and either expensive to compute or expensive to transfer — product details, user profiles, static configuration, third-party API responses with slow-changing values. Poor candidates are highly dynamic or write-heavy data where staleness would cause real problems (e.g., live financial balances), since caching such data either requires aggressive invalidation (adding complexity) or risks serving incorrect information.*

**Q: How do CDNs reduce latency and origin server load for a globally distributed user base?**
*By caching content at edge servers grouped into regional PoPs close to end users, and using DNS-based routing to direct each user's request to their nearest PoP. This keeps most requests from ever reaching the origin server, only falling back to it on a cache miss, and uses TTLs to periodically refresh content so edge caches don't serve permanently stale data.*

**Q: Why does DNS resolution rely so heavily on caching at multiple layers?**
*Fully resolving a domain name can require traversing a chain of servers — recursive resolver, root server, TLD server, authoritative name server — which is expensive to repeat for every request. To avoid this, caching is layered at the operating system, the browser, the recursive resolver, and sometimes the authoritative name server itself, so that only the very first lookup for a given domain typically pays the full cost of the recursive chain.*

---

*Approximate word count: 2,650 words.*
