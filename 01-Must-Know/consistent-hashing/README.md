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

Different systems use different placement mechanisms. Redis Cluster hashes keys into a fixed set of slots and changes which node owns a slot. It can move a subset of slots as membership changes. For a worked walkthrough of ring placement and data movement, see the repository's [consistent hashing case notes](<../../06-Instant Messaging System/consistent-hashing.md>).

## Source

[Consistent Hashing · Hello Interview](https://www.hellointerview.com/learn/system-design/core-concepts/consistent-hashing)
