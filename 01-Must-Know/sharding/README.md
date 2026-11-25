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

## How Cassandra, MongoDB, and DynamoDB shard data

| Database | What you choose | What the database handles | What to check |
| --- | --- | --- | --- |
| **Cassandra** | A partition key for rows read together and clustering columns for order within a partition | A partitioner, typically `Murmur3Partitioner`, hashes the key to a token. In a vnode-based cluster, each physical node owns multiple token ranges on a consistent-hashing ring. The coordinator uses the token and replication strategy to find the replicas | For a high-volume event history, `(user_id, month)` bounds partition size. A read across months queries multiple partitions. One heavily used partition key can still be hot |
| **MongoDB** | A shard key for each sharded collection and a ranged or hashed strategy | The cluster divides the shard-key space into chunks. With a hashed key, chunk boundaries are in the hash space. `mongos` routes queries using shard metadata, and the balancer migrates chunks between shards | Ranged keys can preserve locality for range queries, and hashed keys can spread writes across many distinct values. Queries without a usable shard key can scatter to all shards. Hashing one repeated value still produces one value |
| **DynamoDB** | A partition key and, when useful, a sort key. Define secondary indexes for other query paths | DynamoDB hashes the partition key and manages physical placement, growth, and replication behind the service API. `Query` uses an exact partition key and can constrain the sort key | Choose a key that spreads traffic. If one logical key receives too many independent writes, add a bucket suffix and query across buckets for the full history. Check the key distribution on global secondary indexes too |

[Apache Cassandra architecture](https://cassandra.apache.org/doc/latest/cassandra/architecture/dynamo.html) · [MongoDB sharding](https://www.mongodb.com/docs/manual/sharding/) · [MongoDB balancer](https://www.mongodb.com/docs/manual/core/sharding-balancer-administration/) · [DynamoDB partitioning](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.Partitions.html) · [DynamoDB write sharding](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-sharding.html)

## Interview practice

Use the workload to identify the limiting resource, then explain the shard key, routing, cross-shard queries, and what happens when capacity grows.

### When would you shard a database?

> With 500 million users at 5 KB each, I estimate 2.5 TB of raw data before indexes and replicas. I'd compare storage growth, peak write throughput, and peak read throughput against a tuned primary and its read replicas. If the primary can't sustain our projected writes or storage with enough headroom, I'd shard. For a read bottleneck I'd also check whether indexing, caching, or replicas meet the latency target.

### How would you shard a social app, and what if a celebrity account gets hot?

> I'd hash `user_id` for user profiles and author timelines because those requests include the user ID. A feed across followed authors needs a separate query or feed view. I'd cache a celebrity's hot reads if the freshness target allows it. If that user's independent writes saturate one shard, I'd bucket those writes and budget for merging the buckets on reads. I'd size the initial deployment from peak load and growth, then move token ranges or logical buckets as capacity expands.

### How would you serve global trending posts across shards?

> If each post and its full score live on one shard, ask each shard for its local top K using the same ranking and tiebreaker, then merge the results. If scores are split across shards, combine partial counts before ranking. For frequent requests, I'd publish a precomputed top-K view on a refresh interval that matches how fresh trending needs to be.

### How would you transfer money between accounts on different shards?

> First I'd establish the ledger's invariant. If both entries must commit together, I'd use a database that supports atomic transactions across those shards or place the ledger entries within one transactional boundary. If pending transfers are acceptable, I'd persist the transfer state and use idempotent debit, credit, and compensation steps with reconciliation for failures.

## Source

[Sharding · Hello Interview](https://www.hellointerview.com/learn/system-design/core-concepts/sharding)

[Sharding in system design interviews · Hello Interview](https://www.hellointerview.com/learn/system-design/core-concepts/sharding#sharding-in-system-design-interviews) · [Quick reference · Hello Interview](https://www.hellointerview.com/learn/system-design/core-concepts/sharding/quick-reference)
