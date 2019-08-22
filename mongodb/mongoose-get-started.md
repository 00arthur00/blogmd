---
title: Mongoose简介
date: 2018-08-07 21:24:44
tags: [mongodb, mongoose, nodejs, ORM]
category: [mongodb]
---
## 简介
Mongoose是一个nodejs的库，MongoDB的ORM，官网是http://mongoosejs.com/。
编写MongoDB验证，转换和业务逻辑样板是一种拖累,Mongoose则能简单解决这些问题。Mongoose提供了一个直接的，**基于模式(schema-based)**的解决方案来为您的应用程序数据建模。 它包括**内置的类型转换，验证，查询构建，业务逻辑钩子**等，开箱即用。

## 安装
通过npm在命令行安装mongoose。
``` shell
npm install mongoose
```

## 载入mongoose并连接DB
``` javascript
var mongoose = require('mongoose');
//normally, we should use createConnection() to create a new conenction instead of reuse the default coneection
//http://mongoosejs.com/docs/api.html#mongoose_Mongoose-createConnection
mongoose.connect('mongodb://localhost/test');
```

## 获取连接并处理错误
``` javascript
//default connection
var db = mongoose.connection;
db.on('error', console.error.bind(console, 'connection error:'));
db.once('open', function() {
  // we're connected!
});
```

## 创建schema
**With Mongoose, everything is derived from a Schema.**
``` javascript
var kittySchema = new mongoose.Schema({
  name: String
});
```

## 将schema编译进Model
``` javascript
var Kitten = mongoose.model('Kitten', kittySchema);
```

## 基于Model创建文档
Model是一个我们用来构造文档的类。
``` javascript
var silence = new Kitten({ name: 'Silence' });
console.log(silence.name); // 'Silence'
```

## 向文档添加函数
``` js
// NOTE: methods must be added to the schema before compiling it with mongoose.model()
kittySchema.methods.speak = function () {
  var greeting = this.name
    ? "Meow name is " + this.name
    : "I don't have a name";
  console.log(greeting);
}

var Kitten = mongoose.model('Kitten', kittySchema);
```
添加到schema中methods属性的函数，将会被编译到Model的prototype中，并最终暴露给每个文档实例。
``` js
var fluffy = new Kitten({ name: 'fluffy' });
fluffy.speak(); // "Meow name is fluffy"
```

## 保存文档
``` js
fluffy.save(function (err, fluffy) {
    if (err) return console.error(err);
    fluffy.speak();
  });
```

## 获取文档
* 获取所有文档
``` js
Kitten.find(function (err, kittens) {
  if (err) return console.error(err);
  console.log(kittens);
})
```
* 过滤查找
``` js
Kitten.find({ name: /^fluff/ }, callback);
```
* 向返回的文档添加属性
``` js
testMongoose = async function(){
    var kitty = new Kitten({name:"yyp"});
    let saveReuslt = await kitty.save();
    console.log("saveResult:",saveReuslt);
    
    let findResult = await Kitten.findOne({name:"yyp"});
    console.log("findResult:",findResult);
    findResult.newKey = "newValue";
    console.log("modified failed:",findResult);

    findResult = await Kitten.findOne({name:"yyp"}).lean();
    console.log("findResult:",findResult);
    findResult.newKey = "newValue";
    console.log("modified success:",findResult);
}
```
**通过find返回的是基于schema的document，不能添加属性。如果要额外添加属性需要指定lean(),返回plain js。**
PS： save, findOne等函数返回promise，因此可以使用await