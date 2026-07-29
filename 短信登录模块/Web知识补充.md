### Session、Cookie 和 Token
在 Web 开发和接口鉴权中，**Session、Cookie 和 Token** 是用来解决 HTTP 无状态协议下“**如何识别用户身份**”的三种核心机制。它们并不是孤立的，而是有着密切的演进与协作关系。

- **Cookie**：**存储在浏览器端**的一段小文本数据。每次浏览器向服务器发送请求时，会自动在请求头（`Cookie` Header）中带上它。
    
- **Session**：**存储在服务器端**的用户会话数据结构。服务器在内存/数据库/Redis 中开辟空间，用来保存当前登录用户的状态。
    
- **Token**：**存在客户端**的一段加密签名字符串（如 JWT）。它包含了用户身份信息及签名，服务器**无需存储**，通过解密验证其有效性。

两两之间的关系与演进

1. Session 与 Cookie 的关系：【传统服务端鉴权搭档】

Session 和 Cookie 是**基于服务端存储**的身份认证组合拳。

- **协作机制**：
    
    1. 用户提交账号密码登录成功。
        
    2. 服务器在服务端创建一个 Session 对象，生成一个唯一的标识符 **`SessionId`**（如 `JSESSIONID`）。
        
    3. 服务器将 `SessionId` 通过响应头（`Set-Cookie`）返回给浏览器，浏览器将其保存在 **Cookie** 中。
        
    4. 后续用户发起请求，浏览器**自动携带** Cookie 中的 `SessionId`。
        
    5. 服务器拿到 `SessionId` 去内存/Redis 中查找对应的 Session，查到即代表“已登录”。
        
- **总结**：**Cookie 是载体，Session 是内容。** Cookie 充当了把 `SessionId` 从客户端传回服务端的“快递员”。

2. Cookie 与 Token 的关系：【存储/传输方式与认证凭证】

很多人常把这两者混淆，但实际上它们**不在同一个维度**：

- **Token 是“认证凭证”**（比如无状态的 JWT），代表“你是谁”。
    
- **Cookie 是“传输/存储媒介”**，代表“数据存在哪、怎么发给服务端”。
    
- **协作机制**：
    
    - Token 既可以保存在浏览器的 `LocalStorage` / `SessionStorage` 中（通过代码手动加到请求头 `Authorization: Bearer <token>`），**也可以直接保存在 Cookie 中**。
        
    - 如果把 Token 放在 Cookie 里，利用 Cookie 的 `HttpOnly` 属性可以有效防御 XSS 攻击。


 3. Session 与 Token 的关系：【有状态 vs 无状态的架构演进】

这两者是**两种不同的认证架构设计思想**。

|**对比维度**|**Session 机制 (有状态 - Stateful)**|**Token 机制 / JWT (无状态 - Stateless)**|
|---|---|---|
|**存储位置**|服务端（内存/数据库/Redis）|客户端（LocalStorage / Cookie 等）|
|**服务器开销**|内存开销大（在线用户多时需要存大量 Session）|几乎无存储开销（只需 CPU 计算解密验证）|
|**分布式/集群**|需要做 **Session 共享**（如配置 Redis 统一存储）|**天然支持分布式集群**，任意节点拿到 Token 直接解密验证|
|**作废/注销**|非常简单，服务端直接抹掉 Redis 里的 Session 即可|较难（Token 发出后在过期前一直有效，通常需配合黑名单机制）|

 三者演进的整体脉络

1. **第一阶段 (Cookie 独角戏)**：早期直接把用户敏感信息存 Cookie 里，极不安全且体积受限。
    
2. **第二阶段 (Session + Cookie)**：将敏感数据留服务端（Session），只把无意义的 `SessionId` 存客户端（Cookie）。
    
    - _遇到瓶颈_：多台后端服务器集群时，用户请求打到 A 机器登录存了 Session，打到 B 机器就失联了（除非搞 Redis 集中存储）。跨域或 App/小程序开发对 Cookie 支持不好。
        
3. **第三阶段 (Token / JWT)**：去中心化鉴权。服务端直接把用户信息加密成一个 Token 扔给客户端，服务端**不存任何状态**，哪台服务器拿到 Token 只要用同样的密钥解密成功，就认可用户身份（黑马点评项目中用到的基于 Redis/JWT 的 Token 鉴权就是典型应用）。






### `HttpServletRequest`、`HttpServletResponse`和 `HttpSession`

