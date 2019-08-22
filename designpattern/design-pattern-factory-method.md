title: Factory Method工厂方法
tags:
  - design pattern
categories: []
date: 2019-01-17 11:51:00
---
## “对象创建”模式
* 通过“对象创建” 模式绕开new,来避免对象创建(new)过程中所导致的紧耦合(依赖具体类),从而支持对象创建的稳定。它是接口抽象之后的第一步工作。
* 典型模式:Factory Method/Abstract Factory/Prototype/Builder

## 动机(Motivation)
* 在软件系统中,经常面临着创建对象的工作;由于需求的变化,需要创建的对象的具体类型经常变化。
* 如何应对这种变化?如何绕过常规的对象创建方法(new),提供一种“封装机制”来避免客户程序和这种“具体对象创建工作”的紧耦合?

## 模式定义
定义一个用于创建对象的接口,让子类决定实例化哪一个类。Factory Method使得一个类的实例化延迟(目的:解耦,手段:虚函数)到子类。

## 结构(Structure)
<div align=center>
![structure](/images/designpattern/factorymethod/structure.png)
</div>

## 要点总结
* Factory Method模式用于隔离类对象的使用者和具体类型之间的耦合关系。面对一个经常变化的具体类型,紧耦合关系(new)会导致软件的脆弱。
* Factory Method模式通过面向对象的手法,将所要创建的具体对象工作延迟到子类,从而实现一种扩展(而非更改)的策略,较好地解决了这种紧耦合关系。
* Factory Method模式解决“单个对象”的需求变化。缺点在于要求创建方法/参数相同。

## 伪码
### C++
``` cpp
//抽象类
class ISplitter{
public:
    virtual void split()=0;
    virtual ~ISplitter(){}
};


//工厂基类
class SplitterFactory{
public:
    virtual ISplitter* CreateSplitter()=0;
    virtual ~SplitterFactory(){}
};


//具体类
class BinarySplitter : public ISplitter{
    
};

class TxtSplitter: public ISplitter{
    
};

class PictureSplitter: public ISplitter{
    
};

class VideoSplitter: public ISplitter{
    
};

//具体工厂
class BinarySplitterFactory: public SplitterFactory{
public:
    virtual ISplitter* CreateSplitter(){
        return new BinarySplitter();
    }
};

class TxtSplitterFactory: public SplitterFactory{
public:
    virtual ISplitter* CreateSplitter(){
        return new TxtSplitter();
    }
};

class PictureSplitterFactory: public SplitterFactory{
public:
    virtual ISplitter* CreateSplitter(){
        return new PictureSplitter();
    }
};

class VideoSplitterFactory: public SplitterFactory{
public:
    virtual ISplitter* CreateSplitter(){
        return new VideoSplitter();
    }
};
```

### golang
``` golang
package main

import "fmt"

type Splitter interface {
	split()
}
type BinarySplitter struct{}

func (b *BinarySplitter) split() {
	fmt.Println("Binary splitter")
}

type PictureSplitter struct{}

func (p *PictureSplitter) split() {
	fmt.Println("picture splitter")
}

type ISplitterFactory interface {
	CreateObject() Splitter
}

type BinaryFactory struct{}

func (b *BinaryFactory) CreateObject() Splitter {
	return &BinarySplitter{}
}

type PictureFactory struct{}

func (p *PictureFactory) CreateObject() Splitter {
	return &PictureSplitter{}
}

type MainObject struct {
	factory ISplitterFactory
}

func (m *MainObject) Run() {
	spliiter := m.factory.CreateObject()
	spliiter.split()
}
func main() {
	m := MainObject{&BinaryFactory{}}
	m.Run()

	m2 := MainObject{&PictureFactory{}}
	m2.Run()
}
```