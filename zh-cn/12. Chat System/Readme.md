# 第 12 章：设计聊天系统

## 引言
**聊天系统**支持用户之间实时发送消息。本章讨论如何设计一个聊天应用，包含以下功能：
- **一对一聊天**
- **群聊（最多 100 人）**
- **在线状态提示**
- **多设备支持**
- **推送通知**

系统目标为 **5,000 万日活跃用户（DAU）**，并永久保存聊天记录。

---

## 第 1 步：理解问题

### 需求
1. **功能：**
   - 一对一聊天和群聊（最多 100 名成员）。
   - 文本消息（最多 100,000 个字符）。
   - 在线和离线状态提示。
   - 支持多台设备。
   - 推送通知。
2. **规模：**面向 5,000 万 DAU 设计。
3. **存储：**永久保存聊天记录。

---

## 第 2 步：高层设计

### 通信协议
1. **发送端：**使用 HTTP 发送消息，并通过持久连接提高效率。

      <div style="margin-left:2rem">
      <img src="../../12.%20Chat%20System/images/basic-design.png" alt="Basic Design" width="500">
      </div>

2. **接收端：**
   - **轮询：**
      - 客户端定期询问服务器是否有新消息。
      - 由于频繁发送重复请求，效率较低。

         <img src="../../12.%20Chat%20System/images/polling.png" alt="Polling" width="400">

   - **长轮询：**
      - 服务器会保持连接，直到有消息到达。
      - 对不活跃用户来说效率较低。

         <img src="../../12.%20Chat%20System/images/long-polling.png" alt="Long Polling" width="400">

   - **WebSocket：**
      - 建立双向持久连接，实现实时通信；发送和接收消息都使用这种方式。
      - 通过 WebSocket（ws）协议发送和接收消息。

         <img src="../../12.%20Chat%20System/images/websocket.png" alt="WebSocket" width="400">

---

### 组件

<div style="margin-left:5rem">
   <img src="../../12.%20Chat%20System/images/high-level-stateless-arch.png" alt="High Level Architecture" height="350">
   <img src="../../12.%20Chat%20System/images/high-level-statefull-arch.png" alt="High Level Architecture" height="350" width="550">
</div>

1. **无状态服务：**
   - 处理注册、登录和用户资料管理。
   - 与服务发现集成，以推荐最合适的聊天服务器。
2. **有状态服务：**
   - 聊天服务器维护持久的 WebSocket 连接。
   - 负责消息投递和同步。
3. **第三方集成：**
   - 推送通知服务会在有新消息时通知用户。
   - 通知系统的实现方式可参考“通知系统”章节。

---

### 设计

客户端与聊天服务器之间保持持久的 WebSocket 连接，实现实时消息传递。

<div style="margin-left:3rem">
      <img src="../../12.%20Chat%20System/images/high-level-design.png" alt="High Level Design" width="450">
</div>

- 聊天服务器负责发送和接收消息。
- Presence 服务器负责管理用户的在线和离线状态。
- API 服务器处理用户登录、注册、修改资料等操作。
- 通知服务器发送推送通知。
- 最后，使用键值存储保存聊天记录。聊天记录使用键值存储的原因如下：
   - 易于横向扩展。
   - 键值存储的访问延迟很低。
   - 关系型数据库不擅长处理长尾数据。索引变大后，随机访问的开销会很高。
   - 其他成熟聊天应用也采用键值存储，例如 Facebook Messenger 和 Discord。

以下是一对一聊天和群聊的数据模型：
- 主键是消息 ID，可用于确定消息顺序。
- 群聊使用 `(channel_id, message_id)` 作为复合主键。
  - 可以使用 Snowflake 等全局 64 位序列号生成器生成 ID。
  - 更好的做法是使用本地序列号生成器。本地生成的 ID 只需在群组内唯一。
  - 使用本地 ID 的原因是，只要能在一对一会话或群组会话内维护消息顺序就足够了。

      <img src="../../12.%20Chat%20System/images/one-to-one-chat.png" alt="One to one chat design" width="300">
      <img src="../../12.%20Chat%20System/images/group-chat.png" alt="Group chat design" width="300">

## 第 3 步：深入设计

### 服务发现

<div style="margin-left:3rem">
   <img src="../../12.%20Chat%20System/images/zookeeper.png" alt="Zookeeper" width="400">
</div>

- 服务发现的主要作用是根据地理位置、服务器容量等条件，为客户端推荐最合适的聊天服务器。
- 使用 **Apache ZooKeeper**，根据地理位置和服务器容量等条件分配聊天服务器。
- 通过高效分配负载来降低延迟。

### 消息流程
#### 一对一聊天
1. 用户 A 向聊天服务器 1 发送消息。
2. 聊天服务器 1 为消息分配唯一 ID，并将其存储在键值存储中。
3. 如果用户 B 在线，消息会转发到聊天服务器 2，并通过持久的 WebSocket 连接发送。
4. 如果用户 B 离线，则发送推送通知。

#### 群聊
<div style="margin-left:3rem">
   <img src="../../12.%20Chat%20System/images/group-chat-flow.png" alt="Group Chat Flow" width="400">
</div>

- 将消息分别复制到群组中每位接收者的收件箱。
- 这种方式简化了同步，但群组较大时成本会很高。
- 接收端的一位用户可能会收到多位用户发来的消息。每位接收者都有一个收件箱（消息同步队列），其中包含来自不同发送者的消息。

---

#### 消息同步

许多用户会使用多台设备，因此需要在设备之间同步消息。每台设备都维护一个名为 `cur_max_message_id` 的变量，记录本设备上最新的消息 ID。符合以下两个条件的消息会被视为新消息：

<div style="margin-left:3rem">
   <img src="../../12.%20Chat%20System/images/message-synchronization.png" alt="Message Synchronization" width="400">
</div>

- 接收者 ID 与当前登录用户 ID 相同。
- 键值存储中的消息 ID 大于 `cur_max_message_id`。

---

### 在线状态
1. **心跳机制：**
   <div style="margin-left:3rem">
      <img src="../../12.%20Chat%20System/images/heartbeat-mechanism.png" alt="Heartbeat Mechanism" width="400">
   </div>

   - 客户端定期向 Presence 服务器发送心跳，以表明自己在线。
   - 如果超过阈值仍未收到心跳（例如 `x = 30`），就将用户标记为离线。

2. **扇出模型：**

   <div style="margin-left:3rem">
      <img src="../../12.%20Chat%20System/images/fanout-presence.png" alt="Fanout Presence" width="400">
   </div>

   - 使用发布订阅模型向好友推送状态更新，每对好友维护一个频道。
   - 用户 A 的在线状态变化时，会向三个频道发布事件：A-B、A-C 和 A-D。
   - 用户 B、C、D 分别订阅这三个频道，因此能收到在线状态更新。
   - 这种设计适用于人数较少的用户群组。

---

## 其他注意事项
### 可扩展性
- **横向扩展：**随着用户数增长增加服务器。
- **负载均衡：**在服务器之间均匀分配流量。
- **缓存：**降低数据库负载并改善延迟。

### 错误处理
- **重试机制：**通过重试和排队处理消息投递失败。
- **服务器故障：**发生故障时通过服务发现分配新的服务器。

### 后续扩展
1. **媒体支持：**增加照片和视频处理，包括压缩和云存储。
2. **端到端加密：**保护消息隐私。
3. **客户端缓存：**减少数据传输量，提高性能。
4. **加快加载速度：**使用地理位置分散的缓存网络。
