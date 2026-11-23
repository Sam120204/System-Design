# Layered Architecture Design — Separation of Responsibilities

Most managed services, such as cloud databases, message queues, Kubernetes, caches, search services, and object storage, separate the **control plane** from the **data plane**. They may use different names, but the underlying idea is the same: separate management, orchestration, configuration, and lifecycle operations from business traffic, reads and writes, and computation.

## Characteristics of Managed Services

* **Fault isolation:** If the control plane fails, users may be unable to create resources, scale them, or change configuration, while the data plane can often continue serving requests, at least for a while. Conversely, data plane instability should not bring down the global control system.
* **Different scaling patterns:** The control plane handles relatively few but complex management requests. The data plane handles high throughput with low latency. Separation lets each plane be optimized, scaled, and rate-limited independently.
* **Clearer security and permission boundaries:** The control plane handles authorization, policies, quotas, and auditing. The data plane exposes only the necessary data access interfaces and can run in a more restricted network, such as a VPC or private subnet.
* **Easier multitenancy:** The control plane owns the tenant and resource models. The data plane can use either a shared cluster with isolation or a dedicated instance per tenant.
* **Independent engineering and evolution:** The control plane is closer to product and platform logic, while the data plane is closer to a distributed systems engine. Teams, release schedules, and SLOs can evolve independently.

## Common Responsibilities

The **control plane** typically handles:

* Resource orchestration: creating and deleting instances, scaling, upgrades, and migrations
* Configuration and policy: distributing settings, feature flags, quotas, billing, and tenant management
* Identity and security: authentication, authorization, RBAC, auditing, and certificate and key lifecycles
* Health and operations: monitoring, alerting, automated repair, scheduling, and incident workflows

The **data plane** typically handles:

* Business traffic: reads and writes, message delivery, queries, and object uploads and downloads
* Data replication and consistency: replicas, logs, sharding, leader election, and consensus
* Performance-sensitive paths: caching, batching, rate limiting, backpressure, and protocol implementation
* Placement near storage, networks, data, or users to minimize latency

## When to Separate Them

If a service meets any two of these conditions, separating the planes will likely become necessary:

1. **High availability:** Management system failures must not interrupt business traffic.
2. **Heavy data traffic:** The performance path must stay simple, fast, and predictable.
3. **Multitenancy or strong isolation:** Permissions, auditing, and quotas are complex.
4. **Automated operations:** Upgrades, migrations, and scaling happen frequently.

## Key Principle: The Upper Layer Controls the Lower Layer Through a Feedback Loop

The data plane should be controllable. The control plane programs its behavior:

* **Control plane:** Publishes the desired state.
* **Data plane:** Executes it and reports the observed state.

The control loop is:

`Observe → Decide → Act → Converge`

## Two-Way Communication Between the Control and Data Planes

### A. Control Channel (Act)

There are two common approaches:

* **Declarative updates:** The control plane writes the desired state to a source of truth, such as an API, database, or etcd. The data plane watches or polls it and reconciles its state, as in Kubernetes.
* **Imperative workflows:** The control plane initiates a sequence of actions for migration, rollback, or repair. A state machine tracks progress, with idempotent retries for failed steps.

Implementation requirements:

* **Idempotency:** Replaying the same desired state must be safe.
* **Versioning:** Specifications, schemas, and configurations must support rollback.
* **Rate limiting:** Control plane instability must not overload the data plane.
* **Security boundaries:** Use least privilege, auditing, and mutual authentication.

### B. Telemetry Channel (Observe)

Reporting status alone is insufficient; telemetry must support decisions:

* Metrics: SLOs, queue lag, resource utilization, and hotspots
* Events: leader election, eviction, failure reasons, and reconciliation errors
* Traces: latency breakdowns along critical paths, so decisions are based on the right bottlenecks
* Sampling and aggregation: prevent the observability system from overwhelming itself

## How This Maps to Cloud and Managed Services

* **Data plane = the fast path for business traffic and user requests.** Examples include database queries and writes, object storage PUT/GET, API gateway forwarding, message delivery, and compute or network packet forwarding. It is characterized by **high QPS, low latency, and strict SLOs**.
* **Control plane = orchestration and decisions for resources and configuration.** Examples include creating instances and clusters, scaling, scheduling shards and replicas, distributing routing rules, rotating keys, issuing certificates, triggering failover, and computing policies. It is generally **lower frequency but more complex and stateful**, with many workflows; it can often tolerate higher latency and eventual consistency.

## Benefits

### 1. Separate Control and Data Responsibilities

