# 第 28 章：证券交易所

## 简介

本章将设计一个**电子证券交易所**。

它的基本功能是高效撮合买家和卖家。

主要证券交易所包括 **NYSE（纽约证券交易所）**、**NASDAQ（纳斯达克）**等。

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/world-stock-exchanges.png" alt="世界各地的证券交易所" width="500" />
</div>

---

## 第一步：理解问题并明确设计范围

- C：我们要交易哪些证券？股票、期权还是期货？
- I：为简化问题，只考虑股票。
- C：支持哪些订单操作？下单、撤单、改单？订单类型支持限价、市价还是条件单？
- I：需要支持下单和撤单。订单类型只考虑限价单。
- C：系统需要支持盘后交易吗？
- I：不需要，只考虑正常交易时间。
- C：能否描述交易所的基本功能？
- I：客户可以提交或撤销限价单，并实时收到撮合成交结果。他们还应能实时查看订单簿。
- C：交易所规模有多大？
- I：同时交易的用户有数万名，约有 100 个交易标的。每天有数十亿笔订单。还需要支持合规风控检查。
- C：需要做哪些风控检查？
- I：先做简单的风控，例如限制用户每天最多交易 100 万股苹果股票。
- C：用户钱包如何处理？
- I：下单前要确保客户资金充足。用于待处理订单的资金需要冻结，直到订单最终完成。

### **非功能性需求**

面试官提供的规模表明，我们要设计的是中小型交易所，同时需要灵活扩展，以便未来支持更多交易标的和用户。

其他非功能性需求：

- **可用性**：至少 99.99%。停机可能损害交易所声誉。
- **容错能力**：需要具备容错能力和快速恢复机制，以限制生产事故的影响。
- **延迟**：往返延迟应达到毫秒级，并关注第 99 百分位。第 99 百分位延迟持续过高，会给少数用户带来糟糕体验。
- **安全性**：需要账户管理系统。为了符合法律要求，需要通过 KYC 验证用户身份。还应保护面向公网的资源免受 DDoS 攻击。

### **粗略估算**

- 100 个交易标的，每天 10 亿笔订单。
- 正常交易时间为 09:30 至 16:00（6.5 小时）。
- QPS = 10 亿 ÷ 6.5 ÷ 3600 ≈ 43,000。
- 峰值 QPS = 5 × QPS ≈ 215,000。
- 市场开盘时交易量会显著增加。

---

## 第二步：提出高层设计并取得认可

### **证券交易基础知识**

先讨论与证券交易所相关的一些基础概念。

经纪商负责协调交易所与终端用户之间的交互，例如 Robinhood、Fidelity 等。

机构客户使用专业交易软件进行大额交易，需要特殊处理。例如，为了避免影响市场，大额交易时会把订单拆分。

订单类型：

- **限价单**：以固定价格买入或卖出。可能无法立即成交，也可能只部分成交。
- **市价单**：不指定价格，立即按当前市场价格成交。

价格术语：

- **Bid（买价）**：买家愿意为股票支付的最高价格。
- **Ask（卖价）**：卖家愿意接受的最低价格。

美国市场有三个层级的报价数据：L1、L2 和 L3。

L1 市场数据包括最佳买入/卖出价格和数量：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/l1-price.png" alt="L1 行情" width="500" />
</div>

L2 包含更多价格档位：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/l2-price.png" alt="L2 行情" width="500" />
</div>

L3 显示各个价格档位以及每个档位上的排队数量：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/l3-price.png" alt="L3 行情" width="500" />
</div>

K 线展示指定时间区间的开盘价、收盘价、最高价和最低价：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/candlestick.png" alt="K 线图" width="500" />
</div>

FIX 是一种用于交换证券交易信息的协议，大多数供应商都会使用。证券交易信息示例：

