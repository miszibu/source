---
title: ES-CCR
date: 2020-07-29 23:33:11
tags: [Elasticsearch]
---

CCR (Cross Cluster Replication), 

本文会介绍如何在**ES6.7**环境下搭建CCR 双向复制模型， 并测试**HTTP**和**TCP Client**(ES7.0版本为Deprecated, 8.0正式移除)两种模式是否能够实现数据跨集群备份。

<!--more-->

# 集群搭建及参数设置

## 集群搭建：

我们需要搭建两个ES集群，一个为主集群，一个为备份集群。但是他们需要使用**相同的集群名称**

> 为什么需要相同的集群名称？
>
> 正常情况下，如果与ES的连接都是走HTTP协议的，那肯定使用不同的集群名称来区分集群。
> 在本文中，为了同时支持TCP Client，TCP Client需要配置集群名称，如果主副集群名称不一致，当主集群下线时，请求会forward到备用集群，因此我们需要确保相同的配置也能连接到备用集群，其中就包括证书，集群名称，账户密码等。

## 环境设置

### Define Remote Cluster

我们需要为主副集群都配置另外一个集群的连接信息.

```shell
curl -XPUT -u elastic:xxxxxxxxxxx "http://{{primary_cluster}}:{{http_port}}/_cluster/settings?pretty" -H 'Content-Type: application/json' -d'
{
  "persistent" : {
    "cluster" : {
      "remote" : {
        "{{cluster-name}}" : {
          "seeds" : [
            "{{node-0}}:{{tcp-port}}",
            "{{node-1}}:{{tcp-port}}",
            "{{node-2}}:{{tcp-port}}"
          ]
        }
      }
    }
  }
}
'
```

查看Remote Cluster定义是否正确

```shell
#Get _cluster/settings

curl -XGET -u elastic:xxxxxxxxxxx "http://{{server}}:{{http_port}}/_cluster/settings?pretty"

#Get remote information

curl -XGET -u elastic:xxxxxxxxxxx "http://{{server}}:{{http_port}}/_remote/info?pretty" 
```

### Template 设置修改

CCR需要Primary Indices的 `soft_deletes` 设置是激活的，而这个参数在7.0以后的版本是默认激活的，因此在6.7中我们需要手动激活。

我们也需要添加 aliases  ，因为对follower index我们会给到一个follower的前缀(见Auto-follower pattern)，因为集群Kibana中配置了很多默认的Index-Pattern, 它们是匹配不到有follower 前缀的索引的，在这里添加一个别名指向自己就可以解决这个问题。

当然修改index-pattern也是一个方案，index-pattern 的修改会影响到dashboard和visualization的数据展示，为了避免导致的问题，添加别名可能是更好的方案，考虑到对集群的影响下。

#### Beat Template Change

如果使用Beats来发送数据，我们可以预先在es上创建tempaltes.

在两个集群上同时上传template, 注意beats的yml文件中需要预先设置es连接

```shell
./metricbeat setup --template -c metricbeat_es_cl1.yml

./metricbeat setup --template -c metricbeat_es_cl1.yml
```

修改Template

```shell
#获取template内容
GET  _template/{{template_name}}

#更新template
PUT _template/{{template_name}}
{
    {Other-Setting}
    "settings": {
        "index": {
             "soft_deletes": {
                  "enabled": "true"
              },
          }
    }
    "aliases": {"{{alias-name}}":{}}
}'
```

#### Self-generated Template

对于我们自己创建的索引而言，我们也同时需要做这样的操作，和上面基本一样。



### 创建Auto-Follower Pattern

我们需要在两个集群上都创建`auto-follower pattern`. 

* 需要声明remote_cluster name， 因为我们可能会有很多remote-cluster connection.
* 还需要声明leader index pattern，既怎么样的Primary Index需要被选中，这里只支持* 通配符。


follower-index-pattern中，我们使用了leader_index 这个字符串，这是一个内嵌的变量，会被映射为primary index的名字，而其他的花括号都是需要填入真实的数据

```shell
GET /_ccr/auto_follow/
PUT /_ccr/auto_follow/{{auto-follower-pattrn-name}}
{
    "remote_cluster" : "{{remote-cluster-name}}",
    "leader_index_patterns" : [
        "metricbeat-6.7.0-*"
    ],
    "follow_index_pattern" : "follower-{{leader_index}}"
}
```

## Loadbalance

所有的数据都不直接连接到ES集群，而是连接到LoadBalance,  当集群下线，由LB自动来切换集群。

