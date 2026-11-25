# Caching

A cache keeps a reusable copy of data or a computed response closer to the caller. Start with a repeated access pattern, choose a key that identifies the result, and specify how stale a copy may be. Cache hits save time and upstream work. Cache misses still need enough capacity at the source of truth.

## Where to cache

### Shared application cache

![Application servers read a shared cache and fall back to the database on a miss](https://www.hellointerview.com/_next/static/media/i-fdmuqwKz7t.2y32e8ylfd7zm.svg)

Several app instances can share Redis or Memcached for objects such as profiles and computed feeds. A network call still reaches the cache, so measure hit and miss latency as well as origin load. Store only the representation needed by the request and include the tenant or authorization scope when it affects the result.

### CDN edge cache

![A client reads through a nearby CDN edge that can reach the origin server](https://www.hellointerview.com/_next/static/media/8-VTEX8ILxuC.1qposik0ha4oq.svg)

A content delivery network (CDN) operates edge locations called **points of presence (PoPs)**. They can cache images, video segments, static HTML, and public API responses according to the application's cache policy.

1. A user in India requests a product image whose origin is in Virginia. CDN routing directs the request to an edge location close to the user on the network.
2. On a **cache hit**, the edge returns its fresh copy directly. On a **cache miss**, the edge fetches from an upstream CDN cache or the origin, then stores the response if its policy permits.
3. Later requests for the same image and cache key can hit that edge while the cached response remains fresh. Another edge may still have to fetch a copy.

The source article illustrates the latency difference with roughly **250–300 ms** for the distant origin and **20–40 ms** for a nearby edge hit. These numbers are illustrative. Actual latency depends on routing, cache state, and response size. [CDN example · Hello Interview](https://www.hellointerview.com/learn/system-design/core-concepts/caching#cdn-content-delivery-network)

If an image-sizing parameter changes the response, include it in the cache key. Versioned asset URLs let you publish updates under new keys. Give changing public responses a freshness policy, and restrict shared caching for personalized responses. [Edge locations and origin routing](../networking-fundamentals/14-regionalization.md#how-a-cdn-works-at-each-edge-location)

### Client-side cache

![A client keeps a local cache before sending a request to the application](https://www.hellointerview.com/_next/static/media/rCCAPLqdCG10.3jjqfty1czxwl.svg)

Browsers can reuse HTTP responses or revalidate a stored response with an `ETag`. Mobile apps may keep a local copy for offline use. Expiry, version checks, and sync are especially important when devices can hold a copy for a long time. [HTTP cache semantics](../networking-fundamentals/06-http.md)

### In-process cache

![Each application server keeps a small cache in its own process](https://www.hellointerview.com/_next/static/media/SElGEsWUAM_-.402yv8qq8u703.svg)

Process memory avoids even a shared-cache network hop. It suits small, hot values such as configuration. Every replica holds its own cache, so rolling deployments and restarts can leave it cold, and a value can expire at different times on different instances.

## Read and write patterns

### Cache-aside

![The application checks the cache, reads from the database on a miss, and fills the cache](https://www.hellointerview.com/_next/static/media/rps7cY2Z78dC.2hte0db5lfbvr.svg)

The application checks the cache first. On a miss it reads the database, stores the result with an appropriate TTL, and returns it. For a write, commit to the database, then invalidate or update the cached copy. The application owns the cache policy and the miss path.

### Write-through

![Writes pass through a cache layer and update the database synchronously](https://www.hellointerview.com/_next/static/media/0sCq9wO17cwY.0w-chlnxgaxlp.svg)

The cache layer writes through to the backing store before acknowledging success. The extra work adds write latency and can populate entries that never get read. Define how partial failures are handled, since a cache update and a database update remain separate unless the design coordinates them.

### Write-behind

![Writes reach the cache first and flush to the database asynchronously](https://www.hellointerview.com/_next/static/media/EeFY1o-NsaC6.3jue8v55es_mo.svg)

The cache layer acknowledges a write before flushing to the database. Batching can improve write throughput, but buffered data can be lost on a crash. If acknowledged writes must survive, persist them in a durable log or queue first and handle replay, ordering, and backpressure.

### Read-through

![A cache layer handles a miss by loading the result from the database](https://www.hellointerview.com/_next/static/media/_HDdSiMZHoWQ.3zwz0jqx7amir.svg)

The application asks a cache abstraction for a value. That abstraction owns the loader and populates the value on a miss. A CDN fetching an object from its origin is an example.

## Expiration, eviction, and invalidation

**TTL** limits how long an entry is considered fresh. **Eviction** frees capacity. LRU favors recently accessed items, LFU favors frequently accessed ones, and FIFO removes the oldest inserted entries first. A cache can evict a fresh value under memory pressure.

**Invalidation** removes a copy when its source changes. A profile update can commit to the database and then delete the profile cache key. A reader that fetched an older version before the commit can still refill the key after deletion. For stricter freshness, use versions or a coordinated fill policy. A durable change event can make invalidation retryable. Decide which fields can use stale values and which operations must read an authoritative store. [CAP Theorem](../cap-theorem/README.md)

## Stampedes and hot keys

![Many simultaneous cache misses overwhelm a database after a popular value expires](https://www.hellointerview.com/_next/static/media/HwQJEtYDMm14.35h3zejev60q7.svg)

- **Stampede.** When a popular entry expires, many callers can try to rebuild it. Coalesce requests for the same key so one fetch is in flight. Jitter TTLs across keys to spread simultaneous expirations. Early refresh or serving a stale version during refresh can help where freshness rules permit.
- **Hot key.** A key can overload one cache node even with a high hit rate. Short-lived local copies or replicated reads can spread load. Track per-key and per-shard traffic to see whether the bottleneck is one object or a whole shard. [Consistent Hashing](../consistent-hashing/README.md#virtual-nodes-and-hot-keys)
- **Cache outage.** Rate-limit or shed fallback traffic when the database cannot handle the full uncached load. Track hit rate, misses, eviction, origin QPS, and tail latency together.

## Worked example of a profile cache

Suppose profile pages drive 500 reads per second against the database at peak. With a 95% hit rate in a shared cache, about 25 reads per second would reach the database on steady-state misses, excluding refresh and write traffic.

Use cache-aside with a key such as `profile:user_42`. Choose a TTL based on how much profile staleness the product can tolerate and use LRU if older reads are less likely to recur. After a profile edit commits, invalidate the cached copy. Coalesce repeated misses for the same profile and bound database fallback during a cache outage. Compare latency and origin load for warm, cold, and unavailable cache states.

## Interview practice

Start with a repeated read or expensive computation. Explain what goes in the cache, the key and placement, how it gets populated, how it expires or is invalidated, and how the source handles misses and outages.

### When would you add Redis to a profile endpoint?

> Suppose profiles drive 500 peak database reads per second and each read takes 30 ms. With a 95% cache hit rate, I'd expect about 25 database reads per second from steady-state misses. I'd use a shared Redis cache and cache-aside on `profile:{user_id}`, adding tenant or visibility scope if it changes the response. I'd set the TTL from the staleness budget, use an eviction policy such as LRU for memory pressure, and invalidate after a profile edit commits. I'd measure the hit rate, origin load, and p95 latency to confirm the benefit.

### Where would you cache a trending feed and its images?

> I'd cache frequently reused feed results in Redis when the rankings can lag, using a key such as `trending:posts:global` and a refresh interval that matches the freshness target. I'd serve public images from a CDN with versioned URLs and include any variant that affects the bytes in the cache key. I'd consider short-lived in-process copies if one shared-cache key becomes a bottleneck.

### What happens when a popular key expires or Redis goes down?

> For one expiring key, I'd coalesce concurrent misses so one request rebuilds it. I can jitter TTLs to spread expirations across different keys and serve stale results during refresh if the endpoint allows them. If Redis goes down, I'd cap database fallback concurrency and shed excess work so the origin survives the traffic spike.

### How would you handle freshness after writes?

> I'd commit updates to the authoritative store and invalidate the cached copy. If concurrent fills can race with invalidation, I'd use versions or a coordinated fill when freshness matters. I can serve profile display fields with bounded staleness, but I'd confirm ticket availability from authoritative inventory before booking. If I propose write-behind for throughput, I'd explain how a durable buffer protects acknowledged writes and how the service replays them.

## Source

[Caching · Hello Interview](https://www.hellointerview.com/learn/system-design/core-concepts/caching)

[Caching in system design interviews · Hello Interview](https://www.hellointerview.com/learn/system-design/core-concepts/caching#caching-in-system-design-interviews)
