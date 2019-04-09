# Dispatchers

## 简介
Akka MessageDispatcher是使Akka Actors工作的必需，可以说，它是机器的引擎。所有MessageDispatcher的实现也是一个ExecutionContext，这意味着它们可以用来执行任意代码，例如Future。

## 默认的dispatcher
每个Actor System都有一个默认的dispatcher，用于在没有为Actor配置任何其他配置的情况下。可以配置默认的dispatcher，

如果创建一个ActorSystem时传入一个ExecutionContext，则此ExecutionContext将用作该ActorSystem中的所有dispatcher的默认执行程序。如果没有提供ExecutionContext，它将回退到在`akka.actor.default-dispatcher.default-executor.fallback`。默认情况下，这是一个“fork-join-executor”，它在大多数情况下提供了出色的性能。

## 查看dispatcher

Dispatchers实现了ExecutionContext接口，因此可以用Future来运行。
```java
// this is scala.concurrent.ExecutionContext
// for use with Futures, Scheduler, etc.
final ExecutionContext ex = system.dispatchers().lookup("my-dispatcher");
```

## 给actor设置一个dispater
如果想给Actor一个不同于默认dispatcher的dispatcher，需要做两件事，其中第一件是配置调度器:
```conf
my-dispatcher {
  # Dispatcher is the name of the event-based dispatcher
  type = Dispatcher
  # What kind of ExecutionService to use
  executor = "fork-join-executor"
  # Configuration for the fork join pool
  fork-join-executor {
    # Min number of threads to cap factor-based parallelism number to
    parallelism-min = 2
    # Parallelism (threads) ... ceil(available processors * factor)
    parallelism-factor = 2.0
    # Max number of threads to cap factor-based parallelism number to
    parallelism-max = 10
  }
  # Throughput defines the maximum number of messages to be
  # processed per actor before the thread jumps to the next actor.
  # Set to 1 for as fair as possible.
  throughput = 100
}
```
> 注意，parallelism-max并没有设置ForkJoinPool分配的线程总数的上限。它是一个专门讨论池保持运行的热线程数量的设置，以减少处理新传入任务的延迟。

另一个使用“线程池执行器”的例子是:
```conf
blocking-io-dispatcher {
  type = Dispatcher
  executor = "thread-pool-executor"
  thread-pool-executor {
    fixed-pool-size = 32
  }
  throughput = 1
}
```
> 线程池执行器dispatcher由一个`java.util.concurrent.ThreadPoolExecutor`实现。

然后像往常一样创建actor并在部署配置中定义dispatcher。
```java
ActorRef myActor = system.actorOf(Props.create(MyActor.class), "myactor");
```

```conf
akka.actor.deployment {
  /myactor {
    dispatcher = my-dispatcher
  }
}
```

部署配置的另一种选择是在代码中定义dispatcher。如果在配置文件中定义dispatcher，则将使用配置文件中的值而不是编程提供的参数。

```java
ActorRef myActor =
    system.actorOf(Props.create(MyActor.class).withDispatcher("my-dispatcher"), "myactor3");
```
> 在withDispatcher中指定的dispatcher和部署配置中的dispatcher属性实际上是配置的路径。在这个例子中，它是一个顶级部分，但你可以把它作为子部分，在这里你可以使用“.”来表示子部分，像这样"foo.bar.my-dispatcher"

## dispatcher的类型
有三种不同类型的消息dispather:

1. Dispatcher

这是一个基于事件的dispatcher，它将一组actor绑定到线程池。如果没有指定调度器，则使用默认调度器。
    - 可共享：无限制
    - mailbox：为每个actor创建一个
    - use cases：默认的分配器，bulkheading
    - drive by: `java.util.concurrent.ExecutorService`。在`akka.dispatcher.ExecutorServiceConfigurator`中指定使用“executor”、“fork-join-executor”、“thread-pool-executor”或FQCN。

2. PinnedDispatcher

这个dispatcher为每个使用它的actor提供一个唯一的线程;也就是说，每个actor都有自己的线程池，其中只有一个线程。
    - 可共享：不
    - mailboxes：为每个actor创建一个
    - use cases: bulkheading
    - driven by:任意`akka.dispatch.ThreadPoolExecutorConfigurator`。默认“thread-pool-executor”。

3. CallingThreadDispatcher

这个dispatcher只在当前线程上运行调用。这个dispatcher不创建任何新线程，但是可以从不同的线程并发地为同一个actor使用它。
    - 可共享：无限制
    - mailboxes：为每个acto每一个线程r创建一个
    - use cases: testing
    - driven by:调用线程

