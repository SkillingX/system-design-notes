# 第 7 章：设计分布式系统中的唯一 ID 生成器

## 引言
本章讨论如何为分布式系统设计**唯一 ID 生成器**。传统的自增主键不适用于分布式环境，因为它会带来可扩展性和同步方面的挑战。我们需要生成唯一、可按时间排序的 64 位数字 ID，并满足以下要求：
- ID 必须**唯一，并按日期排序**。
- ID 必须能够放入 **64 位**。
- 系统每秒应能生成 **超过 10,000 个 ID**。

---

## 第 1 步：理解问题
### 基本需求
- ID 必须唯一、为数字，并且不超过 64 位。
- ID 随时间递增，但不要求每次都严格 `+1`。
- ID 应能按日期排序。
- 系统必须支持高吞吐量（每秒 10,000 个 ID）。

---

## 第 2 步：高层设计方案
### 1. 多主复制
- <strong>方案：</strong>使用数据库 `auto_increment`，并按步长递增，例如每台服务器使用 `+k`，其中 k 为服务器数。

    <p align="left">
    <img src="../../07.%20Unique-Id%20Generator/images/multi-master.png" alt="Multi Master" width="400">
    </p>

- **缺点：**
  - 跨数据中心扩展困难。
  - ID 不一定随时间递增。
  - 增加或移除服务器时存在扩展问题。

### 2. UUID（通用唯一标识符）
- **方案：**
    - 使用 UUID，在每台服务器上独立生成 128 位唯一标识符。
    - 各服务器可以独立生成 UUID，无需彼此协调。

        <p align="left">
        <img src="../../07.%20Unique-Id%20Generator/images/uuid.png" alt="UUID generator" width="600">
        </p>

- **优点：**
  - 服务器之间无需协调。
  - 可以随 Web 服务器轻松扩展。
- **缺点：**
  - 超出 64 位的要求。
  - ID 不能按时间排序，而且不一定是数字。

### 3. Ticket Server
- <strong>方案：</strong>使用一个中心化数据库服务器递增并分配 ID。

    <p align="left">
    <img src="../../07.%20Unique-Id%20Generator/images/ticket-server.png" alt="UUID generator" width="500">
    </p>

- **优点：**
  - 小规模系统中实现简单。
  - 生成数字 ID。
- **缺点：**
  - 存在单点故障。
  - 多服务器环境下有同步难题。

### 4. Twitter Snowflake 方案
- **方案：**

    <div style="margin-left:3rem">
      <img src="../../07.%20Unique-Id%20Generator/images/twitter-snowflake.png" alt="Snowflake approach" width="500">
    </div>
    <div style="margin-left:3rem">
      <img src="../../07.%20Unique-Id%20Generator/images/snowflake-id-breakdown.png" alt="Snowflake ID breakdow" width="500">
    </div>

    - 将 ID 划分为多个部分，以保证唯一性和可扩展性。
    - <strong>符号位（1 位）：</strong>始终为 `0`，可用于区分有符号数和无符号数。
    - <strong>时间戳（41 位）：</strong>自定义纪元以来经过的毫秒数（Twitter 默认纪元为 `1288834974657`，对应 2010 年 11 月 4 日 01:42:54 UTC），确保 ID 按时间排序。
    - <strong>数据中心 ID（5 位）：</strong>最多可标识 `2^5 = 32` 个数据中心。
    - <strong>机器 ID（5 位）：</strong>每个数据中心内最多可标识 `2^5 = 32` 台机器。
    - <strong>序列号（12 位）：</strong>记录同一毫秒内一台机器生成的 ID 数量，每毫秒最多支持 `2^12 = 4096` 个 ID。序列号每毫秒重置为 `0`。

- **优点：**
    - <strong>可扩展性：</strong>多台服务器合计每秒可处理 10,000 个以上的 ID。
    - <strong>按时间排序：</strong>ID 可以按时间排序。
    - <strong>去中心化：</strong>不存在单点故障。

## 第 4 步：其他注意事项
### 1. 时钟同步
- <strong>挑战：</strong>生成 ID 的方案假设各服务器的时钟同步。
- <strong>解决方案：</strong>使用<strong>网络时间协议（NTP）</strong>尽量减小时钟偏差。

### 2. 调整各部分的位数
根据使用场景调整各部分大小，例如减少序列号位数、增加时间戳位数。

### 3. 高可用性
- ID 生成器是关键系统组件，必须具备容错能力。
- 考虑冗余和故障切换机制。
