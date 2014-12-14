让并发和容错更容易：Akka示例教程

BY DIAGO CASTORINA

原文链接：http://www.toptal.com/scala/concurrency-and-fault-tolerance-made-easy-an-intro-to-akka

###挑战

并发程序很难写。程序员必须要处理线程、锁、竞态条件等等，这个过程很容易出错，而且会导致程序代码难以阅读、测试和维护。

所以很多人更倾向于不使用多线程。取而代之的是，他们使用只有单线程的进程，依靠外部的服务（如数据库，队列等）来处理所需的并发或异步操作。虽然这种方法在有些情况下是可行的，但是还有很多其他情况不能奏效。很多实时系统——例如交易或银行业务应用，或实时游戏——等待一个单线程进程完成就太奢侈了（他们需要立即应答！）。其他的一些系统对于计算或资源的要求非常高，如果在程序中不引入并行机制就需要耗时很久（有些情况下可以达到数小时或数天）。

常用的一种单线程方法（例如，在Node.js里广泛使用）是使用基于事件的、非阻塞模式（event-based, non-blocking paradigm，其中paragidigm也有译作成例）。虽然这种方法通过避免上下文切换、锁和阻塞，的确能帮助提高性能，但还是不能解决并发使用多个处理器（需要启动和协调多个独立的处理器）的问题。

那么，这是不是意味着为了构建一个并发程序，除了深入到线程、锁和竞态条件之外你没有别的选择呢？

感谢Akka框架，它为我们提供了一种选择。本教程介绍了Akka的示例并探究了它如何帮助并简化分布式并发应用的实现。

###Akka框架是什么

_这篇文章介绍了Akka并探究了它如何帮助并简化分布式并发应用的实现。_

Akka是JVM（Java虚拟机，下同）平台上构建高并发、分布式和容错应用的工具包和运行时。Akka用Scala语言写成，同时提供了Scala和Java的开发接口。

Akka处理并发的方法基于Actor（没有惯用译法，文中使用原词）模型。在基于Actor的系统里，所有的事物都是actor，就好像在面向对象设计里面所有的事物都是对象一样。但是有一个重要区别——特别是和我们的讨论相关——那就是Actor模型是作为一个并发模型设计和架构的，而面向对象模式则不是。更具体来说，在Scala的actor系统里，actor互相交互并共享信息但并不对交互顺序作出预设。Actor之间共享信息和发起任务的机制是消息传递。

_创建和调度线程、接收和分发消息以及处理竞态条件和同步的所有复杂性，都委托给框架，框架的处理对应用来说是透明的。_

Akka在多个actor和下面的系统之间创建了一层（layer），这样一来，actor只需要处理消息就可以了。创建和调度线程、接收和分发消息以及处理竞态条件和同步的所有复杂性，都委托给框架，框架的处理对应用来说是透明的。

Actor严格遵守响应式声明。响应式应用的目标是通过满足以下一个或多个条件来替换传统的多线程应用：
 - 事件驱动。使用Actor，代码可以异步处理请求并且用独占的方式执行非阻塞操作。
 - 可伸缩性。在Akka里，不修改代码而增加节点是可能的，感谢消息传递和本地透明性（location transparency）。
 - 高弹性。任何应用都会碰到错误并在某个时间点失败。Akka的“监管”（容错性）策略为实现自愈系统提供了方便。
 - 响应式。今天的高性能和快速响应应用需要对用户快速反馈，因此对于事件的响应需要非常及时。Akka的非阻塞、基于消息的策略可以帮助达成这个目标。

###Akka中的Actor是什么

Actor本质上就是接收消息并采取行动处理消息的对象。它从消息源中分离出来，只负责正确识别它接收到的消息类型，并响应地采取行动。

收到一条消息之后，一个actor可能会采取以下一个或多个行动：
 - 执行一些本身的操作（例如进行计算、持久化数据、调用外部的Web服务等等）
 - 把消息或衍生消息转发给另一个actor
 - 实例化一个新的actor并把消息转发给它

或者，如果这个actor认为合适的话，可能会完全忽略这条消息（也就是说，它可能选择不回应）。

