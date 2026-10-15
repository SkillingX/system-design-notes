# 第 21 章：设计广告点击事件聚合系统

## 引言
随着 Facebook、YouTube、TikTok 等平台的发展，**数字广告**已成为一个庞大的行业。

因此，跟踪广告点击事件非常重要。本章探讨如何设计一个达到 Facebook/Google 规模的**广告点击事件聚合系统**。

数字广告有一个称为<strong>实时竞价（RTB）</strong>的流程，用于买卖数字广告库存：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/digital-advertising-example.png" alt="digital-advertising-example" width="500" />
</div>

RTB 的速度很重要，因为交易通常需要在一秒内完成。数据准确性也很重要，因为它会影响广告主支付的费用。

广告主可以根据广告点击事件聚合结果，调整目标受众和关键词等。

---

## 第 1 步：理解问题并明确设计范围
- <strong>候选人：</strong>输入数据是什么格式？
- <strong>面试官：</strong>每天 10 亿次广告点击，总共 200 万个广告。广告点击事件数量每年增长 30%。
- <strong>候选人：</strong>系统需要支持哪些重要查询？
- <strong>面试官：</strong>需要重点考虑：
  - 返回广告 X 在过去 Y 分钟内的点击事件数。
  - 返回过去 1 分钟内点击量最高的 100 个广告。两个参数都应可配置，每分钟聚合一次。
  - 支持根据 `ip`、`user_id`、`country` 过滤上述查询结果。
- <strong>候选人：</strong>需要考虑哪些边界情况？我能想到：
  - 有些事件可能比预期晚到。
  - 可能会出现重复事件。
  - 系统不同部分可能宕机，因此需要考虑系统恢复。
- <strong>面试官：</strong>这些考虑得很好，请纳入设计。
- <strong>候选人：</strong>延迟有什么要求？
- <strong>面试官：</strong>广告点击聚合的端到端延迟可以是几分钟。RTB 的延迟要求低于一秒。广告点击聚合通常用于计费和报表，几分钟的延迟可以接受。

### **功能需求**
- 聚合 `ad_id` 在过去 Y 分钟内的点击数。
- 每分钟返回点击量最高的 100 个 `ad_id`。
- 支持根据不同属性过滤聚合数据。
- 数据集规模达到 Facebook 或 Google 级别。

### **非功能需求**
- 聚合结果用于 RTB 和广告计费，因此准确性很重要。
- 妥善处理延迟事件或重复事件。
- 健壮性：系统应能承受部分故障。
- 延迟：端到端延迟最多为几分钟。

### **数量级估算**
- 10 亿 DAU。
- 假设每位用户每天点击 1 个广告，则每天有 10 亿次广告点击。
- 广告点击 QPS = 10,000。
- 峰值 QPS 是平均值的 5 倍，即 50,000。
- 单次广告点击占用 0.1 KB 存储空间，每日存储需求为 100 GB。
- 每月存储需求为 3 TB。

---

## 第 2 步：提出高层设计并达成共识
本节讨论查询 API、数据模型和高层设计。

### **查询 API 设计**

API 是客户端与服务器之间的契约。本例中的客户端是仪表板用户，例如数据科学家、分析师和广告主。

功能需求如下：
- 聚合 `ad_id` 在过去 Y 分钟内的点击数。
- 返回过去 M 分钟内点击量最高的前 N 个 `ad_id`。
- 支持根据不同属性过滤聚合结果。

需要两个端点来满足这些需求。其中一个端点可以通过查询参数支持过滤。

**查询过去 M 分钟内某个 `ad_id` 的点击总数：**

```
GET /v1/ads/{:ad_id}/aggregated_count
```

查询参数：
- `from`：开始分钟，默认值为当前时间减 1 分钟。
- `to`：结束分钟，默认值为当前时间。
- `filter`：不同过滤策略的标识符，例如 `001` 表示“非美国点击”。

响应：
- `ad_id`：广告标识符。
- `count`：起始时间到结束时间之间的聚合点击数。

**查询过去 M 分钟内点击量最高的前 N 个 `ad_id`：**

```
GET /v1/ads/popular_ads
```

查询参数：
- `count`：点击量最高的广告数量 N。
- `window`：聚合窗口大小，单位为分钟。
- `filter`：不同过滤策略的标识符。

