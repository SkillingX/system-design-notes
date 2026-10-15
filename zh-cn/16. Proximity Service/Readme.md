# 第 16 章：设计附近服务

## 引言
**附近服务**用于查找附近的地点，例如餐厅、酒店、加油站和其他商家。**Google Maps** 和 **Yelp** 等应用使用此类功能，帮助用户查找指定半径内的地点。

## 第 1 步：理解问题并明确范围

### **功能需求**
1. 根据用户位置（纬度、经度）和搜索半径查找商家。
2. 商家可以添加、更新或删除商家信息（不要求实时生效）。
3. 用户请求时返回商家的详细信息。

### **非功能需求**
- <strong>低延迟：</strong>用户应能快速获得响应。
- <strong>数据隐私：</strong>遵守 GDPR 和 CCPA 法规。
- <strong>高可用性：</strong>能够应对繁忙地点在高峰时段出现的流量尖峰。

### **数量级估算**
- 日活跃用户 **1 亿**。
- 系统中有 **2 亿家商户**。
- **搜索 QPS 估算：**
  - 每位用户每天搜索 **5 次**。
  - **搜索 QPS** = (100M × 5) / 86,400 ≈ **5,000 QPS**。

---

## 第 2 步：高层设计

### **API 设计**
#### **搜索附近商家**
`GET /v1/search/nearby`

- **请求参数：**
  - `latitude`：用户所在位置的纬度。
  - `longitude`：用户所在位置的经度。
  - `radius`：搜索半径（默认：5000m）。

#### **商家 API**
| API 端点 | 说明 |
|-----------------------------------|--------------------------------------------------|
| `GET /v1/businesses/{id}` | 获取商家详细信息 |
| `POST /v1/businesses` | 添加新商家 |
| `PUT /v1/businesses/{id}` | 更新商家信息 |
| `DELETE /v1/businesses/{id}` | 从系统中移除商家 |

### **数据模型**
- 两项功能都很常用，读取量较高，因此 MySQL 等关系型数据库比较适合：
  - 搜索附近商家。
  - 查看商家的详细信息。

### **数据 Schema**
- 主要数据库表包括商家表和地理空间索引表。
- 商家表包含商家的详细信息。

### **高层系统架构**
系统由两部分组成：基于位置的服务（LBS）和商家相关服务。

<div style="margin-left:3rem">
    <img src="../../16.%20Proximity%20Service/images/high-level-design.png" alt="HLD" width="400" />
</div>

- **基于位置的服务（LBS）：**
  - 处理基于位置的搜索请求。
  - 这是一个读多写少的服务，不处理写请求。
  - QPS 较高，在人口密集地区尤其是高峰时段；系统本身是无状态的。
- <strong>商家服务：</strong>处理两种请求：
  - 商家所有者创建、更新或删除商家信息。
  - 顾客查看商家的详细信息。
- <strong>负载均衡器：</strong>将流量路由到 LBS 和商家服务。
- **数据库集群：**
  - 针对读多写少的工作负载使用**主副本架构**。
  - LBS 读取的数据与主数据库写入的数据之间可能存在差异。
  - 商家信息不要求实时更新，因此这种不一致通常可以接受。

---

## 第 3 步：查找附近商家的算法

### **方案 1：二维搜索（朴素方案）**

<div style="margin-left:3rem">
    <img src="../../16.%20Proximity%20Service/images/2d-search.png" alt="2D" width="250" />
</div>

最直观的办法是画出一个指定半径的圆，然后找出圆内所有商家。

**SQL 查询：**
```sql
SELECT business_id, latitude, longitude
FROM business
WHERE (latitude BETWEEN :lat - radius AND :lat + radius)
AND (longitude BETWEEN :long - radius AND :long + radius);
```

**问题：**
- <strong>效率低：</strong>需要扫描整个数据库。
- <strong>受一维索引限制：</strong>纬度和经度是两个维度。

一种潜在改进方法是为经度和纬度列建立索引。虽然性能会略有提升，但仍然很慢。

