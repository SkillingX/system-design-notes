# 第 15 章：设计 Google Drive

## 引言
Google Drive 是一项基于云的文件存储和同步服务，用户可以从不同设备存储、访问和分享文件。本章讨论如何设计一个可扩展的系统，并支持以下功能：
- **文件上传和下载**
- **跨设备同步文件**
- **文件分享**
- **文件修订历史**
- **文件编辑、删除和分享通知**

---

## 第 1 步：理解问题

### 关键需求
#### 功能需求
- 上传和下载文件。
- 在多台设备之间同步文件。
- 保存文件修订版本。
- 支持带权限控制的文件分享。
- 文件被编辑、删除或分享时发送通知。

#### 非功能需求
- <strong>可靠性：</strong>不能丢失数据。
- <strong>快速同步：</strong>避免同步延迟让用户等待。
- <strong>带宽效率：</strong>尽量减少不必要的数据传输。
- <strong>可扩展性：</strong>支持 1,000 万日活跃用户（DAU）。
- <strong>高可用性：</strong>服务器故障或网络问题期间仍能持续运行。

### 限制条件与假设
- 每位用户有 **10 GB 免费空间**。
- 文件大小上限：**10 GB**。
- 平均文件上传大小：**500 KB**。
- 上传频率：每位用户每天 **2 个文件**。
- 总存储需求：**500 PB**。

---

## 第 2 步：高层设计
### 单服务器架构
基础架构包含：
1. <strong>Web 服务器：</strong>处理上传和下载。
2. <strong>元数据数据库：</strong>跟踪用户数据、登录信息和文件信息等元数据。
3. <strong>存储目录：</strong>按照 namespace 组织并保存文件。

<div style="margin-left:3rem">
    <img src="../../15.%20Google%20Drive/images/namespaces.png" alt="Namespaces" width="400" />
</div>

- 设置一台 Web 服务器，并将名为 `drive/` 的目录设为根目录，用来保存上传的文件。
- `drive/` 目录下包含多个名为 namespace 的目录。
- 每个 namespace 保存该用户上传的所有文件。
- 将 namespace 与相对路径拼接起来，就可以唯一标识一个文件或文件夹。

这种设计可以作为起点，但不足以支撑规模扩展。

#### API
1. <strong>向 Google Drive 上传文件：</strong>支持两种上传方式。
    - 简单上传：适用于文件较小的情况。
    - 可恢复上传：
        - 端点：`https://api.example.com/files/upload?uploadType=resumable`
        - 发送初始请求，获取可恢复上传 URL。
        - 上传数据并监控上传状态。
        - 如果上传中断，可以恢复上传。
2. **从 Google Drive 下载文件：**
    - 端点：`https://api.example.com/files/download`
3. **获取文件修订版本：**
    - 端点：`https://api.example.com/files/list_revisions`

### 迁移到分布式系统

#### 改进措施
1. <strong>分片：</strong>根据 `user_id` 将存储拆分到多台服务器。
2. <strong>Amazon S3：</strong>使用 S3 实现可扩展、冗余的文件存储，并跨区域复制。

    <img src="../../15.%20Google%20Drive/images/replication.png" alt="Replication" width="600" />

3. <strong>负载均衡器：</strong>将流量分配到多台 Web 服务器。
4. <strong>元数据数据库复制：</strong>通过数据库分片和复制确保可用性。

#### 同步冲突
像 Google Drive 这样的大型存储系统偶尔会发生同步冲突。当两位用户同时修改同一个文件或文件夹时，就会发生冲突。

<div style="margin-left:5rem">
<img src="../../15.%20Google%20Drive/images/sync-conflicts.png" alt="Sync Conflicts" width="600" />
</div>

- 示例中，用户 1 和用户 2 同时尝试更新同一个文件，但系统先处理了用户 1 的文件。
- 用户 1 的更新成功，用户 2 则遇到同步冲突。
- 系统会同时展示该文件的两个副本：用户 2 的本地副本和服务器上的最新版本。
- 用户 2 可以选择合并两个文件，或用其中一个版本覆盖另一个版本。

### 改进后的设计
<div style="margin-left:5rem">
<img src="../../15.%20Google%20Drive/images/high-level-design.png" alt="High Level Design" width="500" />
</div>

1. <strong>用户交互：</strong>用户通过浏览器或移动应用访问系统。
2. **数据块服务器：**
   - 将文件拆分为数据块，每块最大 **4 MB**，并为每个数据块分配唯一哈希值。
   - 数据块独立存储在云存储中，例如 Amazon S3。
   - 按照特定顺序拼接数据块即可重建文件。
