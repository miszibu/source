---
title: Dockerfile基础命令和知识点
date: 2018-12-12 22:54:12
tags: Docker
---

> 在Docker_基础概念一文中，我们已经了解image,container这些容器的基本概念。
>
> 我们日常所需要的基本image可以由官方仓库找到，比如Linux的image。但是对于我们具体的应用而言，需要自己编写Dockerfile来启动。 
>
> 本文主要讲述Dockerfile的一些基础储备和踩坑点。

<!--more-->

------

## 使用Dockerfile定制镜像

使用Docker Commit可以定制镜像，但是存在着镜像不可重复，构建不透明等诸多问题。

Dockerfile是一个文本文件，其内包含了一条条的**指令(Instruction),每一条指令构建一层**，**因此每一条指令的内容，就是描述该层应当如何构建。**

### 简单示例

```shell
# 一般一个image 单独放个文件夹 
# 在该文件夹下 创建Dockerfile 注意UTF8编码格式 否则可能报错
mkdir mynginx 
cd mynginx
touch Dockerfile

vim Dockerfile
FROM nginx
RUN  echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
:wq

# -t 指定tag 名称为mynginx tag为1.0
docker build -t mynginx:v1.0 .

# 运行 容器 映射80端口到 容器内的80端口
docker run -p 80:80 -t mynginx:v1.0
```

## FROM 指定基础镜像

定制镜像可以以一个镜像为基础，在其上进行定制。例如简单示例中，运行了nginx的镜像容器，在对其进行修改。基础镜像是必备的，且一个Dockerfile文件中，FROM是必备的第一条指令。

