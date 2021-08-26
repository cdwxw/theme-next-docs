---
layout: post
title: ElasticSearch - function_score 系列文章（一）简介
date: 2021-08-26 14:48:15
tags:
---

<p>

<!-- more -->

{% note info no-icon %}
博客地址：[isee.wang](/)
{% endnote %}

## 系列文章

1. ElasticSearch - function_score 系列文章（一）简介
2. [ElasticSearch - function_score 系列文章（二）field_value_factor 具体实例](/ElasticSearch-function-score-系列文章（二）field-value-factor-具体实例)
3. [ElasticSearch - function_score 系列文章（三）weight 具体实例](/ElasticSearch-function-score-系列文章（三）weight-具体实例)
4. [ElasticSearch - function_score 系列文章（四）衰减函数 linear、exp、gauss 具体实例](/ElasticSearch-function-score-系列文章（四）衰减函数-linear、exp、gauss-具体实例)

## 预备知识

在使用ES进行全文搜索时，搜索结果默认会以 `文档的相关度` 进行排序，而这个 `文档的相关度`，是可以透过 `function_score` 自己定义的，也就是说我们可以透过使用 `function_score`，来控制“怎麽样的文档相关度更高”这件事

- `function_score` 是专门用于处理文档 _score 的 DSL，它允许为每个主查询 query 匹配的文档应用加强函数，以达到改变原始查询评分 score 的目的
- `function_score` 会在主查询 query 结束后对每一个匹配的文档进行一系列的重打分操作，能够对多个字段一起进行综合评估，且能够使用 filter 将结果划分为多个子集 (每个特性一个filter)，并为每个子集使用不同的加强函数

## function_score 提供了几种加强 _score 计算的函数

1. `weight` : 设置一个简单而不被规范化的权重提升值
    - weight 加强函数和 boost 参数类似，可以用于任何查询，不过有一点差别是 weight 不会被 Lucene nomalize 成难以理解的浮点数，而是直接被应用 (boost会被nomalize)
    - 例如当 weight 为 2 时，最终结果为 new_score = old_score * 2
2. `field_value_factor` : 将某个字段的值乘上 old_score
    - 像是将字段 shareCount 或是 字段 likeCount 作为考虑因素，new_score = old_score * 那个文档的 likeCount 的值
3. `random_score` : 为每个用户都使用一个不同的随机评分对结果排序，但对某一具体用户来说，看到的顺序始终是一致的
4. `衰减函数 (linear、exp、guass)` : 以某个字段的值为基准，距离某个值越近得分越高
5. `script_score` : 当需求超出以上范围时，可以用自定义脚本完全控制评分计算，不过因为还要额外维护脚本不好维护，因此尽量使用 ES 提供的评分函数，需求真的无法满足再使用 script_score

## function_scroe 其他辅助的参数

1. `boost_mode` : 决定 old_score 和 加强score 如何合并
    - multiply(默认) : new_score = old_score * 加强score
    - sum : new_score = old_score + 加强score
    - min : old_score 和 加强score 取较小值，new_score = min(old_score, 加强score)
    - max : old_score 和 加强score 取较大值，new_score = max(old_score, 加强score)
    - replace : 加强score直接替换掉 old_score，new_score = 加强score
2. `score_mode` : 决定 functions 里面的 加强score 们怎么合并，会先合并 加强score 们成一个 总加强score，再使用 总加强score 去和 old_score 做合并，换言之就是会先执行 score_mode，再执行 boost_mode
    - multiply (默认)
    - sum
    - avg
    - first : 使用首个函数(可以有filter，也可以没有)的结果作为最终结果
    - max
    - min
3. `max_boost` : 限制加强函数的最大效果，就是限制 加强score 最大能多少，但要注意不会限制 old_score
    - 如果 加强score 超过了 max_boost 限制的值，会把 加强score 的值设成 max_boost 的值
    - 假设 加强score 是 5，而 max_boost 是 2，因为 加强score 超出了 max_boost 的限制，所以 max_boost 就会把 加强score 改为 2
    - 简单的说，就是 加强score = min(加强score, max_boost)

## function_score 查询模板
   
1. 如果要使用 function_score 改变分数，要使用 function_score 查询
2. 简单的说，就是在一个 function_score 内部的 query 的全文搜索得到的 _score 基础上，给他加上其他字段的评分标准，就能够得到把 "全文搜索 + 其他字段" 综合起来评分的效果
3. 单个加强函数的查询模板
```text
GET 127.0.0.1/mytest/_doc/_search
{
    "query": {
        "function_score": {
            "query": {.....}, //主查询，查询完后这里自己会有一个评分，就是old_score
            "field_value_factor": {...}, //在old_score的基础上，给他加强其他字段的评分，这里会产生一个加强score，如果只有一个加强function时，直接将加强函数名写在query下面就可以了
            "boost_mode": "multiply", //指定用哪种方式结合old_score和加强score成为new_score
            "max_boost": 1.5 //限制加强score的最高分，但是不会限制old_score
        }
    }
}
```
4. 多个加强函数的查询模板
    - 如果有多个加强函数，那就要使用 functions 来包含这些加强函数们，functions 是一个数组，里面放着的是将要被使用的加强函数列表
    - 可以为 functions 里的加强函数指定一个 filter，这样做的话，只有在文档满足此 filter 的要求，此 filter 的加强函数才会应用到文挡上，也可以不指定 filter，这样的话此加强函数就会应用到全部的文挡上
    - 一个文档可以一次满足多条加强函数和多个 filter，如果一次满足多个，那麽就会产生多个 加强score，因此 ES 会使用 score_mode 定义的方式来合并这些 加强score们，得到一个 总加强score，得到 总加强score之后，才会再使用 boost_mode 定义的方式去和 old_score 做合并
    - 像是下面的例子，field_value_factor 和 gauss 这两个加强函数会应用到所有文档上，而 weight 只会应用到满足 filter 的文档上，假设有个文档满足了 filter 的条件，那他就会得到 3 个 加强score，这 3 个 加强score 会使用 sum 的方式合并成一个 总加强score，然后才和 old_score 使用 multiply 的方式合并
```text
GET 127.0.0.1/mytest/_doc/_search
{
    "query": {
        "function_score": {
            "query": {.....},
            "functions": [   //可以有多个加强函数(或是filter+加强函数)，每一个加强函数会产生一个加强score，因此functions会有多个加强score
                { "field_value_factor": ... },
                { "gauss": ... },
                { "filter": {...}, "weight": ... }
            ],
            "score_mode": "sum", //决定加强score们怎麽合并,
            "boost_mode": "multiply" //決定總加強score怎麼和old_score合并
        }
    }
}
```
5. 不要执着在调整 function_score 上
    - 文档相关度的调整非常玄，"最相关的文档" 是一个难以触及的模糊概念，每个人对文档排序有着不同的想法，这很容易使人陷入持续反覆调整，但是确没有明显的进展
    - 为了避免跳入这种死循环，在调整 function_score 时，一定要搭配监控用户操作，才有意义
        - 像是如果返回的文档是用户想要的高相关的文档，那麽用户就会选择前 10 个中的一个文档，得到想要的结果，反之，用户可能会来回点击，或是尝试新的搜索条件
        - 一旦有了这些监控手段，想要调适完美的 function_score 就不是问题
    - 因此调整 function_score 的重点在于，要透过监控用户和用户互动，慢慢去调整我们的搜索条件，而不要妄想一步登天，第一次就把文档的相关度调整到最好，这几乎是不可能的，因为，连用户自己也不知道他自己想要什麽
