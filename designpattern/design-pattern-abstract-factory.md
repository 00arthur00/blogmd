title: abstract factory抽象工厂方法
tags:
  - design pattern
categories: []
date: 2019-01-17 20:03:00
---
## 动机(Motivation)
* 在软件系统中,经常面临着“一系列相互依赖的对象”的创建工作;同时,由于需求的变化,往往存在更多系列对象的创建工作。

* 如何应对这种变化?如何绕过常规的对象创建方法(new),提供一种“封装机制”来避免客户程序和这种“多系列具体对象创建工作”的紧耦合?

## 模式定义
提供一个接口,让该接口负责创建一系列“相关或者相互依赖的对象”,无需指定它们具体的类。

## 结构(Structure)
<div align=center>
![structure](/images/designpattern/abstractfactory/structure.png)
</div>

## 要点总结
* 如果没有应对“多系列对象构建”的需求变化,则没有必要使用Abstract Factory模式,这时候使用简单的工厂完全可以。
* “系列对象”指的是在某一特定系列下的对象之间有相互依赖、或作用的关系。不同系列的对象之间不能相互依赖。
* Abstract Factory模式主要在于应对“新系列”的需求变动。其缺点在于难以应对“新对象”的需求变动。

## 伪码
C++：
```cpp
#include <iostream>

class IProductA{};
class IProductB{};
class IFactory{
public:
    virtual IProductA* createProductA()=0;
    virtual IProductB* createProductB()=0;
};


class ProductA:public IProductA{
public:
    ProductA(){
        std::cout<<"product a"<<std::endl;
    }
};
class ProductA1:public IProductA{
public:
    ProductA1(){
        std::cout<<"product a1"<<std::endl;
    }
};


class ProductB:public IProductB{
public:
    ProductB(){
        std::cout<<"product b"<<std::endl;
    }
};

class ProductB1:public IProductB{
public:
    ProductB1(){
        std::cout<<"product b1"<<std::endl;
    }
};

class FactoryAB{
    IProductA* createProductA(){
        return new ProductA();
    }
    IProductB * createProductB(){
        return new ProductB();
    }
};

class FactoryAB1{
    IProductA* createProductA(){
        return new ProductA1();
    }
    IProductB * createProductB(){
        return new ProductB1();
    }
};
```
golang
``` golang
package main

import "fmt"

type IProductA interface{}
type IProductB interface{}

type IFactory interface {
	CreateProductA() IProductA
	CreateProductB() IProductB
}

type ProductA struct{}
type ProductB struct{}

type ProductA1 struct{}
type ProductB1 struct{}

type FactoryAB struct{}

func (f *FactoryAB) CreateProductA() IProductA {
	fmt.Println("ProductA")
	return &ProductA{}
}
func (f *FactoryAB) CreateProductB() IProductB {
	fmt.Println("ProductB")
	return &ProductB{}
}

type FactoryAB1 struct{}

func (f *FactoryAB1) CreateProductA() IProductA {
	fmt.Println("ProductA1")
	return &ProductA1{}
}
func (f *FactoryAB1) CreateProductB() IProductB {
	fmt.Println("ProductB1")
	return &ProductB1{}
}

type MainObj struct {
	factory IFactory
}

func (m *MainObj) GetObjs() {
	m.factory.CreateProductA()
	m.factory.CreateProductB()
}
func main() {
	m := MainObj{&FactoryAB{}}
	m.GetObjs()

	m1 := MainObj{&FactoryAB1{}}
	m1.GetObjs()
}
```