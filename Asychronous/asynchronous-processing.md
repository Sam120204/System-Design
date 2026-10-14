# 基础知识：异步处理（Asynchronous Processing）

## 1. 核心概念（Core Concepts）：为什么我们需要异步？

在深入技术细节之前，我们需要理解同步（Synchronous）**与**异步（Asynchronous）的本质区别：调用者是否必须等待操作完成，才能继续执行。

### 1.1 同步处理（Synchronous Processing）：传统的“等待”模式

同步处理是传统的执行方式：调用者发送请求，等待结果返回后，才能执行下一步。

- **举例（Example）**：调用发送电子邮件的函数时，当前执行流程要等到邮件服务器连接成功、接收数据并返回确认，才能继续。
- **阻塞 I/O（Blocking I/O）**：执行线程（Thread）在等待网络或磁盘操作期间无法继续处理其他工作。如果这些操作发生在用户界面线程（UI Thread）上，界面就可能冻结，影响用户体验（User Experience, UX）。
- **注意**：同步调用不一定冻结整个程序；其他线程仍可能继续运行。

### 1.2 异步处理（Asynchronous Processing）：发起操作后继续执行

异步处理允许调用者在操作尚未完成时继续执行，结果可以稍后获取。

- **核心理念**：把耗时任务交给后台处理（Background Processing），先告诉用户“处理中（Processing）”，让用户继续操作。
- **异步代码（Asynchronous Code）**：通过 `async/await`、Promise 或 Future 表达异步操作。在典型的非阻塞 I/O（Non-blocking I/O）实现中，等待期间可以释放执行资源来处理其他工作。
- **系统级异步（System-level Asynchrony）**：在微服务（Microservices）中，通过消息队列（Message Queue）把任务交给独立服务，实现服务解耦（Service Decoupling）。
- **即发即忘（Fire-and-forget）**：这是异步使用方式之一，表示调用者不等待或使用结果。异步任务也可以返回结果、报告失败，或支持取消（Cancellation），因此异步不等于即发即忘。

> `async/await` 是代码层面的控制方式，消息队列是系统间传递任务的基础设施。使用 `async/await` 本身并不保证底层操作是非阻塞的，也不意味着任务一定在另一个线程上执行。

### 1.3 回调与通知（Callbacks and Notifications）

如果需要知道异步任务的结果，可以采用完成通知（Completion Notification），也可以查询任务状态。

- **回调（Callback）**：操作完成后，执行预先注册的函数或处理逻辑。
- **Webhook**：服务通过 HTTP 请求通知另一个服务，例如支付平台通知商家“支付成功”。需要考虑鉴权、签名和失败重试。
- **WebSocket / Server-Sent Events（SSE，服务器发送事件）**：把状态更新实时推送（Real-time Push）到前端。WebSocket 支持双向通信；SSE 主要用于服务器向客户端单向推送。
- **轮询（Polling）**：客户端定期查询任务状态。实现简单，在不需要即时更新时仍然是合理选择。

---

## 2. 现代消息架构（Modern Messaging Architecture）

消息队列（Message Queue）是系统级异步处理的常见基础设施。一个典型的消息系统（Messaging System）包含四个部分：

```text
生产者 Producer → 消息代理 Broker / 队列 Queue → 消费者 Consumer
                     传递消息 Message
```

### 2.1 生产者（Producer）

生产者负责创建消息并发送到消息系统。

- **解耦（Decoupling）**：生产者通常只需要知道消息的格式和发送目的地，不需要直接调用消费者，也不需要了解消费者的部署位置或具体实现。
- **批处理（Batching）**：把多条消息一起发送，减少网络请求，提高吞吐量（Throughput）。
- **压缩（Compression）**：在支持压缩的系统中减少传输和存储开销。
- **权衡（Trade-off）**：批处理可能增加等待时间；生产者仍需遵守消息契约（Message Contract），并处理发送失败。

### 2.2 消息（Message）

消息是包含任务或事件数据的数据包。

