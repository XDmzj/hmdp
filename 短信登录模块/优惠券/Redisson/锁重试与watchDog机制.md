
Redisson分布式锁原理：

如何解决可重入问题：利用hash结构记录线程id和重入次数。

如何解决可重试问题：利用信号量和Pub/Sub功能实现等待、唤醒，获取锁失败的重试机制。

如何解决超时续约问题：利用watchDog，每隔一段时间（releaseTime / 3），重置超时时间。

如何解决主从一致性问题：利用Redisson的multiLock，多个独立的Redis节点，必须在所有节点都获取重入锁，才算获取锁成功

缺陷：运维成本高、实现复杂

# 锁重试

**锁重试机制（源码要点）：**

- 入口：`tryLock(waitTime, leaseTime, unit)`
    
- 先抢一次，失败后订阅 Redis Pub/Sub 频道
    
- 进入 `while` 循环，用 **Semaphore（初始 0）** 阻塞当前线程
    
- Semaphore 等待时间 = `min(剩余等待时间, 锁的 TTL)`
    
- **唤醒方式**：解锁 Lua 脚本 `publish` 消息 → `Semaphore.release()` → 线程苏醒重新抢锁
    
- 不空转 CPU，不无脑网络请求



### 1. 入口：`lock()` 和 `tryLock()`

当你调用 `lock.lock()` 时，实际调用链为：
```text

RLock.lock()
  └─ RedissonLock.lock()
       └─ lock(-1, null, false)   // leaseTime=-1 表示不手动设过期时间，启用Watchdog
            └─ tryAcquire(-1, leaseTime, unit, threadId)
                 └─ tryLock() → tryAcquireAsync()
```


核心入口是 `RedissonLock.tryLock(long waitTime, long leaseTime, TimeUnit unit)`：
~~~java


// RedissonLock 源码（简化版）
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
    long time = unit.toMillis(waitTime);      // 最大等待时间
    long current = System.currentTimeMillis();
    long threadId = Thread.currentThread().getId();
    
    // 第1步：先尝试获取一次锁
    Long ttl = tryAcquire(waitTime, leaseTime, unit, threadId);
    
    // 成功（ttl为null），直接返回true
    if (ttl == null) {
        return true;
    }
    
    // 第2步：检查是否超过等待时间
    time -= System.currentTimeMillis() - current;
    if (time <= 0) {
        acquireFailed(waitTime, unit, threadId);
        return false;
    }
    
    // 第3步：订阅释放频道，进入循环等待
    current = System.currentTimeMillis();
    CompletableFuture<RedissonLockEntry> subscribeFuture = subscribe(threadId);
    // 阻塞等待订阅完成（这是一个异步过程，用CountDownLatch同步等待）
    subscribeFuture.get(time, TimeUnit.MILLISECONDS);
    
    try {
        // 再次检查是否超时
        time -= System.currentTimeMillis() - current;
        if (time <= 0) {
            acquireFailed(waitTime, unit, threadId);
            return false;
        }
        
        // 第4步：进入核心循环
        while (true) {
            long currentTime = System.currentTimeMillis();
            
            // 再次尝试获取锁
            ttl = tryAcquire(waitTime, leaseTime, unit, threadId);
            
            if (ttl == null) {
                return true;   // 成功！
            }
            
            time -= System.currentTimeMillis() - currentTime;
            if (time <= 0) {
                acquireFailed(waitTime, unit, threadId);
                return false;  // 超时
            }
            
            currentTime = System.currentTimeMillis();
            
            // 关键：用信号量阻塞等待，等待时间 = min(剩余等待时间, 锁的TTL)
            if (ttl >= 0 && ttl < time) {
                // 锁会在ttl毫秒后释放，等待ttl时间再试
                getEntry(threadId).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
            } else if (ttl >= 0) {
                // 锁的TTL大于剩余等待时间，等到超时
                getEntry(threadId).getLatch().tryAcquire(time, TimeUnit.MILLISECONDS);
            }
            
            time -= System.currentTimeMillis() - currentTime;
            if (time <= 0) {
                acquireFailed(waitTime, unit, threadId);
                return false;
            }
        }
    } finally {
        unsubscribe(subscribeFuture, threadId);
    }
}
~~~

---

### 2. 核心机制一：信号量阻塞（Semaphore）

信号量，就是 Redisson 锁重试机制里负责**把线程挂起**的那个东西。

>信号量是什么？

**信号量（Semaphore）** 是 JDK 提供的一个并发工具，可以把它理解成一个**管理"许可证"的计数器**

- `Semaphore(0)`：初始有 **0 个许可证**
    
- `tryAcquire()`：尝试获取 1 个许可证
    
    - 如果有许可证，拿到，返回 `true`
        
    - 如果 **没有许可证，线程阻塞**，直到有人释放一个许可证
        
- `release()`：释放 1 个许可证，随机唤醒一个正在等待的线程

**一句话：信号量 = 一把能控制"等待/唤醒"的开关。**

