# CAP Theorem

When a network partition prevents replicas from coordinating, a distributed service cannot guarantee both linearizable consistency and availability for the affected operations. The design decision is what clients can read or write on each side while the network is split.

| Property | Meaning in CAP |
| --- | --- |
| Consistency | Operations behave as if they occurred on one copy in an order compatible with real time. A read after a completed write sees that write or a later one |
| Availability | Every request received by a non-failing node eventually receives a valid result for the requested operation. Refusing a request solely because replicas cannot coordinate sacrifices availability for that operation |
| Partition tolerance | The guarantees under discussion must hold even when messages between groups of nodes can be lost or delayed indefinitely |

CAP consistency concerns the behavior clients observe across replicas. ACID consistency concerns preserving database invariants across transactions. An application may need both concepts when it decides how to handle inventory writes.

## Walk through a partition

![A US server accepts a profile update and replicates it to Europe for a reader there](https://www.hellointerview.com/_next/static/media/kDCiksyGIusv.1b9ceu0wl0-ni.svg)

A user changes their display name through the US replica. Under normal replication, the Europe replica receives the update and can show the new name.

![The connection between US and Europe replicas is interrupted while one accepts a write and the other receives a read](https://www.hellointerview.com/_next/static/media/ui_Mdimd0FfA.29o8_fa8_bcar.svg)

Suppose the regions lose contact after the update commits in the US and before it reaches Europe. The Europe replica cannot tell whether it has the latest value. If the service promises linearizable reads, Europe must wait, route to an authoritative copy if reachable, or reject the read when it cannot establish freshness. If the product permits stale data, Europe can immediately show the last replicated display name. A service that also accepts writes on both sides needs an explicit conflict-resolution policy when they reconnect.

## Set policy per operation

| Operation | Behavior during a partition |
| --- | --- |
| Reserve the last seat | Require the authoritative writer or a quorum that can safely enforce the reservation. Decline or defer the operation if the required coordination is unavailable |
| Browse event descriptions | Serve a local replica or cached copy when the product permits a stale description |
| Create a mutual match | If the product allows only one match record per pair, use an authoritative write or an idempotent reconciliation design that enforces that invariant before publishing the match |
| Browse another user's profile | Serve an available local copy. A separate read-your-writes guarantee can make the editing user see their own update once the app can route to a copy that has it |

For a ticketing service, the seat map can be approximate while browsing. Confirmation is the operation that must enforce the invariant that a seat can have at most one active reservation. Document the behavior clients see when that operation cannot complete. [Regionalization](../networking-fundamentals/14-regionalization.md) · [API Design](../api-design/README.md)

## Describe the consistency guarantee

- **Linearizable.** An operation becomes visible as if it happened once between invocation and completion. Later operations observe the result in real-time order.
- **Causal.** A reader observes effects after the writes they depend on, such as seeing a post before a comment on it.
- **Read-your-writes.** A session sees its own completed updates, even if other clients can see older versions.
- **Eventual.** Replicas converge if updates stop and replication can make progress. State the application's expected staleness separately if it needs a time bound.

The behavior depends on the operation, replication protocol, and configuration. During a partition, a quorum-based service may keep the majority side operating while the minority cannot safely complete the same operation. Test a split as well as recovery, including how stale reads or conflicting writes become visible after communication returns.

## Source

[CAP Theorem · Hello Interview](https://www.hellointerview.com/learn/system-design/core-concepts/cap-theorem)