- **JSON（JavaScript Object Notation）**：可读性好，工具支持广泛，适合多数业务场景。
- **Protocol Buffers（Protobuf）/ Apache Avro**：采用模式（Schema）定义数据结构，通常更紧凑，适合对性能和数据契约有较高要求的场景。需要管理模式演进（Schema Evolution）。
- **常见元数据（Metadata）**：消息 ID（Message ID）、事件类型（Event Type）、创建时间（Timestamp）、追踪 ID（Trace ID）和模式版本（Schema Version）。

### 2.3 消息代理（Message Broker）

消息代理负责接收、缓冲（Buffering）、存储和路由（Routing）消息，可以把它理解为系统中的“邮局”。

- **持久化（Persistence）**：根据配置把消息保存到持久存储，降低进程或节点故障造成消息丢失的风险。
- **保留策略（Retention Policy）**：消息是否在确认后删除、保留多久，以及能否重新读取，取决于具体产品和配置。
- **托管服务（Managed Services）**：例如 Amazon SQS 和 Confluent Cloud，可以减少自建集群的运维工作（Operations）。

### 2.4 消费者（Consumer）

消费者接收消息并执行对应的业务逻辑。

- **常驻工作进程（Long-running Worker）**：持续拉取（Pull）或订阅消息。实际实现通常使用客户端库、长轮询（Long Polling）或阻塞等待，避免死循环空转。
- **Kubernetes + KEDA（Kubernetes Event-driven Autoscaling）**：根据队列积压量（Queue Backlog）或消费延迟（Consumer Lag），自动调整消费者 Pod 的数量。
- **无服务器计算（Serverless）/ 函数即服务（Function as a Service, FaaS）**：例如 AWS Lambda，通过事件源映射（Event Source Mapping）处理 SQS 消息，按调用次数和执行资源计费。队列等配套服务仍可能收费。
- **确认（Acknowledgement, ACK）**：消费者成功处理后确认消息；失败时通常重试（Retry），超过策略限制后可进入死信队列（Dead-letter Queue, DLQ）。具体确认机制因产品而异。

---

## 3. 常见架构变体（Messaging and Routing Patterns）

消息如何流转，取决于业务模型和消息系统提供的能力。

### 3.1 工作队列（Work Queue）：抢单模式

一个队列连接多个竞争消费者（Competing Consumers）。

- **机制**：一条任务消息通常交给一个消费者执行，多个 Worker 分摊任务。失败重试时，同一消息可能再次交给原消费者或其他消费者，因此仍需处理重复投递（Duplicate Delivery）。
- **场景（Use Cases）**：图片处理（Image Processing）、邮件发送（Email Delivery）、后台任务（Background Jobs）。
- **优势**：实现负载均衡（Load Balancing）和水平扩展（Horizontal Scaling）；任务增加时可以增加 Worker。

### 3.2 发布/订阅（Publish/Subscribe, Pub/Sub）：广播模式

同一个事件可以被多个独立系统处理。

- **机制**：生产者发布到主题（Topic），多个订阅（Subscriptions）或消费者组（Consumer Groups）各自接收或读取事件。
- **实战例子**：电商系统发布“订单完成（Order Completed）”事件：
  - 风控服务（Risk Control Service）：检查是否存在刷单或欺诈。
  - 客户关系管理服务（Customer Relationship Management, CRM）：发送优惠券。
  - 数据分析服务（Analytics Service）：更新销售报表。
- **优势**：新增积分服务（Rewards Service）时，可以增加订阅，无需修改订单系统中逐个调用下游服务的逻辑。
- **注意**：不同系统的分发方式不同。例如 Kafka 通常让不同消费者组独立读取同一日志（Log）；同一组内的消费者分担分区，不会各自收到全部消息。

### 3.3 多租户 Webhook 路由（Multi-tenant Webhook Routing）

软件即服务（Software as a Service, SaaS）平台经常需要把事件推送给不同客户，即租户（Tenant）。

