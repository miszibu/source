---
title: ES-查询-聚合篇
date: 2020-09-20 15:52:39
tags: [Elasticsearch]
---

聚合函数用于在查询和过滤后的数据集上，进行按类聚合以获取总体统计数据。聚合函数根据其作用方式和原理，被分为以下四个类型

* **Bucketing**: 将文档根据设置好的判定条件，划分到不同的 Bucket 中去。（类似 SQL 的 **GroupBy**）
* **Metric**: 计算一组文档的数据，如 **max，min，avg** 等。当直接使用时会对所有文档计算，**通常作为在 Bucketing 函数的子函数出现**，计算每个 Bucket 的相关 Metric 数据。
* **Matrix:** 顾名思义，该类聚合函数在多个字段上生效，用于生成矩阵。不支持 Scripting（自定义脚本）。
* **Pipeline:** 用于聚合其他聚合函数的结果。

> Notes: 聚合函数嵌套，可以将聚合函数不断嵌套，每个子函数都会在父聚合的结果和输出上继续执行。通常就是 Bucketing 聚合+Metrics 子聚合用于对于不同 Buckets 进行数据总结，再使用 Pipeline 聚合对 Buckets 进行筛选过滤排序展示。

<!--more-->

# 聚合函数的 Json结构

```json
"aggregations" : {
    "<aggregation_name>" : {
        "<aggregation_type>" : {
            <aggregation_body>
        }
        [,"meta" : {  [<meta_data_body>] } ]?
        [,"aggregations" : { [<sub_aggregation>]+ } ]?
    }
    [,"<aggregation_name_2>" : { ... } ]*
}

"aggs": {
  "name_aggs": {
    "terms": {
      "field": "name.keyword"
    },
    "aggs": {
      "color_aggs": {
        "terms": {
          "field": "color.keyword"
        }
      }
    }
  }
```

`aggregations` 可以使用`aggs`简化代替。 

`<aggregation_name>`只是聚合函数名称，使用具有意义的名字表示聚合函数的功能。

`<aggregation_type>`是聚合函数的具体类型，如Buckets 聚合中常见的 terms（按名称聚合）。



# Demo case for introduction 

## Summary

本小节通过一个例子展示，聚合函数中三大类是如何组合使用，来实现一个具体的功能。

聚合函数组合的通常顺序如下：

> Bucketing聚合将原始数据装了 Buckets 中，Metrics聚合对各个 Buckets 中数据进行统计，最后使用 Pipeline聚合，对所有的 Buckets进行筛选和排序。



我们最终的目标是实现以下的这条 SQL

```sql
SELECT model,SUM(price) AS total_price FROM cars GROUP BY model HAVING total_price > 100000 ORDER BY total_price DESC LIMIT 2;
```

## 动态 Mapping 导入原始数据

```json
POST _bulk
{"index":{"_index":"cars","_type":"doc","_id":"1"}}
{"name":"bmw","date":"2017-06-01", "color":"red", "price":30000}
{"index":{"_index":"cars","_type":"doc","_id":"2"}}
{"name":"bmw","date":"2017-06-30", "color":"blue", "price":50000}
{"index":{"_index":"cars","_type":"doc","_id":"3"}}
{"name":"bmw","date":"2017-08-11", "color":"red", "price":90000}
{"index":{"_index":"cars","_type":"doc","_id":"4"}}
{"name":"ford","date":"2017-07-15", "color":"red", "price":20000}
{"index":{"_index":"cars","_type":"doc","_id":"5"}}
{"name":"ford","date":"2017-07-01", "color":"blue", "price":40000}
{"index":{"_index":"cars","_type":"doc","_id":"6"}}
{"name":"bmw","date":"2017-08-01", "color":"green", "price":10000}
{"index":{"_index":"cars","_type":"doc","_id":"7"}}
{"name":"jeep","date":"2017-07-08", "color":"red", "price":110000}
{"index":{"_index":"cars","_type":"doc","_id":"8"}}
{"name":"jeep","date":"2017-08-25", "color":"red", "price":230000}
```

## 查看动态 Mapping

```shell
GET cars/_mappings

{
  "cars" : {
    "mappings" : {
      "properties" : {
        "color" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "date" : {
          "type" : "date"
        },
        "name" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "price" : {
          "type" : "long"
        }
      }
    }
  }
}

```

## Buckets Terms 聚合

使用size:0 过滤掉所有命中的 Docs，来简化返回。

第一步，我们希望数据按照名字聚合，因此选择 name.keyword 对其使用 **terms** 进行聚合。

```json
POST cars/_search
{
	"size": 0,
	"aggs": {
		"name_aggs": {
			"terms": {
				"field": "name.keyword"
			}
		}
	}
}
```

