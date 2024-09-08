---
title: ES-查询-Highlight
date: 2020-10-06 15:26:05
tags: [Elasticsearch]
---

Elasticsearch 的高亮（highlight）可以从搜索结果中多个字段中**获取突出显示的摘要**，以便向用户显示查询匹配的位置。

响应结果的 highlight 字段中包括高亮的字段和高亮的片段。Elasticsearch 默认会用 `<em></em>` 标签标记关键字。

高亮操作需要获取文档filed的实际内容，因此当一个filed没有存储时(Mapping 中没有设置 `store`为 `true`), 真实的`_source`将会被加载，来获取相关 field的内容。

<!--more-->

ES 支持以下三种不同的 Highlighters, 通过在 Mapping 中设置 highlighter type来决定使用哪种类型(下文展开)

* `unified`(Default):
* `plain`: 会在内存中维护一定数据结构，因此不适合对大量文档和多个 fileds 进行高亮的场景。
* `fvh(fast vector highlighter)`:  不支持 Span Queries

```json
// A Basic highlight case
GET /_search
{
    "query" : {
        "match": { "content": "kimchy" }
    },
    "highlight" : {
        "fields" : {
            "content" : {}
        }
    }
}
```



# STH in common 

## 配置高亮标签

默认情况下，高亮文字会被 `<em></em>`包裹，可以通过以下设置来修改标签。

对于 fast vector highlighter而言，可以设置根据重要性来设置多个 tags。

```json
// Basic Case
GET /_search
{
    "query" : {
        "match": { "user": "kimchy" }
    },
    "highlight" : {
        "pre_tags" : ["<tag1>"],
        "post_tags" : ["</tag1>"],
        "fields" : {
            "body" : {}
        }
    }
}
// For FVH
GET /_search
{
    "query" : {
        "match": { "user": "kimchy" }
    },
    "highlight" : {
        "pre_tags" : ["<tag1>", "<tag2>"],
        "post_tags" : ["</tag1>", "</tag2>"],
        "fields" : {
            "body" : {}
        }
    }
}
```

也可以使用内嵌的 Styled tag  schema

```json
GET /_search
{
    "query" : {
        "match": { "user": "kimchy" }
    },
    "highlight" : {
        "tags_schema" : "styled",
        "fields" : {
            "comment" : {}
        }
    }
}

{
  "_index" : "zibu_rack1",
  "_type" : "_doc",
  "_id" : "9FbZynQBebG7Z5xLMRPO",
  "_score" : 0.18232156,
  "_source" : {
    "contentLength" : 11,
    "content" : "dog and cat"
  },
  "highlight" : {
    "content" : [
      """<em class="hlt1">dog</em> and cat"""
    ]
  }
}

Set to styled to use the built-in tag schema. The styled schema defines the following pre_tags and defines post_tags as </em>.

<em class="hlt1">, <em class="hlt2">, <em class="hlt3">,
<em class="hlt4">, <em class="hlt5">, <em class="hlt6">,
<em class="hlt7">, <em class="hlt8">, <em class="hlt9">,
<em class="hlt10">
```



## 设置高亮分段

每个高亮字段都可以控制其高亮片段(Highlighted fragment)的长度（默认是100）和数量（默认是5）。

可以将 `number_of_fragments`设置为0，从而不会不产生高亮片段而是返回整个文档字段 ，在部分场景如（文档标题和地址等场景）非常好用。

> PS: 上面是官方原文，蛮奇怪的，默认的 Fragment  Size 是100，举例的几个场景根本不会超过100。

```json
GET /zibu_rack1/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "term": {
            "content": "dog"
          }
        }
      ]
    }
  },
  "highlight": {
    "fields": {
      "content": {
        "fragment_size": 3,
        "number_of_fragments": 10
      }
    }
  }
}

"highlight" : {
  "content" : [
    "<em>dog</em>",
    "<em>dog</em>",
    "<em>dog</em>"
  ]
}
```

## 全局设置高亮参数

你可以全局设置高亮参数也可以在每个 field 中选择性的单独设置，field setting 会覆盖 global setting。