- **挑战**：大客户 A 的消息量很大，可能占满共享资源，使小客户 B 的消息延迟。这称为吵闹邻居问题（Noisy Neighbor Problem）。
- **事件总线（Event Bus）**：例如 Amazon EventBridge，根据事件内容和规则路由到相应目标。
- **租户隔离（Tenant Isolation）**：为不同租户或服务等级（Service Tier）配置独立队列、并发限制（Concurrency Limits）或速率限制（Rate Limits）。
- **可靠性（Reliability）**：配置重试、指数退避（Exponential Backoff）、随机抖动（Jitter）和死信队列。
- **安全（Security）**：发送方生成签名（Signature），接收方验证签名和时间戳，降低伪造请求与重放攻击（Replay Attacks）的风险。

---

## 4. 挑战与解决方案（Challenges and Trade-offs）

异步处理降低服务之间的直接依赖，但也引入新的复杂性。

### 4.1 消息顺序（Message Ordering）

- **问题**：多个消费者并行处理时，消息完成顺序可能与发送顺序不同。例如“发货（Ship Order）”先于“创建订单（Create Order）”完成，可能导致业务错误。
- **局部有序（Ordering per Key）**：按照订单号等业务标识，把相关消息送到同一个有序通道。
  - **Kafka 消息键（Message Key）**：在分区策略与分区数量保持稳定时，同一键通常进入同一分区（Partition）。Kafka 保证分区内的日志顺序；消费者还需避免并行处理打乱业务执行顺序。
  - **Amazon SQS FIFO 消息组 ID（Message Group ID）**：同组消息按序交付，不同组可以并行。标准队列（Standard Queue）不提供同样的严格顺序保证。
- **权衡**：全局有序（Global Ordering）会限制并行能力；通常只要求同一订单或同一账户内有序。

### 4.2 最终一致性（Eventual Consistency）

- **问题**：用户下单成功后，积分服务还没处理事件，因此积分暂时没有更新。不同服务的数据存在传播延迟（Propagation Delay）。
- **业务容忍度**：报表、积分等功能通常可以接受短暂延迟，具体取决于业务要求。
- **乐观 UI 更新（Optimistic UI Update）**：前端可以提前展示预期变化，但应标记为待确认（Pending），并在失败时回滚（Rollback）或重新同步，避免把预计结果当成已完成事实。
- **关键业务**：资金扣减、库存预留等操作需要明确的一致性约束（Consistency Constraints），不能只依赖前端预估。

### 4.3 消息重复与幂等性（Duplicate Messages and Idempotency）

- **至少一次投递（At-least-once Delivery）**：为避免消息丢失，很多消息系统允许重试，因此消费者可能收到重复消息。
- **幂等性（Idempotency）**：同一操作执行多次，产生的业务效果与执行一次相同。
- **危险例子**：每次收到消息都执行 `UPDATE account SET money = money - 10`，重复消息就会导致重复扣款。
- **幂等键（Idempotency Key）**：使用唯一的 `transaction_id` 识别同一笔业务操作，通过唯一约束（Unique Constraint）记录已经处理的操作。

```text
开始数据库事务 BEGIN TRANSACTION
  1. 插入 transaction_id 到已处理记录表（有唯一约束）
  2. 插入成功 → 执行扣款 → 提交事务 COMMIT
  3. ID 已存在 → 跳过扣款，结束重复处理
处理成功后确认消息 ACK
```

**去重记录（Deduplication Record）与扣款必须在同一个数据库事务（Database Transaction）中原子提交**。否则可能出现“记录已写入但没有扣款”或“扣款成功但没有记录”的情况。调用外部支付等服务时，还需使用该服务支持的幂等键或其他协调机制。

### 4.4 可观测性（Observability）

