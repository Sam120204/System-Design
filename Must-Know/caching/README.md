# Caching

A cache keeps a reusable copy of data or a computed response closer to the caller. Start with a repeated access pattern, choose a key that identifies the result, and specify how stale a copy may be. Cache hits save time and upstream work. Cache misses still need enough capacity at the source of truth.

## Where to cache

### Shared application cache

![Application servers read a shared cache and fall back to the database on a miss](https://www.hellointerview.com/_next/static/media/i-fdmuqwKz7t.2y32e8ylfd7zm.svg)

Several app instances can share Redis or Memcached for objects such as profiles and computed feeds. A network call still reaches the cache, so measure hit and miss latency as well as origin load. Store only the representation needed by the request and include the tenant or authorization scope when it affects the result.

### CDN edge cache

![A client reads through a nearby CDN edge that can reach the origin server](https://www.hellointerview.com/_next/static/media/8-VTEX8ILxuC.1qposik0ha4oq.svg)

A CDN PoP can return a cached image, video segment, or public response without a trip to the origin. A miss reaches a higher cache tier or the origin and can fill the edge cache. Decide which fields change the response and therefore belong in its cache key. Restrict shared caching of personalized content. [Edge locations and origin routing](../networking-fundamentals/14-regionalization.md#how-a-cdn-works-at-each-edge-location)

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

## Source

[Caching · Hello Interview](https://www.hellointerview.com/learn/system-design/core-concepts/caching)