响应：
- `ad_id` 列表。

### **数据模型**

系统中既有原始数据，也有聚合数据。

原始数据示例：

```
[AdClickEvent] ad001, 2021-01-01 00:00:01, user 1, 207.148.22.22, USA
```

结构化数据示例：

| ad_id | click_timestamp | user | ip | country |
|---|---|---|---|---|
| ad001 | 2021-01-01 00:00:01 | user1 | 207.148.22.22 | USA |
| ad001 | 2021-01-01 00:00:02 | user1 | 207.148.22.22 | USA |
| ad002 | 2021-01-01 00:00:02 | user2 | 209.153.56.11 | USA |

聚合数据示例：

| ad_id | click_minute | filter_id | count |
|---|---|---|---|
| ad001 | 202101010000 | 0012 | 2 |
| ad001 | 202101010000 | 0023 | 3 |
| ad001 | 202101010001 | 0012 | 1 |
| ad001 | 202101010001 | 0023 | 6 |

`filter_id` 用于满足过滤需求。

| filter_id | region | IP | user_id |
|---|---|---|---|
| 0012 | US | * | * |
| 0013 | * | 123.1.2.3 | * |

为了快速返回过去 M 分钟内点击量最高的前 N 个广告，我们还会维护以下结构：

| 字段 | 类型 | 说明 |
|---|---|---|
| `window_size` | integer | 聚合窗口大小 M，单位为分钟 |
| `update_time_minute` | timestamp | 最近一次更新的时间戳，精确到分钟 |
| `most_clicked_ads` | array | JSON 格式的广告 ID 列表 |

保存原始数据和保存聚合数据，各有什么优缺点？
- 原始数据可以充分利用完整数据集，支持数据过滤和重新计算。
- 聚合数据量较小，查询速度较快。
- 原始数据会占用更多存储空间，查询速度较慢。
- 聚合数据是衍生数据，因此会损失一部分信息。

在本设计中，我们会结合使用两种方式：
- 建议保留原始数据以便调试。聚合逻辑出现 bug 时，可以定位问题并回填数据。
- 也需要保存聚合数据，以获得更快的查询性能。
- 可以将原始数据存入冷存储，降低额外的存储成本。

选择数据库时，需要考虑以下因素：
- 数据是什么类型：关系型、文档型还是 Blob？
- 工作负载是读密集、写密集，还是两者兼有？
- 是否需要事务？
- 查询是否依赖 SUM、COUNT 等 OLAP 函数？

原始数据的平均 QPS 为 10,000，峰值 QPS 为 50,000，因此系统以写入为主。读取量则较低，因为原始数据大多在出现问题时才用作备份。

关系型数据库可以完成这项工作，但扩展写入能力可能比较困难。也可以使用 Cassandra 或 InfluxDB，它们对高写入负载有更好的原生支持。

另一个方案是使用 Amazon S3，并采用 ORC、Parquet 或 AVRO 等列式数据格式。由于这种技术栈不太熟悉，我们选择 Cassandra。

聚合数据需要同时支持大量读取和大量写入：仪表板和告警会不断查询聚合数据，而聚合服务每分钟都会写入新的聚合结果。因此这里也使用同一种数据存储 Cassandra。

### **高层设计**

系统架构如下：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/high-level-design-1.png" alt="high-level-design-1" width="500" />
</div>

输入和输出数据流都是无界流。

为了避免使用同步数据接收端，导致消费者崩溃时整个系统停滞，我们将通过消息队列（Kafka）进行异步处理，解耦消费者和生产者。

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/high-level-design-2.png" alt="high-level-design-2" width="500" />
</div>

第一个消息队列保存广告点击事件：

| ad_id | click_timestamp | user_id | ip | country |
|---|---|---|---|---|

第二个消息队列保存按分钟聚合的广告点击数：

| ad_id | click_minute | count |
|---|---|---|

它还保存每分钟聚合的热门广告 Top N 结果：

| update_time_minute | most_clicked_ads |
|---|---|

第二个消息队列用于实现端到端的恰好一次原子提交语义：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/atomic-commit.png" alt="atomic-commit" width="500" />
</div>