- **问题**：HTTP 请求已经返回，但业务仍在后台执行，单看请求日志无法判断整条流程是否成功。
- **分布式链路追踪（Distributed Tracing）**：通过消息元数据传播追踪上下文（Trace Context），关联生产者、Broker 和消费者上的操作。可使用 OpenTelemetry，以及 Jaeger 或 Datadog 等追踪工具。
- **关键指标（Key Metrics）**：队列深度（Queue Depth）、最老消息年龄（Age of Oldest Message）、消费延迟（Consumer Lag）、处理耗时（Processing Latency）、重试次数（Retry Count）和死信数量（DLQ Message Count）。
- **关联日志（Correlated Logs）**：记录 Trace ID、Message ID 和业务 ID，方便定位某个订单或任务的完整处理过程。

---

## 5. 技术选型指南（Technology Selection Guide）

选型前，先分清消息表达的是什么，以及消费后还要不要保留：

- **任务／命令（Task / Command）**：希望某个处理者执行一件事，例如“发送邮件（Send Email）”。通常由一组 Worker 分摊，成功后结束任务。
- **事件（Event）**：记录已经发生的事实，例如“订单已支付（Order Paid）”。风控、积分、报表等多个系统可能各自需要处理。
- **事件流（Event Stream）**：持续保存一系列事件，让不同系统独立读取，并在需要时重新处理历史数据。

这些是设计需求，不是互斥的产品分类。例如 Pub/Sub 既能广播事件，也能让多个 Worker 分摊任务；RabbitMQ 也支持广播和流式存储。

### 5.1 云托管消息服务（Cloud-managed Messaging Services）

共同点是减少基础设施运维，按使用量计费，支持自动扩展。差别主要在**消息如何分发、谁管理消费进度、能否重放，以及如何连接其他服务**。

#### 5.1.1 Amazon SQS（Simple Queue Service）：任务缓冲与分摊

**模型：生产者 → 队列（Queue）→ 一组 Worker。**

例如用户上传图片后，API 把 `ResizeImage` 任务放入队列，多个图片处理 Worker 从同一个队列取任务。API 不需要等待图片处理完成。

- **竞争消费（Competing Consumers）**：同一个队列中的 Worker 分摊任务，不会给每个 Worker 广播一份。要让邮件、积分、报表三个系统分别收到事件，通常需要各自的队列，再由 SNS（Simple Notification Service）或 EventBridge 等服务分发。
- **可见性超时（Visibility Timeout）**：Worker 取到消息后，消息暂时对其他 Worker 不可见。处理成功后调用 `DeleteMessage`；如果 Worker 崩溃或超时未删除，消息可再次被取走。耗时任务可以延长可见性超时。
- **Standard Queue 与 FIFO Queue**：Standard 提供至少一次投递、尽力排序（Best-effort Ordering）；FIFO 使用 `MessageGroupId` 保证同组按序交付，并提供有范围和时限的发送去重（Deduplication）。应用仍需处理超时重投和业务重复执行。
- **存储与重放**：消息保留受保留期（Retention Period）限制，成功删除后不能像 Kafka 那样通过移动读取位置重放。SQS 主要保存“尚未完成的任务”，不承担长期事件历史库的角色。
- **适合**：邮件发送、图片处理、削峰填谷（Buffering Traffic Spikes），以及上游比下游快时的任务缓冲。

**具体例子**：短时间内上传 10 万张图片，可以先积压到 SQS，再让 Worker 按处理能力消化。队列能缓冲积压，但仍要控制任务等待时间和成本。

#### 5.1.2 Amazon EventBridge：事件筛选与服务集成

**模型：事件源 → 事件总线（Event Bus）→ 匹配规则或订阅 → 不同目标（Targets）。**

例如订单服务发布 `OrderPaid` 事件，消息包含 `order_id`、`amount` 和 `region`。总线可以把高金额订单送到风控服务，把所有已支付订单送到积分服务，把特定地区的订单送到对应业务系统。

