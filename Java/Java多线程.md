---
title: Java多线程
tags: 
grammar_cjkRuby: true
---
time: `2020-3-12 `
[TOC]
参考:

多线程系列: https://segmentfault.com/t/%E5%A4%9A%E7%BA%BF%E7%A8%8B
什么是线程池: https://blog.csdn.net/xiao__gui/article/details/51064317
不要使用Executors: https://blog.csdn.net/u010321349/article/details/83927012?depth_1-utm_source=distribute.pc_relevant.none-task&utm_source=distribute.pc_relevant.none-task
``` java
ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue
```

![ThreadPool线程池图解](./attachments/1583994371087.drawio.html)