

```java
@Component
public class RedisIdWorker {

    @Resource
    private StringRedisTemplate stringRedisTemplate;
    /**
     * 开始时间戳
     */
    private static final long BEGIN_TIMESTAMP = 1640995200;
    /**
     * 序列化位数
     */
    private static final int COUNT_BITS = 32;

    /**
     * 生成分布式ID
     * @param keyPrefix
     * @return
     */
    public long nextId(String keyPrefix){
        // 1、生成时间戳
        LocalDateTime now = LocalDateTime.now();
        long nowSecond = now.toEpochSecond(ZoneOffset.UTC);
        long timestamp = nowSecond - BEGIN_TIMESTAMP;
        // 2、生成序列号
        // 以当天的时间戳为key，防止一直自增下去导致超时，这样每天的极限都是 2^{31}
        String date = now.format(DateTimeFormatter.ofPattern("yyyyMMdd"));
        Long count = stringRedisTemplate.opsForValue().increment(ID_PREFIX + keyPrefix + ":" + date);
        // 3、拼接并返回
        return timestamp << COUNT_BITS | count;
    }

    public static void main(String[] args) {
        LocalDateTime time = LocalDateTime.of(2022, 1, 1, 0, 0, 0);
        long second = time.toEpochSecond(ZoneOffset.UTC);
        System.out.println("second = " + second);
    }
}

```


在分布式高并发环境下，利用 Redis 生成一个全局唯一、趋势递增的 64 位 Long 型整数 ID。


### 一、 整体结构拆解

代码将 Java 的 64 位 `long` 类型整数拆分为两部分：

- **高 32 位**：存储**时间戳差值**（以秒为单位）。
    
- **低 32 位**：存储**Redis 当天的自增序列号**。
    
```Plaintext
|------------- 高 32 位 (时间戳差值) -------------|------------- 低 32 位 (Redis 序列号) -------------|
```

### 二、 逐行代码分析

#### 1. 成员变量与常量定义

```Java
// 1. 基准时间戳（2022年1月1日 00:00:00 的秒数：1640995200）
private static final long BEGIN_TIMESTAMP = 1640995200;

// 2. 序列号占用的位数（32位）
private static final int COUNT_BITS = 32;
```

- **基准时间戳**：如果不减去基准时间，当前的秒数直接转二进制会占用更多位数。用 `当前秒数 - 基准秒数`，可以把数字变小，保证高 32 位可以使用约 136 年（$2^{31} \text{秒} \approx 68 \text{年}$，若无符号可达 136 年）。
    

#### 2. `nextId` 方法逻辑拆解

##### 第一步：生成时间戳

```Java
LocalDateTime now = LocalDateTime.now();
long nowSecond = now.toEpochSecond(ZoneOffset.UTC); // 获取当前时间的 UTC 秒数
long timestamp = nowSecond - BEGIN_TIMESTAMP;       // 计算相对时间戳差值
```

- 算出从 2022-01-01 到现在的秒数差。例如当前时间每过 1 秒，`timestamp` 的数值就会 `+1`。
    

##### 第二步：生成序列号

```Java
// 获取当天的年月日字符串，格式为：20260803
String date = now.format(DateTimeFormatter.ofPattern("yyyyMMdd"));

// 调用 Redis 的 INCR 命令自增
Long count = stringRedisTemplate.opsForValue().increment(ID_PREFIX + keyPrefix + ":" + date);
```

- **按天拼接 Key**：Redis Key 类似 `icr:order:20260803`。
    
- **为什么按天划分 Key**：
    
    - **防止溢出**：低 32 位能存的最大整数是 $2^{31} - 1$（约 21 亿）。如果全局只用一个 Key 持续自增，终究会突破 32 位限制。按天重置后，每天都有全新的 21 亿个序号供使用。
        
    - **方便统计**：天然记录了每天的业务请求量。
        

##### 第三步：拼接并返回 ID

```Java
return timestamp << COUNT_BITS | count;
```

- `timestamp << COUNT_BITS`：将时间戳差值左移 32 位，空出右侧的低 32 位（全 0）。
    
- `| count`：按位或运算，将 Redis 返回的自增数字填入右侧空出的低 32 位中。
    

#### 3. `main` 方法的作用

```Java

public static void main(String[] args) {
    LocalDateTime time = LocalDateTime.of(2022, 1, 1, 0, 0, 0);
    long second = time.toEpochSecond(ZoneOffset.UTC);
    System.out.println("second = " + second);
}
```

- 这段代码是开发者用来**获取基准时间戳**的辅助测试方法。运行后打印出的 `1640995200`，就是常量 `BEGIN_TIMESTAMP` 的赋值来源。
    

### 三、 隐藏的优化细节（注意点）

1. **自动拆箱空指针风险（NPE）**：
    
    `stringRedisTemplate.opsForValue().increment(...)` 返回的是包装类型 `Long`。直接与 `long` 做位运算时会触发自动拆箱，如果 Redis 操作因网络等原因返回 `null`，可能会报 `NullPointerException`。生产代码通常会加一层非空判断（如 `count == null ? 0L : count`）。
    
2. **生成 ID 的特性**：
    
    - **全局唯一**：高位时间戳随时间推移递增，同一秒内低位 `count` 递增。
        
    - **趋势递增**：由于高位是时间，生成的 Long 型整数天然随时间变大，对 MySQL B+Tree 主键索引非常友好。