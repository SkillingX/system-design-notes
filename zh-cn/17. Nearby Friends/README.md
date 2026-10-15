# 第 17 章：附近的人

## 引言
本章重点讨论如何为一款可让用户分享位置、发现**附近好友**的应用设计可扩展的后端。

这项服务与“附近服务”章节的主要区别是：本章中的**位置会持续变化**，而前一章中的商家地址基本保持不变。

---

## 第 1 步：理解问题并明确设计范围

以下问题可以帮助推进面试讨论：
- <strong>澄清：</strong>多近才算“附近”？
- <strong>面试官：</strong>5 英里，并且这个数值应该可以配置。
- <strong>澄清：</strong>距离是按直线计算，还是要考虑好友之间有河流等障碍？
- <strong>面试官：</strong>可以按直线距离计算。
- <strong>澄清：</strong>应用有多少用户？
- <strong>面试官：</strong>10 亿用户，其中 10% 使用附近好友功能。
- <strong>澄清：</strong>需要保存位置历史吗？
- <strong>面试官：</strong>需要，这些数据对机器学习等场景可能有价值。
- <strong>澄清：</strong>可以假设不活跃好友会在 10 分钟后从该功能中消失吗？
- <strong>面试官：</strong>可以。
- <strong>澄清：</strong>需要考虑 GDPR 等隐私法规吗？
- <strong>面试官：</strong>为了简化问题，暂时不考虑。

### **功能需求**
- 用户可以在移动应用中查看附近的好友。每位好友旁显示距离和位置更新时间。
- 附近好友列表每隔几秒更新一次。

### **非功能需求**
- <strong>低延迟：</strong>位置更新需要尽快送达。
- <strong>可靠性：</strong>偶尔丢失数据点可以接受，但系统整体需要保持可用。
- <strong>最终一致性：</strong>位置数据不需要强一致性。不同副本间存在几秒延迟可以接受。

### **数量级估算**
以下估算用于确定潜在系统规模：
- 半径 5 英里内的好友视为附近。
- 位置每 30 秒刷新一次。由于人步行速度较慢，无需过于频繁地更新位置。
- 平均每天有 1 亿用户使用此功能，假设其中 10% 为并发用户，即 1,000 万。
- 平均每位用户有 400 个好友，且这些好友都使用附近好友功能。
- 每页显示 20 个附近好友。
- **位置更新 QPS** = 1,000 万 / 30 ≈ 每秒 33.4 万次更新。

---

## 第 2 步：提出高层设计并达成共识

在讨论 API 和数据模型之前，先研究通信协议，因为它不像传统请求-响应通信模式那么常见。

### **高层设计**

从高层来看，我们需要在通信双方之间建立高效的消息传递机制。可以使用点对点协议，但移动应用经常遇到网络连接不稳定和电量受限等问题，所以实际中并不适用。

更实用的办法是使用共享后端作为扇出机制，将位置更新发送给相关好友：

<div style="margin-left:3rem">
    <img src="../../17.%20Nearby%20Friends/images/fan-out-backend.png" alt="fan-out-backend" width="500" />
</div>

后端负责什么？
- 接收所有活跃用户的位置更新。
- 对于每次位置更新，找出所有应接收该更新的活跃用户，并将更新转发给他们。
- 如果好友之间的距离超过配置阈值，就不转发位置数据。

这个方案看起来简单，但关键挑战是如何支撑当前目标规模。

我们先从较简单的设计开始，再在深入设计中讨论更复杂的方案：

<div style="margin-left:3rem">
    <img src="../../17.%20Nearby%20Friends/images/simple-high-level-design.png" alt="simple-high-level-design" width="500" />
</div>

- <strong>负载均衡器：</strong>将流量分配给 REST API 服务器和双向 WebSocket 服务器。
- <strong>REST API 服务器：</strong>处理管理好友、更新资料等辅助任务。
- <strong>WebSocket 服务器：</strong>有状态服务器，将位置更新请求转发到相应客户端。客户端初始化时，服务器还会提供附近好友的位置，后文将详细讨论。
- <strong>Redis 位置缓存：</strong>保存每个活跃用户最近的位置。每个缓存条目都有 TTL；TTL 过期后，用户不再视为活跃，其数据也会从缓存中删除。
- <strong>用户数据库：</strong>保存用户和好友关系数据。可以使用关系型或 NoSQL 数据库。
- <strong>位置历史数据库：</strong>保存用户位置历史，通常不直接用于附近好友功能，而是用于历史数据分析。
- <strong>Redis Pub/Sub：</strong>作为轻量消息总线，为每位用户的位置更新维护独立频道。

