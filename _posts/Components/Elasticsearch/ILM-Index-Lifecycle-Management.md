---
title: ILM_Index_Lifecycle_Management
date: 2020-04-13 00:02:27
tags: [Elasticsearch]
---



## **Introduction**

**ILM**(Index Lifecycle Managment)是 ES6.6.版本发布的新的特性，用于管理索引的生命周期。

在**时序型数据**的场景中，比如 Metricbeat 持续不断的发送 metric 到 ES ，以日期为单位创建 Daily Index。我们会希望，单个 Index 的大小不要超过一个合适值，比如50GB，我们也不希望过时的数据，继续占用磁盘空间，那么就要指定一个规则删除无效 Index。这就是 ILM 可以帮助我们做到的。

在这篇 Blog 中，我们会介绍 ILM 所涉及的相关概念，并通过 ILM 来实现时序型日志的定时删除功能，这在之前的版本中往往是需要 由 Crontab 定期触发 curator 来实现的，相对复杂。

<!-- more -->

## Index Life Cycle Introduction

![index-life-cycle](./ILM-Index-Lifecycle-Management/index-life-cycle.png)

这 ILM 中，Index的生命周期被划分为了4个阶段，**Hot  Warm Cold & Delete**。

Index 可以存在最多四个阶段，但并不必须，可任意拼接。在普通的场景中Hot+Delete 是最为常见的组合。

### Hot Phase

Index刚生成时，处于频繁**读写数据**的状态。此时的 Index 会消耗大量的内存和磁盘空间。

### Warm Phase

举例如时序型日志Index在过了数据输入阶段时，此时**只负责数据的查询**。该阶段的index 则为 Warm Phase

### Cold Phase

处于 Cold Phase 的 Index，数据读取查询的请求锐减，节点只需要消耗少量的系统资源即可。

### Delete Phase

Index 不再不需要，寿终正寝。



## 热温冷架构 Hot-Warm-Cold Architecture

在知道了 Index 的生命管理周期后，我们发现不同时期的 Index 对于系统资源的消耗，需要也是不一样的。那么我们可以通过在启动 ES 节点时，手动设置节点的属性为 Hot warm or Cold。将硬件资源好的节点设置为 HOT,将硬件资源一般的设置为 Cold。

```shell
bin/elasticsearch -Enode.attr.data=hot
bin/elasticsearch -Enode.attr.data=warm
bin/elasticsearch -Enode.attr.data=cold
```

在实际设置 ILM 策略时，就可以指定目标节点的属性，将 Hot Index 存储在 Hot 节点上，以此来最大化系统硬件的使用效率。