为了实现一个actor，需要继承akka.actor.Actor这个trait（一般译为“特征”，译法有一定争议，文中保留原词）并实现receive方法。Actor的receive方法当一个消息发送给它时被（Akka）调用。典型的实现包括使用模式匹配（pattern matching）来识别消息类型并相应作出响应，参见下面的Akka示例：
```
import akka.actor.Actor
import akka.actor.Props
import akka.event.Logging

class MyActor extends Actor {
  def receive = {
    case value: String => doSomething(value)
    case _ => println("received unknown message")
  }
}
```

模式匹配是一种相对优雅的处理消息的技术，相比基于回调的实现，更倾向于产生“更整洁”和更容易浏览的代码。例如，考虑一个简化版的HTTP请求/响应实现。

首先，我们使用JavaScript中基于回调的方式实现：
```
route(url, function(request){
  var query = buildQuery(request);
  dbCall(query, function(dbResponse){
    var wsRequest = buildWebServiceRequest(dbResponse);
    wsCall(wsRequest, function(wsResponse) {
      sendReply(wsResponse);
    });
  });
});
```

现在，我们把它和基于模式匹配实现做个比较：
```
msg match {
  case HttpRequest(request) => {
    val query = buildQuery(request)
    dbCall(query)
  }
  case DbResponse(dbResponse) => {
    var wsRequest = buildWebServiceRequest(dbResponse);
    wsCall(dbResponse)
  }
  case WsResponse(wsResponse) => sendReply(wsResponse)
}
```

虽然基于回调的JavaScript代码更紧凑，但确实更加难以阅读和浏览。相比而言，基于模式匹配的代码对于需要考虑哪些情况及每种情况都是怎么处理的则更加明显。

###Actor系统

把一个复杂的问题不断分解成更小规模的子问题通常是一种可靠的解决问题的技术。这个方法对于计算机科学特别有益（和单一职责原则一致），因为这样容易产生整洁的、模块化的代码，冗余较少甚至没有，而且维护起来相对容易。

在基于actor的设计里，使用这种技术有助于把actor的逻辑组织变成一个层级结构，也就是所谓的Actor系统。Actor系统提供了一个基础构架，通过这个系统actor之间可以进行交互。

在Akka里面，和actor通信的唯一方式就是通过ActorRef。ActorRef代表actor的一个引用，可以阻止其他对象直接访问或操作这个actor的内部信息和状态。消息可以通过一个ActorRef通过下面语法协议中的一种发送到一个actor：
 -`!`("tell") —— 发送消息并立即返回
 -`?`("ask") —— 发送消息并返回一个Future对象，代表一个可能的应答

每个actor都有一个收件箱，用来接收发送过来的消息。收件箱有多种实现方式可以选择，缺省的实现是先进先出（FIFO）队列。

在处理多条消息时，一个actor包含多个实例变量来保持状态。Akka确保actor的每个实例都运行在自己的轻量级线程里，并保证每次只处理一条消息。这样一来，开发者不必担心同步或竞态条件，而每个actor的状态都可以被可靠地保持。

Akka的Actor API中提供了每个actor执行任务所需要的有用信息：
 - `sender`:当前处理消息的发送者的一个`ActorRef`引用
 - `context`：actor运行上下文相关的信息和方法（例如，包括实例化一个新的actor的方法`actorOf`）
 - `supervisionStrategy`：定义用来从错误中恢复的策略
 - `self`：actor本身的`ActorRef`引用

_Akka确保actor的每个实例都运行在自己的轻量级线程里，并保证每次只处理一条消息。这样一来，开发者不必担心同步或竞态条件，而每个actor的状态都可以被可靠地保持。_

为了把这些教程组织起来，让我们来考虑一个简单的例子：统计一个文本文件中单词数量。

为了演示我们Akka示例的目的，我们把这个问题分解成两个子任务；即，（1）一个统计每行单词数量的“孩子”任务和（2）一个汇总这些单行单词数量，得到文件里单词总数的“父亲”任务。

父actor会从文件中装载每一行，然后委托一个子actor来计算那一行的单词数量。当子actor完成之后，它会把结果用消息发回给父actor。父actor会收到单词数量（每一行）的消息并保存一个整个文件单词总数的计数器，这个计数器会在完成后返回给调用者。

