# 第 18 章：Google Maps

## 引言
我们将设计一个简化版的 **Google Maps**。

关于 Google Maps 的一些事实：
- 于 2005 年推出。
- 提供多种服务，包括卫星影像、街道地图、实时交通状况和路线规划。
- 截至 2021 年，有 10 亿日活跃用户，覆盖全球 99% 的地区，每天更新 2,500 万条实时位置信息。

---

## 第 1 步：理解问题并明确设计范围

候选人与面试官之间的示例问答：
- **候选人：**我们要服务多少日活跃用户？
- **面试官：**10 亿 DAU。
- **候选人：**重点讨论哪些功能？
- **面试官：**位置更新、导航、ETA、地图渲染。
- **候选人：**道路数据规模有多大？我们能访问这些数据吗？
- **面试官：**我们从多个来源获得了道路数据，原始数据有 TB 级。
- **候选人：**需要考虑交通状况吗？
- **面试官：**需要，这样才能准确估算到达时间。
- **候选人：**需要支持步行、骑行、驾车等不同出行方式吗？
- **面试官：**需要。
- **候选人：**需要支持多站点路线吗？
- **面试官：**本次面试先不讨论。
- **候选人：**商家地点和照片呢？
- **面试官：**问得好，不过本次无需考虑。

我们聚焦三个关键功能：用户位置更新、包含 ETA 的导航服务、地图渲染。

### **非功能需求**
- **准确性：**不能给用户错误的路线指引。
- **流畅导航：**用户应能流畅地浏览地图。
- **数据和电池用量：**客户端应尽可能少用数据和电量，这对移动设备尤为重要。
- 还需要满足总体可用性和可扩展性要求。

### **地图基础知识**

开始设计前，需要先了解一些地图相关概念。

#### 定位系统

地球是一个绕轴旋转的球体。位置由纬度（南北方向）和经度（东西方向）定义：

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/partitioning-system.png" alt="partitioning-system" width="500" />
</div>

#### 从 3D 转换到 2D

将 3D 中的点转换到 2D 平面上的过程称为“地图投影”。

有多种投影方式，各有优缺点。几乎所有投影方式都会使实际几何形状产生失真。

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/map-projections.png" alt="map-projections" width="500" />
</div>

Google Maps 选择了经过修改的墨卡托投影，称为“Web Mercator”。

#### 地理编码

地理编码是将地址转换为地理坐标的过程。

反向过程称为“反向地理编码”。

一种实现方式是使用插值：利用多个来源的数据（例如 GIS），这些数据会将街道网络映射到地理坐标空间中。

#### Geohash

Geohash 是一种编码系统，可以将地理区域编码成由字母和数字组成的字符串。

它将世界看作一个展开的平面，并递归地将其细分为四个象限：

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/geohashing.png" alt="geohashing" width="500" />
</div>

#### 地图渲染

地图渲染通过瓦片实现。我们不会把整张地图渲染成一张巨大的自定义图片，而是将世界拆分为更小的瓦片。

客户端只下载所需瓦片，再像拼马赛克一样将它们拼起来。

不同缩放级别对应不同瓦片。客户端根据当前缩放级别选择合适的瓦片。

例如，缩小到查看整个世界时，只需要下载一张 256x256 的瓦片来表示整张世界地图。

#### 用于导航算法的道路数据处理

在大多数路径规划算法中，路口表示为节点，道路表示为边：

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/road-representation.png" alt="road-representation" width="500" />
</div>

大多数导航算法使用经过修改的 Dijkstra 或 A* 算法。

寻路性能会受到图规模的影响。为了支持大规模场景，不能将整个世界表示成一张图并在其上运行算法。

相反，我们使用类似地图瓦片的技术，将世界逐步划分为更小的图。

路由瓦片保存对相邻瓦片的引用，算法在遍历互相连接的瓦片时，可以拼接出更大的道路图：

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/routing-tiles.png" alt="routing-tiles" width="500" />
</div>

这种技术可以显著减少内存带宽用量，并且只加载给定起点和终点所需的瓦片。

不过，对于更长的路线，拼接小而详细的路由瓦片仍然会消耗大量时间和内存。因此，我们会提供不同详细程度的路由瓦片，算法根据目标位置选用合适的瓦片：

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/map-routing-hierarchical.png" alt="map-routing-hierarchical" width="500" />
</div>

### **数量级估算**

需要存储以下数据：
- 世界地图：根据需要存储的所有瓦片估算约为 70 PB；不过，非常相似的瓦片（例如大片沙漠地区）可以压缩。
- 元数据：占用空间很小，可以不纳入估算。
- 道路信息：以路由瓦片的形式保存。

导航请求的 QPS 估算：10 亿 DAU，每位用户每周使用 35 分钟，即每天总计 50 亿分钟。
假设 GPS 更新请求经过批量处理，系统约需支撑 20 万 QPS，峰值为 100 万 QPS。

---

## 第 2 步：提出高层设计并达成共识

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/high-level-design.png" alt="high-level-design" width="500" />
</div>

### **位置服务**

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/location-service.png" alt="location-service" width="500" />
</div>

