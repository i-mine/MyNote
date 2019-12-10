---
title: Python pyenv应用 
tags: 
grammar_cjkRuby: true
---
time: `2019-11-27 `

```
export PYENV_ROOT="$HOME/cost_exporler_reporter/pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
```

冻结依赖

``` bash
pip freeze > requirements.txt
```

导入依赖

``` bash
pip install -r requirements.txt
```

## 安装pyhive前置依赖
```
报错信息：
sasl/sasl.h: No such file or directory
 
-- 解决办法：
# yum search sasl
 
-- 安装：
yum -y install cyrus-sasl cyrus-sasl-devel cyrus-sasl-lib
```

pyhive依赖问题:

I seem to have found further information in another libsasl-related question, and the solution in Cloudera's Python-SASL GitHub.

The problem is that the sasl Python package is linked to an older version of the native library: libsasl2.so.2, which existed on RHEL/CentOS 6. On RHEL/CentOS 7, there's libsasl2.so.3, installed by cyrus-sasl-lib into /usr/lib64/.

The solution is to reinstall the sasl Python package:
```
pip uninstall sasl
pip install sasl
```
## 打包问题
### zip压缩打包
加上-y可以把软链接也打包进去,防止执行命令丢失软链接
zip -ry ../execute.zip ./*

