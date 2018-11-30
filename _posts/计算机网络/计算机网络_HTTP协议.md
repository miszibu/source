---
title: 回计算机网络-HTTP协议
date: 2018-04-11 09:44:08
tags: [HTTP，计算机网络] 
---

## HTTP简介

Hyper Text Transfer Protocol（超文本传输协议），基于TCP/IP通信协议传递数据。位于OSI模型的应用层。

基于BS模型，浏览器作为HTTP客户端通过URL向HTTP服务端即WEB服务器发送请求并相应。

<!--more-->

## 主要特点

1. 简单快速:通过**Get Post Update Delete Put Head Options Trace**等方法，传送请求方法和路径。
2. 灵活：允许传输任意类型的数据对象。以Content-Type加以标记。
3. 无连接：每次连接只处理一个请求，不会长时间维护一个电路连接。每次处理完请求，并受到应答，断开连接，采用这种方式节省传输时间。
4. 无状态：对于事务处理没有存档记忆。无状态在临时应答的时候速度较快，但后续处理需要重传就较为消耗资源。各有利弊
5. 支持BS和CS模式。

## URL与URI

Uniform Resource Identifier:统一资源标识符。

Uniform Resource Locator:统一资源定位器。

## HTTP之请求消息Request

##### 请求行（request line）、请求头部（header）、空行和请求数据四个部分组成

![](../img/HTTP请求.png)

POST请求数据例子：

```http
POST / HTTP1.1                 //请求行
Host:www.wrox.com			   //请求头
User-Agent:Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1; .NET CLR 2.0.50727; .NET CLR 3.0.04506.648; .NET CLR 3.5.21022)
Content-Type:application/x-www-form-urlencoded
Content-Length:40
Connection: Keep-Alive
                                //空行
name=Professional%20Ajax&publisher=Wiley//请求数据
```

## HTTP之响应消息Response

##### 状态行、消息报头、空行和响应正文等四个部分组成。

~~~http
HTTP/1.1 200 OK
Date: Fri, 22 May 2009 06:07:21 GMT
Content-Type: text/html; charset=UTF-8

<html>
      <head></head>
      <body>
            <!--body goes here-->
      </body>
</html>
~~~

## 状态码

| 分类   | 分类描述                    |
| ---- | ----------------------- |
| 1**  | 信息，服务器收到请求，需要请求者继续执行操作  |
| 2**  | 成功，操作被成功接收并处理           |
| 3**  | 重定向，需要进一步的操作以完成请求       |
| 4**  | 客户端错误，请求包含语法错误或无法完成请求   |
| 5**  | 服务器错误，服务器在处理请求的过程中发生了错误 |

```
200 OK                        //客户端请求成功
400 Bad Request               //客户端请求有语法错误，不能被服务器所理解
401 Unauthorized              //请求未经授权，这个状态代码必须和WWW-Authenticate报头域一起使用 
403 Forbidden                 //服务器收到请求，但是拒绝提供服务
404 Not Found                 //请求资源不存在，eg：输入了错误的URL
500 Internal Server Error     //服务器发生不可预期的错误
503 Server Unavailable        //服务器当前不能处理客户端的请求，一段时间后可能恢复正常
```

## 其他

**Content-Encoding**：HTTP协议传输数据可压缩，使用Content-Encoding来设定压缩格式

## HTTP1.1不同处

**持久连接**：通过引入持久连接，一次TCP连接可以被多个请求复用。客户端在最后一次请求时，发送Connection:close来明确服务器关闭TCP连接。

**管道机制**:客户端可以在一个TCP连接中，同时请求多个资源，服务器桉请求顺序回复。而不是想以前一样等一个请求结束后再发送另一个请求。

**Content-Length**:1.0版本，一次请求就是一个TCP连接。在1.1版本中，加入Content-Length，来确保本次回应的长度。

**分块传输编码**：当服务器端需要长时间操作时，通过分块将数据分段传输。这样的流模式相对于传统的缓存模式效率更高。

## HTTP2.0

**二进制协议**：2.0的头信息和数据体都是二进制，并且被统称为帧(frame),二进制协议，是的HTTP2.0可以定义额外的帧。这为未来的高级应用打下了基础。

**多工**：HTTP/2 复用TCP连接，在一个连接里，客户端和浏览器都可以同时发送多个请求或回应，而且不用按照顺序一一对应，这样就避免了"队头堵塞"。

举例来说，在一个TCP连接里面，服务器同时收到了A请求和B请求，于是先回应A请求，结果发现处理过程非常耗时，于是就发送A请求已经处理好的部分， 接着回应B请求，完成后，再发送A请求剩下的部分。

这样双向的、实时的通信，就叫做多工（Multiplexing）。

**数据流**：更为高级的数据流，同一个包中的数据可能来自于不同的请求。通过数据流ID来区分数据流属于不同的请求。同时规定，客户端发出的数据流，ID一律为奇数，服务器发出的，ID为偶数。

数据流发送到一半的时候，客户端和服务器都可以发送信号（`RST_STREAM`帧），取消这个数据流。1.1版取消数据流的唯一方法，就是关闭TCP连接。这就是说，HTTP/2 可以取消某一次请求，同时保证TCP连接还打开着，可以被其他请求使用。

客户端还可以指定数据流的优先级。优先级越高，服务器就会越早回应。

数据流发送到一半的时候，客户端和服务器都可以发送信号（`RST_STREAM`帧），取消这个数据流。1.1版取消数据流的唯一方法，就是关闭TCP连接。这就是说，HTTP/2 可以取消某一次请求，同时保证TCP连接还打开着，可以被其他请求使用。

客户端还可以指定数据流的优先级。优先级越高，服务器就会越早回应。

**头信息压缩**：请求信息中，有很多字段重复，比如Cookie和UserAgent。这些重复的数据一定程度上影响速度，浪费带宽。通过引入头信息压缩机制，一方面通过`gzip`或`compress`压缩了头信息,另一方面，客户端和服务器同时维护一张头信息表，所有字段都会存入这个表，生成一个索引号，以后就不发送同样字段了，只发送索引号，这样就提高速度了。

**服务器推送**：允许服务及未经允许，主动向客户端发送资源。这叫做服务器推送。

## HTTPS

HTTPS是HTTP协议的加密版本，HTTP/2协议只有在HTTPS的版本才能使用。

**证书**：花钱买一张二进制文件证书，包含网站公钥和元数据。共有三种认证

- **域名认证**（Domain Validation）：最低级别认证，可以确认申请人拥有这个域名。对于这种证书，浏览器会在地址栏显示一把锁。
- **公司认证**（Company Validation）：确认域名所有人是哪一家公司，证书里面会包含公司信息。
- **扩展认证**（Extended Validation）：最高级别的认证，浏览器地址栏会显示公司名。

还分为三种覆盖范围。

> - **单域名证书**：只能用于单一域名，`foo.com`的证书不能用于`www.foo.com`
> - **通配符证书**：可以用于某个域名及其所有一级子域名，比如`*.foo.com`的证书可以用于`foo.com`，也可以用于`www.foo.com`
> - **多域名证书**：可以用于多个域名，比如`foo.com`和`bar.com`

**修改连接**

网页使用HTTPS协议后，所有的资源都需要改成HTTPS，否则不会被加载。

**301重共享**

修改WEB服务器配置，使用301重定向，将HTTP协议访问导向HTTPS协议

Nginx

> ```
> server {
>   listen 80;
>   server_name domain.com www.domain.com;
>   return 301 https://domain.com$request_uri;
> }
>
> ```

Apache

> ```
> RewriteEngine On
> RewriteCond %{HTTPS} off
> RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]
> ```

