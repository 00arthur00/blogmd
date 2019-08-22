---
title: thanos弃用gossip采用文件发现服务(file discovery service)的原因
date: 2019-06-28 23:49:50
tags:
  - 监控
  - thanos
categories:
  - 监控
---

## 总结
1. 做不了负载均衡(可直接访问peer的IP地址)
2. 私有IP地址推断有可能会出错
3. 没必要让每个peer都知道对方的存在,Thanos query/rule知道store就可以了
4. 作者觉得维护麻烦，省了一件事

## 详细信息
[官网](https://thanos.io/proposals/201809_gossip-removal.md/)
* Gossip has been proven to be extremely confusing for the new users. Peer logic and really confusing `cluster.advertise-address` that was sometimes able to magically deduce private IP address (and sometimes not!) were leading to lots of questions and issues. Something that was made for quick ramp-up into Thanos (“just click the button and it auto-joins everything”) created lots of confusion and made it harder to experiment. 

** Gossip给新用户带来困扰。Peer逻辑和让人困扰的`cluster.advertise-address`带来很多问题。`cluster.advertise-address`有时候能神奇的推断出私有IP地址，有时候不可以！Gossip这种自动加入的特性使其难以进行试验。**

* Auto-join logic has its price. All peers connected were aware of each other. All were included in metadata propagation. Membership project community made an awesome job to optimize N^N communication (all-to-all), but nevertheless, it is something that is totally unnecessary to Thanos. And it led to wrong assumptions that e.g sidecar needs to be aware of another sidecar etc. Thanos Querier is the only component that requires to be aware of other StoreAPIs. It is clear that all-to-all communication is neither necessary nor educative. 

自动加入逻辑是有代价的。**所有的用户都相互知道对方的存在。所有信息都在包含在元信息传播中。** 成员关系工程社区对这种**N^N的通信(所有-所有)**的通信做了非常好的优化，然而，有时候这对Thanos来说是完全没必要的。并且导致错误的假设，比如，sidecar需要知道别的sidecar。Thanos Query是为一个一个需要知道其它store API的组件。显然这种所有-所有的通信既没必要也没有可学习的。

* Unless specifically configured (which requires advanced knowledge) Gossip uses mix of TCP and UPD underneath its custom app level protocol. This is a no-go if you use L7 loadbalancers and proxies. 

**除非特意配置(需要高级知识)，Gossip在定制的APP层协议下混合使用TCP和UDP。如果你使用7层负载均衡或者代理，这是不可行的。**

* Global gossip is really difficult to achieve. BTW, If you know how to setup this, please write a blog about it! (: 

难以实现。

* With the addition of the simplest possible solution to give Thanos Querier knowledge where are StoreAPIs (static --store flag), we needed to implement health check and metadata propagation anyway. In fact, StoreAPI.Info was already there all the time. 


* Gossip operates per peer level and there is no way you can abstract multiple peers behind loadbalancer. This hides easy solutions from our eyes, e.g how to make Store Gateway HA. Without gossip, you can just use Kubernetes HA Service or any other loadbalancer. To support Store Gateway HA for gossip we would end up implementing LB logic in Thanos Querier (like proposed here) 

**Gossip在对等层运行，没有办法可以抽象多个peer到负载均衡器之后。**

* At some point we want to be flexible and allow other discovery mechanisms. Gossip does not work for everyone and static flags are too.. static. (: We need File SD for flexibility anyway. 

** 有时候，我们希望一定的灵活性并且允许其它的发现机制。Gossip并不对每个人都有用，static flag又太静态了。。。 **

* One thing less to maintain.

** 少维护一件事 **



