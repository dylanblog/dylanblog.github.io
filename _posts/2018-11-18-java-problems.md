---
layout: post
title: "直播一次问题排查过程"
subtitle: ''
author: "DylanYu"
header-style: text
tags:
  - Java
  - JVM
  - top
  - jmap
  - jinfo
  - jstat
  - Good
---

<a href="http://www.importnew.com/28754.html" target="_blank">原文地址</a>

现象

收到系统报警，查看一台机器频繁FULLGC，且该服务超时。
这是一台4核8G的机器, 使用jdk1.8.0_45-b14。

我们可以直接通过jstat等来观察。这次我先通过CPU开始。
top查看后该java进程的运行状况为
```
Tasks: 161 total,   3 running, 158 sleeping,   0 stopped,   0 zombie
Cpu(s): 32.1%us,  1.9%sy,  0.0%ni, 65.9%id,  0.0%wa,  0.0%hi,  0.1%si,  0.0%st
Mem:   8059416k total,  7733088k used,   326328k free,   147536k buffers
Swap:  2096440k total,        0k used,  2096440k free,  2012212k cached
PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
15178 x   20   0 7314m 4.2g  10m S 98.9 55.1   4984:05 java
```
RES 占用4.2g CPU占用98%
```
top -H -p 15178
```
后发现
```
PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
15234 x   20   0 7314m 4.2g  10m R 70.2 55.1 781:50.32 java
15250 x   20   0 7314m 4.2g  10m S 52.9 55.1 455:12.27 java
```
一个PID为 15234 的轻量级进程，对应于java中的线程的本地线程号。
通过printf '%x\n' 15234计算出16进制的 3b82
通过jstack -l 15178 然后搜索该线程,发现其为
"Concurrent Mark-Sweep GC Thread" os_prio=0 tid=0x00007ff3e4064000 nid=0x3b82 runnable

定位了确实由GC导致的系统不可用。
```
jinfo -flags
```
查看该VM参数
```
Non-default VM flags: -XX:CICompilerCount=3 -XX:CMSFullGCsBeforeCompaction=0 -XX:CMSInitiatingOccupancyFraction=80 -XX:+DisableExplicitGC -XX:ErrorFile=null -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=null -XX:InitialHeapSize=2147483648 -XX:MaxHeapSize=2147483648 -XX:MaxNewSize=348913664 -XX:MaxTenuringThreshold=6 -XX:MinHeapDeltaBytes=196608 -XX:NewSize=348913664 -XX:OldPLABSize=16 -XX:OldSize=1798569984 -XX:+PrintCommandLineFlags -XX:+PrintGC -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+UseCMSCompactAtFullCollection -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseConcMarkSweepGC -XX:+UseParNewGC
```
jmap -heap 查看统计结果

