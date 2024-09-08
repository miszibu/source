---
title: ES-查询-Sort
date: 2020-10-06 14:22:31
tags: [Elasticsearch]
---

对查询过滤后的结果集进行排序，可以对基于 **字段，数组，Nested(JsonArr)**类型进行排序。

当存在多条排序语句时，后续排序只对排序结果相同的文档进行重新排序，而非对全部结果姐

* **字段类型**： 直接排序
* **数值数组**： 选择排序模式，计算排序值，基于排序值排序
* **nested**: 原理同数值数组类型，可以添加filter查询，来针对匹配过滤查询的Json文档才执行排序计算
* geo distance sorting: todo

<!--more-->

```json
// 先根据第一个排序规则进行排序，若存在文档拥有相同的值，再对它们使用下一条排序规则继续排序。
GET /my_index/_search
{
    "sort" : [
        {"price" : {"order" : "asc", "mode" : "avg"}},
        "user",
        { "name" : "desc" },
        { "age" : "desc" },
        "_score"
    ],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

**排序规则**
* **asc:** 升序，除_score外其他类型默认升序
* **desc:** 降序，对于_score默认为降序

## **数组排序: Sort mode**

当某字段为数值数组类型时，可以使用以下不同的Sort mode来计算出该数组的一个统计值，对于该值再进行排序。

* min
* max
* sum
* avg
* median: 选择中间的数据

```json
// price为数值数组
PUT /my_index/_doc/1?refresh
{
   "product": "chocolate",
   "price": [20, 4]
}

POST /_search
{
   "query" : {
      "term" : { "product" : "chocolate" }
   },
   "sort" : [
	  // avg值为（20+4）/2,使用该值进行升序排序
      {"price" : {"order" : "asc", "mode" : "avg"}}
   ]
}
```

## **nested**

原理同数值数组类型，可以添加filter查询，来针对匹配过滤查询的Json文档才执行排序计算

```json
POST /_search
{
   "query": {
      "match_all": {}
   },
   "sort" : [
      {
		      // 取parent.child.age的最小值为统计值升序排序
         "parent.child.age" : {
            "mode" :  "min",
            "order" : "asc",
            "nested": {
               "path": "parent",
               "filter": {// 过滤掉部分Json Docs
                  "range": {"parent.age": {"gte": 21}}
               },
               "nested": {
                  "path": "parent.child",
                  "filter": {// 双重过滤
                     "match": {"parent.child.name": "matt"}
                  }
               }
            }
         }
      }
   ]
}
```

## **Missing value**

对于没有被排序字段排序的文档，默认会被放到最后，也可以设置为`_first`放置最前。

```json
GET /_search
{
    "sort" : [
        { "price" : {"missing" : "_last"} }
    ],
    "query" : {
        "term" : { "product" : "chocolate" }
    }
}
```