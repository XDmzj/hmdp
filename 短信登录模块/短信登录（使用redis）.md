
# 问题分析

![[Pasted image 20260729105809.png]]
在传统的 Session 方案中：

- **存在分布式集群问题**：Session 默认保存在单台服务器的内存中。一旦部署多台 Tomcat 搞负载均衡，用户的请求打到 A 服务器登录成功，下一次请求打到 B 服务器就会变成未登录状态（即 **Session 共享问题**）。
    
- **内存开销大**：随着在线用户增多，服务器内存容易暴涨。
    

改为 **Redis + Token** 方案后：

- **天然支持分布式**：Redis 作为独立的集中式缓存数据库，所有后端节点共同访问一份 Redis 实例，共享登录凭证。
    
- **无状态解耦**：服务端无需维护本地会话，直接将随机生成的 Token 存储在 Redis 中，响应给客户端即可。


# 原理流程图

![[Pasted image 20260729141725.png]]

![[Pasted image 20260729141732.png]]


# 代码

## 验证码发送


Service层，将用户信息存入session改为存入redis，
![[Pasted image 20260729145614.png]]
注意：
redis中验证码和用户键值对要设置有效期
redis中key要使用前缀来区分，不能直接写用户电话


## 登录


```java
@Override  
public Result login(LoginFormDTO loginForm, HttpSession session){  
    //1.校验手机号  
    String phone = loginForm.getPhone();  
    if(RegexUtils.isPhoneInvalid(phone)){  
        return Result.fail("手机号格式错误！");  
    }  
  
    //2.从redis中获取验证码并进行校验  
    String cacheCode = stringRedisTemplate
					    .opsForValue()
					    .get(LOGIN_CODE_KEY+phone);  
    String code = loginForm.getCode();  
    if(cacheCode==null || !cacheCode.equals(code)){  
        //3.不一致报错  
        return Result.fail("验证码错误");  
    }  
  
    //4.一致，根据手机号查询用户,select * from tb_user where phone = ?  
    User user = query().eq("phone", phone).one();  
  
    //5.判断用户是否存在  
    if(user==null){  
        //6.不存在，创建用户并保存,只需要填充phone和nickname字段即可  
        user = createUserWithPhone(phone);  
    }  
  
    //7.保存用户信息到redis  
    //7.1随机生成token,作为登录令牌  
    String token = UUID.randomUUID().toString();  
    //7.2将user对象转为hash存储  
    UserDTO userDTO = BeanUtil.copyProperties(user,UserDTO.class);  
    Map<String, Object> userMap = BeanUtil.beanToMap(userDTO,new HashMap<>(), 
//解决id字段是整型而不是字符串类型       
 CopyOptions.create()
.setIgnoreNullValue(true)
.setFieldValueEditor((fieldName,fieldValue)->fieldValue.toString())  
    );  
    //7.3存储  
    String tokenKey = LOGIN_USER_KEY+token;  
    stringRedisTemplate.opsForHash().putAll(tokenKey,userMap);   //添加hash对象  
    stringRedisTemplate.expire(tokenKey, LOGIN_USER_TTL, TimeUnit.MINUTES);  
  
    //8.返回token  
    return Result.ok(token);  
}
```


`createUserWithPhone`:
![[Pasted image 20260729150231.png]]

这段代码是黑马点评项目中 **基于 Redis 实现短信验证码登录/注册** 的核心业务逻辑（`UserServiceImpl` 中的 `login` 方法）。

它的核心使命是：**校验验证码 $\rightarrow$ 登录/自动注册用户 $\rightarrow$ 生成 Token 并把脱敏用户信息存入 Redis $\rightarrow$ 返回 Token 给前端**。

### 代码逐句拆解

1. 校验手机号格式
```Java
String phone = loginForm.getPhone();
if(RegexUtils.isPhoneInvalid(phone)){
    return Result.fail("手机号格式错误！");
}
```

