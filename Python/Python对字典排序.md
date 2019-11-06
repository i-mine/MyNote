---
title: Python对字典排序
tags: python
grammar_cjkRuby: true
---
[TOC]

# 字典排序
## 按照value排序
首先我们需要统计一个列表中的数据个数:
```python
word_list = ["hadoop","sqoop","hadoop","spark","sqoop","hadoop"]
#对列表进行去重
word_set = set(word_list)
word_count = {}
for i in word_set:
	word_count[i] = word_list.count(i)

```
现在我们的word_count字典里面已经对数据进行了统计,但是无序的
```python?linenums
>>> word_list = ["hadoop","sqoop","hadoop","spark","sqoop","hadoop","flink"]
>>> word_set = set(word_list)
>>> print(word_set)
{'spark', 'sqoop', 'hadoop', 'flink'}
>>> word_count = {}
>>> for i in word_set:
...     word_count[i] = word_list.count(i)
... 
>>> 
>>> print(word_count)
{'spark': 1, 'sqoop': 2, 'hadoop': 3, 'flink': 1}

```

排序方法:
dict本身是无序的,所以只有先变成list
```python
>>> sort_count = sorted(word_count.items(), key=lambda x:x[1],reverse=True)
>>> print(sort_count)
[('hadoop', 3), ('sqoop', 2), ('spark', 1), ('flink', 1)]

```
reverse 为`True`可以是降序,为`False`则为升序
其中lambda中换成`x[0]`则可以按照key进行排序.

