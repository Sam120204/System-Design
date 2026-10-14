# System Design

系统设计学习笔记与案例，覆盖 Web 应用基础、数据库扩展、异步消息架构、数据密集型系统和 Kubernetes。内容以中文解释为主，保留英文技术术语，方便学习、复习和系统设计面试。

## 目录结构（Repository Structure）

```text
System-Design/
├── README.md
├── LICENSE
├── web development/     # Web 基础、扩容、数据库与术语
│   └── images/          # Web 笔记引用的架构图
├── Asychronous/         # 异步处理与消息系统
├── 数据密集系统/          # 架构权衡、数据模型与系统设计案例
│   └── assets/          # 系统设计案例的架构图
└── Kubernetes/          # 容器编排与部署实践
```

## Web 开发与扩展（Web Development and Scalability）

[Web Development 阅读指南](<web development/README.md>) 按请求处理、服务扩容、数据库扩展和延迟优化组织学习顺序。

| 笔记（Note） | 内容（Topics） |
| --- | --- |
| [Web 应用基础（Web Application Basics）](<web development/overview.md>) | Request / Response、路由、模板、MVC、REST API |
| [Web 可扩展性（Scalability）](<web development/scalability.md>) | 单机瓶颈、垂直与水平扩展、服务隔离、无状态服务 |
| [云负载均衡（Cloud Load Balancing）](<web development/云负载均衡详解.md>) | 后端发现、健康检查、流量分配、L4 / L7 |
| [Backend Numbers Everyone Should Know](<web development/Backend Numbers Everyone Should Know.md>) | 延迟量级、IOPS / QPS、缓存与容量估算 |
| [关系型数据库（Relational Databases）](<web development/关系型数据库.md>) | MySQL 复制、故障切换、容灾、数据分区 |
| [数据复制和分区（Replication and Partitioning）](<web development/数据复制和分区.md>) | 分片与复制的区别，以及两者如何组合 |
| [Database Partition / Sharding](<web development/database-sharding.md>) | 分片时机、分片键选择和设计代价 |
| [延迟优化案例（Latency Optimization）](<web development/延迟优化案例.md>) | 社交时间线、缓存、消息队列与异步 Fan-out |
| [System Design 术语速记（Terminology）](<web development/terminology.md>) | 常用中英文术语及一句话解释 |

## 异步处理与消息系统（Asynchronous Processing and Messaging）

| 笔记（Note） | 内容（Topics） |
| --- | --- |
| [基础知识：异步处理（Asynchronous Processing）](Asychronous/asynchronous-processing.md) | 同步与异步、Producer / Broker / Consumer、Work Queue、Pub/Sub、顺序、幂等与可观测性；SQS、EventBridge、Google Cloud Pub/Sub、Kafka、Redpanda 和 RabbitMQ 的具体比较 |

## 数据密集型系统与设计案例（Data-intensive Systems and Design Cases）

| 笔记（Note） | 内容（Topics） |
| --- | --- |
| [Design Is All About Trade-offs](数据密集系统/design-is-all-about-trade-off.md) | 从业务需求出发理解设计权衡 |
| [Web 应用前端架构（Web Application Frontend Architecture）](数据密集系统/Web应用前端架构-状态管理与扩展组件.md) | HTTP Session、会话状态、文件存储与扩展组件 |
| [NoSQL：Transaction 与数据库选型](数据密集系统/NoSQL.md) | 访问模式、事务边界与数据库选择 |
| [Design Instagram：User Generated Content](<数据密集系统/User Generated Content case (ins).md>) | 图片上传、Feed、Pull / Push / Hybrid、缓存 |
| [Design TikTok](<数据密集系统/案例分析 - Design TikTok.md>) | 视频上传、异步转码、播放切片、对象存储与 CDN |
| [Design a News Aggregation System](<数据密集系统/design news aggregator system.md>) | 新闻发现、近重复检测、Story Clustering、个性化 Feed、排序、分页与故障处理 |

## Kubernetes 与部署（Kubernetes and Deployment）

| 笔记（Note） | 内容（Topics） |
| --- | --- |
| [Kubernetes Overview](<Kubernetes/K8s overview.md>) | 容器编排、kubeadm、minikube、kubectl 与集群架构 |
| [Pod & ReplicaSet](Kubernetes/Pod与ReplicaSet.md) | Pod 管理、副本控制与相关命令 |
| [Deployment](Kubernetes/deployment.md) | 声明式部署、版本升级与回滚 |
| [Service & Namespace](Kubernetes/Service与Namespace.md) | 服务访问、命名空间与资源配额 |

## 建议阅读顺序（Suggested Reading Order）

1. 从 [Web 应用基础](<web development/overview.md>) 开始，理解一个请求如何经过应用和数据库。
2. 阅读扩容、负载均衡、容量估算、复制和分片，建立性能与可用性的判断依据。
3. 阅读 [异步处理](Asychronous/asynchronous-processing.md)，理解队列、事件分发、消息重试和一致性。
4. 用 Instagram、TikTok 和新闻聚合案例练习需求分析、架构设计与取舍。
5. 阅读 Kubernetes 笔记，理解服务如何部署、扩展和更新。

面试或写设计文档时，可以随时查阅 [术语速记](<web development/terminology.md>)。案例中的容量数字是练习假设；产品行为和命令应结合所用版本的官方文档核对。

## License

本仓库采用 [MIT License](LICENSE)。