_（注意以下提供的Akka教程的例子只是为了教学目的，所以没有关心所有的边界条件，性能优化等。同时，完整可编译版本的代码示例可以在这个gist中找到）_

让我们首先看一个子`StringCounterActor`的示例实现：
``
case class ProcessStringMsg(string: String)
case class StringProcessedMsg(words: Integer)

class StringCounterActor extends Actor {
  def receive = {
    case ProcessStringMsg(string) => {
      val wordsInLine = string.split(" ").length
      sender ! StringProcessedMsg(wordsInLine)
    }
    case _ => println("Error: message not recognized")
  }
}
```

这个actor有一个非常简单的任务：接收`ProcessStringMsg`消息（包含一行文本），计算这行文本中单词的数量，并把结果通过一个`StringProcessedMsg`消息返回给发送者。请注意我们已经实现了我们的类，使用`！`（“告知”）方法发出`StringProcessedMsg`消息（即发出消息并立即返回）。

好了，现在我们来关注父`WordCounterActor`类：
```
1.  case class StartProcessFileMsg()
2.
3.  class WordCounterActor(filename: String) extends Actor {
4.
5.    private var running = false
6.    private var totalLines = 0
7.    private var linesProcessed = 0
8.    private var result = 0
9.    private var fileSender: Option[ActorRef] = None
10.
11.   def receive = {
12.     case StartProcessFileMsg() => {
13.       if (running) {
14.         // println just used for example purposes;
15.         // Akka logger should be used instead
16.         println("Warning: duplicate start message received")
17.       } else {
18.         running = true
19.         fileSender = Some(sender) // save reference to process invoker
20.         import scala.io.Source._
21.         fromFile(filename).getLines.foreach { line =>
22.           context.actorOf(Props[StringCounterActor]) ! ProcessStringMsg(line)
23.           totalLines += 1
24.         }
25.       }
26.     }
27.     case StringProcessedMsg(words) => {
28.       result += words
29.       linesProcessed += 1
30.       if (linesProcessed == totalLines) {
31.         fileSender.map(_ ! result)  // provide result to process invoker
32.       }
33.     }
34.     case _ => println("message not recognized!")
35.   }
36. }
```

```
case class StartProcessFileMsg()

class WordCounterActor(filename: String) extends Actor {
  private var running = false
  private var totalLines = 0
  private var linesProcessed = 0
  private var result = 0
  private var fileSender: Option[ActorRef] = None

  def receive = {
    case StartProcessFileMsg() => {
      if (running) {
         // println just used for example purposes;
         // Akka logger should be used instead
         println("Warning: duplicate start message received")
      } else {
         running = true
         fileSender = Some(sender) // save reference to process invoker
         import scala.io.Source._
         fromFile(filename).getLines.foreach { line =>
           context.actorOf(Props[StringCounterActor]) ! ProcessStringMsg(line)
           totalLines += 1
         }
       }
    }
    case StringProcessedMsg(words) => {
      result += words
      linesProcessed += 1
      if (linesProcessed == totalLines) {
        fileSender.map(_ ! result)  // provide result to process invoker
      }
    }

    case _ => println("message not recognized!")
  }
}
```

这里面有很多细节，我们来逐个详细考察一下（注意讨论中所引用的行号基于以上代码示例）。

首先，请注意要处理的文件名被传给了`WordCounterActor`的构造方法（第3行）。这意味着这个actor只会用来处理一个单独的文件。这样通过避免重置状态变量（`running`，`totalLines`，`linesProcessed`和`result`）也简化了开发者的编码工作，因为这个实例只使用一次（也就是说处理一个单独的文件），然后就丢弃了。

接下来，观察到`WordCounterActor`处理了两种类型的消息：
 - `StartProcessFileMsg`（第12行）
  - 从最初启动`WordCounterActor`的外部actor接收到的消息
  - 收到这个消息之后，`WordCounterActor`首先检查它收到的是不是一个重复的请求
  - 如果这个请求是重复的，那么`WordCounterActor`生成一个警告，然后就不做别的了（第16行）
  - 如果这不是一个重复的请求：
   - `WordCounterActor`在`fileSender`实例变量（注意这是一个Option[ActorRef]而不是一个Option[Actor]）中保存发送者的一个引用。当处理最终的`StringProcessedMsg`（从一个`StringCounterActor`子类中接收，如下文所述）时，为了以后的访问和响应，这个`ActorRef`是必需的。
   - 然后`WordCounterActor`读取文件，当文件中每行都装载后，就会创建一个`StringCounterActor`孩子，需要处理的包含行文本的消息就会传递给它（第21-24行）。
 - `StringProcessedMsg`（第27行）
  - 当完成处理分配给它的行之后，从孩子`StringCounterActor`处接收到的消息
  - 收到此消息之后，`WordCounterActor`会把文件的行计数器增加，如果所有的行都处理完毕（也就是说，当`totalLines`和`linesProcessed`相等），它会把最终结果发给原来的`fileSender`（第28-31行）。

再次需要注意的是，在Akka里，actor之间通信的唯一机制就是消息传递。消息是actor之间唯一共享的东西，而且因为多个actor可能会并发访问同样的消息，所以为了避免竞态条件和不可预期的行为，消息的不可变性非常重要。

因此，因为case class默认是不可变的并且可以和模式匹配无缝集成，所以用case class的形式来传递消息是很常见的。（Scala中的Case class就是常规的类，但通过模式匹配提供了可以递归分解的机制）。

让我们通过运行整个应用的示例代码来结束这个例子。

```
object Sample extends App {

