---
title: Plantuml和Mermaid使用 
tags: UML,skills
renderNumberedHeading: true
grammar_mermaid: true
grammar_plantuml: true
---
`时间`：2019-11-24

[TOC]


# Plantuml 
中文教程：http://plantuml.com/zh/starting

## 时序图
**什么是时序图？**
时序图（Sequence Diagram）是显示对象之间交互的图，这些对象是按时间顺序排列的。时序图中显示的是参与交互的对象及其对象之间消息交互的顺序。
时序图包括的建模元素主要有：对象（Actor）、生命线（Lifeline）、控制焦点（Focus of control）、消息（Message）等等。

plantuml中使用`->`,`-->`传递消息,仅适用于时序图。
每一次消息传递和动作说明都位于`:`之后。
### Plantuml时序图 HelloWorld
例子1:

```plantuml!
Alice -> Bob: Authentication Request
Bob --> Alice: Authentication Response

Alice -> Bob: Another authentication Request
Alice <-- Bob: another authentication Response
```
更多细致的展示效果另见：[Plantuml时序图中小技巧](https://github.com/i-mine/MyNote/blob/master/Skills/UML/Plantuml%E6%97%B6%E5%BA%8F%E5%9B%BE%E4%B8%AD%E5%B0%8F%E6%8A%80%E5%B7%A7.md
)
### 时序图中参与者
**参与者的类型有如下几种：**
- actor
- boundary
- control
- entity
- database
- collections

我们可以先通过定义参与者的类型和名字，然后通过`->`去关联先后关系。
```plantuml!
participant Foo0
actor Foo1
boundary Foo2
control Foo3
entity Foo4
database Foo5
collections Foo6
Foo0 -> Foo1:  To actor
Foo1 -> Foo2 : To boundary
Foo1 -> Foo3 : To control
Foo1 -> Foo4 : To entity
Foo1 -> Foo5 : To database
Foo1 -> Foo6 : To collections
```

在使用过程中，为了方便：
- 可以使用`as`去重命名参与者
- 可以通过RGB的值去更改参与者的颜色。
- 使用`’`或者`/''/`可以进行单行和多行注释。

```plantuml!
actor Bob #blue
' The only difference between actor
'and participant is the drawing
participant Alice
participant "I have a really\nlong name" as L #99FF99
/' You can also declare:
   participant L as "I have a really\nlong name"  #99FF99
  '/

Alice->Bob: Authentication Request
Bob->Alice: Authentication Response
Bob->L: Log transaction
```



## 类图


```plantuml!
title  decoupling
'skinparam packageStyle rect/' 加入这行代码，样式纯矩形'/
skinparam backgroundColor #EEEBDC
skinparam roundcorner 20
skinparam sequenceArrowThickness 2
'skinparam handwritten true
class Rxbus {
+IEventSubscriber iEventSubscriber
+addSubscriber(IEventSubscriber)
+sendMessageTo(Event)
}

interface IEventSubscriber
Rxbus --> IEventSubscriber
namespace com.xueshuyuan.cn.view #purple{

interface ILoginView{
+void loginWithPw()
+void loginByToken()
+void register()
+void success()
}
class LoginActivity<<接收反馈并更新UI>> {
+void loginWithPw()
+void loginByToken()
+void register()
+void success()
}

ILoginView <|--[#red] LoginActivity
}

namespace com.xueshuyuan.cn.presenter #orange{
interface ILoginPresenter{
+void loginByToken()
+void register()
}
class LoginPresenterImpl<<承接事件及接收通知处理并转发反馈>> {
+ILoginView iLoginView
+ILoginManager iLoginManager
+LoginPresenterImpl(ILoginView, ILoginManager)
+void loginWithPw()
+void loginByToken()
+void register()
+void receiveMessage(Event)
}

ILoginPresenter <|--[#red] LoginPresenterImpl
com.xueshuyuan.cn.view.LoginActivity <..[#red] LoginPresenterImpl :  Dependency
com.xueshuyuan.cn.moudle.LoginManagerImpl <..[#red] LoginPresenterImpl :  Dependency
.IEventSubscriber <|..[#red] ILoginPresenter
}


namespace com.xueshuyuan.cn.moudle #green{
interface ILoginManager{
+void loginWithPw()
+void loginByToken()
+void register()
+boolean checkUserExit()
+boolean checkPw()
}
class LoginManagerImpl<<承接事件及Rxbus发送通知>> {
+void loginWithPw()
+void loginByToken()
+void register()
+boolean checkUserExit()
+boolean checkPw()
+void sendMessageToXX(Event)
}

ILoginManager <|--[#red] LoginManagerImpl
}

```

