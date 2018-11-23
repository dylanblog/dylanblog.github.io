---
layout: post
title: "Decorator策略模式"
subtitle: ''
author: "DylanYu"
header-style: text
tags:
  - 设计模式
  - 装饰器模式
---

动态的给一个对象添加额外的功能

> ComponentImpl 和 Decorator 类都实现了 IComponent 接口，不同的是 ComponentImpl 提供了具体实现，
> 而 Decorator 是先聚合 ComponentImpl 接着在自己的实现方法即 operation() 方法中做些处理（即装饰）
> 后再调用 ComponentImpl 对象的具体实现
> ![image](https://dylanblog.github.io/img/in-post/2018-11-23-gof-decrator.png)
