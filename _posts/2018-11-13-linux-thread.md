---
layout: post
title: "Linux下查看消耗CPU的线程"
subtitle: ''
author: "DylanYu"
header-style: text
tags:
  - Linux
  - 消耗CPU
  - 线程
---

javaweb 项目部署后发现很耗cpu，需要查出问题所在
写个测试程序，记相关步骤：

故意写个死循环
```
public class TestCpu {

    public static void main(String[] args) {
        while (true){
            new Object();
        }
    }
}
```

终端：
top
查看消耗cpu的进程 PID=2864
![image](https://dylanblog.github.io/img/in-post/2018-11-13-linux-thread-1.png)

ps -mp 2864 -o THREAD,tid,time 查看线程TID=2866
![image](https://dylanblog.github.io/img/in-post/2018-11-13-linux-thread-2.png)


把线程ID转为16进制
printf "%x\n" 2866

然后查看堆栈信息 。 （发现某些情况下不用将threadId转换成16进制，更加jstack输出确定）
jstack 2864 |grep b32 -A 30

![image](https://dylanblog.github.io/img/in-post/2018-11-13-linux-thread-3.png)
