---
title: ES-IngestPipeline
date: 2020-10-07 16:30:26
tags: [Elasticsearch]
---

Ingest pipeline 用于拦截数据索引请求，对数据执行预定义处理逻辑然后再继续索引数据。

只要节点具有 `Ingest role`就能够执行 Ingest Pipeline，每个索引都可以设置默认Ingest  Pipeline，该参数可以在索引时覆盖。

<!--more-->

```json
// 创建一个 Pipeline
PUT _ingest/pipeline/my-pipeline-id
{
  "description" : "describe pipeline",
  "processors" : [
    {
      "set" : {
        "field": "foo",
        "value": "bar"
      }
    }
  ]
}
```

# 如何测试 Ingest Pipeline

使用 `_simulate` API 可以模拟执行 Ingest Pipeline，需要指定 **Ingest Pipeline Id** 或者**在 Json 中定义 Ingest Pipeline**, 并携带文档内容。

```json
POST _ingest/pipeline/my-pipeline-id/_simulate
{
  "docs" : [
    { "_source": {/** first document **/} },
    { "_source": {/** second document **/} },
    // ...
  ]
}

POST _ingest/pipeline/_simulate
{
  "pipeline" :
  {
    "description": "_description",
    "processors": [
      {
        "set" : {
          "field" : "field2",
          "value" : "_value"
        }
      }
    ]
  },
  "docs": [
    {
      "_index": "index",
      "_id": "id",
      "_source": {
        "foo": "bar"
      }
    },
    {
      "_index": "index",
      "_id": "id",
      "_source": {
        "foo": "rab"
      }
    }
  ]
}
```

# Pipeline 中如何访问数据

Pipeline 中的 Processor具有文档的读写权限，能够获取文档的**内容和元数据(Metadata)**。同时也能获取 **ingest元数据**

* Doc Metadata:  `_index`, `_type`, `_id`, `_routing`
* Ingest Metadata: `_ingest.timestamp`

```json
POST _ingest/pipeline/_simulate
{
  "pipeline": {
    "description": "zibu_test_pipeline",
    "processors": [
      {
        "set": {
          "field": "_source.context", // 为某字段赋值
          "value": "{{_index}} {{_source.content}}" //目标索引名称和字段值将会合成一个字符串，中间有一个空格
        }
      },
      {
        "set": {
          "field": "ingestTime", // _source可以不加
          "value": "{{_ingest.timestamp}}" // 访问 Ingest Metadata
        }
      }
    ]
  },
  "docs": [
    {
      "_index": "zibu",
      "_source": {
        "content": "zibu",
        "indexName": "zibu_rack1"
      }
    }
  ]
}
```

# Pipeline 中的条件判断语句

## `?.` 解决嵌套字段的NPE 问题

`?.`运算符会对前一个字段值进行非空检查，如果为`Null`则停止继续向下访问。因此在处理嵌套的字段时，使用`?.`运算符，来解决 NPE 问题。

因此对于嵌套类型的所有路径，都使用`?.`是正确的。

直接看栗子

```json
// 传入了两个文档，一个具有 network.name.last 对象，一个没有
// 当我们使用了?.操作符时，Pipeline 正常运行
// 当取出任意一个?.操作符时，不具有 network.name.last的文档都会出现 NPE 问题。
POST _ingest/pipeline/_simulate
{
  "pipeline": {
    "processors": [
      {
        "drop": {
          "if": "ctx.network?.name?.last == 'li'" // 我们为两层嵌套的父元素都增加了?.操作符来处理 NPE 问题
        }
      },
      {
        "set": {
          "field": "network?.aaa",
          "value": "bbb"
        }
      }
    ]
  },
  "docs": [
    {
      "_index": "zibu",
      "_source": {
        "network": {
          "name": {
            "last": "li"
          }
        }
      }
    },
    {
      "_index": "zibu",
      "_source": {}
    }
  ]
}
```

## Painless 复杂的条件判断

`if`的条件判断其实就是 Painless Script，但是需要有一个 Boolean 返回值，那么如果熟悉Painless 语法就可以写出更为复杂的判断的语句。

```json
PUT _ingest/pipeline/not_prod_dropper
{
  "processors": [
    {
      "drop": {
        "if": "Collection tags = ctx.tags;if(tags != null){for (String tag : tags) {if (tag.toLowerCase().contains('prod')) { return false;}}} return true;"
      }
    }
  ]
}

PUT _ingest/pipeline/not_prod_dropper
{
  "processors": [
    {
      "drop": {
        "if": """
            Collection tags = ctx.tags;
            if(tags != null){
              for (String tag : tags) {
                  if (tag.toLowerCase().contains('prod')) {
                      return false;
                  }
              }
            }
            return true;
        """
      }
    }
  ]
}
```



## Pipeline组合形成流处理

基于 If 的条件判断和 `pipeline processor`,能够实现 Pipeline 之间的组合，更为实用和模块化。

比如下面的例子，就使用了一个 Pipeline，然后根据字段值的不同连接不同的 Pipeline.

```json
PUT _ingest/pipeline/logs_pipeline
{
  "description": "A pipeline of pipelines for log files",
  "version": 1,
  "processors": [
    {
      "pipeline": {
        "if": "ctx.service?.name == 'apache_httpd'",
        "name": "httpd_pipeline"
      }
    },
    {
      "pipeline": {
        "if": "ctx.service?.name == 'syslog'",
        "name": "syslog_pipeline"
      }
    },
    {
      "fail": {
        "message": "This pipeline requires service.name to be either `syslog` or `apache_httpd`"
      }
    }
  ]
}
```

# 处理Pipeline错误

TODO



# Processors

TODO

