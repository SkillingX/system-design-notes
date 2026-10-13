# 第 25 章：实时游戏排行榜

## 简介

本章将为一款在线手机游戏设计一个**排行榜**：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/leaderboard.png" alt="排行榜" width="500" />
</div>

---

## 第一步：理解问题并明确设计范围

- C：排行榜分数如何计算？
- I：玩家每赢一场比赛就获得 1 分。
- C：所有玩家都会进入排行榜吗？
- I：会。
- C：排行榜是否按时间段划分？
- I：每个月都会开始一场新赛事，并生成一份新的排行榜。
- C：能否只关注前 10 名用户？
- I：需要展示前 10 名，并显示指定用户的名次。如果时间允许，还可以讨论如何展示排行榜中某个用户附近的玩家。
- C：每场赛事有多少用户？
- I：500 万 DAU、2500 万 MAU。
- C：每场赛事平均会进行多少场比赛？
- I：每位玩家平均每天进行 10 场比赛。
- C：如果两位玩家得分相同，如何确定排名？
- I：这种情况下他们的排名相同。如果时间允许，可以讨论如何打破平局。
- C：排行榜需要实时更新吗？
- I：需要，我们希望展示实时结果，或尽可能接近实时的结果。不能只展示批量生成的历史结果。

### **功能性需求**

- 展示排行榜前 10 名玩家。
- 显示指定用户的名次。
- （加分项）展示指定用户上方和下方各 4 个名次的玩家。

### **非功能性需求**

- 实时更新分数。
- 分数更新后应实时反映在排行榜上。
- 系统应具备良好的可扩展性、可用性和可靠性。

### **粗略估算**

假设有 5000 万 DAU，且玩家在 24 小时内均匀分布，则平均每秒有 50 个用户。实际分布通常不均匀，因此估算高峰在线用户数为每秒 250 人。

获得分数的 QPS：假设每位用户平均每天进行 10 场比赛，则 50 人/秒 × 10 = 500 QPS。峰值 QPS 为 2500。

获取排行榜前 10 名的 QPS：假设用户平均每天打开一次排行榜，则 QPS 为 50。

---

## 第二步：提出高层设计并取得认可

### **API 设计**

首先需要一个更新用户分数的 API：

~~~http
POST /v1/scores
~~~

该 API 接收两个参数：`user_id` 和用户赢得比赛后获得的 `points`。

该 API 只能由游戏服务器访问，客户端不应直接调用。

下一个 API 用于获取排行榜前 10 名玩家：

~~~http
GET /v1/scores
~~~

响应示例：

~~~json
{
  "data": [
    {
      "user_id": "user_id1",
      "user_name": "alice",
      "rank": 1,
      "score": 12543
    },
    {
      "user_id": "user_id2",
      "user_name": "bob",
      "rank": 2,
      "score": 11500
    }
  ],
  ...
  "total": 10
}
~~~

也可以获取指定用户的分数：

~~~http
GET /v1/scores/{:user_id}
~~~

响应示例：

~~~json
{
    "user_info": {
        "user_id": "user5",
        "score": 1000,
        "rank": 6,
    }
}
~~~

### **高层架构**

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/high-level-architecture.png" alt="高层架构" width="500" />
</div>

- 玩家赢得比赛后，客户端向游戏服务发送请求。
- 游戏服务验证胜利是否有效，并调用排行榜服务更新玩家分数。
- 排行榜服务在排行榜存储中更新用户分数。
- 玩家调用排行榜服务获取排行榜数据，例如前 10 名玩家和指定玩家的名次。

另一种备选设计是让客户端直接在排行榜服务中更新分数：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/alternative-design.png" alt="备选设计" width="500" />
</div>

这种方案不安全，容易受到中间人攻击。玩家可以设置代理并随意修改自己的分数。

还有一点需要注意：对于由服务器管理游戏逻辑的游戏，客户端无需显式调用服务器记录胜利。服务器会根据游戏逻辑自动完成记录。

