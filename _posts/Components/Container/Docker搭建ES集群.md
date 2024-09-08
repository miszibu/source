---
title: Docker搭建ES集群
date: 2020-04-26 17:04:29
tags: [Docker]
---

使用 Docker 搭建本地ES 集群

<!--more-->



## 下载镜像

从 [Docker Hub](https://hub.docker.com/layers/elasticsearch/library/elasticsearch/7.6.2/images/sha256-f29607b18ae0d086c444b93bae5b4a949a7f75dd8bec68b5b0e0f4be8c1ea3a7?context=explore)中选择对应版本的 ES 镜像下载，本次案例中选择了 v7.6.2 

在技术这方面，肯定还是选择相对新的版本研究比较好。

> docker pull docker.elastic.co/elasticsearch/elasticsearch:7.6.2

## 启动容器

### 启动单节点

```sh
docker run --name es_single_node -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" elasticsearch:7.6.2
```

### 启动多节点集群

新建一个`docker-compose.yml` 文件

```yaml
version: '2.2'
services:
  es01:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.6.2
    container_name: es01
    environment:
      - node.name=es01
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es02,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - data01:/Users/ligaofeng/test_homes/elastic-docker/es_cluster_node1/data
    ports:
      - 9200:9200
      - 9300:9300
    networks:
      - elastic
  es02:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.6.2
    container_name: es02
    environment:
      - node.name=es02
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - data02:/Users/ligaofeng/test_homes/elastic-docker/es_cluster_node2/data
    networks:
      - elastic
  es03:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.6.2
    container_name: es03
    environment:
      - node.name=es03
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es02
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - data03:/Users/ligaofeng/test_homes/elastic-docker/es_cluster_node3/data
    networks:
      - elastic

volumes:
  data01:
    driver: local
  data02:
    driver: local
  data03:
    driver: local

networks:
  elastic:
    driver: bridge
```

1. 确保 Docker 拥有足够多的内存来分配给三个 ES 集群。在 perference 页修改 Docker-desktop 可分配资源的量。本案例中，每个实例默认 JVM 大小为512MB，共有三个实例。每个 ES 进程JVM内存大小，不宜超过系统内存的50%。因此我们分配了4GB。

2. 使用 `docker-compose` 命令启动 docker

   > docker-compose up

3. 查看集群

   > curl -X GET "localhost:9200/_cat/nodes?v&pretty"

4. 关闭集群

   > docker-compose down
   >
   > // 关闭集群并删除s数据卷
   >
   > Docker-compose down -v 

5. 使用 docker logs 查看日志

## Shell

```shell
$ES_VERSION=7.6.2
$ES_CONFIG_HOME=/Users/ligaofeng/test_homes/pkgs/elasticsearch-7.6.2/config
$ES_INSTALLTION_HOME=/Users/ligaofeng/test_homes/elastic-docker/

mkdir -p $ES_INSTALLTION_HOME/es_cluster_node0
mkdir -p $ES_INSTALLTION_HOME/es_cluster_node1
mkdir -p $ES_INSTALLTION_HOME/es_cluster_node2
cp -rf $ES_CONFIG_HOME $ES_INSTALLTION_HOME/es_cluster_node0
cp -rf $ES_CONFIG_HOME $ES_INSTALLTION_HOME/es_cluster_node1
cp -rf $ES_CONFIG_HOME $ES_INSTALLTION_HOME/es_cluster_node2

docker pull docker.elastic.co/elasticsearch/elasticsearch:$ES_VERSION


```



## Reference

[Install es with docker](https://www.elastic.co/guide/en/elasticsearch/reference/7.6/docker.html)