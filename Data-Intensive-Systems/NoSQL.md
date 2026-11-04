# NoSQL：Transaction 与数据库选型

PostgreSQL 可以存 JSON，MongoDB、DynamoDB 等 NoSQL 数据库也能支持 transaction（事务）。因此，**有没有 JSON 或 transaction 功能，不足以决定选型**。更关键的问题是：数据通常怎样被读取和修改？数据之间有多少关联？一个业务操作需要同时保证哪些数据正确？

## 先看 Access Pattern（访问模式）

| 问题 | 更适合优先考虑 RDBMS（关系型数据库）的情况 | 更适合优先考虑 NoSQL 的情况 |
| --- | --- | --- |
| Query pattern（查询方式） | 需要 JOIN、复杂 aggregation（聚合）、ad hoc query（临时查询）或报表，查询条件经常变化 | 查询路径固定，通常能根据已知的 key 或少量 index（索引）访问数据 |
| Data relationship（数据关系） | 实体之间关联密集，需要 foreign key（外键）或跨多行、多表维护约束 | 数据可以按文档或 partition（分区）组织，跨实体关联较少 |
| Consistency boundary（一致性边界） | 一个操作经常需要原子地修改多处数据 | 大多数操作落在单个 document、item 或 partition 内；跨边界的协调可以单独处理 |
| Scalability（扩展方式） | 接受关系模型和 transaction 带来的协调成本 | 愿意围绕 access pattern 设计数据布局，以换取更容易预测的 throughput（吞吐量）与扩展方式 |

这些是判断方向，不是硬规则。RDBMS 也能扩展，NoSQL 也能提供 index、aggregation 或 transaction；具体能力和代价取决于产品及部署方式。

RDBMS 通常以明确的 schema、SQL 查询和 transaction 约束为核心，适合关系复杂、查询会变化的场景。NoSQL 是一组不同类型数据库的统称：document store（文档数据库）、wide-column store（宽列数据库）和 key-value store（键值数据库）的模型并不一样。它们常要求先确定主要 access pattern，再据此设计文档结构、primary key（主键）或 partition key（分区键）。

**实用的选型顺序**：先列出核心读写请求，再确定 consistency boundary；最后比较查询能力、latency（延迟）、throughput、扩展方式、运维成本和团队经验。不要只对照产品功能清单。

## NoSQL 支持 Transaction，意味着什么？

Transaction 把一组读写作为一个提交单位：要么一起生效，要么失败时不留下部分提交。ACID 分别指 Atomicity（原子性）、Consistency（一致性）、Isolation（隔离性）和 Durability（持久性）。Transaction 是有用的能力，但通常需要额外的 coordination（协调）、conflict detection（冲突检测）或 retry（重试）。尤其当一个 transaction 跨多个 document、partition 或节点时，系统需要处理更多网络通信和故障情况。**支持 transaction**与**适合把所有业务操作都写成大 transaction**是两回事。

如果业务规则可以自然地放进一个 document、item 或 partition，就不必为了实现正确性而频繁使用 cross-partition transaction（跨分区事务）。反过来，如果订单、库存和支付等数据必须一起满足严格约束，就要明确数据库能提供什么保证，以及超出 transaction boundary（事务边界）时如何处理失败。Idempotency（幂等）、retry 和 compensation（补偿）可以帮助处理跨服务流程，但它们不自动等价于一个 ACID transaction。

下面比较几种数据库的 transaction boundary 与代价。**Spanner 属于 distributed SQL / NewSQL，这里作为对照，不归入 NoSQL。**

## MongoDB：Multi-document Transaction

MongoDB 从 4.0 起支持 replica set（副本集）上的 multi-document transaction（多文档事务），从 4.2 起支持 sharded cluster（分片集群）中的 distributed transaction（分布式事务）。Transaction 可以涉及多个 document 和 collection（集合）；跨 shard（分片）时，需要协调多个参与 shard。

- **如何工作**：单个 shard 内部使用其复制与 transaction 机制；cross-shard transaction（跨分片事务）由 coordinator（协调者）组织提交，采用类似 two-phase commit（2PC，两阶段提交）的过程。`readConcern: "snapshot"` 可让 transaction 读取一致的 snapshot（快照），但这不等于 strict serializability（严格可串行化）。
- **适合什么**：少量、短时间内必须一起 commit（提交）的跨文档修改。
- **代价**：相比单文档操作，transaction 增加协调、资源占用与冲突处理成本；cross-shard transaction 还增加网络往返，并可能与 chunk migration（分片数据迁移）等操作相互影响。故障时，应用也要按错误类型处理 retry 或结果不确定的 commit。