<div style="margin-left:3rem">
    <img src="../../17.%20Nearby%20Friends/images/redis-pubsub-usage.png" alt="redis-pubsub-usage" width="500" />
</div>

上图中，WebSocket 服务器订阅其连接用户对应的频道；收到位置更新后，就将其转发给相应用户。

### **周期性位置更新**

周期性位置更新流程如下：

<div style="margin-left:3rem">
    <img src="../../17.%20Nearby%20Friends/images/periodic-location-update.png" alt="periodic-location-update" width="500" />
</div>

- 移动客户端向负载均衡器发送位置更新。
- 负载均衡器通过该客户端的持久连接，将更新转发给对应的 WebSocket 服务器。
- WebSocket 服务器将位置数据写入位置历史数据库。
- 更新位置缓存中的数据。WebSocket 服务器也会把该用户的位置保存在内存中，以便后续计算距离。
- WebSocket 服务器通过 Redis Pub/Sub 将位置数据发布到用户频道。
- Redis Pub/Sub 将位置更新广播给该用户频道的所有订阅者，也就是负责该用户好友的服务器。
- 订阅该频道的 WebSocket 服务器收到更新后，计算应将更新发送给哪些用户，并进行发送。

更详细的流程如下：

<div style="margin-left:3rem">
    <img src="../../17.%20Nearby%20Friends/images/detailed-periodic-location-update.png" alt="detailed-periodic-location-update" width="500" />
</div>

平均每次位置更新需要转发给 40 位好友：用户平均有 400 位好友，任意时刻其中约 10% 在线。

### **API 设计**
需要支持以下 WebSocket 操作：
- 周期性位置更新：用户向 WebSocket 服务器发送位置数据。
- 客户端接收位置更新：服务器发送好友位置数据和时间戳。
- WebSocket 客户端初始化：客户端发送用户位置，服务器返回附近好友的位置数据。
- 订阅新好友：好友首次上线等情况下，WebSocket 服务器向移动客户端发送需要跟踪的好友 ID。
- 取消订阅好友：好友下线等情况下，WebSocket 服务器向移动客户端发送需要取消订阅的好友 ID。

辅助功能使用传统的 HTTP API 和请求-响应负载。

### **数据模型**
- 位置缓存保存 `user_id` 到 `lat,long,timestamp` 的映射。Redis 很适合充当缓存，因为系统只关心当前位置，而且需要通过 TTL 淘汰数据。
- 位置历史表在关系型表格中保存相同数据，包括上述四个字段。可以使用 Cassandra，因为它适合写入密集型负载。

---

## 第 3 步：深入设计

讨论如何扩展高层设计，使其能够支撑目标规模。

### **各组件的扩展方式**

- <strong>API 服务器：</strong>可以通过自动扩缩组和复制服务器实例轻松扩展。
- <strong>WebSocket 服务器：</strong>可以增加服务器实例，但移除服务器时需要平滑关闭现有连接。例如，可以先在负载均衡器中将服务器标记为“draining”，停止向它分配新连接，然后再将它从服务器池中移除。
- <strong>客户端初始化：</strong>客户端首次连接服务器时，服务器获取用户好友列表，订阅他们在 Redis Pub/Sub 中的频道，从缓存中获取他们的位置，然后将位置转发给客户端。
- <strong>用户数据库：</strong>可以根据 `user_id` 对数据库分片。也可以通过专门的服务和 API 暴露用户与好友数据，并交由独立团队维护。
- <strong>位置缓存：</strong>可以通过增加 Redis 节点轻松分片。TTL 限制了某一时刻的最大内存占用，但系统仍需处理很大的写入负载。
- <strong>Redis Pub/Sub 服务器：</strong>未使用的频道不会消耗内存。因此，可以为所有使用附近好友功能的用户预先分配频道，避免用户上线时临时创建频道并通知活跃 WebSocket 服务器。

### **深入扩展 Redis Pub/Sub 组件**

