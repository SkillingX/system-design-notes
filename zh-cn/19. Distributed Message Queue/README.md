# 第 19 章：设计分布式消息队列

## 引言

本章将设计一个**分布式消息队列**。

消息队列的优点：
- **解耦：**消除组件之间的紧密耦合，让它们可以独立更新。
- **提高可扩展性：**生产者和消费者可以根据流量独立扩展。
- **提高可用性：**系统某个部分宕机时，其他部分仍可继续与队列交互。
- **提高性能：**生产者可以发送消息，无需等待消费者确认。

常见消息队列实现包括 Kafka、RabbitMQ、RocketMQ、Apache Pulsar、ActiveMQ 和 ZeroMQ。

严格来说，Kafka 和 Pulsar 并不是消息队列，而是事件流平台。不过两类产品的功能逐渐趋同，界限也越来越模糊。

本章将设计一个支持更高级功能的消息队列，例如长时间保留数据、重复消费消息等。

---

## 第 1 步：理解问题并明确设计范围

消息队列应当支持一些基本功能：生产者生产消息，消费者消费消息。不过，性能、消息投递和数据保留等方面还有不同的设计考量。

以下是候选人与面试官之间的一组示例问答：
- **候选人：**消息是什么格式，平均大小是多少？只有文本吗？
- **面试官：**消息只有文本，通常是几 KB。
- **候选人：**消息可以重复消费吗？
- **面试官：**可以，多个消费者可以重复消费同一条消息。这是新增需求，传统消息队列不支持。
- **候选人：**消息是否必须按照生产顺序消费？
- **面试官：**是，需要保证顺序。这是传统消息队列不支持的新增需求。
- **候选人：**数据保留时间有什么要求？
- **面试官：**消息需要保留两周。这是新增需求。
- **候选人：**需要支持多少生产者和消费者？
- **面试官：**越多越好。
- **候选人：**希望支持哪种数据投递语义：至多一次、至少一次还是恰好一次？
- **面试官：**至少一次是必需的。理想情况下，三种语义都支持，并且可以配置。
- **候选人：**端到端延迟和吞吐量有什么目标？
- **面试官：**既要支持日志聚合等场景中的高吞吐量，也要支持传统场景中的低吞吐量。

### **功能需求**
- 生产者向消息队列发送消息。
- 消费者从队列中消费消息。
- 消息可以消费一次或多次。
- 历史数据可以截断。
- 消息大小在 KB 级别。
- 需要保证消息顺序。
- 可以配置数据投递语义：至多一次、至少一次或恰好一次。

### **非功能需求**
- **高吞吐量或低延迟：**根据使用场景配置。
- **可扩展性：**系统应为分布式架构，并能应对消息量突然激增。
- **持久性与耐久性：**数据应持久化到磁盘并在节点间复制。

传统消息队列通常不支持数据保留，也不保证消息顺序。这样可以大幅简化设计，后文会讨论这一点。

---

## 第 2 步：提出高层设计并达成共识

消息队列的关键组件如下：

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/message-queue-components.png" alt="message-queue-components" width="500" />
</div>

- 生产者向队列发送消息。
- 消费者订阅队列并消费其中的消息。
- 消息队列作为中间服务，将生产者和消费者解耦，使它们可以独立扩展。
- 生产者和消费者都是客户端，消息队列是服务器。

### **消息模型**

第一种消息模型是点对点模式，常见于传统消息队列：

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/point-to-point-model.png" alt="point-to-point-model" width="500" />
</div>

- 消息被发送到队列，并且只会被一位消费者消费。
- 可以有多个消费者，但每条消息只消费一次。
- 消息被确认消费后，会从队列中移除。
- 点对点模型不保留数据，而我们的设计会保留数据。

另一方面，发布订阅模型更常见于事件流平台：

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/publish-subscribe-model.png" alt="publish-subscribe-model" width="500" />
</div>

- 在该模型中，消息与某个主题相关联。
- 消费者订阅主题，并接收发送到该主题的所有消息。

### **主题、分区与 Broker**

如果某个主题的数据量太大，该怎么办？一种扩展方式是将主题拆分为多个分区（也称分片）：

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/partitions.png" alt="partitions" width="500" />
</div>

- 发送到主题的消息会均匀分布到各个分区。
- 承载分区的服务器称为 Broker。
- 每个主题都像一个队列，按照 FIFO 顺序处理消息。消息顺序只在单个分区内保证。
- 消息在分区中的位置称为 **offset**。
- 每条生产的消息都会发送到某个特定分区。分区键决定消息进入哪个分区。
  - 例如，可以使用 `user_id` 作为分区键，保证同一用户的消息有序。