另一个需要考虑的问题是，是否应在游戏服务器和排行榜服务之间放置消息队列。如果其他服务也需要接收比赛结果，消息队列会很有用。但目前的面试需求并未明确要求这一点，因此设计中暂不加入：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/message-queue-based-comm.png" alt="基于消息队列的通信" width="500" />
</div>

### **数据模型**

下面讨论排行榜数据的存储选项：关系型数据库、Redis 和 NoSQL。

NoSQL 方案将在深入设计一节讨论。

#### 关系型数据库方案

如果暂时不考虑规模且用户数量不多，关系型数据库足以满足需求。

可以先为每个月创建一张简单的排行榜表（作者注：这不太合理。只需增加一个 `month` 列，就可以避免每个月维护新表的麻烦）：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/leaderboard-table.png" alt="排行榜数据表" width="500" />
</div>

表中还可以加入其他数据，但这些数据与要执行的查询无关，因此省略。

用户赢得一分时会发生什么？

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/user-wins-point.png" alt="用户获得分数" width="500" />
</div>

如果用户还不在表中，需要先插入：

~~~sql
INSERT INTO leaderboard (user_id, score) VALUES ('mary1934', 1);
~~~

之后的请求只需更新分数：

~~~sql
UPDATE leaderboard set score=score + 1 where user_id='mary1934';
~~~

如何找到排行榜中的顶尖玩家？

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/find-leaderboard-position.png" alt="查找排行榜名次" width="500" />
</div>

可以执行以下查询：

~~~sql
SELECT (@rownum := @rownum + 1) AS rank, user_id, score
FROM leaderboard
ORDER BY score DESC;
~~~

不过，这种方式性能不佳，因为它需要扫描数据库表中的所有记录并排序。

可以通过在 `score` 上创建索引，并使用 `LIMIT` 避免扫描所有数据来优化：

~~~sql
SELECT (@rownum := @rownum + 1) AS rank, user_id, score
FROM leaderboard
ORDER BY score DESC
LIMIT 10;
~~~

但如果用户不在排行榜前列，而我们需要查找其排名，这种方案就不容易扩展。

#### Redis 方案

我们需要一种能够支持数百万玩家的方案，并避免依赖复杂的数据库查询。

Redis 是内存数据存储，速度很快，而且它提供了适合本场景的数据结构：有序集合。

有序集合类似编程语言中的集合，可以按指定条件保持数据有序。其内部使用哈希表维护键（user_id）与值（score）的映射，并使用跳表按分数对用户排序：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/sorted-set.png" alt="有序集合" width="500" />
</div>

跳表如何工作？

- 它是一种支持快速查找的链表。
- 它由有序链表和多层索引组成。

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/skip-list.png" alt="跳表" width="500" />
</div>

这种结构使大规模数据集中的特定值能够被快速查找。以下示例包含 64 个节点：在普通链表中需要遍历 62 个节点才能找到目标值；使用跳表则只需经过 11 个节点：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/skip-list-performance.png" alt="跳表性能" width="500" />
</div>

有序集合始终保持数据有序，因此比关系型数据库更适合本场景，代价是插入和查找操作的时间复杂度为 O(logN)。

相比之下，在关系型数据库中查找指定用户的排名，需要执行如下嵌套查询：

~~~sql
SELECT *,(SELECT COUNT(*) FROM leaderboard lb2
WHERE lb2.score >= lb1.score) RANK
FROM leaderboard lb1
WHERE lb1.user_id = {:user_id};
~~~

在 Redis 中需要哪些操作来维护排行榜？

- **ZADD**：用户不存在时将其加入集合；已存在时更新分数。时间复杂度 O(logN)。
- **ZINCRBY**：按指定分值增加用户得分。如果用户不存在，则从 0 分开始。时间复杂度 O(logN)。
- **ZRANGE/ZREVRANGE**：按分数获取指定范围的用户。可以指定升序/降序、偏移量和结果数量。时间复杂度 O(logN+M)，其中 M 为结果数量。
- **ZRANK/ZREVRANK**：按升序/降序获取指定用户的名次。时间复杂度 O(logN)。

