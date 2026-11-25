# Consistent Hashing

Consistent hashing maps keys to a changing set of servers while limiting how many keys move when a server joins or leaves. This matters for sharded storage, distributed caches, and routing that benefits from assigning a key to the same backend.

## Start with the placement problem

A ticketing service can begin with one database for events. When events are spread across several databases, the router needs a deterministic rule to find the owner of an event ID on every read and write. [Sharding](../sharding/README.md)

![A client connects through an application server to one database](https://www.hellointerview.com/_next/static/media/hxu2qcuU0ilH.0ke4y1r6opcss.svg)

![The application server routes requests to a set of three databases](https://www.hellointerview.com/_next/static/media/d79abC8D0oM9.0rglhcl744h6w.svg)

## Modulo hashing when membership changes

With `hash(event_id) % server_count`, a stable key hash gives a stable destination while the server count stays fixed. A key with hash 10 selects zero-based slot 1 with three servers and slot 2 with four servers.

![The server routes events to three databases using hash of the event ID modulo three](https://www.hellointerview.com/_next/static/media/WPVMxzwjqECM.1z7cg7fbua4xg.svg)

When the divisor changes, keys held by healthy servers can also move. With uniformly distributed hashes and fixed zero-based server IDs, going from three servers to four changes the destination for about 75% of keys. Removing a server changes the divisor again. In a cache that can cause a surge of misses, and in a database it can require considerable copying.

![Adding a fourth database changes the modulo rule and can redistribute data across the existing databases](https://www.hellointerview.com/_next/static/media/assMxUcKI1rJ.0p-i9qiss4026.svg)

![Removing a database changes the modulo rule and can redistribute keys from healthy databases too](https://www.hellointerview.com/_next/static/media/AD4zdHOdkvou.0hc6-13vqc_fd.svg)

## Place keys on a ring

Map server tokens and key hashes into the same fixed circular space. A key belongs to the first token clockwise from its hash, including a token exactly at the key's position. If the lookup passes the highest token, it wraps to the smallest one.

In the article's ring, database tokens start at 0, 25, 50, and 75. Adding DB5 at 90 transfers the interval `(75, 90]` from DB1 to DB5. With uniform key hashes over a 100-position ring, that is about 15% of keys.

![Adding DB5 at position 90 moves only the affected interval on the ring](https://www.hellointerview.com/_next/static/media/tIMVd3-xeV8n.3atsz08dmw67v.svg)

Removing DB2 at 25 moves its former interval `(0, 25]` to the next clockwise token, DB3 at 50. Other primary ownership ranges stay where they are.

![Removing DB2 transfers its interval to DB3, the next clockwise owner](https://www.hellointerview.com/_next/static/media/kuuYPX3Z3C-g.3935pl87w1djg.svg)

## Virtual nodes and hot keys

![Several virtual tokens per database are interleaved around the same hash ring](https://www.hellointerview.com/_next/static/media/c4QXQ-8FZC_W.3zu-s0zd1-05j.svg)

A **virtual node** is a token owned by a physical server. Assigning many tokens helps spread key ranges more evenly. When a server leaves, several surviving servers can take its intervals. Token counts can also reflect server capacity. One popular key still has one primary owner, so a surge on that key may need cached or replicated reads, or a way to split independently writable data. [Cache hot keys](../caching/README.md#stampedes-and-hot-keys)

## Replication and migration

The ring determines placement. For durable storage, replication and failure handling determine where existing copies can be read and when another replica may safely take writes. A planned membership change can stream the affected ranges, catch up concurrent writes, and cut over using an updated placement map. A cache can often let displaced keys refill on demand, with limits to protect the origin.

Different systems use different placement mechanisms. Cassandra uses a consistent-hashing token ring. Redis Cluster hashes each key into one of 16,384 fixed slots and can reassign selected slots as membership changes. DynamoDB hashes the partition key and manages physical partition placement behind its service API. [Cassandra architecture](https://cassandra.apache.org/doc/latest/cassandra/architecture/dynamo.html) · [Redis Cluster specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/) · [DynamoDB partitioning](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.Partitions.html)

For a worked walkthrough of ring placement and data movement, see the [consistent hashing case notes](<../../06-Instant Messaging System/consistent-hashing.md>).

## Interview practice

In a product design interview, name the partition key, explain which requests include it, and let the storage or cache system handle placement. For infrastructure questions such as designing a distributed cache, database, or message broker, walk through how clients find an owner, which keys move during a membership change, and how requests continue during a failure.

### Why choose consistent hashing over `hash(key) % server_count`?

> For a distributed cache that adds or removes servers, I want most keys to keep their current owner. Changing from three servers to four with modulo hashing remaps about 75% of uniformly hashed keys with stable server IDs. On a ring, adding a token changes primary ownership only for the interval it takes from the next clockwise token. That preserves most warm cache entries and limits the load on the origin during resizing.

### How would you look up a key and explain exactly what moves?

> Keep the tokens sorted and binary-search for the first token at or above the key's hash. Wrap to the smallest token if the key is past the last one. With tokens at 0, 25, 50, and 75, hashes 85 and 97 both route to 0. Add a token at 90 and only `(75, 90]` moves, so 85 goes to 90 and 97 stays at 0. If the token at 25 is removed, hashes in `(0, 25]` move to 50. The lookup takes `O(log V)` for `V` tokens.

### What do virtual nodes improve, and what if a popular event still overloads one node?

> I'd give each physical server several interleaved tokens. When a server leaves, its ranges can pass to several different survivors. That spreads ownership, but an event key still has a single primary owner. For a popular concert, I'd cache public event details or load-balance reads across replicas if the freshness target allows it. If the bottleneck is writes over independent seat records, I'd consider buckets based on seat ID and account for fan-out when reading the whole venue. Seat reservation still needs an authoritative concurrency check.

### What happens when a storage node fails, and how would you add capacity safely?

> For a temporary failure, route eligible requests to existing replicas according to the database's consistency and failover rules. The ring tells me where copies belong. For a permanent removal, rebuild missing copies to restore the replication factor. To add capacity, stream only the affected ranges, catch up concurrent writes, verify the replicas, and then switch routing. For a cache that can rebuild entries from the origin, I'd coalesce misses and limit fallback load while the new owner warms up.

### How would your explanation change for Cassandra, Redis Cluster, or DynamoDB?

> For Cassandra, I'd say the partition key hashes to a token that determines replica placement. For Redis Cluster, I'd explain that the key maps to a fixed slot and the cluster moves slot ownership when nodes change. For DynamoDB, I'd name an access-pattern-based partition key and explain that AWS hashes it and manages physical placement. In each case, I'd check for hot keys, since balancing the key space still leaves individual keys with very different traffic.

## Source

[Consistent Hashing · Hello Interview](https://www.hellointerview.com/learn/system-design/core-concepts/consistent-hashing)

[When to use consistent hashing in an interview · Hello Interview](https://www.hellointerview.com/learn/system-design/core-concepts/consistent-hashing#when-to-use-consistent-hashing-in-an-interview)
