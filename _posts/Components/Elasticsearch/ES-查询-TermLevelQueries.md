---
title: ES-查询-TermLevelQueries
date: 2020-10-07 08:33:55
tags: [Elasticsearch]
---

Term查询需要对**关键词进行全匹配，如果存在空格和大小写的问题也会影响匹配。**

因此对于 `Keyword`类型的字段必须完全一致，对于`text`类型的字段则必须跟分词后的 `token`完成一致。文档才算匹配。

* **Term**： 返回**精确匹配term**(必须完全一致)的文档，不能正常使用于Text字段(解释见下文)。

* Terms： 同Term查询，且允许对多个 terms进行查询，只要匹配其中任意一个 term，文档就算被匹配。
  * Terms lookup: 将某个文档的字段的值作为Term list.

* Terms set: 同Terms，但是可以限制最少匹配的terms数量。

* **Wildcard**: 可以在想要查询的terms中添加通配符。

* **Regexp**: Wildcard的扩充版，支持在查询的Terms中使用正则表达式。如果查询不如预期，请考虑Lucence解析正则时，有限状态机设置。

* Prefix: 匹配字段值中包含某个特定前缀的文档

* **Range**: 针对Date, 数字，Range类型的范围查询

* Fuzzy: 返回term与查询term相似的文档， 会对输入的文档进行根据编辑距离的模糊匹配。

* **Exists**: 返回某个字段值不为空的所有文档。

* IDs: 批量返回与输入ID相匹配的文档。

<!--more-->

#  Term

**返回精确匹配查询语句的文档，但是不能正常使用于Text字段。**

Term查询必须精确匹配，比如下面查询则只会匹配 user 字段值为`Kimchy`的文档，哪怕多了个空格也无法成功匹配。

```json
GET /_search
{
    "query": {
        "term": {
            "user": {
                "value": "Kimchy", 
                "boost": 1.0 // Optional,浮点数，默认为1.0， 用于调整查询语句的相关性评分
            }
        }
    }
}
```

对于 text 字段而言，es 会使用使用analysis 对其进行分词，这导致想从 text 字段类型中找出准确的值非常困难。

通过以下例子来演示下，term 和 match 查询对于 text 字段类型的不同结果。

1. 创建 Index

   ```json
   PUT my_index
   {
       "mappings" : {
           "properties" : {
               "full_text" : { "type" : "text" }
           }
       }
   }
   ```

2. 插入演示数据
   由于字段类型是 text，ES在文本分析阶段将 `Quick Brown Foxes!` 转为了 `[quick, brown, fox]` 

   ```json
   POST my_index/_doc
   {
     "full_text":   "Quick Brown Foxes!"
   }
   POST my_index/_doc
   {
     "full_text":   "Quick"
   }
   ```

3. 使用 `term`查询，以下查询均无法匹配文档，因为字段 full_text 已经不再保存确切的 term `Quick Brown Foxes!`了，而是保存 `[quick, brown, fox]` ，因此使用 term 查询无法匹配。

   ```json
   GET my_index/_search?pretty
   {
     "query": {
       "term": {
         "full_text": "Quick Brown Foxes!"
       }
     }
   }
   
   GET my_index/_search?pretty
   {
     "query": {
       "term": {
         "full_text": "Quick"
       }
     }
   }
   ```

4. Match 查询可以匹配两个文档。

   ```json
   GET my_index/_search?pretty
   {
     "query": {
       "match": {
         "full_text": "Quick Brown Foxes!"
       }
     }
   }
   ```

# Terms

给定一个 term list，返回指定 filed中包含任意一个 term的文档。

```json
// 返回user 字段匹配 kimchy 或 elasticsearch的文档
GET /_search
{
    "query" : {
        "terms" : {
            "user" : ["kimchy", "elasticsearch"],
            "boost" : 1.0
        }
    }
}
```

## Terms lookup

Terms查询需要指定 term list, Terms lookup 可以指定某个索引中的某个文档中的某个字段的全部值作为 term list.

```json
// 选择my_index的id为2的文档的color字段的值作为term list
GET my_index/_search?pretty
{
  "query": {
    "terms": {
        "color" : {
            "index" : "my_index",
            "id" : "2",
            "path" : "color"
        }
    }
  }
}
```

# Terms set

Terms set: 同Terms，但是可以限制最少需要匹配的terms数量。需要指定 terms 和最小匹配 term 的数量（通过 script 或者 field 来指定）

> 其实这个功能有点鸡肋，我猜测本质上还是一个 boolean query，那我们直接以 boolean query 来实现需求即可。

* **terms**: 想要匹配的 term list
* **minimum_should_match_field**: 最小需要匹配的 term 数量，这里需要指定一个 numeric 类型的字段。
* **minimum_should_match_script**：可以使用 script 语法