- **内容筛选（Content-based Filtering）**：根据事件字段决定发送到哪里，适合多来源、多目标的业务事件集成。
- **目标集成（Target Integration）**：连接 Lambda、SQS、Step Functions 等服务，减少自己编写转发代码的工作。具体可用目标依总线类型和配置而定。
- **与 SQS 配合**：总线决定“谁需要这件事”，每个下游的 SQS 队列负责“缓冲积压并分配给 Worker”。例如 `OrderPaid → EventBridge → Email SQS / Rewards SQS`。
- **失败处理（Failure Handling）**：配置投递重试和死信队列；总线把事件交给目标成功，并不代表目标内部的业务已经完成。例如成功送入 SQS 后，邮件仍需由消费者发送。

**版本区别不能忽略**：当前官方文档区分以下两种自定义总线：

| 总线类型（Bus Type） | 消费与路由模型 | 保留、顺序与重放 |
| --- | --- | --- |
| Custom Event Bus - Classic | 总线所有者创建规则（Rules）和目标（Targets） | 常规投递不保证顺序；需要预先配置归档（Archive）来重放匹配事件，重放也不保证原始顺序 |
| Custom Event Bus | 消费者创建订阅者（Subscribers），各自筛选、转换并投递到目标 | 提供可配置事件保留、从历史位置开始消费，以及有序投递能力；具体保证依配置而定 |

因此，不能笼统地说“EventBridge 不支持保留、重放或顺序”。阅读旧架构案例时，先确认它使用哪种总线。下面的订单分发图使用 Classic 的规则模型：

```text
OrderPaid → EventBridge Classic Event Bus
              ├─ 规则：全部已支付订单 → Email SQS → Email Workers
              ├─ 规则：全部已支付订单 → Rewards SQS → Rewards Workers
              └─ 规则：amount > 1000 → Risk Lambda
```

**适合**：按事件内容分发、AWS 服务事件接入、SaaS 集成，以及多个业务系统响应同一事件。

#### 5.1.3 Google Cloud Pub/Sub：主题广播与独立消费进度

**模型：生产者 → 主题（Topic）→ 多个订阅（Subscriptions）→ 各订阅的消费者。**

最关键的区别是：**不同订阅各自获得事件；同一订阅下的多个消费者分摊消息。**

```text
OrderPaid → orders Topic
              ├─ email Subscription → Email Worker A / B 分摊处理
              ├─ rewards Subscription → Rewards Workers
              └─ analytics Subscription → Analytics Workers
```

- **独立进度（Independent Consumption Progress）**：积分订阅处理慢，不会要求邮件订阅也等着它；每个订阅分别管理积压和确认状态。
- **拉取／推送（Pull / Push）**：Pull 由客户端请求消息或使用 StreamingPull 持续接收；Push 由服务向消费者的 HTTP 端点发送消息。两者的确认方式和可用能力不同。
- **确认期限（Acknowledgement Deadline）**：消费者要及时确认（ACK）或延长期限，否则消息可能再次投递；Push 通常通过成功的 HTTP 响应确认。
- **顺序与重复**：默认不保证顺序，通常按至少一次投递设计。可以启用基于排序键（Ordering Key）的消息排序，但要满足发布区域等要求。Exactly-once Delivery 仅适用于支持的 Pull 订阅，并有区域范围等条件；不能据此假定外部数据库写入或支付也恰好执行一次。
- **保留与重放（Retention and Replay）**：可以结合消息保留、快照（Snapshot）和 `Seek` 重新投递历史消息。能重放的范围取决于 Topic／Subscription 的保留设置，已确认消息也要有相应保留配置。
- **适合**：GCP 内的业务事件广播、多个服务独立消费，以及与 Dataflow、BigQuery 等构建数据管道。

#### 5.1.4 三者如何选择？

