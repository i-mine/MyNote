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
#### 参与者用法
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
#### 参与者的使用技巧
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

参与者可以给自己发送消息,消息可以通过\n换行.

```plantuml!
Alice->Alice: This is a signal to self.\n Italso demonstrates \n multiline.
```

### 时序图Page样式
#### Page title,header,footer
我们可以通过
title [Title Content]
header [Header Content]
footer [Footor Content]
来定义时序图中单个Page中的title,header,footer

```plantuml!
header Page Header
footer Page %page% of %lastpage%

title Example Title

Alice -> Bob : message 1
Alice -> Bob : message 2
```
#### 时序图中的分割方法
可以使用`==`关键词来分割多个步骤.

```plantuml!
== Initialization ==

Alice -> Bob: Authentication Request
Bob --> Alice: Authentication Response

== Repetition ==

Alice -> Bob: Another authentication Request
Alice <-- Bob: another authentication Response
```
#### 延迟
可以使用`...`表示延迟,并且可以在中间添加注释.

``` plantuml!
Alice -> Bob: Authentication Request
...
Bob --> Alice: Authentication Response
...5 minutes latter...
Bob --> Alice: Bye !
```

### 生命线的激活与撤销
#### 生命线的使用
生命线用来表示一个参与者存活的时间段,通过竖着的矩形条来表示参与者参与的时间点和参与的时间段.
分别使用activate,deactivate来表示参与和退出参与,destory则表示生命线的终结.
``` plantuml!
User -> A: DoWork
activate A

A -> B: << createRequest >>
activate B

B -> C: DoWork
activate C
C --> B: WorkDone
destroy C

B --> A: RequestCreated
deactivate B

A -> User: Done
deactivate A
```
#### 创建参与者
可以通过把 create 放在第一次接受消息之前,用来强调本次消息在创建新的对象.

``` plantuml!
ob -> Alice : hello

create Other
Alice -> Other : new

create control String
Alice -> String
note right : You can also put notes!

Alice --> Bob : ok
```
#### 激活，停用，创建的快捷语法
- `++` 激活->之后的参与者的生命线.
- `--` 停用->之前的参与者的生命线
-  `**` 用来表示新建参与者
-  `!!` 用来表示销毁参与者

为了方便停用激活的生命线(deactivate),可以使用return message,来终止一个最近的生命线.

``` plantuml!
alice -> bob ++ : hello
bob -> bob ++ : self call
bob -> bib ++  #005500 : hello
bob -> george ** : create
return done
return rc /'用来撤销bob->bob的生命线'/
bob -> george !! : delete
return success
```
#### 进入和发出消息
如果只想关注部分图示，你可以使用进入和发出箭头。

使用方括号`[`和`]`表示图示的左、右两侧。

``` plantuml!
[->A  : DoWork
activate A
A->A : Internal Call
activate A
A->] : <<createRequest>>
A<--]: RequestCreated
deactivate A
[<- A: done
```
#### 包裹参与者
可以使用box和end box画一个盒子来将多个参与者包裹起来.

``` plantuml!
box "Internal Service" #LightBlue
	participant Bob
	participant Alice
end box
participant Other

Bob -> Alice : hello
Alice -> Other : hello
```

## 类图
> 类图显示了系统的静态结构.


**类图中的主要元素分为三层:** 
- 上层: 类名
- 中层: 属性
- 下层: 方法

**类图之间的6种关系:**
- 关联
- 依赖
- 聚合
- 组合
- 泛化
- 实现