用户获得一分时会执行：

~~~redis
ZINCRBY leaderboard_feb_2021 1 'mary1934'
~~~

每个月都会创建一个新的排行榜，旧排行榜则迁移到历史存储。

用户获取排行榜前 10 名时会执行：

~~~redis
ZREVRANGE leaderboard_feb_2021 0 9 WITHSCORES
~~~

返回结果示例：

~~~text
[(user2,score2),(user1,score1),(user5,score5)...]
~~~

如果用户想查看自己在排行榜中的位置呢？

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/leaderboard-position-of-user.png" alt="用户在排行榜中的位置" width="500" />
</div>

假设我们知道用户在排行榜中的位置，可以通过以下查询轻松获取其附近的玩家：

~~~redis
ZREVRANGE leaderboard_feb_2021 357 365
~~~

可以使用 `ZREVRANK <user-id>` 获取用户的名次。

估算一下存储需求：

- 假设最坏情况下，2500 万 MAU 都参加当月比赛。
- ID 是 24 字符串，分数是 16 位整数，则每条记录需要 26 字节，2500 万条约需要 650 MB。
- 即使考虑跳表开销导致存储成本翻倍，这些数据也能轻松放入现代 Redis 集群。

另一个非功能性需求是支持每秒 2500 次更新。从处理能力来看，单个 Redis 服务器完全可以胜任。

其他需要注意的事项：

- 可以启动 Redis 副本，避免 Redis 服务器崩溃时丢失数据。
- 还可以利用 Redis 持久化机制，进一步降低崩溃导致的数据丢失风险。
- 需要两张 MySQL 辅助表：一张保存用户名、显示名等用户资料，另一张记录用户赢得比赛等事件。
- 第二张 MySQL 表可以在基础设施故障时用于重建排行榜。
- 作为小型性能优化，可以缓存前 10 名玩家的用户资料，因为这些资料会被频繁访问。

---

## 第三步：深入设计

### **使用云服务还是自行管理**

可以选择自行部署和管理服务，也可以使用云服务商代为管理。

如果自行管理，可以使用 Redis 存储排行榜数据，使用 MySQL 存储用户资料；如果需要扩展数据库，还可以为用户资料加入缓存：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/manage-services-ourselves.png" alt="自行管理服务" width="500" />
</div>

另一种方案是使用云服务托管大部分组件。例如，可以使用 AWS API Gateway 将 API 调用路由到 AWS Lambda 函数：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/api-gateway-mapping.png" alt="API 网关映射" width="500" />
</div>

AWS Lambda 允许我们运行代码，而无需自行管理或配置服务器。它只在需要时运行，并且可以自动扩展。

用户获得分数的示例：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/user-scoring-point-lambda.png" alt="Lambda 处理得分更新" width="500" />
</div>

用户获取排行榜的示例：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/user-retrieve-leaderboard.png" alt="用户获取排行榜" width="500" />
</div>

Lambda 是一种无服务器架构实现方式，我们无需管理扩缩容和环境配置。

如果从头开始构建这款游戏，作者建议采用这种方式。

### **扩展 Redis**

在 500 万 DAU 的规模下，从存储和 QPS 两方面看，单个 Redis 实例就足够了。

但如果用户增长 10 倍，达到 5 亿 DAU，则需要 65 GB 存储，QPS 也会增长到 25 万。

这种规模需要分片。

一种方式是按范围对数据分区：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/range-partition.png" alt="范围分区" width="500" />
</div>

在这个示例中，我们根据用户分数进行分片，并在应用代码中维护 user_id 到分片的映射。映射本身可以存放在 MySQL 或另一个缓存中。

获取前 10 名玩家时，只需查询分数最高的分片（`[900-1000]`）。

