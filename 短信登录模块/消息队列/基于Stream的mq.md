
## Redis Stream 实现消息队列

---

### 一、Stream 是什么？

**Stream 是 Redis 5.0 引入的专门为消息队列设计的数据结构。** 它解决了 List 和 Pub/Sub 的致命缺点，是 Redis 家族里**最接近专业 MQ** 的实现。

一句话定位：**带 ACK（确认机制）、支持消费组、消息可持久化的 Redis 消息队列。**

---

### 二、核心概念

| 概念 | 说明 | 类比 |
|:---|:---|:---|
| **Stream** | 消息队列本体，存储消息 | 一个 Topic |
| **消息** | 一条记录，包含多个 field-value | 一条数据 |
| **消息 ID** | 每条消息的唯一标识，自动生成 | 主键 |
| **Consumer Group** | 消费组，组内消费者**分摊**消息 | 一组工人流水线 |
| **Consumer** | 组内的一个消费者 | 一个工人 |
| **Pending List** | 已投递但未 ACK 的消息列表 | 工单待确认 |

---

### 三、消息 ID 结构

```
1692000000000-0
   │           │
   │           └── 同一毫秒内的序号
   └────────────── 毫秒时间戳
```

-   自动生成：`XADD` 时不指定，Redis 自动生成
-   保证**严格递增**：越新消息 ID 越大
-   也可以手动指定，如 `1680000000000-1`

---

### 四、核心命令

#### 1. `XADD` — 生产者发消息

![[Pasted image 20260814152728.png]]

```bash
XADD mystream * field1 value1 field2 value2
```

-   `mystream`：队列名
-   `*`：让 Redis 自动生成消息 ID
-   后面是 field-value 对（和 Hash 一样）
-   返回生成的消息 ID

**Java 代码：**

```java
// 生产者
Map<String, String> message = new HashMap<>();
message.put("userId", "1001");
message.put("voucherId", "5001");
String messageId = stringRedisTemplate.opsForStream()
        .add("order_stream", message);
```

---

#### 2. `XREAD` — 简单消费（无消费组）

![[Pasted image 20260814152757.png]]

```bash
XREAD COUNT 1 BLOCK 2000 STREAMS mystream 0
```

-   `COUNT 1`：一次最多取 1 条
-   `BLOCK 2000`：阻塞 2 秒，没有消息就等
-   `0`：从最早的消息开始读
-   `$`：只读新来的消息（不读历史）

**缺点**：`XREAD` 没有 ACK，消息读了但不确认，无法处理"读了没处理完就崩溃"的情况。

---

#### 3. 消费组命令

**创建消费组：**
![[Pasted image 20260814160343.png|552]]
```bash
XGROUP CREATE mystream group1 0
```

-   `group1`：消费组名
-   `0`：从最早的消息开始消费
-   `$`：只消费新消息

**组内消费：**
![[Pasted image 20260814160324.png]]
```bash
XREADGROUP GROUP group1 consumer1 COUNT 1 BLOCK 2000 STREAMS mystream >
```

-   `>`：只读取**从未投递过**的新消息
-   组内的多个消费者**分摊**消息，一条消息只给一个消费者

**确认消息（ACK）：**

```bash
XACK mystream group1 1692000000000-0
```

**查看待确认消息（Pending）：**

```bash
XPENDING mystream group1
```

---

### 五、Java 完整代码