```
Debugger attached successfully.
Server compiler detected.
JVM version is 25.45-b02
using parallel threads in the new generation.
using thread-local object allocation.
Concurrent Mark-Sweep GC
Heap Configuration:
   MinHeapFreeRatio         = 40
   MaxHeapFreeRatio         = 70
   MaxHeapSize              = 2147483648 (2048.0MB)
   NewSize                  = 348913664 (332.75MB)
   MaxNewSize               = 348913664 (332.75MB)
   OldSize                  = 1798569984 (1715.25MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)
Heap Usage:
New Generation (Eden + 1 Survivor Space):
   capacity = 314048512 (299.5MB)
   used     = 314048512 (299.5MB)
   free     = 0 (0.0MB)
   100.0% used
Eden Space:
   capacity = 279183360 (266.25MB)
   used     = 279183360 (266.25MB)
   free     = 0 (0.0MB)
   100.0% used
From Space:
   capacity = 34865152 (33.25MB)
   used     = 34865152 (33.25MB)
   free     = 0 (0.0MB)
   100.0% used
To Space:
   capacity = 34865152 (33.25MB)
   used     = 0 (0.0MB)
   free     = 34865152 (33.25MB)
   0.0% used
concurrent mark-sweep generation:
   capacity = 1798569984 (1715.25MB)
   used     = 2657780412087605496 (2.5346569176555684E12MB)
   free     = 15057529128475 MB
   1.4777186518907263E11% used
32778 interned Strings occupying 3879152 bytes.
```
使用jstat -gcutil 15178 1s观察一段时间GC状况
```
jstat -gccause 15178 1s
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT    LGCC                 GCC
100.00   0.00 100.00 100.00  97.78  95.59   1906   26.575 61433 217668.188 217694.763 Allocation Failure   Allocation Failure
  0.00   0.00  96.51 100.00  97.78  95.59   1906   26.575 61433 217672.991 217699.566 Allocation Failure   No GC
100.00   0.00 100.00 100.00  97.78  95.59   1906   26.575 61434 217672.991 217699.566 Allocation Failure   Allocation Failure
100.00   0.00 100.00 100.00  97.78  95.59   1906   26.575 61434 217672.991 217699.566 Allocation Failure   Allocation Failure
100.00   0.00 100.00 100.00  97.78  95.59   1906   26.575 61434 217672.991 217699.566 Allocation Failure   Allocation Failure
```
可以看到Old区满了，并且Eden区域的对象没有触发YGC直接晋升到Old区中，但是Full GC没有释放出空间。这是由于在当老年代的连续空间小于新生代所有对象大小时，MinorGC前会检查下平均每次晋升到Old区的大小是否大于Old区的剩余空间，如果大于或者当前的设置HandlePromotionFailure为false则直接触发FullGc,否则会先进行MinorGC。
关于FullGC和MajorGC的区别，可以不要太纠结.