- **逻辑**：从前端传来的 `LoginFormDTO` 中提取手机号，利用正则表达式工具类 `RegexUtils` 校验其是否符合中国大陆手机号规范（11位数字、1开头等）。
- **作用**：防御性编程，避免无效或恶意手机号进入后端的后续复杂流程。


2. 校验验证码
```Java
String cacheCode = stringRedisTemplate
        .opsForValue()
        .get(LOGIN_CODE_KEY + phone);
String code = loginForm.getCode();
if(cacheCode == null || !cacheCode.equals(code)){
    return Result.fail("验证码错误");
}
```

- **逻辑**：
    
    - 用手机号拼接前缀（如 `login:code:13812345678`）作为 Key，去 Redis 中获取之前发送短信时存入的验证码 `cacheCode`。
        
    - 将 Redis 拿到的 `cacheCode` 与前端用户输入的 `code` 进行比对。
        
- **细节**：如果 `cacheCode == null`，说明验证码**已过期（超过了 TTL）** 或者 **压根没有发送过验证码**；如果两者不匹配，直接返回“验证码错误”。

3. 查询或自动注册用户
```Java
User user = query().eq("phone", phone).one();

if(user == null){
    user = createUserWithPhone(phone);
}
```
- **逻辑**：
    - 使用 MyBatis-Plus 提供的 `query().eq("phone", phone).one()` 在数据库 `tb_user` 表中按手机号精准查找用户。
    - **自动注册**：如果 `user == null`（老用户未查到），说明是新用户首次登录。调用辅助方法 `createUserWithPhone(phone)` 自动为其创建账号（生成随机昵称如 `user_xxxx`、默认头像，并插入数据库）。
        

4. 生成登录凭证 (Token)
```Java
String token = UUID.randomUUID().toString();
```

- **逻辑**：使用 `UUID` 生成一个唯一的 36 位无规则随机字符串（如 `c8a2d1e0-4a8f-4b12-9c3f-5d2e7f8a9b0c`）作为用户的登录凭证 **Token**。
    
- **作用**：客户端后续请求所有接口时，都需要在 Header 中携带这个 Token，用来标识“我是谁”。
    

 5. 用户数据脱敏 (User $\rightarrow$ UserDTO)
```Java
UserDTO userDTO = BeanUtil.copyProperties(user, UserDTO.class);
```

- **逻辑**：使用 Hutool 的 `BeanUtil` 工具类将数据库实体对象 `User` 拷贝为 `UserDTO`。
    
- **为什么不能直接用 `User`？**
    
    - **数据安全/脱敏**：`User` 实体类中可能包含密码（`password`）、创建时间、手机号等敏感信息。存入 Redis 时只保留 `id`、`nickname`、`icon` 等公开字段。
        
    - **节约内存**：减少 Redis 中的存储开销。
        

6. 数据类型转换 (UserDTO $\rightarrow$ Map)
```Java
Map<String, Object> userMap = BeanUtil.beanToMap(userDTO, new HashMap<>(),
        CopyOptions.create()
                   .setIgnoreNullValue(true)
                   .setFieldValueEditor((fieldName, fieldValue) -> fieldValue.toString())
);
```
- **核心痛点与避坑点（重点）**：
    - 项目中使用的 Redis 序列化器是 `StringRedisTemplate`，它要求 Hash 结构中的 **Key 和 Value 必须全是 `String` 类型**。
    - 但 `UserDTO` 中的 `id` 字段是 `Long` 类型（整型）。如果直接用普通的 `opsForHash().putAll()` 存入，Spring 会因为无法将 `Long` 自动强转为 `String` 而抛出 **`ClassCastException`**。
- **解决手段**：
    - 利用 Hutool 的 `CopyOptions` 策略，设置 `setFieldValueEditor`。
    - 当将对象转 Map 时，**把属性的值统一调用 `.toString()` 强制转为字符串**（如将 `Long 1001L` 转为 `"1001"`），保证 Map 里的所有 Value 均为 `String` 类型。
        

