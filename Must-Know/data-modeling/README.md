# Data Modeling

A data model defines the records a system stores, how it identifies them, and what relationships or invariants it enforces. Start with the operations the system must support. A social app needs to create posts, list a user's recent posts, read comments for one post, and prevent duplicate likes. Each operation implies a query or write path with its own expected volume and consistency requirement.

## Relational model and constraints

![PostgreSQL schema sketch showing users, posts, and comments with primary keys, foreign keys, and indexes](https://www.hellointerview.com/_next/static/media/iJpNDDfc128G.40oeukhq2cm25.svg)

| Table | Keys and relationships | Access pattern |
| --- | --- | --- |
| `users` | `user_id` primary key, unique `email` when required | Read a profile by ID or find an account by email |
| `posts` | `post_id` primary key, `user_id` foreign key | Read a post or list posts by author |
| `comments` | `comment_id` primary key, foreign keys to post and author | Page through comments for a post |
| `likes` | Composite primary key `(user_id, post_id)`, both foreign keys | Check or insert one user's like, without permitting duplicates |

A primary key identifies a record, foreign keys enforce references where the database supports them, and unique constraints protect domain rules. For posts by author, a composite index on `(user_id, created_at DESC, post_id DESC)` supports stable cursor pagination. Use a similar index starting with `post_id` for comments. If the app also needs to enumerate likes by post, use an index with `post_id` first. [Indexing](../indexing/README.md) · [API pagination](../api-design/README.md#design-lists-failures-and-updates)

## Other database models

### Document databases

Store a bounded group of fields in a JSON-like document when the application usually reads and updates them together. An embedded profile and settings object can be retrieved with one lookup. A potentially unbounded collection of posts or comments fits better in separately pageable records. Duplicated shared fields need an update or versioning policy.

### Key-value stores

![Server checks a key-value cache for a flattened response and falls back to normalized PostgreSQL tables](https://www.hellointerview.com/_next/static/media/pMnQOJzNrhzt.3724lgiaiotx8.svg)

Store a value under a known key, such as `profile:user_42`. The value can be a session or a flattened view assembled from multiple tables. Design the key around the lookup the app needs. If one value derives from multiple records, define how writes invalidate or rebuild it. [Caching](../caching/README.md)

### Wide-column databases

![User-keyed wide-column layout with posts organized by timestamps](https://www.hellointerview.com/_next/static/media/j7TmVSyZxUxN.3we8qi0spxq-r.svg)

Group records by a partition key and order within it using clustering columns where the engine supports them. For a user's recent posts, `user_id` can be the partition key and `(created_at, post_id)` the clustering columns. This supports ordered reads within a user. Add time buckets if one user's partition can grow beyond operational limits, and account for extra queries that span buckets.

### Graph databases

![Users and posts represented as nodes joined by posted and liked relationships](https://www.hellointerview.com/_next/static/media/YYVCZux140EO.3qpvtfpbt-7xj.svg)

Model entities as nodes and relationships as edges. A follower recommendation can traverse relationships such as `FOLLOWS` to discover people two steps away. Graph traversal fits workloads where variable-depth relationships are a primary access pattern. Bound depth and fan-out, since a highly connected node can still lead to expensive queries.

## Normalize the source and derive read views

Store an author's current name in the user record and reference the author from posts. This keeps a rename to one canonical update. If a measured feed query needs a faster shape, create a denormalized read model, such as a cache of post cards or a stored like count. Define who updates it, how quickly changes should appear, and how to rebuild it. An asynchronous count consumer can deduplicate by event ID and periodically reconcile with the canonical likes table.

Think separately about the fields a product wants to preserve historically. A checkout receipt may need the price charged at purchase time, even after the product's current price changes.

## Scale around ownership and queries

![Application routes posts to database shards that own different ID ranges](https://www.hellointerview.com/_next/static/media/d5ZtVcQ9_ZST.2-lrbb2o0_z5_.svg)

Co-locate data used in the same transaction when practical. Sharding posts by `user_id` keeps the author timeline on one shard, while a feed built from many authors may need fan-out or a precomputed view. A time-only key can direct all new writes to the same shard. Choose the partition key using the expected query mix, growth, and traffic skew. [Sharding](../sharding/README.md)

## Source

[Data Modeling · Hello Interview](https://www.hellointerview.com/learn/system-design/core-concepts/data-modeling)