### 更好的方案
- 上一种方案的问题在于，数据库索引只能加快一个维度上的搜索。
- 更合适的方案是使用地理空间索引将二维数据表示为一维数据。
  - 哈希类：均匀网格、GeoHash。
  - 树类：四叉树、Google S2、RTree。

  <div style="margin-left:3rem">
    <img src="../../16.%20Proximity%20Service/images/geospatial-index-types.png" alt="Geospatial index types" width="500" />
  </div>

### **方案 2：均匀划分网格**

  <div style="margin-left:3rem">
    <img src="../../16.%20Proximity%20Service/images/even-grid.png" alt="Even Grid" width="400" />
  </div>

- **将世界划分为固定大小的网格。**
- <strong>问题：</strong>商家分布不均（城市密集，农村稀疏）。

### **方案 3：GeoHash**
- 以赤道和本初子午线为界，将地球划分为四个象限，然后将每个网格继续划分为四个更小的网格。
- 每个网格可以通过交替处理经度和纬度的位来表示。
- 持续重复上述细分过程。

  <div style="margin-left:3rem">
    <img src="../../16.%20Proximity%20Service/images/geohash.png" alt="Geohash" width="300" />
    <img src="../../16.%20Proximity%20Service/images/geohash-1.png" alt="Geohash" width="285" />
  </div>

- <strong>将纬度和经度编码为一个字母数字字符串。</strong>GeoHash 有 12 个精度级别。
- **分层网格结构**可以高效搜索。
- 根据下表，选择满足搜索半径要求的最短 GeoHash 长度。
  <div style="margin-left:3rem">
    <img src="../../16.%20Proximity%20Service/images/geohash-radius-mapping.png" alt="Geohash Radius" width="400" />
  </div>
- GeoHash 保证两个值的共享前缀越长，它们的位置越接近。

- **挑战：**
  <div style="margin-left:3rem">
    <img src="../../16.%20Proximity%20Service/images/boundary-issue.png" alt="Boundary Issue" width="300" />
  </div>

  - <strong>网格边界问题：</strong>靠近网格边缘的商家可能会被漏掉。
    - 两个位置可能非常接近，但完全没有共同前缀，例如分别位于赤道两侧。
    - 两个位置可能有较长的共同前缀，但属于不同的 GeoHash 网格。
  - 解决办法：还需要搜索相邻网格。

### **方案 4：四叉树**

四叉树是一种树形数据结构，会递归地把二维空间划分为四个象限。每个内部节点恰有四个子节点，分别代表该空间的四个子区域。
- 四叉树是内存数据结构，在每台 LBS 服务器上运行，并在服务器启动时构建。

  <div style="margin-left:3rem">
    <img src="../../16.%20Proximity%20Service/images/quadtree.png" alt="Quadtree" width="500" />
  </div>

- 根节点会递归地拆分为四个象限，直到每个节点包含的商家不超过 x 个（本例中为 100 个）。

  <div style="margin-left:3rem">
    <img src="../../16.%20Proximity%20Service/images/building-quadtree.png" alt="Building Quadtree" width="500" />
  </div>

- 四叉树索引占用的内存不多（通常为 GB 级），可以轻松放入一台服务器。
- 构建树的时间复杂度为 nlogn，可能需要几分钟。
- **适合 k 近邻查询**，例如查找最近的加油站。

  <div style="margin-left:3rem">
    <img src="../../16.%20Proximity%20Service/images/realworld-quadtree.png" alt="Real World Quadtree" width="400" />
  </div>

#### 运维考量
- 对于大约 2 亿家商户，服务器启动时构建四叉树可能需要几分钟。
- 构建期间无法处理流量，因此新版本应逐步发布到一部分服务器。
- 更新或添加商家时，最简单的方法是增量重建四叉树，但这会导致大量缓存失效。
- 也可以直接更新四叉树，但实现更复杂，需要锁机制。

### **方案 5：Google S2**
Google S2 使用 Hilbert 曲线将球面映射为一维索引。在 Hilbert 曲线上彼此接近的两点，在一维空间中也会彼此接近。

  <div style="margin-left:3rem">
    <img src="../../16.%20Proximity%20Service/images/hilbert-curve.png" alt="Hilbert curve" width="300" />
    <img src="../../16.%20Proximity%20Service/images/geofence.png" alt="Geofence" width="355" />
  </div>

