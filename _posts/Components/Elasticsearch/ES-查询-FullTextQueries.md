---
title: ES-查询-FullTextQueries
date: 2020-10-05 15:50:50
tags: [Elasticsearch]
---

当你需要做全文查询时，如果你对查询的terms顺序没有要求，则使用` match query`，如果对顺序有要求，则使用 `match_phrase query` 。如果你对你需要更细粒度的查询则可以考虑 `query_string, intervals`。

* `intervals`: 可以对匹配项的顺序和之间的距离进行的细粒度的匹配。
* `match`: 对输入进行分词，默认情况只要文档字段**匹配任意一个Term**即可。（可设置为Terms全匹配,即词组匹配，match_phrase）。 支持模糊，同义词，最小匹配terms等查询。
* `match_bool_prefix`: Analyzer对输入的参数进行分词，最后一个分词会变为`prefix`查询，其他分词将会变为`term`查询。所有这些查询会以`Boolean.should`进行连接。
* `match_phrase`: 对分词后的Terms进行**顺序和内容的全匹配**，即只有目标字段同时包含所有分词，分词顺序按序且分词间隔在规定范围内（默认为0，分词需要紧挨，参数slop调整）才行。
* `match_phrase_prefix`：  基本等同于match_phrase, 只是最后一个Terms只需要前缀匹配即可。
* `multi_match`: 对多个字段使用`match query`,字段名称可以使用通配符。 至于使用哪种 Match Query，取决于Type（`best_fields`,`most_fields`等）设置。
* `common`: 暂时没有深入了解，相对专业化，与`stopwords`相关。
* `query_string`: 支持简洁的Lucene查询字符串语法，允许您在单个查询字符串中指定AND | OR | NOT条件和多字段搜索。 仅限于专业用户。
* `simple_query_string`: 一个相对简化的`query_string`查询。

<!--more-->

# match

match查询对输入的String进行分词查询，默认情况下`operator为 or`，因此文档具有任意一个 term就算匹配。

以下参数相对重要且高频：

* **operator**: **or(default) | and**, and 表示需要匹配所有 terms, or 表示只要匹配任意一个 term 即可。
* **analyzer**：指定查询所使用的分词器，当不声明时将按照 字段分析器，index 默认分析器，standard 分析器的顺序向下匹配。
* **minimum_should_match**:  默认情况下，operator 为 or，那么实际上match 查询会被转为 boolean should查询，该参数就可以用于控制 should子句匹配的数量。

```json
GET /_search
{
    "query": {
        "match" : {
            "message" : {
                "query" : "this is a test"
            }
        }
    }
}
```



# match_bool_prefix

`match query` 的本质实际上就是将输入文本进行分词，将每一个词作为`term query`的输入，以 `boolean query`的 should连接组合在一起。

而 `match_bool_prefix` 则是将最后一个term 作为 `prefix query`的输入。

所有的参数同 `match query` 一致。

```json
GET /_search
{
    "query": {
        "match_bool_prefix" : {
            "message" : "quick brown f"
        }
    }
}

GET /_search
{
    "query": {
        "bool" : {
            "should": [
                { "term": { "message": "quick" }},
                { "term": { "message": "brown" }},
                { "prefix": { "message": "f"}}
            ]
        }
    }
}
```



# match_phrase

短语匹配，相较于` match query` 额外增加了对 terms顺序的要求，默认情况下要求文档字段中的 terms 顺序要与查询文本中 terms 顺序完全一致才能返回，但是可以通过修改 `slop`参数来调整最大允许的间隔 terms 数量。

举例说当未设置 slop 值时，该参数默认值为0，则 dog pig cat 不会被匹配，dog cat 可以被匹配。

当设置 slop为1，dog pig cat，dog cat 都能被匹配。

```json
GET /_search
{
    "query": {
        "match_phrase" : {
            "message" : "dog cat "
        }
    }
}
```



# match_phrase_prefix

同`match_phrase`，但是一个 term会以`prefix query`的形式去匹配。

> Note: 有可能出现 quick brown fox无法被以下查询所匹配的可能性，这是由于 ES 自动补充查询的机制造成的，一般情况下可以基于延长关键词来解决。

```json
GET /_search
{
    "query": {
        "match_phrase_prefix" : {
            "message" : {
                "query" : "quick brown f"
            }
        }
    }
}
```



# multi_match

其实这是一个很鸡肋的查询，它提供了对多个 filed 的查询和许多不同的 type 用于支持不同的场景，但是实际上我们当我们有相关的需求时，第一个会想到 boolean 查询，因此本小节略过，参考 `boolean query`

```json
GET /_search
{
  "query": {
    "multi_match" : {
      "query" : "this is a test",
      "fields" : [ "subject^3", "*.name" ] // ^3 表示 boost 值为3， 通配符*也支持
    }
  }
}
```