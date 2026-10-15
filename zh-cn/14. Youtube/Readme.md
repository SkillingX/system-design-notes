# 第 14 章：设计 YouTube

## 引言
YouTube 是一个大规模视频流媒体平台，支持上传、播放和多种互动。本章重点讨论如何设计一个可扩展的视频流系统，并满足以下核心功能：
- **快速上传视频**
- **流畅播放视频**
- **支持调整视频画质**
- **降低基础设施成本**
- **高可用性和可靠性**

### 关键数据（2020 年）
- **月活跃用户 20 亿**
- **每天播放 50 亿个视频**
- **37% 的移动互联网流量来自 YouTube**
- 支持 **80 种语言**
- **2019 年广告收入为 151 亿美元**

---

## 第 1 步：理解问题并确定范围

### 核心功能
1. 上传视频。
2. 观看视频。

### 支持平台
- 移动应用、Web 浏览器和智能电视。

### 假设
- <strong>日活跃用户（DAU）：</strong>500 万。
- <strong>平均视频大小：</strong>300 MB。
- <strong>上传上限：</strong>每个视频最大 1 GB。
- <strong>每日存储需求：</strong>150 TB。
- <strong>CDN 成本：</strong>500 万 × 5 个视频 × 0.3 GB × 0.02 美元 = 每天 150,000 美元（使用 Amazon CloudFront）。

---

## 第 2 步：高层设计

### 组件

<div style="margin-left:3rem">
    <img src="../../14.%20Youtube/images/high-level-design.png" alt="High Level Design" width="400">
</div>

1. <strong>客户端：</strong>智能手机、电脑和电视等设备。
2. <strong>CDN（内容分发网络）：</strong>存储并流式传输视频。
3. <strong>API 服务器：</strong>处理视频流传输以外的所有用户交互，例如上传和更新元数据。
4. <strong>元数据数据库：</strong>保存视频元数据，例如标题、说明和大小。
5. <strong>原始存储：</strong>保存上传视频的 Blob 存储。
6. <strong>转码服务器：</strong>将视频转换为多种分辨率和格式。
7. <strong>转码后存储：</strong>保存转码视频的 Blob 存储。

---

### 核心工作流程
#### 1. 视频上传流程
- **并行处理：**
  1. 将视频上传到原始存储。
  2. 更新数据库中的视频元数据。

- **视频上传步骤：**

    <div style="margin-left:3rem">
        <img src="../../14.%20Youtube/images/video-uploading-flow.png" alt="Video Upload Flow" width="500">
    </div>

    - [1] 将视频上传到 Blob 存储。
    - [2] 转码服务器将视频转换为多种格式。
    - [3] 转码完成后，并行执行以下两个步骤：
        - [3a] 将转码后的视频发送到转码后存储。
        - [3b] 将转码完成事件放入完成队列。
    - [3a.1] 将视频分发到 CDN。
    - [3b.1] 完成处理器更新元数据并通知用户。

- **元数据上传步骤：**

    <div style="margin-left:3rem">
        <img src="../../14.%20Youtube/images/metadata-upload.png" alt="Metadata Upload" height="500">
    </div>

    - 客户端并行发送请求以更新视频元数据。
    - 请求包含文件名、大小、格式等视频元数据。

#### 2. 视频流播放流程

<div style="margin-left: 3em;">
  <img src="../../14.%20Youtube/images/video-streaming-flow.png" alt="Video Streaming Flow" height="400">
</div>

- 视频通过 CDN 边缘服务器直接流式传输，以尽量降低延迟。
- 常见的视频流协议包括 MPEG_DASH、Apple HLS 和 Adobe HDS。
- *不同的流协议支持不同的视频编码格式和播放器。*

---

## 第 3 步：深入设计

### 视频转码
#### 重要性
1. 原始视频会占用大量存储空间；转码可以减少存储占用。
2. 确保视频能在不同设备和浏览器上播放。
3. 根据网络状况调整视频质量。

#### 组件
- <strong>容器：</strong>封装视频、音频和元数据，例如 MP4、AVI。
- <strong>编解码器：</strong>用于压缩和解压缩的算法，例如 H.264、VP9。

#### 有向无环图（DAG）模型
<div style="margin-left: 3em;">
    <img src="../../14.%20Youtube/images/dag-video-transcoding.png" alt="DAG Video Transcoding" width="600">