该服务负责记录用户的位置更新：
- 每隔 `t` 秒发送一次位置更新。
- 位置数据流可以用于持续改进服务，例如提供更准确的 ETA、监控交通状况、检测道路封闭情况以及分析用户行为。

客户端无需每次都立即向服务器发送位置更新，可以先在客户端批量收集，再一次发送多条更新：

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/location-update-batches.png" alt="location-update-batches" width="500" />
</div>

即使进行了优化，Google Maps 规模的系统仍然会承受很大的负载。因此，可以使用针对大量写入优化的数据库，例如 Cassandra。

也可以使用 Kafka 高效地流式处理位置更新，以便后续分析。

位置更新请求示例：

```
POST /v1/locations
Parameters
  locs: JSON encoded array of (latitude, longitude, timestamp) tuples.
```

### **导航服务**

该组件负责在合理时间内找到 A 到 B 的快速路线（可以接受一定延迟）。路线不一定是最快的，但准确性很重要。

请求示例：

```
GET /v1/nav?origin=1355+market+street,SF&destination=Disneyland
```

响应示例：

```json
{
  "distance": {"text":"0.2 mi", "value": 259},
  "duration": {"text": "1 min", "value": 83},
  "end_location": {"lat": 37.4038943, "Ing": -121.9410454},
  "html_instructions": "Head <b>northeast</b> on <b>Brandon St</b> toward <b>Lumin Way</b><div style=\"font-size:0.9em\">Restricted usage road</div>",
  "polyline": {"points": "_fhcFjbhgVuAwDsCal"},
  "start_location": {"lat": 37.4027165, "lng": -121.9435809},
  "geocoded_waypoints": [
    {
       "geocoder_status" : "OK",
       "partial_match" : true,
       "place_id" : "ChIJwZNMti1fawwRO2aVVVX2yKg",
       "types" : [ "locality", "political" ]
    },
    {
       "geocoder_status" : "OK",
       "partial_match" : true,
       "place_id" : "ChIJ3aPgQGtXawwRLYeiBMUi7bM",
       "types" : [ "locality", "political" ]
    }
  ],
  "travel_mode": "DRIVING"
}
```

目前还没有考虑交通状况变化和重新规划路线，这些内容会在深入设计中讨论。

### **地图渲染**

由于地图瓦片数据集达到 PB 级别，客户端不可能在本地保存全部数据。

客户端需要根据自身位置和缩放级别按需从服务器获取瓦片。

什么时候获取新瓦片？用户缩放地图时，以及导航过程中即将进入新瓦片时。

如何向客户端提供地图瓦片？
- 可以动态构建瓦片，但这会给服务器造成巨大负载，也难以缓存。
- 根据 GeoHash 静态提供地图瓦片，客户端可以自行计算瓦片位置。瓦片可以静态存储，并通过 CDN 提供。

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/static-map-tiles.png" alt="static-map-tiles" width="500" />
</div>

CDN 可以让用户从距离最近的边缘节点（Point of Presence，POP）获取地图瓦片，以尽量降低延迟：

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/cdn-vs-no-cdn.png" alt="cdn-vs-no-cdn" width="500" />
</div>

确定地图瓦片的方式有以下选择：
- 可以在客户端计算地图瓦片对应的 GeoHash。如果采用这种方式，需要谨慎考虑长期方案，因为之后要求客户端更新会很困难。
- 也可以提供一个简单 API，由 API 代替客户端计算地图瓦片 URL，但会增加一次 API 调用。

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/map-tile-url-calculation.png" alt="map-tile-url-calculation" width="500" />
</div>

---

## 第 3 步：深入设计

### **数据模型**

讨论如何存储系统涉及的不同类型数据。

#### 路由瓦片

初始道路数据集来自多个来源。之后可以根据位置更新数据持续改进道路信息。

道路数据是非结构化的。我们会运行定期的离线处理流水线，将原始数据转换为应用需要的、基于图结构的路由瓦片。

这些瓦片不需要数据库功能，因此无需放入数据库。可以将其存储在 S3 对象存储中，并积极使用缓存。

还可以使用相关库，将邻接列表高效压缩为二进制文件。

#### 用户位置数据

用户位置数据对更新交通状况及开展其他分析很有用。

可以使用 Cassandra 存储这类数据，因为它适用于写入密集型场景。

示例数据行：

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/user-location-data-torw.png" alt="user-location-data-row" width="500" />
</div>

#### 地理编码数据库

该数据库保存纬度/经度坐标对与地点之间的键值映射。

由于读取频繁、写入较少，可以使用读取速度快的 Redis。

#### 预计算的世界地图图像

如前所述，我们会预先生成地图瓦片图像，并将其存储在 CDN 中。

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/precomputed-map-tile-image.png" alt="precomputed-map-tile-image" width="500" />
</div>

### **服务**

#### 位置服务

先详细讨论该服务的数据库设计以及用户位置的存储方式。

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/location-service-diagram.png" alt="location-service-diagram" width="500" />
</div>

可以使用 NoSQL 数据库来承载位置更新带来的大量写入负载。用户位置经常变化，收到新更新后旧位置很快就会过期，因此我们优先保证可用性，而不是一致性。

