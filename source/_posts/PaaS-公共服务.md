---
layout: post
title: PaaS 公共服务
date: 2021-01-05 13:43:00
tags:
- 原创
categories:
- Dgg
- Paas
---

<p>

<!-- more -->

## 🌟 [消息中心](http://172.16.0.25/technology-architecture-center/technology-project/dgg-tac-message-center/tree/crisps)

- 提供接口：根据模板id获取模板内容
- 短信运营商回调（含状态报告、上行回复）
- 取消mongo日志归档定时任务，已移至中台系统

## 🌟 [技术中台管理系统](http://172.16.0.25/technology-architecture-center/technology-department/dgg-framework-manage/tree/crisps)

- 消息中心通道、账户、签名、模板、日志功能
- 缓存管理Redis多数据源配置、梯度更新策略、白名单设置

## 🌟 [第三方服务平台](http://172.16.0.25/technology-architecture-center/technology-department/dgg-tac-external-platform.git)

- 信息查询：身份证实名认证、手机号三要素认证、银行卡四要素认证、企业工商信息查询、OCR图片文字识别
- OSS文件查询：STS临时凭证授权、文件查询精确匹配和模糊匹配、文件物理删除和逻辑删除、恢复逻辑删除文件等

## 🌟 [分布式ID](http://172.16.0.25/technology-architecture-center/technology-department/PAAS/tree/crisps/dgg-framework-idcreater)

## 🌟 [TiCDC同步服务](http://172.16.0.25/technology-architecture-center/technology-project/crisps-paas-ticdc)

## 🌟 [录音中间层服务](http://172.16.0.25/technology-architecture-center/technology-project/crisps-paas-voice-record)

## 🌟 [iboss站内信](http://172.16.0.25/wangxianwei/cloud-message-instation)

***
