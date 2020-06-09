---
layout: post
title:  [翻译]Micrometer原理
date:   2020-06-09
categories: 
  - java
comments: false
---

> 最近我们在开始增加对系统的监控预警，初步计划是采用Micrometer在JVM中针对指标进行统计，再通过Prometheus汇总/持久化数据，通过Grafana进行数据呈现。其中的重点就是在JVM的数据采集，但是中文网络上大部分都是基本的demo和简单的说明，所以对官方文档中的Concept进行了翻译
 
<!-- more -->

## 1.目的

Micrometer是一个针对以JVM为容器的的应用的指标统计统计工具，它针对大多数常用监控系统提供了一套简单的封装接口，使得你可以脱离服务商的约束而去优化你的JVM应用。Micrometer的设计目标就是最小化额外的指标采集行为却能提供最好的指标采集效果。

Micrometer _不是_ 一个分布式的监控系统或者一个事件记录器. Adrian Cole的 [监控的三种方式](https://www.dotconferences.com/2017/04/adrian-cole-observability-3-ways-logging-metrics-tracing)很详细的阐述了不同类型的监控系统的主要差异.

## 2.支持的监控系统

Micrometer包含一个有可扩展的[SPI](https://en.wikipedia.org/wiki/Service_provider_interface)的核心模块,一系列针对不同监控系统的适配模块(每一个都被称为注册中心/registry)以及一个测试套件。在这个监控系统中，有三个非常需要理解的重要角色：

 - 维度（Dimensionality）是指数据指标是否能够通过key/value的标签（tag）来进行增强。如果一个系统不支持多维的监控，它就是分层监控，也就是说它只能支持一个水平的指标名称。而Micrometer在给分层监控进行数据发布的时候，会将tag的key/value降维拼接到指标名称中（译者注：这里比较难翻译，其实就是指系统监控的维度能否嵌套，如果不能通过tag来进行嵌套，则只能把原本通过tag来进行区分的子项指标挨个平滑降维变为一个个独立的指标）。
 
  多维监控系统 | 分层监控系统 
  -|-
  AppOptics, Atlas, Azure Monitor, Cloudwatch, Datadog, Datadog StatsD, Dynatrace, Elastic, Humio, Influx, KairosDB, New Relic, Prometheus, SignalFx, Sysdig StatsD, Telegraf StatsD, Wavefront | Graphite, Ganglia, JMX, Etsy StatsD 
 
 - 频度聚合（Rate aggregation）
 - 发布（Publishing）

## 3.注册中心/Registry
## 4.量具/Meter
## 5.量具命名
## 6.量具过滤器
## 7.速率聚合
## 8.总量计数器/Counters
## 9.存量计数器/Gauges
## 10.计时器/Timers
## 11.分布式统计/Distribution Summaries
## 12.长任务计时器/Long task timers
## 13.直方统计和百分比统计/Histograms and percentiles