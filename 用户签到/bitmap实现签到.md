### 一、业务场景

用户每天签到一次，系统需要：

1. 记录某用户今天是否已签到
    
2. 统计某用户连续签到天数
    
3. 统计某用户某月总签到天数
    

**传统做法**：数据库存签到表，每个用户每天一条记录。数据量巨大。

**Redis BitMap 方案**：用一个 bit 位表示一天，签到 = 1，未签 = 0。极大节省内存
![[Pasted image 20260822121716.png]]

### 二、BitMap 核心思想

| 概念        | 说明                    |
| --------- | --------------------- |
| BitMap 本质 | 一个字符串，每个 bit 位是 0 或 1 |
| 一个用户一个月   | 需要 30~31 个 bit        |
| 一个用户一年    | 需要 365 个 bit ≈ 46 字节  |
| 一亿用户一年    | 约 4.6 GB（数据库可能要几十 TB） |


### 三、核心命令

|操作|命令|
|---|---|
|签到（第 N 天设为 1）|`SETBIT key N 1`|
|查看某天是否签到|`GETBIT key N`|
|统计签到总天数|`BITCOUNT key`|
|统计连续签到天数|`BITFIELD key GET u31 0`|



# 实现

![[Pasted image 20260822121746.png]]


service
```java
    /**
     * 用户签到
     *
     * @return
     */
    @Override
    public Result sign() {
        // 获取当前登录用户
        Long userId = ThreadLocalUtls.getUser().getId();
        // 获取日期
        LocalDateTime now = LocalDateTime.now();
        // 拼接key
        String keySuffix = now.format(DateTimeFormatter.ofPattern(":yyyyMM"));
        String key = USER_SIGN_KEY + userId + keySuffix;
        // 获取今天是本月的第几天
        int dayOfMonth = now.getDayOfMonth();
        // 写入Redis SETBIT key offset 1
        stringRedisTemplate.opsForValue().setBit(key, dayOfMonth - 1, true);
        return Result.ok();
    }

```