~~~text
8=FIX.4.2 | 9=176 | 35=8 | 49=PHLX | 56=PERS | 52=20071123-05:30:00.000 | 11=ATOMNOCCC9990900 | 20=3 | 150=E | 39=E | 55=MSFT | 167=CS | 54=1 | 38=15 | 40=2 | 44=15 | 58=PHLX EQUITY TESTING | 59=0 | 47=C | 32=0 | 31=0 | 151=15 | 14=0 | 6=0 | 10=128 |
~~~

### **高层设计**

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/high-level-design.png" alt="高层设计" width="500" />
</div>

交易流程：

- 客户通过交易界面下单。
- 经纪商将订单发送到交易所。
- 订单通过客户端网关进入交易所。网关负责验证、限流、身份认证等，然后将订单转发给订单管理器。
- 订单管理器根据风险管理器设置的规则进行风控检查。
- 风控检查通过后，订单管理器确认用户钱包中有足够资金。
- 订单被发送到撮合引擎。找到匹配项时，撮合引擎会为买卖双方各生成一笔成交（fill）。两笔成交都会排序，以确保结果具有确定性。
- 成交结果返回给客户。

市场数据流程（M1–M3）：

- 撮合引擎生成成交流，并发送给市场数据发布器。
- 市场数据发布器构建 K 线图，并将数据发送到数据服务。
- 市场数据存储在专用存储中，用于实时分析。经纪商连接到数据服务，及时获取市场数据。

报告流程（R1–R2）：

- 报告器从订单和成交中收集所有必要的报告字段，并将它们写入数据库。
- 报告字段包括 client_id、price、quantity、order_type、filled_quantity 和 remaining_quantity。

交易流程处于关键路径上，而其他流程不在关键路径上，因此它们的延迟要求不同。

#### 交易流程

交易流程处于关键路径上，因此应针对低延迟进行高度优化。

核心是撮合引擎，也称为交叉引擎。其主要职责：

- 维护每个交易标的的订单簿，即该标的的买入/卖出订单列表。
- 撮合买入和卖出订单。每次撮合产生两笔成交，分别对应买卖双方。此功能必须快速且准确。
- 将成交流作为市场数据分发。
- 以确定性顺序生成撮合结果，这是实现高可用性的基础。

接下来是排序器。它是使撮合引擎具备确定性的关键组件，会为每个入站订单和出站成交分配序列 ID：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/sequencer.png" alt="排序器" width="500" />
</div>

为入站订单和出站成交分配序号有几个原因：

- 保证及时性和公平性。
- 快速恢复和重放。
- 保证恰好一次处理。

从概念上说，可以使用 Kafka 作为排序器，因为它本质上是入站和出站消息队列。但为了实现更低延迟，我们会自行实现排序器。

订单管理器负责管理订单状态，也与撮合引擎交互，发送订单并接收成交。

订单管理器的职责：

- 将订单发送给风控流程，例如验证用户交易量是否低于 100 万股。
- 根据用户钱包检查订单，确认有足够资金完成交易。
- 将订单发送给排序器，再传给撮合引擎。为减少带宽，只向撮合引擎传递必要的订单信息。
- 从排序器接收成交，再通过客户端网关发送给经纪商。

订单管理器实现中的主要挑战是状态转换管理。一种可行方案是事件溯源（见深入设计一节）。

最后，客户端网关负责接收用户订单并发送给订单管理器。其职责如下：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/client-gateway.png" alt="客户端网关" width="500" />
</div>

客户端网关位于关键路径上，因此应保持轻量。

可以为不同客户设置多个客户端网关。例如，托管（colo）引擎是交易所数据中心中由经纪商租用的交易服务器：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/client-gateways.png" alt="多个客户端网关" width="500" />
</div>

#### 市场数据流程

市场数据发布器从撮合引擎接收成交，并根据成交流构建订单簿和 K 线图。

数据会发送给数据服务，由数据服务向订阅者提供聚合数据：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/market-data.png" alt="市场数据流程" width="500" />
</div>

