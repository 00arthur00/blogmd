title: observer观察者模式
tags:
  - design pattern
categories: []
author: ''
date: 2019-01-16 16:48:00
---
## 动机(Motivation)
* 在软件构建过程中,我们需要为某些对象建立一种“通知依赖关系” ——一个对象(目标对象)的状态发生改变,所有的依赖对象(观察者对象)都将得到通知。如果这样的依赖关系过于紧密,将使软件不能很好地抵御变化。

* 使用面向对象技术,可以将这种依赖关系弱化,并形成一种稳定的依赖关系。从而实现软件体系结构的松耦合。

## 模式定义(GoF)
定义对象间的一种一对多(变化)的依赖关系,以便当一个对象(Subject)的状态发生改变时,所有依赖于它的对象都得到通知并自动更新。

## 结构(Structure)
<div align=center>
![structure](/images/designpattern/observer/structure.png)
</div>

## 要点总结
* 使用面向对象的抽象,Observer模式使得我们可以独立地改变目标与观察者,从而使二者之间的依赖关系达致松耦合。
* 目标发送通知时,无需指定观察者,通知(可以携带通知信息作为参数)会自动传播。
* 观察者自己决定是否需要订阅通知,目标对象对此一无所知。
* Observer模式是基于事件的UI框架中非常常用的设计模式,也是MVC模式的一个重要组成部分。

## 伪代码
C++
``` cpp
class subject;
class observer
{
public:
    virtual void update()=0;
    virtual ~observer();
};

class ConcreteObserver:public observer{
private:
    subject * m_subject;
    int state;
public:
    ConcreteObserver(subject *s):m_subject(s),state(0){}
    void update(){
        state = m_subject->GetState();
    }
}

class subject{
private:
    list<observer*> m_lObserver;
public:
    void Attach(observer *o){m_lObserver.push(o);}
    void Detach(observer *o){m_lObserver.pop(o);}
    virtual int GetState()=0;
    virtual void SetState(int)=0;
    void Notify(){
        for(auto o:observer){
            o.update();
        }
    }
}

class ConcreteSubject:public subject{
private:
    int state;
public:
    int GetState() override{return state;}
    int SetState(int s){state=s;}
}
```

golang:
``` golang
package main

import "fmt"

type Subject struct {
	State     int32
	Observers []Observer
}

func (s *Subject) Attach(o Observer) {
	s.Observers = append(s.Observers, o)
}

func (s *Subject) Detach(o Observer) {
	target := s.Observers[:0]
	for index := 0; index < len(s.Observers); index++ {
		if s.Observers[index] != o {
			target = append(target, s.Observers[index])
		}
	}
	s.Observers = target
}
func (s *Subject) Notify() {
	for index := 0; index < len(s.Observers); index++ {
		s.Observers[index].update()
	}
}

type Observer interface {
	update()
}
type ConcreteObserver1 struct {
	s     *Subject
	State int32
}

func (c *ConcreteObserver1) update() {
	c.State = c.s.State
	fmt.Println("concrete observer1:", c.State)
}

type ConcreteObserver2 struct {
	s     *Subject
	State int32
}

func (c *ConcreteObserver2) update() {
	c.State = c.s.State
	fmt.Println("concrete observer2:", c.State)
}

func main() {
	s := Subject{Observers: make([]Observer, 0)}
	o1 := ConcreteObserver1{s: &s}
	o2 := ConcreteObserver2{s: &s}

	s.Attach(&o1)
	s.Attach(&o2)

	s.State = 10
	s.Notify()
	s.Detach(&o1)
	s.Notify()
}
```