---
layout: post
title: ElasticSearch - function_score 系列文章（三）weight 具体实例
date: 2021-08-26 15:48:53
tags:
---

<p>

<!-- more -->

{% note info no-icon %}
博客地址：[isee.wang](/)
{% endnote %}

## 系列文章

1. [ElasticSearch - function_score 系列文章（一）简介](/ElasticSearch-function-score-系列文章（一）简介)
2. [ElasticSearch - function_score 系列文章（二）field_value_factor 具体实例](/ElasticSearch-function-score-系列文章（二）field-value-factor-具体实例)
3. ElasticSearch - function_score 系列文章（三）weight 具体实例
4. [ElasticSearch - function_score 系列文章（四）衰减函数 linear、exp、gauss 具体实例](/ElasticSearch-function-score-系列文章（四）衰减函数-linear、exp、gauss-具体实例)

## weight 具体实例

### 1. 先准备数据和索引，在 ES 插入三笔数据，其中 language 是 keyword 类型，like 是 integer 类型(代表点赞量)

```text
{ "language": "java", "like": 5 }
{ "language": "python", "like": 5 }
{ "language": "go", "like": 10 }
```

### 2. functions是一个数组，里面放着的是将要被使用的加强函数列表，我们在里面使用了3个filter去过滤数据，并且每个filter都设置了一个加强函数，并且还使用了一个会应用到所有文档的field_value_factor加强函数

- 可以为列表里的每个加强函数都指定一个filter，这样做的话，只有在文档满足此filter的要求，此filter的加强函数才会应用到文挡上，也可以不指定filter，这样的话此加强函数就会应用到全部的文挡上
- 一个文档可以一次满足多条加强函数和多个filter，如果一次满足多个，那麽就会产生多个加强score
- 因此ES会先使用score_mode定义的方式来合并这些加强score们，得到一个总加强score，得到总加强score之后，才会再使用boost_mode定义的方式去和old_score做合并
```text
GET 127.0.0.1/mytest/_doc/_search
{
    "query": {
        "function_score": {
            "query": {
                "match_all": {}  //match_all查出来的所有文档的_score都是1
            },
            "functions": [
                //第一个filter(使用weight加强函数)，如果language是java，加强score就是2
                {
                    "filter": {
                        "term": {
                            "language": "java"
                        }
                    },
                    "weight": 2
                },
                //第二个filter(使用weight加强函数)，如果language是go，加强score就是3
                {
                    "filter": {
                        "term": {
                            "language": "go"
                        }
                    },
                    "weight": 3
                },
                //第三个filter(使用weight加强函数)，如果like数大于等于10，加强score就是5
                {
                    "filter": {
                        "range": {
                            "like": {
                                "gte": 10
                            }
                        }
                    },
                    "weight": 5
                },
                //field_value_factor加强函数，会应用到所有文档上，加强score就是like值
                {
                    "field_value_factor": {
                        "field": "like"
                    }
                }
            ],
            "score_mode": "multiply", //设置functions里面的加强score们怎麽合并成一个总加强score
            "boost_mode": "multiply" //设置old_score怎麽和总加强score合并
        }
    }
}
```
```json
"hits": [
    {
        "_score": 150, //go同时满足filter2、filter3，且还有一个加强函数field_value_factor产生的加强，因此加强score为3, 5, 10，总加强score为3*5*10=150
        "_source": { "language": "go", "like": 10 }
    },
    {
        "_score": 10, //java只满足filter1，但是因为还有field_value_facotr产生的加强score，因此加强score为2, 5，总加强score为2*5=10
        "_source": { "language": "java", "like": 5 }
    },
    {
        "_score": 5, //python不满足任何filter，因此加强score只有field_value_factor的like值，就是5
        "_source": { "language": "python", "like": 5 }
    }
]
```

### 3. 其实weight加强函数也是可以不和filter搭配，自己单独使用的，只是这样做没啥意义，因为只是会给全部的文档都增加一个固定值而已

- 不过就DSL语法上来说，他也像其他加强函数一样，是可以直接使用而不用加filter的
```text
GET 127.0.0.1/mytest/_doc/_search
{
    "query": {
        "function_score": {
            "query": {
                "match_all": {}
            },
            "functions": [
                {
                    "weight": 3
                }
            ]
        }
    }
}
```
```json
"hits": [
    {
        "_score": 3,
        "_source": { "language": "go", "like": 10 }
    },
    {
        "_score": 3,
        "_source": { "language": "python", "like": 5 }
    },
    {
        "_score": 3,
        "_source": { "language": "java", "like": 5 }
    }
]
```

### 4. weight加强函数也可以用来调整每个语句的贡献度，权重weight的默认值是1.0，当设置了weight，这个weight值会先和自己那个{}里的每个句子的评分相乘，之后再通过score_mode和其他加强函数合并

- 下面的查询，公式为new_score = old_score * [ (like值 * weight1) + weight2 ]
- 公式解析 : weight1先加强like值(只能使用乘法)，接着再透过score_mode定义的方法(sum)和另一个加强函数weight2合并，得到一个总加强score，最后再使用boost_mode定义的方法(默认是multiply)和old_score做合并，得到new_score
```text
GET 127.0.0.1/mytest/_doc/_search
{
    "query": {
        "function_score": {
            "query": {
                "match_all": {}
            },
            "functions": [
                {
                    "field_value_factor": {
                        "field": "like"
                    },
                    "weight": 3
                },
                {
                    "weight": 20
                }
            ],
            "score_mode": "sum"
        }
    }
}
```
```json
"hits": [
    {
        "_score": 50,
        "_source": { "language": "go", "like": 10 }
    },
    {
        "_score": 35,
        "_source": { "language": "python", "like": 5 }
    },
    {
        "_score": 35,
        "_source": { "language": "java", "like": 5 }
    }
]
```
