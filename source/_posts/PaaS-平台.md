---
layout: post
title: PaaS 平台
date: 2021-01-05 13:42:49
tags:
- 原创
categories:
- Dgg
- Paas
---

**阅读更多**

<!-- more -->

## 🌟 [Redis 组件](http://mindoc.dgg188.cn/docs/paas1_1/redisann)

- 基于 `Spring Cache` 扩展实现注解式缓存，并接入nacos动态配置缓存更新策略
- 自定义注解方式实现分布式锁

## 🌟 [Mongo 组件](http://mindoc.dgg188.cn/docs/paas1_1/mongodbtx)

- 基于4.x版本 `MongoDB` 数据库支持事务
- 新增批处理器，支持多数据源，实现 `定时任务` + `异步批处理`

## 🌟 [JAR包扫描 SDK](http://mindoc.dgg188.cn/docs/paas1_1/jarversion)

- 通过SPI机制实现脚手架工程启动时自动扫描所有依赖JAR包信息

## 🌟 [DOC文档解析 SDK](http://mindoc.dgg188.cn/docs/paas1_1/docutils)

- 实现word模板文件内容解析、内容提取、转换PDF格式、文件MD5值校验等功能
- 文件上传后异步回调 `PaaS平台-第三方OSS服务`，接入 [OSS框架](http://mindoc.dgg188.cn/docs/paas1_1/oss_framework)

## 🌟 [第三方服务 SDK](http://mindoc.dgg188.cn/docs/paas1_1/externalservice)

- 信息查询：身份证实名认证、手机号三要素认证、银行卡四要素认证、企业工商信息查询、OCR图片文字识别
- 消息中心：发送短信、企业微信、邮件、APP推送，模板内容查询
- OSS文件查询：STS临时凭证授权、文件查询精确匹配和模糊匹配、文件物理删除和逻辑删除、恢复逻辑删除文件等

## 🌟 [操作日志 SDK](http://mindoc.dgg188.cn/docs/paas1_1/operlog)

- 提供项目操作日志统一异步写入ES、统一筛选查询、置顶等功能

***

[PaaS 平台](http://mindoc.dgg188.cn/docs/paas1_1/paas)
