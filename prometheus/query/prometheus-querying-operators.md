title: prometheus查询-运算符
tags:
  - prometheus
categories:
  - 监控
date: 2019-03-23 14:19:00
---
[原文](https://prometheus.io/docs/prometheus/2.8/querying/operators/)基于prometheus2.8

## 二元运算符

Prometheus的查询语言支持基本的逻辑和算术运算符。对于两个即时向量之间的操作，可以修改[匹配行为](https://prometheus.io/docs/prometheus/2.8/querying/operators/#vector-matching)。

### 算术二元运算符

Prometheus中存在以下二进制算术运算符：
* \+ (addition)
* \- (subtraction)
* \* (multiplication)
* / (division)
* % (modulo)
* ^ (power/exponentiation)

二进制算术运算符定义在标量/标量，矢量/标量和矢量/矢量值对之间。

**在两个标量之间**，行为是显而易见的：它们评估另一个标量，这是运算符应用于两个标量操作数的结果。

**在即时向量和标量之间**，将运算符应用于向量中的每个数据样本的值。例如,如果时间序列即时向量乘以2，则结果是另一个向量，其中原始向量的每个样本值乘以2。

**在两个即时向量之间**，二进制算术运算符应用于左侧向量中的每个条目及其右侧向量中的匹配元素。 结果将传播到结果向量中，并删除指标名称。右侧向量中没有匹配条目的条目不是结果的一部分。

### 比较二元运算符

Prometheus中存在以下二进制比较运算符：
* == (equal)
* != (not-equal)
* \> (greater-than)
* < (less-than)
* \>= (greater-or-equal)
* <= (less-or-equal)

比较运算符定义在标量/标量，矢量/标量和矢量/矢量值对之间。默认情况下会过滤。可以通过在运算符之后提供bool来修改它们的行为，这将为值返回0或1而不过滤。

**在两个标量之间**，必须提供bool修饰符，并且这些运算符会产生另一个标量，即0（false）或1（true），具体取决于比较结果。

**在即时向量和标量之间**，将这些运算符应用于向量中的每个数据样本的值，并且从结果向量中删除比较结果为false的向量元素。如果提供了bool修饰符，则将被删除的向量元素的值为0，而将保留的向量元素的值为1。

**在两个即时向量之间**，这些运算符默认表现为过滤器，应用于匹配条目。表达式不正确或在表达式的另一侧找不到匹配项的向量元素将从结果中删除，而其他元素将传播到具有其原始（左侧）度量标准名称的结果向量中 标签值。 如果提供了bool修饰符，则已经删除的向量元素的值为0，而保留的向量元素的值为1，左侧标签值为1。

**在两个即时向量之间**，这些运算符默认表现为过滤器，应用于匹配条目。表达式不为true或在表达式的另一侧找不到匹配项的向量元素将从结果中删除，而其他元素将生成到到具有其原始（左侧）指标名称和标签值的结果向量。如果提供了bool修饰符，则已经删除的向量元素的值为0，而保留的向量元素的值为1，左侧标签值为1。

### 逻辑/设置二元运算符
这些逻辑/集合二元运算符仅在即时向量之间定义：
* and (intersection)
* or (union)
* unless (complement)

**vector1 and vector2**产生一个由vector1元素组成的向量，其中vector2中的元素具有完全匹配的标签集。其他元素被删除。度量标准名称和值从左侧向量取。

**vector1 or vector2**产生的向量包含vector1的所有原始元素（标签集+值），另外还包含vector2与vector1中没有匹配标签集的所有元素。

**vector1 unless vector2**产生一个向量，该向量由vector1的元素组成，但vector2中没有元素与精确匹配的标签集。两个向量中的所有匹配元素都被删除。

## 向量匹配
向量之间的操作尝试在左侧的每个条目的右侧向量中找到匹配元素。 匹配行为有两种基本类型：一对一和多对一/一对多。

### 一对一向量匹配

一对一从操作的每一侧找到一对唯一条目。 在默认情况下，这是格式为`vector1 <operator> vector2`之后的操作。如果两个条目具有完全相同的标签集和相应的值，则它们匹配。`ignoring`关键字允许在匹配时忽略某些标签，而on关键字允许将所考虑的标签集减少到提供的列表：
```
<vector expr> <bin-op> ignoring(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) <vector expr>
```

例子，输入:
```
method_code:http_errors:rate5m{method="get", code="500"}  24
method_code:http_errors:rate5m{method="get", code="404"}  30
method_code:http_errors:rate5m{method="put", code="501"}  3
method_code:http_errors:rate5m{method="post", code="500"} 6
method_code:http_errors:rate5m{method="post", code="404"} 21

method:http_requests:rate5m{method="get"}  600
method:http_requests:rate5m{method="del"}  34
method:http_requests:rate5m{method="post"} 120
```

例子，请求：
```
method_code:http_errors:rate5m{code="500"} / ignoring(code) method:http_requests:rate5m
```

这将返回一个结果向量，其中包含每个方法的状态代码为500的HTTP请求部分，在过去的5分钟内进行测量。没有`ignoring(code)`就没有匹配，因为度量标准不共享同一组标签。方法put和del的条目没有匹配，并且不会显示在结果中：
```
{method="get"}  0.04            //  24 / 600
{method="post"} 0.05            //   6 / 120
```

### 多对一和一对多向量匹配
多对一和一对多匹配指的是“一”侧的每个向量元素可以与“多”侧的多个元素匹配的情况。 必须使用group_left或group_right修饰符明确请求，其中left/right确定哪个向量具有更高的基数。
```
<vector expr> <bin-op> ignoring(<label list>) group_left(<label list>) <vector expr>
<vector expr> <bin-op> ignoring(<label list>) group_right(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) group_left(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) group_right(<label list>) <vector expr>
```
随组修饰符提供的标签列表包含来自“一”侧的其他标签，以包含在结果度量标准中。对于on标签，只能出现在其中一个列表中。每次结果向量的序列必须是唯一可识别的。

分组修饰符只能用于[比较](https://prometheus.io/docs/prometheus/2.8/querying/operators/#comparison-binary-operators)和[算术](https://prometheus.io/docs/prometheus/2.8/querying/operators/#arithmetic-binary-operators)。默认情况下，`运算符and, unless和or`与右向量中的所有可能条目匹配。

在这种情况下，左向量每个方法标签值包含多个条目。因此，我们使用group_left表明这一点。右侧的元素现在与多个元素匹配，左侧具有相同的方法标签：
```
{method="get", code="500"}  0.04            //  24 / 600
{method="get", code="404"}  0.05            //  30 / 600
{method="post", code="500"} 0.05            //   6 / 120
{method="post", code="404"} 0.175           //  21 / 120
```
多对一和一对多匹配是高级用例，应该仔细考虑。 通常正确使用`ignoring（<labels>）`可提供所需的结果。

## 聚合运算符
Prometheus支持以下内置聚合运算符，这些运算符可用于聚合单个即时向量的元素，从而生成具有聚合值的较少元素的新向量：

* sum (calculate sum over dimensions)
* min (select minimum over dimensions)
* max (select maximum over dimensions)
* avg (calculate the average over dimensions)
* stddev (calculate population standard deviation over dimensions)
* stdvar (calculate population standard variance over dimensions)
* count (count number of elements in the vector)
* count_values (count number of elements with the same value)
* bottomk (smallest k elements by sample value)
* topk (largest k elements by sample value)
* quantile (calculate φ-quantile (0 ≤ φ ≤ 1) over dimensions)

这些运算符可以用于聚合所有标签维度，也可以通过包含`without`或`by`子句来保留不同的维度。
```
<aggr-op>([parameter,] <vector expression>) [without|by (<label list>)]
```
`parmameter`仅用于count_values，quantile，topk和bottomk。 `without`从结果向量中删除列出的标签，而所有其他标签都保留输出。 `by`删除未在`by`子句中列出的标签，即使它们的标签值在向量的所有元素之间是相同的。

`count_values`输出每个唯一样本值的一个时间序列。每个系列都有一个额外的标签。该标签的名称由聚合参数给出，标签值是唯一的样本值。每个时间序列的值是样本值存在的次数。

`topk`和`bottomk`与其他聚合器的不同之处在于，输入样本的子集（包括原始标签）在结果向量中返回。`by`和`without`仅用于存储输入向量。

例子:

如果指标http_requests_total具有按`application`,`instance`和`group`标签扇出的时间序列，我们可以通过以下方式计算每个`application`和`group`在所有实例上看到的HTTP请求总数：
```
sum(http_requests_total) without (instance)
```
等价于:
```
 sum(http_requests_total) by (application, group)
```
如果我们只对我们在所有`application`中看到的HTTP请求总数感兴趣，我们可以简单地写：
```
sum(http_request_total)
```
要计算运行每个构建版本的二进制文件的数量，我们可以编写：
```
count_values("version", build_version)
```
要在所有实例中获取5个最大的HTTP请求计数，我们可以编写：
```
topk(5, http_requests_total)
```
## 二进制运算符优先级
以下列表显示了Prometheus中二进制运算符的优先级，从最高到最低。

1. ^
2. *, /, %
3. +, -
4. ==, !=, <=, <, >=, >
5. and, unless
6. or

具有相同优先级的运算符是左关联的。例如，2*3％2相当于（2*3）％2。但是^是右关联的，因此2^3^2相当于2^（3^2）。