title: prometheus tsdb deeper and lower (1)
date: 2019-08-09 13:58:30
tags:
  - prometheus
  - 监控
categories:
  - 监控
---
这个[博客](https://fabxc.org/tsdb/)和这个[talk](https://www.youtube.com/watch?v=b_pEevMAC3I)将为您提供有关TSDB如何工作的高级概念，但对于那些想要了解的人来说，这将是一篇较低级别的详细博文。这将有助于人们更好地理解内部结构，并可能成为那些希望做出贡献的人的起点。

# 基础

TSDB是一个数据库。它存储时间序列数据。现在什么是时间序列？时间序列是一系列样本，其中每个样本都是时间戳和值。即一个时间序列是[（t0，v0），（t1 v1），（t2，v2）......]

现在几乎每个地方都存在时间序列，例如，Austin的天气就是一个例子：
[（2017年12月5日下午1点，22点），（2017年12月6日下午1点，6点），（2017年12月7日，-1）....]，除了tsdb采用毫秒精度时间戳和只有float64值。但是我们如何表示或命名时间序列？我们用一组(labelname=labelvalue)表示每个时间序列。例如，Austin的天气可以表示为{name=weather,place=austin,unit=celsius}。

现在我们需要存储数百万个序列的数据，每个序列都有数百万个样本。然后我们需要能够查询数据。查询现在是二维的，即我们不仅选择时间序列来检索样本数据，还选择时间范围来选择数据。指定时间范围很容易，它只是说从t=t0到t=t1

现在，在选择时间序列时，我们的数据模型非常容易。我们只需指定labelname-labelvalue约束。例如，要选择**所有**天气时间序列，我们只需使用{name=weather}进行查询。要为Austin选择天气，您只需查询：{name=weather,place=austin}。但约束不仅仅是一个简单的等于约束，我们也有不等于和正则表达式。例如，为Austin以外的每个城市选择天气数据只是:{name=weather,place!=austin}。对于高级用例，我们可以使用类似{name=weather,place=~au.*}的内容，其中=~这里代表正则表达式约束，以选择以"au"开头的所有城市的天气数据。

# 高级视图

现在我们知道了数据和查询的是怎样的，让我们看看tsdb的结构以及它是如何实现的。现在我们有很多数据和很多序列，我们需要存储数据然后有一个索引来帮助我们查询存储的数据。

## 数据存储格式

每个序列的数据都是单独存储的，我们需要存储大量数据。样本是时间戳和值，如果我们有8字节时间戳和8字节值，则每个样本有16个字节。如果我们有500万活跃时间序列，摄入时间为30秒，我们每秒会摄取166K样本，如果我们想将这些数据存储1个月，我们需要7TB的存储空间，如果我们愿意的话存储6个月的数据，然后它会爆炸到42TB，这是一个很大的空间。我们做了一些很好的压缩，使得处理大量数据成为可能。
 
这种压缩来自[Facebook的gorilla论文](gorilla paper)。我们分别压缩时间戳和值。我们使用Prometheus定期搜索数据的属性。

现在从压缩我们可以看到，要访问第n个样本，您需要知道第n个样本，依此类推，直到第一个样本。现在你不想解压缩潜在的数十万个值来访问最后一个样本，或者中间的一些样本。因此，我们将数据分块，即将大约120个样本存储(chunk)在一起，并标记这120个样本的开始和结束时间。如果我们需要在特定时间戳访问该值，我们将找到该序列的相关块(chunk)并开始解码。

现在，实际数据存储在大块文件中，其中每个块(chunk)彼此相邻放置，块的开始的偏移量存储在索引中。

## 索引

现在，当我们累积数据时，每个序列最终会有几个块。[索引](https://github.com/prometheus/tsdb/blob/master/docs/format/index.md)需要帮助我们回答查询，这有助于将labelvalue-labelname匹配器约束转换为具有所有块(chunk)的序列。我们为每个系列分配增量序列ID(seriesID)，然后为每个（lname，lvalue）对构建一个排序[posting list](https://github.com/prometheus/tsdb/blob/master/docs/format/index.md)。即，我们将所有具有该(lname，lvalue)对的序列存储在其labelset中。例如：

name = weather 10,16,20,22,30,35,50,60,75,100,110,120
city = austin 10,12,20,25,50,108,120

现在，当我们想要回答查询{name=weather,city=austin}时，我们只需要获取[两个序列](two series)并获得[id的交集](https://github.com/prometheus/tsdb/blob/07ef80820ef1250db82f9544f3fcf7f0f63ccee0/querier.go)，这应该为我们提供满足查询的所有序列。这里我们基本上采用了所有匹配"name = weather"的序列，并将那些与"city=austin"匹配的序列相交。虽然我们可以回答简单的相等查询，但我们将无法回答正则表达式和不等于查询。

为此，我们为每个labelname[存储所有可能的labelvalue](https://github.com/prometheus/tsdb/blob/07ef80820ef1250db82f9544f3fcf7f0f63ccee0/querier.go)：

city: [austin, auckland, hyderabad, berlin, munich…]
name: [temp, weather, humidity, … ]

当我们得到一个正则表达式查询时，我们查看哪个值匹配正则表达式，查询然后检索该posting list。即。如果我们有austin, auckland, pittsburgh，如果我们有正则表达式查询"au.*"，我们将检查哪个标签值匹配"au.*"，在这种情况下austin和auckland，并获得城市的发布列表city=austin, city=auckland。然后我们将使用seriesID的并集来获得与正则表达式匹配器匹配的seriesID。

现在我们已经有了相关的序列，我们需要找到该序列的相关块。在索引本身，我们还存储了seriesID->[chunk1，chunk2，chunk3 ......]映射，其中我们将开始时间，结束时间和偏移量存储到块文件中块的开头。

现在你可以看到，一旦我们得到一个查询，我们从匹配器中找到相关的序列，然后在指定的时间范围内找到所有重叠的块(chunk)并解码它们以获得所有相关的值。

------------------ INCOMPLETE -----------------

## 多个DB

现在，序列不是恒定的。他们来去匆匆。有些序列只收到几个小时样本，而我们的数据库可以保存数月的数据。现在，如果我们有一个包含所有序列的大型索引，那么如果我们请求2小时的数据，可能只有100K序列在那个时间段内收到了样本，但我们在整个过程中收到样本时间范围内可能有超过5M的序列。这意味着每个查询，即使在1分钟的时间范围内，也会触及所有序列的巨大膨胀指数，这只会随着数据库变大而变得更糟。

这是Prometheus之前数据库的问题之一。实际上要解决这个问题(此处文档不完善,需要插入sharding的逻辑和好处)

# internals

现在让我们看看应用程序如何使用tsdb并查看内部结构。该应用程序使用[`* tsdb.DB`对象](https://gist.github.com/Gouthamve/752a312efe40268a8d4ef844c95ed0d2).

这里我们创建一个db，传递一个文件夹和选项。选项基本上是说，将[预写日志记录](https://en.wikipedia.org/wiki/Write-ahead_logging)到磁盘，丢弃超过15天的数据，并将大小为15分钟的块，然后压缩到1小时，然后压缩到4小时的最大时间范围。

现在，该应用程序使用一个appender：[tsdb.Appender](https://godoc.org/github.com/prometheus/tsdb#Appender) 。当我们执行[app.Add(serries,sample)](https://github.com/prometheus/tsdb/blob/07ef80820ef1250db82f9544f3fcf7f0f63ccee0/head.go#L480-L493)时，我们检查数据库中是否存在序列，如果序列存在,则获取其ref，然后在appender中对ref添加样本。如果序列不存在，请在数据库中创建系列，然后将其添加到新的ref。[CODE](https://github.com/prometheus/tsdb/blob/07ef80820ef1250db82f9544f3fcf7f0f63ccee0/head.go#L878-L891)

当我们调用[Commit()](https://github.com/prometheus/tsdb/blob/07ef80820ef1250db82f9544f3fcf7f0f63ccee0/head.go#L524-L563)时，我们继续将数据添加到 * head中的实际序列中，使其可用于查询而不仅仅是在appender中。

注意：这不是孤立的，查询有时可以从提交中读取部分数据。

**注: 这个问题已经有PR解决了https://github.com/prometheus/tsdb/pull/306**

-------------- INCOMPLETE, SKIP ---------------------

## 追加路径

现在我们看到数据被附加到`db.head`内存块，我们在上面解释的在内存中维护的可变块。它有序列索引LIKE THIS和实际系列存储在这里。

现在当我们调用`Appender()`时，我们创建了headAppender结构，它存储了该事务的所有数据。现在，当我们调用Add()时，我们在这里添加数据，我们在这里创建序列。

当调用commit时，我们继续将序列附加到引用的序列中。

------------------------------------------------

## 压缩路径

现在，我们需要定期将头块刷新到磁盘以减少内存使用量。我们根据时间范围而不是头块大小进行刷新。这个持续时间是我们在选项中传递的最小块大小，在这种情况下，它是15分钟。

现在每个[压缩循环](https://github.com/prometheus/tsdb/blob/07ef80820ef1250db82f9544f3fcf7f0f63ccee0/db.go#L331-L402)，[每1分钟](https://github.com/prometheus/tsdb/blob/07ef80820ef1250db82f9544f3fcf7f0f63ccee0/db.go#L250)，我们检查一下[头块是否有超过22.5分钟（1.5 * 15分钟）的未刷入数据](https://github.com/prometheus/tsdb/blob/07ef80820ef1250db82f9544f3fcf7f0f63ccee0/db.go#L347-L351)。这个0.5的缓冲区是为了确保我们可以附加旧的时间戳。[如果确实存在22.5分钟的数据，我们将最后15分钟的数据作为新块刷新到磁盘](https://github.com/prometheus/tsdb/blob/07ef80820ef1250db82f9544f3fcf7f0f63ccee0/db.go#L352-L362)并[从内存中删除数据](https://github.com/prometheus/tsdb/blob/def6e5a57439cffe7b44a619c05bce4ac513a63e/db.go#L556)。

现在我们还谈到了随着时间的增长，我们如何将较小的块压缩成更大的块。如果headblock足够小以至于不能刷入，我们会看是否有任何需要压缩的块。如果有，我们收集它们，[然后将它们压缩在一起](https://github.com/prometheus/tsdb/blob/07ef80820ef1250db82f9544f3fcf7f0f63ccee0/db.go#L389-L391)。

## 请求路径
现在让我们看看查询是如何工作的。 这稍微复杂一些。
```
q, err := db.Querier(1, 1000)

rem, err := labels.NewRegexpMatcher("a", "b.*")
seriesSet, err := q.Select(rem)

for seriesSet.Next() {
	// Get each Series
	s := seriesSet.At()
	fmt.Println("Labels:", s.Labels())
	fmt.Println("Data:")
	it := s.Iterator()
	for it.Next() {
		ts, v := it.At()
		fmt.Println("ts =", ts, "v =", v)
	}
	if err := it.Err(); err != nil {
		panic(err)
	}
}
```

我们首先获得最小时间和最大时间的查询器，然后根据匹配器选择序列。 对于每个序列，我们检索标签和样本。

现在当[db.Querier(1,1000)](https://github.com/prometheus/tsdb/blob/master/db.go#L634-L666)时，我们将选取传递时间范围数据的所有存储块。当调用q.Select时，我们继续并获取[每个匹配器的postings list](https://github.com/prometheus/tsdb/blob/07ef80820ef1250db82f9544f3fcf7f0f63ccee0/querier.go#L211)并将[它们相交](https://github.com/prometheus/tsdb/blob/07ef80820ef1250db82f9544f3fcf7f0f63ccee0/querier.go#L217)。

现在，我们有了seriesID，我们在这些seriesID上[返回迭代器](https://github.com/prometheus/tsdb/blob/07ef80820ef1250db82f9544f3fcf7f0f63ccee0/querier.go#L156-L170)。现在，每当调用seriesSet.Next()时，我们首先获取该seriesID的[labelset](https://github.com/prometheus/tsdb/blob/07ef80820ef1250db82f9544f3fcf7f0f63ccee0/querier.go#L508-L516)和chunk元数据，然后从[实际的chunk文件中查找块](https://github.com/prometheus/tsdb/blob/07ef80820ef1250db82f9544f3fcf7f0f63ccee0/querier.go#L588)。