获取用户排名时，需要先计算用户在所属分片内的名次，再加上其他分片中分数更高的所有用户数。后者是 O(1) 操作，因为可以使用 info keyspace 命令快速获取每个分片的记录总数。

另一种方式是使用 Redis Cluster 进行哈希分区。Redis Cluster 是一种代理机制，会根据分区策略（类似但不完全等同于一致性哈希）将数据分布到不同 Redis 节点：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/hash-partition.png" alt="哈希分区" width="500" />
</div>

这种方式下计算前 10 名玩家会比较困难。需要先从每个分片获取前 10 名，再由应用程序合并结果：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/top-10-players-calculation.png" alt="计算排行榜前十名" width="500" />
</div>

哈希分区有一些限制：

- 如果要获取前 K 名且 K 很大，由于需要从所有分片获取大量数据，延迟可能会上升。
- 分区数量越多，延迟越高。
- 没有直接的方法确定某个用户的排名。

因此，作者倾向于在本问题中采用固定分区。

其他需要注意的事项：

- 最佳实践是为写入密集型 Redis 节点预留所需内存两倍的空间，以便在需要时保存快照。
- 可以使用 Redis-benchmark 工具跟踪 Redis 配置的性能，并根据数据做出决策。

### **备选方案：NoSQL**

另一个方案是使用适合以下场景的 NoSQL 数据库：

- 写入量大。
- 能够高效地在同一个分区内按分数排序。

DynamoDB、Cassandra 和 MongoDB 都适合考虑。

本章选择 DynamoDB。它是全托管 NoSQL 数据库，具有可靠的性能和良好的可扩展性；需要查询非主键字段时，也可以使用全局二级索引。

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/dynamo-db.png" alt="DynamoDB" width="500" />
</div>

先设计一张用于存储国际象棋游戏排行榜的表：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/chess-game-leaderboard-table-1.png" alt="国际象棋排行榜表一" width="500" />
</div>

这种设计可以正常工作，但如果需要按分数查询，就无法很好地扩展。因此，可以把分数设置为排序键：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/chess-game-leaderboard-table-2.png" alt="国际象棋排行榜表二" width="500" />
</div>

这种设计还有一个问题：按月份分区会造成热点分区，因为最新月份的访问量会远高于其他月份。

可以使用写入分片技术，为每个键附加一个分区编号。分区编号通过 `user_id % num_partitions` 计算：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/chess-game-leaderboard-table-3.png" alt="国际象棋排行榜表三" width="500" />
</div>

需要权衡分区数量：

- 分区越多，写入可扩展性越好。
- 但读取可扩展性会下降，因为聚合结果时需要查询更多分区。

使用这种方式，需要采用前文介绍的 scatter-gather 技术；随着分区数量增加，处理时间复杂度也会增加：

<div style="margin-left:3rem">
    <img src="../../25.%20Real-time%20Gaming%20Leaderboard/images/scatter-gather-2.png" alt="Scatter-gather 查询" width="500" />
</div>

要合理评估分区数量，需要进行基准测试。

这个 NoSQL 方案仍有一个主要缺点：难以计算某个用户的具体名次。

如果系统规模大到需要分片，也许可以改为告诉用户他们的分数处于哪个“百分位”。

可以定期运行 CRON 任务分析分数分布，并据此确定用户所在的百分位，例如：

~~~text
10th percentile = score < 100
20th percentile = score < 500
...
90th percentile = score < 6500
~~~

---

## 第四步：总结

如果时间允许，还可以讨论：

- **更快获取用户资料**：可以用 Redis 哈希缓存用户对象，映射关系为 `user_id -> user object`。这样比查询数据库更快。
- **打破平局**：两位玩家分数相同时，可以按最近一场比赛的时间排序。
- **系统故障恢复**：如果 Redis 大规模故障，可以遍历 MySQL WAL 条目，并通过一次性脚本重建排行榜。
