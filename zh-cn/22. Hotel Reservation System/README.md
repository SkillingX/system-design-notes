# 第 22 章：酒店预订系统

## 简介

本章将设计一个类似万豪国际酒店集团的**酒店预订系统**。同样的设计思路也适用于 Airbnb、机票预订和电影票预订等系统。

---

## 第一步：理解问题并明确设计范围

在开始设计之前，应先向面试官提问，以澄清需求范围：

- C：系统规模有多大？
- I：我们要为一家拥有 5000 家酒店、100 万间客房的连锁酒店构建网站。
- C：顾客是在预订时付款，还是到店后付款？
- I：预订时全额付款。
- C：顾客只能通过网站预订吗？是否需要支持电话等其他预订方式？
- I：顾客只能通过网站或应用预订。
- C：顾客可以取消预订吗？
- I：可以。
- C：还有其他需要考虑的事项吗？
- I：可以超额预订 10%。酒店会出售比实际房间数更多的房间，因为他们预计会有顾客取消预订。
- C：时间有限，我们先聚焦于酒店信息页、客房详情页、预订客房、管理后台和超额预订支持。
- I：可以。
- I：还有一点，酒店价格会不断变化。假设每间客房的价格每天都会变化。
- C：好的。

### **非功能性需求**

- 支持高并发：旺季时，可能有大量顾客同时尝试预订同一家酒店。
- 延迟适中：用户预订时延迟越低越好，但系统花几秒钟处理也可以接受。

### **粗略估算**

- 总计 5000 家酒店、100 万间客房。
- 假设客房入住率为 70%，平均入住时长为 3 天。
- 预计每日预订量：100 万 × 0.7 ÷ 3 ≈ 24 万笔。
- 每秒预订量：24 万 ÷ 每天约 10^5 秒 ≈ 3 笔。平均预订 TPS 较低。

再估算一下 QPS。假设用户到达预订页面前需要经过三个步骤，并且每个页面的转化率为 10%，那么每 3 笔预订大约对应 30 次预订页面访问，以及 300 次酒店客房详情页访问。

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/qps-estimation.png" alt="QPS 估算" width="500" />
</div>

---

## 第二步：提出高层设计并取得认可

接下来讨论 API 设计、数据模型和高层设计。

### **API 设计**

这里的 API 设计聚焦于酒店预订系统所需的核心端点，并采用 RESTful 风格。

完整系统还需要支持根据大量条件搜索客房等更丰富的 API，但本节不讨论这部分，因为它们在技术上没有太大挑战，超出了本节范围。

**酒店相关 API**

- GET /v1/hotels/{id}：获取酒店详细信息。
- POST /v1/hotels：添加新酒店，仅供运营人员使用。
- PUT /v1/hotels/{id}：更新酒店信息，仅供运营人员使用。
- DELETE /v1/hotels/{id}：删除酒店，仅供运营人员使用。

**客房相关 API**

- GET /v1/hotels/{id}/rooms/{id}：获取客房详细信息。
- POST /v1/hotels/{id}/rooms：添加客房，仅供运营人员使用。
- PUT /v1/hotels/{id}/rooms/{id}：更新客房信息，仅供运营人员使用。
- DELETE /v1/hotels/{id}/rooms/{id}：删除客房，仅供运营人员使用。

**预订相关 API**

- GET /v1/reservations：获取当前用户的预订历史。
- GET /v1/reservations/{id}：获取预订详细信息。
- POST /v1/reservations：创建新预订。
- DELETE /v1/reservations/{id}：取消预订。

创建预订的请求示例：

~~~json
{
  "startDate":"2021-04-28",
  "endDate":"2021-04-30",
  "hotelID":"245",
  "roomID":"U12354673389",
  "reservationID":"13422445"
}
~~~

