---
title: mongodb基础知识
date: 2018-07-18 21:23:11
tags: [数据库, mongodb]
category: [mongodb]
---
mongodb是一种非关系型的数据库，适合于原型的快速搭建。
## 关键特性(key feature)来自[官网](https://docs.mongodb.com/manual/introduction/)
1. 高性能
    * 支持内嵌数据模型以减少数据库系统IO
    * 索引支持快速请求并能包含内嵌文档和数组(array)
2. 丰富的请求语言
    * [支持CRUD操作](https://docs.mongodb.com/manual/crud/)
    * [数据聚合](https://docs.mongodb.com/manual/core/aggregation-pipeline/)pipeline
    * [文本检索](https://docs.mongodb.com/manual/text-search/)
    * [地理信息请求](https://docs.mongodb.com/manual/tutorial/geospatial-tutorial/)
3. 高可用
    * [replica set](https://docs.mongodb.com/manual/replication/)
    * **automatic failover**
    * data redundancy
4. 水平扩展(通过sharding)
    * Sharding distributes data across a cluster of machines.
    * Starting in 3.4, MongoDB supports creating zones of data based on the shard key. In a balanced cluster, MongoDB directs reads and writes covered by a zone only to those shards inside the zone. See the Zones manual page for more information.

## 基本概念`
* **文档**(document)是mongodb中数据的基本单元，非常类似于关系型数据库中的行。但更具表现力。
* **集合**(collection)可以看作是一个拥有动态模式(dynamic schema)的表。
* MongoDB的一个实例可以拥有多个相互独立的数据库(database)，每个数据库都有自己的集合。
* 每个文档都有一个特殊的键"**_id**",这个键在文档所属的集合中是唯一的。

## 文档
* 在JS中，文档被表示为对象
* 区分**类型**和**大小写**，如{"foo":3}和{"Foo":"3"}类型和大小写都不同。
* 文档中的键值对是**有序的**，如{"x":1,"y":2}和{"y":2,"x":1}是不同的。通常字段顺序并不重要，无需让数据库模式依赖特定的顺序（MongoDB会对字段进行重新排序）。

## 集合
* 集合是动态模式的。但是创建多个集合也是有必要的，基于**数据管理（否则会特别混乱），查询速度，磁盘IO（分开当然会少了），创建索引**的原因。
* 命名。
    * 不能以”system."开头
    * 可以创建子集合。**MongoDB中推荐使用子集合进行组织，非常高效。**

## 数据库
* 特殊数据库
    * admin，用户认证。
    * local，本地使用，不可复制给远程（主从同步）。
    * config，mongodb中分片设置时，分片信息存储在config数据库中。

## 数据类型
* 基本类型：null，boolean，数字，字符串(UTF-8),日期，正则表达式，数组，内嵌文档，对象id，二进制数据(非UTF-8的)和代码
* 日期： 存储的是新纪元以来的**毫秒数**
* 内嵌文档
    * 优势：查一次就可以，不用级联查询
    * 劣势：每个内部的文档都需要修改
* _id的默认类型是ObjectId,使用12个字节存储。
    * Object格式如下:
>|0|1|2|3|4|5|6|7|8|9|10|11|
>|时间戳 |机器 |PID|计数器 |
    * 0-3为时间戳；4-6为hash(machine)得到的机器；7-8为PID；10-11为计数器。**可表示相同时间相同机器不同实例的不同计数**。
    * 每秒钟最多允许每个进程拥有2^24个不同的ObjectId
    * _id是自动生成的，如果没有制定mongodb会自动创建一个。但**通常是由客户端驱动完成。**


## 运行数据库
* 在ubuntu下运行，"sudo apt-get install mongodb"即可
* 默认端口号为**27017**.
* mongo shell客户端为“mongo”，直接"mongo"可以连接本地数据库
```
arthur@yyp-laotop:~/blog$ mongo
MongoDB shell version v3.6.3
connecting to: mongodb://127.0.0.1:27017
MongoDB server version: 3.6.3
Server has startup warnings:
2018-07-18T21:02:18.035+0800 I STORAGE  [initandlisten]
2018-07-18T21:02:18.035+0800 I STORAGE  [initandlisten] ** WARNING: Using the XFS filesystem is strongly recommended with the WiredTiger storage engine
2018-07-18T21:02:18.035+0800 I STORAGE  [initandlisten] **          See http://dochub.mongodb.org/core/prodnotes-filesystem
2018-07-18T21:02:19.953+0800 I CONTROL  [initandlisten]
2018-07-18T21:02:19.953+0800 I CONTROL  [initandlisten] ** WARNING: Access control is not enabled for the database.
2018-07-18T21:02:19.953+0800 I CONTROL  [initandlisten] **          Read and write access to data and configuration is unrestricted.
2018-07-18T21:02:19.953+0800 I CONTROL  [initandlisten]
>
```
* 命令
    * db,用于显示当前数据库
    * use,用于更改数据库
    * 辅助函数与等家函数
>    use foo          <==> db.getSisterDB("foo")
>    show dbs         <==> db.getMongo().getDBs()
>    show collections <==>db.getCollectionNames()
    * 更多的mongoshell命令，直接去[官网](https://docs.mongodb.com/manual/mongo/)查看具体的[函数](https://docs.mongodb.com/manual/reference/method/)。

## 帮助
* mongo shell中查看帮助。输入"help"
```
> help
        db.help()                    help on db methods
        db.mycoll.help()             help on collection methods
        sh.help()                    sharding helpers
        rs.help()                    replica set helpers
        help admin                   administrative help
        help connect                 connecting to a db help
        help keys                    key shortcuts
        help misc                    misc things to know
        help mr                      mapreduce
        show dbs                     show database names
        show collections             show collections in current database
        show users                   show users in current database
        show profile                 show most recent system.profile entries with time >= 1ms
        show logs                    show the accessible logger names
        show log [name]              prints out the last segment of log in memory, 'global' is default
        use <db_name>                set current database
        db.foo.find()                list objects in collection foo
        db.foo.find( { a : 1 } )     list objects in foo where a == 1
        it                           result of the last line evaluated; use to further iterate
        DBQuery.shellBatchSize = x   set default number of items to display on shell
        exit                         quit the mongo shell
```
* 查函数实现。输入函数，不加括号。
```
> db.foo.find
function (query, fields, limit, skip, batchSize, options) {
    var cursor = new DBQuery(this._mongo,
                             this._db,
                             this,
                             this._fullName,
                             this._massageObject(query),
                             fields,
                             limit,
                             skip,
                             batchSize,
                             options || this.getQueryOptions());

    {
        const session = this.getDB().getSession();

        const readPreference = session._serverSession.client.getReadPreference(session);
        if (readPreference !== null) {
            cursor.readPref(readPreference.mode, readPreference.tags);
        }

        const readConcern = session._serverSession.client.getReadConcern(session);
        if (readConcern !== null) {
            cursor.readConcern(readConcern.level);
        }
    }

    return cursor;
}
``` 