jmap -histo 15178 | less 查看一下对象实例数量和空间占用
看到前面的一种数据各占用几百兆内存。总和在1935483656，和堆空间基本相同。
```
num     #instances         #bytes  class name
----------------------------------------------
   1:      14766305      796031864  [C
   2:      14763842      354332208  java.lang.String
   3:       8882440      213178560  java.lang.Long
   4:       1984104      174601152  com.x.x.x.model.Order
   5:       3994139       63906224  java.lang.Integer
   6:       1984126       63492032  java.util.concurrent.FutureTask
   7:       1984371       47624904  java.util.Date
   8:       1984363       47624712  java.util.concurrent.LinkedBlockingQueue$Node
   9:       1984114       47618736  java.util.concurrent.Executors$RunnableAdapter
  10:       1984104       47618496  com.x.x.fyes.service.impl.OrderServiceImpl$$Lambda$11/284227593
  11:        262144       18874368  org.apache.logging.log4j.core.async.RingBufferLogEvent
  12:          7841       15312288  [B
  13:         17412        8712392  [Ljava.lang.Object;
  14:        262144        6291456  org.apache.logging.log4j.core.async.AsyncLoggerConfigHelper$Log4jEventWrapper
  15:         12116        4299880  [I
  16:         99594        3187008  java.util.HashMap$Node
  17:         16318        1810864  java.lang.Class
  18:          2496        1637376  io.netty.util.internal.shaded.org.jctools.queues.MpscArrayQueue
  19:         49413        1185912  java.net.InetSocketAddress$InetSocketAddressHolder
  20:         49322        1183728  java.net.InetAddress$InetAddressHolder
  21:         49321        1183704  java.net.Inet4Address
  22:          6116        1134384  [Ljava.util.HashMap$Node;
  23:         49412         790592  java.net.InetSocketAddress
  24:          6249         549912  java.lang.reflect.Method
  25:         11440         457600  java.util.LinkedHashMap$Entry
  26:           704         431264  [Ljava.util.WeakHashMap$Entry;
  27:         12680         405760  java.util.concurrent.ConcurrentHashMap$Node
  28:          6286         352016  java.util.LinkedHashMap
  29:          9272         296704  java.lang.ref.WeakReference
  30:           139         281888  [Ljava.nio.channels.SelectionKey;
  31:           616         258464  [Ljava.util.concurrent.ConcurrentHashMap$Node;
  32:          5709         228360  java.lang.ref.SoftReference
  33:          3840         217944  [Ljava.lang.String;
  34:          4493         215664  java.util.HashMap
  35:            65         210040  [Ljava.nio.ByteBuffer;
  36:           859         188144  [Z
  37:          5547         177504  java.util.concurrent.locks.ReentrantLock$NonfairSync
  38:          4391         175640  java.util.TreeMap$Entry
  39:           404         174400  [Lio.netty.util.Recycler$DefaultHandle;
  40:          4348         173920  java.util.WeakHashMap$Entry
  41:          4096         163840  org.jboss.netty.util.internal.ConcurrentIdentityHashMap$Segment
  42:          2033         162640  java.lang.reflect.Constructor
  43:          6489         155736  java.util.ArrayList
  44:          3750         150000  java.lang.ref.Finalizer
```
主要寻找这个列表中的业务对象和集合对象
其中的Order和OrderServiceImpl$$Lambda$11/284227593引起了我的注意。
找到该位置代码后，其代码为
```
private ExecutorService executorService = Executors.newFixedThreadPool(10, new DefaultThreadFactory("cacheThread"));
    @Override
    public Order save(Order order) {
        order.setCreated(new Date(order.getCreateTime()));
        Order save = orderRepository.save(order);
        executorService.submit(() -> {
            orderCacheService.cacheOrder(save);
            OrderModel orderModel = new OrderModel();
            BeanUtils.copyProperties(save, orderModel);
            Result result = fyMsgOrderService.saveOrder(orderModel);
            LOGGER.info("Msg Return {}", JSON.toJSONString(result));
        });
        return save;
    }
```
大体逻辑是先保存到ES中，然后使用线程数量无界队列大小为10的固定线程池执行保存到远程缓存以及使用RPC发送给另一个服务。
这段代码写的有些随性。
dump后等待下载dump文件的同时, 在thread stack中查找这个方法所在类的相关线程状态
基本如下所示:
```
"cacheThread-1-3" #152 prio=5 os_prio=0 tid=0x00007ff32e408800 nid=0x3c24 waiting on condition [0x00007ff2f33f4000]
   java.lang.Thread.State: TIMED_WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x0000000089acd400> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:215)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:2078)
        at java.util.concurrent.ArrayBlockingQueue.poll(ArrayBlockingQueue.java:418)
        at com.github.liuzhengyang.simplerpc.core.RpcClientWithLB.sendMessage(RpcClientWithLB.java:251)
        at com.github.liuzhengyang.simplerpc.core.RpcClientWithLB$2.invoke(RpcClientWithLB.java:280)
        at com.sun.proxy.$Proxy104.saveOrder(Unknown Source)
        at com.x.x.fyes.service.impl.OrderServiceImpl.lambda$save$0(OrderServiceImpl.java:48)
        at com.x.x.fyes.service.impl.OrderServiceImpl$$Lambda$11/284227593.run(Unknown Source)
        at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
        at java.util.concurrent.FutureTask.run(FutureTask.java:266)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at io.netty.util.concurrent.DefaultThreadFactory$DefaultRunnableDecorator.run(DefaultThreadFactory.java:144)
        at java.lang.Thread.run(Thread.java:745)
   Locked ownable synchronizers:
        - <0x00000000990675e8> (a java.util.concurrent.ThreadPoolExecutor$Worker)
```
可见很多线程都在发送完成RPC请求后，在RPC结果队列中等待该消息返回结果。
这时查看RPC提供者的状态, 服务所在机器的负载比较低，该提供者的日志已经不再刷新，但是curl localhost:8080能得到相应。最后的几行日志中显示
```
2017-03-18 19:19:38.725 ERROR 17977 --- [ntLoopGroup-3-1] c.g.l.simplerpc.core.RpcServerHandler    : Exception caught on [id: 0xa104b32a, L:/10.4.124.148:8001 - R:/10.12.74.172:53722],
io.netty.util.internal.OutOfDirectMemoryError: failed to allocate 16777216 byte(s) of direct memory (used: 1828716544, max: 1834483712)
    at io.netty.util.internal.PlatformDependent.incrementMemoryCounter(PlatformDependent.java:631) ~[netty-all-4.1.7.Final.jar!/:4.1.7.Final]
    at io.netty.util.internal.PlatformDependent.allocateDirectNoCleaner(PlatformDependent.java:585) ~[netty-all-4.1.7.Final.jar!/:4.1.7.Final]
    at io.netty.buffer.PoolArena$DirectArena.allocateDirect(PoolArena.java:709) ~[netty-all-4.1.7.Final.jar!/:4.1.7.Final]
    at io.netty.buffer.PoolArena$DirectArena.newChunk(PoolArena.java:698) ~[netty-all-4.1.7.Final.jar!/:4.1.7.Final]
    at io.netty.buffer.PoolArena.allocateNormal(PoolArena.java:237) ~[netty-all-4.1.7.Final.jar!/:4.1.7.Final]
    at io.netty.buffer.PoolArena.allocate(PoolArena.java:213) ~[netty-all-4.1.7.Final.jar!/:4.1.7.Final]
    at io.netty.buffer.PoolArena.allocate(PoolArena.java:141) ~[netty-all-4.1.7.Final.jar!/:4.1.7.Final]
    at io.netty.buffer.PooledByteBufAllocator.newDirectBuffer(PooledByteBufAllocator.java:287) ~[netty-all-4.1.7.Final.jar!/:4.1.7.Final]
    at io.netty.buffer.AbstractByteBufAllocator.directBuffer(AbstractByteBufAllocator.java:179) ~[netty-all-4.1.7.Final.jar!/:4.1.7.Final]
    at io.netty.buffer.AbstractByteBufAllocator.directBuffer(AbstractByteBufAllocator.java:170) ~[netty-all-4.1.7.Final.jar!/:4.1.7.Final]
    at io.netty.buffer.AbstractByteBufAllocator.ioBuffer(AbstractByteBufAllocator.java:131) ~[netty-all-4.1.7.Final.jar!/:4.1.7.Final]
    at io.netty.channel.DefaultMaxMessagesRecvByteBufAllocator$MaxMessageHandle.allocate(DefaultMaxMessagesRecvByteBufAllocator.java:73) ~[netty-all-4.1.7.Final.jar!/:4.1.7.Final]
    at io.netty.channel.nio.AbstractNioByteChannel$NioByteUnsafe.read(AbstractNioByteChannel.java:117) ~[netty-all-4.1.7.Final.jar!/:4.1.7.Final]
    at io.netty.channel.nio.NioEventLoop.processSelectedKey(NioEventLoop.java:642) [netty-all-4.1.7.Final.jar!/:4.1.7.Final]
    at io.netty.channel.nio.NioEventLoop.processSelectedKeysOptimized(NioEventLoop.java:565) [netty-all-4.1.7.Final.jar!/:4.1.7.Final]
    at io.netty.channel.nio.NioEventLoop.processSelectedKeys(NioEventLoop.java:479) [netty-all-4.1.7.Final.jar!/:4.1.7.Final]
    at io.netty.channel.nio.NioEventLoop.run(NioEventLoop.java:441) [netty-all-4.1.7.Final.jar!/:4.1.7.Final]
    at io.netty.util.concurrent.SingleThreadEventExecutor$5.run(SingleThreadEventExecutor.java:858) [netty-all-4.1.7.Final.jar!/:4.1.7.Final]
    at io.netty.util.concurrent.DefaultThreadFactory$DefaultRunnableDecorator.run(DefaultThreadFactory.java:144) [netty-all-4.1.7.Final.jar!/:4.1.7.Final]
    at java.lang.Thread.run(Thread.java:745) [na:1.8.0_45]
2017-03-18 19:19:38.725  INFO 17977 --- [ntLoopGroup-3-1] io.netty.handler.logging.LoggingHandler  : [id: 0xa104b32a, L:/10.4.124.148:8001 - R:/10.12.74.172:53722] CLOSE
2017-03-18 19:19:38.725  INFO 17977 --- [ntLoopGroup-3-1] io.netty.handler.logging.LoggingHandler  : [id: 0xa104b32a, L:/10.4.124.148:8001 ! R:/10.12.74.172:53722] INACTIVE
2017-03-18 19:19:38.725  INFO 17977 --- [ntLoopGroup-3-1] io.netty.handler.logging.LoggingHandler  : [id: 0xa104b32a, L:/10.4.124.148:8001 ! R:/10.12.74.172:53722] UNREGISTERED
```
可见在申请DirectMemory时遇到了OOM Error，这个异常是netty内部的异常，所以没有导致进程退出。。google该异常message后发现了很多类似的关于netty的issue。