| 比较维度（Dimension） | Amazon SQS | Amazon EventBridge | Google Cloud Pub/Sub |
| --- | --- | --- | --- |
| 主要抽象 | Queue：等待处理的任务 | Event Bus：筛选、路由和集成 | Topic + Subscription：广播与独立消费 |
| 多个 Worker 分摊任务 | 同一队列竞争消费 | 通常交给 SQS 或目标服务分摊 | 同一订阅下竞争消费 |
| 多个业务服务各收一份 | 需要多个队列和分发层 | 多条规则或多个订阅者匹配同一事件 | 每个业务服务创建独立订阅 |
| 成功消费后 | 显式删除任务消息 | 取决于总线类型、保留／归档设置 | 确认当前订阅的消息；历史重放依保留设置 |
| 顺序控制 | FIFO 按 Message Group 排序 | Classic 不保证；Custom 有有序投递能力 | 启用 Ordering Key，并满足相关条件 |
| 常见用途 | 图片处理、邮件任务、削峰 | 订单事件路由、云服务与 SaaS 集成 | 业务广播、流式数据接入 |
| 主要成本来源 | API 请求，以及相关数据传输等 | 事件接入／投递、保留／归档／重放等，依功能计费 | 消息数据吞吐、存储和数据传输等 |

**例子**：“请发一封邮件”可以放进 SQS；“订单已经支付，多个系统都需要知道”可以用 EventBridge 或 Pub/Sub；AWS 中想先广播再让各服务独立缓冲，也常见 SNS + 多个 SQS。

托管服务之间不应仅按产品名对应。云平台、投递方式、保留需求和集成目标都会影响选择。

### 5.2 事件流平台（Event Streaming Platforms）

#### 5.2.1 Apache Kafka：保留事件历史，让多个系统独立读取

**模型：生产者 → 分区化的主题日志（Partitioned Topic Log）→ 多个消费者组（Consumer Groups）。**

Kafka 的核心是追加日志（Append-only Log）：消息被读过之后，通常仍会保留到配置的时间或容量限制；启用日志压缩（Log Compaction）的主题则可以按键保留最新状态，而不保存每次历史变化。

- **分区（Partition）**：一个 Topic 拆成多个分区以并行读写。顺序保证在单个分区内，不跨所有分区。
- **偏移量（Offset）**：消费者记录自己读到哪里。处理完成后提交消费位置（Offset Commit），不是删除消息。
- **消费者组（Consumer Group）**：不同组独立读取同一 Topic。在传统 Consumer Group 模型下，同一组内每个分区同一时刻由一个消费者负责；例如 6 个分区、10 个消费者，最多 6 个消费者同时获配分区。
- **重放（Replay）**：把消费位置移回去，或创建一个从历史位置开始的新消费者组，重新处理仍在保留范围内的事件。
- **生态（Ecosystem）**：Kafka Connect 连接数据库和其他数据源／目标；Kafka Streams 处理事件流；CDC 工具可以把数据库变化发布到 Kafka。
- **Exactly-once 的边界**：Kafka 事务和流处理工具可以在支持的 Kafka 读写范围内提供相应保证。跨到外部数据库、邮件或支付服务时，仍需考虑幂等和事务边界。

**具体例子**：网站把 `PageViewed`、`ProductClicked`、`OrderPaid` 写入 Kafka。实时风控、销售报表和推荐系统各用一个消费者组独立读取。一个月后新增分析算法，可以重读保留的历史事件，不需要让原业务重新产生它们。

**代价**：需要设计分区键（Partition Key）、分区数量、复制（Replication）、保留期和消费者重平衡（Rebalancing）。即使用托管 Kafka，也仍需管理这些应用层决策。

#### 5.2.2 Redpanda：使用 Kafka API 的另一种事件流实现

Redpanda 同样提供 Topic、Partition、Offset 和 Consumer Group，面向持久化事件流。它实现 Kafka 协议／API 兼容性（Kafka API Compatibility），许多 Kafka 客户端可以继续使用。