#### 报告流程

报告器不在关键路径上，但仍然是重要组件。

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/reporting-flow.png" alt="报告流程" width="500" />
</div>

报告器负责交易历史、税务报告、合规报告、结算等。报告流程对延迟要求不高，准确性和合规性更重要。

### **API 设计**

客户通过经纪商与证券交易所交互，以提交订单、查看成交和市场数据、下载历史数据进行分析等。

客户端网关与经纪商之间使用 RESTful API 通信。

对于机构客户，会使用专有协议满足其低延迟需求。

创建订单：

~~~http
POST /v1/order
~~~

参数：

- symbol：股票代码，字符串。
- side：买入或卖出，字符串。
- price：限价单价格，长整型。
- orderType：限价单或市价单（本设计只支持限价单），字符串。
- quantity：订单数量，长整型。

响应：

- id：订单 ID，长整型。
- creationTime：系统创建订单的时间，长整型。
- filledQuantity：已成功成交的数量，长整型。
- remainingQuantity：待成交数量，长整型。
- status：new/canceled/filled，字符串。
- 其余属性与输入参数相同。

获取成交：

~~~http
GET /execution?symbol={:symbol}&orderId={:orderId}&startTime={:startTime}&endTime={:endTime}
~~~

参数：

- symbol：股票代码，字符串。
- orderId：订单 ID，可选，字符串。
- startTime：查询的起始时间，Unix 时间戳 [11]，长整型。
- endTime：查询的结束时间，Unix 时间戳，长整型。

响应：

- executions：范围内的成交数组，包含下列字段。
- id：成交 ID，长整型。
- orderId：订单 ID，长整型。
- symbol：股票代码，字符串。
- side：买入或卖出，字符串。
- price：成交价格，长整型。
- orderType：限价单或市价单，字符串。
- quantity：已成交数量，长整型。

获取订单簿：

~~~http
GET /marketdata/orderBook/L2?symbol={:symbol}&depth={:depth}
~~~

参数：

- symbol：股票代码，字符串。
- depth：订单簿每一侧的深度，整数。

响应：

- bids：包含价格和数量的数组。
- asks：包含价格和数量的数组。

获取 K 线：

~~~http
GET /marketdata/candles?symbol={:symbol}&resolution={:resolution}&startTime={:startTime}&endTime={:endTime}
~~~

参数：

- symbol：股票代码，字符串。
- resolution：K 线图的时间窗口长度，以秒为单位，长整型。
- startTime：窗口开始时间，Unix 时间戳，长整型。
- endTime：窗口结束时间，Unix 时间戳，长整型。

响应：

- candles：K 线数据数组，包含下列字段。
- open：每根 K 线的开盘价，双精度浮点数。
- close：每根 K 线的收盘价，双精度浮点数。
- high：每根 K 线的最高价，双精度浮点数。
- low：每根 K 线的最低价，双精度浮点数。

### **数据模型**

交易所中主要有三类数据：

- 产品、订单和成交。
- 订单簿。
- K 线图。

#### 产品、订单和成交

产品描述交易标的的属性，例如产品类型、交易代码、UI 显示代码等。

这类数据不经常变化，主要用于 UI 展示。

订单代表买入或卖出指令。成交是撮合后生成的结果。

数据模型如下：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/product-order-execution-data-model.png" alt="产品、订单和成交数据模型" width="500" />
</div>

订单和成交会出现在三类流程中：

- 在关键路径上，为保证高性能，它们在内存中处理，并保存在排序器中以便恢复。
- 报告器将订单和成交写入数据库，以支持报告相关场景。
- 成交会转发给市场数据服务，用于重建订单簿和 K 线图。

#### 订单簿

订单簿是按价格档位组织的某个金融工具的买卖订单列表。

高效的数据结构需要满足：

