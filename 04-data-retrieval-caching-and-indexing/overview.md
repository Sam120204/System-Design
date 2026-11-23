# Data Retrieval: Caching and Indexing

As datasets grow from gigabytes to terabytes, disk I/O often becomes a query bottleneck. A common optimization is to **reduce slow storage access by keeping hot data or frequently used lookup paths in memory**. Caches and indexes do this in different ways.

## 1. The Access Latency Ladder

These figures are rough orders of magnitude. Actual latency depends on the hardware, workload, and access pattern.

| Storage tier | Typical access latency |
| --- | ---: |
| L1 cache | ~0.5–1 ns |
| RAM | ~100 ns |
| NVMe SSD random read | ~10–100 µs |
| HDD random read | ~5–10 ms |

Even a fast NVMe SSD usually takes orders of magnitude longer than RAM for a random read.

## 2. Caching: Keeping Hot Data in Memory

Web systems commonly use in-process `HashMap`s, Redis, or Memcached as caches. They typically use hash tables to find a value by key, although a complete cache request may also incur network and serialization costs.

### Fast lookups

The average lookup complexity of a hash table is **O(1)**: the number of lookup steps usually stays roughly constant as the dataset grows. Hash collisions, resizing, and other cache-system overhead still affect real performance.

### Hits, misses, and capacity

Memory is limited, so a cache usually holds only a hot subset of the data:

- **Hit:** The item is in the cache and can be returned directly.
- **Miss:** The item is absent, so the system must fetch it from the database or another source. A higher hit rate generally means fewer requests reach that source.

When space runs low, an **eviction policy** chooses what to remove. For example, LRU (Least Recently Used) removes items that have not been accessed recently. **TTL (Time To Live)** specifies when an item expires, primarily to limit staleness. Expiration can free space, but it is not the same as capacity-based eviction.

## 3. Indexing: Shortening the Search Path

Hash lookups are good for exact matches, but an unordered hash table is a poor fit for range queries. Database indexes can support larger datasets, range queries, and ordered access. The index itself can live on disk; it does not all have to fit in memory.

### B+ trees: Signposts in memory

B+ trees are a common index structure in relational databases:

- **Internal nodes** hold keys and child pointers used for navigation.
- **Leaf nodes** hold index entries, which may contain the data or pointers to rows, depending on the index type.

Each node can have many children, so even a large index is usually only a few levels deep. In MySQL InnoDB, index pages are persisted on disk, while the root and frequently accessed internal pages are often kept in the Buffer Pool. A lookup follows these pages to a leaf. If that leaf page is not cached, it requires disk I/O. A secondary-index lookup may also need to fetch a data page using the primary key, so one query is not guaranteed to need only one disk read.

### Why is an index a space-for-time tradeoff?

An index uses extra storage and adds maintenance work to writes. In return, the database can locate relevant **pages** without scanning many unrelated pages for a selective query. A full table scan can still be the better choice when a query needs most of the table.

### LSM trees: Optimizing the write path

Instead of updating an ordered structure on disk for every write, an LSM tree first writes new data to an in-memory **MemTable**. It then flushes data in batches to on-disk **SSTables** and merges those files in the background. Real systems also use a log to recover writes that were still in memory when a failure occurred.

This turns many random updates into batched writes, but reads may have to check the MemTable and multiple SSTables. Bloom filters can skip files that definitely do not contain the requested key. LSM trees are often useful for write-heavy workloads, at the cost of read amplification and background compaction.

## 4. Caching vs. Indexing

| | Cache | Index |
| --- | --- | --- |
| What it stores | A disposable, rebuildable copy of hot data | A lookup structure maintained with the underlying data |
| When a lookup finds nothing | Cache miss; fetch from the source | No matching index entry, or the query cannot use that index |
| Main purpose | Reduce latency and database load from repeated requests | Locate records in large datasets; support range queries and ordering |

In short, **caching reduces repeated reads; indexing reduces how much data each read must search**. Systems often use both.
