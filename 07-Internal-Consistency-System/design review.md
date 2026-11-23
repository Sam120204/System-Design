# Internal Consistency: Design Review

In data systems, consistency is often discussed at two levels:

- **Internal consistency** preserves business invariants within one system or source of truth. For example, the highest bid must never decrease, an account balance must not go negative, and inventory must not be oversold.
- **External consistency** concerns the relationship between the source of truth and other systems or effects, such as caches, search indexes, messages, notifications, payments, and reports. The required relationship may be immediate or eventually consistent, depending on the product contract.

## Internal consistency

First identify the **consistency domain**: all data that must change together to preserve an invariant. A single database can often protect that domain with transactions, constraints, and concurrency control. If it spans partitions or replicas, the design also needs coordination or a single ordered writer.

### 1. Transactions and isolation

- **Serializable** isolation makes concurrent transactions behave as if they ran one at a time. It provides a strong default for cross-row invariants, at a cost in contention and retries.
- **Snapshot isolation / repeatable read** can be faster, but behavior varies by database. Snapshot isolation can permit *write skew* when transactions read the same condition and update different rows.
- Use a transaction when the invariant fits within one database transaction, ideally on one shard.

### 2. Pessimistic locking

Lock the relevant rows before changing them, for example with `SELECT ... FOR UPDATE`. This fits high-contention operations or cases where a failed attempt is expensive. Keep transactions short, and use the database lock to protect database state; a distributed lock by itself does not make a database update atomic.

### 3. Optimistic concurrency control (OCC)

Read without locking, then detect a conflict at commit with a version check or conditional update, such as `UPDATE ... WHERE id = ? AND version = ?`. If no row is updated, retry or return a conflict. This works well when conflicts are uncommon; hot keys need bounded retries and backoff.

### 4. Constraints and atomic operations

Put invariants in the data store when possible: unique keys, foreign keys, `CHECK` constraints, atomic increments, and compare-and-set operations. For example, an inventory decrement should be conditional on sufficient stock, so the check and update happen atomically.

### 5. Single writer / per-key serialization

Route writes for one entity, such as an `auctionId`, through one ordered writer or partition. This simplifies competing updates, but failover must preserve the committed order and prevent an old writer from continuing to write.

### 6. Replication consistency

For multiple replicas of the same data, consensus protocols such as Raft or Paxos can agree on a committed write order. Quorum-based designs can also provide strong guarantees when their read and write rules are designed together. A quorum alone is not enough for linearizable reads; the system must also handle leader changes, stale replicas, and read validation.