- 常数时间查找：获取某个价格档位或价格区间内的成交量。
- 快速添加、执行和取消订单。
- 查询最佳买价/卖价。
- 遍历价格档位。

订单簿撮合示例：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/order-book-execution.png" alt="订单簿撮合" width="500" />
</div>

完成这笔大额订单后，买卖价差会扩大，价格也会上升。

订单簿实现的伪代码示例：

~~~java
class PriceLevel{
    private Price limitPrice;
    private long totalVolume;
    private List<Order> orders;
}

class Book<Side> {
    private Side side;
    private Map<Price, PriceLevel> limitMap;
}

class OrderBook {
    private Book<Buy> buyBook;
    private Book<Sell> sellBook;
    private PriceLevel bestBid;
    private PriceLevel bestOffer;
    private Map<OrderID, Order> orderMap;
}
~~~

为了提高效率，可以使用双向链表代替普通列表：

- 新订单以 O(1) 时间放到链表尾部。
- 撮合订单以 O(1) 时间从链表头部删除订单。
- 撤单时需要从订单簿中删除订单。通过 `orderMap` 可以用 O(1) 时间查找并删除订单（因为 `Order` 保存了指向链表前一个元素的引用）。

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/order-book-impl.png" alt="订单簿实现" width="500" />
</div>

市场数据服务也会使用这种数据结构来重建订单簿。

#### K 线图

市场数据服务根据一个时间区间内处理的订单计算 K 线数据：

~~~java
class Candlestick {
    private long openPrice;
    private long closePrice;
    private long highPrice;
    private long lowPrice;
    private long volume;
    private long timestamp;
    private int interval;
}

class CandlestickChart {
    private LinkedList<Candlestick> sticks;
}
~~~

为了避免占用过多内存，可以进行以下优化：

- 使用预分配的环形缓冲区存储 K 线，减少内存分配次数。
- 限制内存中的 K 线数量，并将其余数据持久化到磁盘。

我们会使用内存列式数据库（例如 KDB）进行实时分析。收盘后，数据会持久化到历史数据库。

---

## 第三步：深入设计

现代交易所有一个值得注意的特点：与大多数软件系统不同，它们通常会把所有东西都运行在一台大型服务器上。

下面深入探讨设计细节。

### **性能**

对于交易所来说，各个百分位的整体延迟都很重要。

如何降低延迟？

- 减少关键路径上的任务数量。
- 通过减少网络/磁盘使用量和/或缩短任务执行时间，减少每个任务耗时。

为了减少关键路径上的任务，我们移除了所有不必要的职责，甚至移除了日志，以获得最佳延迟。

如果采用原始设计，会遇到几个瓶颈：服务之间的网络延迟，以及排序器的磁盘使用。

这种设计的端到端延迟可以达到几十毫秒，但我们的目标是几十微秒。

因此，我们把所有内容放在一台服务器上，并让进程通过 mmap 作为事件总线进行通信：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/mmap-bus.png" alt="基于 mmap 的事件总线" width="500" />
</div>

另一种优化是使用应用循环（通过 while 循环执行关键任务），并将其固定在同一个 CPU 上，以避免上下文切换：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/application-loop.png" alt="应用循环" width="500" />
</div>

应用循环还可以避免锁竞争，也就是多个线程争抢同一个资源。

下面看看 mmap 的工作方式：它是一个 UNIX 系统调用，用于将磁盘上的文件映射到应用程序内存中。

一种技巧是在 `/dev/shm` 中创建文件。这个目录代表“共享内存”，因此完全不需要访问磁盘。

### **事件溯源**

