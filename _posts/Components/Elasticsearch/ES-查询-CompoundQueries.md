---
title: ES-查询-CompoundQueries
date: 2020-10-03 18:10:55
tags: [Elasticsearch]
---

* `bool` query: 具有以下四种模式，任意搭配，must和should子句叠加计算出最终文档评分 
	* must: 必须匹配
	* should: 当不存在must和filter时，必须匹配should查询子句中的一条。否则可以不匹配。 可以配置Should子句最小匹配条数。
	* filter: 必须匹配，且不纳入计算相关性评分
	* must_not: 必须不匹配，且不纳入计算相关性评分
* `boosting` query: **该查询可以用于当你不希望排除某些文档，却想降低他们的评分时**。设置positive和negative查询，返回所有匹配positive查询的文档。 当文档同时匹配negative查询时，将评分乘以negative_boost.
* `constant_score` query: **给予文档固定评分**。查询文档，并设置所有匹配文档的评分为设定好的`boost`值。
* `dis_max` query： **类似 Boolean_should, 匹配查询越多，得分越高**。使用多个查询语句查询，当匹配多条查询时，计算相关性评分为=最高分+SUM（其他评分*tie_breaker）
* `function_score` query: **自定义评分计算方式**。 设置多个filter函数，将其评分加权计算，最后根据设置好的方式汇总。 

<!--more-->

# Boolean

只有 `must` 和` should` 子句中的查询才会计算评分，且最终评分为所有评分之和。