**Note:**这一部分的功能都是基于[shard allocation awarness](https://www.elastic.co/guide/en/elasticsearch/reference/6.7/allocation-awareness.html) 来实现的，不是必须，此处不再展开。

## Policy

Index 存在4个不同的生命周期，policy 则规定了每个生命周期，对Index 所进行的操作。Policy 不一定需要包含全部四个周期，举例而言，我们可以创建只包含 Hot 和 Delete 阶段的 Policy。.

### Timing

参数`min_age`决定了Indices 何时进入下一个生命周期。只有当 Indices 的时长（Indices在 rollover以后时长会被刷新）大于 `min_age`时，才会进入下一个生命周期。

对任意周期，未指定时`min_age`的默认值为0秒。

```json
PUT _ilm/policy/my_policy
{
  "policy": {
    "phases": {
      "warm": {
        "min_age": "1d",
        "actions": {
          "allocate": {
            "number_of_replicas": 1
          }
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

如上图的案例中，只有 indices的创建时间大于30天，Indices 才会进入 Delete Phase，被删除。

之前阶段的 actions 尚未完成时，indices 暂时不进入下一周期，直至之前阶段的 Action 完成。

### Phase Execution

可以通过 [Explain Lifecycle API](https://www.elastic.co/guide/en/elasticsearch/reference/6.7/ilm-explain-lifecycle.html)来获取 Indices 的状态。

```json
GET <index>/_ilm/explain
```

### Actions

Actions 决定了在个生命阶段，Indices 所做的操作，在不同阶段所支持的 Actions 也是不同的。

Actions 的定义顺序决定了 Actions 的执行顺序，这点需要注意。

具体内容请参考[官方 API](https://www.elastic.co/guide/en/elasticsearch/reference/6.7/_actions.html).

### Full Policy

我们通过一个Policy 案例来详细介绍下。下面的 API 会生成一个名为 full_policy 的 Policy。当处于 Hot阶段的 Indices 的时长大于7天或大小大于50GB 时，将会 rollover 到一个新的 Indices。

After 30 days it enters the warm phase and increases the replicas to 2, force merges and shrinks. After 60 days it enters the cold phase and allocates to "cold" nodes, and after 90 days the index is deleted.

```json
PUT _ilm/policy/full_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_age": "7d",
            "max_size": "50G"
          }
        }
      },
      "warm": {
        "min_age": "30d",
        "actions": {
          "forcemerge": {
            "max_num_segments": 1
          },
          "shrink": {
            "number_of_shards": 1
          },
          "allocate": {
            "number_of_replicas": 2
          }
        }
      },
      "cold": {
        "min_age": "60d",
        "actions": {
          "allocate": {
            "require": {
              "type": "cold"
            }
          }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```



## Practice 

在测试案例中，我们会使用 metricbeat 来生产数据，以分钟为划分在 ES 上创建时序型Indices。通过 ILM Policy来删除时长超过3分钟的 Indices。实际的 Case 中往往以日期划分。

> Note: Metricbeat 与 ILM 存在集成，可以在 Metricbat 端直接配置 ILM，我们此处只是使用 Metricbeat来生成数据，作演示。

### 1. Create a ILM policy

创建一个名为 common_beat_policy 的LIM Policy，并声明当 indices 的时长超过3分钟时，进入 Delete 阶段。

所有被该 Policy 管理的 Indices，时长超过3分钟的都会被删除。

```json
PUT _ilm/policy/common_beat_policy   
{
  "policy": {
    "phases": {
      "delete": {
        "min_age": "3m",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

### 2. Applying a policy to an index template

通过 Get API 获取 Meticbeat-6.7.0 templates，然后将`index.lifecycle.name`参数添加到 template中。那么所有使用该 Template的 Index 都将会应用`comon_beat_policy`

```json
GET _template/metricbeat-6.7.0

PUT _template/metricbeat-6.7.0
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1,
    "index.lifecycle.name": "common_beat_policy"
  }
  ....
}
```

### 3. Modify ILM Polling Interval

ILM Service 会在后台轮询执行 Policy，默认间隔时间为 10 分钟，为了更快地看到效果，我们将其修改为 1 秒。

```json
PUT _cluster/settings
{
  "persistent": {
    "indices.lifecycle.poll_interval":"1s"
  }
}
```

### 4. Startup metricbeat to generate data

以下为 Meticbeat 配置，数据将被写入到本地监听9200端口的 ES 中。并以分钟为划分来创建 Index。

当我们更新`output.elasticsearch.index`时，需要指定 `template.name & template.pattern`

```yml
output.elasticsearch:
  # Array of hosts to connect to.
  hosts: ["localhost:9200"]
  index: "metricbeat-%{[beat.version]}-%{+yyyy.MM.dd.hh.mm}"
setup.template.name: "metricbeat-%{[beat.version]}"
setup.template.pattern: "metricbeat-%{[beat.version]}-*"
```

### 5. Check Indices

通过查看 Indices 可以发现，只有最近的3分钟的 indices 存在，查看具体的 Indices 也可以看到`common_beat_policy`已经生效。

```json
// Check the indices
GET _cat/indices
yellow open metricbeat-6.7.0-2020.04.14.12.51 TPWc03HZSPyk1Pc7Zz16jA 1 1 189 0 282.8kb 282.8kb
yellow open metricbeat-6.7.0-2020.04.14.12.52 zzbBHGQ-SuOHBUu65re5UQ 1 1  92 0 129.8kb 129.8kb
yellow open metricbeat-6.7.0-2020.04.14.12.50 90OhX4rTTJOgp-OLHv0y-g 1 1 196 0 286.9kb 286.9kb

// Get the ilm status of index
GET metricbeat-6.7.0-2020.04.14.12.49/_ilm/explain
{
  "indices" : {
    "metricbeat-6.7.0-2020.04.14.12.49" : {
      "index" : "metricbeat-6.7.0-2020.04.14.12.49",
      "managed" : true,
      "policy" : "common_beat_policy",
      "lifecycle_date_millis" : 1586879342034,
      "phase" : "new",
      "phase_time_millis" : 1586879342080,
      "action" : "complete",
      "action_time_millis" : 1586879342080,
      "step" : "complete",
      "step_time_millis" : 1586879342080
    }
  }
}

```



## Reference

[Implementing a Hot-Warm-Cold Architecture with Index Lifecycle Management](https://www.elastic.co/blog/implementing-hot-warm-cold-in-elasticsearch-with-index-lifecycle-management)

[ES6.7 getting-started-index-lifecycle-management.html](https://www.elastic.co/guide/en/elasticsearch/reference/6.7/getting-started-index-lifecycle-management.html)

[Elasticsearch 6.6 Index Lifecycle Management 尝鲜](https://elasticsearch.cn/article/6358)