---
title: Spark 源码阅读
tags: spark
grammar_cjkRuby: true
---
time: `2020-4-9 `
[TOC]

## 1. Shell
### 获取Sark根路径
``` shell
export SPARK_HOME="$(cd "`dirname "$0"`"/..; pwd)"

# "$0" 当前脚本名
# `dirname "$0"` 当前脚本目录
# $(cd "`dirname "$0"`"/..;pwd) 先切换到脚本父目录，并获取路径。就是获取Spark根目录路径
```
### 空值替换

``` shell
# 导入sparkconf目录,如果SPARK_CONF_DIR为空，返回${SPARK_HOME}/conf
export SPARK_CONF_DIR="${SPARK_CONF_DIR:-"${SPARK_HOME}/conf"}"
# 这里${value1:-value2} 表示当value1为空或者没有设置时，用value2替换 
```
### java路径判断

``` shell
# Find the java binary
# 加载java路径,赋值给RUNNER
if [ -n "${JAVA_HOME}" ]; then         # 判断java环境变量不为0,获取java路径
  RUNNER="${JAVA_HOME}/bin/java"
else
  if [ "$(command -v java)" ]; then    # 如果为0,查看java路径
    RUNNER="java"
  else
    echo "JAVA_HOME is not set" >&2    # 查看不到报错,并错误退出
    exit 1
  fi
fi

```