聚合服务可以使用 MapReduce 框架：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/ad-count-map-reduce.png" alt="ad-count-map-reduce" width="500" />
</div>

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/top-100-map-reduce.png" alt="top-100-map-reduce" width="500" />
</div>

每个节点只负责一项任务，并将处理结果发送给下游节点。

Map 节点负责从数据源读取数据，然后过滤并转换数据。

例如，Map 节点可以根据 `ad_id` 将数据分发到不同的聚合节点：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/map-node.png" alt="map-node" width="500" />
</div>

另一种方案是将广告分发到不同的 Kafka 分区，再让聚合节点在消费者组中直接订阅分区。不过，Map 节点可以在后续处理之前清理或转换数据。

另外，我们可能无法控制数据的生产方式，因此同一个 `ad_id` 相关的事件可能被发送到不同分区。

Aggregate 节点在内存中按分钟统计每个 `ad_id` 的广告点击事件。

Reduce 节点收集 Aggregate 节点的聚合结果，并生成最终结果：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/reduce-node.png" alt="reduce-node" width="500" />
</div>

这种 DAG 模型采用 MapReduce 范式，将大数据通过并行分布式计算转换为规模较小的数据。

在 DAG 模型中，中间数据保存在内存中，不同节点通过 TCP 或共享内存通信。

下面看看这种模型如何支持不同使用场景。

**场景 1：聚合点击总数**

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/use-case-1.png" alt="use-case-1" width="500" />
</div>

- 使用 `ad_id % 3` 对广告进行分区。

**场景 2：返回点击量最高的前 N 个广告**

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/use-case-2.png" alt="use-case-2" width="500" />
</div>

- 本例聚合点击量最高的 3 个广告，但可以轻松扩展到前 N 个广告。
- 每个节点维护一个堆数据结构，以便快速查找点击量最高的前 N 个广告。

**场景 3：过滤数据**
为了支持快速过滤数据，可以预先定义过滤条件，并根据条件预先聚合：

| ad_id | click_minute | country | count |
|---|---|---|---|
| ad001 | 202101010001 | USA | 100 |
| ad001 | 202101010001 | GPB | 200 |
| ad001 | 202101010001 | others | 3000 |
| ad002 | 202101010001 | USA | 10 |
| ad002 | 202101010001 | GPB | 25 |
| ad002 | 202101010001 | others | 12 |

这种技术称为**星型模型（star schema）**，广泛用于数据仓库。用于过滤的字段称为**维度**。

该方法的优点：
- 易于理解和构建。
- 可以复用当前聚合服务，在星型模型中添加更多维度。
- 由于结果已提前计算，可以快速按过滤条件访问数据。

该方法的局限是会产生更多数据桶和记录，尤其是在有很多过滤条件时。

---

## 第 3 步：深入设计

深入讨论几个较有意思的话题。

### **流处理与批处理**

前面提出的高层架构属于流处理系统。以下比较三种系统：

| 比较项 | 服务（在线系统） | 批处理系统（离线系统） | 流处理系统（近实时系统） |
|---|---|---|---|
| 响应性 | 快速响应客户端 | 无需响应客户端 | 无需响应客户端 |
| 输入 | 用户请求 | 有界、有限大小的输入，通常数据量很大 | 无界输入（无限数据流） |
| 输出 | 对客户端的响应 | 物化视图、聚合指标等 | 物化视图、聚合指标等 |
| 性能衡量 | 可用性、延迟 | 吞吐量 | 吞吐量、延迟 |
| 示例 | 在线购物 | MapReduce | Flink [13] |

本设计结合了批处理和流处理：

我们使用流处理，在数据到达时进行处理，并近实时生成聚合结果。另一方面，我们通过批处理备份历史数据。

同时包含批处理路径和流处理路径的架构称为 Lambda 架构。它的缺点是需要维护两条处理路径和两套代码。

Kappa 是一种替代架构，在同一条处理路径中统一批处理与流处理。其核心思想是只使用一个流处理引擎。

Lambda 架构：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/lambda-architecture.png" alt="lambda-architecture" width="500" />
</div>

Kappa 架构：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/kappa-architecture.png" alt="kappa-architecture" width="500" />
</div>

我们的高层设计使用 Kappa 架构，因为历史数据的重新处理也会经过聚合服务。