**选型提醒**：如果一个常见业务操作总要跨许多 document 或 shard，先检查 document boundary（文档边界）与 shard key（分片键）是否符合 access pattern，再评估 transaction 性能。

## Cassandra：单个 Partition 内的 Conditional Update

Cassandra 的 lightweight transaction（LWT，轻量级事务）主要提供 `IF NOT EXISTS`、`IF column = value` 等 conditional update（条件更新）。它解决的是**单个 partition 范围内的并发竞争**，不是任意多个 partition 之间的通用 ACID transaction。

- **如何工作**：LWT 使用 Paxos，让参与 replica（副本）就条件是否成立及写入结果达成一致。
- **适合什么**：创建唯一记录、防止关键数据被并发覆盖等少量关键路径。
- **代价**：需要多轮 replica 通信，latency 和 throughput 通常明显差于普通写入；大量使用 LWT 会改变原本写入路径的性能特征。

**选型提醒**：如果业务经常需要跨多个 partition 一起修改并保持 strong consistency（强一致），不能把 LWT 当成通用 transaction 使用。

## DynamoDB：范围受限的 ACID Transaction

DynamoDB 提供 `TransactWriteItems` 和 `TransactGetItems`。Transaction 可以涉及同一 AWS account（账户）、同一 Region 内的多个 table（表），但**最多 100 个不同 item，总数据量不超过 4 MB**。相关 transaction 操作提供 serializable isolation（可串行化隔离）；具体到与非 transaction API 混用时，应查看各 API 的 isolation guarantee（隔离保证）。

- **如何工作**：服务在涉及的 item 之间协调 commit 并检测冲突。应用通过 transaction API 声明需要一起成功或失败的操作；普通读写仍可使用常规 API。
- **适合什么**：有明确 item 集合、规模受限的 atomic read/write（原子读写），例如同时更新几条相关记录。
- **代价**：transaction 比普通单 item 操作需要更多协调与容量，冲突时可能失败并需要 retry；transaction 范围受 item 数量、大小、account 和 Region 限制。

**选型提醒**：如果一个业务操作频繁触及大量 item，或要求 cross-Region atomic commit（跨地域原子提交），就需要重新设计数据边界或选择其他方案。

## Spanner / NewSQL：把 Distributed Transaction 作为核心能力

Spanner 提供关系模型、SQL 和跨 shard、跨 Region 的 read-write transaction（读写事务）。它的目标是在分布式部署下保持强 transaction 语义。TiDB、CockroachDB 也走 distributed SQL 路线，但具体协议和保证不能直接等同于 Spanner。

- **如何工作**：Spanner 使用 replica group（复制组）保存数据，并结合 lock（锁）、MVCC（多版本并发控制）和 TrueTime 来协调 transaction、确定 commit order（提交顺序）。TrueTime 给出带不确定性范围的时间；系统通过 commit wait（提交等待）保证 external consistency（外部一致性）：如果一个 transaction 已完成，之后才开始的 transaction 不会被排在它之前。
- **适合什么**：关系复杂、需要 SQL，同时又要求跨 shard 或多 Region strong consistency 的场景。
- **代价**：跨 shard、跨 Region 协调会增加写入 latency；系统与基础设施复杂，成本通常高于单 Region、单 key 的简单读写。使用托管 Spanner 时，时钟基础设施由服务提供方维护。

## 一句话总结

技术选型的关键不只是“哪些数据库支持 transaction”，还要看**业务的主要 access pattern 和 consistency boundary，是否与数据库擅长的操作范围一致**。先问哪些数据必须一起正确，再看查询会怎样变化、transaction 会跨多少 partition，以及为此愿意付出多少 latency 和运维成本。

## 参考资料

- [MongoDB：Transactions](https://www.mongodb.com/docs/manual/core/transactions/)
- [MongoDB：Transactions Production Considerations](https://www.mongodb.com/docs/manual/core/transactions-production-consideration/)
- [Apache Cassandra：CQL Conditional Updates and LWT](https://cassandra.apache.org/doc/latest/cassandra/developing/cql/dml.html)
- [AWS：DynamoDB Transaction APIs](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/transaction-apis.html)
- [Google Cloud：Spanner TrueTime and External Consistency](https://cloud.google.com/spanner/docs/true-time-external-consistency)