- 每个消费者可以订阅一个或多个分区。多个消费者消费同一主题的消息时，可以组成一个消费者组。

### **消费者组**

消费者组是一起消费某个主题消息的一组消费者：

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/consumer-groups.png" alt="consumer-groups" width="500" />
</div>

- 按消费者组复制消息，而不是按每个消费者复制。
- 每个消费者组维护自己的 offset。
- 消费者组并行读取消息可以提高吞吐量，但会削弱顺序保证。
- 可以限制一个分区只允许一个组内消费者订阅，以缓解此问题。
- 因此，组内消费者数量不能超过分区数量。

### **高层架构**

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/high-level-architecture.png" alt="high-level-architecture" width="500" />
</div>

- **客户端：**包括生产者和消费者。生产者向指定主题推送消息；消费者组订阅主题消息。
- **Broker：**承载多个分区。每个分区保存某个主题的一部分消息。
- **数据存储：**在分区中保存消息。
- **状态存储：**保存消费者状态。
- **元数据存储：**保存配置和主题属性。
- **协调服务：**负责服务发现（哪些 Broker 存活）和 Leader 选举（由哪个 Broker 担任 Leader 并负责分配分区）。

---

## 第 3 步：深入设计

为了实现高吞吐量并满足较长的数据保留要求，我们作出了以下重要设计选择：
- 选择充分利用现代 HDD 特性和操作系统磁盘缓存策略的磁盘数据结构。
- 消息数据结构不可变，避免额外复制。在高流量系统中应尽量避免复制。
- 围绕批处理设计写入，因为小型 I/O 不利于实现高吞吐量。

### **数据存储**

要为消息选择合适的数据存储，需要了解消息的特性：
- 写入密集，也可能读取密集。
- 不需要更新或删除操作。传统消息队列中存在“删除”操作，是因为消息不保留。
- 主要采用顺序读写模式。

可选方案：
- **数据库：**不太理想，因为一般数据库不擅长同时处理读写密集型系统。
- **预写日志（WAL）：**只支持追加写入的纯文本文件，非常适合 HDD。
  - 将分区拆分成多个段，避免维护过大的单个文件。
  - 旧段为只读状态，只有最新段接受写入。

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/wal-example.png" alt="wal-example" width="500" />
</div>

WAL 文件配合传统 HDD 使用时效率很高。

有一种误解认为 HDD 访问速度很慢，但实际速度很大程度上取决于访问模式。顺序访问（本系统采用的模式）下，HDD 可达到每秒数 MB 的读写速度，足以满足需求。我们还可以充分利用操作系统对磁盘数据进行积极内存缓存的机制。

### **消息数据结构**

生产者、队列和消费者之间的消息 Schema 应保持一致，以避免额外复制并提升处理效率。

消息结构示例：

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/message-structure.png" alt="message-structure" width="500" />
</div>

消息的 key 指定它所属的分区。例如，可以使用 `hash(key) % numPartitions` 进行映射。为了提高灵活性，生产者可以覆盖默认 key，以控制消息分布到哪些分区。

消息的 value 是消息负载，可以是纯文本或压缩后的二进制块。

**注意：**与传统键值存储不同，消息 key 不必唯一。允许 key 重复，甚至允许 key 缺失。

消息中的其他字段：
- **Topic：**消息所属的主题。
- **Partition：**消息所属分区的 ID。
- **Offset：**消息在分区中的位置。可通过 `topic`、`partition`、`offset` 定位消息。
- **Timestamp：**消息存储的时间。
- **Size：**消息大小。
- **CRC：**用于确保消息完整性的校验和。

通过增加额外字段，也可以支持过滤等其他功能。

### **批处理**

批处理对系统性能至关重要。生产者、消费者和消息队列都会使用批处理。

批处理很重要，原因包括：
- 操作系统可以将多条消息合并处理，分摊昂贵的网络往返开销。
- 消息会以批次形式顺序写入 WAL，从而产生大量顺序写入，并有效利用磁盘缓存。

延迟和吞吐量之间需要权衡：
- 批次越大，吞吐量越高，但延迟也越高。
- 批次越小，吞吐量越低，但延迟也越低。

如果系统作为传统消息队列部署，需要较低延迟，可以将批次调小。

如果针对吞吐量优化，可能需要为每个主题配置更多分区，以弥补顺序磁盘写入速度较慢的问题。

### **生产者流程**

生产者想要向某个分区发送消息时，应该连接到哪个 Broker？

一种方法是加入路由层，将消息路由到正确的 Broker。启用复制后，正确的 Broker 是 Leader 副本：

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/routing-layer.png" alt="routing-layer" width="500" />
</div>