| 比较维度（Dimension） | Apache Kafka | Redpanda |
| --- | --- | --- |
| 核心数据模型 | 分区日志、Offset、Consumer Group | 同类模型，提供 Kafka API 兼容性 |
| 服务端实现 | 主要使用 Java / Scala，运行于 JVM | 主要使用 C++，采用 Seastar 框架 |
| 集群协调 | 现代 Kafka 使用 KRaft；旧部署可能使用 ZooKeeper | 使用内置 Raft 协调与复制 |
| 工具与生态 | Kafka Connect、Kafka Streams 和广泛的集成生态 | 可复用许多 Kafka 客户端和工具，提供 rpk、Console 等工具 |
| 部署选择 | 自建或多家厂商的托管服务 | 自建或 Redpanda Cloud |
| 需要核对 | 运维能力、版本与生态依赖 | 所需 API／工具兼容性、版本、授权与功能范围 |

**如何选择**：已有 Kafka 团队和成熟集成时，Kafka 通常更容易延续现有体系。希望保留 Kafka API，并评估另一种部署与性能方案时，可以考虑 Redpanda。不能只凭实现语言判断谁更快、更便宜；应使用实际消息大小、吞吐、保留期和故障场景做评估。

#### 5.2.3 什么时候值得采用事件流平台？

- 多个系统需要独立处理同一份持续产生的数据。
- 需要保留历史事件，支持重新计算、补数据（Backfill）或新增消费者。
- 需要 CDC、实时分析（Real-time Analytics）或流处理（Stream Processing）。

如果需求只是“后台发送邮件，成功后删除任务”，一个普通任务队列往往更直接。Kafka 可以承担异步任务，但传统分区消费模型下，失败任务、重试与跳过消息需要额外设计。

### 5.3 传统消息代理（Traditional Message Brokers）

#### 5.3.1 RabbitMQ：通过交换机把消息送到不同队列

**模型：生产者 → 交换机（Exchange）→ 绑定（Binding）→ 队列（Queue）→ 消费者。**

Exchange 决定路由，Queue 保存待处理消息。一条消息可以路由到多个队列；同一队列中的多个消费者则分摊任务。

| 交换机类型（Exchange Type） | 路由方式 | 例子 |
| --- | --- | --- |
| Direct | Routing Key 与 Binding Key 精确匹配 | `email` 进入邮件队列，`sms` 进入短信队列 |
| Fanout | 广播到所有绑定队列，不看 Routing Key | 订单事件同时进入积分和报表队列 |
| Topic | 按点分隔的 Routing Key 做模式匹配 | `order.*` 匹配 `order.paid`；`order.#` 也可匹配 `order.eu.paid` |
| Headers | 根据消息头（Headers）及匹配规则路由 | 按 `region=eu`、`format=json` 等属性分发 |

Topic Exchange 中，`*` 匹配一个词，`#` 匹配零个或多个词。RabbitMQ 的 Topic Exchange 是一种路由机制，与 Kafka 的 Topic 日志不是同一个概念。

```text
Order Service
  → Topic Exchange，Routing Key = order.paid
      ├─ Binding = order.* → Audit Queue → Audit Workers
      └─ Binding = order.paid → Rewards Queue → Rewards Workers
```

#### 5.3.2 可靠性与消费控制（Reliability and Consumption Control）

- **消费者确认（Consumer ACK）**：成功处理后确认，普通队列中的消息随后被移除。消费者在确认前断开时，未确认消息通常可重新投递。
- **生产者确认（Publisher Confirms）**：生产者确认 Broker 是否已接受发布。它与 Consumer ACK 分属两端，不能互相替代；还要处理消息未路由到任何队列的情况。
- **预取数量（Prefetch Count）**：限制一个消费者持有的未确认消息数，避免某个 Worker 预取太多任务，让其他 Worker 闲置。
- **死信交换机（Dead-letter Exchange, DLX）**：满足死信条件时，把消息重新路由到指定交换机，再进入用于排查或重试的队列。重复 `NACK + requeue` 可能形成快速失败循环，需要设计退避和重试上限。
- **持久化与复制**：队列持久性（Queue Durability）、消息持久性（Message Persistence）、确认机制和复制配置要一起考虑，不能只设置一个 `durable` 就假定不会丢消息。

#### 5.3.3 队列类型与重放（Queue Types and Replay）

