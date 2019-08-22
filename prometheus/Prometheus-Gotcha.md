title: Prometheus Gotcha
tags:
  - prometheus
  - 监控
categories:
  - 监控
date: 2019-04-03 13:15:00
---
## 问题描述
* promethues版本2.6.0
* 数据写入：我们自己完成了一个router用于从接收数据，然后写入influxdb。写入的使用了proetheus的remote write API进行的二次发开(未配置prometheus的remote write)。
* 数据读取：prometheus配置了remote read可以从influxdb拉数据进行展示
* 其中一段时间prometheus配了一些target拉取数据存入了tsdb，之后又把这些target去掉了，同事router也在一直写入，未中断。Target拉取的数据和写入influxDB的数据是一样的。
* 过了几天时间用instant-vector方式查询target相关的查数据查不到了

## Root Cause
### toubleshooting
1. 通过influxdb客户端发现数据一直写入，说明router和influxdb都没有问题。
2. 怀疑是prometheus的external label配置问题，将external添加、修改、删除均不起作用
3. 怀疑是prometheus版本问题，将新prometheus 2.6.0及配置文件放入新文件夹(newFolder)，influxdb中的数据可以查到了。
4. 对newFolder和原文件夹，发现tsdb目录的内容有区别
5. 删除原文件夹中的tsdb数据，influxdb中的数据也可以查到了
6. 问题定位到tsdb存储，经和别的同学了解原来有一段时间通过target存入了相同的数据到tsdb
7. 怀疑是只读取了tsdb信息，但由于tsdb存入的时间太早了，所以没有搜索到。

### tsdb、prometheus remote read初始化及配置
通过阅读源码，得到如下流程：
1. tsdb初始化：创建tsdb.ReadyStorage对象，保存tsdb client并设置`startTimeMargin`为2倍的tsdb块保存时间最小值(cfg.tsdb.MinBlockDuration,此参数从命令行配置，默认2小时,此时startTimeMargin为4小时)。
* startTimeMargin用于设置tsdb的启动时间(starttime)，如果tsdb中没有数据，则未当前时间+startTimeMargin作为启动时间。
* 否则直接使用tsdb中的块的最小时间作为启动时间

2. remote read初始化: 用prometheus的remote read及tsdb的tarttime，进行初始化。
* 其中比较关键的是[`read_recent`](https://prometheus.io/docs/prometheus/2.6/configuration/configuration/#remote_read)配置默认为false。
* 如果为true，则会进行remote read请求
* 如果为false，则会进行filter操作，query的时间大于tsdb starttime则，不采用remote read
``` golang
func PreferLocalStorageFilter(next storage.Queryable, cb startTimeCallback) storage.Queryable {
	return storage.QueryableFunc(func(ctx context.Context, mint, maxt int64) (storage.Querier, error) {
		localStartTime, err := cb()
		if err != nil {
			return nil, err
		}
		cmaxt := maxt
		// Avoid queries whose timerange is later than the first timestamp in local DB.
		if mint > localStartTime {
			return storage.NoopQuerier(), nil
		}
		// Query only samples older than the first timestamp in local DB.
		if maxt > localStartTime {
			cmaxt = localStartTime
		}
		return next.Querier(ctx, mint, cmaxt)
	})
}
```

## root cause
* 当tsdb中有数据时(存在数据块)，则启动时间为存入数据块的最小时间，query到来时，当前时间大于启动时间，直接从tsdb中读取，而tsdb中数据太老了读取不到。
* 删除tsdb数据后，query到来时，`startTimeMargin`为4小时，启动时间为query到来时间4小时后,当前时间小于启动时间，会通过remote read查询。


## Solution
1. 保证prometheus不配置target，即不往tsdb里写入数据，此时启动时间永远为4个小时以后，remote read始终执行。
2. 配置prometheus的read_recent为true，此时remote read也会执行，但tsdb中如果有数据也会取出来，merge之后产生结果。

项目中大部分没有配置target，建议采用2中的方式，不会有overhead。