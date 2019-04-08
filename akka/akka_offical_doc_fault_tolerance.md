# akka官方文档——容错

## 1.介绍
正如在Actor System中所解释的，每个Actor都是其子Actor的监管，因此每个Actor都定义了处理错误的监管策略。这一策略是Actor System中不可分割的一部分，且一旦定义后不可更改。


## 2.创建监管策略
下面几节将更深入地解释故障处理机制和备选方案。

```java
private static SupervisorStrategy strategy =
    //如果在withinTimeRange时间范围内，子Actor重启次数超过maxNofRetries的话，子Actor将停止
    new OneForOneStrategy(
        10,
        Duration.ofMinutes(1),          //每分钟最多重启子Actor 10次
        DeciderBuilder.match(ArithmeticException.class, e -> SupervisorStrategy.resume())
            .match(NullPointerException.class, e -> SupervisorStrategy.restart())
            .match(IllegalArgumentException.class, e -> SupervisorStrategy.stop())
            .matchAny(o -> SupervisorStrategy.escalate())
            .build());

@Override
public SupervisorStrategy supervisorStrategy() {
  return strategy;
}
```
Akka提供了两种策略实现：OneForOneStrategy和AllForOneStrategy。

- OneForOneStrategy：只有当前抛出异常的Actor响应该策略；
- AllForOneStrategy：该Actor监管的所有子Actor都响应该策略。

如果在`withinTimeRange`时间段内，子Actor重启超过`maxNOFRetries`次，子Actor将停止工作。

- maxNOFRetries = -1, withinTimeRange = Duration.Inf()，表示子Actor将立即重启
- maxNOFRetries = -1, withinTimeRange = non-infinite，maxNOFRetries视为1
- maxNOFRetries非负，withinTimeRange = Duration.Inf()，表示时间段无限长，但无论多长，一旦重启超过maxNOFRetries次，子Actor将停止。

当想在`/user`中使用


### 默认的监管策略
如果actor中没有定义监管策略的话，下列异常将被默认处理
- `ActorInitializationException` 停止出错的子actor
- `ActorKilledException` 停止出错的子actor
- `DeathPactException`  停止出错的子actor
- `Exception`  重启出错的子actor
-  其他`Throwable`  扔给父actor

如果一直传递到根监管者，还是按照上面定义的默认处理策略来处理。

### 停用监管策略
当子actor出错就停止它，且当监管者收到子actor的DeathWatch信号时再选择合适的处理动作，这一点非常类似于Erlang的策略。当想在`/user`中使用该策略，可通过StoppingSupervisorStrategy来配置。

### 记录Actor失败
默认情况下，如果该向上传递该故障，`SupervisorStrategy`会记录Actor的失败日志。向上传递的故障应该在层次结构的更高级别进行处理，并可能被记录下来。

在实例化时，可以设置`loggingEnabled`为false来禁用`SupervisorStrategy`的默认日志记录。再通过`Decider`来自定义日志记录。在监管Actor内部声明SupervisorStrategy时，当前失败的子Actor可以作为`sender`的引用。

也可以通过重写`logFailure`来自定义`SupervisorStrategy`的日志记录。

## 顶层actor的监管
那些用system.actofOF()创建出来的actor，将被“/user”监管。在这种情况下不使用任何特殊规则，监管者使用默认配置的策略。


## 测试程序
下一节将展示不同监管策略在实践中的效果。

首先，我们定义一个Supervisor:
```java
public
    // #supervisor
    static class Supervisor extends AbstractActor {

        LoggingAdapter log = Logging.getLogger(system, this);

        // #strategy
        private static SupervisorStrategy strategy =
                new OneForOneStrategy(
                        10,
                        Duration.ofMinutes(1),

                        DeciderBuilder.match(ArithmeticException.class, e -> SupervisorStrategy.resume())
                                .match(NullPointerException.class, e -> SupervisorStrategy.restart())
                                .match(IllegalArgumentException.class, e -> SupervisorStrategy.stop())
                                .matchAny(o -> SupervisorStrategy.escalate())
                                .build());

        @Override
        public SupervisorStrategy supervisorStrategy() {
            return strategy;
        }

        // #strategy
        @Override
        public Receive createReceive() {
            return receiveBuilder()
                    .match(
                            Props.class,
                            props -> {
                                getSender().tell(getContext().actorOf(props), getSelf());
                            })
                    .build();
        }
    }

```
这个Supervisor将被用来创造一个子Actor，我们可以用这个Actor做实验:
```java
public
    // #child
static class Child extends AbstractActor {
    int state = 0;

    LoggingAdapter log = Logging.getLogger(system, this);

    @Override
    public Receive createReceive() {
        return receiveBuilder()
                .match(
                        Exception.class,
                        exception -> {
                            throw exception;
                        })
                .match(Integer.class, i -> state = i)
                .matchEquals("get", s -> getSender().tell(state, getSelf()))
                .build();
    }
}
```
创建Actor
```java
Props superprops = Props.create(Supervisor.class);
ActorRef supervisor = system.actorOf(superprops, "supervisor");
ActorRef child =
    (ActorRef) Await.result(ask(supervisor, Props.create(Child.class), 5000), timeout);
```

第一个测试将演示Resume指令，通过在actor中设置一些非初始状态来进行测试，并使其失败:
```java
child.tell(42, ActorRef.noSender());
assert Await.result(ask(child, "get", 5000), timeout).equals(42);
child.tell(new ArithmeticException(), ActorRef.noSender());
assert Await.result(ask(child, "get", 5000), timeout).equals(42);
```
正如您所看到的，值42在错误处理指令之后仍然存在。现在，如果将失败改为更严重的NullPointerException，情况就不再是这样了:
```java
child.tell(new NullPointerException(), ActorRef.noSender());
assert Await.result(ask(child, "get", 5000), timeout).equals(0);
```
最后，如果出现致命的IllegalArgumentException，该子Actor将被Supervisor终止:
```java
final TestProbe probe = new TestProbe(system);
probe.watch(child);
child.tell(new IllegalArgumentException(), ActorRef.noSender());
probe.expectMsgClass(Terminated.class);
```

到目前为止，Supervisor完全不受子程序失败的影响，因为指令集确实处理了它。在异常的情况下，这将可能是错误的，Supervisor会升级故障。

