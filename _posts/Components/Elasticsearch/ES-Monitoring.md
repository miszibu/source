---
title: ES-Monitoring
date: 2020-09-11 14:19:04
tags: [Elasticsearch]
---

通过 ES Monitoring 监控ELK Stack 各个组件的 Metric数据。

<!--more-->

# 启用 Monitoring

## Beats

在 Beats 端，收集Monitoring Data 有两种方式，

* **Internal Collection**：自己收集 metric，并直接发送数据给 Monitoring ES，相比 metricbeat收集 metric 的方式，就不需要维护其他的 metricbeat。
* **Metricbeat Collection**：在 ES7.3以后的版本，新增了由 Metricbeat来收集其他 Beats Metric的方式。

### Internal Collection

```yml
# Enable Monitoring In Filebeat 
monitoring:
  enabled: true
  # GET / 获取目标ES集群的 UUID
  cluster_uuid: PRODUCTION_ES_CLUSTER_UUID 
  elasticsearch:
    hosts: ["https://example.com:9200", "https://example2.com:9200"] 
    # API Key || User/Pwd 任选一个
    api_key:  id:api_key 
    username: beats_system
    password: somepassword
```

### Metricbeat Collection

```yml
# 配置被监控组件（以 Filebeat 为例）
# Expose http port 
http.enabled: true
http.port: 5067
# Disable internal metric collection
monitoring.enabled: false

# ==========================================
# 在被监控组件所处的服务器上安装 Metricbeat(不同服务器为什么不行？不是通过 Http 请求获取 metric 的吗？有待测试)
# 打开 beat-xpack 模块
./metricbeat modules enable beat-xpack
# 配置被监控组件相关信息
vi modules.d/beat-xpack.yml 
- module: beat
  metricsets:
    - stats
    - state
  period: 10s
  hosts: ["http://localhost:5066"]
  #username: "user"
  #password: "secret"
  xpack.enabled: true
  
# ==========================================
# 修改ES相关配置信息
vi metricbeat.yml
output.elasticsearch:
  # Array of hosts to connect to.
  hosts: ["http://es-mon-1:9200", "http://es-mon2:9200"] 
  # Optional protocol and basic auth credentials.
  #protocol: "https"
  #api_key:  "id:api_key" 
  #username: "elastic"
  #password: "changeme"
```

# ES Enable Monitoring

作为监控数据的最终目的地，ES需要激活 monitoring功能。

```shell
PUT _cluster/settings
{
  "persistent": {
    "xpack.monitoring.collection.enabled": true
  }
}
```

# View Stack Monitroing In kibana



![](/Users/ligaofeng/blog/source/_posts/Components/Elasticsearch/ES-Monitoring/1.png)