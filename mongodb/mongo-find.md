---
title: mongodb 查询(Find/Read)
date: 2018-07-22 21:32:45
tags: [数据库, mongodb]
category: [mongodb]
---
## find简介
### mongodb中使用find进行查询。
返回一个集合中文档的子集。空文档{}即find中不含参数，则返回所有文档。
### find的第二个参数，指定返回的想要的键。
如:
```
> db.foo.find({},{a:1})
{ "_id" : ObjectId("5b51e76d5f10c91eacafc4ce"), "a" : 16 }
{ "_id" : ObjectId("5b51e76d5f10c91eacafc4cf"), "a" : 12 }
```
value的值为1，则返回响应的键值对。value为0，则不返回。
**_id在默认情况下会返回。**可以通过设置_id为0，不返回_id。如：
```
> db.foo.find({},{a:1,_id:0})
{ "a" : 16 }
{ "a" : 12 }
```
## 条件查找
查询可以进行条件查找，比如范围查找，OR子句，NOT，AND
### 范围查找
利用**比较**操作符,可用于数字日期等。$ne能用于所有类型。更多类型[**查看官网 Query and Protection opeators**](https://docs.mongodb.com/manual/reference/operator/query/)
| 操作符 | 含义 | 
|-------|------|
| $lt | 小于 |
| $lte | 小于等于 |
| $gt | 大于 |
| $gte | 大于等于 |
| $ne | 不等于 |
```
> db.foo.find({a:{$lt:20,$gt:15}})
{ "_id" : ObjectId("5b51e76d5f10c91eacafc4ce"), "a" : 16 }
```
### OR查询
有两种方式可以进行or查询： $in可以查询一个键的多个值； $or更通用，可以在多个健中查询任意的给定值。**$OR型查询($or和$in)，第一个条件应该尽可能匹配更多的文档，这样才是最高校的。**
#### $in
```
> db.foo.find({a:{$in:[16,100]}})
{ "_id" : ObjectId("5b51e76d5f10c91eacafc4ce"), "a" : 16 }
{ "_id" : ObjectId("5b548f0c321015df48524076"), "a" : 100 }
```
$nin为$in的反义词，查询不匹配的文档。
#### $or
```
> db.foo.find({$or:[{a:100},{_id: ObjectId("5b51e76d5f10c91eacafc4ce")}]})
{ "_id" : ObjectId("5b51e76d5f10c91eacafc4ce"), "a" : 16 }
{ "_id" : ObjectId("5b548f0c321015df48524076"), "a" : 100 }
```
### $not
$not是元条件句，可以用子任何其它条件之上。 $not与正则表达式配合使用极为有用，用来查询与特定模式不匹配的文档。
```
> db.foo.find({name:{$not:/joey?/}})
{ "_id" : ObjectId("5b51e76d5f10c91eacafc4ce"), "a" : 16 }
{ "_id" : ObjectId("5b51e76d5f10c91eacafc4cf"), "a" : 12 }
{ "_id" : ObjectId("5b548f0c321015df48524076"), "a" : 100 }
{ "_id" : ObjectId("5b54939f321015df48524079"), "name" : "yyp" }
```
### 条件语义
条件语句是指find的查询的条件语句。
1. **基本上：条件语句是内层文档的键，而修改器是外层文档的健。**
2. 一个键可以有任意多个条件，但是**一个键不可以对应多个修改器**。
3. **有一些元操作符(meta-operator)也位于外层文档中。**如$and, $or和$nor

## 特定类型查询
### null
 **null不仅匹配某个键为null的文档，而且还会匹配不包含这个键的文档**。
### 正则表达式
 mongodb使用**perl兼容的正则表达式库（PCRE）库来匹配正则表达式**，任何PCRE支持的正则表达式语法都能被mongodb接受。
### 查询数组
https://docs.mongodb.com/manual/reference/operator/query/ 
| Name | Description |
|---|----|
| $all | Matches arrays that contain all elements specified in the query. |
| $elemMatch | Selects documents if element in the array field matches all the specified $elemMatch conditions. |
| $size | Selects documents if the array field is a specified size. |
1. $all通过多个元素来匹配数组，**数组中元素包含all中元素即可匹配。**
2. $size，数组大小匹配的。
```
db.food.find({$size:{$gt:3}})
```
3. $slice, 用于find的第二个参数，返回某个key匹配的数组元素的一个子集，**$slice指定方向和个数**。如
   * db.findOne(criteria, {"comments":{$slice:10}})，返回开始的10个
   * db.findOne(criteria, {"comments":{$slice:-10}})，返回最后的10个
   * db.findOne(criteria, {"comments":{$slice:23:10}})，返回第24-33个
4. 数组和范围查找的相互作用
  对查询{x:{$gte:10,$lte:20}}：
  * 如果x不是数组，则会返回10-20之间的精确匹配。
  * 如果x是数组，则只要满足x>=10或者x<=20条件中的一个就会认为匹配。
  **如集合中包含文档{x:5}; {x:15};{x:25};{x:[5,25]}，则会返回{x:15}及{x:[5,25]}**
  * 使用$elemMatch解决这个问题，{x:{$elemMatch:{$gte:10,$lte:20}}}.**会返回同时满足两个条件的。但只能用于数组，**所以不会有任何匹配的结果返回

### 查询内嵌文档
查询内嵌文档有两种方法：**查询整个文档**和**只针对其键值对**进行查询。
* 整个文档
如对：{"name":{first:"yp",last:"y"}，"age":18}
```
> db.foo.find({"name":{first:"yp",last:"y"}})
{ "_id" : ObjectId("5b57280298c701c5751ecd9b"), "name" : { "first" : "yp", "last" : "y" }, "age" : 18 }
```
**整个文档查询，与顺序有关。如果改为{life:"jll",name:"yyp"}，则不能匹配.**
```
> db.foo.find({"name":{last:"y",first:"yp"}})
> db.runCommand({"getLastError":1})
{
        "connectionId" : 1,
        "n" : 0,
        "syncMillis" : 0,
        "writtenTo" : null,
        "err" : null,
        "ok" : 1
}
```
* 只针对内嵌文档的特定键进行查询是比较好的做法。这样，即使数据模式改变，也不会导致所有查询因为精确匹配而一下都挂掉。如下所示，和字段的请求顺序无关了。**用点"."表示法查询内嵌文档**。
```
> db.foo.find({"name.last":"y","name.first":"yp"})
{ "_id" : ObjectId("5b57280298c701c5751ecd9b"), "name" : { "first" : "yp", "last" : "y" }, "age" : 18 }
```
* **如果文档在数组内，且不是进行整个文档的精确匹配（对一个文档的多个键进行匹配），则需要使用$elemMatch。**

## $where查询
$where语句最常见的应用是比较文档的两个键值的值是否相等。**$where语句查询速度慢，不能使用索引，如非必要不要使用$where.**
### 服务器端脚本
**在服务器端执行javascript脚本应注意安全性。如果使用不当，有可能受到攻击，类似sql注入。不过，在接受输入时遵循一些规则，就可以安全的使用JS。也可以在运行mongod时指定--noscripting选项，完全关闭javascript的执行。**

## 游标
find()的返回值，就是游标(Cursor)
### limit,skip,sort
limit：限制返回结果的数量；
skip：忽略一定数量的结果；
sort：排序。value为1是升序，-1为降序。如db.foo.find().sort({"name":1,age:-1})，为按name升序，按age降序。
> 比较顺序
> mongodb中一个key，可能对应多个类型的value，如整型，布尔型和null。对这种混合类型的key进行排序，其排序顺序是预先定义好的。排序顺序如下：
> (1) 最小值，(2)null,(3)数字型，(4)字符串，(5)对象/文档，(6)数组，(7)二进制数据,(8)对象ID，(9)布尔型，(10)日期性，(11)时间戳，(12)正则表达式，(13)最大值
### 避免使用skip略过大量结果
用skip略过少量文档还是不错的。但是要是数据量很大，skip就会变得很慢，因为要先找到需要被略过的数据，然后再抛弃。并且mongodb对skip优化的不好。
#### 不用skip对结果进行分页
方法是添加某种有序字段，如date，find的时候直接根据字段值作为查询条件。就可以不用skip了。
#### 随即选取文档
集合中随机挑选一个文档是很常见的问题。最笨（也很慢）的做法是，先查总数，然后产生一个0到文档总数的随机值，然后skip随机数之前的文档。查询和skip都会很消耗时间。
换个思路，从集合中找一个随机数还是有恩多的办法的。先增加一个random字段，插入数据的时候用Math.random()产生0-1之间的数字。查询的时候：
```
> var random = Math.random()
> result = db.foo.findOne({"random":{$gte:random}})
```
若结果为null(不存在比随机数大的值)，则换个方向使用$lt，如果还是为null，则表为空，返回null也说的通。
**但是这种技巧需要建立随机数的索引。**
### 高级查询选项
简单查询(plain query)为直接find；封装查询(wrapped query)，如find().sort()不是直接发送find查询，而是把sort的内容包含在一个更大的文档里再查询。
绝大多数的驱动程序都提供了辅助函数，用于向查询中添加各种选项：
* $maxscan: interger
指定扫描次数的上限。
```
> db.foo.find(criteria)._addSpecial("$maxscan":20)
```
* $min: document
查询条件的开始。文档必须与索引的键完全匹配。在内部使用中通常使用$lt代替$min。**可以使用$min指定查询的下界，在复杂查询中有用。**
* $max: document
查询条件的结束。文档必须与索引的键完全匹配。
* showDiskLoc:true
用于显示文档在磁盘中的位置
### 获取一致性结果
如果一边用cursor获取一边修改，则可能查询会永远不停。因为修改的数据存入的时候，可能会append到数据库的后面，而cursor是动态获取，因此修改过的数据被再次获取，周而复始。**nodejs中需要使用.toArray来一次获取所有的。** shell中可以使用find().snapshot()来查询的结果进行快照。
### 游标生命周期
* 在服务器端，游标消耗内存和其他资源。游标遍历了所有结果以后，或者客户端法来消息要求终止，则数据库释放这些资源。
* 在客户端，如果遍历完了结果，或者游标离开了作用域，驱动会法一条特别的消息让服务器端销毁。**如果客户端没有迭代完，且在作用于内，如果一个游标10分钟内没有使用的话，数据库会自动销毁。**
* 如果不想使用这种“超时销毁”的机制。多是驱动实现了immortal函数或者类似机制。

## 数据库命令
一种非常特殊的**查询类型**叫数据库命令(database command)。如db.foo.drop()等价的database commnd为db.runCommand({"drop":"foo"}).**可以通过db.listCommands()查看所有的数据库命令。**
```
> db.runCommand({"drop":"foo"})
{ "ns" : "test.foo", "nIndexesWas" : 1, "ok" : 1 }
```
原理：
数据库命令总会返回一个包含“ok“的文档。如果ok的值为1，则说明成功了；如果是0，则说明失败了，会有一个errmsg键说明原因。
mongodb中的命令被实现为一类**特殊的查询**，这些特殊的查询会在$cmd上执行。如drop命令被转换为 db.$cmd.find({"drop":"foo"})。mongodb服务器受到$cmd上的查询时，不会进行查询，而是使用特殊逻辑进行处理。
**如果当前位于其他数据库，但需要执行一个管理员命令，可以使用adminCommand()而不是runCommand().**