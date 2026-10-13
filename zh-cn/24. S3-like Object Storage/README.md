# 第 24 章：类 S3 对象存储

## 简介

本章将设计一种类似 **Amazon S3** 的**对象存储**服务。

存储系统大致分为三类：

- **块存储**
- **文件存储**
- **对象存储**

**块存储**是一类存储设备，早在 20 世纪 60 年代就已出现，HDD 和 SSD 都是例子。这类设备通常直接连接到服务器，也可以通过高速网络协议连接。服务器可以将原始块格式化后作为文件系统使用，也可以将设备直接交由服务器管理。

**文件存储**构建在块存储之上，提供更高层的抽象，便于管理文件夹和文件。

**对象存储**牺牲一部分性能，以换取高持久性、超大规模和低成本。它面向“冷”数据，主要用于归档和备份。对象存储没有层级目录结构，所有数据都以扁平结构中的对象形式保存。与其他存储类型相比，它相对较慢。大多数云服务商都提供对象存储，例如 Amazon S3 和 Google GCS。

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/storage-comparison.png" alt="存储类型比较" width="500" />
</div>

| 比较项 | 块存储 | 文件存储 | 对象存储 |
|--------|--------|----------|----------|
| 内容可变性 | 是 | 是 | 否（支持对象版本控制） |
| 成本 | 高 | 中到高 | 低 |
| 性能 | 中到高，甚至很高 | 中到高 | 低到中 |
| 一致性 | 强一致性 | 强一致性 | 强一致性 [5] |
| 数据访问方式 | SAS/iSCSI/FC | 标准文件访问、CIFS/SMB 和 NFS | RESTful API |
| 可扩展性 | 中等 | 高 | 极高 |
| 适用场景 | 虚拟机（VM）、数据库 | 通用文件系统访问 | 二进制数据、非结构化数据 |

对象存储相关术语：

- **Bucket（存储桶）**：对象的逻辑容器，名称全局唯一。
- **Object（对象）**：存储在某个存储桶中的独立数据，包含对象数据和元数据。
- **Versioning（版本控制）**：在同一个存储桶中保留某个对象的多个版本。
- **Uniform Resource Identifier（URI，统一资源标识符）**：每个资源都有唯一 URI。
- **Service-level Agreement（SLA，服务级别协议）**：服务提供方与客户之间的协议。

Amazon S3 Standard-Infrequent Access 存储类别的 SLA：

- 跨多个可用区的数据持久性为 99.999999999%。
- 即使整个可用区被摧毁，数据仍能保持韧性。
- 设计可用性为 99.9%。

---

## 第一步：理解问题并明确设计范围

- C：系统需要包含哪些功能？
- I：创建存储桶、上传/下载对象、版本控制、列出存储桶中的对象。
- C：数据的典型大小是多少？
- I：我们需要高效存储超大对象和小对象。
- C：每年存储多少数据？
- I：100 PB。
- C：能否假设数据持久性为 6 个 9（99.9999%），服务可用性为 4 个 9（99.99%）？
- I：可以，听起来合理。

### **非功能性需求**

- **100 PB 数据**
- **6 个 9 的数据持久性**
- **4 个 9 的服务可用性**
- **存储效率**：在维持高可靠性和性能的同时降低存储成本。

### **粗略估算**

对象存储的瓶颈很可能是磁盘容量或每秒 I/O 操作数（IOPS）。

假设：

- 20% 是小对象（小于 1 MB），60% 是中等对象（1–64 MB），20% 是大对象（大于 64 MB）。
- 一块 SATA、7200 转硬盘每秒可进行约 100–150 次随机寻道（100–150 IOPS）。

基于这些假设，可以估算系统能够持久化的对象总数：

- 为简化计算，取每类对象的中位数大小：小对象 0.5 MB、中等对象 32 MB、大对象 200 MB。
- 假设 100 PB 存储空间（10^11 MB）有 40% 被使用，则约有 6.8 亿个对象。
- 如果每个对象的元数据为 1 KB，则存储元数据需要约 0.68 TB 空间。

---

## 第二步：提出高层设计并取得认可

在进入设计前，先了解对象存储的一些特性：

- **对象不可变**：对象存储中的对象不可变（其他存储系统并非如此）。可以删除或替换对象，但不能更新对象内容。
- **键值存储**：对象 URI 就是键，可以通过 HTTP 请求获取对象内容。
- **写一次、读多次**：数据通常只写入一次，但会被读取多次。根据 LinkedIn 的一项研究，95% 的操作是读取。
- 支持小型和大型对象。

