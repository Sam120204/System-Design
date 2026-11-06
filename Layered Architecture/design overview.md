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

# Key Principle: The Upper Layer Controls the Lower Layer Through a Feedback Loop

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
