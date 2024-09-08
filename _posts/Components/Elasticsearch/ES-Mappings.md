---
title: ES-Mappings
date: 2020-09-09 23:06:01
tags: [Elasticsearch]
---

涉及知识点：

*  Template Mapping所支持Type
* 对于不同的查询需求，设置什么 Type

<!--more-->

> **Mapping的优先级**：在创建索引时，会按照 template 的优先级，由低到高覆盖性的合成的 Mapping。



## Dynamic Mapping

，会自动应用所有匹配的template,按照template的优先级，由低到高进行覆盖式的合成，最后应用到索引上。
但当没有匹配到任何templates时，索引会自动生成对应mapping,这就Dynamic Mapping。

动态Mapping存在着不足，比如当一个字段被映射为Integer类型后，相同字段值为String类型的文档将无法插入索引。

## 手动创建Mapping

由于Dynamic Mapping的不可控，我们往往需要手动根据字段的类型和需求来创建Mapping。

>Tips: 当字段类型较多时，可以先动态生成，再删除数据后，更新mapping后再重新导入数据。

## 字段类型

### 字符串类型

* **text:** 支持分词，用于全文搜索
* **keyword:** 不支持分词，用于聚合和排序

如果安装了分词器，我们可以为text类型指定IK分词器。 

```json
# 模糊搜索+精确匹配
"contents": {
	"type": "text",
	"fields": {
		"field": {
			"type": "keyword"
		}
	},
	"analyzer": "ik_max_word",
	"fielddata": true
}

# 模糊搜索
"contents": {
	"type": "text",
	"analyzer": "ik_max_word"
}

# 精确匹配
"contents": {
	"type": "keyword"
}

# 不需要索引
"contents": {
	"type": "keyword"
	"index": false
}

```

#### text

**工作机制：** [text](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/text.html)类型用于索引 `full-text`字段，比如诗歌，产品描述,邮件正文等。该类型字符串会被`analyzer`分词为多个terms,然后建立反向索引。
text类型字段允许在全文中，进行分词查询。但不用于排序，基本也不用于聚合（Aggregation）.
**适用场景：** 希望对内容进行分词查询，全文搜索的场景。

```json
# fielddata: true. 该参数默认为false,当设置为true时，将在内存中使用fieldata结构，用于支持对`text`类型的排序，聚合或scripting。
# 激活fielddata， 会消耗大量内存，请根据情况使用。
PUT my_index
{
  "mappings": {
    "_doc": {
      "properties": {
        "full_name": {
          "type":  "text",
		  "analyzer": "ik_max_word",
		  "fielddata"： true
        }
      }
    }
  }
}
```

#### Keyword

[Keyword](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/keyword.html) 类型用于索引结构化的数据，比如邮件地址，主机名，状态 和标签等。
这些类型的数据往往用于过滤（过滤所有状态为不可用的记录）,用于排序 或者 用于聚合。**Keyword类型只有与查询值完全一致时，才会被匹配。**

```json
PUT my_index
{
  "mappings": {
    "_doc": {
      "properties": {
        "tags": {
          "type":  "keyword"
        }
      }
    }
  }
}
```


#### Keyword + Field

很多情况下，我们需要一个字段同时支持全文查找和聚合排序等功能，就需要设置该字段类型为 `text` 和 `Keyword`。
```json
PUT my_index
{
  "mappings": {
    "_doc": {
      "properties": {
        "city": {
          "type": "text",
          "fields": {
            "field": { 
              "type":  "keyword"
            }
          }
        }
      }
    }
  }
}
```

### 数字类型

* **long**: 带符号的64位整数
* **integer**: 带符号的32位整数
* **short**: 带符号的16位整数 [-32768，32767]
* **byte**: 带符号的8位整数， [-128，127]
* **double**: 双精度64位浮点数
* **float**: 单精度32位浮点数
* **half_float**: 半精度16位浮点数
* **scaled_float**: 缩放类型的浮点数，需配合缩放因子（scaling_factor）一起使用

对于整数类型而言（long, integer, short， byte），从搜索查询的角度而言，尽可能选择小的能提高查询效率。
对于浮点数而言（double, float, half_float, scaled_float），`-0.0`和`+0.0`是两个不同的数值，这点需要注意，在范围查询时选择两者中范围更大的作为边界。
其中scaled_float，比如价格只需要精确到分，那么缩放因子的值就是100, **在选择浮点数时，如果小数位明确，则优先使用`scaled_float`**。

```json
"mappings": {
	"properties": {
	   "status": {
	      "type": "byte"
	   },
	   "year": {
	      "type": "short"
	   },
	   "id": {
	      "type": "long"
	   },
	   "price": {
	      "type": "scaled_float",
		  "scaling_factor": 100
	   },
	}
}
```

### 日期类型 date

ES支持以下三种日期格式：
* 格式化的日期字符串
* 一个13位long类型表示的毫秒时间戳
* 一个integer类型表示的10位普通时间戳

在ES内部，日期类型会被转化为UTC时间（如果指定了时区）并存储为long类型表示的毫秒时间戳。
值得注意的是在Kibana上的Discovery，它会根据用户所处的时区，来进行转换，查询。

```json
"mappings": {
	"properties": {
	   "postdate": {
	      "type": "date"
	   }
	}
}
```

### 布尔类型 boolean

### 二进制类型 binary 

### 范围类型 Range

### 数组类型 Arrays

在 ES 中,没有单独 `array`的数据类型。任何 field 类型都支持[0, N]个数据，但是这些数据的类型应该相同

> Tips: 除了 Object 类型，Object 数组也可以被插入，但是无法对单个元素匹配查询。

- an array of strings: [ `"one"`, `"two"` ]
- an array of integers: [ `1`, `2` ]
- an array of arrays: [ `1`, [ `2`, `3` ]] which is the equivalent of [ `1`, `2`, `3` ]
- an array of objects: [ `{ "name": "Mary", "age": 12 }`, `{ "name": "John", "age": 10 }`]

### Object 数组 Nested

默认的数组不支持 Object 元素，需要 `Nested` 来存储 Object 数组，这样数组中每一个 Object 都可以单独匹配查询。

> 对于 Object 元素，使用 `Flattened`是一种更为开销更低的方式。具体内容略




## Reference
https://segmentfault.com/a/1190000017215813?utm_source=sf-related