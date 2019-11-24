---
title: Plantuml时序图中小技巧
tags: UML,skills
renderNumberedHeading: true
grammar_mermaid: true
grammar_plantuml: true
---
`时间`：2019-11-24

- 如果想改变箭头的颜色，可以通过在第一个`-`后面添加RGB的值来改变
```plantuml!
Bob -[#000]> Alice : hello
Alice -[#0000FF]->Bob : ok
```
- 如果希望消息可以编号，可以声明`autonumber`,来自动对消息排序
```plantuml!
Bob -[#000]> Alice : hello
Alice -[#0000FF]->Bob : ok
```
语句 autonumber [number1] [number2] 中number1用于指定编号的初始值，而 number2 可以同时指定编号的初始值和每次增加的值。
```plantuml!
autonumber
Bob -> Alice : Authentication Request
Bob <- Alice : Authentication Response

autonumber 15
Bob -> Alice : Another authentication Request
Bob <- Alice : Another authentication Response

autonumber 40 10
Bob -> Alice : Yet another authentication Request
Bob <- Alice : Yet another authentication Response
```

