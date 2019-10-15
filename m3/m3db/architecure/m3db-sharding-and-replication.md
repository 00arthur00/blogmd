title: m3db分片和复制
tags:
  - m3db
  - 时间序列数据库
categories: []
date: 2019-05-16 22:57:00
---
## 分片

时间序列键被散列到一组固定的虚拟分片。然后将虚拟分片分配给物理节点。可以将M3DB配置为使用任何散列函数和配置数量的分片。默认情况下，[murmur3](https://en.wikipedia.org/wiki/MurmurHash)用作散列函数，并配置4096个虚拟分片。

## 优点

分片在整个M3DB堆栈中提供了各种好处：

1. 它们使得水平扩展更容易，并且可以在集群级别上添加/删除节点而无需停机。
2. 它们在内存级别提供更细粒度的锁粒度。
3. 它们通知文件系统组织，属于同一分片的数据将被一起使用/删除，并且可以保存在同一文件中。

## 复制(Replication)

每个副本的虚拟分片放置逻辑分片，并具有可配置的隔离（区域感知，机架识别等）。例如，当使用机架感知隔离时，定位副本数据的数据中心机架集与定位所有其他副本数据的机架不同。

复制在写入期间是同步的，并且根据配置的一致性级别将通知客户端关于一致性级别和复制实现的写入是成功还是失败。

## 复本(Replica)

每个副本都有自己的每个虚拟分片的单个逻辑分片的分配。

从概念上讲，它可以定义为：
``` golang
Replica {
  id uint32
  shards []Shard
}
```

## 分片状态
每个分片在概念上可以定义为：
``` golang
Shard {
  id uint32
  assignments []ShardAssignment
}

ShardAssignment {
  host Host
  state ShardState
}

enum ShardState {
  INITIALIZING,
  AVAILABLE,
  LEAVING
}
```

## 分片分配
分片的分配存储在etcd中。添加，删除或替换节点分片时，会为分配的每个分片分配目标状态。

要使写入对于给定副本显示为成功，它必须对该分片的所有已分配主机成功。这意味着如果存在给定的分片，其中主机被指定为LEAVING而另一个主机被分配为INITIALIZING，则给定的副本写入这两个主机必须看起来成功返回成功以写入该给定副本。然而，目前只有可用的分片计入一致性，在计算写入成功/错误时将LEAVING和INITIALIZING分片组合在一起的工作尚未完成，参见[问题417](https://github.com/m3db/m3/issues/417)。

当在INITIALIZING状态中发现分配新分片时，由节点本身自行引导分片，并在完成后通过调用集群管理API将状态转换为AVAILABLE。使用比较并以原子方式设置它将删除仍分配给先前拥有它的节点的LEAVING分片，并将新节点上的分片状态从INITIALIZING状态转换为AVAILABLE。

节点将不会开始为新分片提供读取，直到变为AVAILABLE，这意味着直到它们为这些分片提供了引导数据。

## 群集操作

### 节点添加
将节点添加到群集时，会为其分配分片，从而可以从现有节点中公平地减轻负载。 分配给新节点的分片将变为INITIALIZING，然后节点发现它们需要被引导，并将使用所有可用的副本开始引导数据。 将从现有节点中删除的分片标记为LEAVING。

### 节点宕机
需要明确地从群集中取出节点。 如果节点发生故障并且不可用，则执行读取的客户端将从副本中为该节点拥有的分片范围提供错误。 在此期间，它将依赖来自其他副本的读取来继续不间断的操作。

### 节点移除
删除节点后，它拥有的分片将分配给群集中的现有节点。剩余的服务器发现它们现在拥有INITIALIZING的分片，需要进行自举，并将使用所有可用的副本开始引导数据。
