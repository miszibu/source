---
title: docker-基本概念
date: 2018-07-5 11:50:01
tags: [Docker]
---

> Build , Ship and Run Any App , Anywhere
>
> 容器的发展历史，从VM的hypervisor模拟硬件 到 Linux上cgroup, namespace等概念的出现。最终发展到现在容器，而最有名的就是Docker了。
>
> 本文主要讲解docker的特点和安装使用，并介绍镜像，容器，仓库，端口容器互联等功能。
>
> 文章内容会同步更新在docker脑图中
>
> [zibuのDocker mind picture](https://www.processon.com/view/link/5b3db86de4b0ade3e26d6e7f)

<!-- more -->

------



## 初识容器和Docker

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



## 核心概念

* **镜像 image**：镜像类似于虚拟机镜像，docker容器运行镜像时，会在镜像的最上层创建一个可写层，镜像本身是**只读**的。
* **容器 container** : 类似于一个轻量级的沙箱，Docker利用容器来**运行和隔离应用**。容器是从镜像创建的应用运行实例。可以将其启动，开始，停止，删除，而这些**容器彼此之间**都是**隔离**的，**互不可见**的。
* **仓库 depository**：Docker在很多地方有借鉴Git，例如仓库就类似于Git的仓库，是Docker集中存放镜像文件的地方。一个**仓库注册服务器，有多个仓库**，每一个**仓库存放一类的镜像**，**通过Tag区分**。Docker Hub是docker的官方镜像。



## 镜像Imgae:

操作系统分为**内核**和**用户空间**。对于 Linux 而言，内核启动后，会挂载 `root` 文件系统为其提供用户空间支持。而 Docker 镜像（Image），就相当于是一个 `root` 文件系统。比如官方镜像 `ubuntu:18.04` 就包含了完整的一套 Ubuntu 18.04 最小系统的 `root` 文件系统。

**Docker 镜像是一个特殊的文件系统**，除了提供容器运行时所需的程序、库、资源、配置等文件外，还包含了一些为运行时准备的一些配置参数（如匿名卷、环境变量、用户等）。镜像不包含任何动态数据，其内容在构建之后也不会被改变。

### 分层存储

因为镜像包含操作系统完整的 `root` 文件系统，其体积往往是庞大的，因此在 Docker 设计时，就充分利用 [Union FS](https://en.wikipedia.org/wiki/Union_mount) 的技术，将其设计为分层存储的架构。所以严格来说，镜像并非是像一个 ISO 那样的打包文件，镜像只是一个虚拟的概念，其实际体现并非由一个文件组成，而是由一组文件系统组成，或者说，由多层文件系统联合组成。

镜像构建时，会一层层构建，前一层是后一层的基础。**每一层构建完就不会再发生改变，后一层上的任何改变只发生在自己这一层**。比如，删除前一层文件的操作，实际不是真的删除前一层的文件，而是仅在当前层标记为该文件已删除。在最终容器运行的时候，虽然不会看到这个文件，但是实际上该文件会一直跟随镜像。因此，在构建镜像的时候，需要额外小心，每一层尽量只包含该层需要添加的东西，任何额外的东西应该在该层构建结束前清理掉。

分层存储的特征还使得镜像的复用、定制变的更为容易。甚至可以用之前构建好的镜像作为基础层，然后进一步添加新的层，以定制自己所需的内容，构建新的镜像。



## 容器Container

镜像（`Image`）和容器（`Container`）的关系，就像是面向对象程序设计中的 `类` 和 `实例` 一样，镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。

容器的实质是进程，但与直接在宿主执行的进程不同，容器进程运行于属于自己的独立的 [命名空间](https://en.wikipedia.org/wiki/Linux_namespaces)。因此容器可以拥有自己的 `root` 文件系统、自己的网络配置、自己的进程空间，甚至自己的用户 ID 空间。容器内的进程是运行在一个隔离的环境里，使用起来，就好像是在一个独立于宿主的系统下操作一样。这种特性使得容器封装的应用比直接在宿主运行更加安全。也因为这种隔离的特性，很多人初学 Docker 时常常会混淆容器和虚拟机。

前面讲过镜像使用的是分层存储，容器也是如此。每一个容器运行时，是以镜像为基础层，在其上创建一个当前容器的存储层，我们可以称这个为容器运行时读写而准备的存储层为**容器存储层**。

容器存储层的生存周期和容器一样，容器消亡时，容器存储层也随之消亡。因此，**任何保存于容器存储层的信息都会随容器删除而丢失。**

按照 Docker 最佳实践的要求，**容器不应该向其存储层内写入任何数据，容器存储层要保持无状态化**。所有的文件写入操作，都应该使用 [数据卷（Volume）](https://yeasy.gitbooks.io/docker_practice/data_management/volume.html)、或者绑定宿主目录，在这些位置的读写会跳过容器存储层，直接对宿主（或网络存储）发生读写，其性能和稳定性更高。

数据卷的生存周期独立于容器，容器消亡，数据卷不会消亡。因此，使用数据卷后，容器删除或者重新运行之后，数据却不会丢失。

***



## Docker镜像使用 || Docker 容器管理 

 相关命令以列表模式不方便显示，详情查看processon脑图。



***

## Docker数据管理

* 数据卷 Data Volumes：容器内数据直接映射到本地主机环境
* 数据卷迁移 Data Volume Containers：使用特定容器维护数据卷



***



## 端口映射与容器互联

### 1.映射容器内应用的服务端口到本地宿主主机

​	在启动容器时，如果不指定对应的参数，在容器外部是无法通过网络来访问容器内的网络应用和服务。	

​	**-P** : Docker会随机映射一个49000~49900的端口到内部容器开放的网络端口

​	**-p** : -p 5000:5000 将本地的5000端口映射到虚拟机的5000端口上

​		多次使用可以映射多个端口 如：-p 5000:5000 -p 80：80

​		-p 127.0.0.1:5000:5000/udp 映射端口到指定地址 并指定UDP端口

​		-p 127.0.0.1::5000 映射到指定地址的任意端口

​		docker port containerName 5000： 查看某个容器 某个端口的映射

### 2.互联机制实现多个容器间通过容器名快速访问

​	使用**--name**自定义容器命名:docker run -d -P **--name** newName imageName

​	docker run -d -P --link db:db newName imageName 

​	**--link name:alias**  : name 要连接的容器名称 db 连接的名称

​	

***



## 使用Dockerfile创建镜像

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
docker build -t name:tag  . #直接build当前目录下的Dockerfile
docker build -t name:tag  /tmp/docker_builder  #-t 给Dockerfile创建的image打标签
docker build -t name:tag -f /usr/zibu/dockerfiel #使用-f标签指定路径
```



***

## 相关资料

《Docker技术入门与实战 第二版》