3. <strong>云存储：</strong>将数据块保存在云存储中，以实现可扩展性和冗余。
4. <strong>冷存储：</strong>将不活跃的文件移到冷存储以降低成本。
5. <strong>负载均衡器：</strong>将请求均匀分配给 API 服务器，确保系统高效运行。
6. **API 服务器：**
   - 处理用户身份验证、资料管理和文件元数据更新。
   - 管理所有非上传流程。
7. **元数据数据库与缓存：**
   - 保存用户、文件、数据块和版本的元数据。
   - 缓存频繁访问的元数据，以加快读取。
8. **通知服务：**
   - 使用**发布订阅系统**通知客户端文件变更，例如添加、编辑和删除。
   - 确保客户端可以拉取最新更新。
9. <strong>离线备份队列：</strong>临时保存文件变更信息，离线客户端重新上线后可以同步。

---

## 第 3 步：深入设计

### 元数据数据库
下图是经过简化的版本，只展示最重要的数据表和字段。
#### Schema 设计
- <strong>用户表：</strong>保存用户资料和偏好。
- <strong>文件表：</strong>保存文件元数据，例如大小、名称和路径。
- <strong>数据块表：</strong>跟踪用于重建文件的数据块。
- <strong>文件版本表：</strong>保存文件修订历史。

<div style="margin-left:5rem">
<img src="../../15.%20Google%20Drive/images/metadata-database.png" alt="Metadata Database" width="500" />
</div>

---

### 文件上传流程

1. **文件上传：**
   - 数据块服务器将文件拆分为数据块，并进行压缩和加密。
   - 将数据块上传到数据块服务器，并保存在 S3 中。
2. **元数据上传：**
   - 客户端将元数据发送到 API 服务器。
   - 将元数据以 `pending` 状态保存到数据库。
3. **完成：**
   - S3 触发回调，将文件状态更新为 `uploaded`。
   - 通知服务通知相关用户。

<div style="margin-left:5rem">
<img src="../../15.%20Google%20Drive/images/upload-flow.png" alt="Upload Flow" width="500" />
</div>

---

### 文件同步
1. <strong>增量同步：</strong>只传输已修改的数据块，而不是整个文件。

    <div style="margin-left:2rem">
    <img src="../../15.%20Google%20Drive/images/delta-sync.png" alt="Delta Sync" width="400" />
    </div>

2. <strong>压缩：</strong>根据文件类型使用不同的压缩算法压缩数据块。
3. **冲突解决：**
   - 先处理的版本优先。
   - 将冲突版本分别保存，供用户处理。

<div style="margin-left:5rem">
<img src="../../15.%20Google%20Drive/images/file-sync.png" alt="File Sync" width="400" />
</div>

---

### 文件下载流程
如果文件在其他位置被添加或编辑，就会触发下载流程。客户端有两种方式获知变更：
- 如果文件被其他客户端修改时客户端 A 在线，通知服务会告知客户端 A。
- 如果文件被其他客户端修改时客户端 A 离线，数据会保存在缓存中。离线客户端重新上线后，会拉取最新变更。

客户端获知文件发生变化后，会先通过 API 服务器请求元数据，再下载数据块并重建文件。

1. <strong>触发：</strong>通知服务告知客户端文件已更新。
2. <strong>获取元数据：</strong>客户端通过 API 获取更新后的元数据。
3. <strong>下载数据块：</strong>客户端从数据块服务器下载更新后的数据块，并重建文件。

<div style="margin-left:3rem">
<img src="../../15.%20Google%20Drive/images/download-flow.png" alt="Download Flow" width="600" />
</div>

---

### 通知服务
1. <strong>用途：</strong>让客户端了解文件变更。
2. <strong>机制：</strong>使用**长轮询**发送异步通知。
3. <strong>示例：</strong>文件被添加、编辑或删除时，向所有相关客户端推送通知。

---

### 存储优化
1. <strong>去重：</strong>通过哈希比较，在账号级别移除重复数据块。
2. **版本策略：**
   - 限制保存的修订版本数量。
   - 对频繁编辑的文件优先保留近期版本。
3. <strong>冷存储：</strong>将很少访问的文件移到较便宜的存储中，例如 Amazon S3 Glacier。

---

### 故障处理
1. <strong>负载均衡器故障：</strong>启用备用负载均衡器。
2. <strong>数据块服务器故障：</strong>将待处理任务重新分配给其他服务器。
3. **元数据数据库故障：**
   - 将从节点提升为主节点。
   - 将流量转发到剩余副本。
4. <strong>云存储故障：</strong>通过跨区域复制获取不可用的文件。
5. <strong>通知服务故障：</strong>客户端重新连接到其他服务器。
