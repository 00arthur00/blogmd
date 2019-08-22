---
title: 初学nodejs的几个问题
date: 2018-07-05 23:36:20
tags: [js, nodejs]
category: [nodejs]
---
## this的用法，在什么时候使用?[一般四种](https://www.cnblogs.com/pabitel/p/5922511.html)
1. 在一般函数方法中使用 this 指代全局对象
```
function test(){
　　　this.x = 1;
　　　alert(this.x);
}
test(); // 1
console.log(x)//1
```
2. 作为对象方法调用，this 指代上级对象
```
function test(){
　　alert(this.x);
}
var o = {};
o.x = 1;
o.m = test;
o.m(); // 1
```
3. 作为构造函数调用，this 指代new 出的对象
```
function test(){
　　this.x = 1;
}
var o = new test();
alert(o.x); // 1
//运行结果为1。为了表明这时this不是全局对象，我对代码做一些改变：
var x = 2;
function test(){
　　this.x = 1;
}
var o = new test();
alert(x); //2
```
4. apply 调用 ，apply方法作用是改变函数的调用对象，此方法的第一个参数为改变后调用这个函数的对象，this指代第一个参数
```
var x = 0;
function test(){
　　alert(this.x);
}
var o={};
o.x = 1;
o.m = test;
o.m.apply(); //0
//apply()的参数为空时，默认调用全局对象。因此，这时的运行结果为0，证明this指的是全局对象。如果把最后一行代码修改为
o.m.apply(o); //1
```
## const在什么时候使用
const和let一样是块范围的(block-scoped),被const声明的变量不能被重新赋值或者重新声明。注意，const只是创建了一个只读的变量的引用，并不表示变量变量是不可变的。变量的函数或者属性可以随便调用与赋值。
[MDN官方文档](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/const)
## var, let什么关系
let和var的区别，var要么是函数范围内的要么是全局的，没有块的概念。let则是块范围的。
>let allows you to declare variables that are limited in scope to the block, statement, or expression on which it is used. This is unlike the var >keyword, which defines a variable globally, or locally to an entire function regardless of block scope.
```
// i IS NOT known here
// j IS NOT known here
// k IS known here, but undefined
// l IS NOT known here

function loop(arr) {
    // i IS known here, but undefined
    // j IS NOT known here
    // k IS known here, but has a value only the second time loop is called
    // l IS NOT known here

    for( var i = 0; i < arr.length; i++ ) {
        // i IS known here, and has a value
        // j IS NOT known here
        // k IS known here, but has a value only the second time loop is called
        // l IS NOT known here
    };

    // i IS known here, and has a value
    // j IS NOT known here
    // k IS known here, but has a value only the second time loop is called
    // l IS NOT known here

    for( let j = 0; j < arr.length; j++ ) {
        // i IS known here, and has a value
        // j IS known here, and has a value
        // k IS known here, but has a value only the second time loop is called
        // l IS NOT known here
    };

    // i IS known here, and has a value
    // j IS NOT known here
    // k IS known here, but has a value only the second time loop is called
    // l IS NOT known here
}

loop([1,2,3,4]);

for( var k = 0; k < arr.length; k++ ) {
    // i IS NOT known here
    // j IS NOT known here
    // k IS known here, and has a value
    // l IS NOT known here
};

for( let l = 0; l < arr.length; l++ ) {
    // i IS NOT known here
    // j IS NOT known here
    // k IS known here, and has a value
    // l IS known here, and has a value
};

loop([1,2,3,4]);

// i IS NOT known here
// j IS NOT known here
// k IS known here, and has a value
// l IS NOT known here
```
## exports, module.exports 和 module什么关系
看V8.x稳定版的[官方文档](https://nodejs.org/dist/latest-v8.x/docs/api/modules.html)有详细解释。
* exports是module.exports的快捷方式([exports shortcut](https://nodejs.org/dist/latest-v8.x/docs/api/modules.html#modules_exports_shortcut))
* 在文件内，module.exports.f = ... 可以简写为 exports.f = .... 
* exports用于*文件范围内*，而module.exports则和*require*函数相关，为require函数的返回值。E.g.
> The exports variable is available within a module's file-level scope, and is assigned the value of module.exports before the module is evaluated.
```
module.exports.hello = true; // Exported from require of module
exports = { hello: false };  // Not exported, only available in the module

```
* 常用写法
```
module.exports = exports = function Constructor() {
  // ... etc.
};
```
* require的假想实现（只是为了说明module.exports和exports的关系）,require使用的是module.exports而非exports
```
function require(/* ... */) {
  const module = { exports: {} };
  ((module, exports) => {
    // Module code here. In this example, define a function.
    function someFunc() {}
    exports = someFunc;
    // At this point, exports is no longer a shortcut to module.exports, and
    // this module will still export an empty default object.
    module.exports = someFunc;
    // At this point, the module will now export someFunc, instead of the
    // default object.
  })(module, module.exports);
  return module.exports;
}
```
* require()函数，寻找module的过程。详细解释顺着[这](https://nodejs.org/dist/latest-v8.x/docs/api/modules.html#modules_all_together)往下看。写几个要点
1.  core module直接返回
2. '/'从root路径开始找
3. './, ../'相对路径找
4. 去node module里着，递归向上找包含node_modules的路径。
5. 抛异常 “not found"
```
require(X) from module at path Y
1. If X is a core module,
   a. return the core module
   b. STOP
2. If X begins with '/'
   a. set Y to be the filesystem root
3. If X begins with './' or '/' or '../'
   a. LOAD_AS_FILE(Y + X)
   b. LOAD_AS_DIRECTORY(Y + X)
4. LOAD_NODE_MODULES(X, dirname(Y))
5. THROW "not found"

LOAD_AS_FILE(X)
1. If X is a file, load X as JavaScript text.  STOP
2. If X.js is a file, load X.js as JavaScript text.  STOP
3. If X.json is a file, parse X.json to a JavaScript Object.  STOP
4. If X.node is a file, load X.node as binary addon.  STOP

LOAD_INDEX(X)
1. If X/index.js is a file, load X/index.js as JavaScript text.  STOP
2. If X/index.json is a file, parse X/index.json to a JavaScript object. STOP
3. If X/index.node is a file, load X/index.node as binary addon.  STOP

LOAD_AS_DIRECTORY(X)
1. If X/package.json is a file,
   a. Parse X/package.json, and look for "main" field.
   b. let M = X + (json main field)
   c. LOAD_AS_FILE(M)
   d. LOAD_INDEX(M)
2. LOAD_INDEX(X)

LOAD_NODE_MODULES(X, START)
1. let DIRS=NODE_MODULES_PATHS(START)
2. for each DIR in DIRS:
   a. LOAD_AS_FILE(DIR/X)
   b. LOAD_AS_DIRECTORY(DIR/X)

NODE_MODULES_PATHS(START)
1. let PARTS = path split(START)
2. let I = count of PARTS - 1
3. let DIRS = []
4. while I >= 0,
   a. if PARTS[I] = "node_modules" CONTINUE
   b. DIR = path join(PARTS[0 .. I] + "node_modules")
   c. DIRS = DIRS + DIR
   d. let I = I - 1
5. return DIRS
```
## prototype什么时候使用
Object.prototype:可以为所有 Object 类型的对象添加属性。属性是所有对象共有的，对象通过new来获取，如：
```
function Func(){
    console.log("in Func")
}
Func.prototype.greet = function(){
    console.log('in greeting')
}
var f = new Func(); //output "in Func"
f.greet(); //output "in greeting". 这里可以看出是可以访问prototype定义的函数的
```
更复杂的是可以做hook或者构造子类,[MDN官网有例子](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/prototype)。
## js中有没有key是string类型的字典？

js中创建对象可以是：
```
var car = {
    length:3,
    brand:"bba"
}
console.log(car.length)
console.log(car['length'])
```
然后可以通过car.brand对对象car的属性brand进行访问
在ubuntu18.04测试car['brand']也是可用的
