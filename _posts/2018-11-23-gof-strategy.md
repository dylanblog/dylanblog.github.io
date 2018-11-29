---
layout: post
title: "Strategy策略模式"
subtitle: ''
author: "DylanYu"
header-style: text
tags:
  - 设计模式
  - 策略模式
---

PlatformTransactionManager

https://www.jianshu.com/p/6d7797f2d74c

#### 模式介绍
    定义一系列的算法， 把它们一个个封装起来， 并且使它们可相互替换。 本模式使得算法可独立于使用它的客户而变化

#### 解决的问题
    解决同一个问题，有不同的算法实现，使用方可根据需要选择不同的算法实现，比如：支付，短信网关

#### Spring中的应用

```
InstantiationStrategy -> SimpleInstantiationStrategy
```

#### UML图

![image](https://dylanblog.github.io/img/in-post/2018-11-23-golf-strategy.png)
> 通过UML图可以看出，具体的策略实现都是继承或者实现了同一个类或者接口，然后提供一个上下文对象组合这些具体实现供调用方使用

#### 应用举例
    支付问题的实现，比如现在系统要对接微信，支付宝和银联三种支付方式，这种情况可以使用策略模式：
    1. 定义一个IPayStrategy接口，定义prePay和callback两个接口
    2. 根据各个支付方式实现具体的支付类：WxPayStrategy，AliPayStrategy，UnionPayStrategy
    3. 创建一个PayService组合各种支付类型，供调用方使用

> 1. 在简单工厂模式中我们只需要传递相应的条件就能得到想要的一个对象，然后通过这个对象实现算法的操作。
> 2. 策略模式，使用时必须首先创建一个想使用的类对象，然后将该对象作为参数传递进去，通过该对象调用不同的算法
> 3. 在简单工厂模式中实现了通过条件选取一个类去实例化对象，策略模式则将选取相应对象的工作交给模式的使用者，它本身不去做选取工作。