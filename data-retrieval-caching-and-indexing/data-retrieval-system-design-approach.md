# Data Retrieval: A System Design Approach

Choose a retrieval structure by starting with the query, not the data structure. Three useful questions are:

1. **What access pattern matters?** Exact-key lookup favors hashing; ordered scans and ranges favor search trees or skip lists; prefix, text, spatial, and similarity queries need structures built for those operations. A heap maintains only the next minimum or maximum, not a fully searchable order.
2. **How much of the data must the structure cover?** A cache may hold only hot items and can miss. An index is maintained for the rows and fields it covers, though a particular query may not be able to use it. Partial indexes are an explicit exception to full row coverage.
3. **What shape is the data?** A single ordered key, multiple spatial dimensions, text terms, and graph relationships call for different representations.

Also consider read/write ratio, memory budget, update cost, and whether the structure must survive a restart.

## 1. Sequential Structures: Arrays, Lists, and Skip Lists

- **Array:** Fast access by position; searching an unsorted array is a linear scan. A sorted array supports binary search but makes insertion expensive.
- **Linked list:** Cheap insertion or removal when the location is known; finding an item is generally a linear scan.
- **Skip list:** Layered links provide expected **O(log n)** search and update while preserving sorted order. It is useful for ranked sets and in-memory ordered indexes.

**Design examples:** Auction listings; game leaderboards.

## 2. Ordered Trees and Priority Structures

- **Balanced binary search trees** such as AVL and red-black trees support **O(log n)** lookup and update, plus ordered iteration.
- **B-trees and B+ trees** pack many keys into each page, keeping disk-based indexes shallow. B+ trees are common in database indexes and file systems.
- **Segment trees and Fenwick trees** answer range aggregates efficiently. Fenwick trees are especially suited to prefix sums; segment trees can support sums, minima, maxima, and other associative aggregates.
- **Heaps / priority queues** expose the minimum or maximum in **O(1)** and support insertion and removal of the top item in **O(log n)**. They do not support general range queries efficiently.

**Design examples:** Ranked statistics; cron schedulers and delayed-job queues. A timing wheel is another scheduling option with different tradeoffs.

## 3. Hash-Based Structures

- **Hash table:** Average **O(1)** exact-key lookup; common in caches, key-value stores, and language maps. It does not preserve key order.
- **Hash index:** Supports equality lookups but usually not range scans.
- **Consistent hashing:** Maps keys and nodes onto a logical ring to limit key movement when nodes join or leave. It is a sharding strategy rather than a local lookup index.

**Design examples:** Distributed cache; URL shortener; rate limiter.

## 4. Prefix Structures: Tries and Radix Trees

- **Trie:** Organizes keys by character or bit prefix, making prefix lookup natural. Common uses include autocomplete and IP routing.
- **Radix / Patricia tree:** Compresses chains of single-child trie nodes to reduce space. Adaptive radix trees (ART) add node layouts that adapt to the number of children.

**Design example:** Autocomplete and search suggestions.

## 5. Bitsets and Bitmap Indexes

- **Bitset / bitmap:** Bit `i` indicates whether item `i` belongs to a set. Bitwise operations make intersection and union fast, especially over dense integer ID ranges.
- **Compressed bitmap:** Formats such as Roaring choose representations for different containers, often saving space while retaining fast set operations. Bitmap indexes are useful in analytical queries.

**Design examples:** Permission sets; online-presence tracking; crawler URL deduplication after URLs are mapped to integer IDs.

## 6. Graph Structures

- **Adjacency list:** Stores each node's neighbors; efficient for sparse graphs and neighborhood traversal.
- **Adjacency matrix:** Stores edges in a two-dimensional array; fast edge checks but potentially **O(V²)** space.
- **Specialized graph indexes:** Precomputed structures can accelerate particular graph queries. HNSW, for example, uses a navigable graph for approximate nearest-neighbor search over vectors.

**Design example:** “People You May Know” or follow recommendations.

## 7. Text Search Structures

- **Inverted index:** Maps each term to a posting list of document IDs, often with positions or scores. It is the core of full-text search systems such as Elasticsearch and Solr.
- **Suffix array / suffix tree:** Supports substring search within text, usually in specialized workloads rather than a general web search index.

**Design example:** Search posts on a social platform.

## 8. Spatial and Multidimensional Structures

- **k-d tree:** Partitions a low-dimensional space for range and nearest-neighbor queries; performance can degrade as dimensions grow.
- **R-tree / quadtree:** Groups spatial regions for bounding-box and nearby-object queries.
- **Grid or Geohash buckets:** Maps locations into cells for a simple nearby-candidate search, followed by an exact distance check.

**Design examples:** Nearby restaurants, ride matching, food delivery, and nearby friends.

## 9. Logs and LSM Trees

- **Append-only log:** Records writes sequentially. Write-ahead logs (WALs) support recovery; event logs such as Kafka also support replay and consumption.
- **LSM tree:** Buffers writes in an ordered in-memory **MemTable**, flushes sorted **SSTables** to disk, and compacts them in the background. Reads may inspect multiple levels, so Bloom filters and block caches help control read cost. RocksDB, LevelDB, HBase, and TiKV use LSM-based storage engines.

**Design examples:** Write-heavy key-value storage and the storage layer of a distributed message queue.

The best choice is usually a combination: for example, a cache for repeated reads, a B+ tree or LSM index for durable lookup, and a specialized index for text or location queries.
