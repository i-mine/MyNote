---
title: Yaml
tags: skills
grammar_cjkRuby: true
---
time: `2019-12-11 `
[TOC]
### Yaml基本语法
- 大小写敏感
- 使用缩进表示层级关系
- 使用空格缩进,不允许使用TAB
- 缩进空格数不敏感,相同层级对齐
- `#`表示注释

### 数据结构
- 对象: key: vaue(value前必须有一个空格)
- 数组
``` yaml
array:
  - a
  - b
  - c
```
- 常量
``` yaml
# 数值可以直接表示
number: 2.3

# 布尔值用 true 或是 false 表示
isOnline: true

# null 用波浪线表示
isNull: ~

# 时间采用 ISO 8601 格式表示（时间和日期之间使用 T 连接，最后使用 + 代表时区）
time: 2019-08-01T19:02:31+08:00

# 日期用复合 ISO 8601 格式表示
data: 2019-08-01

# 字符串比较复杂
# 默认不用引号
str: shiyanlou
# 如果字符串中间有空格或是特殊字符时，字符串需要放在引号内（单双都可以）
str: 'this is a string'
# 如果字符串中间有单引号，需要用两个单引号进行转义处理
str: 'he''s name is liming'
```
### 特殊符号
#### ---和...
`---`表示一个文档开始,可以将多个文档写到一个文件中.
`...`表示一个文档的结束
```yaml
---
master:
   ip: 127.0.0.1
...
---
node
   ip: 127.0.0.2
```
#### !!
`!!`可以做类型转化

``` yaml
#把数字和布尔类型强转成字符串
str:
  - !!str 123
  - !!str true
```
#### * 和 & 
`&` 可以用来定义一个锚点,`*`可以引用锚点

```yaml
# 这里使用 &name 为 shixiaomei 设置了一个锚点（引用）
hr:
  - shixiaolou
  - &name shixiaomei
# 这里使用 *name 引用锚点，指代 shixiaomei
cto:
  - *name
  - shiyanlou
```

#### > 和 |
`>`表示折叠换行,`|`表示保留换行符

``` yaml
# 结果是 this: Foo\nBar\n
this: |
  Foo
  Bar
  
# 结果是 that: Foo Bar\n
that: >
  Foo
  Bar
#由上可见,字符串末尾的换行默认是保留的,可以在`|`添加`+`选择保留换行,`-`选择删除末尾换行.
#结果是: this: Foo\nBar 
this: |-
 Foo
 Bar
 
# 字符串可以写成多行,但是从第二行开始,必须有一个单空格缩进,换行符会自动被转为空格
# 结果:str: 这是一段 多行 字符串
str: 这是一段
  多行
  字符串
```