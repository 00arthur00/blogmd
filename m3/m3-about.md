title: 关于m3
tags:
  - 时间序列数据库
  - m3db
categories: []
date: 2019-05-16 11:22:00
---
[原文](https://m3db.github.io/m3/)
## 关于

在使用开源指标解决方案并大规模发现问题（例如可靠性，成本和操作复杂性）之后，M3从头开始创建，为Uber提供本地分布式时间序列数据库，高度动态且高性能 聚合服务，查询引擎和其他支持基础设施。

## 重要功能

M3具有多种功能，作为分立组件提供，使其成为大规模时间序列数据的理想平台：

* 分布式时间序列数据库M3DB，为时间序列数据和倒排索引提供可扩展存储。
* 一个边车进程，M3Coordinator，允许M3DB作为普罗米修斯的长期存储。
* 一个分布式查询引擎，M3Query，支持PromQL和Graphite（M3QL即将推出）。
* 聚合层M3Aggregator，作为专用度量聚合器/降采样器运行，允许以不同分辨率的各种保留期限存储度量。

## 开始
注意：在生产中运行之前，请务必阅读我们的[操作指南](https://m3db.github.io/m3/operational_guide/index.md)！

开始使用M3就像遵循其中一个操作指南,很简单:
* [单个M3DB节点部署](https://m3db.github.io/m3/how_to/single_node/)
* [集群M3DB部署](https://m3db.github.io/m3/how_to/cluster_hard_way/)
* [Kubernetes上的M3DB](https://m3db.github.io/m3/how_to/kubernetes/)
* [部署上的孤立M3Query](https://m3db.github.io/m3/how_to/query/)

## 技术支持
如需支持任何问题，有关M3或其操作的问题，或留下任何评论，可以通过多种方式联系团队：
* [Gitter](https://gitter.im/m3db/Lobby)
* [Email](https://groups.google.com/forum/#!forum/m3db)
* [Github issues](https://github.com/m3db/m3/issues)
