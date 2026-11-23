# Sharding

Sharding divides a dataset across multiple database servers so each owns a subset of the records. It can add storage and write capacity when one server is the bottleneck. A table can also be partitioned within one database to improve operations such as partition pruning or retention work. Replication creates copies for availability and possibly read capacity, and a sharded deployment often replicates each shard.

![An original database at capacity splits record ranges across three shards](https://www.hellointerview.com/_next/static/media/mWKZDof4xdai.2ntxc6vbv7i7v.svg)

## Choose a shard key and routing rule

List the most frequent reads, writes, and transactions. Pick a stable, high-cardinality key that appears in common queries, keeps related records together, and spreads load under realistic traffic. For a product that mostly reads each user's orders, `user_id` can keep an order history on one shard. Check whether heavy users can still dominate a shard.

| Routing strategy | How it works | Trade-off |
| --- | --- | --- |
| Range | Divide ordered keys into nonoverlapping intervals, such as user IDs below 1 million and from 1 million to below 2 million | Supports range locality. Monotonically increasing keys can concentrate new writes |
| Hash | Hash the shard key into a bucket or slot and map it to a shard | Spreads many independent keys. A range in the original key may span many shards |
| Directory | Resolve a tenant, account, or bucket through a placement map | Permits custom placement and migration. The map needs availability and safe updates |

Fixed logical buckets let a system move a subset of buckets to a new server. Simple `hash(key) % server_count` can remap many keys when the count changes. Consistent hashing is another placement strategy that reduces remapping. [Consistent Hashing](../consistent-hashing/README.md)

## Handle hot shards

![One shard receives much more traffic than its neighbors because it owns a celebrity account](https://www.hellointerview.com/_next/static/media/yGa07BWkF-vk.1vvt68f120_0m.svg)

Evenly distributed records can still produce uneven traffic. For a celebrity account, cache or replicate hot reads when freshness allows. A directory can isolate an especially busy tenant. If one logical key must sustain more writes than one shard can handle, split independently writable data by a second dimension or a set of buckets. That increases fan-out for reads that need the entire key, so check the query pattern before splitting. Watch shard-level QPS, CPU, storage, and tail latency. [Caching](../caching/README.md)

## Cross-shard reads and writes

![A service asks each of three shards for ten posts, merges their results, and returns the global top ten](https://www.hellointerview.com/_next/static/media/egaEcwXA_OXo.0_yghj6qgua-0.svg)

A lookup with the shard key goes to its owner. For a global top-ten list, the service can fetch each shard's local top ten and merge them if each item belongs to one shard and the shards use the same ranking and tiebreaker. For scores built by combining data across shards, aggregate the scores before ranking. Scatter-gather adds calls and makes tail latency sensitive to slow shards. For frequent global queries, consider a materialized aggregate, a search index, or a lookup table with its own freshness policy.

Keep related writes together when they must commit atomically. A supported distributed transaction can enforce atomicity across shards at the cost of coordination. A saga coordinates steps and compensating actions when the business can tolerate intermediate states. It needs retriable, idempotent steps and a plan for failed compensation. [Data Modeling](../data-modeling/README.md)

## Move data safely

Adding capacity means moving ownership as well as copying rows. For an online move, copy the target key range, catch up writes from a change stream or the database's migration mechanism, validate the copy, then switch routing with a cutover that prevents both shards from accepting authoritative writes. Let stale routers refresh or redirect, and retain the old copy until rollback or cleanup policy permits deletion. Managed systems differ in which steps they automate.

## Source

[Sharding · Hello Interview](https://www.hellointerview.com/learn/system-design/core-concepts/sharding)