</div>

- 视频转码计算成本高、耗时长。
- DAG 模型定义编码、缩略图生成和添加水印等任务。
- 支持并行处理视频。

- 原始视频会拆分为视频、音频和元数据。
    - 视频编码：转换视频以支持不同分辨率、编解码器和比特率。
    - 缩略图：可以由用户上传，也可以由系统自动生成。
    - 水印：叠加在视频上的图片，其中包含可用于识别视频的信息。

### 视频转码架构

<div style="margin-left: 3em;">
<img src="../../14.%20Youtube/images/video-transcoding-architecture.png" alt="Video Transcoding" width="600">
</div>

1. <strong>预处理器：</strong>按照 GOP 对齐方式将视频拆成较小的片段，负责以下 4 项工作。

    <div style="margin-left: 3em;">
        <img src="../../14.%20Youtube/images/dag-config.png" alt="DAG Config" width="500">
    </div>

    - 视频切分：按照 GOP 对齐方式将视频流切分，必要时进一步拆成较小片段。
    - 为旧客户端按照 GOP 对齐方式切分视频。
    - 根据客户端程序员编写的配置文件生成 DAG。
    - 将 GOP 和元数据保存在临时存储中。编码失败时，系统可以使用已保存的数据重试。

2. <strong>DAG 调度器：</strong>将任务组织为串行或并行阶段。
    <div style="margin-left: 3em;">
        <img src="../../14.%20Youtube/images/dag-scheduler.png" alt="DAG Scheduler" width="500">
    </div>

    - 将 DAG 拆分为多个任务阶段，并将任务放入资源管理器的任务队列。
    - 阶段 1：视频、音频和元数据。
    - 阶段 2：视频文件进一步拆分为视频编码和缩略图两个任务。

3. <strong>资源管理器：</strong>负责高效管理资源分配，包含 3 个队列和一个任务调度器。
    <div style="margin-left: 3em;">
        <img src="../../14.%20Youtube/images/resource-manager.png" alt="Resource Manager" width="700">
    </div>

    - 任务队列：包含待执行任务的优先级队列。
    - Worker 队列：包含 worker 利用率信息的优先级队列。
    - 运行队列：包含当前运行的任务以及执行这些任务的 worker。
    - 任务调度器：选择最合适的任务和 worker，并指示选定的任务 worker 执行任务。

4. <strong>任务 Worker：</strong>执行转码和其他操作。
    <div style="margin-left: 3em;">
        <img src="../../14.%20Youtube/images/task-worker.png" alt="Task Worker" width="250">
   </div>

    - 不同的任务 worker 可以执行不同类型的任务。

5. <strong>临时存储：</strong>保存中间数据，以供重试。
    - 存储系统的选择取决于数据类型、大小、访问频率和数据生命周期等因素。
6. <strong>输出：</strong>准备好分发的转码视频。

---

## 系统优化

### 速度优化
1. <strong>并行上传视频：</strong>将视频拆分为较小片段，以便更快地上传，并支持断点续传。

    <img src="../../14.%20Youtube/images/video-split.png" alt="Video Split" width="600">

2. <strong>分布式上传中心：</strong>使用靠近用户的 CDN 作为上传入口。
3. <strong>并行处理：</strong>使用消息队列解耦模块，以提高并行度。

    <img src="../../14.%20Youtube/images/message-queue1.png" alt="Message Queue" width="600">
    <img src="../../14.%20Youtube/images/message-queue2.png" alt="Message Queue" height="170" width="500">

### 安全优化
1. <strong>预签名 URL：</strong>限制只有授权用户才能上传视频。

    <img src="../../14.%20Youtube/images/pres-signed-urls.png" alt="Pre Signed" width="500">

2. **保护视频：**
   - 使用 **DRM 系统**，例如 Apple FairPlay、Google Widevine。
   - 使用 **AES 加密**。
   - 添加**水印**。

### 降低成本的优化
1. 只通过 CDN 服务热门视频；不太热门的视频从高容量服务器提供。
2. 对很少访问的视频按需编码。
3. 根据视频热度进行区域化分发。
4. 构建自有 CDN 并与 ISP 合作，以降低带宽成本。

---

## 错误处理
### 可恢复错误
对失败的上传、转码或资源分配任务进行重试。

### 不可恢复错误
停止处理格式错误的视频并返回错误码。