- 路由层从元数据存储读取副本分配计划，并在本地缓存。
- 生产者将消息发送到路由层。
- 消息被转发到 Broker 1，它是该分区的 Leader。
- Follower 副本从 Leader 拉取新消息。收到足够的确认后，Leader 提交数据并向生产者响应。

设置副本是为了实现容错。

该方案可行，但有以下缺点：
- 增加了一个组件，导致额外的网络跳数。
- 不支持消息批处理。

为缓解这些问题，可以将路由层嵌入生产者：

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/routing-layer-producer.png" alt="routing-layer-producer" width="500" />
</div>

- 网络跳数减少，延迟降低。
- 生产者可以控制消息被路由到哪个分区。
- 缓冲区允许在内存中批处理消息，并在单个请求中发送更大的批次，从而提高吞吐量。

批次大小是吞吐量和延迟之间的经典权衡。

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/batch-size-throughput-vs-latency.png" alt="batch-size-throughput-vs-latency" width="500" />
</div>

- 批次越大，等待提交的时间越长。
- 批次越小，请求越早发送，延迟越低，但吞吐量也越低。

### **消费者流程**

消费者指定分区中的 offset，随后接收从该位置开始的一批消息：

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/consumer-example.png" alt="consumer-example" width="500" />
</div>

设计消费者时需要考虑使用推送还是拉取模型：
- **推送模型：**Broker 收到消息后立即推送给消费者，因此延迟较低。
  - 如果消费速度赶不上生产速度，消费者可能会过载。
  - 由于消费速率由 Broker 控制，难以适配处理能力不同的消费者。
- **拉取模型：**消费者自行控制消费速率。
  - 消费速度较慢时，消费者不会过载；可以通过扩容追赶积压。
  - 拉取模型更适合批处理，因为推送模型下 Broker 不知道消费者能处理多少消息。
  - 使用拉取模型时，消费者可以主动获取较大的消息批次。
  - 缺点是延迟较高，而且没有新消息时仍会产生额外网络请求。可以通过长轮询缓解后一个问题。

因此，大多数消息队列（包括我们的设计）都选择拉取模型。

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/consumer-flow.png" alt="consumer-flow" width="500" />
</div>

- 新消费者订阅主题 A，并加入组 1。
- 通过对组名进行哈希，找到对应的 Broker 节点。这样，同一组的所有消费者都会连接到同一个 Broker。
- 注意，消费者组协调器与协调服务（ZooKeeper）不是同一组件。
- 协调器确认消费者已加入组，并将分区 2 分配给该消费者。
- 分区分配策略有多种，例如轮询、范围分配等。
- 消费者从最近一次 offset 开始获取消息。状态存储保存消费者 offset。
- 消费者处理消息，并向 Broker 提交 offset。这两个操作的先后顺序会影响消息投递语义。

### **消费者再平衡**

消费者再平衡负责决定每个分区由哪些消费者负责。

当消费者加入或离开，或者增加或移除分区时，就会触发这一流程。

Broker 作为协调器，在组织再平衡流程中发挥重要作用。

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/consumer-rebalancing.png" alt="consumer-rebalancing" width="500" />
</div>

- 同一组中的所有消费者都连接到同一个协调器。协调器通过对组名进行哈希来确定。
- 消费者列表发生变化时，协调器会选择新的组 Leader。
- 组 Leader 计算新的分区分配计划并报告给协调器，再由协调器广播给其他消费者。

协调器停止收到某组消费者的心跳后，就会触发再平衡：

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/consumer-rebalance-example.png" alt="consumer-rebalance-example" width="500" />
</div>

下面看看消费者加入某个组时会发生什么：

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/consumer-join-group-usecase.png" alt="consumer-join-group-usecase" width="500" />
</div>

- 最初，组内只有消费者 A，由它消费所有分区。
- 消费者 B 发送请求，申请加入该组。
- 协调器在某次心跳响应中通知所有组成员进行被动再平衡。
- 所有消费者重新加入该组后，协调器会选出一位 Leader，并通知其他成员选举结果。
- Leader 生成分区分配计划并发送给协调器，其他消费者等待该计划。
- 消费者开始消费新分配分区中的消息。

消费者离开组时会发生以下情况：

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/consumer-leaves-group-usecase.png" alt="consumer-leaves-group-usecase" width="500" />
</div>

- 消费者 A 和 B 在同一个组中。
- 消费者 B 请求离开该组。
- 协调器收到 A 的心跳后，通知组成员开始再平衡。
- 其余步骤相同。