如果聚合逻辑出现严重 bug 等原因导致需要重新计算聚合数据，可以根据保存的原始数据重新计算：
- 重新计算服务从原始存储中读取数据。这是一个批处理任务。
- 将读取的数据发送到专用聚合服务，避免影响实时聚合服务。
- 将聚合结果发送到第二个消息队列，再更新聚合数据库中的结果。

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/recalculation-example.png" alt="recalculation-example" width="500" />
</div>

### **时间**

聚合需要使用时间戳，可以在两个地方生成：
- 事件时间：广告点击发生的时间。
- 处理时间：服务器处理事件时的系统时间。

由于使用了异步处理（消息队列）并且存在网络延迟，事件时间和处理时间之间可能有明显差异：
- 使用处理时间会导致聚合结果不准确。
- 使用事件时间则需要处理延迟事件。

没有完美方案，需要权衡：

| 时间戳类型 | 优点 | 缺点 |
|---|---|---|
| 事件时间 | 聚合结果更准确 | 客户端时间可能不正确，或者时间戳可能由恶意用户生成 |
| 处理时间 | 服务器时间戳更可靠 | 事件延迟到达时，时间戳不准确 |

由于数据准确性很重要，我们会使用事件时间进行聚合。

为缓解事件延迟问题，可以使用称为“水位线”的技术。

下面的示例中，事件 2 错过了应参与聚合的窗口：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/watermark-technique.png" alt="watermark-technique" width="500" />
</div>

如果有意延长聚合窗口，就能降低事件遗漏的可能性。窗口延长的部分称为“水位线”：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/watermark-2.png" alt="watermark-2" width="500" />
</div>

- 水位线较短：事件遗漏的可能性较高，但延迟较低。
- 水位线较长：事件遗漏的可能性较低，但延迟较高。

无论水位线多长，都仍有事件遗漏的可能。但无需为了概率极低的事件过度优化。

可以通过每日结束时进行对账来处理这类不一致。

### **聚合窗口**

窗口函数有四种：
- 滚动（固定）窗口。
- 跳跃窗口。
- 滑动窗口。
- 会话窗口。

本设计在广告点击聚合中使用滚动窗口：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/tumbling-window.png" alt="tumbling-window" width="500" />
</div>

而在过去 M 分钟点击量最高的前 N 个广告聚合中使用滑动窗口：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/sliding-window.png" alt="sliding-window" width="500" />
</div>

### **投递保证**

我们聚合的数据用于计费，因此数据准确性优先级很高。

因此需要讨论：
- 如何避免重复处理事件。
- 如何确保所有事件都得到处理。

可以使用三种投递保证：至多一次、至少一次和恰好一次。

在大多数情况下，如果允许少量重复数据，至少一次就足够了。但本系统不适用，因为即使只有几个百分点的差异，也可能造成数百万美元的偏差。因此需要使用恰好一次投递语义。

### **数据去重**

最常见的数据质量问题之一是数据重复。

重复数据可能来自多种来源：
- 客户端：客户端可能重复发送同一事件。带有恶意目的的重复事件最好交给风险引擎处理。
- 服务器故障：聚合服务节点在聚合过程中宕机，上游服务未收到确认，因此重新发送事件。

下面的示例展示了最后一跳因未收到事件确认而导致数据重复：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/data-duplication-example.png" alt="data-duplication-example" width="500" />
</div>

在该例中，offset 100 会被多次处理并发送到下游。

一种缓解方式是将最近处理的 offset 存储到 HDFS/S3，但这样可能导致结果无法到达下游：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/data-duplication-example-2.png" alt="data-duplication-example-2" width="500" />
</div>

最后，可以在与下游交互时原子性地保存 offset。为实现这一点，需要执行分布式事务：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/data-duplication-example-3.png" alt="data-duplication-example-3" width="500" />
</div>

<strong>个人补充：</strong>如果下游系统可以幂等地处理聚合结果，就不需要分布式事务。

### **扩展系统**

讨论如何随着系统增长进行扩展。

系统包含三个相互独立的组件：消息队列、聚合服务和数据库。由于彼此解耦，可以独立扩展。