  import akka.util.Timeout
  import scala.concurrent.duration._
  import akka.pattern.ask
  import akka.dispatch.ExecutionContexts._

  implicit val ec = global

  override def main(args: Array[String]) {
    val system = ActorSystem("System")
    val actor = system.actorOf(Props(new WordCounterActor(args(0))))
    implicit val timeout = Timeout(25 seconds)
    val future = actor ? StartProcessFileMsg()
    future.map { result =>
      println("Total number of words " + result)
      system.shutdown
    }
  }
}
```

请注意这里的`?`方法是怎样发送一条消息的。用这种方法，调用者可以使用返回的Future对象，当完成之后可以打印出最终结果并最终通过停掉Actor系统退出程序。

###Akka的容错和监管者策略

在一个actor系统里，每个actor都是其子孙的监管者。如果一个actor处理消息时失败，它就会暂停自己及其子孙并发送一个消息给它的监管者，通常是以异常的形式。

_在Akka中，监管者策略是定义你的系统容错行为的主要而且直接的机制。_

在Akka中，一个监管者对于从子孙传递上来的异常的反应和处理方式称作监管者策略。监管者策略是定义你的系统容错行为的主要而且直接的机制。

当一条消息指示一个失败到达了一个监管者，它会采取下列行动之一：
 - __恢复孩子（及其子孙），保持内部状态。__ 当孩子的状态没有被错误破坏，还可以继续正常工作的时候，可以使用这种策略。
 - __重启孩子（及其子孙），清除内部状态。__ 这种策略应用的场景和第一种正好相反。如果孩子的状态已经被错误破坏，在它可以被用到Future之前有必须要重置其内部状态。
 - __永久地停掉孩子（及其子孙）。__ 这种策略可以用在下面的情境中：错误条件并不能被修正，但是并不影响后面执行的操作，这些操作可以在失败孩子不存在的情况完成。
 - __停掉自己并向上传播错误。__ 在监管者不知道如何处理错误就把错误传递给自己的监管者。

而且，一个Actor可以决定是否把行动应用在失败的子孙身上或者是也应用到它的兄弟。有两种预定义的策略：
 - `OneForOneStrategy`：只应用到指定行动到失败的孩子
 - `AllForOneStrategy`：把指定行动应用到所有子孙

下面是一个使用`OneForOneStrategy`的简单例子：
```
import akka.actor.OneForOneStrategy
import akka.actor.SupervisorStrategy._
import scala.concurrent.duration._

override val supervisorStrategy =
 OneForOneStrategy() {
   case _: ArithmeticException      => Resume
   case _: NullPointerException     => Restart
   case _: IllegalArgumentException => Stop
   case _: Exception                => Escalate
 }
```
