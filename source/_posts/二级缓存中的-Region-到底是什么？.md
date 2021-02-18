---
layout: post
title: 二级缓存中的 Region 到底是什么？
date: 2021-01-15 14:05:54
tags:
- 转载
categories:
- Cache
- J2Cache
---

<p>

<!-- more -->

不时有人来询问 J2Cache 里的 Region 到底是什么概念，这里做统一的解答。

J2Cache 的 Region 来源于 Ehcache 的 Region 概念。

一般我们在使用像 Redis、Caffeine、Guava Cache 时都没有 Region 这样的概念，特别是 Redis 是一个大哈希表，更没有这个概念。

在实际的缓存场景中，不同的数据会有不同的 TTL 策略，例如有些缓存数据可以永不失效，而有些缓存我们希望是 30 分钟的有效期，有些是 60 分钟等不同的失效时间策略。在 Redis 我们可以针对不同的 key 设置不同的 TTL 时间。但是一般的 Java 内存缓存框架（如 Ehcache、Caffeine、Guava Cache 等），它没法为每一个 key 设置不同 TTL，因为这样管理起来会非常复杂，而且会检查缓存数据是否失效时性能极差。所以一般内存缓存框架会把一组相同 TTL 策略的缓存数据放在一起进行管理。

J2Cache 的 Region 概念对应关系如下所示：

| Ehcache     | region      |
| :---------: | :---------: |
| Caffeine    | Cache       |
| Guava Cache | Cache       |

像 Caffeine 和 Guava Cache 在存放缓存数据时需要先构建一个 Cache 实例，设定好缓存的时间策略，如下代码所示：

```java
Caffeine<Object, Object> caffeine = Caffeine.newBuilder();
caffeine = caffeine.maximumSize(size).expireAfterWrite(expire, TimeUnit.SECONDS);
Cache<String, Object> theCache = caffeine.build();
```

这时候你才可以往 theCache 写入缓存数据，而不能再单独针对某一个 key 设定不同的 TTL 时间。

而 Redis 可以让你非常随意的给不同的 key 设置不同的 TTL。

J2Cache 是内存缓存和 Redis 这类集中式缓存的一个桥梁，因此它只能是兼容两者的特性。

J2Cache 默认使用 Caffeine 作为一级缓存，其配置文件位于 caffeine.properties 中。一个基本使用场景如下：

```properties
#########################################
# Caffeine configuration
# [name] = size, xxxx[s|m|h|d]
#########################################

default = 1000, 30m
users = 2000, 10m
blogs = 5000, 1h
```

上面的配置定义了三个缓存 Region ，分别是：

1. 默认缓存，大小是 1000 个对象，TTL 是 30 分钟
2. users 缓存，大小是 2000 个对象，TTL 是 10 分钟
3. blogs 缓存，大小是 5000 个对象，TTL 是 1 个小时

例如我们可以用 users 来存放用户对象的缓存，用 blogs 来存放博客对象缓存，两种的 TTL 是不同的。

而 default 是当我们调用如下方法时：

```java
public void set(String region, String key, Object value)
```

如果我们传入的 region 参数（假设为：region1）没有在 caffeine.properties 中定义的话，那 J2Cache 会自动创建一个名为 region1 的缓存 Region，其配置和 default 的配置一致。

所以要用好缓存首先要确保以下几点：

1. 根据业务规划好不同的 region 来存放不同的缓存数据
2. 根据实际情况确定每个 region 的缓存数据数量和 TTL 时间
3. 尽量必要未经定义直接使用一个全新的 region （避免使用 default 数据）

本文转载自：https://my.oschina.net/javayou/blog/3031773
