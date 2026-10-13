# 第 20 章：设计指标监控与告警系统

## 引言
本章重点讨论如何设计一个高度可扩展的**指标监控与告警系统**。这类系统对于保证高可用性和可靠性至关重要。

---

## 第 1 步：理解问题并明确设计范围

指标监控系统可能有很多含义。例如，如果面试官只关心基础设施指标，就不应该设计日志聚合系统。

先来了解问题：
- **候选人：**我们为谁构建这个系统？是大型科技公司的内部监控系统，还是 DataDog 这样的 SaaS？
- **面试官：**仅供内部使用。
- **候选人：**需要收集哪些指标？
- **面试官：**操作系统指标，例如 CPU 负载、内存和磁盘空间；也要收集每秒请求数等高层指标。不包含业务指标。
- **候选人：**要监控的基础设施有多大？
- **面试官：**1 亿日活用户、1,000 个服务器池，每个池有 100 台机器。
- **候选人：**数据需要保留多久？
- **面试官：**假设保留 1 年。
- **候选人：**长期存储时，可以降低指标数据的分辨率吗？
- **面试官：**新收到的指标保留 7 天；之后 30 天聚合到 1 分钟分辨率；30 天后再聚合到 1 小时分辨率。
- **候选人：**支持哪些告警渠道？
- **面试官：**电子邮件、电话、PagerDuty 或 Webhook。
- **候选人：**需要收集错误日志、访问日志等日志吗？
- **面试官：**不需要。
- **候选人：**需要支持分布式系统追踪吗？
- **面试官：**不需要。

### **高层需求与假设**
需要监控的基础设施规模很大：
- 1 亿 DAU。
- 1,000 个服务器池 × 每池 100 台机器 × 每台约 100 个指标，即约 **1,000 万个指标**。
- 数据保留 1 年。
- 数据保留策略：原始数据保留 7 天，1 分钟分辨率数据保留 30 天，1 小时分辨率数据保留 1 年。

可监控的指标包括：
- CPU 负载。
- 请求数。
- 内存使用量。
- 消息队列中的消息数。

### **非功能需求**
- **可扩展性：**系统需要扩展，以容纳更多指标和告警。
- **低延迟：**仪表板和告警查询需要低延迟。
- **可靠性：**系统需要高度可靠，避免漏掉重要告警。
- **灵活性：**未来应能轻松接入新技术。

哪些需求不在范围内？
- **日志监控：**ELK Stack 是这一场景的常见方案。
- **分布式系统追踪：**收集请求在系统多个服务之间流转时的生命周期数据。

---

## 第 2 步：提出高层设计并达成共识

### **基础概念**
指标监控与告警系统由五个核心组件组成：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/metrics-monitoring-core-components.png" alt="metrics-monitoring-core-components" width="500" />
</div>

- **数据收集：**从不同来源收集指标数据。
- **数据传输：**将数据从来源传输到指标监控系统。
- **数据存储：**组织并存储收到的数据。
- **告警：**分析数据、检测异常并生成告警。
- **可视化：**通过图表展示数据。

### **数据模型**

指标数据通常以时间序列形式记录，包含一组带时间戳的值。时间序列可以通过名称以及可选的一组标签来标识。

示例 1：生产服务器实例 i631 在 20:00 的 CPU 负载是多少？

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/metrics-example-1.png" alt="metrics-example-1" width="500" />
</div>

可以使用下表中的数据：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/metrics-example-1-data.png" alt="metrics-example-1-data" width="500" />
</div>

时间序列由指标名称、标签以及特定时间上的一个数据点来标识。

示例 2：过去 10 分钟内，us-west 区域所有 Web 服务器的平均 CPU 负载是多少？

```
CPU.load host=webserver01,region=us-west 1613707265 50

CPU.load host=webserver01,region=us-west 1613707265 62

CPU.load host=webserver02,region=us-west 1613707265 43

CPU.load host=webserver02,region=us-west 1613707265 53

...

CPU.load host=webserver01,region=us-west 1613707265 76

CPU.load host=webserver01,region=us-west 1613707265 83
```

我们可以从存储中读取类似数据来回答这个问题。对这些行中最后一列的值求平均，就能得到平均 CPU 负载。

上面展示的格式称为行协议（line protocol），Prometheus、OpenTSDB 等很多主流监控软件都使用它。

每个时间序列包含的内容如下：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/time-series-data-example.png" alt="time-series-data-example" width="500" />
</div>

