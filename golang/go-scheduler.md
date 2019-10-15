---
title: go scheduler
date: 2019-10-15 15:56:51
tags: [golang, scheduler]
category: [golang]
---

# MPG
充分利用多核的优势跑更多的任务。

M: 系统线程

P: 调度器的上下文

G: 任务(go routine)的上下文

# 调度过程
```
                              balance
go func() --> G --> P|local <=========> global
                    |
                    | 唤醒或者新建
                    |
                    M
                    |
                    |
     execute<----schedule
       |            |
       |            |
      G.fn-------->goexit
```
1. 语句go func()创建G
2. G存在本地队列或者平衡到全局队列
3. 新建或者唤醒M
4. 进入调度循环
5. 竭力获取G并执行
6. 执行结束进入下一循环

# details

返回了新创建的G之后会继续执行用户逻辑层代码。Schedule函数在每个goexit函数里面会调用。

对于3满足下面三个条件会用调用wakep，wakep会唤醒或者新建M:
1. pidle不为空
2. 非main goroutine
3. 没有M自旋等待P或者G

对于5，竭力获取G并执行的过程如下:
1. 有1/61的概率从全局P队列获取G，如果全局队列有G
2. 从本地P队列获取
3. 用findrunnable函数，再次从本地P获取，之后是全局P，网络任务，再就是别的运行中的P|M中steal了，如果再没有就一直等到有为止(block住了)

对于G.fn:
1. 如果G.fn包含cgo或者用syscall方式block在IO上，需要将当前的P和M解绑，P可能会被别的M取走
2. 对于RawSysCall类型的系统调用，则直接等待返回(因为速度快的才采用这种方式)

# 一些常量
1. M的最大数量是10000
2. 最小堆栈是2k
3. local P的长度是256，还有一个nextG
4. g0默认stack空间是8K
