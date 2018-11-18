---
layout: post
title: "java工具-jmap"
subtitle: ''
author: "DylanYu"
header-style: text
tags:
  - Java
  - jmap
  - 内存分析
---

#### jmap
- 分析 JAVA Application的内存使用时，jmap是一个很实用的轻量级工具，可使用jmap查看heap概要情况，初略分析内存使用情况
- jmap可用于生产heap dump文件

##### 常用参数
- -heap 打印heap使用概要，可粗略的查看内存使用情况
    jmap -heap pid

````
fs@inspur92:~/test/llxdata/081005/tmp$ jmap -heap 30774
Attaching to process ID 30774, please wait...
Debugger attached successfully.
Server compiler detected.JVM version is 20.1-b02 

using thread-local object allocation.
Parallel GC with 8 thread(s) 

Heap Configuration:  
    MinHeapFreeRatio = 40   
    MaxHeapFreeRatio = 70   
    MaxHeapSize      = 1073741824 (1024.0MB)  
    NewSize          = 1310720 (1.25MB)   
    MaxNewSize       = 17592186044415 MB   
    OldSize          = 5439488 (5.1875MB)   
    NewRatio         = 2   
    SurvivorRatio    = 8   
    PermSize         = 21757952 (20.75MB)   
    MaxPermSize      = 268435456 (256.0MB) 

Heap Usage:PS Young GenerationEden Space:   
    capacity = 353107968 (336.75MB)   
    used     = 9083624 (8.662818908691406MB)   
    free     = 344024344 (328.0871810913086MB)
    2.572477775409475% used
From Space:   
    capacity = 2359296 (2.25MB)   
    used     = 0 (0.0MB)   
    free     = 2359296 (2.25MB)   
    0.0% used
To Space:   
    capacity = 2359296 (2.25MB)   
    used     = 0 (0.0MB)   
    free     = 2359296 (2.25MB)   
    0.0% used
PS Old Generation  
    capacity = 715849728 (682.6875MB)   
    used     = 47522208 (45.320709228515625MB)   
    free     = 668327520 (637.3667907714844MB)   
    6.638573172720407% used
PS Perm Generation   
    capacity = 40435712 (38.5625MB)   
    used     = 40067528 (38.21137237548828MB)   
    free     = 368184 (0.35112762451171875MB)   
    99.08945834810575% used
```

- -histo 生成类的统计报表

jmap -histo pid

```
MacBook-Pro:~ yuwei$ jmap -histo 33053

 num     #instances         #bytes  class name
----------------------------------------------
   1:          2018       17774416  [B
   2:         15938        1491584  [C
   3:          1692        1065312  [I
   4:          5853         431184  [Ljava.lang.Object;
   5:          3717         414688  java.lang.Class
   6:         15224         365376  java.lang.String
   7:          7257         232224  java.util.concurrent.ConcurrentHashMap$Node
   8:          6776         108416  java.lang.Object
   9:          3256         104192  java.util.HashMap$Node
  10:           914          80432  java.lang.reflect.Method
  11:          1435          68880  gnu.trove.THashMap
  12:            54          62976  [Ljava.util.concurrent.ConcurrentHashMap$Node;
  13:           377          43456  [Ljava.util.HashMap$Node;
  14:           593          42696  java.lang.reflect.Field
  15:          1046          41840  java.lang.ref.Finalizer
  16:            78          41184  io.netty.util.internal.shaded.org.jctools.queues.atomic.MpscAtomicArrayQueue
  17:           625          35000  java.util.zip.ZipFile$ZipFileInputStream
  18:          1410          33840  org.jetbrains.jps.model.ex.JpsElementContainerImpl
  19:           434          27776  io.netty.buffer.PoolSubpage
  20:          1112          25696  [Ljava.lang.Class;
  21:           561          22440  java.lang.ref.SoftReference
  22:           457          21936  java.util.HashMap
  23:           265          21200  java.lang.reflect.Constructor
  24:           518          20720  java.util.LinkedHashMap$Entry
```

统计信息中类型如果下：

 baseType| Type
 ---|---
    B|byte
    C|char
    D|double
    F|float
    I|int
    J|long
    L<className>|reference
    S|short
    Z|bool
    [|array

- -heap:format=b
    产生一个HeapDump文件，此为生成heapdump文件的重要参数。 例：jmap -heap:format=b 2657 会产生一个heap.bin的heapdump文件。 需要注意的是，此生成heapdump的参数为JDK1.5，在1.6中的格式为: jmap -dump:live,format=b,file=xxx 2657 这里更加强大一些，可以指定是存活的对象，还有生成heapdump的文件名。
    当被测应用使用内容较大时（4G以上），dump需要花费较长时间，很可能导致dump失败。


> 1. 如果程序内存不足或者频繁GC，很有可能存在内存泄露情况，这时候就要借助Java堆Dump查看对象的情况。
> 2. 要制作堆Dump可以直接使用jvm自带的jmap命令
> 3. 可以先使用jmap -heap命令查看堆的使用情况，看一下各个堆空间的占用情况。
> 4. 使用jmap -histo:[live]查看堆内存中的对象的情况。如果有大量对象在持续被引用，并没有被释放掉，那就产生了内存泄露，就要结合代码，把不用的对象释放掉。
> 5. 也可以使用 jmap -dump:format=b,file=<fileName>命令将堆信息保存到一个文件中，再借助jhat命令查看详细内容
> 6. 在内存出现泄露、溢出或者其它前提条件下，建议多dump几次内存，把内存文件进行编号归档，便于后续内存整理分析。


在ubuntu中第一次使用jmap会报错：Error attaching to process: sun.jvm.hotspot.debugger.DebuggerException: Can't attach to the process，这是oracla文档中提到的一个bug:http://bugs.java.com/bugdatabase/view_bug.do?bug_id=7050524,解决方式如下：
echo 0 | sudo tee /proc/sys/kernel/yama/ptrace_scope 该方法在下次重启前有效。
永久有效方法 sudo vi /etc/sysctl.d/10-ptrace.conf 编辑下面这行: kernel.yama.ptrace_scope = 1 修改为: kernel.yama.ptrace_scope = 0 重启系统，使修改生效。


**如果GC执行时间满足下面所有的条件，就意味着无需进行GC优化了。**
- Minor GC执行的很快（小于50ms）
- Minor GC执行的并不频繁（大概10秒一次）
- Full GC执行的很快（小于1s）
- Full GC执行的并不频繁（10分钟一次）