7. 写入 Redis 并设置 TTL
```Java
String tokenKey = LOGIN_USER_KEY + token;
stringRedisTemplate.opsForHash().putAll(tokenKey, userMap);   // 添加 Hash 对象
stringRedisTemplate.expire(tokenKey, LOGIN_USER_TTL, TimeUnit.MINUTES);
```

- **逻辑**：
    - 拼接 Key，格式为 `login:token:c8a2d1e0-xxx`。
    - 使用 Redis 的 **Hash 数据结构**（`opsForHash().putAll`）将处理好的 `userMap` 批量写入 Redis。
    - 为该 TokenKey 设置 30 分钟（`LOGIN_USER_TTL`）的**硬性过期时间**。
        

8. 返回 Token 给前端
```Java
return Result.ok(token);
```

- **逻辑**：登录成功，将生成的 `token` 字符串封装进统一响应结果 `Result` 对象中返回给前端页面。前端收到后会将它存在 `localStorage` 中。




## 登录验证



拦截器
```java
//预拦截  
@Override  
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {  
    //1.获取请求头中的token  
    String token = request.getHeader("authorization");  
    if(StrUtil.isBlank(token)){  
        return true;  
    }  
  
    //2.基于token获取redis中的用户  
    String key = RedisConstants.LOGIN_USER_KEY + token;  
    Map<Object, Object> userMap = stringRedisTemplate.opsForHash().entries(key);  
    //3.判断用户是否在存在  
    if(userMap.isEmpty()){  
        return true;  
    }  
    //5.将查询到的hash数据转为UserDTO对象  
    UserDTO userDTO = BeanUtil.fillBeanWithMap(userMap,new UserDTO(),false);  
  
    //6.存在，保存用户信息到ThreadLocal  
    UserHolder.saveUser(userDTO);  
  
    //7.刷新token的有效期  
    stringRedisTemplate.expire(key,RedisConstants.LOGIN_USER_TTL, TimeUnit.MINUTES);  
  
    //8.放行  
    return true;  
}
```

这段代码是黑马点评项目中 **`RefreshTokenInterceptor`（Token 刷新拦截器）** 的核心实现方法 `preHandle`。

它的根本作用是：**“无差别地尝试刷脸”** —— 无论用户访问的是什么接口，只要请求头里带了有效 Token，就去 Redis 里把用户信息查出来存入 `ThreadLocal`，并**把该 Token 在 Redis 里的过期时间重新重置为 30 分钟（无感续期）**。

### 代码逐行拆解

1. 获取请求头中的 Token
```Java
String token = request.getHeader("authorization");
if(StrUtil.isBlank(token)){
    return true;
}
```

- **逻辑**：从前端发来的 HTTP 请求头 `Header` 中获取名为 `authorization` 的字段（即前端登录后保存的 UUID Token）。
    
- **为什么 Blank 就 `return true` 放行？**
    - 因为这个拦截器的职责**不是阻止访问**，而是**尝试刷新 Token**。
    - 如果请求没带 Token（比如用户访问的是首页、店铺列表等无需登录的公开页面），就直接放行给后续流程，**拦截重任交给下一个 `LoginInterceptor`**。

2. 基于 Token 查询 Redis 中的 Hash 数据
```Java
String key = RedisConstants.LOGIN_USER_KEY + token;
Map<Object, Object> userMap = stringRedisTemplate.opsForHash().entries(key);
```

- **逻辑**：拼接 Redis 中的 Key（格式如 `login:token:uuid-xxx`），通过 `opsForHash().entries(key)` 从 Redis 中提取以 Hash 格式存储的用户信息，返回一个 `Map`。

3. 判断 Redis 中是否存在该用户
```Java
if(userMap.isEmpty()){
    return true;
}
```

- **逻辑**：如果从 Redis 拿到的 `userMap` 是空的，说明 **Token 已经过期被 Redis 删除了**，或者传入的是一个**伪造的无效 Token**。
    