| Occur      | Description                                                  |
| ---------- | ------------------------------------------------------------ |
| `must`     | 文档**必须匹配**must查询，**且计算得分**                     |
| `filter`   | 文档**必须匹配** filter查询，**但不计算得分**。由于 filter查询 |
| `should`   | 当不存在must和filter时，必须匹配should查询子句中的一条，否则可以不匹配。<br />`minimum_should_match`用于限制最小 should 子句匹配数量。 |
| `must_not` | 文档**必须不匹配** must_not 查询，**且不计算得分**。同样属于[filter context](https://www.elastic.co/guide/en/elasticsearch/reference/7.2/query-filter-context.html),，意味着没有评分且查询会被缓存 |

```json
POST _search
{
  "query": {
    "bool" : {
      "must" : {
        "term" : { "user" : "kimchy" }
      },
      "filter": {
        "term" : { "tag" : "tech" }
      },
      "must_not" : {
        "range" : {
          "age" : { "gte" : 10, "lte" : 20 }
        }
      },
      "should" : [
        { "term" : { "tag" : "wow" } },
        { "term" : { "tag" : "elasticsearch" } }
      ],
      "minimum_should_match" : 1,
      "boost" : 1.0
    }
  }
}
```

# Boosting

设置positive和negative查询，返回所有匹配positive查询的文档。 当文档同时匹配negative查询时，将评分乘以negative_boost.

```json
GET /_search
{
    "query": {
        "boosting" : {
            "positive" : {
                "term" : {
                    "text" : "apple"
                }
            },
            "negative" : {
                 "term" : {
                     "text" : "pie tart fruit crumble tree"
                }
            },
            "negative_boost" : 0.5 
        }
    }
}
```

# Constant Score

包裹一个 filter query，所有匹配的文档的相关性评分等同于 `boost`的值。

```json
GET /_search
{
    "query": {
        "constant_score" : {
            "filter" : {
                "term" : { "user" : "kimchy"}
            },
            "boost" : 1.2
        }
    }
}
```



# Disjunction max

**返回匹配至少一个查询语句的文档**，且**综合评分=查询语句中的最高分+SUM(tiebreaker*其他查询语句评分)**

tie_breaker值为在[0,1]的区间内的浮点数，**默认为0.0**

```json
GET /_search
{
    "query": {
        "dis_max" : {
            "queries" : [
                { "term" : { "title" : "Quick pets" }},
                { "term" : { "body" : "Quick pets" }}
            ],
            "tie_breaker" : 0.7
        }
    }
}
```



# Function score query

`fuction_score`查询提供了**自定义文档相关性评分**的一种方式。用户定义**一个 Query** 和**多个 filter functions**，来为每个文档计算出一个最终得分。
**工作机制：** 

1. **Query:** 一个query来查询文档，并为每个文档计算得分，若设置了 `boost`参数，则将得分*boost 值。
2. **Filter:** 对所有文档执行filter functions，若匹配，则根据所设置`function_score`函数计算得分。最后根据`score_mode`综合计算出最终得分。
   1. 举例说明：当一个文档匹配三个 filter functions，且得分分别为1，2，3时，默认`score_mode为 mutiply`,则最终得分为3分。
   2. `weight`: 可以使用 weight 函数，为当前 filter function设置评分，如果`score_mode为avg`，则 weight 值只作为评分比例使用。
   3. **如果文档不匹配所有 filter functions，则该文档的 filter moudle 评分为1.**
3. **综合计算：**最后基于`boost_mode`计算出query和filter的综合评分。

```json
GET /_search
{
    "query": {
        "function_score": {
          "query": { "match_all": {} },// query模块
          "boost": "5", // query 模块的打分*boost=query 模块的最终得分
          "functions": [ // 过滤函数模块，每个过滤函数基于function score(wight,random_score 等)计算评分
              {
                  "filter": { "match": { "test": "bar" } },
                  "random_score": {}, 
                  "weight": 23 //当 score_mode为avg,weight作为评分比例使用
              },
              {
                  "filter": { "match": { "test": "cat" } },
                  "weight": 42 // 当 score_mode不为avg时，最终得分为filter得分(默认1)*weight
              }
          ],
          "score_mode": "max", //决定了如何计算filter functions的最终得分,默认为multiply
          "boost_mode": "multiply", //决定了如何综合计算query function和filter function的最终得分，默认为multiply
          "max_boost": 42, // filter function 模块最大评分限制
          "min_score" : 42 // 所有相关性评分低于min_score的文档会被过滤
        }
    }
}
```


## score_mode

score_mode决定了如何计算**filter functions**的最终得分

* **multiply (default)**： scores are multiplied
* sum： scores are summed
* **avg**: scores are averaged, 过滤函数中的weight参数，只有在`score_mode`为avg时才生效。**SUM(score*weight)/SUM(weight)**
* first： the first function that has a matching filter is applied
* max： maximum score is used
* min： minimum score is used

## boost_mode

boost_mode决定了如何综合计算**query function和filter functions**的最终得分

* **multiply(default)**: query score and function score is multiplied 
* replace: only function score is used, the query score is ignored
* sum: query score and function score are added
* avg: average
* max: max of query score and function score
* min: min of query score and function score

## function_score: 自定义各个查询过滤函数的评分

### **script_score**

使用脚本表达式基于文档中的数字变量计算出新的得分，来代替原有的文档评分。新的得分不能为负，否则会报错。

`script_score` 只能用于计算 query函数的评分，且不与 filter functions 模块兼容。

```json
GET /_search
{
    "query": {
        "function_score": {
            "query": {
                "match": { "message": "elasticsearch" }
            },
            "script_score" : {
                "script" : {
                    "params": {
                        "a": 5,
                        "b": 1.2
                    },
                    "source": "params.a / Math.pow(params.b, doc['likes'].value)"
                }
            }
        }
    }
}
```

### Field value factor

类似于 `script_socre`的简单操作 版本，提供了以下几个参数，按照**modifier(facotr* doc['field'].value)**的公式来计算得分

* Field: 参数名称
* factor: 倍数因子
* modifier: `none`, `log`, `log1p`, `log2p`, `ln`, `ln1p`, `ln2p`, `square`, `sqrt`, or `reciprocal`. Defaults to `none`.

```json
GET /_search
{
    "query": {
        "function_score": {
            "field_value_factor": {//modifier(facotr* doc['field'].value)
                "field": "likes",
                "factor": 1.2,
                "modifier": "sqrt",
                "missing": 1 // 如果参数不存在则使用 missing 的值作为评分
            }
        }
    }
}
```

### weight

filter function的最终得分=  得分(默认1分)* weight

### Random

```json
// 返回[0,1)的浮点数
// 可以指定某个参数作为种子值
random: {}
```

### Decay functions

Decay函数非常类似于`range query`, 但是range query有明确的边界线，一旦超过就不会被匹配，而decay function则是**基于目标值与原始值之间的距离来调整文档的评分**。
Decay中文翻译为腐蚀，衰变。 我觉得衰变可以很好理解该函数的定义，距离中心越远，衰变的越厉害。
从下面的例子，来快速了解Decay functions：

```json
// Deacy Function的模板，以geo-point为例
// 由于没有设置offset值，那么在origin点的文档，评分不变。
// 在origin点距离2km处，评分削减为原值的0.33倍。
// 在距离小于2KM内处，根据所选的衰变算法，计算衰变的比例，介于（0.33-1）之间。
// 在距离大于2KM内处，根据所选的衰变算法，计算衰变的比例，介于[0-0.33）之间。
"DECAY_FUNCTION": { // 支持线性衰变，指数衰变和高斯衰变，下文会具体展开。
    "FIELD_NAME": { // 目前只支持 对number, date, geo-point三种类型
          "origin": "11, 12", // origin: 定义数字的中心区域
          "scale": "2km",     // scale: 定义了衰变的速度。
          "offset": "0km",
          "decay": 0.33
    }
}
// 对于date类型的decay function
// 在[2013-09-12，2013-09-22]之间，文档评分不变。
// 在[2013-09-07，2013-09-12）,(2013-09-22，2013-09-27]之间，文档评分根据所选衰变函数，乘以[0.5,1)的衰变值。
// 距离更大的文档，文档评分根据所选衰变函数，乘以[0,0.5)的衰变值。
GET /_search
{
    "query": {
        "function_score": {
            "gauss": {
                "date": {
                      "origin": "2013-09-17", 
                      "scale": "10d",
                      "offset": "5d", 
                      "decay" : 0.5 
                }
            }
        }
    }
}
```

#### **核心参数**： 

* **origin(Required)**: 中心点，用于计算其他参与和它的距离。对于Date类型可以使用ES的时间表达式，如`now-1h`
* **scale(Required)**: 文档离中心点的距离等于 scale 的值时，此时评分=原评分*decay。
* **offset**(Optional): 默认为0，decay函数只会应用于距离大于offset的文档。offset参数将中心点扩大到了中心区域，只有出了中心区域的文档，才会应用decay function.
* **decay**(Optional): 默认为0.5，定义了根据scale给定的距离如何给文档打分。

#### **支持的dacay fucntions**

* **gauss(高斯)：** 高斯函数介于指数和线性之间，对于不在 scale 区间的文档，评分比例曲线归0。
* **exp(指数)**：指数函数不会归0，始终提供了一个相对较低的评分比例。**个人觉得指数函数最适合普遍地址搜索的场景。**
* **linear(线性)**：对于不在 scale 区间的文档，评分比例线性归0。

![](./ES-查询-CompoundQueries/decay_2d.png)

> Note: Decay Function 还有很多细节的配置和具体计算评分的公式，需要使用时可以继续深入了解下。
>
> https://www.elastic.co/guide/en/elasticsearch/reference/7.2/query-dsl-function-score-query.html#function-field-value-factor