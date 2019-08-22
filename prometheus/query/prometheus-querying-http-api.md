title: prometheus查询HTTP API
tags:
  - prometheus
  - 监控
categories:
  - 监控
date: 2019-03-23 18:36:00
---
[原文](https://prometheus.io/docs/prometheus/2.8/querying/api/)基于prometheus2.8

在Prometheus服务器上的`/api/v1`下可以访问当前稳定的HTTP API。 将在该端点下添加任何非破坏性的添加项。

## 格式概述

API响应格式是JSON。每个成功的API请求都会返回2xx状态代码。

到达API处理程序的无效请求将返回JSON错误对象和以下HTTP响应代码之一：
* 400 Bad Request，当参数缺失或参数不正确时
* 422 Unprocessable Entity，当一个表达式不能被执行时[`(RFC4918)`](https://tools.ietf.org/html/rfc4918#page-78).
* 503 Service Unavailable，当请求超时或者异常终止

对于在到达API端点之前发生的错误，可能返回其他非2xx代码。

如果存在不阻止请求执行的错误，则可以返回警告数组。成功收集的所有数据都将在数据字段中返回。

JSON响应信封格式如下：
```
{
  "status": "success" | "error",
  "data": <data>,

  // Only set if status is "error". The data field may still hold
  // additional data.
  "errorType": "<string>",
  "error": "<string>",

  // Only if there were warnings while executing the request.
  // There will still be data in the data field.
  "warnings": ["<string>"]
}
```
输入时间戳可以以RFC3339格式提供，也可以以秒为单位提供给Unix时间戳，可选的小数位数用于亚秒级精度。输出时间戳始终表示为Unix时间戳，以秒为单位。

可以以`[]`结尾的查询参数的名称。

`<series_selector>`占位符指的是Prometheus时间序列选择器，如`http_requests_total`或`http_requests_total{method =~"(GET|POST)"}`，需要进行URL编码。

`<duration>`占位符指的是`[0-9] + [smhdwy]`形式的Prometheus持续时间字符串。例如，`5m`指的是5分钟的持续时间。

`<bool>`占位符引用布尔值（字符串`true`和`false`）。

## 表达式查询

可以在单个时刻或在一段时间内评估查询语言表达。 以下部分描述了每种表达式查询的API端点(endpoint)。

### 即时查询
以下端点在单个时间点评估即时查询：
```
GET /api/v1/query
```
URL查询参数：
* `query=<string>`：Prometheus表达式查询字符串。
* `time=<rfc3339 | unix_timestamp>`：评估时间戳,可选。
* `timeout = <duration>`：评估超时,可选。默认为`-query.timeout`标志的值并受其限制。

如果省略`time`参数，则使用当前服务器时间。

查询结果的`data`部分具有以下格式：
```
{
  "resultType": "matrix" | "vector" | "scalar" | "string",
  "result": <value>
}
```

`<value>`是指查询结果数据，具有不同的格式，具体取决于`resultType`。 请参阅[表达式查询结果格式](https://prometheus.io/docs/prometheus/latest/querying/api/#expression-query-result-formats)。

以下示例在评估表达式`up`在`2015-07-01T20：10：51.781Z`:
```
$ curl 'http://localhost:9090/api/v1/query?query=up&time=2015-07-01T20:10:51.781Z'
{
   "status" : "success",
   "data" : {
      "resultType" : "vector",
      "result" : [
         {
            "metric" : {
               "__name__" : "up",
               "job" : "prometheus",
               "instance" : "localhost:9090"
            },
            "value": [ 1435781451.781, "1" ]
         },
         {
            "metric" : {
               "__name__" : "up",
               "job" : "node",
               "instance" : "localhost:9100"
            },
            "value" : [ 1435781451.781, "0" ]
         }
      ]
   }
}
```

### 区间查询
以下端点在一段时间内评估表达式查询：
```
GET /api/v1/query_range
```
URL查询参数：
* `query=<string>`:Prometheus表达式查询字符串。
* `start=<rfc3339|unix_timestamp>`:开始时间戳。
* `end=<rfc3339|unix_timestamp>`:结束时间戳。
* `step=<duration|float>`:以持续时间格式查询分辨率步长或浮点秒数。
* `timeout=<duration>`：评估超时,可选。默认为`-query.timeout`标志的值并受其限制。

查询结果的`data`部分具有以下格式:
```
{
  "resultType": "matrix",
  "result": <value>
}
```
有关`<value>`占位符的格式，请参阅[范围向量结果格式](https://prometheus.io/docs/prometheus/latest/querying/api/#range-vectors)。

以下示例在30秒范围内评估表达式，查询分辨率为15秒。
```
$ curl 'http://localhost:9090/api/v1/query_range?query=up&start=2015-07-01T20:10:30.781Z&end=2015-07-01T20:11:00.781Z&step=15s'
{
   "status" : "success",
   "data" : {
      "resultType" : "matrix",
      "result" : [
         {
            "metric" : {
               "__name__" : "up",
               "job" : "prometheus",
               "instance" : "localhost:9090"
            },
            "values" : [
               [ 1435781430.781, "1" ],
               [ 1435781445.781, "1" ],
               [ 1435781460.781, "1" ]
            ]
         },
         {
            "metric" : {
               "__name__" : "up",
               "job" : "node",
               "instance" : "localhost:9091"
            },
            "values" : [
               [ 1435781430.781, "0" ],
               [ 1435781445.781, "0" ],
               [ 1435781460.781, "1" ]
            ]
         }
      ]
   }
}
```

## 查询元数据

### 按标签匹配器查找系列
以下端点返回与特定标签集匹配的时间系列列表。
```
GET /api/v1/series
```
URL查询参数：
* `match[]=<series_selector>`:重复的系列选择器参数，用于选择要返回的序列。 必须至少提供一个`match[]`参数。
* `start=<rfc3339|unix_timestamp>`：开始时间戳。
* `end=<rfc3339|unix_timestamp>`：结束时间戳。

查询结果的`data`部分包含一个对象列表，这些对象包含标识每个序列的标签名称/值对。

以下示例返回与选择器`up`或`process_start_time_seconds{job ="prometheus"}`匹配的所有系列：
```
$ curl -g 'http://localhost:9090/api/v1/series?match[]=up&match[]=process_start_time_seconds{job="prometheus"}'
{
   "status" : "success",
   "data" : [
      {
         "__name__" : "up",
         "job" : "prometheus",
         "instance" : "localhost:9090"
      },
      {
         "__name__" : "up",
         "job" : "node",
         "instance" : "localhost:9091"
      },
      {
         "__name__" : "process_start_time_seconds",
         "job" : "prometheus",
         "instance" : "localhost:9090"
      }
   ]
}
```

### 获取标签名称

以下端点返回标签名称列表：
```
GET /api/v1/labels
POST /api/v1/labels
```

JSON响应的`data`部分是字符串标签名称的列表。

这是一个例子:
```
$ curl 'localhost:9090/api/v1/labels'
{
    "status": "success",
    "data": [
        "__name__",
        "call",
        "code",
        "config",
        "dialer_name",
        "endpoint",
        "event",
        "goversion",
        "handler",
        "instance",
        "interval",
        "job",
        "le",
        "listener_name",
        "name",
        "quantile",
        "reason",
        "role",
        "scrape_job",
        "slice",
        "version"
    ]
}
```

### 查询标签值

以下端点返回提供的标签名称的标签值列表：
```
GET /api/v1/label/<label_name>/values
```
JSON响应的`data`部分是字符串标签值的列表。

此示例查询`job`标签的所有标签值：
```
$ curl http://localhost:9090/api/v1/label/job/values
{
   "status" : "success",
   "data" : [
      "node",
      "prometheus"
   ]
}
```

## 表达式查询结果格式

表达式查询可能会在数据部分的`result`属性中返回以下响应值。 `<sample_value>`占位符是数字样本值。JSON不支持特殊的浮点值，例如`NaN`，`Inf`和`-Inf`，因此样本值将作为带引号的JSON字符串而不是原始数字传输。

### 范围向量

范围向量作为结果类型`matrix`返回。相应的`result`属性具有以下格式：
```
[
  {
    "metric": { "<label_name>": "<label_value>", ... },
    "values": [ [ <unix_time>, "<sample_value>" ], ... ]
  },
  ...
]
```

### 即时向量
即时向量作为结果类型`vector`返回。相应的`result`属性具有以下格式：
```
[
  {
    "metric": { "<label_name>": "<label_value>", ... },
    "value": [ <unix_time>, "<sample_value>" ]
  },
  ...
]
```

### 标量

标量结果作为结果类型`scalar`返回。相应的`result`属性具有以下格式：
```
[ <unix_time>, "<scalar_value>" ]
```

### 字符串
字符串结果作为结果类型`string`返回。相应的`result`属性具有以下格式：
```
[ <unix_time>, "<string_value>" ]
```

## 目标机器
以下端点返回Prometheus目标发现的当前状态概述：
```
GET /api/v1/targets
```
活动目标和删除目标都是响应的一部分。`labels`表示重新标记发生后的标签集。`discoveredLabels`表示在发生重新标记之前在服务发现期间检索到的未修改标签。
```
$ curl http://localhost:9090/api/v1/targets
{
  "status": "success",
  "data": {
    "activeTargets": [
      {
        "discoveredLabels": {
          "__address__": "127.0.0.1:9090",
          "__metrics_path__": "/metrics",
          "__scheme__": "http",
          "job": "prometheus"
        },
        "labels": {
          "instance": "127.0.0.1:9090",
          "job": "prometheus"
        },
        "scrapeUrl": "http://127.0.0.1:9090/metrics",
        "lastError": "",
        "lastScrape": "2017-01-17T15:07:44.723715405+01:00",
        "health": "up"
      }
    ],
    "droppedTargets": [
      {
        "discoveredLabels": {
          "__address__": "127.0.0.1:9100",
          "__metrics_path__": "/metrics",
          "__scheme__": "http",
          "job": "node"
        },
      }
    ]
  }
}
```

## 规则

`/rules`API端点返回当前加载的警报和记录规则列表。此外，它还返回由每个警报规则的Prometheus实例触发的当前活动警报。

**由于`/rules`端点相当新，它没有与总体API v1相同的稳定性保证。**
```
GET /api/v1/rules
```
```
$ curl http://localhost:9090/api/v1/rules

{
    "data": {
        "groups": [
            {
                "rules": [
                    {
                        "alerts": [
                            {
                                "activeAt": "2018-07-04T20:27:12.60602144+02:00",
                                "annotations": {
                                    "summary": "High request latency"
                                },
                                "labels": {
                                    "alertname": "HighRequestLatency",
                                    "severity": "page"
                                },
                                "state": "firing",
                                "value": 1
                            }
                        ],
                        "annotations": {
                            "summary": "High request latency"
                        },
                        "duration": 600,
                        "health": "ok",
                        "labels": {
                            "severity": "page"
                        },
                        "name": "HighRequestLatency",
                        "query": "job:request_latency_seconds:mean5m{job=\"myjob\"} > 0.5",
                        "type": "alerting"
                    },
                    {
                        "health": "ok",
                        "name": "job:http_inprogress_requests:sum",
                        "query": "sum(http_inprogress_requests) by (job)",
                        "type": "recording"
                    }
                ],
                "file": "/rules.yaml",
                "interval": 60,
                "name": "example"
            }
        ]
    },
    "status": "success"
}
```

## 告警

`/alerts`端点返回所有活动警报的列表。


**由于`/alerts`端点相当新，它没有与总体API v1相同的稳定性保证。**

```
GET /api/v1/alerts
```
```
$ curl http://localhost:9090/api/v1/alerts

{
    "data": {
        "alerts": [
            {
                "activeAt": "2018-07-04T20:27:12.60602144+02:00",
                "annotations": {},
                "labels": {
                    "alertname": "my-alert"
                },
                "state": "firing",
                "value": 1
            }
        ]
    },
    "status": "success"
}
```

## 查询目标机器元数据

以下端点返回有关目标正在刮取的指标标准的元数据。**本接口是实验性的**，将来可能会发生变化。
```
GET /api/v1/targets/metadata
```

URL查询参数：
* `match_target=<label_selectors>`:通过标签集匹配目标的标签选择器。如果留空则选择所有目标。
* `metric=<string>`：用于检索元数据的指标名称。如果留空，则检索所有指标标准元数据。
* `limit=<number>`:要匹配的最大目标数。

查询结果的`data`部分包含一个包含指标元数据和目标标签集的对象列表。

以下示例从前两个目标机器返回`go_goroutines`指标的所有元数据条目，标签为`job="prometheus"`。

```
curl -G http://localhost:9091/api/v1/targets/metadata \
    --data-urlencode 'metric=go_goroutines' \
    --data-urlencode 'match_target={job="prometheus"}' \
    --data-urlencode 'limit=2'
{
  "status": "success",
  "data": [
    {
      "target": {
        "instance": "127.0.0.1:9090",
        "job": "prometheus"
      },
      "type": "gauge",
      "help": "Number of goroutines that currently exist.",
      "unit": ""
    },
    {
      "target": {
        "instance": "127.0.0.1:9091",
        "job": "prometheus"
      },
      "type": "gauge",
      "help": "Number of goroutines that currently exist.",
      "unit": ""
    }
  ]
}
```

以下示例返回标签`instance="127.0.0.1:9090"`的所有目标的所有指标标准的元数据。
```
curl -G http://localhost:9091/api/v1/targets/metadata \
    --data-urlencode 'match_target={instance="127.0.0.1:9090"}'
{
  "status": "success",
  "data": [
    // ...
    {
      "target": {
        "instance": "127.0.0.1:9090",
        "job": "prometheus"
      },
      "metric": "prometheus_treecache_zookeeper_failures_total",
      "type": "counter",
      "help": "The total number of ZooKeeper failures.",
      "unit": ""
    },
    {
      "target": {
        "instance": "127.0.0.1:9090",
        "job": "prometheus"
      },
      "metric": "prometheus_tsdb_reloads_total",
      "type": "counter",
      "help": "Number of times the database reloaded block data from disk.",
      "unit": ""
    },
    // ...
  ]
}
```

## Alertmanagers

以下端点返回Prometheus alertmanager发现的当前状态概述：
```
GET /api/v1/alertmanagers
```
活动和丢弃的Alertmanagers都是响应的一部分。
```
$ curl http://localhost:9090/api/v1/alertmanagers
{
  "status": "success",
  "data": {
    "activeAlertmanagers": [
      {
        "url": "http://127.0.0.1:9090/api/v1/alerts"
      }
    ],
    "droppedAlertmanagers": [
      {
        "url": "http://127.0.0.1:9093/api/v1/alerts"
      }
    ]
  }
}
```

## 状态
以下状态端点显示当前的Prometheus配置。
### config
以下端点返回当前加载的配置文件：
```
GET /api/v1/status/config
```

配置作为转储的YAML文件返回。由于YAML库的限制，不包括YAML注释。
```
$ curl http://localhost:9090/api/v1/status/config
{
  "status": "success",
  "data": {
    "yaml": "<content of the loaded config file in YAML>",
  }
}
```

### flags
以下端点返回Prometheus配置的标志值：
```
GET /api/v1/status/flags
```
所有值都以“字符串”的形式出现。
```
$ curl http://localhost:9090/api/v1/status/flags
{
  "status": "success",
  "data": {
    "alertmanager.notification-queue-capacity": "10000",
    "alertmanager.timeout": "10s",
    "log.level": "info",
    "query.lookback-delta": "5m",
    "query.max-concurrency": "20",
    ...
  }
}
```
**添加于v2.2**

## TSDB管理API
这些是为高级用户公开数据库功能的API。除非设置了`--web.enable-admin-api`，否则不会启用这些API。

我们还公开了一个gRPC API，其定义可以在[这里](https://github.com/prometheus/prometheus/blob/master/prompb/rpc.proto)找到。**这是实验性的，将来可能会发生变化。**

### 快照
快照会将所有当前数据的快照创建到TSDB数据目录下的`snapshots/<datetime>-<rand>`中，并将该目录作为响应返回。 它可以选择跳过仅存在于头块中但尚未压缩到磁盘的快照数据。
```
POST /api/v1/admin/tsdb/snapshot?skip_head=<bool>
```
```
$ curl -XPOST http://localhost:9090/api/v1/admin/tsdb/snapshot
{
  "status": "success",
  "data": {
    "name": "20171210T211224Z-2be650b6d019eb54"
  }
}
```
快照现在位于`<data-dir>/snapshots/20171210T211224Z-2be650b6d019eb54`
**添加与v2.1**

### 删除序列
DeleteSeries删除时间范围内所选序列的数据。实际数据仍然存在于磁盘上，并在将来的压缩中清除，或者可以通过点击Clean Tombstones端点来明确清理。

如果成功，则返回`204`。

```
POST /api/v1/admin/tsdb/delete_series
```
URL查询参数：
* `match[]=<series_selector>`:选择要删除的系列的重复标签匹配器参数。必须至少提供一个`match[]`参数。
* `start=<rfc3339|unix_timestamp>`:开始时间戳,可选，默认为最短可能时间。
* `end=<rfc3339|unix_timestamp>`:结束时间戳,可选，默认为最长可能时间。

不提及开始和结束时间将清除数据库中匹配系列的所有数据。

例子:
```
$ curl -X POST \
  -g 'http://localhost:9090/api/v1/admin/tsdb/delete_series?match[]=up&match[]=process_start_time_seconds{job="prometheus"}'
```
**添加于v2.1**

### 清洁墓碑
CleanTombstones从磁盘中删除已删除的数据并清理现有的逻辑删除。这可以在删除系列后使用以释放空间。

如果成功，则返回`204`。
```
POST /api/v1/admin/tsdb/clean_tombstones
```
这不需要参数或正文。

```
$ curl -XPOST http://localhost:9090/api/v1/admin/tsdb/clean_tombstones
```

**添加于v2.1**
