---
title: ES6 and Beyond
date: 2018-07-06 21:57:56
tags: [js, nodejs]
category: [nodejs]
---
从[官方文档](https://nodejs.org/en/docs/)摘了几个感兴趣的点

## ECMAScript 2015 (ES6)功能分成三组 shipping, staged和in progress
1. All shipping features, which V8 considers stable, are turned on by default on Node.js and do NOT require any kind of runtime flag.
```
    shipping被V8认为是稳定的，默认打开的，不需要设置。
```
2. Staged features, which are almost-completed features that are not considered stable by the V8 team, require a runtime flag: --harmony.
```
    staged，快完成了，不稳定。用--harmony参数打开
```
3. In progress features can be activated individually by their respective harmony flag, although this is highly discouraged unless for testing purposes. Note: these flags are exposed by V8 and will potentially change without any deprecation notice.
```
    In progress, 正在搞，需要--harmony再加专用的参数打开。仅用于测试极端不建议开启，而且这些专用参数有可能以后修改了也没有deprecated提示。
```

## 不同版本nodejs默认携带的功能
去[这个网站](https://node.green/)查

## 哪些功能是In progress的
```
node --v8-options | grep "in progress"
```

## 从架构一开始就用了--harmony，是否应该删？
如果生产环境，还是删了吧，或者升级V8到包含你功能的稳定版本。
```
The current behaviour of the --harmony flag on Node.js is to enable staged features only. After all, it is now a synonym of --es_staging. As mentioned above, these are completed features that have not been considered stable yet. If you want to play safe, especially on production environments, consider removing this runtime flag until it ships by default on V8 and, consequently, on Node.js. If you keep this enabled, you should be prepared for further Node.js upgrades to break your code if V8 changes their semantics to more closely follow the standard.
```

## 当前nodejs对应的V8版本
```
node -p process.versions.v8
```
