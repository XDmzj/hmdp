
# Redisson 是什么？

**官方定义：** Redisson 是一个在 Redis 基础上实现的 Java 驻内存数据网格（In-Memory Data Grid）。

**通俗说**
**Redisson**：**高级功能工具包**。它在 Lettuce 这种底层通信工具之上，把分布式锁、队列、信号量这些复杂逻辑替你写好了。你不用再关心怎么发命令、写 Lua 脚本，直接调用 `lock.lock()` 这种更符合 Java 习惯的方法就行


## 为什么要用redisson
经过[[集群下的一人一单#优化1]][[集群下的一人一单#优化2]]之后，实现的分布式锁已经达到**生产可用级别**了，但是还不够完善

有以下问题：
分布式锁不可重入：不可重入是指同一线程不能重复获取同一把锁。比如，方法A中调用方法B，方法A需要获取分布式锁，方法B同样需要获取分布式锁，线程1进入方法A获取了一次锁，进入方法B又获取一次锁，由于锁不可重入，所以就会导致死锁

分布式锁不可重试：获取锁只尝试一次就返回false，没有重试机制，这会导致数据丢失，比如线程1获取锁，然后要将数据写入数据库，但是当前的锁被线程2占用了，线程1直接就结束了而不去重试，这就导致数据发生了丢失

分布式锁超时释放：超市释放机机制虽然一定程度避免了死锁发生的概率，但是如果业务执行耗时过长，期间锁就释放了，这样存在安全隐患。锁的有效期过短，容易出现业务没执行完就被释放，锁的有效期过长，容易出现死锁，所以这是一个大难题！

我们可以设置一个较短的有效期，但是加上一个 心跳机制 和 自动续期：在锁被获取后，可以使用心跳机制并自动续期锁的持有时间。通过定期发送心跳请求，显示地告知其他线程或系统锁还在使用中，同时更新锁的过期时间。如果某个线程持有锁的时间超过了预设的有效时间，其他线程可以尝试重新获取锁。

主从一致性问题：如果Redis提供了主从集群，主从同步存在延迟，线程1在旧主节点获取了锁，在锁信息未同步到其余从节点时主节点宕机，从节点上位，线程2在新主节点又一次获取了锁
此时有两个线程获取了锁，这不符合业务逻辑

---

## 和之前用的 `StringRedisTemplate` 有什么区别？

| 维度           | `StringRedisTemplate` | Redisson                 |
| :----------- | :-------------------- | :----------------------- |
| 定位           | 底层 Redis 命令的薄封装       | 高级分布式服务框架                |
| 分布式锁         | 需要自己写命令、Lua脚本         | 直接 `lock.lock()`，开箱即用    |
| 可重入锁         | 需要自己用 Hash 实现         | 内置，`state` 计数器自然支持       |
| 锁自动续期        | 需要自己写看门狗              | 内置 Watchdog，自动续期         |
| 读写锁/信号量      | 需要自己实现                | 原生提供 `RReadWriteLock` 等  |
| 红锁 (RedLock) | 需要自己实现完整算法            | 一行代码获取 `RedissonRedLock` |
| 集群支持         | 需要自己处理主从切换            | 自动感知集群拓扑，处理故障转移          |
| 异步模型         | 同步阻塞                  | 完整支持异步、响应式编程             |

**一句话：** `StringRedisTemplate` 是给你 Redis 原语的工具箱，Redisson 是帮你把分布式场景里的轮子都造好了。



---

# 基本使用（作为分布式锁）

![[Pasted image 20260807142053.png]]

![[Pasted image 20260807142102.png]]

好的，以下是用你自己的理解和语言整理的笔记内容，适合直接粘贴到 Obsidian：



## Redisson 分布式锁使用流程

### 核心理解
Redisson 本质上就是一个**集成好的、更高级的 Redis 工具包**。它在底层通信（Lettuce）之上，把分布式锁、看门狗续期、红锁等复杂逻辑都封装好了，直接用就行。

### 基本使用流程

**第一步：导入依赖**

在 `pom.xml` 中添加 Redisson 的 Spring Boot Starter：

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.23.5</version>
</dependency>
```

**第二步：创建配置**

可以用 YAML 配置，也可以写 Java Config 类，二选一。

-   **YAML 配置**：
    ```yaml
    spring:
      redis:
        redisson:
          config: |
            singleServerConfig:
              address: "redis://127.0.0.1:6379"
    ```

-   **Java Config 类**：
    ```java
    @Configuration
    public class RedissonConfig {
        @Bean
        public RedissonClient redissonClient() {
            Config config = new Config();
            config.useSingleServer().setAddress("redis://127.0.0.1:6379");
            return Redisson.create(config);
        }
    }
    ```

**第三步：代码中使用**

注入 `RedissonClient`，然后 `tryLock` 加锁，`unlock` 解锁，正常使用即可。

```java
@Autowired
private RedissonClient redissonClient;

public void doSomething() {
    RLock lock = redissonClient.getLock("lock:key");
    
    // 尝试加锁，拿不到直接返回 false
    boolean locked = lock.tryLock();
    if (!locked) {
        return;
    }
    
    try {
        // 核心业务逻辑
    } finally {
        lock.unlock();
    }
}
```

### 三种加锁模式

| 方式 | 代码 | 说明 |
|:---|:---|:---|
| 一直等 | `lock.lock()` | 拿不到锁就死等，Watchdog 自动续期 |
| 试一次 | `lock.tryLock()` | 拿不到立即返回 false |
| 带超时 | `lock.tryLock(10, 30, TimeUnit.SECONDS)` | 最多等 10 秒，锁有效期 30 秒（手动设有效期，Watchdog 不启动） |



---

# Redisson可重入锁的原理

流程图

![[Pasted image 20260807154146.png]]



Redisson可重入锁采用了redis的hash类型，多了state这个重入计数字段，所以不再用setnx命令，只能用lua保证获取锁操作的原子性
即：
Redisson 可重入锁用 Hash 存储线程标识和重入次数。由于需要多条命令配合操作，它在内部通过 Lua 脚本实现加锁逻辑，脚本中用 `exists` 等命令来保证“不存在才创建”的互斥性，与 `SETNX` 原理一致，但不再单独使用 `SETNX` 命令


获取锁操作脚本
![[Pasted image 20260807154137.png]]


释放锁操作脚本
![[Pasted image 20260807154504.png]]


可重入锁在代码写法上和普通锁**完全一样**，因为 Redisson 的 `RLock` 本身就内置了可重入能力。你不需要额外做任何事。