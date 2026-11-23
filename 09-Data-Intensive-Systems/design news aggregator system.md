# Design a News Aggregation System

设计一个类似 Google News 的新闻聚合系统。系统从大量媒体发现文章，把针对同一事件的报道组织到一起，并提供新鲜的个性化信息流。用户可以关注来源、分类和话题，也可以隐藏来源或要求看到更多/更少类似内容。

## 功能需求

1. Users should be able to subscribe to news sources, categories, and keyword topics to personalize their feed.
2. Users should be able to view a personalized feed that updates in near real time with new and relevant stories.
3. Users should be able to manage preferences.

## 非功能需求

- Article-to-feed eligibility: P95 under 2 minutes after discovery.
- First feed page: P99 under 300 ms.
- Feed-read availability: 99.9%.
- The system must tolerate at-least-once delivery, skewed publishers, breaking-news spikes, and regional failures.

## High-level Diagram

```text
Source Registry/Scheduler -> Fetch -> Durable Ingest
 -> Normalize/Version/Dedup -> Article Store
 -> Story Cluster -> Interest Timeline / Index

User/Follow/Profile -> Feed API
 -> bounded timeline selection/merge
 -> cluster dedup/rank/diversity/hard filter
 -> Feed Session / First-page Cache
```

## Deep Dive

### 采集流水线

#### 1 Source Registry 与自适应发现

Registry 控制哪些来源可抓取以及如何抓取：

- Atom/RSS URL、News Sitemap URL 或 Crawl Seed。
- 允许的 Domain 和 Redirect Policy。
- 估计发布速率。
- 上次成功发现和文章抓取。
- ETag/Last-Modified 状态。
- 按域名的请求预算和退避。

分片 Scheduler 按 `next_fetch_at` 排序任务。Worker 租赁到期任务，使用按域名 Token Bucket，并自适应频率：

- 来源正在发布时提高轮询频率。
- 长时间无变化时降低频率。
- 遇到错误、429 或 5xx 时退避。
- 优先已知高频突发新闻来源，但不饿死长尾来源。

Atom 是标准化的新闻和 Web 内容分发格式。News Sitemap 和 Article Structured Data 可以暴露 URL、标题、作者和发布时间/更新时间。没有 Feed 的来源由 Crawler 发现和抓取已批准页面。

#### 2 近重复检测

近重复包括转载副本、Mirror、Print/Mobile 版本和抓取复制。

一种实用流水线：

1. 按语言、时间窗口、实体和粗粒度 Content Fingerprint 筛选候选。
2. 在候选块内比较 SimHash/MinHash 或 Embedding 相似度。
3. 把高置信副本标记进 Duplicate Group。
4. 选择一个 Canonical Presentation，同时保留来源记录和 Provenance。

这样避免 All-Pairs 比较。阈值需要在误合并与重复噪音之间取舍。

#### 3 Story Clustering

Story Clustering 用于组织同一事件的独立报道。

对每个新的 Canonical Article：

1. 使用语言、实体/话题、位置和时间召回近期 Cluster 候选。
2. 结合标题/正文 Embedding 相似度和实体/时间/位置一致性打分。
3. 高于阈值则加入最佳 Cluster，否则创建新 Cluster。
4. 更新 Cluster Feature 并发布 `StoryClusterUpdated`。
5. 异步 Reconciliation 可以合并/拆分 Cluster 并重新分配成员。

Online Path 优先速度，Reconciliation Path 优先质量。所有成员变更都要版本化，以便 Feed Cache 和 Index 按 Story/Version 失效。

### Feed 生成

#### 1 为什么不做完整 Fanout-on-Write？

“Technology”之类宽泛话题或全球媒体可能拥有数千万关注者。把每篇文章写入所有关注者 Inbox 会产生：

- 巨大写放大。
- 昂贵的删除/更正 Fanout。
- 突发新闻期间的 Hot Key 和 Queue 峰值。
- 对非活跃用户的大量无效工作。

#### 2 为什么不做纯 Fanout-on-Read？

每次请求都读取并合并数百条无界 Timeline，会使延迟和成本不可预测。

#### 3 混合策略