```java
// ============ 生产者 ============
public void produce(Long userId, Long voucherId) {
    Map<String, String> message = new HashMap<>();
    message.put("userId", userId.toString());
    message.put("voucherId", voucherId.toString());
    stringRedisTemplate.opsForStream().add("order_stream", message);
}

// ============ 消费者组初始化 ============
@PostConstruct
public void initGroup() {
    try {
        stringRedisTemplate.opsForStream()
            .createGroup("order_stream", "order_group");
    } catch (Exception e) {
        // 消费组已存在，忽略
    }
}

// ============ 消费者 ============
public void consume() {
    while (true) {
        List<MapRecord<String, Object, Object>> records =
            stringRedisTemplate.opsForStream().read(
                Consumer.from("order_group", "consumer_1"),
                StreamReadOptions.empty().count(1).block(Duration.ofSeconds(2)),
                StreamOffset.create("order_stream", ReadOffset.lastConsumed())
            );
        
        for (MapRecord<String, Object, Object> record : records) {
            String messageId = record.getId();
            Map<Object, Object> body = record.getValue();
            
            // 处理消息...
            handleMessage(body);
            
            // 确认消息
            stringRedisTemplate.opsForStream()
                .acknowledge("order_stream", "order_group", messageId);
        }
    }
}
```

---

### 六、消息流转完整流程

```
生产者                          Redis Stream                    消费者组
   │                                │                               │
   │ XADD mystream * ... ──────────→│                               │
   │                                │  消息存入 Stream              │
   │                                │  [{id1:...}, {id2:...}]      │
   │                                │                               │
   │                                │←── XREADGROUP 读消息 ──────────│
   │                                │                               │
   │                                │── 返回 id1 给消费者A ──────────→│
   │                                │                               │
   │                                │   id1 进入 Pending List        │
   │                                │                               │
   │                                │←── XACK id1 ───────────────────│
   │                                │   id1 从 Pending List 移除     │
   │                                │   消息被确认，不再重复投递       │
```

---

### 七、Panding List（待确认列表）机制

**这是 Stream 和 List 最本质的区别：**

```
消费者通过 XREADGROUP 取走消息后：

1. 消息进入该消费者的 Pending List
2. 消费者处理后调用 XACK
3. XACK 后消息才从 Pending List 移除

如果消费者崩溃，没来得及 XACK：

  消息永远留在 Pending List
  → 其他消费者可以用 XAUTOCLAIM 接管
  → 或者原消费者重启后从 Pending List 重新处理
```

**解决了 List 的最大问题：消息取走后崩溃就丢了。**

---

### 八、和 List / Pub/Sub 对比

| 维度 | List | Pub/Sub | **Stream** |
|:---|:---|:---|:---|
| 消息持久化 | ✅ 存内存 | ❌ 不存 | ✅ 可持久化 |
| ACK 确认 | ❌ 取出就删 | ❌ 无 | ✅ `XACK` |
| 消费组 | ❌ 无 | ❌ 无 | ✅ 支持 |
| 消息重试 | ❌ 无 | ❌ 无 | ✅ Pending + `XAUTOCLAIM` |
| 一对多 | ❌ 一条一人取 | ✅ 广播 | ✅ 不同消费组各读各的 |
| 阻塞消费 | ✅ `BRPOP` | ❌ 订阅模式 | ✅ `BLOCK` 参数 |
| 历史消息 | ✅ 可取 | ❌ 无历史 | ✅ 按 ID 读取 |
| 生产级可靠性 | ❌ | ❌ | ✅ 接近专业 MQ |

---

### 九、适用场景

| 场景 | 是否适合 |
|:---|:---|
| 订单、支付等**不能丢消息**的 | ✅ 适合 |
| 需要多个消费者组**各自独立消费** | ✅ 适合 |
| 需要消息重试 / 死信处理 | ✅ 适合（需自己实现死信逻辑） |
| 简单通知广播 | 不如 Pub/Sub 简单 |
| 海量日志采集 | 不如 Kafka 吞吐高 |

---

### 十、总结笔记

-   **Stream**：Redis 5.0 的正式 MQ 方案，带 ACK + 消费组
-   **核心概念**：消息 ID、Consumer Group、Pending List、XACK
-   **解决 List 的痛点**：取走即丢 → 有 Pending 确认机制
-   **解决 Pub/Sub 的痛点**：不持久 → 消息落盘不丢
-   **优势**：复用 Redis，不用额外部署 MQ，可靠性接近专业 MQ
-   **局限**：吞吐量不如 Kafka，生态不如 RabbitMQ 丰富