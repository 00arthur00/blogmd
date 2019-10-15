title: 模板方法(template method)
tags: []
categories: []
date: 2019-01-16 11:49:00
---
## 动机(Motivation)
* 在软件构建过程中,对于某一项任务,它常常有稳定的整体操作结构,但各个子步骤却有很多改变的需求,或者由于固有的原因(比如框架与应用之间的关系)而无法和任务的整体结构同时实现。
* 如何在确定稳定操作结构的前提下,来灵活应对各个子步骤的变化或者晚期实现需求?

## 结构化软件设计流程
<div align=center>
![producer](/images/designpattern/templatemethod/normalprocess.png)
</div>

## 面向对象软件设计流程
<div align=center>
![producer](/images/designpattern/templatemethod/oop.png)
</div>

## 早绑定与晚绑定
<div align=center>
![producer](/images/designpattern/templatemethod/binding.png)
</div>

### 模式定义(GoF)

定义一个操作中的算法的骨架 (稳定),而将一些步骤延迟(变化)到子类中。Template Method使得子类可以不改变(复用)一个算法的结构即可重定义(override 重写)该算法的某些特定步骤。

## 结构(Structure)
<div align=center>
![producer](/images/designpattern/templatemethod/structure.png)
</div>
## 要点
* Template Method模式是一种非常基础性的设计模式,在面向对象系统中有着大量的应用。它用最简洁的机制(虚函数的多态性)为很多应用程序框架提供了灵活的扩展点,是代码复用方面的基本实现结构。
* 除了可以灵活应对子步骤的变化外,“不要调用我,让我来调用你”的反向控制结构是Template Method的典型应用。 
*  在具体实现方面,被Template Method调用的虚方法可以具有实现,也可以没有任何实现(抽象方法、纯虚方法),但一般推荐将它们设置为protected方法。

## 伪码
C++:
``` cpp
class Library{
protected:
    virtual void op2()=0;
public:
	void op1(){};
	void op3();
	void run(){
    	op1();
        op2();
        op3();
    }
	virtual ~Library(){}
}

class Application:public Library{
protected:
	void op2() override{}
}

int main(){
	Library* pLib=new Application();
	lib->Run();
	delete pLib;
}
```

golang:
``` golang
package main

import "fmt"

type LibraryInterface interface {
	OP2()
}

type Library struct {
	LibraryInterface
}

func (l *Library) Run() {
	l.OP1()
	l.OP2()
	l.OP3()
}

func (l *Library) OP1() {
	fmt.Println("OP1")
}
func (l *Library) OP3() {
	fmt.Println("OP3")
}

type APP struct {
	Library
}

func (a *APP) OP2() {
	fmt.Println("OP2")
}

func main() {
	a := Library{LibraryInterface: &APP{}}
	a.Run()
}
```