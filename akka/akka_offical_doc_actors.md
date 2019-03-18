
[toc]

# akka官方文档——actor模型


## Actors

### 1.创建一个actor
定义一个继承AbstractActor的类，并重写`createReceive()`方法。再通过`Prop`来创建一个actor
```
static class FirstActor extends AbstractActor {
  final ActorRef child = getContext().actorOf(Props.create(MyActor.class), "myChild");

  @Override
  public Receive createReceive() {
    return receiveBuilder().matchAny(x -> getSender().tell(x, getSelf())).build();
  }
}
```


### 2.Actor的API
一个继承`AbstractActor`的Actor定义了以下API:
- `createReceive()`:用于设置actor的“初始行为”。                  
如果当前actor行为与接收到的消息不匹配，则调用`unhandling`，默认情况下`unhandling`将在actor系统上发布`akka.actor.UnhandledMessage(message, sender, recipient)`事件流。(将配置项`akka.actor. Debug .unhandle`设置为`on`，可将它们转换为实际的调试消息)。
- `getSelf()`:获取当前actor的ActorRef引用
- `getSender()`:获取最后接收到的消息的发送方Actor的引用
- `supervisorStrategy()`:用于监视子角色的策略，用户可重写         
这种策略通常是宣布在Actor为了获得决策者中的Actor的内部状态功能:因为失败是作为发送给主管的消息进行通信的，并像处理其他消息一样进行处理失败(尽管在正常行为之外),Actor中的所有值和变量,作为`sender`的引用(将直接报告失败的孩子;如果最初的失败发生在一个遥远的后代中，它仍然会向其上一级报告)。
- `getContext()`:获取actor和当前消息的上下文信息，例如：
    - 创建子actor的工厂方法(actorOf)
    - actor所属的系统
    - actor parent的监管
    - 该actor监督的子actor
    - actor生命周期监测
    - `Become`/`Unbecome`中描述的hotswap行为栈

剩下的可见方法是用户可重写的的actor生命周期钩子函数，如下:
```
public void preStart() {}

public void preRestart(Throwable reason, Optional<Object> message) {
  for (ActorRef each : getContext().getChildren()) {
    getContext().unwatch(each);
    getContext().stop(each);
  }
  postStop();
}

public void postRestart(Throwable reason) {
  preStart();
}

public void postStop() {}
```
上面显示的实现是`AbstractActor`类提供的缺省值。

### Actor的生命周期

