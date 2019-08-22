title: 'prometheus tsdb deeper and lower(2) - Krasi’s Notes! '
date: 2019-08-12 11:31:39
tags:
  - prometheus
  - 监控
categories:
  - 监控
---
在写入和读取数据时，它会执行此操作

* 磁盘上的持久数据 - 块
* 内存缓存称为Head - 批量磁盘写入并减少SSD磨损

当试图理解执行逻辑时，遵循持久化块读取和写入路径可能更容易入手。

**块(Chunk)** - 是字节格式的序列样本的实际持有者。压缩三角形并按[120](https://github.com/prometheus/tsdb/blob/def6e5a57439cffe7b44a619c05bce4ac513a63e/head.go#L1207)个样本分组，以实现快速解压缩。

`memSeries`结构是所有`memChunks`保存到磁盘之前的持有者。

接口:

https://github.com/prometheus/tsdb/blob/def6e5a57439cffe7b44a619c05bce4ac513a63e/chunkenc/chunk.go#L43-L49

通过XORChunk结构实现:

https://github.com/prometheus/tsdb/blob/def6e5a57439cffe7b44a619c05bce4ac513a63e/chunkenc/xor.go#L1-L386 

**Series** 为labelName/labelValue对,例如:http_requests=post,http_requests=get

**Samples**为timestamp/value对,例如:t:10,v:1234     t:11,v:1200

[**头(Head)**](https://github.com/prometheus/tsdb/blob/def6e5a57439cffe7b44a619c05bce4ac513a63e/head.go#L53-L73)-  
首先，数据保留在内存中以避免将每个样本写入磁盘，因此这允许对大批样本进行单次写入。保存数据的结构是在
`stripeSeries`使用`map[uint64]`由id索引。
每个stripeSeries都有自己的互斥锁，以减少锁争用。

我们通过headAppender将序列/样品添加到Head中。这是将样本分组添加到单个事务中，只有在调用**Commit()**之后才能查询。
```
type headAppender struct {
	head       *Head
	mint, maxt int64
	series  []RefSeries
	samples []RefSample
}
```

`headAppender`实现了Appender接口。
* Add(l labels.Labels, t int64, v float64) - 使用标签哈希它检查Head中是否存在（在`stripeSeries`中）。如果缺少引用它会创建一个新的引用并使用它将新的memSeries添加到head的stripeSeries中。还将标签添加到自己的`[]RefSeries`切片（为什么????），而不是使用引用调用`AddFast`. **注:新版本的代码实现是调用AddFast**

* AddFast(ref uint64, t int64, v float64) -  使用ref获取指向时间序列的指针并将样本添加到其`[]RefSample`切片与引用序列的指针一起使用。通过这种方式在`Commit`所有t，v对将被添加到由保存的指针引用的相同序列。

* Commit() - [将所有`headAppender`序列和样本写入WAL文件](○https://github.com/prometheus/tsdb/blob/def6e5a57439cffe7b44a619c05bce4ac513a63e/head.go#L526-L531)。 [遍历所有样本并将它们附加到它们通过保存在它们自己的结构中的系列指针引用的序列中。这也使它们可用于查询](○https://github.com/prometheus/tsdb/blob/def6e5a57439cffe7b44a619c05bce4ac513a63e/head.go#L535-L547)。

* RollBack() - 准备一个空头的appendPool，这样在请求新的appender时它不会分配新的内存。???? 如果我们在调用Rollback后尝试使用现有的appender会发生什么？ **注:调用Rollback之后就不能重用appender了,[详见](https://github.com/prometheus/tsdb/blob/d5b3f0704379a9eaca33b711aa0097f001817fc2/db.go#L89-L90)

**WAL** - append only文件,用于在发生崩溃时恢复内存中序列,每个WAL段的Head块是256MB，它被截断为重启时写入的内容。再次分区为256MB，你写入了256MB，创建一个新段。

**DB** - 根对象用于设置:
 - 要使用的目录
 - 压缩器
 - WAL文件
 - 头部
 - 读取和写入Meta文件。
 
**Block** - 持久存储在由单个目录中写入的连续时间序列的非重叠时间窗口限制的磁盘组块上.**注:新版本有实验性质的允许时间窗口重叠的选项**

**Compaction(压缩)** - 将块合并在一起以减少查询时间

**Meta文件** - json文件描述块(block)-时间范围，压缩级别等

**Tombstones(墓碑)** - 时间序列设置为在下一次压缩时删除。查询时也会排除这些。

**index(索引)** - 查询数据时使用的索引。压缩数据时会写入索引。postings list - 包含与列表关联的给定标签对的序列引用列表。[map[labels.Label][]uint64](https://github.com/prometheus/tsdb/blob/c848349f07c83bd38d5d19faa5ea71c7fd8923ea/index/postings.go#L39)  -  foo，bar  -  1,3,12,14  - 这意味着带有值bar的标签foo与ID 1,3 ......关联 - 这被用作引用表来获取我们需要的序列。
符号 -  ??????????

非常好的高级概述索引，块，墓碑文件格式:https://github.com/prometheus/tsdb/tree/master/docs/format

# Opening the DB - tsdb.Open

选择时间序列：

创建一个查询器对象，以便能够在1,1000个时间范围内选择数据

`querier, err := db.Querier(1, 1000)` - 读取所有块（包括头部）的元信息，并返回给定时间范围内每个块的块查询器。

现在我们需要一些过滤

`matcher:=labels.NewEqualMatcher("foo", "bar")` - 过滤结果的Matcher接口。

现在我们可以使用此匹配器运行Select
``` golang
querier.Select(matcher)
```

首先，我们遍历时间范围内所有Block的所有Postings，以找出那些块包含这些Matchers的时间序列。
``` golang
Head struct {
	// All series addressable by their ID or hash.
	series *stripeSeries - prealocated arrays with locks , partitioned by hash 
}
```

通过headAppender向Head添加序列.
headAppender用于一个包含一下内容的切片:
``` golang
[]RefSeries 
	Ref    	uint64
	Labels labels.Labels
[]RefSample
	Ref 	uint64
	T  	int64
	V  	float64
	series *memSeries struct {
	ref          uint64
	lset         labels.Labels
	chunks       []*memChunk
	chunkRange   int64
	firstChunkID int
	nextAt    int64 // timestamp at which to cut the next chunk.
	lastValue float64
	app chunkenc.Appender // Current appender for the chunk.
}
```


`headAppender.Add`  - 仅获取或创建Series（而不是样本）.通过标签哈希检查Head.stripeSeries中是否存在Series. 如果不存在则生成新的id并通过`head.stripeSeries.getOrSet()`将新的memSeries添加到全局缓存中.如果创建了新的`memSeries`，`headAppender`还会向`headAppender.[]RefSeries`添加新元素。


接下来是调用`headAppender.AddFast`来添加样本。

`headAppender.AddFast`  - 获取缓存序列，进行一些时间检查以确保样本在`headAppender`时间范围内，并将样本添加到`headAppender.[]RefSample`及其引用和指向该序列的指针。（?????如果我们有引用，它为什么要添加完整系列？）

`headAppender.Commit()` - 将所有`[]RefSample`和`[]RefSeries`写入WAL（因此我们不会在发生崩溃时将其丢失），而不是遍历所有`[]RefSample`并将T和V样本值附加到调用`RefSample.series.append(T，V)`的序列指针。这开始填充chunk并将每个块中的每120个样本分组。

**Head -> HeadAppender**