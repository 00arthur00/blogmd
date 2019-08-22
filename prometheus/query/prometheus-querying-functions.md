title: prometheus查询-函数
tags:
  - prometheus
  - 监控
categories:
  - 监控
date: 2019-03-23 15:58:00
---
[原文](https://prometheus.io/docs/prometheus/2.8/querying/functions/)基于prometheus2.8

一些函数具有默认参数，例如`year(v=vector(time()) instant-vector)`。 这意味着有一个参数v是一个即时向量，如果没有提供，它将默认为表达式向量的值`vector(time())`。

* abs()

`abs(v instant-vector)`返回输入向量，所有样本值都转换为其绝对值。

* absent()

`absent(v instant-vector)`如果传递给它的向量具有任何元素，则返回空向量;如果传递给它的向量没有元素，则返回值为1的1元素向量。

这对于在给定指标名称和标签组合不存在时间序列时发出警报非常有用。
```
absent(nonexistent{job="myjob"})
# => {job="myjob"}

absent(nonexistent{job="myjob",instance=~".*"})
# => {job="myjob"}

absent(sum(nonexistent{job="myjob"}))
# => {}
```
在第二个例子中，`absent()`试图从输入向量中导出1元素输出向量的标签。

* ceil()
`ceil(v instant-vector)`将v中所有元素的样本值舍入到最接近的整数。

* changes()

对于每个输入时间系列，`change(v range-vector)`将返回其值在所提供的时间范围内更改的次数作为即时向量。


* clamp_max()

`clamp_max(v instant-vector,max scalar)`钳制v中所有元素的样本值，使其上限为max。

* clamp_min()

`clamp_min(v instant-vector, min scalar)`钳制v中所有元素的样本值，使其下限为min。

* day_of_month()

`day_of_month(v=vector(time()) instant-vector) `返回UTC中每个给定时间的月中的某天。 返回值为1到31。

* day_of_week()

`day_of_week(v=vector(time()) instant-vector)`返回UTC中每个给定时间的星期几。 返回值为0到6，其中0表示星期日等。

* days_in_month()

`days_in_month(v=vector(time()) instant-vector)`返回UTC中每个给定时间的月中天数。 返回值为28到31。

* delta()

`delta(v range-vector)`计算范围向量v中每个时间系列元素的第一个和最后一个值之间的差值，返回具有给定增量和等效标签的即时向量。 delta被外推以覆盖范围向量选择器中指定的全时间范围，因此即使样本值都是整数，也可以获得非整数结果。

以下示例表达式返回现在和2小时之前CPU温度的差异：
```
delta(cpu_temp_celsius{host="zeus"}[2h])

```
`delat`仅能被用于gauge类型。

* deriv()

`deriv(v range-vector)`使用[简单线性回归](https://en.wikipedia.org/wiki/Simple_linear_regression)计算范围向量v中时间序列的每秒导数。

`deriv`仅能被用于gauge类型。

* exp()

`exp(v instant-vector)`计算v中所有元素的指数函数。特殊情况是：
```
Exp(+Inf) = +Inf
Exp(NaN) = NaN
```

* floor()
`floor(v instant-vector)`将v中所有元素的样本值舍入为最接近的整数。

* histogram_quantile()

`histogram_quantile(φ float, b instant-vector)`从[直方图](https://prometheus.io/docs/concepts/metric_types/#histogram)的桶`b`计算φ-分位数(0≤φ≤1)。(有关φ-分位数的详细解释和直方图指标类型的使用，请参见[直方图和摘要](https://prometheus.io/docs/practices/histograms)。)`b`中的样本是每个桶中的观察计数。每个样本必须具有标签`le`，其中标签值表示桶的包含上限。（没有这种标签的样本会被忽略。）[直方图指标](https://prometheus.io/docs/concepts/metric_types/#histogram)类型自动提供带有_bucket后缀和相应标签的时间序列。

使用`rate()`函数指定分位数计算的时间窗口。

示例：直方图指标为http_request_duration_seconds，要计算过去10分钟内请求持续时间的第90个百分位数，请使用以下表达式：
```
histogram_quantile(0.9, rate(http_request_duration_seconds_bucket[10m]))
```
在`http_request_duration_seconds`中为每个标签组合计算分位数。要聚合,请使用`sum()`聚合器包裹在`rate()`函数。由于`histogram_quantile()`需要`le`标签，因此必须将其包含在`by`子句中。 以下表达式按`job`聚合第90个百分点：
```
histogram_quantile(0.9, sum(rate(http_request_duration_seconds_bucket[10m])) by (job, le))
```
要聚合所有内容，请仅指定le标签：
```
histogram_quantile(0.9, sum(rate(http_request_duration_seconds_bucket[10m])) by (le))
```
`histogram_quantile()`函数通过假设桶内的线性分布来插值分位数值。最高桶必须具有+ Inf的上限。(否则，返回NaN)如果分位数位于最高桶中，则返回第二个最高桶的上限。如果该桶的上限大于0，则假设最低桶的下限为0.在这种情况下，在该桶内应用通常的线性插值。 否则，对于位于最低桶中的分位数，返回最低桶的上限。

如果`b`包含少于两个桶，则返回`NaN`。对于`φ<0`，返回`-Inf`。对于`φ> 1`，返回`+Inf`。

* holt_winters()

`holt_winters(v range-vector,sf scalar,tf scalar)`根据v中的范围生成时间序列的平滑值。平滑因子sf越低，对旧数据的重要性越高。趋势因子tf越高，则考虑的数据趋势越多。sf和tf都必须介于0和1之间。

`holt_winters`只能用于gauge类型。

* hour()

`hour(v=vector(time()) instant-vector)`以UTC为单位返回每个给定时间的小时。 返回值为0到23。

* idelta()

`idelta(v range-vector)`

`idelta(v range-vector)`计算范围向量v中最后两个样本之间的差异，返回具有给定增量和等效标签的即时向量。

`idelta`只能被用于gauge类型。

* increase()

`increase(v range-vector)`计算范围向量中时间序列的增加。单调性中断（例如由于目标重启而导致的计数器重置）会自动调整。increase外推以覆盖范围向量选择器中指定的全时间范围，因此即使计数器仅以整数增量增加，也可以获得非整数结果。

以下示例表达式返回范围向量中每个时间系列在过去5分钟内测量的HTTP请求数：
```
increase(http_requests_total{job="api-server"}[5m])
```
`increase`只应与counter一起使用。它是rate(v)的语法糖乘以指定时间范围窗口下的秒数，应该主要用于人类可读性。在记录规则中使用rate，以便每秒一致地跟踪增量。

* irate()

`irate(v range-vector)`基于最后两个数据点计算范围向量中时间序列的每秒即时增长率。单调性中断（例如由于目标重启而导致的计数器重置）会自动调整。

以下示例表达式返回范围向量中每个时间序列的两个最新数据点的最多5分钟的HTTP请求的每秒速率：
```
irate(http_requests_total{job="api-server"}[5m])
```
只应在绘制易失性及快速移动计数器时使用`irate`。警报和缓慢移动计数器的使用`rate`，因为速率的简短更改可以重置`FOR子`句，并且难以阅读完全由稀有峰值组成的图形。

注意，当将`irate()`与聚合运算符(例如sum())或随时间聚合的函数(任何以`_over_time`结尾的函数)组合时，请始终首先采用`irate()`，然后进行聚合。否则，当目标重新启动时，`irate()`无法检测计数器重置。

* label_join()

对于`v`中的每个时间序列,`label_join(v instant-vector,dst_label string,separator string,src_label_1 string,src_label_2 string，...)`使用`separator`连接所有`src_labels`的所有值，并返回包含连接的标签`dst_label`的时间序列值。 此函数中可以有任意数量的src_labels。

此示例将返回一个向量，每个时间序列都有一个foo标签，其中添加了值a,b,c：
```
label_join(up{job="api-server",src1="a",src2="b",src3="c"}, "foo", ",", "src1", "src2", "src3")
```
* label_replace()

对于`v`中的每个时间序列，`label_replace(v instant-vector, dst_label string, replacement string, src_label string, regex string)`将正则表达式`regex`与标签`src_label`相匹配。如果匹配，则返回时间序列，标签`dst_label`替换为`replacement`。`$1`替换为第一个匹配的子组，`$2`替换为第二个等。如果正则表达式不匹配，则返回时间序列不变。

此示例将返回一个向量，每个时间序列都有一个`foo`标签，其值为`a`：
```
label_replace(up{job="api-server",service="a:c"}, "foo", "$1", "service", "(.*):.*")
```

* ln()

`ln(v instant-vector)`计算v中所有元素的自然对数。特殊情况是：
```
ln(+Inf) = +Inf
ln(0) = -Inf
ln(x < 0) = NaN
ln(NaN) = NaN
```

* log2()

`log2(v instant-vector)`计算v中所有元素的二进制对数。特殊情况等同于`ln`中的特殊情况。

* log10()

`log10(v instant-vector)`计算v中所有元素的十进制对数。特殊情况与`ln`中的相同。

* minute()

`minute(v=vector(time()) instant-vector)`返回UTC中每个给定时间的小时分钟。 返回值为0到59。

* month()

`month(v=vector(time()) instant-vector)`返回UTC中每个给定时间的一年中的月份。 返回值为1到12，其中1表示1月等。

* predict_linear()

`predict_linear(v range-vector,t scalar)`使用[简单线性回归](https://en.wikipedia.org/wiki/Simple_linear_regression)，基于范围向量v预测从现在开始的时间序列t秒的值。


`predict_linear`只应与gauge一起使用。

* rate()

`rate(v range-vector)`计算范围向量中时间序列的每秒平均增长率。单调性中断（例如由于目标重启而导致的计数器重置）会自动调整。 此外，计算推断到时间范围的末端，允许错过刮擦或刮擦循环与范围的时间段的不完美对齐。

以下示例表达式返回范围向量中每个时间系列在过去5分钟内测量的每秒HTTP请求率：
```
rate(http_requests_total{job="api-server"}[5m])
```
`rate`应仅用于计数器。它最适用于警报和缓慢移动计数器的图形。

注意，当将`rate()`与聚合运算符(例如`sum()`)或随时间聚合的函数（任何以`_over_time`结尾的函数）组合时，始终首先采用`rate()`，然后聚合。否则，当目标重新启动时，`rate()`无法检测计数器重置。

* resets()

对于每个输入时间序列，`reset(v range-vector)`将提供的时间范围内的计数器复位数作为即时向量返回。两个连续样本之间的值的任何减少都被解释为计数器重置。

`resets`仅被用于counters.

* round()

`round(v instant-vector,to_nearest = 1 scalar)`将v中所有元素的样本值舍入为最接近的整数。通过四舍五入解决关系。可选的`to_nearest`参数允许指定样本值应舍入的最近倍数。 这个倍数也可能是一个分数。

* scalar()

给定单元素输入向量，`scalar(v instant-vector)`将该单个元素的样本值作为标量返回。如果输入向量不具有恰好一个元素，则标量将返回NaN。

* sort()

`sort(v instant-vector)`按升序返回按其样本值排序的向量元素。

* sort_desc()

与`sort`相同，但按降序排序。

* sqrt()

`sqrt(v instant-vector)`计算v中所有元素的平方根。

* time()

`time()`返回自1970年1月1日UTC以来的秒数。请注意，这实际上并不返回当前时间，而是返回计算表达式的时间。

* timestamp()

`timestamp(v instant-vector)`返回给定向量的每个样本的时间戳，作为自1970年1月1日UTC以来的秒数。

此函数于Prometheus 2.0中添加。

* vector()

`vector(s scalar)`将标量`s`作为没有标签的向量返回。

* year()

`year(v=vector(time()) instant-vector)`返回UTC中每个给定时间的年份。

* `<aggregation>`_over_time()

以下函数允许聚合给定范围向量的每个系列随时间的变化并返回具有每系列聚合结果的即时向量：

* avg_over_time(range-vector):指定区间内所有点的平均值。
* min_over_time(range-vector):指定时间间隔内所有点的最小值。
* max_over_time(range-vector):指定时间间隔内所有点的最大值。
* sum_over_time(range-vector):指定时间间隔内所有值的总和。
* count_over_time(range-vector):指定时间间隔内所有值的计数。
* quantile_over_time(scalar, range-vector):指定间隔内的值的φ-分位数（0≤φ≤1）。
* stddev_over_time(range-vector):指定区间内值的总体标准方差。
* stdvar_over_time(range-vector): 指定区间中值的总体标准差。

请注意，即使值在整个时间间隔内的间隔不均匀，指定时间间隔内的所有值在聚合中都具有相同的权重。