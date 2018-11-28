---
layout: post
title: "Java并发编程：CountDownLatch、CyclicBarrier和Semaphore"
subtitle: ''
author: "DylanYu"
header-style: text
tags:
  - Java
  - CountDownLatch
  - CyclicBarrier
  - Semaphore
---

1. CountDownLatch和CyclicBarrier都能够实现线程之间的等待，只不过它们侧重点不同：

    - CountDownLatch一般用于某个线程A等待若干个其他线程执行完任务之后，它才执行；
    - 而CyclicBarrier一般用于一组线程互相等待至某个状态，然后这一组线程再同时执行；
    - 另外，CountDownLatch是不能够重用的，而CyclicBarrier是可以重用的。

2. Semaphore其实和锁有点类似，它一般用于控制对某组资源的访问权限。

3. Object level lock vs class level lock

    - Synchronization in Java guarantees that no two threads can execute a synchronized method, which requires same lock, simultaneously or concurrently.
    - synchronized keyword can be used only with methods and code blocks. These methods or blocks can be static or non-static both.
    - When ever a thread enters into Java synchronized method or block it acquires a lock and whenever it leaves synchronized method or block it releases the lock. Lock is released even if thread leaves synchronized method after completion or due to any Error or Exception.
    - Java synchronized keyword is re-entrant in nature it means if a synchronized method calls another synchronized method which requires same lock then current thread which is holding lock can enter into that method without acquiring lock.
    - Java synchronization will throw NullPointerException if object used in synchronized block is null. For example, in above code sample if lock is initialized as null, the “synchronized (lock)” will throw NullPointerException.
    - Synchronized methods in Java put a performance cost on your application. So use synchronization when it is absolutely required. Also, consider using synchronized code blocks for synchronizing only critical section of your code.
    - It’s possible that both static synchronized and non static synchronized method can run simultaneously or concurrently because they lock on different object.
    - According to the Java language specification you can not use synchronized keyword with constructor. It is illegal and result in compilation error.
    - Do not synchronize on non final field on synchronized block in Java. because reference of non final field may change any time and then different thread might synchronizing on different objects i.e. no synchronization at all. Best is to use String class, which is already immutable and declared final.