The data plane must handle traffic peaks while maintaining low latency and stability. The control plane acts more like a scheduling and orchestration system, using queues, retries, idempotency, and eventual consistency to manage complex operations. Separation lets each plane use its own technology stack and reliability strategy: for example, a high-performance forwarding or storage engine for the data plane, and a workflow engine, database, and message queue for the control plane.

### 2. Reliability and Blast Radius

* A control plane failure slows or blocks **resource creation, changes, and scaling**.
* A data plane failure causes **user requests to fail directly**.

With the planes separated, the data plane can continue serving requests using previously distributed configuration and established connections during a control plane outage, where possible. This limits the outage's impact.

### 3. Clearer Security and Permission Model

The control plane holds high-privilege capabilities such as creating and deleting resources, changing configuration, and managing keys. The data plane only needs permission to perform authorized actions. Separating them makes **least privilege, tenant isolation, and security boundaries by region or cell** easier to enforce, including distinct control plane permissions and data plane access boundaries for each tenant or cell.

---

# Layered Architecture Design — Topology Layers

**Topology layering** (a multi-tier or hierarchical topology) organizes request paths, caches, storage, and failure domains into tiers based on network distance, geography, or access cost. Nearby tiers serve common requests quickly; deeper tiers aggregate traffic and hold more authoritative data. This pattern appears in CDNs, object storage, message queues, search, gateways, distributed databases, and edge services.

Topology layers and the **responsibility layers** described above are independent design dimensions:

* **Responsibility layers** answer: Who makes decisions, and who serves the fast path? This is the control plane/data plane split.
* **Topology layers** answer: Where do requests and data travel, where can a request finish, and where is traffic aggregated?

A regional edge cache, for example, belongs to the data plane by responsibility and to an upper tier by topology. Its configuration may come from a separate control plane.

## Characteristics of a Layered Topology

* **Locality first:** Serve a request near the user or data when possible, avoiding unnecessary cross-city or cross-region hops.
* **Fan-in and fan-out control:** Many clients can converge on a smaller number of upstream requests. A regional shield, for example, merges misses from multiple edge sites before contacting the origin. Responses may then fan out to many clients.
* **Bounded failure domains:** Partition capacity into POPs, regions, or cells so a failure affects a limited area and does not cascade across the system.
* **Explicit cost model:** Cross-region transfer, provider links, origin I/O, and authoritative queries can be expensive. Layers reduce how often traffic uses these paths.
* **Operations by tier:** Measure, rate-limit, degrade, and recover each tier separately so an incident can be located and contained.

## Common Topology Patterns

### 1. Cache and Origin Hierarchy

`L1 Edge → L2 Shield/Parent → Origin`

* **L1 Edge:** Close to users. Serves hot objects, terminates TLS, and performs lightweight Layer 7 work.
* **L2 Shield/Parent:** A regional aggregation point. Improves effective cache hit rate, coalesces duplicate misses, and protects the origin.
* **Origin:** The authoritative source for the object or response. It serves misses and defines the freshness or consistency rules for cached copies.

The aim is to turn many edge misses for the same item into a small number of controlled origin requests. An extra tier is useful only when its saved origin traffic outweighs the added hop and operating complexity.

### 2. Geographic and Failure-Domain Hierarchy

`Global policy → Region → AZ or POP → Cell → Node`

* **Global:** Chooses broad placement, routing, and disaster recovery policy.
* **Region:** A large geographic boundary for latency, data residency, and disaster recovery.
* **AZ:** An availability-zone failure domain within a region.
* **POP:** An edge point of presence near users. A POP is not necessarily inside an application's AZ hierarchy; it is shown here as an alternative site-level tier.
* **Cell:** A smaller, mostly self-contained service unit with bounded capacity and blast radius. Depending on the system, cells can live within an AZ or span multiple AZs.
* **Node:** The machine or instance that performs work.

This pattern is common in databases, queues, search clusters, gateways, and edge compute. The exact nesting varies by provider and service; the important design question is which failures and traffic spikes each boundary contains.

### 3. Entry Routing Hierarchy

`DNS/GSLB and/or Anycast/BGP → POP entry → Layer 7 router → Upstream`

* **DNS/GSLB:** Makes a coarse entry choice using geography, provider, capacity, and health signals.
* **Anycast/BGP:** Routes a shared IP address toward a network-reachable POP. The selected POP depends on routing policy and network conditions, so it is not guaranteed to be geographically closest.
* **POP Layer 7 router:** Uses host, path, headers, tenant, and version to choose a cluster or backend.
* **Upstream:** The service that handles the request, possibly in another cell or region.