- **为什么同样 `return true`？**
    
    - 同样是因为该拦截器只管“有则刷，无则过”。既然 Token 失效了，就不存 `ThreadLocal`，直接放行。如果是需要登录的接口，后面的 `LoginInterceptor` 会发现 `ThreadLocal` 是空的，进而抛出 401 并拦截


4. 将 Map 数据转换为 UserDTO
```Java
UserDTO userDTO = BeanUtil.fillBeanWithMap(userMap, new UserDTO(), false);
```

- **逻辑**：利用 Validation/Hutool 工具类，把 Redis 查出来的 `Map<Object, Object>` 自动填充映射回 Java 的 `UserDTO` 对象。
    
- **参数 `false`**：表示在填充属性时，如果遇到了忽略错误的异常或者找不到对应的字段时，不抛出异常。

5. 保存用户信息到 `ThreadLocal`
```Java
UserHolder.saveUser(userDTO);
```

- **逻辑**：将当前登录用户的 `userDTO` 存入 `ThreadLocal` 工具类 `UserHolder` 中。
    
- **作用**：`ThreadLocal` 与当前线程绑定。这意味着在这个请求后续经过的所有的 `Controller`、`Service` 代码中，只要调用 `UserHolder.getUser()` 就能随时拿到当前登录人的 ID 等信息，**无需在各个方法间层层传递用户参数**。

6. 刷新 Token 的有效期（核心难点）
```Java
stringRedisTemplate.expire(key, RedisConstants.LOGIN_USER_TTL, TimeUnit.MINUTES);
```

- **逻辑**：重新给 Redis 里的这个 Key 设置过期时间（例如 30 分钟）。
    
- **目的（无感续期）**：**只要用户还在持续操作系统**（比如每隔几分钟点一下页面），每次发请求经过这个拦截器，Token 的 TTL 就会被重新重置为 30 分钟。这样用户就不会出现“正在刷着页面，突然硬性掉线”的糟糕体验。

7. 放行
```Java
return true;
```

- **逻辑**：返回 `true`，告诉 Spring MVC 当前拦截器执行完毕，允许请求进入下一个拦截器或目标 `Controller`。
    
#### 优化思路

![[Pasted image 20260729165905.png]]

 核心设计思想小结

1. **双拦截器配合**：
    
    - **`RefreshTokenInterceptor`（本段代码）**：拦截 **`/**`（所有请求）**，只负责 **查 Redis -> 存 ThreadLocal -> 刷 TTL**。
        
    - **`LoginInterceptor`（第二个拦截器）**：拦截 **需要登录的敏感请求**，只负责检查 **`ThreadLocal` 是否为空**。若空则直接 `response.setStatus(401)` 并 `return false`。
        
2. **职责分离**：如果不分成两个拦截器，只拦截敏感接口，那么用户如果在 30 分钟内一直在浏览公开页面（如首页、商品详情），Token 就会因为没有触发敏感接口而过期，导致用户点击“下单”时突然提示未登录。这种双拦截器设计完美避开了该问题。


# 补充
针对你提出的这 4 个疑惑，我们逐一进行解答：
### 一、 `CopyOptions` 是什么东西？

`CopyOptions` 是第三方工具库 **Hutool**（`cn.hutool.core.bean.copier.CopyOptions`）中提供的一个**配置选项类**，专门配合 `BeanUtil` 工具类使用。

它相当于一个“复制策略自定义选项”，用来控制对象与对象、或者对象与 Map 之间进行属性拷贝时的细节行为。

常见配置项：

- `setIgnoreNullValue(true)`：如果源对象的某个属性是 `null`，拷贝时忽略它，不覆盖目标属性。
    
- `setIgnoreError(true)`：拷贝过程中如果类型转换出错，忽略异常继续拷贝。
    
- `setFieldValueEditor((fieldName, fieldValue) -> ...)`：**属性值编辑器**。这是代码里的核心——在拷贝属性值前，允许你对每个字段的值进行动态修改。
    