| 类型（Type） | 主要用途 | 消费与存储特点 |
| --- | --- | --- |
| Classic Queue | 普通任务队列与灵活路由 | 现代版本中通常不复制；成功 ACK 后移除消息 |
| Quorum Queue | 需要复制和故障容忍的任务队列 | 基于 Raft 复制，成功 ACK 后仍是任务完成语义；复制增加资源开销 |
| Stream / Super Stream | 需要保留和重放的流式数据 | 追加日志，按保留策略保存，通过消费位置读取；Super Stream 支持分区扩展 |

因此，“RabbitMQ 消费后删除消息”主要描述普通队列。RabbitMQ Streams 也能保留和重放，选型时要比较具体的数据模型。

#### 5.3.4 与 SQS、Kafka 的区别

- **RabbitMQ 与 SQS**：都能承担任务队列。SQS 由 AWS 托管，通过 API 和可见性超时管理任务；RabbitMQ 提供 Exchange 路由、Consumer ACK、Prefetch 等控制，适合需要这些机制或部署在自有环境的系统。RabbitMQ 也有托管方案，自建则需管理节点、磁盘、复制和升级。
- **RabbitMQ 普通队列与 Kafka**：前者围绕“消息是否完成处理”，后者围绕“持久事件日志和读取位置”。Kafka 适合历史重放和独立消费者组；RabbitMQ 普通队列适合逐条确认、任务分发和灵活路由。
- **RabbitMQ Streams 与 Kafka**：两者都支持日志保留和重放，应进一步比较分区、客户端、工具生态、运维及已有部署。

**具体例子**：内部通知系统根据 `notification.email`、`notification.sms`、`notification.push` 把任务送到不同队列，各队列使用不同 Worker、Prefetch 和失败处理策略。需要这种路由与消费控制，并且能够承担部署管理时，RabbitMQ 是合理选择。

### 5.4 用同一个订单系统理解选型（One Order System, Different Requirements）

| 业务需求（Requirement） | 常见选择（Common Options） | 选择原因（Reason） |
| --- | --- | --- |
| 支付成功后发邮件，多个 Worker 分摊 | SQS / RabbitMQ Queue / Pub/Sub 单个订阅 | 缓冲任务，确认成功，失败重试 |
| 支付事件让积分、风控、报表各处理一次 | EventBridge / Pub/Sub 多订阅 / RabbitMQ 多队列 | 每个业务服务有独立的事件处理路径 |
| 根据金额、地区等事件字段分发到 AWS 服务 | EventBridge | 内容筛选与目标集成 |
| 保留订单事件，未来重新计算销售报表 | Kafka / Redpanda；也可评估 Pub/Sub 重放或 RabbitMQ Streams | 历史保留和独立读取是核心需求 |
| 内部任务按路由键、消息头精细分发 | RabbitMQ | Exchange 与 Binding 提供明确的路由控制 |

一个系统可以组合使用这些产品。例如 EventBridge 负责业务事件分发，SQS 负责下游任务缓冲，Kafka 保存供分析系统重放的事件历史。每增加一个组件，也会增加成本和维护工作，应由明确需求驱动。

### 5.5 官方参考（Official References）

产品能力会随版本变化，具体保证、配额和计费应以所用版本的文档为准。

- [Amazon SQS Developer Guide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)
- [EventBridge Custom Event Bus](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-custom-bus.html) 与 [Default / Classic Event Bus](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-event-bus.html)
- [Google Cloud Pub/Sub Overview](https://cloud.google.com/pubsub/docs/overview) 与 [Exactly-once Delivery](https://cloud.google.com/pubsub/docs/exactly-once-delivery)
- [Apache Kafka Introduction](https://kafka.apache.org/intro)
- [Introduction to Redpanda](https://docs.redpanda.com/current/get-started/intro-to-events/)
- [RabbitMQ Topic Exchange Tutorial](https://www.rabbitmq.com/tutorials/tutorial-five-python) 与 [RabbitMQ Streams](https://www.rabbitmq.com/docs/streams)