消费者长时间没有发送心跳时，处理流程也类似：

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/consumer-no-heartbeat-usecase.png" alt="consumer-no-heartbeat-usecase" width="500" />
</div>

### **状态存储**

状态存储保存分区与消费者之间的映射，以及分区最近一次被消费的 offset。

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/state-storage.png" alt="state-storage" width="500" />
</div>

组 1 的 offset 为 6，表示此前的消息均已消费。如果消费者崩溃，新消费者会从该消息之后继续处理。

消费者状态的数据访问模式：
- 读写操作频繁，但数据量较小。
- 数据经常更新，但很少删除。
- 随机读写。
- 数据一致性很重要。

根据这些需求，ZooKeeper 等快速键值存储很合适。

### **元数据存储**

元数据存储保存配置和主题属性，例如分区数、保留时间和副本分布。

元数据变化不频繁、数据量小，但对一致性要求很高。ZooKeeper 是不错的选择。

### **ZooKeeper**

ZooKeeper 是构建分布式消息队列的重要组件。

它是一种分层键值存储，常用于分布式配置、同步服务和命名注册（即服务发现）。

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/zookeeper.png" alt="zookeeper" width="500" />
</div>

采用这种设计后，Broker 只需维护消息数据；元数据和状态存储由 ZooKeeper 负责。

ZooKeeper 还可帮助选举 Broker 副本的 Leader。

### **复制**

分布式系统中不可避免会出现硬件故障。可以通过复制实现高可用。

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/replication-example.png" alt="replication-example" width="500" />
</div>

- 每个分区会复制到多个 Broker 上，但只有一个 Leader 副本。
- 生产者向 Leader 副本发送消息。
- Follower 从 Leader 拉取复制的消息。
- 足够多的副本完成同步后，Leader 向生产者返回确认。
- 每个分区的副本分配方式称为副本分布计划。
- 分区 Leader 会生成副本分布计划并保存到 ZooKeeper。

### **同步副本**

需要解决的一个问题是，如何让分区的 Leader 和 Follower 之间保持消息同步。

同步副本（ISR）是与 Leader 保持同步的分区副本。

`replica.lag.max.messages` 定义了副本最多可以落后 Leader 多少条消息，超过后就不再视为同步。

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/in-sync-replicas-example.png" alt="in-sync-replicas-example" width="500" />
</div>

- 已提交 offset 为 13。
- 两条新消息已写入 Leader，但尚未提交。
- ISR 中的所有副本同步某条消息后，该消息才会提交。
- 副本 2 和 3 已完全追上 Leader，因此属于 ISR。
- 副本 4 落后了，因此暂时从 ISR 中移除。

ISR 反映了性能与持久性之间的权衡：
- 为了避免生产者丢失消息，发送确认前应确保所有副本都已同步。
- 但较慢的副本会导致整个分区不可用。

确认策略可以配置。

`ACK=all` 表示 ISR 中的所有副本都必须同步消息。消息发送较慢，但持久性最高。

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/ack-all.png" alt="ack-all" width="500" />
</div>

`ACK=1` 表示 Leader 收到消息后就向生产者确认。消息发送较快，但持久性较低。

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/ack-1.png" alt="ack-1" width="500" />
</div>

`ACK=0` 表示生产者无需等待 Leader 确认就发送消息。消息发送最快，但持久性最低。

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/ack-0.png" alt="ack-0" width="500" />
</div>

在消费者侧，可以让所有消费者连接分区 Leader，并从那里读取消息：
- 这种设计最简单，也最容易运维。
- 一个组内的同一分区只会分配给一位消费者，因此限制了与 Leader 副本的连接数。
- 只要主题不是特别热门，Leader 副本的连接数通常不会太高。
- 可以通过增加分区和消费者来扩展热门主题。
- 在某些场景中，也可以让消费者从 ISR 中的副本读取，例如消费者位于另一个数据中心时。

ISR 列表由 Leader 维护，Leader 会跟踪自身与每个副本之间的延迟。

### **可扩展性**

下面评估如何扩展系统的不同部分。

#### 生产者

与消费者相比，生产者规模小得多。增加或移除生产者实例即可轻松扩展。

#### 消费者

不同消费者组之间相互隔离，因此可以随时轻松增删消费者组。

消费者再平衡可以平滑处理消费者组中的成员变化。

消费者组再平衡帮助系统实现可扩展性和容错能力。

#### Broker

Broker 如何处理故障？

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/broker-failure-recovery.png" alt="broker-failure-recovery" width="500" />
</div>

- Broker 故障后，剩余副本仍足够，因此不会丢失分区数据。
- 选出新的 Leader 后，Broker 协调器会将故障 Broker 上的分区重新分配给现有副本。
- 现有副本接管新的分区并作为 Follower，直到追上 Leader 并加入 ISR。