注意，`reservationID` 是幂等键，用于避免重复预订。详情见[并发问题](#并发问题)一节。

### **数据模型**

在选择数据库之前，先考虑系统需要支持哪些访问模式：

- 查看酒店详细信息。
- 根据日期范围查找可用客房类型。
- 记录预订。
- 查询某笔预订或用户过往的预订历史。

从估算结果看，系统规模不大，但需要为流量高峰做好准备。

因此我们选择关系型数据库，原因如下：

- 关系型数据库适合读多写少的系统。
- NoSQL 数据库通常针对写入进行了优化，但本系统写入量不会太大，因为只有一部分访问网站的用户最终会预订。
- 关系型数据库提供 ACID 保证。对于这类系统很重要，否则可能无法避免余额为负、重复扣款等问题。
- 数据结构清晰，关系型数据库可以轻松建模。

下面是数据库结构设计：

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/schema-design.png" alt="数据库结构设计" width="500" />
</div>

大多数字段都不言自明。值得一提的是 `status` 字段，它表示某间客房当前所处的状态：

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/status-state-machine.png" alt="客房状态机" width="500" />
</div>

这个数据模型适用于 Airbnb 这样的系统，但不适用于酒店场景，因为用户预订的不是某间具体客房，而是某种客房类型。客房号码会在预订时再分配。我们会在[改进后的数据模型](#改进后的数据模型)一节解决这一问题。

### **高层设计**

本设计采用近年来非常流行的微服务架构：

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/high-level-design.png" alt="高层设计" width="500" />
</div>

- **用户**：通过手机或电脑预订酒店客房。
- **管理员**：执行管理操作，例如退款或取消付款等。
- **CDN**：缓存 JS 包、图片、视频等静态资源。
- **公共 API 网关**：托管服务，支持限流、身份验证等功能。
- **内部 API**：仅授权人员可以访问，通常由 VPN 保护。
- **酒店服务**：提供酒店和客房的详细信息。数据较为静态，因此可以积极缓存。
- **房价服务**：提供未来不同日期的客房价格。价格取决于酒店某天的入住率。
- **预订服务**：接收预订请求并预留客房，同时跟踪预订和取消导致的客房库存变化。
- **支付服务**：处理付款，并在成功后更新预订状态。
- **酒店管理服务**：仅供授权人员使用，用于管理和查看预订、酒店等信息。

服务之间可以使用 gRPC 等 RPC 框架进行通信。

---

## 第三步：深入设计

接下来深入讨论：

- 改进后的数据模型。
- 并发问题。
- 可扩展性。
- 如何解决微服务中的数据不一致问题。

### **改进后的数据模型**

如前文所述，需要调整 API 和数据库结构，以便预订某种客房类型，而不是某间特定客房。

预订 API 不再接受 `roomID`，而是接受 `roomTypeID`：

~~~json
POST /v1/reservations
{
  "startDate":"2021-04-28",
  "endDate":"2021-04-30",
  "hotelID":"245",
  "roomTypeID":"12354673389",
  "roomCount":"3",
  "reservationID":"13422445"
}
~~~

更新后的数据库结构如下：

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/updated-schema.png" alt="更新后的数据库结构" width="500" />
</div>

- **room**：保存客房信息。
- **room_type_rate**：保存某类客房的价格信息。
- **reservation**：记录顾客的预订数据。
- **room_type_inventory**：保存酒店客房的库存信息。

下面详细看看 `room_type_inventory` 表，因为它更值得关注：

- **hotel_id**：酒店 ID。
- **room_type_id**：客房类型 ID。
- **date**：某个具体日期。
- **total_inventory**：客房总数，减去暂时下架的客房数。
- **total_reserved**：指定 `hotel_id`、`room_type_id` 和日期下已预订的客房总数。

这张表还有其他设计方式，但为每个 `hotel_id`、`room_type_id` 和日期保存一行数据，可以简化预订管理和查询。

每天通过 CRON 定时任务预先填充表中的数据。

示例数据：

| hotel_id | room_type_id | date | total_inventory | total_reserved |
|----------|--------------|------------|-----------------|----------------|
| 211 | 1001 | 2021-06-01 | 100 | 80 |
| 211 | 1001 | 2021-06-02 | 100 | 82 |
| 211 | 1001 | 2021-06-03 | 100 | 86 |
| 211 | 1001 | ... | ... | |
| 211 | 1001 | 2023-05-31 | 100 | 0 |
| 211 | 1002 | 2021-06-01 | 200 | 16 |
| 2210 | 101 | 2021-06-01 | 30 | 23 |
| 2210 | 101 | 2021-06-02 | 30 | 25 |

下面的 SQL 查询可以检查某种客房是否有空房：

~~~sql
SELECT date, total_inventory, total_reserved
FROM room_type_inventory
WHERE room_type_id = ${roomTypeId} AND hotel_id = ${hotelId}
AND date between ${startDate} and ${endDate}
~~~

基于这些数据，如何检查指定数量的客房是否可预订？注意，我们支持超额预订：

~~~text
if (total_reserved + ${numberOfRoomsToReserve}) <= 110% * total_inventory
~~~

现在估算一下存储规模：

- 5000 家酒店。
- 每家酒店有 20 种客房类型。
- 5000 × 20 × 2 年 × 365 天 = 7300 万行。

7300 万行数据并不算多，单台数据库服务器就能处理。不过，配置读副本（也可以跨可用区部署）有利于提高可用性。

追问：如果预订数据太多，单台数据库无法处理，怎么办？

- 只存储当前和未来的预订数据。历史预订可以迁移到冷存储。
- 对数据库分片。由于查询总会带上 `hotel_id`，可以按照 `hash(hotel_id) % servers_cnt` 分片。

### **并发问题**

还需要解决另一个重要问题：重复预订。

有两种情况需要处理：

- 同一用户重复点击“预订”。
- 多个用户同时尝试预订同一间客房。

下图展示第一种情况：

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/double-booking-single-user.png" alt="同一用户重复预订" width="500" />
</div>

可以通过两种方式解决：

- 客户端处理：用户点击后，前端禁用预订按钮。但如果用户关闭了 JavaScript，就看不到按钮变灰。
- 幂等 API：在 API 中加入幂等键，使用户无论调用端点多少次，同一操作只会执行一次：

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/idempotency.png" alt="幂等处理" width="500" />
</div>

流程如下：

- 用户填写信息并开始预订时，系统生成一个预订订单，并分配全局唯一标识符。
- 使用上一步生成的 `reservation_id` 提交预订请求。
- 如果用户再次点击“完成预订”，请求仍会携带相同的 `reservation_id`，后端就能识别这是重复预订。
- 在数据库中为 `reservation_id` 列添加唯一约束，避免存储多个相同 ID 的记录。

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/unique-constraint-violation.png" alt="唯一约束冲突" width="500" />
</div>

如果多个用户同时预订同一间客房呢？

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/double-booking-multiple-users.png" alt="多用户同时预订" width="500" />
</div>

假设事务隔离级别不是可串行化：

- 用户 1 和用户 2 同时尝试预订同一间客房。
- 事务 1 检查客房数量，发现足够。
- 事务 2 也检查客房数量，同样发现足够。
- 事务 2 预订客房并更新库存。
- 事务 1 仍然看到 100 间客房中已有 99 间被预订，因此也成功预订。
- 两个事务都成功提交了更改。

可以使用某种锁定机制解决这个问题：

- 悲观锁。
- 乐观锁。
- 数据库约束。

预订客房所使用的 SQL 如下：

~~~sql
# step 1: check room inventory
SELECT date, total_inventory, total_reserved
FROM room_type_inventory
WHERE room_type_id = ${roomTypeId} AND hotel_id = ${hotelId}
AND date between ${startDate} and ${endDate}

# For every entry returned from step 1
if((total_reserved + ${numberOfRoomsToReserve}) > 110% * total_inventory) {
  Rollback
}

# step 2: reserve rooms
UPDATE room_type_inventory
SET total_reserved = total_reserved + ${numberOfRoomsToReserve}
WHERE room_type_id = ${roomTypeId}
AND date between ${startDate} and ${endDate}

Commit
~~~

#### 方案一：悲观锁

悲观锁会在更新记录时加锁，阻止其他事务同时更新。

在 MySQL 中，可以使用 `SELECT... FOR UPDATE` 查询实现。它会锁定查询选中的行，直到事务提交。

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/pessimistic-locking.png" alt="悲观锁" width="500" />
</div>

优点：

- 防止应用更新正在被修改的数据。
- 实现简单，可以通过串行化更新避免冲突；数据竞争激烈时很有用。

缺点：

- 多个资源被锁定时可能发生死锁。
- 可扩展性较差：事务持锁时间过长，会影响其他访问该资源的事务。
- 如果查询选中大量资源且事务持续时间较长，影响会更严重。

作者不推荐这种方式，因为它存在可扩展性问题。

#### 方案二：乐观锁

乐观锁允许多个用户同时尝试更新同一条记录。

常见实现方式有两种：版本号和时间戳。由于服务器时钟可能不准确，建议使用版本号。

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/optimistic-locking.png" alt="乐观锁" width="500" />
</div>

- 在数据库表中新增一个 `version` 列。
- 用户修改数据库行之前，先读取版本号。
- 用户更新行时，将版本号加 1 并写回数据库。
- 数据库会进行校验，拒绝新版本号未超过旧版本号的更新。

乐观锁通常比悲观锁快，因为它不需要锁定数据库。但在并发量很高时，由于回滚次数增加，性能可能下降。

优点：

- 防止应用修改过期数据。
- 不需要在数据库中加锁。
- 数据竞争较少、更新冲突罕见时更合适。

缺点：

- 数据竞争激烈时性能较差。

由于预订 QPS 不算太高，乐观锁适用于本系统。

#### 方案三：数据库约束

这种方案与乐观锁很相似，但保护条件通过数据库约束实现：

~~~sql
CONSTRAINT `check_room_count` CHECK((`total_inventory - total_reserved` >= 0))
~~~

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/database-constraint.png" alt="数据库约束" width="500" />
</div>

优点：

- 实现简单。
- 数据竞争较少时效果良好。

缺点：

- 和乐观锁类似，数据竞争激烈时性能较差。
- 数据库约束不像应用代码那样容易进行版本控制。
- 并非所有数据库都支持约束。

由于实现简单，这也是适用于酒店预订系统的方案。

### **可扩展性**

酒店预订系统的负载通常不高。

不过，面试官可能会问：如果系统被扩展到 Booking.com 这样的大型热门旅游网站，该怎么办？在这种情况下，QPS 可能会扩大 1000 倍。

遇到这种情况时，需要弄清楚瓶颈在哪里。所有服务都是无状态的，因此可以通过副本轻松扩展。

数据库则是有状态的，扩展方式没有那么直接。

一种方法是数据库分片：把数据拆分到多个数据库中，每个数据库存储一部分数据。

由于所有查询都会按 `hotel_id` 筛选，因此可以按它分片。假设 QPS 为 30,000，把数据库分成 16 个分片后，每个分片处理 1875 QPS，仍在单个 MySQL 集群的承载能力范围内。

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/database-sharding.png" alt="数据库分片" width="500" />
</div>

还可以用 Redis 缓存客房库存和预订数据，并设置 TTL，让过期日期的旧数据自动清除。

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/inventory-cache.png" alt="库存缓存" width="500" />
</div>

库存数据按 `hotel_id`、`room_type_id` 和日期存储：

~~~text
key: hotelID_roomTypeID_{date}
value: 对应酒店、客房类型和日期的可用客房数。
~~~

数据一致性通过 CDC 流式机制异步维护：读取数据库变更，并将变更应用到另一个系统。Debezium 是一个常见的数据库变更同步到 Redis 的方案。

使用这种机制时，缓存和数据库可能会在一段时间内不一致。本系统可以接受这种情况，因为数据库会阻止无效预订。

这会给 UI 带来一些问题：用户可能需要刷新页面才会看到“客房已订完”。不过，即使没有缓存不一致，只要用户在预订前犹豫过久，也可能遇到客房已被订完的情况。

缓存的优点：

- 降低数据库负载。
- Redis 在内存中管理数据，性能较高。

缓存的缺点：

- 很难维护缓存与数据库之间的一致性，需要考虑不一致对用户体验的影响。

### **服务之间的数据一致性**

单体应用可以使用共享的关系型数据库来保证数据一致性。

在微服务设计中，我们采用了混合方案：部分服务彼此独立，但预订和库存 API 由同一个服务处理。

这样做是为了利用关系型数据库的 ACID 保证来确保一致性。

不过，面试官可能会质疑这不是纯粹的微服务架构，因为纯微服务通常要求每个服务有独立数据库：

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/microservices-vs-monolith.png" alt="微服务与单体架构" width="500" />
</div>

这可能导致一致性问题。在单体服务中，可以利用关系型数据库的事务能力实现原子操作：

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/atomicity-monolith.png" alt="单体服务中的原子操作" width="500" />
</div>

当一个操作跨越多个服务时，要保证原子性就困难得多：

<div style="margin-left:3rem">
    <img src="../../22.%20Hotel%20Reservation%20System/images/microservice-non-atomic-operation.png" alt="微服务中的非原子操作" width="500" />
</div>

有一些常见技术可以处理这类数据不一致问题：

- **两阶段提交**：一种数据库协议，可保证多个节点之间的事务原子提交。不过性能不佳，因为只要有一个节点滞后，所有节点都会阻塞。
- **Saga**：由一系列本地事务组成；如果工作流中的某一步失败，就会触发补偿事务。这是一种最终一致性方案。

解决微服务之间的数据不一致问题很有挑战，会增加系统复杂度。应考虑这种成本是否值得；一种更务实的做法，是把相互依赖的操作封装在同一个关系型数据库中。

---

## 第四步：总结

本章介绍了酒店预订系统的设计。

我们完成了以下步骤：

- 收集需求并进行粗略估算，了解系统规模。
- 在高层设计中介绍 API 设计、数据模型和系统架构。
- 在深入设计中，根据需求变化讨论了不同的数据库结构方案。
- 讨论了竞态条件及其解决方案：悲观锁、乐观锁和数据库约束。
- 讨论了通过数据库分片和缓存扩展系统的方式。
- 最后讨论了如何处理多个微服务之间的数据一致性问题。
