---
layout: post
title: "Factory策略模式"
subtitle: ''
author: "DylanYu"
header-style: text
tags:
  - 设计模式
  - 工厂模式
  - 简单工厂模式
  - 抽象工厂模式
---

简单工厂模式，工厂模式，抽象工厂模式都是创建型模式，都是为了帮助我们把床架对象的逻辑给抽象出来，优化系统结构，增加系统的扩展性

#### 简单工厂模式
一般通过一个静态方法根据参数创建不同的对象，该模式不修改代码的话，是不能实现扩展的

![image](https://dylanblog.github.io/img/in-post/2018-11-23-golf-factory-simple-factory.jpg)

#### 工厂模式
工厂模式根据不同的产品提供不同的工厂实现，该模式可以在同一个等级情况下，实现任意产品的扩展

![image](https://dylanblog.github.io/img/in-post/2018-11-23-golf-factory-factory-method.jpg)

#### 抽象工厂模式
抽象工厂模式是针对一类产品的构造模式，也就是说工厂接口中有生产各种商品的方法，每个具体工厂都会实现所有的创建方法
该模式对于新增产品线（factory实现）很容易，但是对于新增新产品则必须修改代码

![image](https://dylanblog.github.io/img/in-post/2018-11-23-golf-factory-abstract-factory.jpg)

#### 区别

- 简单工厂 ：用来生产同一等级结构中的任意产品。（对于增加新的产品，无能为力）
- 工厂方法 ：用来生产同一等级结构中的固定产品。（支持增加任意产品）
- 抽象工厂 ：用来生产不同产品族的全部产品。（对于增加新的产品，无能为力；支持增加产品族）