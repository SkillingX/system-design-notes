# 第 11 章：设计信息流系统

## 引言
**信息流系统**展示不断更新的帖子列表，这些帖子来自用户的联系人，内容包括状态更新、照片、视频和链接。Facebook 信息流、Instagram Feed 和 Twitter 时间线都是例子。本章探讨如何设计可扩展的信息流系统。

---

## 第 1 步：理解问题

### 需求
1. <strong>平台：</strong>同时支持 Web 和移动应用。
2. **功能：**
   - 用户可以发布帖子。
   - 用户可以在信息流中查看好友的帖子。
3. <strong>排序：</strong>为简化设计，信息流按**时间倒序**排列。
4. **规模：**
   - 用户最多可以有 5,000 个好友。
   - 日活跃用户（DAU）为 1,000 万。
   - 信息流可以包含文字、图片和视频。

---

## 第 2 步：高层设计

### 概览
设计包含两个主要流程：
1. <strong>信息流发布：</strong>用户发布帖子，将其写入数据库并分发到好友的信息流。
2. <strong>信息流构建：</strong>用户读取信息流时，系统聚合好友的帖子并按时间倒序排列。

---

### 信息流 API
1. **发布信息流 API：**
   - **端点：**`POST /v1/me/feed`
   - **参数：**`content`（帖子内容）和 `auth_token`（身份验证）。

2. **获取信息流 API：**
   - **端点：**`GET /v1/me/feed`
   - **参数：**`auth_token`（身份验证）。

### 发布信息流

   <div style="margin-left:3rem">
      <img src="../../11.%20News%20Feed%20System/images/feed-publishing.png" alt="Feed Publishing" width="400">
   </div>

1. <strong>用户操作：</strong>用户通过信息流发布 API 发布帖子。
2. <strong>负载均衡器：</strong>将流量分配给 Web 服务器。
3. <strong>Web 服务器：</strong>验证请求身份，并将请求转发给服务。
4. <strong>帖子服务：</strong>将帖子保存到数据库和缓存。
5. <strong>扇出服务：</strong>将帖子传播到缓存中的好友信息流。
6. <strong>通知服务：</strong>向好友发送通知。

### 构建信息流

   <div style="margin-left:3rem">
      <img src="../../11.%20News%20Feed%20System/images/news-feed-building.png" alt="News Feed Building" width="400">
   </div>

1. <strong>用户操作：</strong>用户通过获取 API 请求自己的信息流。
2. <strong>负载均衡器：</strong>将流量分配给 Web 服务器。
3. <strong>Web 服务器：</strong>将请求转发给信息流服务。
4. <strong>信息流服务：</strong>从信息流缓存获取帖子 ID，再从数据库或缓存获取完整帖子详情。

---

## 第 3 步：深入设计

### 深入讨论信息流发布
1. **Web 服务器：**
   - 使用 `auth_token` 验证用户身份。
   - 执行限流，防止垃圾信息。

2. **扇出服务：**
   - <strong>写时扇出：</strong>在写入时将帖子推送到好友的信息流。
     - <strong>优点：</strong>实时更新，读取信息流速度快。
     - <strong>缺点：</strong>好友很多的用户会消耗大量资源。
   - <strong>读时扇出：</strong>在读取信息流时拉取帖子。
     - <strong>优点：</strong>对不活跃用户更高效。
     - <strong>缺点：</strong>读取信息流速度较慢。
   - <strong>混合方案：</strong>大多数用户使用推送模式，关系较多的用户（例如名人）使用拉取模式。

        <img src="../../11.%20News%20Feed%20System/images/feed-publishing-deep-dive.png" alt="Feed Publishing Deep Dive" width="500">

    **扇出服务**的工作方式如下：

    1. <strong>获取好友 ID：</strong>从图数据库获取好友列表。
    2. <strong>根据缓存过滤好友：</strong>读取缓存中的用户设置，排除特定好友，例如已静音的好友，或不符合分享偏好的好友。
    3. <strong>发送到消息队列：</strong>将过滤后的好友列表和新帖子 ID 发送到消息队列进行处理。
    4. <strong>扇出 worker：</strong>worker 从消息队列读取数据并更新信息流缓存。为了节省内存，缓存保存 `<post_id, user_id>` 映射，而不是完整的用户对象和帖子对象。
    5. <strong>写入信息流缓存：</strong>将新帖子 ID 追加到好友的信息流缓存中。可以配置数量上限，只保留最近的帖子。大多数用户主要查看最新内容，这样可以控制缓存的内存使用量。

        <img src="../../11.%20News%20Feed%20System/images/fanout-service.png" alt="Fanout Service" width="500">

## 深入讨论信息流获取

### 缓存架构
缓存分为五层：
1. <strong>信息流缓存：</strong>保存帖子 ID，便于快速读取。
2. <strong>内容缓存：</strong>保存帖子详情（热门帖子放在热点缓存中）。
3. <strong>社交图谱缓存：</strong>保存用户关系数据。
4. <strong>行为缓存：</strong>跟踪用户行为，例如点赞、回复和分享。
5. <strong>计数器缓存：</strong>保存点赞、回复、关注者等数量。

    <img src="../../11.%20News%20Feed%20System/images/cache-architecture.png" alt="Cache Architecture" width="500">

---

## 关键优化

### 扩展能力
1. **数据库扩展：**
   - 横向扩展和分片。
   - 对高流量查询使用只读副本。
2. <strong>无状态 Web 层：</strong>让 Web 服务器保持无状态，以支持横向扩展。

### 缓存
1. 将频繁访问的数据保存在内存中。
2. 使用多层缓存降低延迟和数据库负载。

### 可靠性
1. <strong>一致性哈希：</strong>在服务器之间均匀分配请求。
2. <strong>消息队列：</strong>解耦系统组件并缓冲流量。

### 监控
1. 跟踪 QPS（每秒查询数）和延迟等关键指标。
2. 监控缓存命中率，并据此调整配置。
