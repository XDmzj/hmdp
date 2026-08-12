## Redisson 的 MultiLock 是什么？

**MultiLock（联锁）** 是 Redisson 提供的一种**将多个独立的锁合并为一个逻辑锁**的机制。它要求**同时成功获取所有子锁**，才算整体加锁成功；任意一个子锁获取失败，整体就失败并释放已获取的锁。
 
---

## 一、为什么需要 MultiLock？

前面你学到了单节点 Redis 分布式锁在主从切换时会丢锁。解决方案之一是**红锁（RedLock）**，即向多个独立的 Redis 节点分别加锁，多数成功才算成功。

**MultiLock 就是红锁的底层基础**，但它更通用——不仅可以用在红锁场景，还可以用在任意"多锁联动"的场景。

**典型场景：**

| 场景 | 说明 |
|:---|:---|
| **红锁** | 多个独立 Redis 主节点，每个节点一把锁，多数成功才算成功 |
| **多资源联动** | 一个业务需要同时锁定"库存锁"和"订单锁"，缺一不可 |
| **多集群同步** | 跨机房、跨集群的锁协调 |

---

## 二、MultiLock 的核心逻辑

### 加锁流程

```
MultiLock 包含 3 个子锁：lock1, lock2, lock3

1. 依次向 lock1、lock2、lock3 发起加锁请求
2. 统计成功数量
3. 如果成功数 >= 要求数（通常是一半以上），整体加锁成功
4. 如果成功数不够，释放所有已成功的子锁，整体加锁失败
```

### 解锁流程

```
对所有子锁逐一解锁（无论之前加锁是否成功）
```

---

## 三、代码怎么写

### 方式一：手动组装 MultiLock

```java
@Autowired
private RedissonClient redisson1;  // 连接 Redis 节点 1
@Autowired
private RedissonClient redisson2;  // 连接 Redis 节点 2
@Autowired
private RedissonClient redisson3;  // 连接 Redis 节点 3

public void doSomething() {
    RLock lock1 = redisson1.getLock("lock:order");
    RLock lock2 = redisson2.getLock("lock:order");
    RLock lock3 = redisson3.getLock("lock:order");
    
    // 创建 MultiLock
    RLock multiLock = redisson1.getMultiLock(lock1, lock2, lock3);
    
    // 用法和普通锁完全一样
    multiLock.lock();
    try {
        // 核心业务
    } finally {
        multiLock.unlock();
    }
}
```

### 方式二：直接用红锁（MultiLock 的特化版）

```java
RLock lock1 = redisson1.getLock("lock:order");
RLock lock2 = redisson2.getLock("lock:order");
RLock lock3 = redisson3.getLock("lock:order");

// RedLock 是 MultiLock 的子类，要求多数成功
RLock redLock = redisson1.getRedLock(lock1, lock2, lock3);

redLock.lock();
try {
    // 业务
} finally {
    redLock.unlock();
}
```

**两者的关系：**

```java
// Redisson 源码
public class RedissonRedLock extends RedissonMultiLock {
    
    // 红锁要求：成功数 > 节点数 / 2
    @Override
    protected int failedLocksLimit() {
        return locks.size() - (locks.size() / 2 + 1);
    }
    
    // 允许失败的时间窗口
    @Override
    protected long calcLockTime(long leaseTime) {
        return leaseTime;
    }
}
```

| 对比 | MultiLock | RedLock |
|:---|:---|:---|
| 成功条件 | 可自定义（默认全部成功） | 过半成功（N/2+1） |
| 用途 | 通用多锁联动 | 高可用分布式锁 |
| 节点要求 | 无 | 多个独立 Redis 主节点 |

---

## 四、和普通锁用法的区别

```java
// 普通单节点锁
RLock lock = redisson.getLock("lock:order");
lock.lock();
try { ... } finally { lock.unlock(); }

// MultiLock（多节点）
RLock lock1 = redisson1.getLock("lock:order");
RLock lock2 = redisson2.getLock("lock:order");
RLock lock3 = redisson3.getLock("lock:order");
RLock multiLock = redisson1.getMultiLock(lock1, lock2, lock3);
multiLock.lock();  // 用法完全一样
try { ... } finally { multiLock.unlock(); }
```

**你不需要改变任何使用习惯**，`RLock` 接口统一了所有锁的调用方式。

---

## 五、完整学习路径总结

你的分布式锁学习路线，现在已经走到了最顶层：

```
第 1 层：Redis 原子命令（SETNX）
         ↓
第 2 层：SET NX EX + Lua 脚本（原子解锁）
         ↓
第 3 层：Redisson RLock（可重入 + Watchdog + 锁重试）
         ↓
第 4 层：MultiLock / RedLock（多节点容错）
```

| 层级                      | 解决的问题             |
| :---------------------- | :---------------- |
| `SETNX`                 | 基本互斥              |
| `SET NX EX` + Lua       | 防死锁、防误删           |
| `RLock`                 | 可重入、自动续期、智能重试     |
| **MultiLock / RedLock** | **主从切换锁丢失、跨节点协调** |

---

## 六、注意事项

1.  **性能开销**：MultiLock 需要向多个节点发请求，网络耗时和故障概率都更大
2.  **节点独立性**：做红锁时，各 Redis 节点必须完全独立，不能有主从关系
3.  **不是银弹**：红锁本身在社区也有争议（如 Martin Kleppmann 的批评），极端情况下仍可能不安全
4.  **够用原则**：大部分场景单节点 Redis + Redisson 锁已经够用，主从切换丢锁的概率极低，不必过度设计