数据可以按如下方式可视化：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/time-series-data-viz.png" alt="time-series-data-viz" width="500" />
</div>

- x 轴表示时间。
- y 轴表示要查询的维度，例如指标名称、标签等。

该系统的访问模式以写入为主，并有突发读取。我们收集大量指标，但读取频率不高；例如发生故障时，数据读取会在短时间内激增。

数据存储系统是此设计的核心：
- 不建议使用通用数据库来解决这个问题，尽管通过专家级调优也能实现较大规模。
- 理论上可以使用 NoSQL 数据库，但很难设计出可扩展的 Schema，来高效存储和查询时间序列数据。

有许多数据库专门针对时间序列数据存储进行了优化，其中不少还提供自定义查询接口，以便高效查询时间序列数据：
- OpenTSDB 是一种分布式时间序列数据库，但基于 Hadoop 和 HBase。如果没有现成的相关基础设施，就很难采用这项技术。
- Twitter 使用 MetricsDB，Amazon 提供 Timestream。
- 最流行的两种时间序列数据库是 InfluxDB 和 Prometheus。
- 它们用于存储大量时间序列数据，都基于内存缓存和磁盘存储。

InfluxDB 的规模示例：配置 8 核 CPU 和 32 GB RAM 后，每秒可写入超过 25 万次：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/influxdb-scale.png" alt="influxdb-scale" width="500" />
</div>

不要求你理解指标数据库的内部实现，因为这是小众知识；只有在简历中提到过时，面试官才可能追问。

面试时只需理解指标是时间序列数据，并了解 InfluxDB 等常用时间序列数据库。

时间序列数据库的一项优势是可以根据标签，高效地聚合和分析大量时间序列数据。例如，InfluxDB 会为每个标签建立索引。

不过，必须控制标签的基数，也就是不要使用太多唯一标签。

### **高层设计**

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/high-level-design.png" alt="high-level-design" width="500" />
</div>

- **指标来源：**应用服务器、SQL 数据库、消息队列等。
- **指标收集器：**收集指标数据，并写入时间序列数据库。
- **时间序列数据库：**以时间序列形式保存指标，并提供自定义查询接口以分析大量指标。
- **查询服务：**方便地查询并获取时间序列数据库中的数据。如果数据库接口足够强大，也可以完全替代该服务。
- **告警系统：**向不同告警目标发送告警通知。
- **可视化系统：**以图表形式展示指标。

---

## 第 3 步：深入设计

深入讨论系统中几个较有意思的部分。

### **指标收集**

偶尔丢失一些指标数据并不关键，因此客户端可以发送数据后不等待确认。

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/metrics-collection.png" alt="metrics-collection" width="500" />
</div>

指标收集有拉取和推送两种方式。

拉取模型示例如下：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/pull-model-example.png" alt="pull-model-example" width="500" />
</div>

采用该方案时，指标收集器需要维护最新的服务列表和指标端点列表。可以使用 ZooKeeper 或 etcd 来实现服务发现。

服务发现中保存了从哪些服务收集指标、何时收集等配置规则：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/service-discovery-example.png" alt="service-discovery-example" width="500" />
</div>

指标收集流程详解：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/metrics-collection-flow.png" alt="metrics-collection-flow" width="500" />
</div>

- 指标收集器从服务发现中获取配置元数据，包括拉取间隔、IP 地址、超时和重试参数。
- 指标收集器通过预定义的 HTTP 端点（例如 `/metrics`）拉取指标数据。通常由客户端库实现该端点。
- 另一种方式是，指标收集器向服务发现注册变更事件通知；服务端点变化时，服务发现会通知收集器。
- 还可以让指标收集器定期轮询指标端点配置的变化。

在当前规模下，一台指标收集器不够，需要多个实例。同时，还需要某种同步机制，避免两个收集器重复收集同一指标。

一种解决方案是将收集器和服务器放到一致性哈希环上，并将每台服务器只分配给一台收集器：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/consistent-hash-ring.png" alt="consistent-hash-ring" width="500" />
</div>

推送模型则由服务主动将指标发送给指标收集器：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/push-model-example.png" alt="push-model-example" width="500" />
</div>

通常会在服务实例旁边安装一个收集代理。代理从服务器收集指标，并推送给指标收集器。

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/metrics-collector-agent.png" alt="metrics-collector-agent" width="500" />
</div>

