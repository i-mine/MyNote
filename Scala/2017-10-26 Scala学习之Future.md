---
title: 2017-10-26 Scala学习之Future
tags: scala,Future
grammar_cjkRuby: true
---
http://blog.csdn.net/thomassuns/article/details/38368583

作为一个上进且充满激情的Scala开发者，你会愿意知道Scala在并行处理方面的成就，或者你正是被这吸引到Scala世界来的呢。Scala让你可以更容易的写出健壮的并行程序而无需像其它语言一样和低阶API打交道。
Scala在这方面的成就来自于两个方面，Future和Actor。前者正是本篇要涉及的内容，我会介绍future的优势以及如何以函数式的方式来使用它。
请确保你的Scala环境为2.9.3以上，以便亲自测试后面的代码。
为什么串行代码会比较糟糕

假设你要弄杯卡普奇诺喝喝。你可以顺序依次执行下述操作：
* 磨咖啡豆
* 烧热水
* 用热水蒸煮磨好的咖啡
* 给牛奶打泡
* 将蒸馏好的咖啡和打泡牛奶混合

翻译成Scala代码，会是下面的样子：

```scala
import scala.util.Try  
// Some type aliases, just for getting more meaningful method signatures: 
//咖啡豆
type CoffeeBeans = String
//磨好的咖啡
type GroundCoffee = String
//水
case class Water(temperature: Int)  
//牛奶
type Milk = String  
//打泡牛奶
type FrothedMilk = String  
//浓咖啡
type Espresso = String  
//卡布奇诺
type Cappuccino = String  
// dummy implementations of the individual steps:  
// 单个步骤的虚拟实现
//磨豆子
def grind(beans: CoffeeBeans): GroundCoffee = s"ground coffee of $beans"  
//加热
def heatWater(water: Water): Water = water.copy(temperature = 85)  
//牛奶打泡
def frothMilk(milk: Milk): FrothedMilk = s"frothed $milk"  
//冲泡
def brew(coffee: GroundCoffee, heatedWater: Water): Espresso = "espresso"  
//混合
def combine(espresso: Espresso, frothedMilk: FrothedMilk): Cappuccino = "cappuccino"  
// some exceptions for things that might go wrong in the individual steps  
// (we'll need some of them later, use the others when experimenting  
// with the code):  
case class GrindingException(msg: String) extends Exception(msg)  
case class FrothingException(msg: String) extends Exception(msg)  
case class WaterBoilingException(msg: String) extends Exception(msg)  
case class BrewingException(msg: String) extends Exception(msg)  
// going through these steps sequentially:  
def prepareCappuccino(): Try[Cappuccino] = for {  
  ground <- Try(grind("arabica beans"))  
  water <- Try(heatWater(Water(25)))  
  espresso <- Try(brew(ground, water))  
 //泡沫 
  foam <- Try(frothMilk("milk"))  
} yield combine(espresso, foam)
```


这样做有几个好处：你有了一个按步执行的指南。进一步讲，你不会被搞混，因为没有了上下文切换。
另一方面，这样一步一步的煮咖啡表示当进行周期长的步骤时你的大脑和身体处在等待状态，在等磨咖啡时，你实际上是被阻塞的，只有在磨好咖啡时，你才开始烧水及后续操作。
这显然是对宝贵资源的浪费。你当然更愿意同时开始一些步骤让它们同时执行。当你烧开水和研磨咖啡弄完时，你就可以开始蒸馏工作，同时去给牛奶打泡。
在写代码时也一样。web服务器用有限的线程处理用户请求并生成返回结果。你不会想因为需要等待数据库查询或调用另一个HTTP服务时而阻塞这些宝贵的线程，因而，当一个处理中的请求在等数据库结果时，为那个请求提供服务的web服务器线程可以接受新的用户请求而不是在那发呆。
“这听上去像是回调机制，我来给你的回调函数再提供一个回调函数就是了!”
当然，你已经非常熟悉这样的架构了- Node.js已是这方面的高手了。 Node.js和其它一些框架正是通过回调机制来实现的。不过很遗憾，这很容易导致回调函数内嵌套多层回调函数，最终导致你的代码很难看懂，也不方便debug。
Scala里的Future也支持回调机制，我很快就会讲到，不过Scala提供了更好的替代方案，所以你在Scala里反而不太会用回调。
“我知道Future，它们根本就没有用处！”
你也许已经熟悉其它一些Future的实现，尤其是Java下得Future实现。对于Java里的Future，你除了主动检查future是否完成或者阻塞在那里等它完成之外，没有其它用法，也就是说，它基本上没有什么用途，用起来绝对不会让你愉快。如果你以为Scala里future也是那样的，那就错了，我们现在就来看看。
Future的含义