维护所有 Pub/Sub 频道大约需要 200 GB 内存，可以使用两台各有 100 GB 内存的 Redis 服务器。

不过，系统需要每秒推送约 1,400 万次位置更新。假设单台服务器每秒可处理约 10 万次推送，则至少需要 140 台 Redis 服务器来承载该负载。

因此，需要使用分布式 Redis 服务器集群来承担高强度的 CPU 负载。

要支持分布式 Redis 集群，需要使用 ZooKeeper 或 etcd 等服务发现组件来跟踪哪些服务器处于活动状态。

服务发现组件需要保存以下信息：

<div style="margin-left:3rem">
    <img src="../../17.%20Nearby%20Friends/images/channel-distribution-data.png" alt="channel-distribution-data" width="500" />
</div>

WebSocket 服务器从 ZooKeeper 获取这份编码数据，判断特定频道位于哪台服务器。为了提高效率，可以在每台 WebSocket 服务器内存中缓存哈希环数据。

扩缩服务器集群时，可以设置每日任务，根据历史流量调整集群规模。也可以超额配置集群，以应对流量峰值。

Redis 集群可以视为有状态的存储服务器：频道需要维护状态，订阅者也需要协作并切换到新分配的集群节点。

扩缩集群时，需要注意以下潜在问题：
- 频道迁移会导致 WebSocket 服务器发送大量重新订阅请求。
- 操作期间客户端可能会有一些位置更新丢失。对于本问题可以接受，但仍应尽量减少。建议在一天中流量最低时进行此类操作。
- 增加或移除服务器时，可以使用一致性哈希来减少需要迁移的频道数量。

<div style="margin-left:3rem">
    <img src="../../17.%20Nearby%20Friends/images/consistent-hashing.png" alt="consistent-hashing" width="500" />
</div>

### **添加或移除好友**
添加或移除好友时，负责受影响用户的 WebSocket 服务器需要订阅或取消订阅好友的频道。

“附近的人”功能属于一个更大的应用。可以假设移动客户端会在相关事件发生时调用回调，然后客户端向 WebSocket 服务器发送消息，请求执行相应操作。

### **好友很多的用户**
可以限制每位用户的好友总数，例如 Facebook 的好友上限是 5,000 人。

处理这类“鲸鱼用户”的 WebSocket 服务器负载会更高，但只要 WebSocket 服务器数量充足，应该可以承受。

### **附近的陌生人**
如果面试官要求增加一个功能，让用户的附近地图偶尔显示随机陌生人，该怎么办？

一种做法是根据 GeoHash 定义一组 Pub/Sub 频道：

<div style="margin-left:3rem">
    <img src="../../17.%20Nearby%20Friends/images/geohash-pubsub.png" alt="geohash-pubsub" width="500" />
</div>

同一 GeoHash 区域内的用户订阅对应频道，以接收随机用户的位置更新：

<div style="margin-left:3rem">
    <img src="../../17.%20Nearby%20Friends/images/location-updates-geohash.png" alt="location-updates-geohash" width="500" />
</div>

为了处理用户靠近边界、但位于相邻 GeoHash 网格的情况，也可以同时订阅多个 GeoHash：

<div style="margin-left:3rem">
    <img src="../../17.%20Nearby%20Friends/images/geohash-borders.png" alt="geohash-borders" width="500" />
</div>

### **Redis Pub/Sub 的替代方案**
除了 Redis Pub/Sub，也可以使用 Erlang。Erlang 是一种针对分布式计算应用优化的通用编程语言。

借助 Erlang，可以创建数百万个相互通信的小型 Erlang 进程，并在分布式 Erlang 应用中同时处理 WebSocket 连接和 Pub/Sub 频道。

不过，Erlang 是一门小众语言，可能很难招到经验丰富的 Erlang 开发人员。

---

## 第 4 步：总结

我们设计出了一个支持附近好友功能的系统。

核心组件：
- <strong>WebSocket 服务器：</strong>在客户端与服务器之间进行实时通信。
- <strong>Redis：</strong>快速读写位置数据，并提供 Pub/Sub 频道。

我们还讨论了如何扩展 RESTful API 服务器、WebSocket 服务器、数据层和 Redis Pub/Sub 服务器，并探讨了 Redis Pub/Sub 的替代方案以及“附近的陌生人”功能。