提高 Broker 容错能力还需要考虑：
- ISR 的最小数量会影响延迟和安全性，可以根据需求调整。
- 如果某分区的所有副本都在同一节点上，会浪费资源。副本应分布在不同 Broker 上。
- 如果某分区的所有副本都崩溃，数据将永久丢失。跨数据中心部署副本有帮助，但会增加延迟。一种变通方案是使用[数据镜像](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=27846330)。

新增 Broker 时，如何重新分配副本？

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/broker-replica-redistribution.png" alt="broker-replica-redistribution" width="500" />
</div>

- 可以暂时允许副本数超过配置值，直到新 Broker 追上进度。
- 新 Broker 追上后，可以移除不再需要的分区副本。

#### 分区

增加分区时，需要通知生产者，并触发消费者再平衡。

从数据存储角度看，可以只将新消息写入新分区，而不复制所有旧消息：

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/partition-exmaple.png" alt="partition-example" width="500" />
</div>

减少分区则更复杂：

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/partition-decrease.png" alt="partition-decrease" width="500" />
</div>

- 分区下线后，新消息只会写入剩余分区。
- 下线分区不会立即删除，因为消费者仍可能读取其中的消息。
- 预先配置的保留时间过后，数据才会被截断并释放存储空间。
- 过渡期间，生产者只向活动分区发送消息，消费者则从所有分区读取。
- 保留期到期后，再对消费者进行再平衡。

### **数据投递语义**

下面讨论不同的数据投递语义。

#### 至多一次

在这种保证下，消息不会重复投递，但也可能完全不投递。

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/at-most-once.png" alt="at-most-once" width="500" />
</div>

- 生产者异步向主题发送消息。若投递失败，不会重试。
- 消费者获取消息后立即提交 offset。如果消费者在处理消息前崩溃，该消息就不会被处理。

#### 至少一次

消息可能被发送多次，但不会有消息一直未处理。

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/at-least-once.png" alt="at-least-once" width="500" />
</div>

- 生产者使用 `ack=1` 或 `ack=all` 发送消息，遇到问题时会持续重试。
- 消费者获取消息后，只有处理完成才提交 offset。
- 如果消费者在处理消息后、提交 offset 前崩溃，消息可能会重复投递。
- 因此，这种语义适用于允许重复数据或可以进行去重的场景。

#### 恰好一次

对系统而言，实现成本非常高，但这是对用户最友好的投递保证：

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/exactly-once.png" alt="exactly-once" width="500" />
</div>

### **高级功能**

下面讨论一些面试中可能会提到的高级功能。

#### 消息过滤

某些消费者可能只想消费某个分区中特定类型的消息。

可以为每一类消息创建单独主题，但如果系统有许多不同场景，这样做的成本会很高：
- 在不同主题中存储相同消息会浪费资源。
- 每增加一项消费者需求，生产者就需要跟着调整，导致两者紧密耦合。

可以通过消息过滤解决这些问题：
- 最简单的做法是在消费者端过滤，但会产生不必要的消费者流量。
- 也可以为消息附加标签，消费者声明自己订阅的标签。
- 还可以根据消息负载过滤，但加密或序列化消息会让这一做法变得困难且不安全。
- 对于更复杂的数学表达式，Broker 可以实现语法解析器或脚本执行器，但这会让消息队列变得笨重。

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/message-filtering.png" alt="message-filtering" width="500" />
</div>

#### 延迟消息与定时消息

某些场景需要延迟或定时投递消息。例如，可以提交一项支付验证任务，让消费者在 30 分钟后检查支付是否成功。

一种实现方式是先将消息发送到 Broker 的临时存储中，到达指定时间后再将消息移动到分区：

<div style="margin-left:3rem">
    <img src="../../19.%20Distributed%20Message%20Queue/images/delayed-message-implementation.png" alt="delayed-message-implementation" width="500" />
</div>

- 临时存储可以由一个或多个专用消息主题实现。
- 定时功能可以通过专用延迟队列或[分层时间轮](http://www.cs.columbia.edu/~nahum/w6998/papers/sosp87-timing-wheels.pdf)实现。

---

## 第 4 步：总结

还可以讨论以下内容：
- **通信协议：**需要支持所有使用场景和大数据量，并校验消息完整性。常见协议包括 AMQP 和 Kafka 协议。
- **重试消费：**如果无法立即处理消息，可以将其发送到专用重试主题，稍后再次尝试。
- **历史数据归档：**旧消息可以备份到 HDFS 或对象存储（例如 S3）等高容量存储中。
