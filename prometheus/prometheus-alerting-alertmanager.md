title: prometheus-alerting-alertmanager
tags:
  - prometheus
  - 监控
categories:
  - 监控
date: 2019-04-09 15:10:00
---
[原文](https://prometheus.io/docs/alerting/alertmanager/)

Alertmanager处理客户端应用程序（如Prometheus服务器）发送的警报。它负责对它们进行重复数据删除，分组和路由，以及正确的接收器集成，例如电子邮件，PagerDuty或OpsGenie。它还负责警报的静音(silencing)和抑制(inhibition)。

以下描述了Alertmanager实现的核心概念。请参阅[配置文档](https://prometheus.io/docs/alerting/configuration)以了解如何更详细地使用它们。

## 分组(Grouping)
分组将类似性质的警报分类为单个通知。在许多系统一次性失败并且数百到数千个警报可能同时发生的较大中断(outage)期间，这尤其有用。

`示例`：发生网络分区时，群集中正在运行数十或数百个服务实例。一半的服务实例无法再访问数据库。Prometheus中的警报规则配置为在每个服务实例无法与数据库通信时发送警报。结果，数百个警报被发送到Alertmanager。

作为用户，人们只想获得单个页面，同时仍能够确切地看到哪些服务实例受到影响。因此，可以将Alertmanager配置为按集群和alertname对警报进行分组，以便发送单个紧凑通知。

通过配置文件中的路由树配置警报的分组，分组通知的定时以及这些通知的接收器。

## 告警抑制(Inhibition)

如果某些其他警报已经触发，则`告警抑制`是抑制某些警报的通知的概念。

`示例`：正在触发警报，通知无法访问整个集群。Alertmanager可以配置为在该特定警报触发时将与该集群有关的所有其他警报静音。这可以防止发送与实际问题无关的数百或数千个触发警报。

`告警抑制通过Alertmanager的配置文件配置。`

## 静音(Silences)

静音(Siliences)是在给定时间内简单地静音警报(mute alerts)的简单方法。基于匹配器配置静音，就像路由树一样。检查传入警报是否与活动静音的相等或匹配正则表达式。如果符合，则不会发送该警报的通知。

`在Alertmanager的Web界面中配置静音。`

## 客户行为

Alertmanager对其客户的行为有[特殊要求](https://prometheus.io/docs/alerting/clients)。这些仅适用于不使用Prometheus发送警报的高级用例。

## 高可用性

Alertmanager支持配置以创建用于高可用性的集群。可以使用[`--cluster- *`](https://github.com/prometheus/alertmanager#high-availability)标志进行配置。

不要在Prometheus和它的Alertmanagers之间加载平衡流量，而是将Prometheus指向所有Alertmanagers的列表。