DNS and Anycast are alternative or complementary ways to reach an entry point; they are not always consecutive steps in every request. Routing must account for unhealthy and overloaded destinations.

### 4. Storage-Tier Hierarchy

`Memory / SSD → HDD → Object storage / archive`

* **Hot:** Low latency and limited capacity; suitable for frequently accessed data and indexes.
* **Warm:** Larger capacity and higher access latency; suitable for less frequent reads.
* **Cold:** Low cost per stored byte, but slower access or retrieval; suitable for infrequent reads and archives.

This is a hierarchy of **access proximity and cost**, even when tiers share a region. The authoritative copy is determined by the system's write and durability design: an object store may be the source of truth, while an archive may be only a backup.

## When Topology Layers Become Useful

If a service has two or more of these characteristics, a layered topology is often worth evaluating:

1. **Cross-region or cross-provider users:** Network routes and tail latency vary widely.
2. **Both hot and long-tail content:** Hot items benefit from nearby copies; long-tail misses need protection from overwhelming the core.
3. **Expensive origin or core resources:** Bandwidth, storage IOPS, write capacity, or strong consistency operations are costly.
4. **Strong disaster recovery and isolation needs:** A POP, region, or cell failure should affect a limited set of users.
5. **Many tenants or workloads sharing a platform:** Noisy neighbors must be contained within smaller domains.

This is a design heuristic, not a fixed rule. Each added layer brings another hop, cache consistency work, routing logic, and operational overhead. Measure whether it improves the actual latency, availability, and cost targets.

## Key Principle: Build a Manageable Traffic and Data Funnel

In a flat topology, many requests can reach the core directly. In a layered topology, **requests that can finish near the edge should do so**. Requests that must continue inward should be **aggregated, rate-limited, coalesced, and observed**.

| Tier | Primary role | Typical examples |
| --- | --- | --- |
| **Upper: Edge / Front** | Fast and close to users, usually with lightweight or cached state | CDN edge, front gateway, read cache, query-result cache |
| **Middle: Shield / Aggregation** | Merge requests, absorb bursts, and protect the core | CDN shield, regional cache, connection pool, queue buffer, query aggregator |
| **Lower: Core / Origin** | Hold authoritative state and enforce consistency and durability | Primary storage, shard leaders, object store, authoritative search writes |

An upper tier can return a cached, approximate, or degraded result only when the product's correctness and freshness requirements allow it.

## Channels Between Tiers

### A. Data Channel: Requests and Responses

* **Hit:** An upper tier serves the request locally, such as an edge or cache hit.
* **Miss:** It asks a deeper tier and may fill its cache from the response.
* **Failure:** It may try a healthy parent, a controlled origin route, or an approved degraded response. Unrestricted direct-origin fallback can overload the core during a widespread outage.

Implementation requirements:

* **End-to-end timeout budget:** Allocate a share of the total latency budget to each hop, including retries.
* **Request coalescing:** Combine concurrent requests for the same resource or query to prevent a cache stampede.
* **Backpressure and rate limiting:** Reject, queue, or shed excess work at upper tiers before lower tiers become unstable.

### B. Metadata and Control Signals

The control plane may be separate, but topology tiers still need signals that guide traffic and caching:

* **Health and capacity:** Which parent, region, or cell is available or congested?
* **Routing:** Which POP, parent, cell, or backend should receive a request?
* **Cache semantics:** What are the TTL, cache key, `Vary` rules, and negative-cache policy?
* **Feedback:** What are the hit rate, origin request rate, hotspots, latency, and error distribution at each tier?

Without these signals, extra hops add complexity without reliably improving performance or resilience.

## Benefits

### 1. Performance Through Locality

Hot data is served closer to users. Fewer requests cross long or unpredictable network paths, improving typical and tail latency when cache behavior and routing are well tuned.

### 2. Stability Through Core Protection

Upper tiers absorb bursts, middle tiers aggregate and regulate traffic, and the core focuses on correctness and durability. Failure domains and backpressure prevent one local surge from becoming a global overload.

### 3. Lower Use of Expensive Paths

Fewer cross-region transfers and fewer authoritative storage reads make bandwidth, IOPS, and core CPU demand easier to control. The savings must be compared with the cost of the extra tiers.

### 4. Easier Layer-by-Layer Operations

Teams can tune, rate-limit, fail over, or roll out a policy in one region, cell, or tier. Per-tier metrics show where requests slow down and where failures begin.