>Redisson 为什么用信号量？

**用来替代你之前 SimpleRedisLock 里的 `Thread.sleep()` 空转。**

Redisson 的做法（信号量）

~~~java
// 初始 0 个许可证
Semaphore latch = new Semaphore(0);
// 尝试获取 1 个许可证 —— 但没有，线程挂起！
latch.tryAcquire(5000, TimeUnit.MILLISECONDS);
~~~
- 线程**真正被操作系统挂起**，不消耗 CPU
    
- 等待期间不发送任何 Redis 请求

**什么时候被唤醒？**
持锁者释放时，Redis Pub/Sub 通知 → `latch.release()` → 线程苏醒抢锁：
~~~java
latch.release();  // 释放一个许可证
~~~
信号量许可证 +1，之前 `tryAcquire` 的线程立刻被唤醒，重新去抢锁。



>为什么信号量初始值是 0？

**因为初始不能有许可证。**

如果有许可证，`tryAcquire()` 一上来就直接成功返回，线程压根不会等待，也就无法被"释放通知"唤醒了。

**`Semaphore(0)` 的语义就是："先等着，我叫你你再动。"**

>信号量在锁重试中的位置

回到之前讲过的代码：

~~~java

// 1. 抢锁失败，拿到锁的 TTL（剩余存活时间）
Long ttl = tryAcquire(...);
// 2. 没抢到，用信号量阻塞
//    等待时间 = min(锁TTL, 剩余等待预算)
getEntry(threadId).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
//        ↑
//   getLatch() 返回的就是 Semaphore(0)
~~~

**`getLatch()` 就是 `Semaphore(0)`：**

~~~java

public class RedissonLockEntry {
    private final Semaphore latch = new Semaphore(0);  // ← 就在这里
}
~~~



---

### 3. 核心机制二：Redis Pub/Sub 通知唤醒

线程阻塞后，什么时候被唤醒？
**一句话：订阅是用来接收"锁被释放了"的通知的，让等待的线程不用瞎猜，而是被精确唤醒。**

#### 没有订阅时的做法（你之前的 SimpleRedisLock）

~~~java

// 拿不到锁，自己 sleep 一会再试
while (!tryLock()) {
    Thread.sleep(100);  // 瞎猜一个时间
}
~~~
**问题：**

- 锁可能 10 毫秒后就释放了，但你 sleep 了 100ms，浪费 90ms
    
- 锁可能 5 秒后才释放，你每隔 100ms 就发起一次无效的 Redis 请求
    
- 要么等太久，要么请求太多
    

#### 有订阅时的做法

~~~text

线程A：拿不到锁 → 订阅"锁释放频道" → 信号量阻塞
线程B：释放锁 → publish 消息到频道
线程A：收到消息 → 被唤醒 → 立刻去抢锁
~~~
**好处：**

- 锁一释放，立刻收到通知，几乎零延迟
    
- 等待期间不发任何 Redis 请求，零网络开销
    
- 线程被 JDK 信号量挂起，不消耗 CPU
    

**类比：**

- 没订阅 = 你每隔 5 分钟跑去柜台问"到我了没"
    
- 有订阅 = 你取个号，等着广播叫你就行
---

### 4. 完整重试流程图

~~~ text

线程调用 lock.lock()
    │
    ▼
第1次尝试加锁（Lua脚本）
    │
    ├─ 成功 → 返回，Watchdog 启动
    │
    └─ 失败 → 收到 TTL（锁剩余存活时间）
         │
         ├─ 检查超时：剩余等待时间 < 0 → 返回 false
         │
         ├─ 订阅 Redis 频道：redisson_lock__channel:{lock_key}
         │
         ▼
    ┌────── 进入 while 循环 ──────┐
    │                              │
    │  第 N 次尝试加锁（Lua脚本）    │
    │      │                       │
    │      ├─ 成功 → 返回 true     │
    │      └─ 失败 → 收到新 TTL   │
    │           │                  │
    │           ▼                  │
    │   Semaphore.tryAcquire(TTL) │
    │      │                       │
    │      ├─ 收到 Pub/Sub 通知   │
    │      │  Semaphore.release() │
    │      │  被唤醒               │
    │      │                       │
    │      └─ 等待超时            │
    │           自动醒来           │
    │           │                  │
    │           └── 回循环开头 ────┘
    │
    └── 超过最大等待时间 → 返回 false
~~~

---
# watchDog机制
Watchdog（看门狗）是 Redisson 分布式锁中**自动续期**的机制。它解决的核心问题是：**“业务还没执行完，锁过期了怎么办？”**


## 一、为什么要 Watchdog？

### 之前 SimpleRedisLock 的问题

```java
// 加锁时设了 10 秒过期
SET lock:order thread1 NX EX 10
```

如果业务执行了 15 秒：

