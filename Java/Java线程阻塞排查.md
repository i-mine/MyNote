---
title: Java线程阻塞排查
tags: java
grammar_cjkRuby: true
---
time: `2020-3-12 `
[TOC]

> 记一次使用Future.get(timeout)过程中线程超时未中断的经验.

## 1 线程排查
首先是如何排查当前堵住的线程来源于哪部分代码,我们这里使用jstack
比较完整的排查攻略: 
      https://juejin.im/post/5d6207c6f265da03b46bf933
	  https://blog.csdn.net/weiweicao0429/article/details/53185999
### 1.1 jps找到进程PID
![jps](https://raw.githubusercontent.com/i-mine/MyNote/master/images/1583982581660.png)

### 1.2 top -Hp pid查看进程当前启用的线程
![top -Hp pid](https://raw.githubusercontent.com/i-mine/MyNote/master/images/1583983048718.png)

这里按快捷键`shift`+`t`可以占用时间比较长的线程倒序排出来.
这里可以看到有两个线程正在跑,其中一个已经跑了13分钟,具体TIME+代表的含义13:25.61代表
就是`分钟`,`秒`,`毫秒`.

### 1.3 jstack -l pid > stack.txt
这里记录一次jvm进程所有的线程信息,然后将刚才找到的阻塞线程的pid转换成16进制.

``` shell
printf %x pid
```
jvm中的线程号都是根据16进制编码的,打开stack.txt,搜索这个16进制的线程号:
可以看到Future线程仍然在等待完成中,这和设置了超时退出不符,接下来查看Future线程中断的正确用法.
![enter description here](https://raw.githubusercontent.com/i-mine/MyNote/master/images/1583984541777.png)

## 2 线程池中使用Future中断线程
![enter description here](https://raw.githubusercontent.com/i-mine/MyNote/master/images/1583985090863.png)
这里注意,任务超时就会抛出TimeoutException,我们捕获到执行了fuutre.cancel(true)
![enter description here](https://raw.githubusercontent.com/i-mine/MyNote/master/images/1583985959401.png)
根据源码,这里确实是执行了线程中断,那么有什么问题呢,到底是什么导致线程超时执行呢?
> Thread.interrupt()方法不会中断一个正在运行的线程。这一方法实际上完成的是，设置线程的中断标示位，在线程受到阻塞的地方（如调用sleep、wait、join等地方）抛出一个异常InterruptedException，并且中断状态也将被清除，这样线程就得以退出阻塞的状态。
> [引用自:https://www.cnblogs.com/onlywujun/p/3565082.html]

所以我们这次知道interrupt并不会立即中断线程,而是异步中断的方式,而此次超时不中断执行是因为Future执行的task是一个耗时比较长的for循环:
![enter description here](https://raw.githubusercontent.com/i-mine/MyNote/master/images/1583992107305.png)
这种情况下,显然无法让线程退出中断,因为线程内部没有对中断做出反应.
这里有两种方式:
    **1. 加入sleep方法,即可让线程响应中断.**
     **2. 在循环体中判断线程状态.**
方式一:

``` java?linenums&fancy=3
public void doMoreThings() throws InterruptedException {
        for (; ;) {
                Thread.sleep(10);
                System.out.println("Don't bother me");
            }
        
    }
```
方式二:

``` java?linenums&fancy=3,7
 public void doMoreThings() throws InterruptedException {
        for ( ; ; ) {
            if (!Thread.currentThread().isInterrupted()) {
                System.out.println("Don't bother me");
            }else {
                System.out.println(Thread.currentThread().getName()+",Just stop your work, Now!");
                throw new InterruptedException();
            }
        }
    }
```
这样就成功解决此次线程超时阻塞的问题了.<i class="fas fa-coffee"></i>

