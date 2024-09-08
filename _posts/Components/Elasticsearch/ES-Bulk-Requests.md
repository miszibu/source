---
title: ES-Bulk-Requests
date: 2020-07-20 23:31:36
tags: [Elasticsearch]
---

ES Bulk 请求是使用一个请求来批量执行多个操作的一种方法，可以显著的增加索引数据的速度。

<!--more-->

## NDJSON - Bulk 请求的格式

> Bulk请求的格式： MetaData Json + Source Json

这里需要注意的是，请求的Body并不是一个完整的Json，而是由换行符分开的多个Json，这种格式被称为`NDJSON`.

因此，在发送bulk请求时需要将`Content-Type`设置为`application/x-ndjson`

```
action_and_meta_data\n
optional_source\n
action_and_meta_data\n
optional_source\n
....
action_and_meta_data\n
optional_source\n
```

## CRUD - 支持的请求类型

* **index/create**: 需要下一行携带source, 并且与标准的API 共享相同的参数。Create请求会失败，当对应的索引和类型已存在时，Index则会替换和新增对应的文档。
* **delete**: 不需要下一行携带source，用法与标准一致。
* **update**: 需要部分文档，upsert，脚本和参数都可以在下一行中指定。

当使用curl请求时，需要使用 --data-binary参数和不是普通的  -d 参数， 后者会忽略换行符。

### 示例Request Body：

第一步是将要请求的Body写入一个文件中去，注意请求是以换行符来界定Json的，需要使用压缩JSON的格式。

```json
{ "index": { "_id": 1 }}{ "title": "Elasticsearch: The Definitive Guide", "authors": ["clinton gormley", "zachary tong"], "summary" : "A distibuted real-time search and analytics engine", "publish_date" : "2015-02-07", "num_reviews": 20, "publisher": "oreilly" }
{ "index": { "_id": 2 }}
{ "title": "Taming Text: How to Find, Organize, and Manipulate It", "authors": ["grant ingersoll", "thomas morton", "drew farris"], "summary" : "organize text using approaches such as full-text search, proper name recognition, clustering, tagging, information extraction, and summarization", "publish_date" : "2013-01-24", "num_reviews": 12, "publisher": "manning" }
{ "index": { "_id": 3 }}
{ "title": "Elasticsearch in Action", "authors": ["radu gheorge", "matthew lee hinman", "roy russo"], "summary" : "build scalable search applications using Elasticsearch without having to do complex low-level programming or understand advanced data science algorithms", "publish_date" : "2015-12-03", "num_reviews": 18, "publisher": "manning" }
{ "index": { "_id": 4 }}
{ "title": "Solr in Action", "authors": ["trey grainger", "timothy potter"], "summary" : "Comprehensive guide to implementing a scalable search engine using Apache Solr", "publish_date" : "2014-04-05", "num_reviews": 23, "publisher": "manning" }`
```

### Curl 请求

> curl -s -H "Content-Type: application/x-ndjson" -XPOST http://<hostname>:<port>/bookdb_index/_bulk --data-binary "@books" 

请求的URI支持 `/_bulk`, `/{index}/_bulk`. 当我们在URI中指定了index时，它们会被默认设置在批量请求的每个请求头的meta-data中。

**Bulk请求携带的消息数量：**没有一个最佳值，这个最佳值需要由用户不断的benchmark 测试来获得。

**不要使用Http的Chunks功能**：如果使用Http请求，需要避免使用Chunks 分块传输，这会一定程度降低效率。
Chunked Request: 将请求体裁剪为多个包进行传输的一种方式，常用于大文件传输。默认情况下，请求体的大小被存放在Content-Length中。

**请求的返回**：

Bulk请求的返回则是一个由每个独立请求的返回体组成的一个大的JSON。 一**个请求执行失败，并不会影响到后续请求的执行。**

```json
{
    "took": 54,
    "errors": false,
    "items": [
        {
            "index": {
                "_index": "bookdb_index",
                "_type": "book",
                "_id": "1",
                "_version": 2,
                "result": "updated",
                "_shards": {
                    "total": 2,
                    "successful": 2,
                    "failed": 0
                },
                "_seq_no": 1,
                "_primary_term": 1,
                "status": 200
            }
        },
        {
            "index": {
                "_index": "bookdb_index",
                "_type": "book",
                "_id": "2",
                "_version": 1,
                "result": "created",
                "_shards": {
                    "total": 2,
                    "successful": 2,
                    "failed": 0
                },
                "_seq_no": 2,
                "_primary_term": 1,
                "status": 201
            }
        },
        {
            "index": {
                "_index": "bookdb_index",
                "_type": "book",
                "_id": "3",
                "_version": 1,
                "result": "created",
                "_shards": {
                    "total": 2,
                    "successful": 2,
                    "failed": 0
                },
                "_seq_no": 3,
                "_primary_term": 1,
                "status": 201
            }
        },
        {
            "index": {
                "_index": "bookdb_index",
                "_type": "book",
                "_id": "4",
                "_version": 1,
                "result": "created",
                "_shards": {
                    "total": 2,
                    "successful": 2,
                    "failed": 0
                },
                "_seq_no": 4,
                "_primary_term": 1,
                "status": 201
            }
        }
    ]
}
```