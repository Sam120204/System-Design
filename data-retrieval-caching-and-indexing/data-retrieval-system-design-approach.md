# Data Retrieval: A System Design Approach

Start with the query requirements, then choose the data structure. Three questions help narrow the options:

1. **Does the query need ordering?**
   - Ordered lookups, range queries, and sorted scans favor search trees or skip lists.
   - Exact-key lookups favor hash tables; ordinary arrays and linked lists do not maintain sorted order automatically.
   - Heaps maintain a minimum or maximum rather than a fully sorted sequence. Spatial and inverted indexes organize data for specialized queries, so they do not fit neatly into an ordered/unordered split.

2. **Does the structure cover the full dataset or only a hot subset?**
   - Database indexes, inverted indexes, and LSM storage structures generally cover the data they are defined to maintain. Partial indexes deliberately cover only selected records.
   - Caches often keep a hot subset and can miss. Hash tables are common, with additional structures for expiration or eviction.

3. **What shape does the data have?**
   - **One-dimensional:** A single key or sequence, using ordinary indexes, arrays, or lists.
   - **Multidimensional / spatial:** Coordinates or geographic regions, using k-d trees, R-trees, QuadTrees, or GeoHash.
   - **Graph:** Relationships modeled explicitly as nodes and edges.

Also consider read/write ratio, memory budget, update cost, and recovery requirements.

## 1. Sequential Structures: Arrays, Linked Lists, and Skip Lists

**Main idea:** Arrays and linked lists organize a sequence; positions or pointers identify elements. Skip lists add layers of links to support efficient ordered search.

- **Array:** Fast access by index. Searching an unsorted array requires a linear scan; a sorted array supports binary search but makes insertion expensive.
- **Linked list:** Efficient insertion or removal when the location is known, but finding an element generally requires a linear scan.
- **Skip list:** Maintains sorted order with expected **O(log n)** search, insertion, and deletion. Useful for in-memory ordered indexes and ranked sets.

**Design examples:**

- Design Auction System.
- [Design Game Scoreboard / Leaderboard — Meta](design-game-scoreboard.md).

## 2. Trees and Priority Structures

**Main idea:** Search trees maintain key order for lookup and range scans; aggregate trees and heaps optimize different operations.

- **Binary search tree / balanced tree:** AVL and red-black trees keep lookup and updates at **O(log n)** while supporting ordered traversal.
- **B-tree / B+ tree:** Stores many keys in each node, making the tree shallow and suitable for disk pages. Common in database and file-system indexes.
- **Segment tree / Fenwick tree (BIT):** Supports range statistics. Fenwick trees are especially useful for prefix sums; segment trees can support sums, minima, maxima, and other aggregates. Applications include online statistics, monitoring, and ranking counts.
- **Heap / priority queue:** Optimized for the next minimum or maximum. Peeking at the top is **O(1)**; insertion and removing the top are **O(log n)**. A heap does not support general range queries efficiently.

**Design examples:**

- Design Cron Job Scheduler.
- Design Delayed Job Queue.

A timing wheel is another option for timers and scheduling; it does not require a heap.

## 3. Hash-Based Structures

**Main idea:** Optimize exact-key lookup, usually with average **O(1)** access, without preserving key order.

- **Hash table:** Common in caches, key-value stores, and language dictionaries or maps.
- **Consistent hashing / hash ring:** Maps keys and servers onto a logical ring to reduce key movement when servers join or leave. This is a distributed sharding strategy rather than a local lookup index.
- **Hash index:** Supports equality lookups but generally does not help with ordered range scans.

**Design examples:**

- Design Distributed Cache.
- Design URL Shortener.
- Design Rate Limiter.

## 4. Prefix Structures: Trie, Radix Tree, and Variants

**Main idea:** Organize keys by character or bit prefix to support prefix matching.

- **Trie:** Each path represents a sequence of characters or bits. Useful for autocomplete, prefix matching, and IP routing tables.
- **Radix tree / Patricia tree:** Compresses chains of nodes to reduce memory use.
- **Adaptive Radix Tree (ART):** Also adapts node layouts to the number of children, improving space usage and lookup efficiency.