Scala里的Future[T]来自scala.concurrent包，它是一个容器类型，代表一个代码最终会返回的T类型结果。不过，代码可能会出错或执行超时，所以当Future完成时，它有可能完全没有被成功执行，这时它会代表一个异常。
Future是一个一次写容器：当一个future完成时，它就不可变了。而且，Future类型仅是提供了一个接口用来读取最终结果。给最终结果赋值的工作是通过Promise来完成的，因而，相关的API设计是有非常明确的功能划分的。在本篇里，我们会专注于前者，即Future，下一篇来讲讲Promise类型。
使用Futures

Scala的Future有几种用法，我将通过用Future来改造前面的煮咖啡的代码来一一解释。首先我们要重写那些可以并行执行的函数，让它们立刻返回Future而不是阻塞时的执行：
```scala
import scala.concurrent.future  
import scala.concurrent.Future  
import scala.concurrent.ExecutionContext.Implicits.global  
import scala.concurrent.duration._  
import scala.util.Random  
  
def grind(beans: CoffeeBeans): Future[GroundCoffee] = Future {  
  println("start grinding...")  
  Thread.sleep(Random.nextInt(2000))  
  if (beans == "baked beans") throw GrindingException("are you joking?")  
  println("finished grinding...")  
  s"ground coffee of $beans"  
}  
  
def heatWater(water: Water): Future[Water] = Future {  
  println("heating the water now")  
  Thread.sleep(Random.nextInt(2000))  
  println("hot, it's hot!")  
  water.copy(temperature = 85)  
}  
  
def frothMilk(milk: Milk): Future[FrothedMilk] = Future {  
  println("milk frothing system engaged!")  
  Thread.sleep(Random.nextInt(2000))  
  println("shutting down milk frothing system")  
  s"frothed $milk"  
}  
  
def brew(coffee: GroundCoffee, heatedWater: Water): Future[Espresso] = Future {  
  println("happy brewing :)")  
  Thread.sleep(Random.nextInt(2000))  
  println("it's brewed!")  
  "espresso"  
}  
 ```
有几个地方需要先解释一下。
首先，Future的联合对象有一个apply方法，这方法需要两个参数：
 
```scala
object Future {  
  def apply[T](body: => T)(implicit execctx: ExecutionContext): Future[T]  
}  
```
 
异步执行的代码以by-name方式赋给body参数。第二个参数是一个隐式参数，意味着如果作用范围内有有匹配的隐式常量存在，我们就无需提供此参数，我们通过导入全局执行环境引入了一个隐式常量。
ExecutionContext是一个可以运行Future的环境，你可以把它当做是类似线程池的东西。因为ExecutionContext是一个隐式常量，所以我们只需要为前个单参数列表赋值。单参数列表可以不用括号而用大括号来包含，大家通常都会这样来调用future方法，这让它看上去更像是在使用语言的特性而不是一般的函数调用。ExecutionContext是所有FutureAPI的隐式常量。
进一步，我们在这个简单的例子里不用做任何的计算，所以用随机休眠来模拟实际计算花费的时间。我们在“计算”前后分别打印出信息，以清楚的表达代码中的不确定和并行特性。
Future返回的计算结果将会在Future被创建（通过ExecutionContext管理的线程）后的一个不确定时间出现。

回调

有时候，在简单场景下，用回调可以更方便。Future的回调函数是偏函数。你可以把一个回调函数传递给onSuccess方法，它仅在Future被成功执行时才调用，计算结果将会传递给回调函数：
```scala 
grind("arabica beans").onSuccess { case ground =>  
  println("okay, got my ground coffee")  r
}  
```
同样的，你可以用onFailure方法来注册一个失败情况的回调函数。回调函数需要接收Throwable参数，只有当Future不能被成功执行时才会调用回调函数。
通常，更好的做法是同时为成功和失败定义两个回调函数，回调函数接收的参数是Try类型：
```scala
import scala.util.{Success, Failure}  
grind("baked beans").onComplete {  
  case Success(ground) => println(s"got my $ground")  
  case Failure(ex) => println("This grinder needs a replacement, seriously!")  
}  
```
上面例子中的graind就有可能抛出异常，这就会导致Future以失败的状态完成。

组合Futures

当你需要嵌套回调函数时就会比较痛苦了。幸运的是，你不需要那样做。Scala的future真正强大的地方是它们可以被组合。
如果你看过本系列的前几篇，你可能已经注意到所有我们讨论过的容器类型都可以让你对它们做map和flatmap操作，或者在for语句中使用它们，我前面提到Future也是一个容器类型，因而Future也可以让你那样做应该没有什么惊奇的吧。
真正的问题是：对一些实际尚未完成的计算执行这些操作意味着什么？

