# 原理介绍

![[Pasted image 20260729091535.png]]

## 梳理补充

发送验证码流程图：健壮性考虑，提交手机号后要验证是否合法
登录注册图：用户不存在时实现注册功能
校验登录图：用于用户访问服务器需要身份验证的资源时，将数据库中的用户信息缓存到treadlocal中，方便使用


# 发送验证码
controller![[Pasted image 20260729092954.png]]

service
![[Pasted image 20260729093001.png]]

这里并没有实际发送验证码给用户，而是只在服务器端终端上打印，因为如果要发送就要用到第三方平台，比较麻烦，重点不在这里

# 登录和注册

controller ![[Pasted image 20260729094641.png]]

service
不贴了
参考[[短信登录（使用redis）]]

## 补充
需不需要返回登录凭证？

**不需要返回登录凭证（如 Token）。**

在基于 **Session** 实现的登录模块中，客户端与服务端的身份识别依赖于 HTTP 协议和浏览器的 Cookie 机制，整体流程如下：

基于 Session 的登录机制原理

1. **客户端发起登录**：用户提交手机号和短信验证码。
    
2. **服务端校验与保存**：
    
    - 服务端校验验证码无误后，从数据库查询或创建用户。
        
    - 服务端将用户信息存入 **`HttpSession`** 对象中（例如 `session.setAttribute("user", userDto)`）。
        
3. **自动生成与设置 Cookie**：
    
    - Tomcat / Spring Boot 会自动为该 Session 分配一个唯一的标识符 —— **`JSESSIONID`**。
        
    - 服务端在响应头中加入 `Set-Cookie: JSESSIONID=xxxxxx; Path=/; HttpOnly`。
        
4. **响应返回**：
    
    - **接口返回值只需要返回成功标识（如 `Result.ok()`）或用户基本信息**，**不需要**在 Response Body 中手动返回任何登录凭证。
        
5. **后续请求携带**：
    
    - 浏览器收到响应后，会自动把 `JSESSIONID` 保存到 Cookie 中。
        
    - 后续前端发起任何请求时，浏览器都会在请求头 `Cookie` 中**自动带上** `JSESSIONID`，服务端拦截器通过 `request.getSession()` 即可获取对应的用户信息。



# 登录验证功能

通过拦截器来进行验证，因为很多接口都会用到验证功能，所以抽出来用拦截器实现
![[Pasted image 20260729103632.png]]



userholder
![[Pasted image 20260729103816.png]]
封装了几个操作threadlocal的方法


controller层，拦截器已经验证用户存在，这里直接从threadlocal中取出用户信息并返回即可，不用service层代码了
![[Pasted image 20260729103700.png]]


设置拦截器，需要在config中注册拦截器，配置拦截器的功能，拦截的路径等等
![[Pasted image 20260729103942.png]]


# 隐藏敏感信息

在之前的代码中，用户注册之后，所有信息直接存在session中，登录验证时也直接将敏感信息传给前端，这是不安全的
所以修改存储代码，用户注册后，只将必要信息存入session即可，这样之后前端获取时也只能获取到安全信息
