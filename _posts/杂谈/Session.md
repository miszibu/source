---
title: Session
date: 2018-04-12 12:18:23
tags: [Session]
---

> **Session**是在服务端保存的一个**数据结构**，用来跟踪用户的状态，这个数据可以保存在**集群、数据库、文件**中；
> **Cookie**是客户端保存用户信息的一种机制，用来记录用户的一些信息，也是实现Session的一种方式。
>
> **Cookie**是一个**实际存在**的东西，即HTTP协议中定义在header中的字段。而**Session**是一个**抽象概念**，即会话。用于存储用户信息，通常借助Cookie中存放SessionID来匹配服务器存储实现Session，是一种更高级的会话状态实现。

<!--more-->

***

### Session的实现

* **CookieID 实现**
* **URL重写**：当Cookie被禁用时，将SeesionID写入连接地址

1.  在实际项目中，一般使用**Redis作为内存数据库**(即分布式Session实现第三种缓存集中式管理)，以K/V的Map形式存储下来。Key就是自己生成的N位Random String，Value存储用户账号登陆信息。
   SessionId 存放在 Cookie中，在B/S模型中，浏览器会在**同一层次域名**下(Cookie存在域下，默认域就是http的路由地址，设置Cookie时要设置到IP地址/根目录下，否则子目录读取不到)，**读取**本地所有**Cookie**放在HTTP请求中，服务器就可以读取到Request中的Cookie，取出SessionId，在Redis中读取数据，判断登录状态获取登录信息。
2. **JWT**：Java Web Token 是一个Java下的Session包，它通过数字压缩使得用户数据Json能被压缩到Cookie中去，读取Cookie的JWTString然后解密即可获得相关Session过期时间和用户数据。

***

### 分布式Session实现

1. **Session复制**：所有的服务器保存相同的Session，这种方法需要广播复制Session,需要一定网络开销。
   * 优点:实现简单，且单台服务器挂了不影响用户访问。
   * 缺点:**不适用大量访问**的场景。
2. **粘性Session**:新用户访问时分配某台机器后，强制制定后续所有请求均落到此机器上。
   * 优点：没有网络开销，实现简单
   * 缺点：一台服务器挂了，就会导致用户Session丢失，造成单点故障。
3. **缓存集中式管理**：在分布式服务器集群中设置Session服务器(常用Redis)，当用户访问不同节点时先从缓存服务器拿Session。
   *  优点：可靠性好，一个用户请求在映射错误，映射到其他服务器时，也能获取Session。
   *  缺点：实现复杂、稳定性依赖于缓存的稳定性、Session信息放入缓存时要有合理的策略写入



## 相关资料

[COOKIE和SESSION有什么区别？](https://www.zhihu.com/question/19786827)

[[分布式Session的几种实现方式]](https://www.cnblogs.com/cxrz/p/8529587.html)