- https://github.com/netty/netty/issues/6221
- 但是simple-rpc框架中没有使用ssl
- 检查netty版本为4.1.7

一个ThreadExecutor的包括引用的总大小占据了1.7g，查看引用该对象的线程的线程栈和之前猜测的一致。

这个实例的ourgoing reference中指向workQueue的大小基本占据了1.7g。这也提醒了我们在使用Executor或Queue要考虑队列的长度问题，是否要设计长度以及溢出时如何处理。

但是第二个系统的内存状况很奇怪，heap各个区域都很少，异常日志显示DirectMemory申请失败。dump下的内存只有500M左右，top中查看进程使用的物理内存在2.6G左右。查看异常处代码可以看到当没有使用io.netty.maxDirectMemory参数设置Netty最大使用的DirectMemory大小时。会自动选择一个最大大小。
在线程栈中可以看出这个发生在Accecptor收到Selector可处理任务后，在其中读数据时，为什么会申请16777216bytes 大约在16M的一个直接内存呢？
在其中的AbstractNioByteChannel中
```
byteBuf = allocHandle.allocate(allocator);
```
而allocHandle的实现DefaultMaxMessagesRecvByteBufAllocator
```
@Override
public ByteBuf allocate(ByteBufAllocator alloc) {
    return alloc.ioBuffer(guess());
}
```
申请buffer的initialCapacity是通过guess()方法得到的。
guess实现在AdaptiveRecvByteBufAllocator中，这个类通过反馈的方式调整下次申请的buffer大小。调整的大小数组是前32小的值都是每次增加16byte, 达到512byte后按照每次乘二的方式。如果实际读到的数据小于上一个数组位置的值，下次申请回收缩，相反大于后面的数组位置时也会进行增加，不过增加的间隔是4，也就是出现16M的情况之前的申请大小是1M并且实际读到的数据大于等于2M。

上述日志表明，当前已经申请的DirectMemory加上将要申请的16M左右DirectMemory超过了1.8G的DirectMemoryLimit。

那netty中使用的DirectByteBuffer什么时候进行释放呢,需要仔细研究下netty代码了。
可以在PlatformDependent中看到与incrementMemoryCounter相对的还有decrementMemoryCounter方法负责减少DIRECT_MEMORY_COUNTER的值，其中freeDirectNoCleaner方法调用了UNSAFE.freeMemory(address)进行直接内存的释放, 跟踪调用链找到了PoolArena的free和reallocate方法，reallocate会在PooledByteBuf中调用capacity进行。

现在通过设置-Dio.netty.maxDirectMemory=0并增加-Dio.netty.leakDetectionLevel=advanced继续观察。

附一段查看JDK8的DirectMemory的程序 https://gist.github.com/liuzhengyang/a0d25510d706c6f4c0805b367ad502da ，使用方式见https://gist.github.com/rednaxelafx/1593521