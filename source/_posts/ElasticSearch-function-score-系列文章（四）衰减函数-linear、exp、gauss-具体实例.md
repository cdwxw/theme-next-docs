---
layout: post
title: ElasticSearch - function_score 系列文章（四）衰减函数 linear、exp、gauss 具体实例
date: 2021-08-26 15:49:03
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
3. [ElasticSearch - function_score 系列文章（三）weight 具体实例](/ElasticSearch-function-score-系列文章（三）weight-具体实例)
4. ElasticSearch - function_score 系列文章（四）衰减函数 linear、exp、gauss 具体实例

## 1. 很多变量都可以影响用户对于酒店的选择，像是用户可能希望酒店离市中心近一点，但是如果价格足够便宜，也愿意为了省钱，妥协选择一个更远的住处

- 如果我们只是使用一个 filter 排除所有市中心方圆 100 米以外的酒店，再用一个filter排除每晚价格超过100元的酒店，这种作法太过强硬，可能有一间房在 500米，但是超级便宜一晚只要10元，用户可能会因此愿意妥协住这间房
- 为了解决这个问题，因此function_score查询提供了一组 衰减函数 (decay functions)， 让我们有能力在两个滑动标准(如地点和价格)之间权衡

## 2. function_score支持的衰减函数有三种，分别是 linear、exp 和 gauss

- linear、exp、gauss三种衰减函数的差别只在于衰减曲线的形状，在DSL的语法上的用法完全一样
    - linear : 线性函数是条直线，一旦直线与横轴0香蕉，所有其他值的评分都是0
    - exp : 指数函数是先剧烈衰减然后变缓
    - guass(最常用) : 高斯函数则是钟形的，他的衰减速率是先缓慢，然后变快，最后又放缓
    ![](/images/ElasticSearch-function-score-系列文章（四）衰减函数-linear、exp、gauss-具体实例/1.png)
- 衰减函数们 (linear、exp、gauss) 支持的参数
    - origin : 中心点，或是字段可能的最佳值，落在原点(origin)上的文档评分_score为满分1.0，支持数值、时间 以及 "经纬度地理座标点"(最常用) 的字段
    - offset : 从 origin 为中心，为他设置一个偏移量offset覆盖一个范围，在此范围内所有的评分_score也都是和origin一样满分1.0
    - scale : 衰减率，即是一个文档从origin下落时，_score改变的速度
    - decay : 从 origin 衰减到 scale 所得的评分_score，默认为0.5 (一般不需要改变，这个参数使用默认的就好了)
    - 以上面的图为例
        - 所有曲线(linear、exp、gauss)的origin都是40，offset是5，因此范围在40-5 <= value <= 40+5的文档的评分_score都是满分1.0
        - 而在此范围之外，评分会开始衰减，衰减率由scale值(此处是5)和decay值(此处是默认值0.5)决定，在origin +/- (offset + scale)处的评分是decay值，也就是在30、50的评分处是0.5分
        - 也就是说，在origin + offset + scale或是origin - offset - scale的点上，得到的分数仅有decay分

## 3. 具体实例一
   
- 先准备数据和索引，在ES插入三笔数据，其中language是keywork类型，like是integer类型(代表点赞量)
```text
{ "language": "java", "like": 5 }
{ "language": "python", "like": 10 }
{ "language": "go", "like": 15 }
```
- 以like=15为中心，使用gauss函数
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
                    "gauss": {
                        "like": {
                            "origin": "15", //如果不设置offset，offset默认为0
                            "scale": "5",
                            "decay": "0.2"
                        }
                    }
                }
            ]
        }
    }
}
```
```json
"hits": [
    {
        "_score": 1,
        "_source": { "language": "go", "like": 15 }
    },
    {
        "_score": 0.2, //因为改变了decay=0.2，所以当位于 origin-offset-scale=10 的位置时，分数为decay，就是0.2
        "_source": { "language": "python", "like": 10 }
    },
    {
        "_score": 0.0016,
        "_source": { "language": "java", "like": 5 }
    }
]
```

## 4. 具体实例二

- 假设有一个用户希望租一个离市中心近一点的酒店，且每晚不超过100元的酒店，而且与距离相比，我们的用户对价格更敏感，那麽使用衰减函数guass查询如下
    - 其中把price语句的origin点设为50是有原因的，由于价格的特性一定是越低越好，所以0~100元的所有价格的酒店都应该认为是比较好的，而100元以上的酒店就慢慢衰减
    - 如果我们将price的origin点设置成100，那麽价格低于100元的酒店的评分反而会变低，这不是我们期望的结果，与其这样不如将origin和offset同时设成50，只让price大于100元时评分才会变低
    - 虽然这样设置也会使得price小于0元的酒店评分降低没错，不过现实生活中价格不会有负数，因此就算price<0的评分会下降，也不会对我们的搜索结果造成影响(酒店的价格一定都是正的)
    - 换句话说，其实只要把origin + offset的值设为100，origin或offset是什麽样的值都无所谓，只要能确保酒店价格在100元以上的酒店会衰减就好了
```text
GET 127.0.0.1/mytest/_doc/_search
{
    "query": {
        "function_score": {
            "functions": [
                //第一个gauss加强函数，决定距离的衰减率
                {
                    "gauss": {
                        "location": {
                            "origin": {  //origin点设成酒店的经纬度座标
                                "lat": 51.5,
                                "lon": 0.12
                            },
                            "offset": "2km", //距离中心点2km以内都是满分1.0，2km外开始衰减
                            "scale": "3km"  //衰减率
                        }
                    }
                },
                //第二个gauss加强函数，决定价格的衰减率，因为用户对价格更敏感，所以给了这个gauss加强函数2倍的权重
                {
                    "gauss": {
                        "price": {
                            "origin": "50", 
                            "offset": "50",
                            "scale": "20"
                        }
                    },
                    "weight": 2
                }
            ]
        }
    }
}
```
