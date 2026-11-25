# Key Technologies

These notes cover where each technology fits in a system design, how its main guarantees work, and which tradeoffs to explain in an interview. Each guide includes diagrams from the corresponding Hello Interview deep dive and worked interview questions.

| Technology | When it fits | What to work through |
| --- | --- | --- |
| [Redis](redis.md) | Low-latency access to cached or derived state | Data structures, hash slots, leases, rate limits, streams, and hot keys |
| [Elasticsearch](elasticsearch.md) | Full-text and geospatial search with ranking and filters | Mappings, shard fan-out, index freshness, and cursor pagination |
| [Kafka](kafka.md) | A durable event log with independent downstream consumers | Partition keys, consumer groups, offsets, retries, and delivery guarantees |
| [Temporal](temporal.md) | Multi-step workflows that need to recover, retry, or wait | Event history, deterministic replay, Activities, timers, and Signals |
| [API Gateway](api-gateway.md) | A shared public API in front of several backend services | Request routing, edge policies, load balancing, and regional failover |
| [Cassandra](cassandra.md) | Write-heavy storage with predictable partition-based queries | Query-driven modeling, partition sizing, replication, and tunable consistency |
| [DynamoDB](dynamodb.md) | Managed key-value or document storage with known access patterns | Partition and sort keys, secondary indexes, conditional writes, and capacity |

[Back to Must-Know](../README.md)
