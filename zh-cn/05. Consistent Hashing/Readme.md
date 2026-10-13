# 第 5 章：设计一致性哈希

## 引言
本章介绍一致性哈希。这项技术对于实现横向扩展、将请求和数据高效地分配到多台服务器至关重要。增加或移除服务器时，一致性哈希可以减少数据重新分配的范围，并帮助均衡数据分布，缓解服务器热点问题。

## 重新哈希问题
### 说明
在传统哈希方法中，例如 `serverIndex = hash(key) % N`，服务器数量发生变化时，数据重新分配会变得棘手。例如：
- 移除服务器后，大部分键都需要重新分配，造成缓存未命中。
- 增加服务器时，也会引起不必要的键重新分配。

<img src="../../05.%20Consistent%20Hashing/images/server-hashing.png" alt="Server hashing" width="450">

- 当服务器池大小固定时，这种方法效果不错。但如果新增服务器或移除现有服务器，就会出现问题。

<img src="../../05.%20Consistent%20Hashing/images/server-hashing-miss.png" alt="Server hashing Miss" width="450">

### 核心问题
服务器数量变化时，大部分键都要重新分配，导致效率低下并增加系统负载。

## 一致性哈希
### 定义
一致性哈希可以确保增加或移除服务器时，只需重新映射一部分键，从而减少干扰并提高可扩展性。

### 关键概念
1. **哈希空间与哈希环：**哈希空间构成一个连续的环，哈希值分布在 `0` 到 `2^160-1` 之间，例如使用 SHA-1 哈希函数。将两端连接起来，就得到一个环。
    <p align="center">
    <img src="../../05.%20Consistent%20Hashing/images/hash-ring.png" alt="Hash Ring" width="450">
    </p>

- 使用相同的哈希函数 f，根据服务器 IP 地址或名称将服务器映射到环上。

    <p align="center">
    <img src="../../05.%20Consistent%20Hashing/images/server-ring.png" alt="Server Ring" width="450">
    </p>

2. **查找服务器：**
- 从键在环上的位置开始顺时针查找，遇到的第一个服务器就是该键对应的服务器。

  <p align="center">
  <img src="../../05.%20Consistent%20Hashing/images/server-lookup.png" alt="Server Lookup" width="450">
  </p>

3. **增加和移除服务器：**
- 增加服务器时，只会重新分配附近的一部分键，这部分键会转移到新服务器上。

  <p align="center">
  <img src="../../05.%20Consistent%20Hashing/images/adding-server.png" alt="Adding Server" width="450">
  </p>

- 移除服务器只会影响它负责范围内的键。被移除服务器上的键会重新分配给顺时针方向的下一台服务器。

  <p align="center">
  <img src="../../05.%20Consistent%20Hashing/images/removing-server.png" alt="Removing Server" width="450">
  </p>

## 挑战与解决方案
### 基础方案的两个问题
1. **分区大小不均：**不同服务器负责的数据分区大小可能不同。
2. **键分布不均：**部分服务器可能会收到远多于其他服务器的键。

### 解决方案：虚拟节点
- 每台服务器在哈希环上由多个均匀分布的虚拟节点表示。
- 虚拟节点可以改善键的分布并平衡负载。虚拟节点数量越多，键分布通常越均衡，因为标准差会变小。

  <p align="center">
  <img src="../../05.%20Consistent%20Hashing/images/virtual-nodes.png" alt="Virtual Nodes" width="450">
  </p>

## 受影响的键
增加或移除服务器时：
- **增加服务器：**受影响的键位于新服务器与其前驱节点之间。
  下面的示例中，服务器 4 被加入哈希环。受影响范围从 s4（新加入的节点）开始，沿环逆时针移动，直到找到一台服务器（s3）。因此，s3 和 s4 之间的键需要重新分配给 s4。

  <p align="center">
  <img src="../../05.%20Consistent%20Hashing/images/server-addition.png" alt="Server Addition" width="450">
  </p>

- **移除服务器：**受影响的键位于被移除服务器与其前驱节点之间。下面的示例中，服务器 s1 被移除，受影响范围从 s1（被移除的节点）开始，沿环逆时针移动，直到找到一台服务器（s0）。因此，s0 和 s1 之间的键需要重新分配给 s2。

  <p align="center">
  <img src="../../05.%20Consistent%20Hashing/images/server-removed.png" alt="Server Removed" width="450">
  </p>

## 一致性哈希的优点
- **减少重新分配：**只需重新分配一部分键。
- **可扩展性：**支持横向扩展。
- **缓解热点：**均衡数据分布，避免服务器过载。

## 实际应用
- Amazon DynamoDB
- Apache Cassandra
- Discord
- Akamai CDN
- Maglev 负载均衡器
