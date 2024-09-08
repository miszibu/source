---
layout: Nginx
title: Nginx使用
date: 2019-03-19 19:09:25
tags: [Nginx] 

---

本文记录Nginx相关配置信息和其实现相关技巧。

Nginx作为web服务器来实现反向代理和负载均衡已经是为人所熟悉的操作了，本文会介绍Nginx的配置文件和基本命令，然后引入Nginx实现负载均衡的方式，加深对Nginx的理解。

<!-- more -->

# Nginx Installation

```shell
#从下面的链接中下载合适的Nginx版本
#http://nginx.org/en/download.html

# Unzip nginx installation package
tar -zxvf nginx-1.16.0.tar.gz 

# install env and tools for building
yum -y install gcc zlib-devel zlib pcre pcre-devel
# installtion for nginx ssl module
yum -y install openssl openssl-devel

# 解压后进入 nginx-1.13.5 目录进行编译安装 
./configure --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module  --add-module=/data/nginx/echo-nginx-module-0.61
make && make install

# 将Nginx加入到系统PATH
export PATH=/usr/local/nginx/sbin:$PATH

# Check Nginx installtion status
nginx -V
```

# Nginx的配置

## 配置详解

```c
#Nginx的worker进程运行用户以及用户组
#user  nobody;

#Nginx开启的进程数
worker_processes  1;

#定义全局错误日志定义类型，[debug|info|notice|warn|crit]
#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#指定进程ID存储文件位置
#pid        logs/nginx.pid;

#事件配置
events {
    #use [ kqueue | rtsig | epoll | /dev/poll | select | poll ];
    #epoll模型是Linux内核中的高性能网络I/O模型，如果在mac上面，就用kqueue模型。
    use kqueue;
    
    #每个进程可以处理的最大连接数，理论上每台nginx服务器的最大连接数为worker_processes*worker_connections。理论值：worker_rlimit_nofile/worker_processes
    worker_connections  1024;
}

#http参数
http {
    #文件扩展名与文件类型映射表
    include       mime.types;
    #默认文件类型
    default_type  application/octet-stream;
    
    #日志相关定义
    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';
    
    #连接日志的路径，指定的日志格式放在最后。
    #access_log  logs/access.log  main;

    #开启高效传输模式
    sendfile        on;
    
    #防止网络阻塞
    #tcp_nopush     on;

    #客户端连接超时时间，单位是秒
    #keepalive_timeout  0;
    keepalive_timeout  65;

    #开启gzip压缩输出
    #gzip  on;
    
	# 1、轮询（默认）
	# 每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器down掉，能自动剔除。 
	upstream polling_strategy { 
    	server glmapper.net:8080; # 应用服务器1
    	server glmapper.net:8081; # 应用服务器2
	} 
    #2、指定权重
	#指定轮询几率，weight和访问比率成正比，用于后端服务器性能不均的情况。 
	upstream  weight_strategy { 
	    server glmapper.net:8080 weight=1; # 应用服务器1
    	server glmapper.net:8081 weight=9; # 应用服务器2
	}
	#3、IP绑定 ip_hash
	#每个请求按访问ip的hash结果分配，这样每个访客固定访问一个后端服务器，
	#可以解决session的问题;在不考虑引入分布式session的情况下，
	#原生HttpSession只对当前servlet容器的上下文环境有效
	upstream ip_hash_strategy { 
    	ip_hash; 
	    server glmapper.net:8080; # 应用服务器1
    	server glmapper.net:8081; # 应用服务器2
	} 
	#4、fair（第三方）
	#按后端服务器的响应时间来分配请求，响应时间短的优先分配。 
	upstream fair_strategy { 
    	server glmapper.net:8080; # 应用服务器1
	    server glmapper.net:8081; # 应用服务器2
    	fair; 
	} 
	#5、url_hash（第三方）
	#按访问url的hash结果来分配请求，使每个url定向到同一个后端服务器，
	#后端服务器为缓存时比较有效。 
	upstream url_hash_strategy { 
	    server glmapper.net:8080; # 应用服务器1
    	server glmapper.net:8081; # 应用服务器2 
	    hash $request_uri; 
    	hash_method crc32; 
	} 

   
    #虚拟主机基本设置
    server {
		#监听的端口号
        listen       80;
        #访问域名
        server_name  localhost;
        #编码格式，如果网页格式与当前配置的不同的话将会被自动转码
        #charset koi8-r;
        #虚拟主机访问日志定义
        #access_log  logs/host.access.log  main;
        #对URL进行匹配 重定向80端口到4000端口
        location / {
	       proxy_pass http://localhost:4000;
		}
    }
    # 监听81端口 实现负载均衡 
    server{
        listen 81;
        server_name weightDispather;
        location / {
            proxy_pass http://weight_strategy
        }
    }
}
```

## Forwared方法块的坑

对于 proxy_pass 的 forwarded URL，当 location 不是 / ，proxy_pass 后的URI不同的写法会有不同的结果。

### 方法块中没有变量

```nginx
location /foo/ {
    proxy_pass http://127.0.0.1:8080;
}
# Request  /foo/bar/baz
# Forward  http://127.0.0.1:8080/foo/bar/baz

location /foo/ {
    //Note the trailing slash       ↓
    proxy_pass http://127.0.0.1:8080/;
}
# Request  /foo/bar/baz
# Forward  http://127.0.0.1:8080/bar/baz
```

