---
layout: post
title: "《Linux Linux性能优化实战》笔记 "
subtitle: ''
author: "DylanYu"
header-style: text
tags:
  - Linux
---

1. CPU上下文切换

    上下文切换分类：

        - 进程上下文切换
        - 线程上下文切换
        - 中断上下文切换

    进程运行空间分为：

        - 内核运行空间
        - 用户态运行空间

    分为0-3共4个Level，值越小优先级越高

    用户态转换到内核态时需要系统调用：
        进程间切换时需要保存现场，
        而系统调用是同一进程内的切换，所以系统调用花费的时间少于经常间的切换，所以多线程比多进程快

    进程的切换是由内核控制和切换的，所以进程切换只会发生在内核。

    进程包含虚拟内存，全局变量，栈等用户态资源，还包含了内核堆栈，寄存器等内核态资源

    进程什么触发调度：
        cpu时间片切换
        进程资源不足
        主动调用sleep挂起进程
        高优先级的进程抢占
        硬件中断

    进程与线程的关系：进程是资源拥有的基本单位，线程是系统调用的基本单位

    上下文切换的两种情况：
        1. 进程间切换，涉及到用户态和内核态的上下文保存
        2. 进程内的线程切换，不涉及用户态资源的保存，只有内核态的上下文保存