- **使用 Hilbert 曲线将地球划分为小网格。**
- 适用于地理围栏，因为它可以用不同层级覆盖任意形状的区域。
- 地理围栏还允许定义包围目标区域的参数。
- 相比固定精度级别，S2 的另一项优点是可以指定最小、最大级别和最大网格数。

## 方案权衡比较

#### GeoHash
- 易于使用和实现，无需构建或重建树。
- 支持固定半径的结果。
- 更新索引简单。
- 无法根据人口密度动态调整网格大小。

#### 四叉树
- 实现稍复杂。
- 支持查找 k 个最近的商家。
- 可以根据人口密度动态调整网格大小。
- 更新索引较复杂，有时需要重建整棵树。

---

## 第 4 步：扩展数据库并设计缓存策略

### **扩展商家表**
- 按 **business ID** 分片，可以让数据均匀分布。
- 表中每家商户对应一行。

| GeoHash | Business ID |
|---------|------------|
| 9q9hvu  | 343        |
| 9q9hvu  | 347        |
| 9q9hvu  | 112        |

### **扩展地理空间索引**
- GeoHash 表可能不适合分片。在这个场景中，所有数据都可以放进一台服务器，因此没有技术上的分片理由。
- 更好的方案是设置只读副本来分担读取负载。

### **缓存策略**
最直观的缓存键是位置坐标，但存在一些问题：
- GPS 坐标不准确。
- 用户移动时，位置坐标会变化。
- 更好的缓存键是 GeoHash。

| 缓存键 | 缓存值 |
|------------|------------|
| `geohash` | 该网格中的商家 ID 列表 |
| `business_id` | 商家详情（名称、地址、评价等） |

---

## 第 5 步：部署策略与最终架构

### **区域和可用区**
- 在多个区域部署 LBS 和商家服务。

### **处理实时更新**
- 商家信息每天批量更新一次。

### **最终系统架构**

  <div style="margin-left:3rem">
    <img src="../../16.%20Proximity%20Service/images/final-design.png" alt="Final Design" width="500" />
  </div>

最终算法如下：

## 查找附近商家的步骤
1. **用户请求：**
   - 用户搜索 500 米内的餐厅。
   - 客户端将<strong>纬度（37.776720）、经度（-122.416730）和半径（500m）</strong>发送给**负载均衡器**。
2. **转发请求：**
   - <strong>负载均衡器（LB）</strong>将请求转发给**基于位置的服务（LBS）**。
3. **计算 GeoHash：**
   - LBS 根据搜索半径确定 **GeoHash 长度**。
   - 查阅参考表，得出 **500m 对应 GeoHash 长度为 6**。
4. **查找相邻 GeoHash：**
   - LBS 计算相邻的 GeoHash，以包含附近区域。
   - 结果列表如下：
     ```
     [my_geohash, neighbor1_geohash, neighbor2_geohash, ..., neighbor8_geohash]
     ```
5. **从 Redis 获取商家 ID：**
   - 对列表中的每个 GeoHash，LBS 查询 **GeoHash Redis 服务器**以获取商家 ID。
   - 并行查询以尽可能降低延迟。
6. **获取并排序商家：**
   - LBS 从**商家信息 Redis 服务器**获取商家详情。
   - 按照与用户位置之间的距离排序。
   - 将排序后的结果返回给客户端。

## 关键优化
- <strong>并行调用 Redis：</strong>缩短响应时间。
- <strong>GeoHash 索引：</strong>实现高效的空间查询。
- <strong>缓存：</strong>加快数据查找和读取。

这种方法可以低延迟、可扩展地查找用户附近的商家。

---

### **选择最佳索引方法**
| 索引方法 | 优点 | 缺点 |
|----------------|------|------|
| **GeoHash** | 易于实现，适用于附近搜索 | 存在边界问题，网格大小固定 |
| **四叉树** | 根据密度动态调整，支持 k 近邻查询 | 更复杂，需要重新平衡树 |
| **Google S2** | 地理围栏功能先进，Google Maps 正在使用 | 实现难度较高 |

---

## 参考资料
1. [GeoHash 算法](https://www.movable-type.co.uk/scripts/geohash.html)
2. [四叉树索引](https://en.wikipedia.org/wiki/Quadtree)
3. [Google S2 Geometry](https://s2geometry.io/)