1. 为来源、话题、分类、位置和 Regional Headlines 预计算有界 Timeline。
2. 读取用户显式关注、强制屏蔽、语言/区域和紧凑 Interest Profile。
3. 选择有界的相关 Timeline 子集。
4. 并行读取每条 Timeline 的 Top K 候选。
5. 增加有界的 Collaborative/Trending 和 Exploration 候选。
6. 按 `story_id` 合并、过滤，保留约 500–2,000 个候选。
7. 先进行轻量排序，再对 Top Subset 使用更重模型。
8. 执行 Policy 和 Diversity Re-ranking。
9. 保存短期 Feed Session Snapshot 或首页 Cache。

对于非常活跃用户或窄兴趣，可以把新 Story ID 推入小型用户缓存作为优化；系统正确性不依赖全量用户写扩散。

#### 4 候选来源

- **显式关注：** 最强的来源/分类/话题匹配。
- **Content Based：** 与衰减用户画像相似的实体/话题。
- **Collaborative：** 与用户交互相似的用户/Cohort。
- **Regional Headlines：** 重要公共新闻和冷启动降级。
- **Trending：** 经过滥用过滤的近期高 Prominence 内容。
- **Exploration：** 受控比例的陌生来源/话题。

每个 Generator 都有配额和 Timeout。单个慢 Generator 不能耗尽请求预算。

#### 5 排序与重排序

示意性一阶段分数：

`score = explicit_match + profile_relevance + freshness_decay + prominence + source_quality + locality + predicted_value - negative_feedback`

具体模型不如 Feature Freshness 和约束重要。

强制过滤：

- 隐藏来源或话题。
- 不符合语言/区域策略。
- 已删除/撤稿/无资格文章。
- 已曝光 Story（取决于产品行为）。

重排序约束：

- 每个 Story Cluster 最多一张卡片。
- 限制同一来源连续或总计出现数量。
- 话题和来源多样性。
- 为重要区域/公共新闻预留最低比例。
- 避免同一持续事件的细微更新大量占据 Feed。
- 在受控预算内插入 Exploration。

互动只是一个信号，不是唯一目标。应优化包含新鲜度、有用互动、来源/Story 多样性、隐藏/投诉率和长期留存的多目标指标。

#### 6 稳定分页

由于排序持续变化，Offset Pagination 不安全。

第一次请求时：

- 创建短期 `feed_session_id`。
- 保存 Top N 排序 Story ID，或确定性 Snapshot Seed/Time。
- 返回首页和不透明 Cursor，其中包含 Session ID、位置、Profile Version 和过期时间。

后续分页时：

- 从 Session Snapshot 继续。
- 重新应用强制删除和来源屏蔽。
- 跳过不存在/撤稿内容，并从保留候选中补位。

刷新创建新 Session，允许新 Story 出现在顶部。这样可避免滚动过程中的重复和漏项。

### 缓存

- 按 Entity 和 Region 缓存 Source/Topic Timeline。
- 按 Story Version 缓存热门 Story Card 和 Full Coverage Membership。
- 使用 `profile_version` 缓存紧凑用户画像/关注。
- 缓存首页或 Feed Session 数十秒到数分钟。
- 把强制屏蔽/撤稿检查放在接近读取路径的位置，使安全变更无需等待普通 TTL。

Cache Key 应包含相关 Language、Region、Policy/Model Version 和 Profile Version。对 Hot Key 使用 Request Coalescing 和带随机扰动的 TTL。

图片可以通过具备版权和安全检查的 Proxy/CDN 提供，或按媒体策略链接。不要让客户端直接抓取 Crawler 提供的任意未验证 URL。

### 故障处理

#### 抓取与解析故障

- 指数退避并加入 Jitter。
- 按来源 Circuit Breaker。
- 有界重试预算后进入 Dead Letter。
- 监控 Source Freshness Lag，而不仅是 Worker Error。
- 保留 Raw Fetch 供 Parser 重放。

#### 排序/特征故障

降级顺序：

1. 已缓存个性化首页。
2. 使用简单新鲜度排序的显式关注 Timeline。
3. 区域/语言 Headlines。
4. Last Known Safe Page。

API 内部应记录 Fallback Metric，不能永久静默降级。

#### 突发新闻热点

- 避免按用户写扩散。
- 缓存热门 Story Card 和代表文章。
- 批量 Membership Update，并发布 Cluster Version Delta。
- 分片高频 Counter，合并近似 Aggregate。
- 限制冗余媒体抓取，但不延迟真正的新来源。
- 每个 Candidate Generator 使用配额，避免一个事件吞噬全部 Feed。
- 对热门 Full Coverage 页面使用 stale-while-revalidate。