```
第 0 秒：线程1 获取锁，过期时间 10 秒
第 10 秒：业务还没跑完，锁自动过期，被 Redis 删除
第 11 秒：线程2 趁虚而入，获取锁成功
第 13 秒：线程1 业务完成，执行解锁 → 误删线程2 的锁
```

**要么设太长（业务异常挂了，锁很久才释放），要么设太短（业务正常还没跑完锁就过期了）。**

### Watchdog 的思路

**不设固定过期时间，而是“只要你还在干活，我就一直给锁续命”。**

---

## 二、触发条件

**只有不手动指定锁持有时间时，Watchdog 才启动。**

```java
// ✅ Watchdog 启动
lock.lock();
lock.tryLock(10, TimeUnit.SECONDS);  // 只传等待时间，不传持有时间

// ❌ Watchdog 不启动
lock.tryLock(10, 30, TimeUnit.SECONDS);  // 传了持有时间 30 秒，你自己管过期
```

源码判断逻辑：

```java
if (leaseTime > 0) {
    // 手动设了过期时间 → 不启动 Watchdog
    future = tryLockInnerAsync(waitTime, leaseTime, unit, threadId);
} else {
    // leaseTime = -1 → 用默认 30 秒加锁，然后启动 Watchdog
    future = tryLockInnerAsync(waitTime, 
               getService().getLockWatchdogTimeout(),  // 默认 30000ms
               TimeUnit.MILLISECONDS, threadId);
    future.thenApply(t -> {
        if (t == null) {
            scheduleExpirationRenewal(threadId);  // 启动 Watchdog
        }
        return t;
    });
}
```

---

## 三、核心流程

```
1. 线程加锁成功
2. Watchdog 启动：创建一个定时任务，延迟 10 秒执行
3. 10 秒后，执行续期 Lua 脚本
4. 续期成功 → 锁过期时间重新设为 30 秒
5. 再安排下一次续期（再过 10 秒执行）
6. 业务完成，调用 unlock() → 取消定时任务 + 删除锁
```

**关键时间参数：**

| 参数 | 默认值 | 说明 |
|:---|:---|:---|
| `lockWatchdogTimeout` | 30000 ms | 锁的默认过期时间 |
| 续期间隔 | `lockWatchdogTimeout / 3` = 10000 ms | 每 10 秒续一次 |

**为什么要 10 秒续一次？** 留足容错空间。即使一次续期失败或网络抖动，还有 20 秒缓冲，不会立刻死锁。

---

## 四、续期 Lua 脚本

```lua
-- KEYS[1]：锁的 Key
-- ARGV[1]：要续期的时长（30000ms）
-- ARGV[2]：线程标识（field）

if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then
    -- 锁还是当前线程的，续期
    redis.call('pexpire', KEYS[1], ARGV[1]);
    return 1;
end;
return 0;  -- 锁已经不在了，不续
```

**原子性地完成"检查锁归属"和"延长过期时间"两个操作。**

---

## 五、Watchdog 停止时机

| 情况 | 行为 |
|:---|:---|
| 主动 `unlock()` | 取消定时任务，删除锁 |
| JVM 崩溃 / 网络断连 | 定时任务在客户端，客户端没了就停了，Redis 锁 30 秒后自动过期 |
| 续期脚本发现锁不存在 | 返回 0，不再递归安排下次续期 |

**不会死锁**——只要客户端不在了，30 秒内锁必然释放。

---

## 六、和锁重试机制的关系

| 机制 | 问题 | 解决方式 |
|:---|:---|:---|
| 锁重试 | 拿不到锁怎么办 | Semaphore + Pub/Sub 通知唤醒 |
| Watchdog | 拿到锁但业务没跑完怎么办 | 定时续期，锁永不过期（只要持有者活着） |

**两者配合的完整流程：**

```
线程1 调用 lock()
  ├─ 加锁成功 → Watchdog 启动，每 10 秒续期
  ├─ 执行业务中...
  │     ├─ 10 秒：续期 → 30 秒
  │     ├─ 20 秒：续期 → 30 秒
  │     └─ ...
  ├─ unlock()
  │     ├─ 执行解锁 Lua → 删除锁 + publish 释放消息
  │     └─ 关闭 Watchdog

线程2 一直在等（信号量阻塞）
  └─ 收到 publish 消息 → 唤醒 → 抢锁成功 → 自己的 Watchdog 启动
```

---

## 七、总结

-   **目的**：防止业务阻塞导致锁自动过期，被其他线程趁虚而入
-   **触发条件**：`lock()` / `tryLock(waitTime)` 时不手动设持有时间
-   **工作原理**：10 秒一次定时任务，用 Lua 脚本检查锁归属并延长过期时间到 30 秒
-   **停止**：`unlock()` 取消任务；客户端崩了，锁 30 秒自然过期
-   **配置**：`Config.setLockWatchdogTimeout(long timeout)` 可修改默认 30 秒
-   **重要**：手动指定锁持有时间（`tryLock(wait, leaseTime, unit)`）会关闭 Watchdog