采用这种模型时，可以先在代理中聚合指标再发送给收集器，从而减少收集器要处理的数据量。

另一方面，如果指标收集器无法承受负载，也可以拒绝推送请求。因此，应将收集器置于负载均衡器后的自动扩缩组中。

哪种方式更好？两种方式各有权衡，不同系统会选择不同方案：
- Prometheus 使用拉取架构。
- Amazon CloudWatch 和 Graphite 使用推送架构。

推送与拉取的主要区别如下：

| 比较项 | 拉取 | 推送 |
|---|---|---|
| 易于调试 | 应用服务器上的 `/metrics` 端点可随时查看指标，甚至可以在个人电脑上查看。拉取方式更好。 | 如果收集器没收到指标，问题可能出在网络上。 |
| 健康检查 | 应用服务器未响应拉取请求时，可以快速判断服务器是否宕机。拉取方式更好。 | 如果收集器没收到指标，问题可能出在网络上。 |
| 短时任务 |  | 部分批处理任务生命周期很短，可能来不及被拉取。推送方式更合适。拉取模式可以增加推送网关解决此问题 [22]。 |
| 防火墙或复杂网络环境 | 拉取指标要求所有指标端点都可访问。在多个数据中心的部署中可能会有问题，并且需要更复杂的网络基础设施。 | 如果指标收集器部署在负载均衡器和自动扩缩组后面，就可以从任何位置接收数据。推送方式更好。 |
| 性能 | 拉取通常使用 TCP。 | 推送通常使用 UDP，因此传输指标的延迟更低。不过，建立 TCP 连接的成本与发送指标负载相比很小。 |
| 数据真实性 | 要收集的应用服务器预先定义在配置文件中，因此采集到的数据来源可信。 | 任何客户端都可能向收集器推送数据。可以通过服务器白名单或身份验证来解决。 |

没有绝对更好的方案。大型组织可能需要同时支持两种方式，有时甚至无法安装推送代理。

### **扩展指标传输流水线**

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/metrics-transmission-pipeline.png" alt="metrics-transmission-pipeline" width="500" />
</div>

无论使用推送还是拉取模型，指标收集器都部署在自动扩缩组中。

如果时间序列数据库宕机，仍可能丢失数据。为降低这种风险，可以增加队列机制：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/queuing-mechanism.png" alt="queuing-mechanism" width="500" />
</div>

- 指标收集器将指标数据写入 Kafka。
- 消费者或 Apache Storm、Flink、Spark 等流处理服务处理数据，再写入时间序列数据库。

该方案有以下优点：
- Kafka 是高度可靠且可扩展的分布式消息平台。
- 将数据收集与数据处理解耦。
- Kafka 保留数据，可以避免数据丢失。

可以为每个指标名称分配一个 Kafka 分区，让消费者按指标名称聚合数据。为了进一步扩展，还可以按标签划分分区，并根据优先级安排指标收集顺序。

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/metrics-collection-kafka.png" alt="metrics-collection-kafka" width="500" />
</div>