These structures appear in routing systems and some in-memory indexes and key-value engines.

**Design example:** Design Autocomplete / Search Suggestions.

## 5. Bitsets and Bitmap Indexes

**Main idea:** Represent membership with bits and perform set operations efficiently.

- **Bitset / bitmap:** Bit `i = 1` means item `i` belongs to the set. Useful for permission sets, online/offline flags, and fast intersections or unions. Space efficiency depends on the size and density of the ID range.
- **Compressed bitmap, such as Roaring:** Uses different representations for sparse and dense containers to save space while retaining fast set operations. Common in analytical databases and OLAP bitmap indexes.

**Design example:** Crawler URL Deduplication, after URLs are mapped to suitable IDs. A bitmap alone does not establish whether two raw URLs are identical; the mapping must handle collisions if exact deduplication is required.

## 6. Graph Structures: Nodes and Edges

**Main idea:** Model relationships directly so that neighborhood traversal and connectivity queries are natural.

- **Adjacency list:** Each node stores its neighbors. Efficient for sparse graphs and traversing relationships.
- **Adjacency matrix:** A two-dimensional array records whether an edge exists or stores its weight. Edge lookup is fast, but space usage is **O(V²)**.
- **Specialized graph indexes:** Precomputed structures can accelerate shortest-path or reachability queries. HNSW uses a navigable graph for approximate nearest-neighbor search over vectors.

Applications include social relationships, recommendations, and route planning.

**Design examples:** People You May Know / Follow Recommendation.

## 7. Search and Text Structures

**Main idea:** Support lookup from a keyword or text pattern to matching documents.

- **Inverted index:** Maps each term to a posting list of document IDs, often with positions or scoring information. It is the core of full-text search engines such as Elasticsearch and Solr.
- **Suffix array / suffix tree:** Supports substring matching within text. More common in specialized text-processing workloads than in general web search indexes.

**Design example:** Design Twitter Search.

## 8. Spatial and Multidimensional Structures

**Main idea:** Organize coordinates and regions for nearby, range, and nearest-neighbor queries.

- **k-d tree:** Divides a k-dimensional space. Useful for low-dimensional nearest-neighbor and range queries; performance can degrade as dimensions grow.
- **R-tree:** Groups objects by bounding rectangles. Common in GIS, maps, and spatial range queries.
- **QuadTree:** Recursively divides a two-dimensional region into four cells. Useful for indexing points and adapting cell sizes to spatial density.
- **Grid / GeoHash buckets:** Maps locations into cells. Query the cells covering a search area, then apply exact distance filtering.

**Design examples:**

- [Design Yelp / 大众点评](design-yelp.md).
- Design Uber.
- Design Food Delivery.
- Nearby Friends.

## 9. Append-Only Logs and LSM Trees

**Main idea:** Favor sequential writes. Logs preserve a sequence of records; LSM trees organize buffered updates into sorted files that are merged over time.

- **Append-only log:** Appends records sequentially. Database write-ahead logs (WALs) support recovery; event logs such as Kafka also support replay and consumption.
- **LSM tree (Log-Structured Merge Tree):** Buffers updates in an ordered in-memory MemTable, often implemented with a skip list or tree. Flushes create sorted on-disk SSTables, and background compaction merges files. A WAL commonly protects updates that have not yet been flushed.
- **Read tradeoff:** Reads may need to inspect the MemTable and multiple files or levels. Bloom filters and block caches help reduce this work.

LevelDB, RocksDB, HBase, and TiKV use LSM-based storage engines. LSM trees still update memory and rewrite files during compaction; “append-only” describes the log more directly than the whole storage engine.

**Design example:** Design Distributed Message Queue. An append-only log is a natural foundation; an LSM engine can support metadata or indexed state, depending on the design.

Real systems often combine several families: a hash-based cache for repeated reads, a B+ tree or LSM engine for durable data, and a spatial or text index for specialized retrieval.
