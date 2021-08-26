---
layout: post
title: ElasticSearch - function_score 系列文章（二）field_value_factor 具体实例
date: 2021-08-26 15:48:33
tags:
---

<p>

<!-- more -->

{% note info no-icon %}
博客地址：[isee.wang](/)
{% endnote %}

## 系列文章

1. [ElasticSearch - function_score 系列文章（一）简介](/ElasticSearch-function-score-系列文章（一）简介)
2. ElasticSearch - function_score 系列文章（二）field_value_factor 具体实例
3. [ElasticSearch - function_score 系列文章（三）weight 具体实例](/ElasticSearch-function-score-系列文章（三）weight-具体实例)
4. [ElasticSearch - function_score 系列文章（四）衰减函数 linear、exp、gauss 具体实例](/ElasticSearch-function-score-系列文章（四）衰减函数-linear、exp、gauss-具体实例)

## field_value_factor 具体实例

### 1. 首先准备数据和索引，在 ES 插入三笔数据，其中 title 是 text 类型，like 是 integer 类型(代表点赞量)

```text
{ "title": "ES 入门", "like": 2 }
{ "title": "ES 进阶", "like": 5 }
{ "title": "ES 最高难度", "like": 10 }
```

### 2. 先使用一般的query，查看普通的查询的评分会是如何

```text
GET 127.0.0.1/mytest/_doc/_search
{
    "query": {
        "match": {
            "title": "ES"
        }
    }
}
```
```json
"hits": [
    {
        "_score": 0.2876821,
        "_source": { "title": "ES 入门", "like": 2 }
    },
    {
        "_score": 0.20309238,
        "_source": { "title": "ES 进阶", "like": 5 }
    },
    {
        "_score": 0.16540512,
        "_source": { "title": "ES 最高难度", "like": 10 }
    }
]
```

### 3. 使用 function_score 的 field_value_factor 改变 _score，将 old_score 乘上 like 的值

```text
GET 127.0.0.1/mytest/_doc/_search
{
    "query": {
        "function_score": {
            "query": {
                "match": {
                    "title": "ES"
                }
            },
            "field_value_factor": {
                "field": "like"
            }
        }
    }
}
```
```json
"hits": [
    {
        "_score": 1.6540513, //原本是0.16540512
        "_source": { "title": "ES 最高难度", "like": 10 }
    },
    {
        "_score": 1.0154619, //原本是0.20309238
        "_source": { "title": "ES 进阶", "like": 5 }
    },
    {
        "_score": 0.5753642, //原本是0.2876821
        "_source": { "title": "ES 入门", "like": 2 }
    }
]
```

### 4. 加上 max_boost，限制 field_value_factor 的最大 加强score

- 可以看到"ES 入门"的加强score是2，在max_boost限制里，所以不受影响
- 而ES进阶和ES最高难度的field_value_factor函数产生的加强score因为超过max_boost的限制，所以被设为3
```text
GET 127.0.0.1/mytest/_doc/_search
{
    "query": {
        "function_score": {
            "query": {
                "match": {
                    "title": "ES"
                }
            },
            "field_value_factor": {
                "field": "like"
            },
            "max_boost": 3
        }
    }
}
```
```json
"hits": [
    {
        "_score": 0.6092771, //原本是0.20309238
        "_source": { "title": "ES 进阶", "like": 5 }
    },
    {
        "_score": 0.5753642, //原本是0.2876821
        "_source": { "title": "ES 入门", "like": 2 }
    },
    {
        "_score": 0.49621537, //原本是0.16540512
        "_source": { "title": "ES 最高难度", "like": 10 }
    }
]
```

### 5. 有时候线性的计算 new_score = old_score * like 值的效果并不是那麽好， field_value_factor 中还支持 modifier、factor 参数，可以改变 like 值对 old_score 的影响

- `modifier` 参数支持的值
    - none : new_score = old_score * like值
        - 默认状态就是none，线性
        ![](/images/ElasticSearch-function-score-系列文章（二）field-value-factor-具体实例/1.png)
    - log1p : new_score = old_score * log(1 + like值)
        - 最常用，可以让like值字段的评分曲线更平滑
        ![](/images/ElasticSearch-function-score-系列文章（二）field-value-factor-具体实例/2.png)
    - log2p : new_score = old_score * log(2 + like值)
    - ln : new_score = old_score * ln(like值)
    - ln1p : new_score = old_score * ln(1 + like值)
    - ln2p : new_score = old_score * ln(2 + like值)
    - square : 计算平方
    - sqrt : 计算平方根
    - reciprocal : 计算倒数
- `factor` 参数
    - factor作为一个调节用的参数，没有modifier那麽强大会改变整个曲线，他仅改变一些常量值，设置factor>1会提昇效果，factor<1会降低效果
    - 假设modifier是log1p，那麽加入了factor的公式就是new_score = old_score * log(1 + factor * like值)
    ![](/images/ElasticSearch-function-score-系列文章（二）field-value-factor-具体实例/3.png)
- 对刚刚的例子加上 `modifier`、`factor`
```text
GET 127.0.0.1/mytest/_doc/_search
{
    "query": {
        "function_score": {
            "query": {
                "match": {
                    "title": "ES"
                }
            },
            "field_value_factor": {
                "field": "like",
                "modifier": "log1p",
                "factor": 2
            }
        }
    }
}
```

### 6. 就算加上了 modifier，但是 "全文评分 与 field_value_factor 函数值乘积" 的效果可能还是太大，我们可以通过参数 boost_mode 来决定 old_score 和 加强score 合并的方法

- 如果将boost_mode改成sum，可以大幅弱化最终效果，特别是使用一个较小的factor时
- 加入了boost_mode=sum、且factor=0.1的公式变为new_score = old_score + log(1 + 0.1 * like值)
![](/images/ElasticSearch-function-score-系列文章（二）field-value-factor-具体实例/4.png)
```text
GET 127.0.0.1/mytest/_doc/_search
{
    "query": {
        "function_score": {
            "query": {
                "match": {
                    "title": "ES"
                }
            },
            "field_value_factor": {
                "field": "like",
                "modifier": "log1p",
                "factor": 0.1
            },
            "boost_mode": "sum"
        }
    }
}
```
