---
layout: post
title: "Java中启动线程start和run方法"
subtitle: ''
author: "DylanYu"
header-style: text
tags:
  - Java
  - Thread
  - Runnable
  - Thread.run
  - Thread.start
---

<a href="https://blog.csdn.net/lai_li/article/details/53070141" target="_blank">原文地址</a>


### 一、区别

Java中启动线程有两种方法，继承Thread类和实现Runnable接口，由于Java无法实现多重继承，所以一般通过实现Runnable接口来创建线程。但是无论哪种方法都可以通过start()和run()方法来启动线程，下面就来介绍一下他们的区别。

**start方法：**

    通过该方法启动线程的同时也创建了一个线程，真正实现了多线程。无需等待run()方法中的代码执行完毕，就可以接着执行下面的代码。此时start()的这个线程处于就绪状态，当得到CPU的时间片后就会执行其中的run()方法。这个run()方法包含了要执行的这个线程的内容，run()方法运行结束，此线程也就终止了。

**run方法：**

    通过run方法启动线程其实就是调用一个类中的方法，当作普通的方法的方式调用。并没有创建一个线程，程序中依旧只有一个主线程，必须等到run()方法里面的代码执行完毕，才会继续执行下面的代码，这样就没有达到写线程的目的。

下面我们通过一个很经典的题目来理解一下：

```
public class Test {
    public static void main(String[] args) {
        Thread t = new Thread(){
            public void run() {
                pong();
            }
        };
        t.run();
        System.out.println("ping");
    }

    static void pong() {
        System.out.println("pong");
    }
}
```

代码如图所示，那么运行程序，输出的应该是什么呢？没错，输出的是”pong ping”。因为t.run()实际上就是等待执行new Thread里面的run()方法调用pong()完毕后，再继续打印”ping”。它不是真正的线程。

而如果我们将t.run();修改为t.start();那么，结果很明显就是”ping pong”，因为当执行到此处，创建了一个新的线程t并处于就绪状态，代码继续执行，打印出”ping”。此时，执行完毕。线程t得到CPU的时间片，开始执行，调用pong()方法打印出”pong”。

如果感兴趣，还可以多加几条语句自己看看效果。

### 二、源码

那么他们本质上的区别在哪里，我们来看一下源码：

```
/**java
     * Causes this thread to begin execution; the Java Virtual Machine
     * calls the <code>run</code> method of this thread.
     * <p>
     * The result is that two threads are running concurrently: the
     * current thread (which returns from the call to the
     * <code>start</code> method) and the other thread (which executes its
     * <code>run</code> method).
     * <p>
     * It is never legal to start a thread more than once.
     * In particular, a thread may not be restarted once it has completed
     * execution.
     *
     * @exception  IllegalThreadStateException  if the thread was already
     *               started.
     * @see        #run()
     * @see        #stop()
     */
    public synchronized void start() {
        /**
         * This method is not invoked for the main method thread or "system"
         * group threads created/set up by the VM. Any new functionality added
         * to this method in the future may have to also be added to the VM.
         *
         * A zero status value corresponds to state "NEW".
         */
        if (threadStatus != 0)
            throw new IllegalThreadStateException();

        /* Notify the group that this thread is about to be started
         * so that it can be added to the group's list of threads
         * and the group's unstarted count can be decremented. */
        group.add(this);

        boolean started = false;
        try {
            start0();
            started = true;
        } finally {
            try {
                if (!started) {
                    group.threadStartFailed(this);
                }
            } catch (Throwable ignore) {
                /* do nothing. If start0 threw a Throwable then
                  it will be passed up the call stack */
            }
        }
    }

    private native void start0();
```

可以看到，当一个线程启动的时候，它的状态（threadStatus）被设置为0，如果不为0，则抛出IllegalThreadStateException异常。正常的话，将该线程加入线程组，最后尝试调用start0方法，而start0方法是私有的native方法（Native Method是一个java调用非java代码的接口）。

我猜测这里是用C实现的，看来调用系统底层还是要通过C语言。这也就是为什么start()方法可以实现多线程。而调用run()方法，其实只是调用runnable里面自己实现的run()方法。

我们再看看Thread里run()的源码：

```
@Override
    public void run() {
        if (target != null) {
            target.run();
        }
    }
```

如果target不为空，则调用target的run()方法，那么target是什么：

```
/* What will be run. */
    private Runnable target;
```

其实就是一个Runnable接口，正如上面代码中new Thread的部分，其实我们就是在实现它的run()方法。所以如果直接调用run，就和一个普通的方法没什么区别，是不会创建新的线程的，因为压根就没执行start0方法。

### 三、实现

前面说了，继承Thread类和实现Runnable接口都可以定义一个线程，那么他们又有什么区别呢？
在《Java核心技术卷1 第9版》第627页提到。可以通过一下代码构建Thread的子类定义一个线程：

```
class MyThread extends Thread {
    public void run() {
        //do Something
    }
}
```

然后，实例化一个对象，调用其start方法。不过这个方法不推荐。应该减少需要并行运行的任务数量。如果任务很多，要为每个任务创建一个独立的线程所付出的代价太多，当然可以用线程池来解决。

实现Runnable接口所具有的优势：
- 避免Java单继承的问题
- 适合多线程处理同一资源
- 代码可以被多线程共享，数据独立，很容易实现资源共享