对象存储的设计理念与 UNIX 类似：保存文件时，会在称为 inode 的数据结构中创建文件名，而文件数据则存放在磁盘上的不同位置。inode 包含一组文件块指针，指向磁盘上的不同位置。

访问文件时，先从 inode 获取元数据，再读取文件内容。

对象存储也类似：使用元数据存储保存文件信息，实际内容则存放在磁盘上：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/object-store-vs-unix.png" alt="对象存储与 UNIX 的比较" width="500" />
</div>

将元数据与文件内容分离后，可以分别扩展不同的存储：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/bucket-and-object.png" alt="存储桶与对象" width="500" />
</div>

### **高层设计**

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/high-level-design.png" alt="高层设计" width="500" />
</div>

- **负载均衡器**：将 API 请求分配到多个服务副本。
- **API 服务**：无状态服务器，编排对元数据存储、对象存储和 IAM 服务的调用。
- **身份与访问管理（IAM）**：统一处理身份验证、授权和访问控制。
- **数据存储**：存储和读取实际数据，操作基于对象 ID（UUID）。
- **元数据存储**：保存对象元数据。

### **上传对象**

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/uploading-object.png" alt="对象上传" width="500" />
</div>

- 通过 HTTP PUT 请求创建名为“bucket-to-share”的存储桶。
- API 服务调用 IAM，确保用户已获授权并具有写入权限。
- API 服务在元数据存储中创建存储桶记录。创建成功后返回成功响应。
- 存储桶创建后，通过 HTTP PUT 请求创建名为“script.txt”的对象。
- API 服务验证用户身份，并确认用户具有写入权限。
- 校验通过后，通过 HTTP PUT 将对象内容发送至数据存储。数据存储持久化对象并返回 UUID。
- API 服务调用元数据存储，创建新记录，其中包含 object_id、bucket_id、bucket_name 等元数据。

对象上传请求示例：

~~~http
PUT /bucket-to-share/script.txt HTTP/1.1
Host: foo.s3example.org
Date: Sun, 12 Sept 2021 17:51:00 GMT
Authorization: authorization string
Content-Type: text/plain
Content-Length: 4567
x-amz-meta-author: Alex

[4567 bytes of object data]
~~~

### **下载对象**

存储桶没有目录层级，但可以将存储桶名称和对象名称拼接起来，模拟文件夹结构。

获取对象的 GET 请求示例：

~~~http
GET /bucket-to-share/script.txt HTTP/1.1
Host: foo.s3example.org
Date: Sun, 12 Sept 2021 18:30:01 GMT
Authorization: authorization string
~~~

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/download-object.png" alt="对象下载" width="500" />
</div>

- 客户端向负载均衡器发送 HTTP GET 请求，例如 GET /bucket-to-share/script.txt。
- API 服务查询 IAM，验证用户是否具有读取该存储桶的权限。
- 验证通过后，从元数据存储中获取对象的 UUID。
- 根据 UUID 从数据存储中取回对象内容并返回给客户端。

---

<!-- sprint 1 -->

## 第三步：深入设计

### **数据存储**

下面是 API 服务与数据存储的交互方式：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/data-store-interactions.png" alt="数据存储交互" width="500" />
</div>

数据存储的主要组件如下：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/data-store-main-components.png" alt="数据存储的主要组件" width="500" />
</div>

数据路由服务通过 RESTful 或 gRPC API 访问数据节点集群。它是无状态服务，可以通过增加服务器扩展。

它的主要职责包括：

- 查询放置服务，找到最适合存储数据的数据节点。
- 从数据节点读取数据并返回给 API 服务。
- 将数据写入数据节点。

放置服务决定哪些数据节点存储某个对象。它维护一个虚拟集群映射，用来描述集群的物理拓扑：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/virtual-cluster-map.png" alt="虚拟集群映射" width="500" />
</div>

该服务还会向所有数据节点发送心跳，以判断是否需要将某些节点从虚拟集群中移除。

由于这是关键服务，建议维护由 5 个或 7 个副本组成的集群，并使用 Paxos 或 Raft 共识算法同步。例如，7 节点集群可以容忍 3 个节点故障。

数据节点存储实际对象数据。通过将数据复制到多个数据节点，可以保证可靠性和持久性。

每个数据节点都运行一个守护进程，定期向放置服务发送心跳。

心跳信息包括：

- 该数据节点管理多少块 HDD 或 SSD？
- 每块磁盘上存储了多少数据？

#### 数据持久化流程

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/data-persistence-flow.png" alt="数据持久化流程" width="500" />
</div>

