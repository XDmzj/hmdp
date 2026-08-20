## Redis GEO 数据结构

---

### 一、GEO 是什么？

GEO 是 Redis 3.2 引入的**地理位置数据类型**，用来存储经纬度，并支持计算距离、查找附近位置等操作。

**典型场景**：

- 附近的商户
- 附近的人
- 计算两个位置的距离

---

### 二、底层原理

GEO **本质上就是 ZSet**。Redis 用 ZSet 来存储地理位置信息，member 是位置名称，score 是把经纬度编码后的一个数值。

| ZSet 概念 | GEO 概念 |
|:---|:---|
| Key | 地理位置集合（如 `shop:geo`） |
| member | 位置名称（如商户 ID） |
| score | 经纬度编码成的 Geohash 值 |

**所以 GEO 继承了 ZSet 的所有特性**，也可以用 ZSet 的命令来操作 GEO 数据。

---

### 三、核心命令

#### 1. `GEOADD`：添加位置

```bash
GEOADD shop:geo 116.397128 39.916527 "shop:1001"
GEOADD shop:geo 116.481488 39.990464 "shop:1002"
```

- `shop:geo`：地理集合的 Key
- `116.397128 39.916527`：经度 纬度
- `"shop:1001"`：位置名称（member）

#### 2. `GEODIST`：计算两点距离

```bash
GEODIST shop:geo shop:1001 shop:1002 km
```

- 返回两个位置之间的距离
- 单位：`m`（米）、`km`（千米）、`mi`（英里）、`ft`（英尺）

#### 3. `GEOSEARCH`：搜索附近位置（Redis 6.2+）

```bash
GEOSEARCH shop:geo FROMLONLAT 116.397128 39.916527 BYRADIUS 5 km
```

- `FROMLONLAT`：以指定经纬度为圆心
- `BYRADIUS 5 km`：半径 5 公里内

#### 4. `GEOSEARCHSTORE`：搜索并存储结果

```bash
GEOSEARCHSTORE result:geo shop:geo FROMLONLAT 116.397128 39.916527 BYRADIUS 5 km
```

- 把搜索结果存到另一个 GEO 集合中

---

### 四、黑马点评中的使用

**业务场景**：根据用户当前位置，查找附近的商户。

```java
public Result queryShopByGeo(Integer typeId, Double x, Double y) {
    String key = SHOP_GEO_KEY + typeId;  // 如 shop:geo:1
    
    // GEOSEARCH 查询附近商户
    List<GeoResult<RedisGeoCommands.GeoLocation<String>>> results = 
        stringRedisTemplate.opsForGeo().search(
            key,
            GeoReference.fromCoordinate(x, y),
            new Distance(5, Metrics.KILOMETERS),
            RedisGeoCommands.GeoSearchCommandArgs.newGeoSearchArgs()
                .includeDistance()     // 返回距离
                .sortAscending()       // 按距离升序
                .limit(10)             // 只取前 10 个
        );
    
    // 解析结果
    List<Map<String, Object>> list = new ArrayList<>();
    for (GeoResult<RedisGeoCommands.GeoLocation<String>> result : results) {
        String shopId = result.getContent().getName();   // 商户ID
        double distance = result.getDistance().getValue(); // 距离
        // 查数据库获取商户详情...
    }
    return Result.ok(list);
}
```

**对应的 Redis 命令**：

```bash
GEOSEARCH shop:geo:1 FROMLONLAT 116.397128 39.916527 BYRADIUS 5 km 
    ASC COUNT 10 WITHCOORD WITHDIST
```

---

### 五、数据准备

在项目启动时，把数据库中的商户经纬度加载到 Redis：

```java
@PostConstruct
public void loadShopGeo() {
    // 1. 查数据库所有商户
    List<Shop> shopList = shopService.list();
    
    // 2. 按类型分组，写入 Redis GEO
    Map<Long, List<Shop>> map = shopList.stream()
            .collect(Collectors.groupingBy(Shop::getTypeId));
    
    for (Map.Entry<Long, List<Shop>> entry : map.entrySet()) {
        Long typeId = entry.getKey();
        String key = SHOP_GEO_KEY + typeId;
        
        for (Shop shop : entry.getValue()) {
            stringRedisTemplate.opsForGeo().add(key, 
                new Point(shop.getX(), shop.getY()),  // 经纬度
                shop.getId().toString()               // member
            );
        }
    }
}
```

---

### 六、GEO 和 ZSet 的关系

| 操作 | GEO 命令 | ZSet 对应 |
|:---|:---|:---|
| 添加 | `GEOADD` | `ZADD` |
| 查询全部 | `GEOPOS` | `ZSCORE` |
| 删除 | `ZREM` | `ZREM` |
| 范围查询 | `GEOSEARCH` | `ZRANGEBYSCORE` |

**GEO 本质上是对 ZSet 的封装**，把经纬度编码成 score 存储，查询时再解码计算距离。

---

### 七、总结

| 问题 | 答案 |
|:---|:---|
| GEO 是什么？ | Redis 地理位置数据类型 |
| 底层是什么？ | ZSet |
| 核心命令 | `GEOADD`、`GEODIST`、`GEOSEARCH` |
| 典型场景 | 附近商户、附近的人 |
| 数据怎么存的？ | member = 位置名，score = Geohash 编码的经纬度 |
| 和 ZSet 区别？ | GEO 是 ZSet 的上层封装，提供地理运算能力 |

**一句话：GEO 就是利用 ZSet 存储经纬度，并提供距离计算、范围搜索等地理功能的扩展类型。**