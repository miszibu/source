---
title: docker-基础入门
date: 2018-07-5 11:50:01
tags: [Docker]
---

> Build , Ship and Run Any App , Anywhere

本文主要讲解使用Dockerfile来创建项目镜像，然后使用容器执行的过程。

Docker相关操作会同步更新在docker脑图中

[zibuのDocker mind picture](https://www.processon.com/view/link/5b3db86de4b0ade3e26d6e7f)

<!-- more -->



------

#### 创建Dockerfile

1.我们需要打包一个muc项目的Image。因此需要muc的代码文件，如果直接拷贝Muc代码文件，需要将其的引用包也同步导入，不方便。直接使用go build 工具进行打包

```shell
 cd $GOPATH/src/muc/src/main
 go build muc.go  #直接build源码，golang会在同目录下生成二进制可执行文件
```

2.在任意地方创建文件夹，在文件夹下放入Dockerfile。

   为了方便和条理性，我在/var/lib/docker文件夹下创建，并且将相关文件移到该目录下

```bash
cd /var/lib/docker
mkdir dockerFiles
cd dockerFiles
touch Dockerfile #必须以Dockerfile命名 否则不识别，一次一个image的dockerfile最好单独放个文件夹存储
mv $GOPATH/src/muc/src/main/muc  . #将muc文件移到当前目录下，为了方便
vi Dockerfile
FROM scratch		#scratch 是一个空的image
ADD muc /			#将muc文件添加到 新镜像的根目录下/
CMD ["/muc"]		#执行muc文件 由于存放到根目录 只需要/muc 否则相对路径是./muc
:wq

docker build -t muc:zibu . #在当前目录下build Dockerfile
docker images #已添加镜像 可使用load/save命令打包镜像
```



***

#### 相关资料

《Docker技术入门与实战 第二版》