我们选择 Cassandra，因为它很符合这些要求。

要保存的数据行示例：

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/user-location-row-example.png" alt="user-location-row-example" width="500" />
</div>

- `user_id` 是分区键，可以快速获取某位用户的所有位置更新。
- `timestamp` 是聚簇键，用于按收到位置更新的时间排序保存数据。

我们还使用 Kafka 将位置更新流式发送给其他需要这些更新的服务：

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/location-update-streaming.png" alt="location-update-streaming" width="500" />
</div>

#### 渲染地图

地图瓦片保存在多个缩放级别。缩放级别最低时，整个世界由一张 256x256 的瓦片表示。

缩放级别每增加一级，地图瓦片数量就会变成原来的四倍：

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/zoom-level-increases.png" alt="zoom-level-increases" width="500" />
</div>

一种优化方式是不通过网络传输整张图片，而是将瓦片表示为矢量（路径和多边形），再由客户端动态渲染瓦片。

这样可以显著节省带宽。

#### 导航服务

该服务负责查找最快路线：

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/navigation-service.png" alt="navigation-service" width="500" />
</div>

下面逐一介绍这个子系统中的组件。

首先，地理编码服务会将地址解析为纬度/经度坐标。

请求示例：

```
https://maps.googleapis.com/maps/api/geocode/json?address=1600+Amphitheatre+Parkway,+Mountain+View,+CA
```

响应示例：

```json
{
   "results" : [
      {
         "formatted_address" : "1600 Amphitheatre Parkway, Mountain View, CA 94043, USA",
         "geometry" : {
            "location" : {
               "lat" : 37.4224764,
               "lng" : -122.0842499
            },
            "location_type" : "ROOFTOP",
            "viewport" : {
               "northeast" : {
                  "lat" : 37.4238253802915,
                  "lng" : -122.0829009197085
               },
               "southwest" : {
                  "lat" : 37.4211274197085,
                  "lng" : -122.0855988802915
               }
            }
         },
         "place_id" : "ChIJ2eUgeAK6j4ARbn5u_wAGqWA",
         "plus_code": {
            "compound_code": "CWC8+W5 Mountain View, California, United States",
            "global_code": "849VCWC8+W5"
         },
         "types" : [ "street_address" ]
      }
   ],
   "status" : "OK"
}
```

路线规划服务根据当前交通状况计算建议路线，并以行程时间为优化目标。

最短路径服务会在对象存储中的路由瓦片上运行 A* 算法的变体，以计算最优路径：
- 接收起点/终点坐标，将地址转换为纬度/经度，并根据坐标推导 GeoHash，以定位路由瓦片。
- 算法从起点所在的路由瓦片开始遍历，直到找到一条通往终点瓦片的足够好路线。

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/shortest-path-service.png" alt="shortest-path-service" width="500" />
</div>

路线规划器会调用 ETA 服务，使用机器学习算法根据交通数据预测到达时间。

排序服务根据用户设置的过滤条件对多条候选路线进行排序，例如避开收费道路或高速公路。

更新服务会异步更新一些重要数据库，使其保持最新状态。

#### 改进：自适应 ETA 和重新规划路线

一种改进方式是根据最新交通数据，动态更新正在行驶中的路线。

一种实现方法是在数据库中保存当前沿路线导航的用户，以及他们将经过的所有瓦片。

数据可能如下所示：

```
user_1: r_1, r_2, r_3, …, r_k
user_2: r_4, r_6, r_9, …, r_n
user_3: r_2, r_8, r_9, …, r_m
...
user_n: r_2, r_10, r21, ..., r_l
```

如果某个瓦片发生交通事故，就可以找出路线经过该瓦片的所有用户，并为他们重新规划路线。

为了减少需要存储的瓦片数量，可以只保存起点路由瓦片，以及多个不同分辨率级别的若干路由瓦片，直到其中一个覆盖终点瓦片：

```
user_1, r_1, super(r_1), super(super(r_1)), ...
```

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/adaptive-eta-data-storage.png" alt="adaptive-eta-data-storage" width="500" />
</div>

采用这种方式，只要检查用户的终点瓦片是否包含交通事故所在瓦片，就能判断用户是否受影响。

还可以跟踪导航用户的所有可选路线，并在存在更快路线时通知用户。

#### 数据投递协议

服务器可以通过以下方式主动向客户端推送数据：
- 移动推送通知不适合，因为负载大小有限，而且 Web 应用无法使用。
- 与长轮询相比，WebSocket 通常更合适，因为服务器计算开销更低。
- 也可以使用服务器发送事件（SSE），但更倾向 WebSocket，因为它支持双向通信，未来可用于最后一公里配送等功能。

---

## 第 4 步：总结

最终设计如下：

<div style="margin-left:3rem">
    <img src="../../18.%20Google%20Maps/images/final-design.png" alt="final-design" width="500" />
</div>

还可以提供多站点导航功能，并将其出售给 Uber 或 Lyft 等企业客户，以帮助他们规划访问一组地点的最优路线。
