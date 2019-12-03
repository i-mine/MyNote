---
title: Linux host配置
tags: Linux
grammar_cjkRuby: true
---
time: `2019-12-3`
执行 sudo 时可能会出现 “sudo: 无法解析主机: xxxxxx” 这样的提示，这是因为云主机的 hostname 不是 localhost，
而在 /etc/hosts 中定义了 127.0.0.1 localhost，这个提示可以忽略。
如果不想要终端出现这样的提示，可以执行以下命令 sudo hostname localhost。
可以强行指定hostname为localhost