[数字钱包章节](#/27.%20%20Digital%20Wallet/README ':ignore')对事件溯源进行了详细讨论，可参考其中的内容。

简单来说，我们不保存当前状态，而是保存不可变的状态转换：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/event-sourcing.png" alt="事件溯源" width="500" />
</div>

- 左侧是传统数据模型。
- 右侧是事件溯源数据模型。

目前的设计如下：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/design-so-far.png" alt="当前设计" width="500" />
</div>

- 外部系统通过 FIX 协议与客户端网关交互。
- 订单管理器收到新订单事件，进行验证并更新内部状态，然后将订单发送给撮合核心。
- 如果订单撮合成功，就会生成 `OrderFilledEvent` 并通过 mmap 发送。
- 其他组件订阅事件存储，并各自执行对应的处理。

另一项优化是将订单管理器作为库打包，让所有组件各自保存一份副本，避免为了管理订单而进行额外调用。

在这个设计中，排序器不再是事件存储，而是一个单写入者：它先为事件排序，然后将事件转发到事件存储：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/sequencer-deep-dive.png" alt="排序器深入设计" width="500" />
</div>

### **高可用性**

我们的目标是达到 99.99% 的可用性，也就是每天最多停机 8.64 秒。

为此，需要识别交易所架构中的单点故障：

- 为关键服务（例如撮合引擎）配置备用实例并保持待命。
- 尽可能自动化故障检测和故障转移，切换到备用实例。

客户端网关等无状态服务可以通过增加服务器轻松水平扩展。

对于有状态组件，如果某个实例不是领导者，它可以处理入站事件，但不能发布出站事件：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/leader-election.png" alt="领导者选举" width="500" />
</div>

可以通过发送心跳检测主副本是否宕机。

该机制只适用于单台服务器内部。如果要进一步扩展，可以将整台服务器设置为热备或温备实例，并在发生故障时进行切换。

为了在不同副本之间复制事件存储，可以使用可靠 UDP，以加快通信速度。

### **容错能力**

如果连温备实例也发生故障怎么办？虽然概率很低，但仍需做好准备。

大型科技公司会将核心数据复制到多个城市的数据中心，以应对自然灾害等情况。

需要考虑以下问题：

- 主实例宕机时，如何以及何时切换到备用实例？
- 如何从备用实例中选举出领导者？
- 恢复时间目标（RTO）是多少？
- 需要恢复哪些功能？系统能否在降级状态下运行？

可以这样处理：

- 系统可能因缺陷而宕机（同时影响主实例和副本）。可以通过混沌工程发现边缘情况和此类灾难性结果。
- 初期可以手动进行故障转移，直到充分了解系统的故障模式。
- 主实例宕机时，可以使用领导者选举（例如 Raft）确定哪个副本成为领导者。

不同服务器之间的复制示例如下：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/replication-across-servers.png" alt="跨服务器复制" width="500" />
</div>

领导者选举任期示例：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/leader-election-terms.png" alt="领导者选举任期" width="500" />
</div>

关于 Raft 的工作方式，可以[查看此资料](https://thesecretlivesofdata.com/raft/)。

最后，还需要考虑数据丢失容忍度：在情况变得严重之前，系统最多可以丢失多少数据？这会决定数据备份频率。

对证券交易所而言，数据丢失不可接受，因此必须频繁备份数据，并依赖 Raft 复制来降低数据丢失的概率。

### **撮合算法**

下面通过伪代码简单说明撮合过程：

~~~java
Context handleOrder(OrderBook orderBook, OrderEvent orderEvent) {
    if (orderEvent.getSequenceId() != nextSequence) {
        return Error(OUT_OF_ORDER, nextSequence);
    }

    if (!validateOrder(symbol, price, quantity)) {
        return ERROR(INVALID_ORDER, orderEvent);
    }

    Order order = createOrderFromEvent(orderEvent);
    switch (msgType):
        case NEW:
            return handleNew(orderBook, order);
        case CANCEL:
            return handleCancel(orderBook, order);
        default:
            return ERROR(INVALID_MSG_TYPE, msgType);

}

Context handleNew(OrderBook orderBook, Order order) {
    if (BUY.equals(order.side)) {
        return match(orderBook.sellBook, order);
    } else {
        return match(orderBook.buyBook, order);
    }
}

Context handleCancel(OrderBook orderBook, Order order) {
    if (!orderBook.orderMap.contains(order.orderId)) {
        return ERROR(CANNOT_CANCEL_ALREADY_MATCHED, order);
    }

    removeOrder(order);
    setOrderStatus(order, CANCELED);
    return SUCCESS(CANCEL_SUCCESS, order);
}

Context match(OrderBook book, Order order) {
    Quantity leavesQuantity = order.quantity - order.matchedQuantity;
    Iterator<Order> limitIter = book.limitMap.get(order.price).orders;
    while (limitIter.hasNext() && leavesQuantity > 0) {
        Quantity matched = min(limitIter.next.quantity, order.quantity);
        order.matchedQuantity += matched;
        leavesQuantity = order.quantity - order.matchedQuantity;
        remove(limitIter.next);
        generateMatchedFill();
    }
    return SUCCESS(MATCH_SUCCESS, order);
}
~~~

该撮合算法使用 FIFO（先进先出）算法，决定同一价格档位中先撮合哪笔订单。

### **确定性**

通过排序器机制可以保证功能上的确定性。

事件实际发生的时间并不重要：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/determinism.png" alt="确定性" width="500" />
</div>

还需要跟踪延迟的确定性，可以通过监控第 99 或第 99.99 百分位延迟来计算。

例如 Java 垃圾回收事件可能造成延迟峰值。

### **市场数据发布器优化**

市场数据发布器从撮合引擎接收撮合结果，并根据这些结果重建订单簿和 K 线图。

由于内存有限，我们只保留部分 K 线。客户可以选择所需的信息粒度；越细粒度的数据可能收费越高：

<div style="margin-left:3rem">
    <img src="../../28.%20Stock%20Exchange/images/market-data-publisher.png" alt="市场数据发布器" width="500" />
</div>

环形缓冲区（也称循环缓冲区）是一种固定大小的队列，首尾相连。其存储空间预先分配，因此可以避免动态内存分配。这种数据结构也是无锁的。

另一种优化环形缓冲区的技术是填充（padding），以确保序列号不会与其他数据位于同一个缓存行中。

### **市场数据分发公平性与组播**

必须确保所有订阅者同时收到数据。如果一方先收到数据，就会获得重要市场信息，并可能利用它操纵市场。

可以在向订阅者发布数据时使用可靠 UDP 组播，以满足这一要求。

数据可以通过互联网以三种方式传输：

- **单播（Unicast）**：一个源发送到一个目标。
- **广播（Broadcast）**：一个源发送到整个子网。
- **组播（Multicast）**：一个源发送到不同子网中的一组主机。

理论上，使用组播可以让所有订阅者同时收到数据。

但 UDP 不可靠，数据可能无法送达所有接收方。不过可以通过重传来增强可靠性。

### **同地托管**

交易所允许经纪商将服务器与交易所部署在同一个数据中心内，这称为同地托管（colocation）。

这样可以大幅降低延迟，可视为一种 VIP 服务。

### **网络安全**

交易所的一些服务面向互联网，因此 DDoS 是一项挑战。可以采取以下措施：

- 隔离公网服务和数据与私有服务，避免 DDoS 攻击影响最重要的客户。
- 使用缓存层保存更新不频繁的数据。
- 加固 URL，降低 DDoS 风险。例如，优先使用 `https://my.website.com/data/recent`，而不是 `https://my.website.com/data?from=123&to=456`，因为前者更容易缓存。
- 建立有效的允许列表和阻止列表机制。
- 使用限流缓解 DDoS 攻击。

---

## 第四步：总结

其他值得关注的内容：

- 并非所有交易所都会把所有东西放在一台大型服务器上，但仍有一些交易所采用这种方式。
- 现代交易所越来越多地依赖云基础设施和自动做市商（AMM），以避免维护订单簿。
