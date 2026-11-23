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

关于 Kafka 究竟是 Queue 还是 Pub/Sub，以及 Event／Command 与工作队列／广播／持久日志的组合，见 [异步系统设计复习](messaging-system-design-review.md)。

### 3.1 工作队列（Work Queue）：抢单模式

一个队列连接多个竞争消费者（Competing Consumers）。

- **机制**：一条任务消息通常交给一个消费者执行，多个 Worker 分摊任务。失败重试时，同一消息可能再次交给原消费者或其他消费者，因此仍需处理重复投递（Duplicate Delivery）。
- **场景（Use Cases）**：图片处理（Image Processing）、邮件发送（Email Delivery）、后台任务（Background Jobs）。
- **优势**：实现负载均衡（Load Balancing）和水平扩展（Horizontal Scaling）；任务增加时可以增加 Worker。

完整案例见 [设计任务调度系统](design-job-scheduler.md)：任务提交、Worker 领取、状态追踪、重试，以及 Cron 和 DAG 扩展。

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

完整案例见 [设计多租户 Webhook 系统](design-webhook.md)：从端点注册、事件路由到投递重试、分区顺序和安全设计。

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

### 5.2 RabbitMQ, RocketMQ, and Kafka: the simple picture

Think of a message broker as a parcel station: a sender drops off a parcel, and a receiver picks it up later. In software, a **producer** sends a message to the broker, and a **consumer** processes it. This lets a request finish sooner (**asynchronous work**), keeps services from calling every downstream service directly (**decoupling**), and buffers sudden traffic spikes. Accepting a message is only the start: the system must still handle delivery, processing failures, and retries.

| Product | Easy way to remember it | Best fit |
| --- | --- | --- |
| **RabbitMQ** | A careful parcel sorter | Flexible routing and ordinary background tasks |
| **RocketMQ** | An order-processing specialist | Business workflows that need delayed, ordered, or transactional messages |
| **Kafka** | A high-volume event log | Large, continuous data streams that several systems read independently |

#### RabbitMQ: route each message to the right queue

RabbitMQ uses an **exchange** to decide which queue receives a message. For example, `notification.email` can go to email workers, `notification.sms` to SMS workers, and an order event can be sent to several queues. Workers process messages and acknowledge success. In an ordinary queue, an acknowledged message is removed; failed work can be retried. RabbitMQ Streams are a separate option that can retain and replay messages.

Choose RabbitMQ when routing rules or task handling matter more than storing a long event history. For example, an internal notification service may have a separate queue and worker pool for email, SMS, and push notifications.

#### RocketMQ: support common order and payment workflows

RocketMQ offers several features that business systems often need:

- **Delayed messages:** check whether an order is still unpaid after 30 minutes, then cancel it *only if it is still unpaid*.
- **Ordered messages:** process events for the same order in the right sequence, such as created → paid → shipped. The application must route that order consistently; this does not create a global order across all messages.
- **Transactional messages:** help coordinate a local database transaction with publishing a message. They do not automatically make two services' databases one atomic transaction.
- **Retries:** try again when a consumer fails, with a plan for messages that repeatedly fail.

RocketMQ is a strong candidate for a busy order or payment system when these features match real requirements. Consumers still need **idempotency**: processing the same message again must not charge or update an order twice.

#### Kafka: keep an event history for independent readers

Kafka stores events in a log for a configured retention period. Reading an event does not remove it. A risk system, a reporting system, and a recommendation system can each read the same stream at their own pace. They can also replay events that are still retained.

Four terms are enough to start:

- **Topic:** a named stream, such as `user-activity`.
- **Partition:** one ordered slice of that stream. More partitions allow more parallel work; ordering is within a partition, not across the whole topic.
- **Consumer group:** one application's readers. Different groups read the same topic independently; workers within a group share the partitions.
- **Offset:** a group's reading position in a partition. Moving it back can replay retained events.

For example, a website writes `PageViewed` and `ProductClicked` events to Kafka. Analytics and recommendations read them in separate consumer groups. A new analysis job can later read retained events without asking the website to produce them again. Kafka is especially useful for logs, user activity, real-time analysis, and data pipelines. For a simple “send one email, then remove the task” workflow, retries and per-task completion are usually easier with an ordinary queue.

### 5.3 Compare the tradeoffs

| Question | RabbitMQ | RocketMQ | Kafka |
| --- | --- | --- | --- |
| What stands out? | Flexible routing and task handling | Business-oriented message features | High-volume event streams and independent readers |
| What happens after consumption? | Ordinary queues remove acknowledged messages; Streams can retain them | Messages can remain available for replay within retention limits | Events remain until the retention policy removes them |
| How is order handled? | Depends on queue and consumer setup | Can preserve order for a business key, such as one order ID | Order is guaranteed within a partition |
| What extra work matters? | Plan routing, acknowledgements, retries, and queue durability | Plan retries, idempotency, and business consistency | Plan partitions, retention, consumer progress, and retries |

You may hear that Kafka handles “millions per second,” RocketMQ “hundreds of thousands,” RabbitMQ “tens of thousands,” or that RabbitMQ always has “microsecond latency.” Treat these as rough impressions, **not performance guarantees**. Message size, hardware, batching, replication, persistence, and consumer work change the result. Measure end-to-end latency and throughput using your own workload.

All three can use durable storage and replicas to reduce message loss. No product offers unconditional “zero data loss”: producer acknowledgements, disk writes, replication, consumer acknowledgements, and failure timing all matter. A retry may also deliver a message again, so consumers must handle duplicates safely.

### 5.4 A practical choice

- **Choose Kafka** when you collect large, continuous streams of logs or user events, need several independent readers, or need to replay retained history.
- **Choose RocketMQ** when an order, payment, or fulfillment system genuinely needs delayed checks, per-order sequencing, or transaction-aware publishing.
- **Choose RabbitMQ** when you need flexible routing or a straightforward background task queue, such as separate email and SMS workers.

Before committing, write down the expected peak message rate and size, acceptable delay, replay window, ordering scope, retry behavior, and failure conditions. Then test with realistic traffic. A system may use more than one product, but each additional broker adds operational cost.

### 5.5 Official references

- [Apache Kafka Introduction](https://kafka.apache.org/intro)
- [Apache RocketMQ Documentation](https://rocketmq.apache.org/docs/)
- [RabbitMQ Documentation](https://www.rabbitmq.com/docs)