```json
GET /_search
{
    "query" : {
        "match": { "user": "kimchy" }
    },
    "highlight" : {
        "number_of_fragments" : 3,
        "fragment_size" : 150,
        "fields" : {
            "body" : { "pre_tags" : ["<em>"], "post_tags" : ["</em>"] },
            "blog.title" : { "number_of_fragments" : 0 },
            "blog.author" : { "number_of_fragments" : 0 },
            "blog.comment" : { "number_of_fragments" : 5, "order" : "score" }
        }
    }
}
```

## 高亮查询

在 Highlight 中我们可以指定其他查询语句，这些查询语句所匹配的信息同样也会被高亮，但是不会影响到整个查询的过程。

```json
GET /_search
{
    "stored_fields": [ "_id" ],
    "query" : {
        "match": {
            "comment": {
                "query": "foo bar"
            }
        }
    },
    "rescore": {
        "window_size": 50,
        "query": {
            "rescore_query" : {
                "match_phrase": {
                    "comment": {
                        "query": "foo bar",
                        "slop": 1
                    }
                }
            },
            "rescore_query_weight" : 10
        }
    },
    "highlight" : {
        "order" : "score",
        "fields" : {
            "comment" : {
                "fragment_size" : 150,
                "number_of_fragments" : 3,
                "highlight_query": {
                    "bool": {
                        "must": {
                            "match": {
                                "comment": {
                                    "query": "foo bar"
                                }
                            }
                        },
                        "should": {
                            "match_phrase": {
                                "comment": {
                                    "query": "foo bar",
                                    "slop": 1,
                                    "boost": 10.0
                                }
                            }
                        },
                        "minimum_should_match": 0
                    }
                }
            }
        }
    }
}
```



# Highlighters

## Unified Highlighter(Default)

The `unified` highlighter uses the Lucene Unified Highlighter. This highlighter breaks the text into sentences and uses the BM25 algorithm to score individual sentences as if they were documents in the corpus. It also supports accurate phrase and multi-term (fuzzy, prefix, regex) highlighting. This is the default highlighter.

```json
GET /_search
{
    "query" : {
        "match": { "user": "kimchy" }
    },
    "highlight" : {
        "fields" : {
            "comment" : {"type" : "unified"}
        }
    }
}
```

## Plain Highlighter

The `plain` highlighter uses the standard Lucene highlighter. It attempts to reflect the query matching logic in terms of understanding word importance and any word positioning criteria in phrase queries.

The `plain` highlighter works best for highlighting simple query matches in a single field. To accurately reflect query logic, it creates **a tiny in-memory index** and re-runs the original query criteria through Lucene’s query execution planner to get access to low-level match information for the current document. This is repeated for every field and every document that needs to be highlighted. If you want to highlight a lot of fields in a lot of documents with complex queries, we recommend using the `unified` highlighter on `postings` or `term_vector` fields.

```json
GET /_search
{
    "query" : {
        "match": { "user": "kimchy" }
    },
    "highlight" : {
        "fields" : {
            "comment" : {"type" : "plain"}
        }
    }
}
```



## Fast Vector Highlighter

The `fvh` highlighter uses the Lucene Fast Vector highlighter. This highlighter can be used on fields with `term_vector` set to `with_positions_offsets` in the mapping. The fast vector highlighter:

- Can be customized with a [`boundary_scanner`](https://www.elastic.co/guide/en/elasticsearch/reference/7.2/search-request-highlighting.html#boundary-scanners).
- Requires setting `term_vector` to `with_positions_offsets` which increases the size of the index
- Can combine matches from multiple fields into one result. See `matched_fields`
- Can assign different weights to matches at different positions allowing for things like phrase matches being sorted above term matches when highlighting a Boosting Query that boosts phrase matches over term matches

The `fvh` highlighter does not support span queries. If you need support for span queries, try an alternative highlighter, such as the `unified` highlighter.

```json
GET /_search
{
    "query" : {
        "match": { "user": "kimchy" }
    },
    "highlight" : {
        "fields" : {
            "comment" : {"type" : "fvh"}
        }
    }
}
```

