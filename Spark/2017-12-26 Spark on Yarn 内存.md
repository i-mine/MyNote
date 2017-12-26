---
title: 2017-12-26 Spark on Yarn 内存
tags: spark
grammar_cjkRuby: true
---
![spark on yarn container的内存分配][1]

图片来自[How-to: Tune Your Apache Spark Jobs (Part 2)][5]

![spark app  WEB UI][2]

![spark on yarn log][3]

根据以上三张图，可以知道spark app 申请的`Executor memory`并不等于它所在的`container memory`,显然`container memory` 需要更多的内存来给Executor，其中包括`MemoryOverhead`。

==MemoryOverhead==是JVM进程中除Java堆以外占用的空间大小，包括方法区（永久代）、Java虚拟机栈、本地方法栈、JVM进程本身所用的内存、直接内存（Direct Memory）等。通过spark.yarn.executor.memoryOverhead设置，单位MB。

  [1]: ./images/1514263225406.jpg
  [2]: ./images/%E6%B7%B1%E5%BA%A6%E6%88%AA%E5%9B%BE_%E9%80%89%E6%8B%A9%E5%8C%BA%E5%9F%9F_20171226120354.png "某一个spark app的Spark UI"
  [3]: ./images/1514262973079.jpg "该app 在yarn中的申请的资源"
  [5]:http://blog.cloudera.com/blog/2015/03/how-to-tune-your-apache-spark-jobs-part-2/