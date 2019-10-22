---
title:  ssh远程访问没有root权限
tags: Linux,issue
grammar_cjkRuby: true
---
# sudo:没有终端存在,且未指定 askpass 程序
问题描述: ansible远程通过ssh执行命令, 报告`sudo:没有终端存在,且未指定 askpass 程序`, 但是指定的ssh访问用户是可以在本地切成root权限.

解决方案:
```bash
sudo visudo
## 增加一行,指定用户mobdev不需要密码
mobdev  ALL=(ALL)  NOPASSWD:ALL

```


