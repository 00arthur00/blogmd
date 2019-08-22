---
title: mongodb文档的创建、更新和删除(CUD)
date: 2018-07-19 21:11:10
tags: [数据库, mongodb]
category: [mongodb]
---
## 插入并保存文档
* 使用insert直接插入文档，如"db.foo.insert{"bar":"bazz"}。**_id会自动添加**。
* 为了提高插入效率，可使用**batchInsert(insertMany)**批量插入。如
```
> db.foo.insertMany([{a:1},{a:2}])
{
        "acknowledged" : true,
        "insertedIds" : [
                ObjectId("5b51e76d5f10c91eacafc4ce"),
                ObjectId("5b51e76d5f10c91eacafc4cf")
        ]
}
```
* mongodb接受的**最大消息长度是48MB**。如果比48MB大，驱动会自动分成多个48MB。
* 插入校验。**单个文档不大于16MB**。

## 删除文档
* 使用remove函数删除。不带参数，将删除所有内容。带参数，则删除指定内容
* 如果是删除整个集合的所有数据，可以使用remove()不带参数，也可是用**集合的drop**函数。后者更快，但不保留集合。

## 更新文档
使用update函数进行更新。update有两个参数，一个是查询文档，用于定位需要更新的目标文档；另一个是修改器(modifier)，用于说明要对找到的文档进行哪些修改。
### 文档替换
最简单的是用一个新的文档完全替换匹配的文档(不带修改器)。
```
> db.user.insertOne({name:"joe",friends:32,enemies:2})
{
        "acknowledged" : true,
        "insertedId" : ObjectId("5b51ebbe5f10c91eacafc4d2")
}
> var joe = db.user.findOne({name:"joe"})
> joe.friends++
32
> db.user.update({name:"joe"},joe)
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
```
### 使用修改器
    * $set: 指定一个字段的值，如果这个字段不存在，则创建它。
    * $inc: **增加或减少**已有的键值，如果不存在则创建它。**专门用来增建或减少数字**。
### 数组修改器（很大的一类）
    * 数组甚至可以作set用
    * $push: 如果数组存在，则在末尾添加；不存在则创建。
```
> db.blog.insertOne({title:"a blog",content:"...."})
{
        "acknowledged" : true,
        "insertedId" : ObjectId("5b51eff340c251ed8729cd28")
}
> db.blog.update({title:"a blog"}, {$push:{comments:{name:"yyp",content:"a good post"}}})
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
> db.blog.findOne()
{
        "_id" : ObjectId("5b51eff340c251ed8729cd28"),
        "title" : "a blog",
        "content" : "....",
        "comments" : [
                {
                        "name" : "yyp",
                        "content" : "a good post"
                }
        ]
}
```
    * 使用$each操作一个数组，可以一次$push多个添加值.
```
> db.blog.update({title:"a blog"},{$push:{year:{$each:[2007,2008,2009]}}})
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
> db.blog.findOne()
{
        "_id" : ObjectId("5b51eff340c251ed8729cd28"),
        "title" : "a blog",
        "content" : "....",
        "comments" : [
                {
                        "name" : "yyp",
                        "content" : "a good post"
                }
        ],
        "year" : [
                2007,
                2008,
                2009
        ]
}
```
    * $slice: **值必须是负数**，用来固定数组的最大长度。与$each和$push配合使用。在删除元素之前(如果元素超过最大限度），可以用$sort排序。
```
db.movies.update({"genre":"horror"},
        {$push:{top10:{
                $each:[1,2,3,4,5,6,7,8,9,10,11],
                $slice:-10,
                $sort:{rating:-1}}}})
```
    * 将数组作为数据集(set).
        * 使用$push和$ne组合，能添加一个。但不如直接使用$addToSet
        * $addToSet，可以避免重复插入；和$each配合，能添加多个
    * 删除元素：
        * $pop:头或尾删除一个，{$pop:{"key":1}} 从数组末尾删除元素；{$pop:{"key":-1}}从头删除元素
        * $pull,会删除所有匹配的，如果数组中有多个，则多个都会被删除。
    * 基于位置的修改器：如果数组有多个值，而我们只想对一部分进行操作，有两种方法操作数组中的值：通过位置(下标从0开始)或者定位操作符"$"
### upsert
**作为udpate函数的第三个参数**，是一种特殊的更新。要是没有找到符合条件的文档，就会使用**这个条件和更新文档为基础创建一个新文档**，如果找到了匹配的文档则正常更新。
是一个原子操作，可以减少竞争条件。
save()函数：如果文档不存在则创建，如果存在则更新。如果文档存在"_id"键，save会调用upsert，否则调用insert。
### 更新多个文档。
**update函数的第4个参数**。默认为false，即只更新第一个匹配条件的文档。**update函数在3.1的[nodejs API]((http://mongodb.github.io/node-mongodb-native/3.1/api/Collection.html#update)中被标记为deprecated,现在应该使用updateOne或者updateMany**。
trick: 获取最后一次命令执行的结果, "db.runCommand({getLastError:1})"
### 返回更新的文档
调用getLastError可以获取更新的有限信息，但是不能获取更新的文档。**可以通过findAndModify命令得到被更新的文档。这对于操作队列及执行其他需要进行原子性取值和赋值的操作十分方便。**
findAndModify is deprecated, [use findOneAndUpdate, findOneAndReplace or findOneAndDelete instead](http://mongodb.github.io/node-mongodb-native/3.1/api/Collection.html#findAndModify)

## 写入安全机制
* **写入安全(write concern)**是一种客户端设置，用于控制写入的安全级别。
* 两种基本的安全写入机制是应答式写入(acknowlege write)和非应答式写入(unackowledge write)。应答式写入：数据库会给出相应，告诉你写入的操作是否成功。非应答式写入：不返回任何响应，所以不知道写入是否成功。