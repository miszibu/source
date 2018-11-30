---
title: docker-基础入门
date: 2018-07-5 11:50:01
tags: docker
---

> Build , Ship and Run Any App , Anywhere

本文主要讲解docker的特点和安装使用，并介绍镜像，容器，仓库，端口容器互联等功能。

文章内容会同步更新在docker脑图中

[zibuのDocker mind picture](https://www.processon.com/view/link/5b3db86de4b0ade3e26d6e7f)

<!-- more -->



------



#### 初识容器和Docker

Docker的构想是通过对应用的**封装**（packaging），**分发**(distribution)，**部署**(deploying)，**运行**(runtime)生	命周期进行管理，达到应用组件，一次封装，到处运行的目的。而应用的概念，不仅局限于一个Web应用，一个编译环境，甚至是一个OS或服务器集群。

相对于传统的虚拟机技术。Docker具备更多的优点。如

|   特性   |           容器           |   虚拟机   |
| :------: | :----------------------: | :--------: |
| 启动速度 |           秒级           |   分钟级   |
|   性能   |      接近原生的性能      |    较弱    |
| 内存代价 |           很小           |    较多    |
| 硬盘使用 |         一般为MB         |  一般为GB  |
| 运行密度 |     单机支持上千容器     | 视性能而定 |
|  隔离性  |         安全隔离         |  完全隔离  |
|  迁移性  | 优秀，打包镜像，轻松迁移 |    一般    |

**同其他虚拟化方式最大的不同**：没有虚拟机管理程序和VMOS。而是直接通过Docker容器支持



------



#### 核心概念

* **镜像 image**：镜像类似于虚拟机镜像，docker容器运行镜像时，会在镜像的最上层创建一个可写层，镜像本身是**只读**的。
* **容器 container** : 类似于一个轻量级的沙箱，Docker利用容器来**运行和隔离应用**。容器是从镜像创建的应用运行实例。可以将其启动，开始，停止，删除，而这些**容器彼此之间**都是**隔离**的，**互不可见**的。
* **仓库 depository**：Docker在很多地方有借鉴Git，例如仓库就类似于Git的仓库，是Docker集中存放镜像文件的地方。一个**仓库注册服务器，有多个仓库**，每一个**仓库存放一类的镜像**，**通过Tag区分**。Docker Hub是docker的官方镜像。



***



#### Docker镜像使用 || Docker 容器管理 

 相关命令以列表模式不方便显示，详情查看processon脑图。



***



#### Docker数据管理

* 数据卷 Data Volumes：容器内数据直接映射到本地主机环境
* 数据卷迁移 Data Volume Containers：使用特定容器维护数据卷



***



#### 端口映射与容器互联

##### 1.映射容器内应用的服务端口到本地宿主主机

​	在启动容器时，如果不指定对应的参数，在容器外部是无法通过网络来访问容器内的网络应用和服务。	

​	**-P** : Docker会随机映射一个49000~49900的端口到内部容器开放的网络端口

​	**-p** : -p 5000:5000 将本地的5000端口映射到虚拟机的5000端口上

​		多次使用可以映射多个端口 如：-p 5000:5000 -p 80：80

​		-p 127.0.0.1:5000:5000/udp 映射端口到指定地址 并指定UDP端口

​		-p 127.0.0.1::5000 映射到指定地址的任意端口

​		docker port containerName 5000： 查看某个容器 某个端口的映射

##### 2.互联机制实现多个容器间通过容器名快速访问

​	使用**--name**自定义容器命名:docker run -d -P **--name** newName imageName

​	docker run -d -P --link db:db newName imageName 

​	**--link name:alias**  : name 要连接的容器名称 db 连接的名称

​	

***



#### 使用Dockerfile创建镜像

Dockerfile是一个文本格式的配置文件，用户可以使用dockerfile来快速创建自定义镜像。

```docker
Dockerfile可以使用#来打注释
FROM debian:jessie #使用debian最好，可以更为精简，或者scratch，这是一个空的image
MAINTAINER zibu #镜像维护人
ADD muc / #将muc添加到镜像的根目录下
ENV NGINX_VERSION 1.10 #指定环境变量，可以被后续的Run指令使用。
RUN**	#运行指定命令 当命令过长使用\划分
EXPOSE 80 443 #声明镜像内服务所监听的端口，只起到声明作用
CMD ['/muc'] # 只有最后一条CMD命令生效，如果手动指定命令，docker
```

**docker build** 会默认读取指定路径下（包括子目录）的dockerfile。

```docker
docker build [选项] 内容路径
ocker build -t name:tag  . #直接build当前目录下的Dockerfile
docker build -t name:tag  /tmp/docker_builder  #-t 给Dockerfile创建的image打标签
docker build -t name:tag -f /usr/zibu/dockerfiel #使用-f标签指定路径
```



***

#### 相关资料

《Docker技术入门与实战 第二版》