在 [Docker Store](https://store.docker.com/) 上有非常多的高质量的官方镜像，有可以直接拿来使用的服务类的镜像，如 [`nginx`](https://store.docker.com/images/nginx/)、[`redis`](https://store.docker.com/images/redis/)、[`mongo`](https://store.docker.com/images/mongo/)、[`mysql`](https://store.docker.com/images/mysql/)、[`httpd`](https://store.docker.com/images/httpd/)、[`php`](https://store.docker.com/images/php/)、[`tomcat`](https://store.docker.com/images/tomcat/) 等；也有一些方便开发、构建、运行各种语言应用的镜像，如 [`node`](https://store.docker.com/images/node)、[`openjdk`](https://store.docker.com/images/openjdk/)、[`python`](https://store.docker.com/images/python/)、[`ruby`](https://store.docker.com/images/ruby/)、[`golang`](https://store.docker.com/images/golang/) 等。可以在其中寻找一个最符合我们最终目标的镜像为基础镜像进行定制。

如果没有找到对应服务的镜像，官方镜像中还提供了一些更为基础的操作系统镜像，如 [`ubuntu`](https://store.docker.com/images/ubuntu/)、[`debian`](https://store.docker.com/images/debian/)、[`centos`](https://store.docker.com/images/centos/)、[`fedora`](https://store.docker.com/images/fedora/)、[`alpine`](https://store.docker.com/images/alpine/) 等，这些操作系统的软件库为我们提供了更广阔的扩展空间。

除了选择现有镜像为基础镜像外，Docker 还存在一个特殊的镜像，名为 `scratch`。这个镜像是虚拟的概念，并不实际存在，它表示一个空白的镜像。

```dockerfile
FROM scratch
```

如果你以 `scratch` 为基础镜像的话，意味着你不以任何镜像为基础，接下来所写的指令将作为镜像第一层开始存在。

不以任何系统为基础，直接将可执行文件复制进镜像的做法并不罕见，比如 [`swarm`](https://hub.docker.com/_/swarm/)、[`coreos/etcd`](https://quay.io/repository/coreos/etcd)。对于 Linux 下静态编译的程序来说，并不需要有操作系统提供运行时支持，所需的一切库都已经在可执行文件里了，因此直接 `FROM scratch` 会让镜像体积更加小巧。使用 [Go 语言](https://golang.org/) 开发的应用很多会使用这种方式来制作镜像，这也是为什么有人认为 Go 是特别适合容器微服务架构的语言的原因之一。

------



## RUN 执行命令

使用RUN来执行命令行命令，因此RUN指令是在定制镜像时，最为常用的指令之一。其格式有两种

* ***shell*格式**：加一个RUN指令的前缀，就像直接在命令行中输入命令一样。

  ```shell
  RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
  ```

* ***exec*格式**：`RUN ["可执行文件", "参数1", "参数2"]`，这更像是函数调用中的格式。

### RUN命令示例

**错误示例**：

```shell
RUN tar -xzvf /usr/local/*.tar.gz
RUN rm -rf /usr/local/*.tar.gz
RUN chmod +x /usr/local/jdk1.8.0_191
```

在上面的例子中，一共执行了3次RUN操作，上文已经提过Dockerfile中的每一个指令都会建立一层镜像，每条指令运行完，该层就会被`commit`，就跟手工建立镜像的过程一样。

因此，示例中一共`commit`了三次，第二个`rm`命令是没有效果的。之间移入的`add` || `copy`操作，已经将文件引入且提交，后续的操作，不会修改之前的commit,而是在新一层标记删除而已。无意义的操作导致了最终的image文件过于臃肿。

> *Union FS 是有最大层数限制的，比如 AUFS，曾经是最大不得超过 42 层，现在是不得超过 127 层。*

**正确示例**：

```shell
RUN tar -xzvf /usr/local/*.tar.gz \
	&& rm -rf /usr/local/*.tar.gz \ 
	&& chmod +x /usr/local/jdk1.8.0_191
```

* 使用`\`来分行代码， 使代码简洁
* `&&`用于顺序执行代码。

------

## 构建镜像

```shell
# -t 指定最终镜像名称 
#  . 在当前上下文环境寻找Dockerfile文件构建
$ docker build -t mynginx:v1.1 .
SSending build context to Docker daemon  2.048kB
Step 1/2 : FROM nginx
 ---> 568c4670fa80
Step 2/2 : RUN echo '<h1>hello, zibuのDocker' > /usr/share/nginx/html/index.html
 ---> Using cache
 ---> 33fe4ae357fa
Successfully built 33fe4ae357fa
Successfully tagged mynginx:v1.1
```

从输出结果中，我们能清晰的看到镜像的构建过程。

1. 选择nginx镜像（若本地没有nginx镜像，则会从远端拉取镜像）
2. 运行nginx镜像，生成容器（示例中没有，而是标注了Using cache，这个可以在下文中Cache机制解释）；对于新启动的容器 进行RUN指令执行，commit后生成了新的镜像

> Tips: docker build 可以支持URL，tar包，输入流等多种方式

------

## 镜像构建上下文（Context）

> docker build -t  name:version . 

这个`.`,其实并不是指定当前目录下寻找Dockerfile。而是指定**上下文路径**。

我们首先要理解`docker build`的原理。Docker其实分为两个部分，`Docker Daemon`和`Docker Client`。Docker引擎提供了一系列的Rest API，我们可以通过这些API与Docker引擎交互。这也就是Docker Compose的实现原理。

当我们在调试远端服务器Docker环境时，一种方式是SSH链接，直接操作Docker，还可以使用Docker Client来连接远程Docker。这样的话就出现了一个问题，如何让服务器获取本地文件。这就引入了上下文的概念。

当构建时，用户指定构建镜像上下文，`docker build`就会打包该路径下的所有文件，上传给Docker引擎。Docker引擎再展开打包文件，进而执行。

### 示例：

```dockerfile
# 正确案例
COPY ./package.json /app/

# 错误案例 
# 相对路径 和 绝对路径在 docker build . 当前目录上的上下文环境都无法被找到
COPY ../package.json /app
COPY /opt/xxxx /app
```

如果观察 `docker build` 输出，我们其实已经看到了这个发送上下文的过程：

```bash
$ docker build -t nginx:v3 .
Sending build context to Docker daemon 2.048 kB
...
```

一般来说，应该会将 `Dockerfile` 置于一个空目录下，或者项目根目录下。如果该目录下没有所需文件，那么应该把所需文件复制一份过来。如果目录下有些东西确实不希望构建时传给 Docker 引擎，那么可以用 `.gitignore` 一样的语法写一个 `.dockerignore`，该文件是用于剔除不需要作为上下文传递给 Docker 引擎的。

默认情况下，如果不额外指定 `Dockerfile` 的话，会将上下文目录下的名为 `Dockerfile` 的文件作为 Dockerfile。这只是默认行为，实际上 `Dockerfile` 的文件名并不要求必须为 `Dockerfile`，而且并不要求必须位于上下文目录中，比如可以用 `-f ../Dockerfile.php` 参数指定某个文件作为 `Dockerfile`。

当然，一般大家习惯性的会使用默认的文件名 `Dockerfile`，以及会将其置于镜像构建上下文目录中。

## 相关资料

[「Allen 谈 Docker 系列」docker build 的 cache 机制](http://open.daocloud.io/docker-build-de-cache-ji-zhi/)

[Docker gitbook](https://yeasy.gitbooks.io/docker_practice/image/build.html)