### 方法块中设置变量

```nginx
set $upstream_endpoint http://just-for-test.com/;
location /foo/ {
    proxy_pass $upstream_endpoint;
}
# Request  /foo/bar/baz
# Forward  http://just-for-test.com/

set $upstream_endpoint http://just-for-test.com;
location /foo/ {
    rewrite ^/foo/(.*) /$1 break;
    proxy_pass $upstream_endpoint;
}
# Request  /foo/bar/baz
# Forward  http://just-for-test.com/bar/baz
```

# 常用命令

```bash
# nginx 配置文件
/etc/nginx/nginx.conf
# nginx 默认配置文件 占用80端口需注意
/etc/nginx/conf.d/default.conf

# 启动nginx
systemctl start nginx.service
# 重启nginx
nginx -s reload
# Stop Nginx
nginx -s stop
# 检查nginx配置
nginx -t -c /tmp/nginx-1.16.0/conf/nginx.conf 
```

# 如何实现高并发

**进程模型**：采用Master/Wroker模式

1. master进程主要负责收集、分发请求。当一个请求过来时，master拉起一个worker进程负责处理这个请求。
2. master进程也要负责监控woker的状态，保证高可靠性
3. woker进程一般设置为跟cpu核心数一致。nginx的woker进程跟apache不一样。apche的进程在同一时间只能处理一个请求，所以它会开很多个进程，几百甚至几千个。而nginx的woker进程在同一时间可以处理请求数只受内存限制，因此可以处理多个请求。

**事件模型**：nginx是异步非阻塞的。

​	每进来一个request，会有一个worker进程去处理。每个worker进程将数据处理到可能发生阻塞的地方，比如向后端转发request,此时并等待请求返回。此时Worker进程，会注册一个请求回返事件，等待事件响应，再去处理该请求。

​	web server的工作性质（io密集型）决定了每个request的大部份生命都是在网络传输中，实际上花费在server机器上的时间片不多。这是几个进程就解决高并发的秘密所在。

# Nginx一致性哈希

​	**普通哈希**

​	nginx的负载均衡时有一个hash $request_uri的选项，这个是类似于LVS的dh。是针对客户端访问的uri来做的绑定。这样客户端访问同一个uri的时候，会被分配到同一个服务器上去。这样提高了缓存的命中率。

​     过程：每个**uri**进行hash计算得到一个数值，这个数值除以整个节点数量取余数。**（取模算法）**

​     缺点：如果一个节点挂了，那么整个全局都会乱掉。因为整个的节点数变了，因为除数变了。

 	**一致性哈希**

​	一致性hash的采用的是除数特别大，假设有一个**hash环**。是个**闭环**。把32位二进制的整数转换为十进制后均匀分布在整个环上。hash结果是除以2^32-1. 那么结果一定是落在环上的。那么，这个点靠近谁，就缓存在谁那里。假设a节点坏了。那么下一次的计算结果就是旁边的邻居。但是邻居的缓存不会受到影响。只是坏掉的A节点会重新去缓存。

# 负载均衡策略

## 轮询(默认)

```bash
# 每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器down掉，能自动剔除。 
upstream polling_strategy { 
    server glmapper.net:8080; # 应用服务器1
    server glmapper.net:8081; # 应用服务器2
} 
```

## 权重

```bash
#指定轮询几率，weight和访问比率成正比，用于后端服务器性能不均的情况。 
upstream  weight_strategy { 
    server glmapper.net:8080 weight=1; # 应用服务器1
    server glmapper.net:8081 weight=9; # 应用服务器2
}
```

## IPHash

```bash
#每个请求按访问ip的hash结果分配，这样每个访客固定访问一个后端服务器，
#可以解决session的问题;在不考虑引入分布式session的情况下，
#原生HttpSession只对当前servlet容器的上下文环境有效
upstream ip_hash_strategy { 
    ip_hash; 
    server glmapper.net:8080; # 应用服务器1
    server glmapper.net:8081; # 应用服务器2
} 
```

> IPHash是根据前三位的值进行Hash，为了将处于同一局域网的请求，落到同一台服务器上。

## 其他负载均衡 需要安装插件

```bash
#4、fair（第三方）
#按后端服务器的响应时间来分配请求，响应时间短的优先分配。 
upstream fair_strategy { 
    server glmapper.net:8080; # 应用服务器1
    server glmapper.net:8081; # 应用服务器2
    fair; 
} 
#5、url_hash（第三方）
#按访问url的hash结果来分配请求，使每个url定向到同一个后端服务器，
#后端服务器为缓存时比较有效。 
upstream url_hash_strategy { 
    server glmapper.net:8080; # 应用服务器1
    server glmapper.net:8081; # 应用服务器2 
    hash $request_uri; 
    hash_method crc32; 
} 
```

# Reference

[Nginx 多进程模型是如何实现高并发的？](https://www.zhihu.com/question/22062795)

[epoll 或者 kqueue 的原理是什么](https://www.zhihu.com/question/20122137/answer/14049112)