为了说明这6种关系可以通过以下一个类图进行说明:
![enter description here](https://raw.githubusercontent.com/i-mine/MyNote/master/images/1574665814549.png)

- 首先车的类图结构为一个抽象类,`<<abstract>>`
- 它有两个继承类:小汽车和自行车;它们之间的关系为实现关系,使用空心箭头的虚线表示.
- 小汽车和SUV之间是继承关系,它们之间的关系为泛化关系,使用带空心箭头的实线表示.
- 小汽车和发动机之间是组合关系,使用带实心箭头的实线表示.
- 学生与班级之间是聚合关系,使用带空心箭头的实线表示.
- 学生与自行车是一种依赖关系,使用带箭头的虚线表示.
- 学生与身份证之间是关联关系,使用一条带箭头实线表示.

plantuml中6种关系的符号:

|  实现   | 泛化     | 组合   | 聚合  | 依赖     |关联     |
| --- | --- | --- | --- | --- | --- |
| Realization    | Generalization     | Composition    | Aggregation|  Dependency  |  Association   |
|   `<|..`   | `<|--`      | `*--`    | `o--`|  `<..`  |  `<--`  |

以上是常见的六种关系，--可以替换成..就可以得到虚线。另外，其中的符号是可以改变方向的，例如：<|--表示右边的类泛化左边的类；--|>表示左边的类泛化右边的类。

``` plantuml!
ClassA <-- ClassB:关联
ClassA <.. ClassB : 依赖
ClassA o-- ClassB:聚合
ClassA *-- ClassB:组合
ClassA <|-- ClassB:泛化
ClassA <|.. ClassB:实现
```
其中
`实现`体现在代码中是指接口的实现(Java)或者抽象类的继承(C++).
`泛化`体现在代码中是指非抽象类之间的继承.
`聚合`,`组合`,`关联`体现在代码中是指成员变量, 是组合和聚合,还是关联需要看实际的实体逻辑.
`依赖`体现在代码中是指某个类的方法使用另一个类的对象作为参数.

### 普通定义

``` plantuml!
Class China {
	String area
	int rivers
	long person
	class Beigjing{}
	interface rich{}
	String getArea()
	long getPerson()
}
```
### 可见性定义
- `-`表示private
- `#`表示protected
- `~`表示package private(默认为包内私有)
- `+`表示public

``` plantuml!
Class China {
	- String area /'-表示private'/
	# int rivers /'#表示protected'/
	+ long person /'+ 表示public'/
	class Beijing{}
	interface rich{}
	
	~ String getArea() /'表示package private'/
	+ long getPerson
}
```
### 抽象和静态
可以使用{static}或者{abstract}来修饰字段或者方法,修饰符需要在行的开头或者末尾使用.
其中 static 修饰的成员变量显示为下划线修饰,抽象方法显示为斜体.
```plantuml!
class DBUtil {
	{static} Driver driver
	{abstract} Session getSession()
}
```
### 类主体自定义
默认情况下类是分为三层的,可以通过`..`,`--`,`==`,`__`进行自定义分层.

``` plantuml!
class Foo1 {
you can use servaral lines
..
first group
==
second group
__
third group
--
fourth group
}

class Foo2{
..Getter..
+ String getName()
+ long getId()
..Setter..
+ setName()
+ setId()
__ private data __
- int age
-- encrypted --
- String password
}
```
### 类图的注释
原型使用class、<<和>>进行定义。
注释分为3类:
- 类注释:注释使用note left of、note right of、note top of、note bottom of关键字进行定义。
- 连接注释: note on link，你可以使用note left on link、note right on link、note top on link、note bottom on link来改变注释的位置。
- 类对象连接注释:注释可以使用..与其他对象进行连接。
``` plantuml!
class MainActivity
note left:左侧注明用途
note right of MainActivity:右侧注明用途
note top of MainActivity:上面注明用途
note bottom of MainActivity:下面注明用途

class List<<general>>
note top of List : 接口类型,xxList extends it

class ArrayList
note left : 基于长度可变的数组的列表
note "This is a floating note" as N1
note "Collection 的衍生接口和类" as NOTE
List .. NOTE
NOTE .. ArrayList

List <|-- ArrayList
note right on link: ArrayList 继承自 List
```
### 抽象类和接口
可以使用abstract或者interface来定义抽象类或者接口，也可以使用annotation、enum关键字来定义注解或者枚举。
抽象类和接口的使用:
``` plantuml!
abstract class AbstractList
abstract class AbstractCollection
interface List
interface Collection
List <|-- AbstractList
Collection <|-- AbstractCollection
Collection <|-- List
AbstractCollection <|--AbstractList
AbstractList<|--ArrayList

class ArrayList {
Object [] elementData
size()
}
```
注解和枚举的使用:

``` plantuml!
enum TimeUnit {
  DAYS
  HOURS
  MINUTES
}

annotation SuppressWarnings

```
### 使用泛型

``` plantuml!
class Bus<? extends Car> {
  int size()
}
Bus *- Wheel

```

### 定义包

```plantuml!
package "java.util" #LightBlue{
	Interface Iterable<E>
	Interface Collection<E>
	abstract AbstractCollection
}
package "java.lang" #yellow {
	class Object
}
AbstractCollection --|> Object : extends
AbstractCollection ..|> Collection : implements
Collection -|> Iterable : extends
```
### 命名空间
在使用包（package）时（区别于命名空间），类名是类的唯一标识。 也就意味着，在不同的包（package）中的类，不能使用相同的类名。
在那种情况下（译注：同名、不同全限定名类），你应该使用命名空间来取而代之。
==你可以从其他命名空间，使用全限定名来引用类， 默认命名空间（译注：无名的命名空间）下的类，以一个“."开头（的类名）来引用（译注：示例中的BaseClass).==


``` plantuml!
namespace net.dummy #DDDDDD {
    Meeting o-- net.foo.Person
    
    .BaseClass <|- Meeting
}

namespace net.foo {
  net.dummy.Person  --|> Person
  

  net.dummy.Meeting o-- Person
}
namespace net.unused {
	interface Person
	net.foo.Person ..|> Person
}


```

### 类图样例

#### 样例1:

``` plantuml!
title Java Exception Layer
namespace java.lang #DDDDDD {
    class Error << unchecked >>
    Throwable <|-- Error
    Throwable <|-- Exception
    Exception <|-- CloneNotSupportedException
    Exception <|-- RuntimeException
    RuntimeException <|-- ArithmeticException
    RuntimeException <|-- ClassCastException
    RuntimeException <|-- IllegalArgumentException
    RuntimeException <|-- IllegalStateException
    Exception <|-- ReflectiveOperationException
    ReflectiveOperationException <|-- ClassNotFoundException
}
  
namespace java.io #DDDDDD {
    java.lang.Exception <|-- IOException
    IOException <|-- EOFException
    IOException <|-- FileNotFoundException
}
  
namespace java.net #DDDDDD {
    java.io.IOException <|-- MalformedURLException
    java.io.IOException <|-- UnknownHostException 
}
```
#### 样例2:
```plantuml!
title 视频播放功能的业务关联图
package 全屏播放业务功能 <<Cloud>> #DeepSkyBlue{
    package 视频播放业务层 <<Frame>> #DodgerBlue{
        package 播放内核 <<Frame>> #Blue{
            interface ADVideoPlayerListener{
                  +{static}{abstract}void onBufferUpdate(int position);//回调，视频缓冲
                  +{static}{abstract}void onAdVideoLoadSucess();//回调，视频加载成功
                  +{static}{abstract}void onAdVideoPlayComplete();//回调，视频正常播放完成
                  +{static}{abstract}void onAdVideoLoadFailed();//回调，视频不能正常播放
                  +{static}{abstract}void onClickVideo();//回调，点击视频区域的
                  +{static}{abstract}void onClickFullScreenBtn();//回调，点击全屏按钮
                  +{static}{abstract}void onClickBackBtn();
                  +{static}{abstract}void onClickPlay();
            }
            class CustomVideoView {
                .. 视频播放器，只有暂停、播放和停止的基本功能 ..
                ..其他功能事件实现在业务层..
                interface ADVideoPlayerListener
                -MediaPlayer mediaPlayer
                -ADVideoPlayerListener listener
                -ScreenEventReceiver mScreenEventReceiver
                ...
            }

            CustomVideoView o-down- ADVideoPlayerListener

        }

        interface AdSDKShellListener {
           +{static}{abstract}ViewGroup getAdParent();
           +{static}{abstract}void onAdVideoLoadSuccess();
           +{static}{abstract}void onAdVideoLoadFailed();
           +{static}{abstract}void onAdVideoPlayComplete();
           +{static}{abstract}void onClickVideo(String url);
        }

        class VideoAdShell{
            ..视频播放器的业务封装层..
            interface ADVideoPlayerListener
            +VideoAdShell(AdValue, AdSDKShellListener, ADFrameImageLoadListener)
            -MediaPlayer mediaPlayer
            -ADVideoPlayerListener listener
            -ScreenEventReceiver mScreenEventReceiver
            ...
        }

        AdSDKShellListener -down-o VideoAdShell
        VideoAdShell -right-|> ADVideoPlayerListener
        VideoAdShell .right.> CustomVideoView

    }



    class VideoFullDialog{
        ..全屏视频播放..
        - AdSDKShellListener mShellListener
        -MediaPlayer mediaPlayer
        -ADVideoPlayerListener listener
        -ScreenEventReceiver mScreenEventReceiver
        +VideoFullDialog(Context, CustomVideoView, AdValue, int)
        +void setShellListener(AdSDKShellListener)//承接业务层回调
        +void setListener(FullToSmallListener)//切换小屏回调
        ...
    }

    VideoFullDialog -up-|> ADVideoPlayerListener
}
VideoAdShell<..>VideoFullDialog


package 客户端 <<Frame>> #LightGray{
package SDK入口 <<Frame>> #gray{
    interface AdSDKShellListener{
        +{static}{abstract}void onAdSuccess();//视频资源加载成功
        +{static}{abstract}void onAdFailed();//无法播放
        +{static}{abstract}void onClickVideo(String url);//点击播放窗口回调
    }

    class VideoAdFactory{
        ..使用构建者模式构建实例,方便用户使用..
        ..使用外观模式封装视频播放SDK,方便使用API..
        -final ViewGroup mParentView;
        -final AdValue mInstance;
        +void setAdResultListener(AdFactoryInterface)
        +void updateAdInScrollView//根据用户滑动页面来判断视频自动播放
        +{static} class Builder
    }

    VideoAdFactory -up-|> AdSDKShellListener
}
    class Client<<用户层>> {
            ..生成VideoAdFactory实例,
            调用视频的API..
    }

    Client -left-> VideoAdFactory
}

```
#### 样例3:
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

Pantuml参考链接:
https://blog.csdn.net/junhuahouse/article/details/80767632
https://www.cnblogs.com/Jeson2016/p/6837017.html
https://design-patterns.readthedocs.io/zh_CN/latest/read_uml.html#id3
http://plantuml.com/zh/class-diagram

# Memeraid
未完待续...