## Metrics Sum 计算总价

```json
POST cars/_search
{
	"size": 0,
	"aggs": {
		"name_aggs": {
			"terms": {
				"field": "name.keyword"
			},
			"aggs": {
				"total_price": {
				   "sum": {
				   		"field": "price"
				   }
				}
			}
		}
	}
}
```

## Pipeline Bucket_selector 过滤

```json
POST cars/_search
{
	"size": 0,
	"aggs": {
		"name_aggs": {
			"terms": {
				"field": "name.keyword"
			},
			"aggs": {
				"total_price": {
				   "sum": {
				   		"field":  "price"
				   }
				},
				"total_price_selector": {
				  "bucket_selector": {
				    "buckets_path": {
				      "totalPrice" : "total_price"
				    },
				    "script": "params.totalPrice>100000"
				  }
				}
			}
		}
	}
}
```

## Pipeline Bucket_sort 基于Bucket 排序

```json
POST cars/_search
{
	"size": 0,
	"aggs": {
		"name_aggs": {
			"terms": {
				"field": "name.keyword"
			},
			"aggs": {
				"total_price": {
				   "sum": {
				   		"field":  "price"
				   }
				},
				"total_price_selector": {
				  "bucket_selector": {
				    "buckets_path": {
				      "totalPrice" : "total_price"
				    },
				    "script": "params.totalPrice>100000"
				  }
				},
				"total_prices_sorter":{
				  "bucket_sort": {
				    "sort": {
				     "total_price": "desc" 
				    }
				  }
				}
			}
		}
	}
}
```

## Final Result

```json
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 8,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "name_aggs" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : "jeep",
          "doc_count" : 2,
          "total_price" : {
            "value" : 340000.0
          }
        },
        {
          "key" : "bmw",
          "doc_count" : 4,
          "total_price" : {
            "value" : 180000.0
          }
        }
      ]
    }
  }
}
```



# Buckets

| Aggregation                                                  | Elasticsearch | MySQL     |
| ------------------------------------------------------------ | ------------- | --------- |
| Childen——父子文档                                            | Yes           | /         |
| **Date Histogram——基于时间(按年/月/日等等)分桶**             | Yes           | Complex   |
| Date Range                                                   | Yes           | Complex   |
| Filter                                                       | Yes           | n/a (yes) |
| Filters                                                      | Yes           | n/a (yes) |
| Geo Distance                                                 | Yes           | /         |
| GeoHash grid                                                 | Yes           | /         |
| Global                                                       | Yes           | n/a (yes) |
| Histogram                                                    | Yes           | Complex   |
| IPv4 Range                                                   | Yes           | Complex   |
| Missing                                                      | Yes           | Yes       |
| Nested                                                       | Yes           | /         |
| Range                                                        | Yes           | Complex   |
| Reverse Nested                                               | Yes           | /         |
| Sampler                                                      | Yes           | Complex   |
| Significant Terms                                            | Yes           | No        |
| **Terms——为字段中每个 Unique 的值建桶**,<br />text 类型则是为每个分词建桶，keyword 类型是为整个字段建桶 | Yes           | Yes       |

# Metrics

| Aggregation                                            | Elasticsearch      | MySQL                       |
| ------------------------------------------------------ | ------------------ | --------------------------- |
| **Avg**                                                | Yes                | Yes                         |
| **Cardinality——去重唯一值**                            | Yes (Sample based) | Yes (Exact)——类似：distinct |
| Extended Stats                                         | Yes                | StdDev bounds missing       |
| Geo Bounds                                             | Yes                | /                           |
| Geo Centroid                                           | Yes                | /                           |
| **Max**                                                | Yes                | Yes                         |
| **Min**                                                | Yes                | Yes                         |
| Percentiles                                            | Yes                | Complex SQL or UDF          |
| Percentile Ranks                                       | Yes                | Complex SQL or UDF          |
| Scripted                                               | Yes                | No                          |
| Stats                                                  | Yes                | Yes                         |
| **Top Hits——为 Buckets中的文档进行排序和指定显示数量** | Yes                | Complex                     |
| Value Count                                            | Yes                | Yes                         |
| **Sum**                                                | Yes                | Yes                         |

# Pipeline 



# Matrix

略，[自行官方文档获取](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-pipeline.html)

# Reference

[Elasticsearch如何实现 SQL语句中 Group By 和 Limit 的功能](https://elasticsearch.cn/article/629#tip4)

[干货 | 通透理解Elasticsearch聚合](https://blog.csdn.net/laoyang360/article/details/82932516)

[ES Reference 7.9 Aggregations](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html)

