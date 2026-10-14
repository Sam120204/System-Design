# Web Development 学习笔记

这组笔记从 Web 请求如何工作，逐步讲到扩容、数据库和系统设计案例。按下面的顺序读即可；每篇也可以单独查阅。

| 顺序 | 笔记 | 主要内容 |
| --- | --- | --- |
| 1 | [Web 应用基础](overview.md) | 请求与响应、路由、模板、数据库、MVC、REST API |
| 2 | [Web 可扩展性](scalability.md) | 单机瓶颈、垂直和水平扩展、无状态服务 |
| 3 | [云负载均衡详解](云负载均衡详解.md) | 后端发现、健康检查、流量分配，以及 L4/L7 的区别 |
| 4 | [Backend Numbers Everyone Should Know](<Backend Numbers Everyone Should Know.md>) | 延迟量级、IOPS/QPS、缓存与服务器容量的粗估 |
| 5 | [关系型数据库](关系型数据库.md) | MySQL 复制、故障处理、数据分区的详细笔记 |
| 6 | [数据复制和分区](数据复制和分区.md) | 用图对比分片与复制，以及两者如何组合 |
| 7 | [Database Partition / Sharding](database-sharding.md) | 何时分片、如何选分片键、分片的代价 |
| 8 | [延迟优化案例](延迟优化案例.md) | 社交时间线的缓存、消息队列和异步 fan-out |

`关系型数据库.md` 记录了更多实现细节；`数据复制和分区.md` 是概念对比；`database-sharding.md` 侧重设计时的判断。这三篇有意从不同角度讲同一主题。

面试和写 design doc 时，可以查阅 [System Design 术语速记](terminology.md)：常用英文、中文和一句话解释，方便对照背诵。

`images/` 存放笔记引用的架构图。目录保持平铺，便于直接找到对应主题；图片统一放在 `images/`，避免与正文混在一起。