> **在这个代码里的作用**：
> 
> 因为 `UserDTO` 中的 `id` 属性是 `Long` 类型，而 `StringRedisTemplate` 要求传入的 Map 值必须全部是 `String`。通过 `setFieldValueEditor` 将每一个字段的值强转为字符串（`fieldValue.toString()`），从而解决了类型转换异常。

### 二、 `expire` 是什么？

`expire` 是 Redis 官方提供的一个核心命令，在 Java 的 `stringRedisTemplate` 中被封装为了一个方法：
```Java
stringRedisTemplate.expire(key, timeout, unit);
```
#### 概念与作用：

1. **设置过期时间（TTL - Time To Live）**：告诉 Redis 这个 Key 只能存活多久（比如 30 分钟）。
    
2. **内存自动清理**：时间一到，Redis 会自动将该 Key 彻底删除，释放内存空间。
    
3. **安全防风险**：
    
    - **验证码场景**：如果验证码不加 `expire`，存进去就永远不会消失，极易被暴力破解或造成内存泄漏。
        
    - **登录 Token 场景**：设置 30 分钟过期，保障账号安全（长时间不操作自动下线）。
        

### 三、 `stringRedisTemplate` 以外，还有别的模板类吗？

有的。在 Spring Data Redis 中，核心的模板类主要有以下几个：

#### 1. `RedisTemplate<K, V>`（通用模板类）

- **特点**：支持泛型，可以指定任意 Key 和 Value 的类型（如 `RedisTemplate<String, Object>` 或 `RedisTemplate<Object, Object>`）。
    
- **序列化机制**：默认使用的是 Java 原生的 JDK 序列化（`JdkSerializationRedisSerializer`）。
    
    - _缺点_：写入 Redis 的数据会带有 JDK 乱码前缀（如 `\xac\xed\x00\x05...`），不仅占用空间，且在 Redis GUI 中极难看懂。
        
    - _解决_：通常需要手动给它配置 `Jackson2JsonRedisSerializer` 或 `GenericJackson2JsonRedisSerializer` 序列化器，把对象直接转成 JSON 字符串存入。
        

#### 2. `StringRedisTemplate`（专门针对 String 的简化模板类）

- **继承关系**：它是 `RedisTemplate<String, String>` 的子类。
    
- **特点**：Key 和 Value **默认全强制使用字符串序列化器**（`StringRedisSerializer`）。
    
- **优势**：
    
    - 性能极高，没有任何冗余的 Java 序列化头信息。
        
    - 在 Redis 客户端工具里看数据非常清晰，跨语言（如 Node.js、Python）读写极其友好。
        
    - **黑马点评等绝大多数主流项目首选该模板类**（遇到对象时手动转 JSON 或转为 String Map 存储）。
        

### 四、 在短信登录模块中，体现了 Redis 的哪些作用？

在这个登录与授权模块中，Redis 扮演了 **4 个关键角色**：

|**Redis 关键作用**|**对应业务应用**|**解决的核心痛点 / 优势**|
|---|---|---|
|**1. 高性能分布式缓存 (Cache)**|存储短信验证码 (`login:code:phone`)|验证码读写极其频繁且声明周期短。直接存 Redis 比查写 MySQL 数据库快上百倍，极大保护了主数据库。|
|**2. 集中式 Session (共享会话)**|存储登录 Token 及其 User 状态 (`login:token:uuid`)|**打破单机限制**。无论后端部署了多少台 Tomcat 搞负载均衡，所有服务器都共享同一个 Redis，完美解决集群下的用户鉴权问题。|
|**3. 数据过期与生命周期管理 (TTL)**|验证码 2 分钟失效、Token 30 分钟失效与无感续期|利用 Redis 原生的 `expire` 特性，无需写后台定时任务脚本去删库，交由 Redis 内核自动淘汰过期数据。|
|**4. 丰富高效的数据结构 (Hash)**|使用 Redis Hash 存储 `UserDTO` 字段|相比将对象序列化为整体 JSON 字符串，Hash 结构将属性作为 KV 散列存储：占用内存更小、且允许未来独立读取或修改单个字段（如单独更新 nickname）。|