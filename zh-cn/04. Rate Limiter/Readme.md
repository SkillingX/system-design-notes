# 第 4 章：设计限流器

## 引言
本章探讨限流器的设计与实现。限流器是一种用于控制客户端或服务流量速率的系统组件，有助于防止滥用、降低成本并保证服务器资源稳定。常见场景包括限制发帖、创建账号和领取奖励的频率。

## 限流的好处
- **防止 DoS 攻击：**阻止过量调用，避免资源耗尽。
- **降低成本：**减少不必要的请求，从而降低服务器开销。
- **防止过载：**过滤过多请求，让服务器性能保持稳定。

## 第 1 步：理解问题
### 主要功能
- 在服务器端对 API 限流。
- 支持多种限流规则。
- 支持分布式环境中的大规模系统。
- 可以作为独立服务，也可以集成在应用代码中。
- 被限流时向用户说明原因。

### 需求
- 准确地限制请求速率。
- 延迟低。
- 内存占用低。
- 支持分布式部署。
- 清晰地处理异常。
- 具有高容错能力。

## 第 2 步：高层设计
### 部署位置选项
<div style="margin-left:2rem">
    <img src="../../04.%20Rate%20Limiter/images/rate_limiter_architecture.png" alt="Rate Limiting Middleware Architecture" width="550">
</div>

1. **客户端实现：**不可靠，因为客户端可能被滥用。
2. **服务器端实现：**控制力和可靠性更好，是首选方案。
3. **中间件（API Gateway）：**将限流集成到请求入口中，方案灵活。

### 部署位置的选择原则
- 评估当前技术栈，选择高效方案。
- 根据业务需求选择合适的算法。
- 使用微服务时，可以考虑 API Gateway。
- 资源有限时，可以考虑商业化解决方案。

## 第 3 步：限流算法
### 1. 令牌桶
<div style="margin-left:2rem">
  <img src="../../04.%20Rate%20Limiter/images/token-bucket.png" alt="Token Bucket Algorithm" width="550">
</div>

- **说明：**以固定速率向桶中补充令牌；每个请求消耗一个令牌。
- **参数：**桶的容量和令牌补充速率。
- **优点：**容易实现、内存效率高，并支持流量突发。
- **缺点：**需要谨慎调整参数。

### 2. 漏桶
<div style="margin-left:2rem">
  <img src="../../04.%20Rate%20Limiter/images/leaking-bucket.png" alt="Leaking Bucket Algorithm" width="550">
</div>

- **说明：**使用 FIFO 队列按固定速率处理请求。
- **优点：**内存效率高，输出速率稳定。
- **缺点：**突发流量可能导致新请求延迟处理。

示例：https://github.com/uber-go/ratelimit

### 3. 固定窗口计数器
<div style="margin-left:2rem">
  <img src="../../04.%20Rate%20Limiter/images/fixed-window-counter.png" alt="Fixed Window Counter" width="550">
</div>

- **说明：**将时间划分为固定区间，并使用计数器限制请求数。
- **优点：**简单高效，适用于特定场景。
- **缺点：**窗口边界处的流量峰值可能突破限额。

如果在时间窗口边界突然出现流量突发，实际放行的请求数可能超过配额。

<img src="../../04.%20Rate%20Limiter/images/fixed-window-issue.png" alt="Fixed Window Issue" width="550">

### 4. 滑动窗口日志
<div style="margin-left:2rem">
  <img src="../../04.%20Rate%20Limiter/images/sliding-window-log.png" alt="Sliding Window Log" width="550">
</div>

- **说明：**记录请求时间戳，以支持滚动时间窗口。
- **优点：**限流准确。
- **缺点：**内存消耗较高。

### 5. 滑动窗口计数器
<div style="margin-left:2rem">
  <img src="../../04.%20Rate%20Limiter/images/sliding-window-counter.png" alt="Fixed Window Counter" width="550">
</div>

- **说明：**结合固定窗口和滑动日志，以平滑流量峰值。
- **优点：**内存效率高，能够应对突发流量。
- **缺点：**采用近似计算，无法做到完全严格的限流。

## 高层架构
<div style="margin-left:2rem">
  <img src="../../04.%20Rate%20Limiter/images/architecture.png" style="margin-left: 40px; margin-top: 40px; margin-bottom: 20px;" alt="Architecture" width="550">
</div>

- **数据存储：**使用内存缓存（例如 Redis），快速执行计数器操作。
- **处理步骤：**
  1. 客户端向中间件发送请求。
  2. 中间件检查 Redis 中的计数器。
  3. 根据限额处理或拒绝请求。

## 进阶考量
### 分布式环境
- **挑战：**竞态条件和同步问题。
- **解决方案：**使用锁、Lua 脚本或 Redis 有序集合；也可以使用中心化数据存储来进行同步。

### 性能优化
- 使用多数据中心部署来降低延迟。
- 使用最终一致性模型进行同步。

### 监控
定期分析数据，评估算法效果，并根据需要调整规则。
