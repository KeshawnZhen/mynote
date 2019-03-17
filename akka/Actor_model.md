# Actor模型

计算机科学中的**Actor模型**是一种并行计算的数学模型，它将**Actor**作为并行计算的通用原语。为了响应接收到的消息，Actor可以：做出本地决策，创建更多Actor，发送更多消息，并确定如何响应接收到的下一条消息。Actor可以修改自己的私有状态，但只能通过消息相互影响（避免需要任何锁）。

Actor模型起源于1973年。它既被用作理论理解计算的框架，也被用作并发系统的几种实际实现的理论基础。

简单来说，Actor之间**只能**通过消息来相互交流，并能根据消息作出响应。至于消息是怎么被传递的，某个Actor一下子接到很多消息怎么来处理的，这些细节问题可以先不用管。

![image](https://user-gold-cdn.xitu.io/2017/3/23/28707ed764976e2eb60a0fce8eea984a?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

如上图，Actor和Actor之间只能用消息来通信，当某个Actor给另一个Actor发消息，消息是有序的，你只管将消息发送给指定的Actor就行了。你可以等待它回复，也可以干其他事情。至于对方Actor会怎么处理你的消息你并不知道。

Actor有以下几个特点：            
- 每个Actor都有一个对应的邮箱
- Actor是串行处理消息的
- Actor的消息是不可变的

Actor在处理并发问题上的优势：         

**1.简化并发编程**        

并发导致最大的问题就是对共享数据的操作，而在面对并发问题时多采用的是用锁去保证共享数据的一致性。但这同样也会带来其他相关问题，比如要去考虑锁的粒度（对方法，程序块等），锁的形式（读锁，写锁等）等问题，这些问题对并发程序来说是至关重要的。但一个初写并发程序的程序员来说，往往不能掌控的很好，这无疑给程序员在编程上提高了复杂性，而且还不容易掌控。但使用Actor就不导致这些问题，首先Actor的消息特性就觉得了在与Actor通信上不会有共享数据的困扰，另外在Actor内部是串行处理消息的，同样不会对Actor内的数据造成污染，用Actor编写并发程序无疑大大降低了编码的复杂度。

**2.提升程序性能**            

Actor用单线程处理消息，那如何保证程序的性能？首先Actor是非常轻量级的，可以再程序中创建许多个Actor（1G内存可以容纳百万级别个Actor），而且Actor是异步的，那么如何利用它的这个特性呢。要做的就是把相应的并发事件尽可能的分割成一个个小的事件，让每个Actor去处理相应的小事件,充分去利用它异步的特点，来提升程序的性能。
其实Scala中原生的Actor并不能完成很多事，不是一套完整的并发解决方案，不适合用于生产环境，比如错误恢复，状态持久化等，所以在较新版本的Scala类库中，Akka包已经取代了原生的Actor。


# Akka
akka作为一套成熟的并发解决方案，已经被业界大量采用，尤其是在金融，游戏等领域，Akka中的容错机制，持久化，远程调用，日志等都是很重要的模块。

用一个程序来尝试使用一个akka         

传递消息的实体         
Msg.java
```
public final class Msg {

    public MsgType order;
    private long time;

    public Msg(MsgType order, long time) {
        this.order = order;
        this.time = time;
    }

    public MsgType getOrder() {
        return order;
    }

    public void setOrder(MsgType order) {
        this.order = order;
    }

    public long getTime() {
        return time;
    }

    public void setTime(long time) {
        this.time = time;
    }
}
```

消息类型枚举           
MsgType.java
```
public enum MsgType {
    TurnOnLight, BoilWater;
}
```

用来接收消息、处理消息的对象           
RobotActor.java
```
public class RobotActor extends UntypedActor {

    private LoggingAdapter log = Logging.getLogger(getContext().system(), this);

    @Override
    public void onReceive(Object o) throws Exception {
        if(o instanceof Msg){
            Msg a = (Msg) o;
            if(a.order == MsgType.TurnOnLight){
                log.info("turn on light, time: " + a.getTime());
            }else if(a.order == MsgType.BoilWater){
                log.info("water boil, time: " + a.getTime());
            }else{
                unhandled(a);
            }
        }else{
            unhandled(o);
        }

    }
}
```

主函数       
```
public class TestRobotActor {

    public static void main(String[] args) {
        //构建一个ActorSystem，后面会提
        ActorSystem system = ActorSystem.create("RobotTest", ConfigFactory.load("akka.conf"));

        //一个对RobotActor的引用
        ActorRef robotActor = system.actorOf(Props.create(RobotActor.class), "RobotActor");

        //传递消息给robotActor
        robotActor.tell(new Msg(MsgType.TurnOnLight, System.currentTimeMillis()), robotActor);
        robotActor.tell(new Msg(MsgType.BoilWater, System.currentTimeMillis()), robotActor);
        robotActor.tell(new Msg(MsgType.BoilWater, System.currentTimeMillis()), robotActor);
    }
}
```



运行结果：
```
[INFO] [03/15/2019 16:14:50.027] [RobotTest-akka.actor.default-dispatcher-5] [akka://RobotTest/user/RobotActor] turn on light, time: 1552637690020
[INFO] [03/15/2019 16:14:50.028] [RobotTest-akka.actor.default-dispatcher-5] [akka://RobotTest/user/RobotActor] water boil, time: 1552637690020
[INFO] [03/15/2019 16:14:50.029] [RobotTest-akka.actor.default-dispatcher-5] [akka://RobotTest/user/RobotActor] water boil, time: 1552637690020
```