一开始想选用Haproxy，然后他forward tcp请求遇到点问题，懒得解决了，就直接用nginx了。

### Nginx Preparation

这里为了方便和尽量不影响测试机器上的环境，我们使用docker 来装nginx.

### Nginx Configuartion

```nginx
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /var/log/nginx/access.log  main;
    sendfile        on;
    #tcp_nopush     on;
    keepalive_timeout  65;
    #gzip  on;
    include /etc/nginx/conf.d/*.conf;
    
    server {
        listen 9001;
        location / {
            proxy_pass http://http_backend;
        }
	}

    upstream http_backend {
       server gdcdc80490:9204;
       server dc3dc80189:9204 backup;
    }
}

stream {
    server {
        listen     9002;
        #TCP traffic will be forwarded to the "stream_backend" upstream group
        proxy_pass stream_backend;
    }

    upstream stream_backend {   
        server gdcdc80490:9303;
        server dc3dc80189:9303 backup;
    }
}
```



### Start Nginx Container

```shell
docker pull nginx:stable
docker run --name {{container_name}} -v {{your_nginx_cfg_path}}:/etc/nginx/nginx.conf:ro -p 9002:9001 -p 9001:9001 -d {{image_id}}
```

## HTTP Test

这里我们使用Metricbeat来发送数据测试HTTP请求的Automatic Failover

Metricbeat Config

Metricbeat connect to loadbalance

```yaml
output.elasticsearch:
  hosts: ["{{nginx_server}}:9001"] 
  protocol: "http"
  username: "elastic"
  password: "{{pwd}}"
```

Startup metricbeat

> ./metricbeat  -c {{config.yml}}

## TCP Test

这里我们使用ES 的Transport Client去作为TCP一端的测试，

```java
    try {
        String truststorePath = "C:\\Users\\xxxx\\xxxx\\Elasticsearch\\Certification\\xxxx-xxx.p12";
        String clusterName = "ES_CL1";

        Settings settings = Settings.builder()
                .put("cluster.name", clusterName)
                .put("client.transport.sniff", true)
                .put("xpack.security.transport.ssl.enabled", true)
                .put("xpack.security.transport.ssl.keystore.path", truststorePath)
                .put("xpack.security.transport.ssl.truststore.path", truststorePath)
                .put("xpack.security.transport.ssl.keystore.password", "logaggcert")
                .put("xpack.security.transport.ssl.truststore.password", "logaggcert")
                .put("xpack.security.transport.ssl.enabled", "true")
                .put("xpack.security.transport.ssl.verification_mode", "certificate")
                .put("xpack.security.user", "elastic:dopdevPassword")
                .build();

        TransportClient transportClient = new PreBuiltXPackTransportClient(settings);

        try {
            transportClient.addTransportAddress(new TransportAddress(InetAddress.getByName("itsrhv17064"), 9002));
        } catch (UnknownHostException e) {
            e.printStackTrace();
        }

        Map<String, Object> json = new HashMap<String, Object>();
        json.put("user", "kimchy_java_api");
        json.put("postDate", new Date());
        json.put("message", "trying out Elasticsearch");
        while (true) {
            IndexResponse response = transportClient.prepareIndex("zibu", "_doc")
                    .setSource(json, XContentType.JSON)
                    .get();
        }
        // on shutdown

        //client.close();
    } catch (Exception e) {
        throw new RuntimeException("Error preparing ESBolt: " + e.getMessage(), e);
    }
```

## 如何测试 

整个测试流程可以分为一下几个步骤

1. 启动MetricBeat 和 ES TCP Client，测试整个DataFlow是否正常。备用集群能否创建follower-index.
2. 关掉主集群，测试备用集群能否正常工作。
3. 重启主集群，测试主集群能否为备用集群上的primary-index来创建follower-index

### Ansible To Manage ES Cluster

```yaml
---
- hosts: es_cl1
  gather_facts: false
  vars:
    ece_storage_path: /data/ecedata
  tasks:
   - name: Start ES Nodes
     shell: hostname
     register: result

   - debug:
      var: result

   - name: Stop ES Nodes
     shell: ps -fe|grep ela |grep -v grep | awk '{print $2}'| xargs kill -9
     ignore_errors: yes
   # 注意设置Java-Home
   - name: Start ES Nodes
     shell: nohup /usr/local/clo/ven/ESCluster671/tools/esnode/bin/elasticsearch -d &
     environment:
        JAVA_HOME: /opt/jdk
     register: eslog

   - debug:
      var: eslog
```