对Future进行Map

你是否总是想成为一个时间旅行者去未来看看？一个Scala开发者就可以！假设你在烧水的过程中想要检查水温是否已经合适了，你可以通过将Future[Water]map成Future[Boolean]来做到：
```scala 
val temperatureOkay: Future[Boolean] = heatWater(Water(25)).map { water =>  
  println("we're in the future!")  
  (80 to 85).contains(water.temperature)  
}  
```
 
赋给temperatureOkay的Future[Boolean]最终将会包含一个成功计算的boolean值。 试着修改一下heatWater的实现，让它抛出异常（比如你的水壶爆掉了啥的），再来观察你会发现 we're in the future 永远不会输出。
当你写传递给map的函数时，你实际上处在未来或者可能处在未来。一旦Future[Water]实例被成功执行完成时，map函数就会马上被执行，这个事件发生的时间可能不是当下。如果Future[Water]执行失败，你传递给map的函数将不会被调用。相反，map将会返回一个包含着Failure的Future[Boolean]。
确保未来是平的

如果一个Future的计算依赖于另一个Future的结果，你会需要flatMap来防止Future的多重嵌套。
例如，假设测量水温需要些时间，所以我们也要让判断水温是否合适的操作异步化。你有一个函数传入Water的实例，并返回一个Future[Boolean]:
```scala 
def temperatureOkay(water: Water): Future[Boolean] = Future {  
  (80 to 85).contains(water.temperature)  
}  
```
用flatMap而不是map以便得到一个Future[Boolean]而不是Future[Future[Boolean]]：
```scala
val nestedFuture: Future[Future[Boolean]] = heatWater(Water(25)).map {  
  water => temperatureOkay(water)  
}  
val flatFuture: Future[Boolean] = heatWater(Water(25)).flatMap {  
  water => temperatureOkay(water)  
}  
```
同样的，map方法仅在Future[Water]实例成功完成后才被调用。
For语句

除了调用flatMap，你还可以用for语句来达到同样目的，但是代码可读性更好。上面的例子就可以这样来写：
```scala
val acceptable:Future[Boolean]=for{  
heatedWater <- heatWater(Water(25))  
okay <- temperatureOkay(heatedWater)  
}yield okay
```
当你有多个需要同时进行的计算时，你需要小心了，因为你已经在for语句之外生成了一个新的Future实例。
```scala
def prepareCappuccinoSequentially(): Future[Cappuccino] = {  
  for {  
    ground <- grind("arabica beans")  
    water <- heatWater(Water(20))  
    foam <- frothMilk("milk")  
    espresso <- brew(ground, water)  
  } yield combine(espresso, foam)  
}  
```
这看上去挺好，不过因为for语句不过是flatMap的另一种表达方式，所以flatMap的调用机制也同样适用，也就是说heatWater生成Future[Water]的语句仅当Future[GroundCoffee]被成功完成时才会被实例化。你可以通过观察函数输出来验证。
因而，你应该在for语句前实例化所有的独立的Future：
```scala
def prepareCappuccino(): Future[Cappuccino] = {  
  val groundCoffee = grind("arabica beans")  
  val heatedWater = heatWater(Water(20))  
  val frothedMilk = frothMilk("milk")  
  for {  
    ground <- groundCoffee  
    water <- heatedWater  
    foam <- frothedMilk  
    espresso <- brew(ground, water)  
  } yield combine(espresso, foam)  
}  
```
现在在for语句开始前我们实例化了三个Future，它们立刻开始同时执行。如果你观察输出，你会看到无序的输出。唯一可以确定的就是“happy brewing“会在最后输出，因为所调用的方法需要来自另外两个Future的返回值，只有另外两个Future被成功完成时才有这两个值。

==注意 #800f00==：整个流程最后的结果是在Future[Cappuccino]中的，因此要对该Future进行阻塞，等待前面结果的的产生。
```scala
val prepare = prepareCappuccino()
    prepare.onComplete{
      case Success(prepare) =>println(s"oh!,great $prepare")
      case Failure(ex) =>throw  new RuntimeException("no coffee")
    }
    val result = Await.result(prepare,Duration.Inf)
```

失败投影

你应该已经注意到，Future[T]是偏向成功的，这让你可以使用`map`, `flatMap`,`filter`等，`都是基于它会被成功完成的前提的`。有时候，你想以优雅的函数的方式来处理未来的失败。你可以呼叫Future[T]的failed方法来得到它的失败投影，即Future[Throwable]。现在你可以对Future[Throwable]执行例如map操作，map函数只有当Future[T]失败完成时才被调用。