![image](https://doc.akka.io/docs/akka/current/images/actor_lifecycle.png)

actor system 中的路径表示一个“位置”，该位置可能由一个活着的actor占据。最初(除了系统初始化的actor之外)路径是空的。当调用`actorOf()`时，它将通过传递的props描述的actor分配给给定的路径。actor由路径和uid标识。

值得注意的是:
- restart
- stop

restart只交换由props定义的actor实例，因此uid保持不变。只要uid相同的，你可以继续使用相同的ActorRef。重启是由actor的父actor依据[监管策略](https://doc.akka.io/docs/akka/current/fault-tolerance.html#creating-a-supervisor-strategy)来处理的，关于重启的含义还有更多的讨论[restart意味着什么](https://doc.akka.io/docs/akka/current/general/supervision.html#supervision-restart)。

当actor停止时，其引用的生命周期就结束了。此时将调用适当的生命周期事件，并将终止通知该actor的监管者。当actor停止后，可以通过使用`actorof()`再次使用路径类创建actor。在这种情况下，新actor的名称将与前一个相同，但UID将不同。actor可以由本身、另一个actor或actor system停止（请参见[actor停止](https://doc.akka.io/docs/akka/current/actors.html#stopping-actors)）。

> 注意：            
需要注意的是，当不再引用actor时，actor不会自动停止，创建的每个actor也必须显式地销毁。惟一的简化是，停止父actor也将递归地停止父actor创建的所有子actor。


ActorRef总是代表一个引用（路径和uid），而不仅仅是一个给定的路径。因此，如果一个actor被停止，一个同名的新actor被创造出来，旧actor的引用就不会指向新的actor。

另一方面，`ActorSelection`指向一个路径(如果使用通配符，则指向多个路径)，并且完全不知道当前使用它的是哪个引用。由于这个原因，不能`ActorSelection`不能被监管。可以通过向`ActorSelection`发送`Identify`消息来解析当前引用的ActorRef，该消息将使用包含正确引用的`ActorIdentity`来回答(请参阅[ActorSelection](https://doc.akka.io/docs/akka/current/actors.html#actorselection))。也可以通过`ActorSelection`的`resolveOne`方法来实现，该方法返回匹配的`ActorRef`的`Future`。

### 生命周期监视
为了在另一个actor终止时得到通知（即永久停止，而不是临时失败和重新启动），actor可以注册自己，以便在终止时接收另一actor发送的`Terminated`消息。此服务由actor系统的`Deathwatch`组件提供。

```
import akka.actor.Terminated;
static class WatchActor extends AbstractActor {
  private final ActorRef child = getContext().actorOf(Props.empty(), "target");
  private ActorRef lastSender = system.deadLetters();

  public WatchActor() {
    getContext().watch(child); // <-- this is the only call needed for registration
  }

  @Override
  public Receive createReceive() {
    return receiveBuilder()
        .matchEquals(
            "kill",
            s -> {
              getContext().stop(child);
              lastSender = getSender();
            })
        .match(
            Terminated.class,
            t -> t.actor().equals(child),
            t -> {
              lastSender.tell("finished", getSelf());
            })
        .build();
  }
}
```

### actor其他钩子函数
```

//此方法在首次创建参与者时调用。
//在重新启动期间，postRestart的默认实现将调用该方法，这意味着通过覆盖该方法，您可以选择该方法中的初始化代码仅为该参与者调用一次，还是为每次重新启动调用一次。
//初始化代码是actor构造函数的一部分，它总是在创建actor类的实例时调用，这在每次重启时都会发生。
public void preStart() {}

```

actor重启只替换实际的actor对象;邮箱的内容不受重启的影响，因此在postRestart钩子返回后将继续处理消息。将不再接收已触发异常的消息。任何发送给正在重新启动的参与者的消息都将像往常一样排队到其邮箱。
```
public void postRestart() {}

```

```
//在停止一个actor之后，调用这个钩子函数，它可以用于从其他服务中注销这个actor。
//此钩子保证在此actor的消息队列被禁用后运行，即发送给已停止actor的消息将被重定向到ActorSystem的死信。
public void postStop() {}
```


### 通过ActorSection确定Actor
每个actor都有一个路径，可以通过路径来获取actor
```
// will look up this absolute path
getContext().actorSelection("/user/serviceA/actor");
// will look up sibling beneath same supervisor
getContext().actorSelection("../joe");
```
也可以通过通配符来获取多个actor
```
// will look all children to serviceB with names starting with worker
getContext().actorSelection("/user/serviceB/worker*");
// will look up all siblings beneath same supervisor
getContext().actorSelection("../*");
```
远程actor地址也可以查询
```
getContext().actorSelection("akka.tcp://app@otherhost:1234/user/serviceB");
```

### 发送消息
消息通过以下方法发送给actor。
- `tell` 的意思是“发了就忘”，例如异步发送消息并立即返回。
- `ask` 异步发送消息并返回一个表示可能的应答的消息。

发送格式
```
// don’t forget to think about who is the sender (2nd argument)
// 我最想吐槽的api设计
target.tell(message, getSelf());
```
发送方引用与消息一起被传递，在接收actor中，可用getSender()方法获取发送方。在actor的内部，通常是getSelf()作为发送方，但是在某些情况下，响应将被路由到其他actor，例如。第二个参数应该是另一个参数。如果不需要响应，则第二个参数可以为null。

**转发消息**

您可以将消息从一个actor转发到另一个actor。这意味着即使消息通过“中介”传递，也要维护原始的发送者地址/引用。这在编写路由器、负载平衡器、复制器等角色时非常有用。
```
target.forward(result, getContext());
```

### 接收消息
actor必须通过实现`AbstractActor`中`createReceive`方法来定义其初始接收行为:
```
@Override
public Receive createReceive() {
  return receiveBuilder().match(String.class, s -> System.out.println(s.toLowerCase())).build();
}
```
返回类型是`AbstractActor.Receive`，其定义了actor可以处理哪些消息，以及应该如何处理消息的实现。可以使用名为`ReceiveBuilder`的构建器构建此类行为。举个例子:
```
import akka.actor.AbstractActor;
import akka.event.Logging;
import akka.event.LoggingAdapter;

public class MyActor extends AbstractActor {
  private final LoggingAdapter log = Logging.getLogger(getContext().getSystem(), this);

  @Override
  public Receive createReceive() {
    return receiveBuilder()
        .match(
            String.class,
            s -> {
              log.info("Received String message: {}", s);
            })
        .matchAny(o -> log.info("received unknown message"))
        .build();
  }
}
```
或者另一种方式
```
import akka.actor.AbstractActor;
import akka.event.Logging;
import akka.event.LoggingAdapter;
import akka.japi.pf.ReceiveBuilder;

public class GraduallyBuiltActor extends AbstractActor {
  private final LoggingAdapter log = Logging.getLogger(getContext().getSystem(), this);

  @Override
  public Receive createReceive() {
    ReceiveBuilder builder = ReceiveBuilder.create();

    builder.match(
        String.class,
        s -> {
          log.info("Received String message: {}", s);
        });

    // do some other stuff in between

    builder.matchAny(o -> log.info("received unknown message"));

    return builder.build();
  }
}
```
或者
```
static class WellStructuredActor extends AbstractActor {

  public static class Msg1 {}

  public static class Msg2 {}

  public static class Msg3 {}

  @Override
  public Receive createReceive() {
    return receiveBuilder()
        .match(Msg1.class, this::receiveMsg1)
        .match(Msg2.class, this::receiveMsg2)
        .match(Msg3.class, this::receiveMsg3)
        .build();
  }

  private void receiveMsg1(Msg1 msg) {
    // actual work
  }

  private void receiveMsg2(Msg2 msg) {
    // actual work
  }

  private void receiveMsg3(Msg3 msg) {
    // actual work
  }
}
```

### 回复消息
这个就简单了，获取发送的ActorRef，再使用`tell`，就可以了
```
getSender().tell(msg, getSelf());
```

### 接收超时
`ActorContext` `setReceiveTimeout`定义了非活动超时，在此超时之后将触发ReceiveTimeout消息的发送。当指定时，receive函数应该能够处理`akka.actor.ReceiveTimeout`消息。1毫秒是支持的最小超时时间。

请注意，接收超时可能在另一条消息进入队列后立即触发并排队接收超时消息;因此，不能保证在接收到接收超时时，必须预先有一段空闲时间，这是通过此方法配置的。

一旦设置好，接收超时将一直有效(即在非活动期之后继续重复触发)。通过持续时间。未定义以关闭此功能。
```
static class ReceiveTimeoutActor extends AbstractActor {
  public ReceiveTimeoutActor() {
    // To set an initial delay
    getContext().setReceiveTimeout(Duration.ofSeconds(10));
  }

  @Override
  public Receive createReceive() {
    return receiveBuilder()
        .matchEquals(
            "Hello",
            s -> {
              // To set in a response to a message
              getContext().setReceiveTimeout(Duration.ofSeconds(1));
            })
        .match(
            ReceiveTimeout.class,
            r -> {
              // To turn it off
              getContext().cancelReceiveTimeout();
            })
        .build();
  }
}
```

### 定时消息

### 停止Actor
通过调用`ActoRefFactory`的`stop`方法（即`ActorContext`或`ActorSystem`）来停止actor。通常，上下文用于停止actor本身或子actor，以及停止顶级actor的系统。actor的实际终止是异步执行的，也就是说，`stop`可能会在参与者停止之前返回。
```
import akka.actor.ActorRef;
import akka.actor.AbstractActor;

public class MyStoppingActor extends AbstractActor {

  ActorRef child = null;

  // ... creation of child ...

  @Override
  public Receive createReceive() {
    return receiveBuilder()
        .matchEquals("interrupt-child", m -> getContext().stop(child))
        .matchEquals("done", m -> getContext().stop(getSelf()))
        .build();
  }
}
```

### Become/Unbecome
可以使用become实现有限状态机。
```
static class HotSwapActor extends AbstractActor {
  private AbstractActor.Receive angry;
  private AbstractActor.Receive happy;

  public HotSwapActor() {
    angry =
        receiveBuilder()
            .matchEquals(
                "foo",
                s -> {
                  getSender().tell("I am already angry?", getSelf());
                })
            .matchEquals(
                "bar",
                s -> {
                  getContext().become(happy);
                })
            .build();

    happy =
        receiveBuilder()
            .matchEquals(
                "bar",
                s -> {
                  getSender().tell("I am already happy :-)", getSelf());
                })
            .matchEquals(
                "foo",
                s -> {
                  getContext().become(angry);
                })
            .build();
  }

  @Override
  public Receive createReceive() {
    return receiveBuilder()
        .matchEquals("foo", s -> getContext().become(angry))
        .matchEquals("bar", s -> getContext().become(happy))
        .build();
  }
}
```

### actor和异常
当actor处理消息时，可能会抛出某种异常，例如数据库异常。

**消息**               

如果在处理消息时抛出异常(即从其邮箱中取出并传递给当前行为)，则此消息将丢失。重要的是要理解它不是放回邮箱的。因此，如果如果想重试消息处理，需要通过捕获异常并重试您的流来处理它。不希望系统死机的话，要确保对重试次数进行限制，(会消耗大量cpu周期)。

**邮箱**

如果在处理消息时抛出异常，则邮箱不会发生任何变化。如果重新启动actor，出现相同的邮箱。所以邮箱上的所有消息也都在那里。

**actor**

如果actor内的代码抛出异常，则该actor将被挂起并启动监管。根据supervisor的决定，actor会被恢复(好像什么都没有发生)、重新启动(清除其内部状态并从头开始)或终止。

### 初始化
actor生命周期的钩子函数提供了一个有用的工具包来实现各种初始化模式。在ActorRef的生命周期中，一个actor可能会经历多次重新启动，其中旧实例被一个新实例替换，使用ActorRef外部的观察者是看不到的。

每次实例化actor时都可能需要初始化，但有时只需要在创建ActorRef时的第一个实例诞生时进行初始化。下面的部分提供了不同初始化需求的模式。

#### 通过constructor来初始化
使用构造函数初始化有很多好处。首先，它使得存储在actor实例的val字段在其生命周期中不更改的任何状态，从而使actor的实现更加健壮。构造函数在创建actor实例时调用actorOf，在重新启动时也调用该构造函数，因此actor的内部始终可以假定已经进行了适当的初始化。这也是这种方法的缺点，因为在某些情况下，希望避免在重启时重新初始化内部。例如，在重启期间保留子actor的状态。
#### 通过preStart来初始化
actor的`preStart()`方法只在第一个实例初始化期间(即在创建其`ActorRef`时)被直接调用一次。在重新启动的情况下，从`postRestart()`调用`preStart()`，因此，如果没有被覆盖，则在每次重新启动时调用`preStart()`。但是，通过覆盖`postRestart()`可以禁用此行为，并确保只有一个`preStart()`调用。        
此模式的一个有用用法是在重新启动期间禁用为child创建新的`ActorRef`。这可以通过重写`preRestart()`来实现。下面是这些生命周期钩子的默认实现:
```
@Override
public void preStart() {
  // Initialize children here
}

// Overriding postRestart to disable the call to preStart()
// after restarts
@Override
public void postRestart(Throwable reason) {}

// The default implementation of preRestart() stops all the children
// of the actor. To opt-out from stopping the children, we
// have to override preRestart()
@Override
public void preRestart(Throwable reason, Optional<Object> message) throws Exception {
  // Keep the call to postStop(), but no stopping of children
  postStop();
}
```
#### 通过消息传递来初始化
在某些情况下，不可能在构造函数中传递参与者初始化所需的所有信息，例如在存在循环依赖项的情况下。在这种情况下，actor应该侦听初始化消息，并使用become()或有限状态机状态转换对actor的初始化和未初始化状态进行编码。
```
@Override
public Receive createReceive() {
  return receiveBuilder()
      .matchEquals(
          "init",
          m1 -> {
            initializeMe = "Up and running";
            getContext()
                .become(
                    receiveBuilder()
                        .matchEquals(
                            "U OK?",
                            m2 -> {
                              getSender().tell(initializeMe, getSelf());
                            })
                        .build());
          })
      .build();
}
```
如果actor可能在初始化之前接收消息，那么一个有用的工具是保存消息直到初始化完成，并在actor初始化之后重新播放消息。