---
title: Shell swithch用法
tags: shell
grammar_cjkRuby: true
---

time: `2020-3-27 `

[TOC]

``` shell?linenums&fancy=20
#! /bin/sh -

name=`basename $0 .sh`
case $1 in
 s|start)
        echo "start..."
        ;;
 stop)
        echo "stop ..."
        ;;
 reload)
        echo "reload..."
        ;;
 *)
        echo "Usage: $name [start|stop|reload]"
        exit 1
        ;;
esac
exit 0
————————————————
版权声明：本文为CSDN博主「love__coder」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/love__coder/article/details/7262160
```
注意：
- *) 相当于其他语言中的default。
-  除了*)模式，各个分支中;;是必须的，;;相当于其他语言中的break
-   | 分割多个模式，相当于or