### 更多配置示例
配置一个固定线程池大小的调度程序，例如，actor执行阻塞IO:
```conf
blocking-io-dispatcher {
  type = Dispatcher
  executor = "thread-pool-executor"
  thread-pool-executor {
    fixed-pool-size = 32
  }
  throughput = 1
}
```
然后使用
```java
ActorRef myActor =
    system.actorOf(Props.create(MyActor.class).withDispatcher("blocking-io-dispatcher"));
```
另一个使用基于内核数量的线程池的示例(例如，对于CPU绑定任务)
```conf
my-thread-pool-dispatcher {
  # Dispatcher is the name of the event-based dispatcher
  type = Dispatcher
  # What kind of ExecutionService to use
  executor = "thread-pool-executor"
  # Configuration for the thread pool
  thread-pool-executor {
    # minimum number of threads to cap factor-based core number to
    core-pool-size-min = 2
    # No of core threads ... ceil(available processors * factor)
    core-pool-size-factor = 2.0
    # maximum number of threads to cap factor-based number to
    core-pool-size-max = 10
  }
  # Throughput defines the maximum number of messages to be
  # processed per actor before the thread jumps to the next actor.
  # Set to 1 for as fair as possible.
  throughput = 100
}
```
使用affinity pool,这类dispatcher可以增加吞吐量。affinity pool尽最大努力保证actor总是运行在同一个线程中。其目的是减少CPU缓存丢失，因而会导致吞吐量显著提高。
```conf
affinity-pool-dispatcher {
  # Dispatcher is the name of the event-based dispatcher
  type = Dispatcher
  # What kind of ExecutionService to use
  executor = "affinity-pool-executor"
  # Configuration for the thread pool
  affinity-pool-executor {
    # Min number of threads to cap factor-based parallelism number to
    parallelism-min = 8
    # Parallelism (threads) ... ceil(available processors * factor)
    parallelism-factor = 1
    # Max number of threads to cap factor-based parallelism number to
    parallelism-max = 16
  }
  # Throughput defines the maximum number of messages to be
  # processed per actor before the thread jumps to the next actor.
  # Set to 1 for as fair as possible.
  throughput = 100
}
```
配置PinnedDispatcher:
```conf
my-pinned-dispatcher {
  executor = "thread-pool-executor"
  type = PinnedDispatcher
}
```
然后使用：
```java
ActorRef myActor =
    system.actorOf(Props.create(MyActor.class).withDispatcher("my-pinned-dispatcher"));
```

## 小心管理阻塞
在某些情况下，不可避免地要执行阻塞操作，即让线程休眠一段不确定的时间，等待外部事件发生。例如遗留的RDBMS驱动程序或消息传递api，其基本原因通常是(网络)I/O导致。
```java
class BlockingActor extends AbstractActor {

  @Override
  public Receive createReceive() {
    return receiveBuilder()
        .match(
            Integer.class,
            i -> {
              Thread.sleep(5000); // block for 5 seconds, representing blocking I/O, etc
              System.out.println("Blocking operation finished: " + i);
            })
        .build();
  }
}
```
面对这种情况，您可能会试图将阻塞调用封装在Future的代码中，并转而使用它，但是这种策略太简单了:当应用程序在增加的负载下运行时，您很可能会发现瓶颈或内存或线程耗尽。
```java
class BlockingFutureActor extends AbstractActor {
  ExecutionContext ec = getContext().getDispatcher();

  @Override
  public Receive createReceive() {
    return receiveBuilder()
        .match(
            Integer.class,
            i -> {
              System.out.println("Calling blocking Future: " + i);
              Future<Integer> f =
                  Futures.future(
                      () -> {
                        Thread.sleep(5000);
                        System.out.println("Blocking future finished: " + i);
                        return i;
                      },
                      ec);
            })
        .build();
  }
}
```

### 问题：阻塞在默认dispatcher上
关键问题在这
```java
ExecutionContext ec = getContext().getDispatcher();
```
使用`getContext().getDispatcher()`会带来问题。因为这个dispatcher默认用于所有其他actor处理。

如果所有可用的线程都被阻塞，那么同一dispatcher上的所有参与者都将因线程而饿死，无法处理传入的消息。