```json
GET /job-candidates/_search
{
    "query": {
        "terms_set": {
            "programming_languages": {
                "terms": ["c++", "java", "php"], //想要匹配的 terms list
                "minimum_should_match_field": "required_matches" //required_matches 是一个数字类型字段的名称
            }
        }
    }
}

GET /job-candidates/_search
{
    "query": {
        "terms_set": {
            "programming_languages": {
                "terms": ["c++", "java", "php"],
                "minimum_should_match_script": {// params.num_terms 是 terms的长度，这是个保留字
                   "source": "Math.min(params.num_terms, doc['required_matches'].value)"
                },
                "boost": 1.0
            }
        }
    }
}

// Boolean 实现
GET /job-candidates/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "term": {
            "programming_languages": {
              "value": "c++"
            }
          }
        },
        {
          "term": {
            "programming_languages": {
              "value": "java"
            }
          }
        },
        {
          "term": {
            "programming_languages": {
              "value": "php"
            }
          }
        }
      ],
      "minimum_should_match": 2
    }
  }

```

# Wildcard

Wildcard 通配符查询其实就是 term查询支持通配符的模式。

`*`通配符操作符支持匹配0或多个字符。

```json
GET /_search
{
    "query": {
        "wildcard": {
            "user": {
                "value": "ki*y",
                "boost": 1.0,
                "rewrite": "constant_score" //对于原生 Lucene 不支持的查询都需要有 rewrite 参数来决定最终评分，默认为 constant_score，1.0
            }
        }
    }
}
```

# Regexp

Wildcard 的加强版，支持正则表达式，一般来说，我们填个 value就完事了，其他的几个参数不需要去考虑，出于性能考虑， 尽量给正则加个头尾以缩小匹配范围。

```json
GET /_search
{
    "query": {
        "regexp": {
            "user": {
                "value": "k.*y",
                "flags" : "ALL",
                "max_determinized_states": 10000,
                "rewrite": "constant_score" //对于原生 Lucene 不支持的查询都需要有 rewrite 参数来决定最终评分，默认为 constant_score，1.0
            }
        }
    }
}
```

# Prefix

不需多言，前缀匹配，用 `Wildcard` 和`regexp`都能实现。

```json
GET /_search
{
    "query": {
        "prefix": {
            "user": {
                "value": "ki"
            }
        }
    }
}
```

# Range

基于范围的查询，适用于**数字类型，Range, Date** 等类型的字段查询。

```json
// 数字类型查询
GET _search
{
    "query": {
        "range" : {
            "age" : {// 字段名称
                "gte" : 10,
                "lte" : 20,
                "boost" : 2.0
            }
        }
    }
}

// Range query
GET _search
{
    "query": {
        "range" : {
            "age" : {// 字段名称
                "gte" : 10,
                "lte" : 20,
                "boost" : 2.0,
                "relation": "INTERSECTS" // INTERSECTS,CONTAINS,WITHIN 对于 Range 类型有三种关系，可选默认相交即可
            }
        }
    }
}

//  实践查询，支持 now
GET _search
{
    "query": {
        "range" : {
            "test_date" : {
                "gte" : "2020-01-10",
                "lt" :  "now/d"
            }
        }
    }
}
```

* **gt**:  (Optional) Greater than.
* **gte**: (Optional) Greater than or equal to.
* **lt**: (Optional) Less than.
* **lte**:  (Optional) Less than or equal to.

# Fuzzy

基于 Term 的模糊查询，如下例子中针对 `doz`进行查询，成功匹配了term ` dog`。

该 查询具有很多参数，本文不予展开。

```json
GET /zibu_rack1/_search
{
    "query": {
        "fuzzy": {
            "content.keyword": {
                "value": "doz"
            }
        }
    }
}

{
  "_index" : "zibu_rack1",
  "_type" : "_doc",
  "_id" : "G7r493QBrJCJPv-xLRl0",
  "_score" : 0.8026485,
  "_source" : {
    "content" : "dog"
  }
}
```



# Exists

非空查询 ，指定一个字段，返回所有该字段有值的文档。这里需要注意的是，只有 `null`和`[]`才会被认定为空，而`""`空字符串并不会被认定为空。

```json
// 该查询回过滤掉所有 user 字段有值的文档
GET /_search
{
    "query": {
        "bool": {
            "must_not": {
                "exists": {
                    "field": "user"
                }
            }
        }
    }
}
```



# IDs

基于 ID 进行查询，没懂这个查询有啥用，直接 GET 查询携带 ID 就可以。

```json
GET /_search
{
    "query": {
        "ids" : {
            "values" : ["1", "4", "100"]
        }
    }
}

GET /zibu_rack1/_doc/Kbq9AXUBrJCJPv-xFBnC
```