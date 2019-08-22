title: prometheus查询入门
tags:
  - prometheus
  - 监控
categories:
  - 监控
date: 2019-03-23 10:05:00
---
[原文](https://prometheus.io/docs/prometheus/2.8/querying/basics/)基于prometheus2.8

Prometheus提供了一种称为PromQL（Prometheus查询语言）的功能查询语言，允许用户实时选择和聚合时间序列数据。表达式的结果可以显示为图形，在Prometheus的表达式浏览器中显示为表格数据，或者由外部系统通过[HTTP API](https://prometheus.io/docs/prometheus/latest/querying/api/)使用。

## Examples
本文档仅供参考。对于学习，从几个[例子](https://prometheus.io/docs/prometheus/latest/querying/examples/)开始可能更容易。

## 表达式语言数据类型
在Prometheus的表达式语言中，表达式或子表达式可以评估为以下四种类型之一：
* **即时向量**(instant vector) - 一组包含每个时间序列的单个样本的时间序列，它们共享相同的时间戳
* **范围向量**(range vector) - 一组时间系列，包含每个时间序列随时间变化的一系列数据点
* **标量**(scalar) - 一个简单的数字浮点值
* **字符串**(String) - 一个简单的字符串值; 目前尚未使用

根据用例（例如，当绘图与显示表达式的输出时），只有自用户指定的表达式部分类型是合法的。例如，返回即时向量的表达式是唯一可以直接绘制的类型。

## 字面值(literals)

### 字符串字面值(string literals)
字符串可以用单引号，双引号或反引号指定为文字。

PromQL遵循与Go相同的转义规则。 在单引号或双引号中，反斜杠开始转义序列，后面可以跟a,b,f,n,r,t,v或\。可以使用八进制（\nnn）或十六进制（\xnn，\unnnn和\Unnnnnnnn）提供特定字符。

在反引号内没有处理转义。与Go不同，Prometheus不会丢弃反引号中的换行符。

例子:
```
"this is a string"
'these are unescaped: \n \\ \t'
`these are not unescaped: \n ' " \t`
```

### 浮点字面值(float literals)
标量浮点值可以字面上写为\[-]\(digits)\[.(digits)]格式的数字。
```
-2.43
```
## 时间序列选择器
### 即时矢量选择器
即时向量选择器允许为给定时间戳（即时）选择一组时间序列和单个样本值：在最简单的形式中，仅指定指标名称。这会生成包含具有此指标名称的所有时间序列的元素的即时向量。

此示例选择具有http_requests_total指标名称的所有时间系列：
```
http_requests_total
```
通过附加一组匹配花括号\({})的标签，可以进一步过滤这些时间序列。

此示例仅选择具有http_requests_total指标名称的时间系列，该名称也将作业标签设置为prometheus，并将其组标签设置为canary：
```
http_requests_total{job="prometheus",group="canary"}
```
还可以对标签值进行负匹配，或者将标签值与正则表达式进行匹配。存在以下标签匹配运算符：
* =：选择与提供的字符串完全相同的标签。
* !=：选择不等于提供的字符串的标签。
* =~：选择正则表达式匹配提供的字符串（或子字符串）的标签。
* !~：选择不与提供的字符串（或子字符串）匹配的标签。

例如，这将为staging,testing和development环境以及除GET之外的HTTP方法选择所有http_requests_total时间序列。
```
http_requests_total{environment=~"staging|testing|development",method!="GET"}
```
与空标签值匹配的标签匹配器也会选择根本没有特定标签集的所有时间序列。正则表达式匹配是完全锚定的。可以为同一标签名称提供多个匹配器。

向量选择器必须指定一个名称或至少一个与空字符串不匹配的标签匹配器。 以下表达式是非法的：
```
{job=~".*"} # Bad!
```
相反，这些表达式是有效的，因为它们都有一个与空标签值不匹配的选择器。
```
{job=~".+"}              # Good!
{job=~".*",method="get"} # Good!
```
通过匹配内部__name__标签将标签匹配器应用于指标名称。例如，表达式http_requests_total等效于{\__name__ ="http_requests_total"}。也可以使用除=(!=,=~,!~)之外的匹配器。以下表达式选择名称以job:开头的所有指标:
```
{__name__=~"job:.*"}
```
Prometheus正则表达式使用[RE2语法](https://github.com/google/re2/wiki/Syntax)。

### 范围向量选择器
范围向量字面值像即时向量字面值一样工作，除了它们从当前时刻选择一系列样本。在语法上，范围持续时间附加在向量选择器末尾的方括号(\[])中，以指定应为每个结果范围向量元素提取多长时间值。

持续时间指定为数字，紧接着是以下单位之一：
* s - seconds
* m - minutes
* h - hours
* d - days
* w - weeks
* y - years

在此示例中，我们选择在过去5分钟内为指标名称为http_requests_total且job标签设置为prometheus的所有时间序列记录的所有值：
```
http_requests_total{job="prometheus"}[5m]
```

### 偏移修改器
偏移修改器允许更改查询中各个即时和范围向量的时间偏移。

例如，以下表达式返回过去相对于当前查询评估时间5分钟的http_requests_total值：
```
http_requests_total offset 5m
```
请注意，偏移修改器始终需要紧跟选择器，即以下内容是正确的：
```
sum(http_requests_total{method="GET"} offset 5m) // GOOD.
```
但是下面的是不正确的:
```
sum(http_requests_total{method="GET"}) offset 5m // INVALID.
```
偏移选择器同样适用范围向量。下面返回一周前的五分钟速率:
```
rate(http_requests_total[5m] offset 1w)
```

## 子查询
子查询允许您针对给定范围和分辨率(resolution)运行即时查询。子查询的结果是范围向量。
语法:
```
<instant_query> '[' <range> ':' [<resolution>] ']' [ offset <duration> ]
```
* resolution是可选的。 

## 运算符
Prometheus支持许多二进制和聚合运算符。这些在[表达式语言运算符](https://prometheus.io/docs/prometheus/2.8/querying/operators/)页面中有详细描述。

## 函数
Prometheus支持多种操作数据的功能。这些在[表达式语言函数](https://prometheus.io/docs/prometheus/2.8/querying/functions/)页面中有详细描述。

## 陷阱

### 陈旧
运行查询时，采样数据的时间戳独立于实际当前时间序列数据。这主要是为了支持聚合（总和，平均等）的情况，其中多个聚合时间序列在时间上不完全对齐。由于它们的独立性，Prometheus需要在每个相关时间序列的时间戳上分配值。它只需在此时间戳之前采用最新的样本即可。

如果目标刮擦或规则评估不再返回先前存在的时间序列的样本，则该时间序列将被标记为陈旧。如果目标被移除，之前很快就会将其先前返回的时间序列标记为陈旧。

如果在时间序列标记为过时后在采样时间戳处评估查询，则不会为该时间系列返回任何值。如果随后在该时间序列中摄取新样本，它们将照常返回。

如果在采样时间戳前5分钟未找到任何样本（默认情况下），则此时间点不返回该时间序列的值。这实际上意味着时间序列在其最新收集的样本超过5分钟或标记为陈旧之后从图表“消失”。

对于在其刮擦中包含时间戳的时间序列，不会标记陈旧性。在这种情况下，仅应用5分钟的阈值。

### 避免慢查询和过载
如果查询需要对大量数据进行操作，则绘制图表可能会超时或使服务器或浏览器过载。 因此，在构建对未知数据的查询时，始终在Prometheus表达式浏览器的表格视图中开始构建查询，直到结果集看起来合理（最多数百个，而不是数千个时间序列）。 只有在您充分过滤或汇总数据后，才能切换到图表模式。 如果表达式仍然需要很长时间来绘制ad-hoc图形，请通过[录制规则](https://prometheus.io/docs/prometheus/2.8/configuration/recording_rules/#recording-rules)预先录制它。

这与Prometheus的查询语言尤其相关，其中像api_http_requests_total这样的简单指标名称选择器可以扩展到具有不同标签的数千个时间序列。还要记住，即使输出只是少量的时间序列，聚合在许多时间序列上的表达式也会在服务器上产生负载。这类似于在关系数据库中对列的所有值求和的速度很慢，即使输出值只是一个数字。