用上面的BlockingFutureActor和下面的PrintActor设置一个应用程序。
```java
class PrintActor extends AbstractActor {
  @Override
  public Receive createReceive() {
    return receiveBuilder()
        .match(
            Integer.class,
            i -> {
              System.out.println("PrintActor: " + i);
            })
        .build();
  }
}
```
```java
ActorRef actor1 = system.actorOf(Props.create(BlockingFutureActor.class));
ActorRef actor2 = system.actorOf(Props.create(PrintActor.class));

for (int i = 0; i < 100; i++) {
  actor1.tell(i, ActorRef.noSender());
  actor2.tell(i, ActorRef.noSender());
}
```
发送100个信息到BlockingFutureActor和PrintActor，并且有大量的`akka.actor.default-dispatcher`线程来处理请求。当你运行上面的代码时，你很可能会看到整个应用程序像这样卡住:
```
>　PrintActor: 44
>　PrintActor: 45
```
PrintActor被认为是非阻塞的，但是它不能继续处理剩余的消息，因为所有的线程都被其他阻塞actor占用和阻塞——从而导致线程饥饿。


在上面的示例中，我们通过向阻塞actor发送数百条消息来加载代码，这将导致阻塞缺省调度程序的线程。然后，Akka中基于fork join池的dispatcher尝试通过向池中添加更多的线程来补偿这种阻塞(default-akka.actor.default-dispatcher 18,19,20，…)。但是，如果这些操作也会立即被阻塞，并且最终阻塞操作将控制整个调度程序，那么这就没有什么帮助了。

本质上，是Thread.sleep操作已经控制了所有线程，并导致在默认调度器上执行的任何操作都都没有资源(包括没有为其配置显式调度器的任意actor)。

### 解决方案：对专用dispatcher施行阻塞操作
隔离阻塞行为、使其不影响系统其余部分的最有效方法之一是为所有这些阻塞操作准备并使用专用的dispatcher。这种技术通常被称为“bulk-heading”或简单的“isolating blocking”。

在application.conf，专用于阻塞行为的dispatcher应配置如下:
```conf
my-blocking-dispatcher {
  type = Dispatcher
  executor = "thread-pool-executor"
  thread-pool-executor {
    fixed-pool-size = 16
  }
  throughput = 1
}
```
基于thread-pool-executor的dispatcher允许我们设置它将承载的线程数量的限制，通过这种方式，我们可以严格控制系统中最多有多少阻塞线程。

确切的大小应该根据您期望在这个调度程序上运行的工作负载以及您正在运行应用程序的机器的内核数量进行微调。通常，内核数量附件的一个小数字是一个很好的默认开始。

当必须进行阻塞时，使用上面配置的dispatcher，而不是默认的dispatcher:
```java
class SeparateDispatcherFutureActor extends AbstractActor {
  ExecutionContext ec = getContext().getSystem().dispatchers().lookup("my-blocking-dispatcher");

  @Override
  public Receive createReceive() {
    return receiveBuilder()
        .match(
            Integer.class,
            i -> {
              System.out.println("Calling blocking Future on separate dispatcher: " + i);
              Future<Integer> f =
                  Futures.future(
                      () -> {
                        Thread.sleep(5000);
                        System.out.println("Blocking future finished: " + i);
                        return i;
                      },
                      ec);
            })
        .build();
  }
}
```
当阻塞操作运行在my-blocking-dispatcher上时，它使用一定线程数量(直到配置的限制)来处理这些操作。在这种情况下，休眠被很好地隔离到这个dispatcher中，缺省值不受影响，允许应用程序的其余部分继续运行，就像没有发生什么糟糕的事情一样。在一段时间空闲之后，由这个调度程序启动的线程将被关闭。

在本例中，其他actor的吞吐量没有受到影响——它们仍然在缺省分派器上提供服务。

这是处理交互性应用程序中的任何类型阻塞的推荐方法。

### 阻塞操作的可用解决方案
“阻塞问题”的非详尽的适当解决办法清单:
- 在一个由rutor管理的actor(或一组actor)中执行阻塞调用，确保配置一个线程池，该线程池要么专门用于此目的，要么足够大。
- 使用Future执行阻塞调用，确保在任何时间点上对此类调用的数量有一个上限(提交数量不限的此类任务将耗尽内存或线程限制)。
- 使用Future执行阻塞调用，为线程池提供一个线程数量上限，该上限适用于应用程序所运行的硬件。
- 使用一个线程管理一组阻塞资源(例如NIO选择器驱动多个通道)，在actor消息发生时分派事件。

第一种可能性特别适合于本质上是单线程的资源，比如数据库句柄，通常一次只能执行一个未完成的查询，并使用内部同步来确保这一点。一种常见的模式是为N个actor创建一个rutor，每个actor封装一个DB连接并处理发送到rutor的查询。然后，必须对数字N进行调优，以获得最大吞吐量，这取决于DBMS部署在什么硬件上。