- API 服务将对象数据转发给数据存储。
- 数据路由服务将数据发送至主数据节点。
- 主数据节点在本地保存数据，并将其复制到两个次级数据节点。复制成功后再返回响应。
- 对象 UUID 返回给 API 服务。

需要注意：

- 给定对象 UUID 后，使用一致性哈希可以确定性地选择其复制组。
- 在步骤 4 中，主数据节点先复制对象数据，再返回响应。这种做法以更高延迟换取强一致性。

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/consistency-vs-latency.png" alt="一致性与延迟的权衡" width="500" />
</div>

#### 数据如何组织

一种简单的数据管理方式是将每个对象存储为单独的文件。

这种方式可行，但文件系统中小文件很多时性能不佳：

- HDD 上的数据块会被浪费，因为每个文件都会占用整个块。常见块大小为 4 KB。
- 文件很多意味着 inode 很多。操作系统难以处理过多 inode，而且 inode 数量也有上限。

可以使用预写日志（WAL）将许多小文件合并成较大的文件，从而解决这些问题。当文件达到容量上限（通常为几 GB）后，就创建一个新文件：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/wal-optimization.png" alt="WAL 优化" width="500" />
</div>

这种做法的缺点是必须串行化对文件的写入。多个核心访问同一个文件时需要相互等待。可以将文件固定分配给特定核心，以避免锁竞争。

#### 对象查找

为了支持在同一个文件中存储多个对象，需要维护一张表，让数据节点能够查到：

- `object_id`
- 保存对象的 `filename`
- 对象起始位置的 `file_offset`
- `object_size`

可以将这张表部署在 RocksDB 这类基于文件的数据库中，也可以使用传统关系型数据库。由于访问模式是低写入、高读取，关系型数据库更合适。

应该如何部署它？一种做法是在集群中单独部署数据库并独立扩展，让所有数据节点访问该数据库。

缺点：

- 需要积极扩展数据库集群以处理所有请求。
- 数据节点与数据库集群之间会增加网络延迟。

另一种方案是利用数据节点只关心自身相关数据这一点，直接在数据节点内部部署关系型数据库。

SQLite 是不错的选择，因为它是轻量级、基于文件的关系型数据库。

#### 更新后的数据持久化流程

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/updated-data-persistence-flow.png" alt="更新后的数据持久化流程" width="500" />
</div>

- API 服务发送保存新对象的请求。
- 数据节点服务将新对象追加到名为“/data/c”的文件末尾。
- 对象映射表中插入一条新记录。

#### 持久性

数据持久性是本设计的重要要求。为了达到 6 个 9 的持久性，需要仔细分析每一种故障情况。

首先需要解决硬件故障问题。可以复制数据节点以降低故障概率。除此之外，还应跨不同故障域复制数据（跨机架、跨数据中心、使用独立网络等）。重大事件可能导致同一故障域内的多台硬件同时故障：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/failure-domain-isolation.png" alt="故障域隔离" width="500" />
</div>

假设典型 HDD 的年故障率为 0.81%，保存三份副本可以达到 6 个 9 的持久性。

通过复制数据节点可以达到所需的持久性，但也可以使用纠删码降低存储成本。

纠删码使用校验位，在发生故障时可以根据这些校验位重建丢失的数据：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/erasure-coding.png" alt="纠删码" width="500" />
</div>

可以把图中的位块看作数据节点。如果其中两个节点宕机，可以使用其余四个节点恢复数据。

纠删码有多种方案。本系统可以使用 8+4 纠删码，并将数据分布在不同故障域中，以提高可靠性：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/erasure-coding-across-failure-domains.png" alt="跨故障域的纠删码" width="500" />
</div>

采用纠删码可以大幅降低存储成本（改善 50%），代价是数据路由服务需要从多个位置收集数据，因此访问速度会下降：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/erasure-coding-vs-replication.png" alt="纠删码与副本复制" width="500" />
</div>

其他需要注意的事项：

