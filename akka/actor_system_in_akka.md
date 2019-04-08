# akka中的actor系统

Actor模型是akka中最核心的内容，那么akka是用什么来管理、组织这些actor的呢？

## actor系统
Actor作为一种封装状态和行为的对象，总是需要一个系统去统一的组织和管理它们，在Akka中即为ActorSystem，其实这非常容易理解，好比一个公司，每个员工都可以看成一个Actor，它们有自己的职位和职责，但是我们需要把员工集合起来，统一进行管理和分配任务，所以我们需要一个相应的系统进行管理，好比这里的ActorSystem对Actor进行管理一样。

### ActorSystem的主要功能
ActorSystem的功能有：
1. 管理调度服务
2. 配置相关参数
3. 日志功能


**1.管理调度服务**

ActorSystem的的精髓在于将任务分拆，直到一个任务小到可以被完整处理，然后将其委托给Actor进行处理，所以ActorSystem最核心的一个功能就是管理和调度整个系统的运行，好比一个公司的管理者，他需要制定整个公司的发展计划，还需要将工作分配给相应的工作人员去完成，保障整个公司的正确运转，其实这里也体现了软件设计中的分治思想，Actor中的核心思想也是这样。

![image](https://user-gold-cdn.xitu.io/2017/4/6/abee270a1cef2fef3d712b8993177171?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

上图是一个简单的开发协作的过程，我觉得这个例子应该可以清晰的表达Akka中Actor的组织结构，当然不仅于此。主要有以下几个特点：

- Akka中Actor的组织是一种树形结构
- 每个Actor都有父级，有可能有子级当然也可能没有
- 父级Actor给其子级Actor分配资源，任务，并管理其的生命状态（监管和监控）

Actor系统往往有成千上万个Actor，使用树形机构来组织管理Actor是非常适合的。

而且Akka天生就是分布式，你可以向一个远程的Actor发送消息，但你需要知道这个Actor的具体位置在哪，这时候你就会发现，树形结构对于确定一个Actor的路径来说是非常有利（比如Linux的文件存储）。

**2.根据配置创建环境**    


一个完善的ActorSystem必须有相关的配置信息，比如使用的日志管理，不同环境打印的日志级别，拦截器，邮箱等等，Akka使用Typesafe配置库，这是一个非常强大的配置库。

下面用一个简单的例子来说明一下ActorSystem会根据配置文件内容去生成相应的Actor系统环境：

1.首先我们按照默认配置打印一下系统的日志级别，
```
ActorSystem system = ActorSystem.create("RobotTest", ConfigFactory.load("akka.conf"));
System.out.println(system.settings().LogLevel());
```
运行结果
```
INFO
```

修改akka.conf文件
```
akka {

    # Log level used by the configured loggers (see "loggers") as soon
    # as they have been started; before that, see "stdout-loglevel"
    # Options: OFF, ERROR, WARNING, INFO, DEBUG
    loglevel = "DEBUG"

}
```
再次执行上述代码。运行结果
```
[DEBUG] [03/15/2019 16:34:35.467] [main] [EventStream(akka://RobotTest)] logger log1-Logging$DefaultLogger started
[DEBUG] [03/15/2019 16:34:35.468] [main] [EventStream(akka://RobotTest)] Default Loggers started
DEBUG
```
可以发现ActorSystem的日志输出级别已经变成了DEBUG。

这里主要是演示ActorSystem可以根据配置文件的内容去加载相应的环境，并应用到整个ActorSystem中，这对于配置ActorSystem环境来说是非常方便的。

**3.日志功能**

Akka拥有高容错机制，这无疑需要完善的日志记录才能使Actor出错后能及时做出相应的恢复策略，比如Akka中的持久化。

## Actor引用，路径和地址

Actor引用是ActorRef的子类，每个Actor有唯一的ActorRef，Actor引用可以看成是Actor的代理，与Actor打交道都需要通过Actor引用，Actor引用可以帮对应Actor发送消息，也可以接收消息，向Actor发送消息其实是将消息发送到Actor对应的引用上，再由它将消息投寄到具体Actor的信箱中，所以ActorRef在整个Actor系统是一个非常重要的角色。

**如何获得Actor引用？**

- 直接创建Actor
- 查找已经存在的Actor

**1.获得ActorRef**        

假定现在由这么一个场景：老板嗅到了市场上的一个商机，准备开启一个新项目，他将要求传达给了经理，经理根据相应的需求，来安排适合的的员工进行工作。

1.创建一些消息
```
//Message.java
public abstract  class Message {

    private String content;

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }

    public Message(String content) {
        this.content = content;
    }
}

//Meeting.java
public class Meeting extends Message {
    public Meeting(String content) {
        super(content);
    }
}

//Business.java
public class Business extends Message {

    public Business(String content) {
        super(content);
    }
}

//Confirm.java
public class Confirm extends Message {
    public Confirm(String content) {
        super(content);
    }
}

//DoAction.java
public class DoAction extends Message {
    public DoAction(String content) {
        super(content);
    }
}

//Done.java
public class Done extends Message{
    public Done(String content) {
        super(content);
    }
}
```

2.我们来创建一家公司，这里就是ActorSystem的化身：
```
public class TestCompany {

    public static void main(String[] args) {
        ActorSystem system = ActorSystem.create("company-system", ConfigFactory.load("akka.conf"));
        ActorRef bossActor = system.actorOf(Props.create(BossActor.class), "boss");

        bossActor.tell(new Business("Fitness industry has great prospects"), bossActor);
    }

}
```
3.这里我们会创建几种角色，比如上面Boss，这里我们还有Manager，Worker。
```
//BossActor.java
public class BossActor extends UntypedActor {


    private List<ActorRef> managerList = new ArrayList<>();

    private LoggingAdapter log = Logging.getLogger(getContext().system(), this);

    private int taskCount = 0;

    @Override
    public void preStart() throws Exception {
        initManagerList();

    }

    @Override
    public void onReceive(Object o) throws Exception {

        if(o instanceof Business){
            log.info("I must to do some thing,go,go,go!");
            System.out.println(getSelf().path().address());
            //发送消息给每个manager
            for (ActorRef manager : managerList) {
                manager.tell(new Meeting("metting to discuss big plans"), getSelf());
            }
        } else if(o instanceof Confirm){

            //
            log.info(getSender().path().name());

            //根据Actor路径查找已经存在的Actor获得Actor的引用
            ActorSelection manager = getContext().actorSelection(getSender().path());
            //ActorRef ma = getSender();
            manager.tell(new DoAction("Do thing"), getSelf());

        } else if (o instanceof Done){
            taskCount++;
            if(taskCount == 3){
                log.info("the project is done, we will earn more money");
                getContext().system().terminate();
            }
        } else{
            unhandled(o);
        }
    }

    private void initManagerList(){
        for(int i = 0; i < 3; i ++){
            managerList.add(getContext().actorOf(Props.create(ManagerActor.class), "manager" + i));
        }
    }

}

//ManagerActor.java
public class ManagerActor extends UntypedActor {

    private LoggingAdapter log = Logging.getLogger(getContext().system(), this);

    @Override
    public void onReceive(Object o) throws Exception {
        if(o instanceof Meeting){
            //接收到来自boss的开会消息，并回复给boss
            ActorRef sender = getSender();
            sender.tell(new Confirm("I receive command"), getSelf());
        } else if(o instanceof DoAction){
            //接受到来自boss的做事消息，分配给worker,把消息转发给worker
            ActorRef workerActor = getContext().actorOf(Props.create(WorkerActor.class), "worker");
            workerActor.forward(new DoAction("Do thing"), getContext());
        } else {
            unhandled(o);
        }
    }
}

//WorkerActor.java
public class WorkerActor extends UntypedActor {

    private LoggingAdapter log = Logging.getLogger(getContext().system(), this);

    @Override
    public void onReceive(Object o) throws Exception {
        if(o instanceof DoAction){
            log.info("I have receive task");
            log.info("woker: " + getSelf().path().name());
            ActorRef sender = getSender();
            //由于消息是由Manager转发的，getSender()的结果是boss
            //这里是worker把工作完成的消息传递给boss
            getSender().tell(new Done("I have done work"), getSelf());
        }
    }
}
```

程序流程图：            
![image](https://github.com/KeshawnZhen/mynote/blob/master/akka/pic/company_model.jpg)


这里主要是有两个知识点：          
- 创建Actor获得ActorRef的两种方式
- 根据Actor路径获得ActorRef

运行结果
```
akka://company-system
[INFO] [03/19/2019 19:39:50.558] [company-system-akka.actor.default-dispatcher-2] [akka://company-system/user/boss] I must to do some thing,go,go,go!
[INFO] [03/19/2019 19:39:50.558] [company-system-akka.actor.default-dispatcher-2] [akka://company-system/user/boss] manager2
[INFO] [03/19/2019 19:39:50.576] [company-system-akka.actor.default-dispatcher-2] [akka://company-system/user/boss] manager1
[INFO] [03/19/2019 19:39:50.576] [company-system-akka.actor.default-dispatcher-2] [akka://company-system/user/boss] manager0
[INFO] [03/19/2019 19:39:50.576] [company-system-akka.actor.default-dispatcher-4] [akka://company-system/user/boss/manager0/worker] I have receive task
[INFO] [03/19/2019 19:39:50.576] [company-system-akka.actor.default-dispatcher-4] [akka://company-system/user/boss/manager0/worker] woker: worker
[INFO] [03/19/2019 19:39:50.576] [company-system-akka.actor.default-dispatcher-2] [akka://company-system/user/boss/manager1/worker] I have receive task
[INFO] [03/19/2019 19:39:50.576] [company-system-akka.actor.default-dispatcher-2] [akka://company-system/user/boss/manager1/worker] woker: worker
[INFO] [03/19/2019 19:39:50.576] [company-system-akka.actor.default-dispatcher-3] [akka://company-system/user/boss/manager2/worker] I have receive task
[INFO] [03/19/2019 19:39:50.576] [company-system-akka.actor.default-dispatcher-3] [akka://company-system/user/boss/manager2/worker] woker: worker
[INFO] [03/19/2019 19:39:50.576] [company-system-akka.actor.default-dispatcher-8] [akka://company-system/user/boss] the project is done, we will earn more money
```


**2.Actor路径与地址**

ActorSystem中的路径类似Unix，每个ActorSystem都有一个根守护者，用`/`表示,在根守护者下有一个名user的Actor，它是所有system.actorOf()创建的父Actor，所以我们程序中bossActor的路径为：

`/user/boss`

地址顾名思义是Actor所在的位置，为什么要有地址这一个概念，这就是Akka强大的理念了，Akka中所有的东西都是被设计为在分布式环境下工作的，所以我们可以向任意位置的Actor发送消息（前提你得知道它在哪），这时候地址的作用就显现出来来，首先我们可以根据地址找到Actor在什么位置，再根据路径找到具体的Actor，比如我们示例程序中bossActor，它的完整位置是

`akka://company-system/user/boss`

其中akka代表纯本地的，Akka中默认远程Actor的位置一般用akka.tcp或者akka.udp开头，当然也可以使用第三方插件。

[参考文档](https://juejin.im/post/58e59524a0bb9f0069069148)