如何扩展消息队列：
- 不限制生产者数量，因此生产者可以轻松扩展。
- 可以将消费者分配到消费者组，并增加消费者数量，以此扩展消费者。
- 为此还要预先创建足够多的分区。
- 如果消费者达到数千个，再平衡可能耗时较长，因此建议在非高峰时段进行。
- 也可以按地理区域划分主题，例如 `topic_na`、`topic_eu` 等。

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/scale-consumers.png" alt="scale-consumers" width="500" />
</div>

如何扩展聚合服务：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/aggregation-service-scaling.png" alt="aggregation-service-scaling" width="500" />
</div>

- 通过增加节点即可轻松扩展 MapReduce 节点。
- 可以通过多线程提高聚合服务的吞吐量。
- 也可以使用 Apache YARN 等资源管理器来实现多进程处理。
- 方案 1 更简单，但方案 2 在实践中更常见，因为它的扩展性更好。
- 多线程示例如下：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/multi-threading-example.png" alt="multi-threading-example" width="500" />
</div>

如何扩展数据库：
- 如果使用 Cassandra，可以通过一致性哈希原生支持横向扩展。
- 集群增加新节点后，数据会自动在所有（虚拟）节点之间重新均衡。
- 该方案不需要手动重新分片。

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/cassandra-scalability.png" alt="cassandra-scalability" width="500" />
</div>

另一个需要考虑的扩展问题是热点：如果某个广告比其他广告更热门、获得更多关注怎么办？

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/hotspot-issue.png" alt="hotspot-issue" width="500" />
</div>

- 上例中，聚合服务节点可以通过资源管理器申请额外资源。
- 资源管理器分配更多资源，避免原节点过载。
- 原节点将事件拆分成 3 组，每个聚合节点处理 100 个事件。
- 聚合结果再写回原聚合节点。

还有更复杂的热点处理方案：
- 全局-本地聚合。
- 拆分不同值聚合。

### **容错**

在聚合节点内，数据是在内存中处理的。如果节点宕机，已处理的数据就会丢失。

可以利用 Kafka 消费者 offset，在其他节点接手后从上次处理位置继续。但还需要维护额外的中间状态，因为我们还要聚合过去 M 分钟内点击量最高的前 N 个广告。

可以在聚合过程中每隔一段时间创建快照：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/fault-tolerance-example.png" alt="fault-tolerance-example" width="500" />
</div>

如果节点宕机，新节点可以读取最近一次提交的消费者 offset 和最新快照，以继续执行任务：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/fault-tolerance-recovery-example.png" alt="fault-tolerance-recovery-example" width="500" />
</div>

### **数据监控与正确性**

聚合数据用于计费，因此必须严格监控系统，以保证数据正确。

可以监控以下指标：
- <strong>延迟：</strong>跟踪不同事件的时间戳，了解系统端到端延迟。
- <strong>消息队列长度：</strong>队列突然增长时，需要增加聚合节点。Kafka 使用分布式提交日志实现，因此需要关注记录滞后等指标。
- <strong>聚合节点的系统资源：</strong>CPU、磁盘、JVM 等。

还需要实现每日结束时运行的批处理对账流程。它会根据原始数据计算聚合结果，并与聚合数据库中的实际数据进行比较：

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/reconciliation-flow.png" alt="reconciliation-flow" width="500" />
</div>

### **替代设计**

在通用系统设计面试中，通常不要求了解大数据处理专用软件的内部实现。

讲清楚思考过程并讨论权衡，比熟悉特定工具更重要，因此本章介绍的是通用方案。

另一种采用现成工具的设计方案是将广告点击数据存储在 Hive 中，并在其上部署 Elasticsearch，以加速查询。

聚合通常使用 ClickHouse 或 Druid 等 OLAP 数据库完成。

<div style="margin-left:3rem">
    <img src="../../21.%20Ad%20Click%20Event%20Aggregation/images/alternative-design.png" alt="alternative-design" width="500" />
</div>

---

## 第 4 步：总结

本章讨论了：
- 数据模型和 API 设计。
- 使用 MapReduce 聚合广告点击事件。
- 扩展消息队列、聚合服务和数据库。
- 缓解热点问题。
- 持续监控系统。
- 使用对账确保正确性。
- 容错能力。

广告点击事件聚合是典型的大数据处理系统。

如果具备以下相关技术的经验，会更容易理解和设计该系统：
- Apache Kafka。
- Apache Spark。
- Apache Flink。