- 三副本复制需要 200% 的额外存储空间，而纠删码只需要 50%。
- [纠删码可达到 11 个 9 的持久性](https://github.com/Backblaze/erasure-coding-durability)，而副本复制为 6 个 9。
- 纠删码需要更多计算来生成和保存校验数据。

总的来说，复制更适用于对延迟敏感的应用；纠删码则有利于降低存储成本并提高持久性。纠删码的实现也困难得多。

#### 正确性校验

如果整块磁盘故障，通常很容易检测。但如果只有部分磁盘内存损坏，就不那么容易发现。

可以使用校验和检测：对文件内容计算哈希值，再用它验证文件完整性。

本系统为每个文件和每个对象保存校验和：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/checksums-for-correctness.png" alt="正确性校验的校验和" width="500" />
</div>

采用 8+4 纠删码时，需要分别读取 8 个数据块，并逐一验证其校验和。

<!-- sprint 2 -->

### **元数据模型**

数据表结构如下：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/metadata-data-model.png" alt="元数据模型" width="500" />
</div>

需要支持的查询：

- 根据名称查找对象 ID。
- 根据名称插入或删除对象。
- 列出存储桶中具有相同前缀的对象。

通常会限制用户可创建的存储桶数量，因此存储桶表较小，可以放在单台数据库服务器上。但仍需要扩展服务器以支持读取吞吐量。

对象表可能无法放入单台数据库服务器，因此需要分片：

- 按 bucket_id 分片会造成热点问题，因为一个存储桶可能有数十亿个对象。
- 按 bucket_id 分片可以让负载分布得更均匀，但查询会很慢。
- 我们选择按 `hash(bucket_name, object_name)` 分片，因为大多数查询都基于对象名或存储桶名。

即使采用这种分片方案，列出存储桶中的对象仍然会比较慢。

### **列出存储桶中的对象**

在单个数据库中，按前缀（类似目录）列出对象的方式如下：

~~~sql
SELECT * FROM object WHERE bucket_id = "123" AND object_name LIKE `abc/%`
~~~

数据库分片后，这类查询很难处理。可以在每个分片上分别执行查询，再在内存中汇总结果。但分页会比较困难，因为不同分片返回的结果数量不同，我们需要分别维护每个分片的 limit/offset。

对象存储通常并不以列出对象的性能为优化重点，因此可以牺牲这部分性能。也可以创建一张用于列出对象的反规范化表，并按 bucket ID 分片。这样列出对象的查询会足够快，因为它只访问一个数据库实例。

### **对象版本控制**

版本控制通过增加一个 `object_version` 列来实现，该列的类型为 TIMEUUID，可以用来对记录排序。

每个新版本都会生成一个新的 `object_id`：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/object-versioning.png" alt="对象版本控制" width="500" />
</div>

删除对象时会创建一个新版本，并使用特殊的 `object_id` 表示对象已删除。查询该对象时返回 404：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/deleting-versioned-object.png" alt="删除有版本记录的对象" width="500" />
</div>

### **优化大文件上传**

可以使用分段上传优化大文件上传：将大文件拆成多个数据块，分别上传：

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/multipart-upload.png" alt="分段上传" width="500" />
</div>

- 客户端调用服务，发起分段上传。
- 数据存储返回一个唯一标识本次上传的 upload ID。
- 客户端将大文件拆成多个数据块，并使用 upload ID 分别上传。
- 每个数据块上传完成后，数据存储返回一个 ETag，即用于标识该分段的 MD5 校验和。
- 所有分段上传完毕后，客户端发送完成分段上传的请求，其中包含 upload_id、分段编号和所有 ETag。
- 数据存储根据这些分段重新组装对象。该过程可能需要几分钟，完成后向客户端返回成功响应。

此时可以删除不再有用的旧分段。可以通过垃圾回收器完成清理。

### **垃圾回收**

垃圾回收是回收不再使用的存储空间的过程。数据变成垃圾的情况包括：

- **延迟删除对象**：对象已标记为删除，但实际数据尚未删除。
- **孤立数据**：例如上传中途失败，需要删除旧分段。
- **损坏数据**：校验和检查未通过的数据。

垃圾回收器也负责回收副本中未使用的空间。使用副本复制时，需要从主副本和其他副本中删除数据；使用 8+4 纠删码时，则需要从全部 12 个节点中删除数据。

为了完成删除，可以使用压缩（compaction）流程：

- 垃圾回收器将未删除的对象从“data/b”复制到“data/d”。
- 复制完成后，通过数据库事务更新 `object_mapping` 表。
- 为避免生成过多小文件，只对超过指定大小阈值的文件执行压缩。

<div style="margin-left:3rem">
    <img src="../../24.%20S3-like%20Object%20Storage/images/compaction.png" alt="存储压缩" width="500" />
</div>

---

## 第四步：总结

本章讨论了：

- 类 S3 对象存储的设计。
- 对象存储、块存储和文件存储之间的区别。
- 存储桶中对象的上传、下载、列出和版本控制。
- 深入设计的数据存储与元数据存储、副本复制与纠删码、分段上传和分片。
