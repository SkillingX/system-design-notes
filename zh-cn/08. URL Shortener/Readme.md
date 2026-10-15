# 第 8 章：设计 URL 短链服务

## 引言
本章讨论如何设计类似 TinyURL 的 URL 缩短服务。系统的主要目标包括**缩短 URL、重定向**，以及能够应对大量流量的**高可扩展性**。

### 需求
- 短 URL 必须**唯一且尽可能短**。
- 每天处理 **1 亿次 URL 生成**，并支持 10 年。
- 高效处理读操作，读写比为 10:1。
- 存储 3,650 亿条记录；10 年约需要 **365 TB** 存储空间。

---

## 第 1 步：高层设计

### API 端点
1. **缩短 URL：**
   - 端点：`POST api/v1/data/shorten`
   - 参数：`{longUrl: longURLString}`
   - 返回：`shortURL`

2. **重定向 URL：**
   - 端点：`GET api/v1/shortUrl`
   - 返回用于重定向的 `longURL`。

    <p align="center">
    <img src="../../08.%20URL%20Shortener/images/url-redirection.png" alt="URL Redirection" width="600">
    </p>

### URL 重定向
- <strong>301 重定向：</strong>301 重定向表示请求的 URL 已“永久”迁移到长 URL。浏览器会缓存响应；之后再次请求同一 URL 时，就不会再访问 URL 短链服务。
- <strong>302 重定向：</strong>表示临时重定向，适合用于点击跟踪等分析场景。

### URL 缩短
<p align="center">
    <img src="../../08.%20URL%20Shortener/images/url-shortening.png" alt="URL Shortening" width="400">
</p>

- 使用**哈希函数**生成短 URL，将长 URL 映射到唯一的短 URL。
- 哈希函数需要满足以下要求：
    - 每个 `longURL` 都必须对应一个 `hashValue`。
    - 可以通过每个 `hashValue` 找回对应的 `longURL`。

---

## 第 2 步：深入设计

### 数据模型
在关系型数据库中存储 `<shortURL, longURL>` 映射，以优化内存使用。表结构包含：
- `id`（主键）
- `shortURL`
- `longURL`

    <img src="../../08.%20URL%20Shortener/images/table-schema.png" alt="Table Schema" width="300">

### 哈希函数
#### 1. Base 62 转换
- 使用 `[0-9, a-z, A-Z]` 中的字符对数字进行编码，共有 **62 个字符**。
- Base 转换是 URL 短链服务中常用的另一种方案。
- 可以为短 URL 分配唯一 ID，再将 ID 转换为 Base 62，得到短 URL。
- 7 个字符的哈希值最多支持 **3.5 万亿个唯一 URL**，足以容纳 3,650 亿个 URL。

**示例：**
将 ID `2009215674938` 转换为 Base 62：
- `2009215674938` → `zn9edcu`。

#### 2. 哈希与冲突处理
- 可以使用 CRC32、MD5 或 SHA-1 等哈希函数。

    <img src="../../08.%20URL%20Shortener/images/hash-function.png" alt="Hash Function" width="500">

- 一种做法是截取哈希值的前 7 个字符，但这可能造成哈希冲突。
- 为解决冲突，可以递归地追加新的预定义字符串，直到不再冲突；但这种方式开销可能很高。
- 也可以使用 **Bloom Filter** 高效查找冲突。

    <p align="center">
    <img src="../../08.%20URL%20Shortener/images/url-lookup.png" alt="URL Lookup" width="500">
    </p>

### 方案比较

- **哈希与冲突处理：**
    - 短 URL 长度固定。
    - 不需要唯一 ID 生成器。
    - 可能发生冲突，需要额外处理。
    - 由于不依赖 ID，无法找到下一个可用的短 URL。

- **Base 62 转换：**
    - 长度不固定，会随 ID 增长。
    - 需要唯一 ID 生成器。
    - 不会发生冲突。
    - 如果 ID 每次递增 1，就容易找到下一个短 URL（可能带来安全问题）。

---

### URL 缩短流程

<p align="center">
    <img src="../../08.%20URL%20Shortener/images/url-shortening-flow.png" alt="URL Shortening" width="500">
</p>

1. 检查数据库中是否存在 `longURL`。
2. 如果存在，返回已有的 `shortURL`。
3. 否则：
   - 使用**分布式 ID 生成器**生成唯一 ID。
   - 使用 Base 62 将 ID 转换为 `shortURL`。
   - 在数据库中保存 `<id, shortURL, longURL>` 映射。

---

### URL 重定向流程
<p align="center">
    <img src="../../08.%20URL%20Shortener/images/url-redirecting-flow.png" alt="URL Shortening" width="600">
</p>

1. 用户点击 `shortURL`。
2. 查询 `<shortURL, longURL>` 映射：
   - 优先查询**缓存**，以加快访问。
   - 如果缓存中没有，再查询数据库。
3. 将用户重定向到 `longURL`。

---

## 其他注意事项
### 限流器
按 IP 限制请求数，防止服务被滥用。

### 可扩展性
1. <strong>Web 层：</strong>保持无状态，通过增删 Web 服务器进行扩展。
2. <strong>数据库层：</strong>使用复制和分片。

### 分析
收集点击率、来源和时间戳等数据，为业务分析提供依据。

### 高可用性与可靠性
通过数据库复制和容错设计，保证服务一致且可靠。