在 Java Web（如 Servlet / Spring MVC）开发中，`HttpServletRequest`、`HttpServletResponse`和 `HttpSession` 是处理 Web 请求时最核心的三大对象。

它们本质上是 **Servlet 规范** 提供的 Java API，用来屏蔽底层的 TCP 网络传输细节，将原始的 HTTP 协议报文封装成结构化的 Java 对象，方便在代码中直接操作。

#### 1. HttpServletRequest  —— 请求对象

什么是 HttpRequest？

当浏览器向服务器发送一个 HTTP 请求时，Tomcat 等 Web 容器会接收到这段原始的字符串报文，并将解析后的所有数据封装成一个 `HttpServletRequest` 对象，传递给服务端的 `Controller` 或 `Servlet`。

它有什么用？【读数据】

它代表**客户端发给服务端的所有信息**，专门用来**读取**客户端传过来的数据：

- **获取请求参数**：获取 URL 查询参数或 Form 表单提交的数据（如 `request.getParameter("username")`）。
    
- **获取请求头 (Header)**：获取浏览器类型、Token、Cookie 等信息（如 `request.getHeader("authorization")`）。
    
- **获取请求体 (Body)**：读取 JSON 数据或上传的文件流。
    
- **获取客户端网络信息**：获取客户端的 IP 地址、访问路径、请求方式（GET/POST）等。
    
- **域对象作用（请求域）**：可以在同一个请求链（如转发 `forward`）中暂存数据（`request.setAttribute()`）。
    

####  2. HttpServletResponse —— 响应对象

什么是 HttpResponse？

`HttpServletResponse` 对象代表**服务端准备发回给客户端的响应**。Web 容器会创建一个空的 HttpResponse 对象传给业务代码，开发者将处理好的结果填充进去，最后容器会将其转换成 HTTP 格式的响应报文发送给浏览器。

它有什么用？【写数据】

它专门用来**控制和生成**返回给客户端的内容：

- **设置响应状态码**：设置 200 (成功)、404 (未找到)、401 (未授权)、500 (服务器错误) 等。
    
- **设置响应头 (Header)**：设置 `Content-Type`（告诉浏览器返回的是 JSON 还是 HTML）、控制缓存、设置跨域头等。
    
- **设置 Cookie**：通过 `response.addCookie(cookie)` 向浏览器写回 Cookie 数据。
    
- **写入响应体 (Body)**：通过输出流（如 `response.getWriter()`）向客户端输出 HTML 页面、JSON 字符串或文件下载流。
    
- **重定向**：调用 `response.sendRedirect("/login")` 让浏览器跳转到新页面。
    

#### 3. HttpSession —— 会话对象

什么是 HttpSession？

因为 HTTP 协议本身是**无状态**的（这一次请求和上一次请求之间没有记忆），为了知道“当前发请求的人是不是刚才登录过的那个用户”，服务端需要开辟一块空间来记忆用户状态。这块**存储在服务端内存中的数据结构**就是 `HttpSession`。

它有什么用？【记状态】

它专门用来在**同一个用户的多次请求之间共享数据**（维持会话）：

- **维护登录状态**：用户登录成功后，将用户信息放入 Session（如 `session.setAttribute("user", userDto)`）。
    
- **会话数据共享**：在后续的其他请求中，直接从 Session 中取出用户信息（如 `session.getAttribute("user")`），以判断用户是否已登录及其权限。
    
- **生命周期管理**：可以设置 Session 的过期时间（如 30 分钟无操作自动销毁），或者在用户点击注销时手动销毁 Session（`session.invalidate()`）。


> **与 Cookie 的联动**：HttpSession 依赖 Cookie。Tomcat 会自动生成一个名为 `JSESSIONID` 的 Cookie 返回给浏览器。浏览器每次发请求带上 `JSESSIONID`，服务端就能凭此找到该用户对应的 `HttpSession` 对象。

#### 总结对比

|**对象名称**|**存储/作用位置**|**生命周期**|**典型使用场景**|
|---|---|---|---|
|**`HttpServletRequest`**|仅在一次请求生命周期内有效|**非常短暂**（请求开始创建，响应结束即销毁）|读取前端传进来的参数、Header、解析 Token。|
|**`HttpServletResponse`**|用于构建写回客户端的数据|**非常短暂**（数据写回客户端后即销毁）|返回 JSON、设置 Cookie、重定向、设置响应状态码。|
|**`HttpSession`**|**服务端内存中**（也可存入 Redis）|**较长**（从创建到超时销毁，或手动 invalidate）|传统 Web 项目中保存登录用户信息、购物车临时数据等。|