在这个问题中使用 Kafka 的主要缺点是维护和运维开销较高。另一种方案是使用 [Gorilla](https://www.vldb.org/pvldb/vol8/p1816-teller.pdf) 等大规模数据接入系统。从可扩展性来看，它也可以与使用 Kafka 排队相媲美。

### **聚合可以发生在哪一层**

指标可以在多个环节进行聚合，不同选择各有权衡：
- **收集代理：**客户端收集代理只支持简单聚合逻辑。例如，收集 1 分钟的计数器数据，再发送给指标收集器。
- **数据接入流水线：**要在写入数据库前聚合数据，需要使用 Flink 等流处理引擎。这可以减少写入量，但由于不再保存原始数据，会损失精度。
- **查询侧：**通过可视化系统运行查询时再聚合数据。不会丢失数据，但处理的数据量大时查询可能很慢。

### **查询服务**

将查询服务与时间序列数据库分离，可以把可视化和告警系统与数据库解耦，使数据库能够与客户端解耦，并可随时更换。

可以在这里增加缓存层，以降低时间序列数据库的负载：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/cache-layer-query-service.png" alt="cache-layer-query-service" width="500" />
</div>

也可以完全不增加查询服务，因为大多数可视化和告警系统都有强大的插件，可对接常见时间序列数据库。如果选择了合适的时间序列数据库，也可能不需要自行添加缓存层。

大多数时间序列数据库不支持 SQL，因为 SQL 不适合查询时间序列数据。以下 SQL 示例用于计算指数移动平均值：

```
select id,
       temp,
       avg(temp) over (partition by group_nr order by time_read) as rolling_avg
from (
  select id,
         temp,
         time_read,
         interval_group,
         id - row_number() over (partition by interval_group order by time_read) as group_nr
  from (
    select id,
    time_read,
    "epoch"::timestamp + "900 seconds"::interval * (extract(epoch from time_read)::int4 / 900) as interval_group,
    temp
    from readings
  ) t1
) t2
order by time_read;
```

下面是用 InfluxDB 查询语言 Flux 实现的同一查询：

```
from(db:"telegraf")
  |> range(start:-1h)
  |> filter(fn: (r) => r._measurement == "foo")
  |> exponentialMovingAverage(size:-10s)
```

### **存储层**

谨慎选择时间序列数据库非常重要。

Facebook 发布的研究表明，约 85% 的线上存储查询读取的是过去 26 小时内的数据。

选择能够利用这一特征的数据库，可以显著提高系统性能。InfluxDB 就是一个例子。

无论选择哪种数据库，都可以使用一些优化方式。

数据编码和压缩可以大幅减少数据大小。优秀的时间序列数据库通常内置了这些功能。

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/double-delta-encoding.png" alt="double-delta-encoding" width="500" />
</div>

上例中，不必保存完整时间戳，而是可以保存时间戳之间的差值。

另一种技术是降采样：将高分辨率数据转换成低分辨率数据，以减少磁盘使用量。

可以对旧数据降采样，并允许数据科学家配置规则，例如：
- 7 天：不降采样。
- 30 天：降采样到 1 分钟分辨率。
- 1 年：降采样到 1 小时分辨率。

例如，以下是分辨率为 10 秒的指标表：

| metric | timestamp | hostname | Metric_value |
|---|---|---|---|
| cpu | 2021-10-24T19:00:00Z | host-a | 10 |
| cpu | 2021-10-24T19:00:10Z | host-a | 16 |
| cpu | 2021-10-24T19:00:20Z | host-a | 20 |
| cpu | 2021-10-24T19:00:30Z | host-a | 30 |
| cpu | 2021-10-24T19:00:40Z | host-a | 20 |
| cpu | 2021-10-24T19:00:50Z | host-a | 30 |

降采样到 30 秒分辨率后：

| metric | timestamp | hostname | Metric_value (avg) |
|---|---|---|---|
| cpu | 2021-10-24T19:00:00Z | host-a | 19 |
| cpu | 2021-10-24T19:00:30Z | host-a | 25 |

最后，还可以使用冷存储保存较旧且很少使用的数据。冷存储的成本低得多。

### **告警系统**

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/alerting-system.png" alt="alerting-system" width="500" />
</div>

配置会加载到缓存服务器中。规则通常采用 YAML 格式，例如：

```
- name: instance_down
  rules:

  # Alert for any instance that is unreachable for >5 minutes.
  - alert: instance_down
    expr: up == 0
    for: 5m
    labels:
      severity: page
```

告警管理器从缓存获取告警配置，并根据配置规则定期调用查询服务。如果某条规则满足条件，就会生成告警事件。

告警管理器还负责：
- 过滤、合并和去重告警。例如，某个实例多次触发同一个告警时，只生成一个告警事件。
- 访问控制：将告警管理操作限制给特定人员。
- 重试：确保告警至少成功投递一次。

告警存储是类似 Cassandra 的键值数据库，保存所有告警状态，并确保通知至少发送一次。告警触发后，会发布到 Kafka。

最后，告警消费者从 Kafka 拉取告警数据，并通过电子邮件、短信、PagerDuty、Webhook 等渠道发送通知。

现实中有许多现成的告警系统，通常很难证明有必要自行开发。

### **可视化系统**

可视化系统展示一段时间内的指标和告警。以下是使用 Grafana 构建的仪表板示例：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/grafana-dashboard.png" alt="grafana-dashboard" width="500" />
</div>

构建高质量的可视化系统非常困难，因此通常没有理由不使用 Grafana 等现成方案。

---

## 第 4 步：总结

最终设计如下：

<div style="margin-left:3rem">
    <img src="../../20.%20Metrics%20Monitoring%20and%20Alerting%20System/images/final-design.png" alt="final-design" width="500" />
</div>
