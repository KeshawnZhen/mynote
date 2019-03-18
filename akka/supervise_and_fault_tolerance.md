# 监管和容错

## 1.监管
Akka中的Actor系统，其中一个重要的概念就是分而治之。既然把任务分配给Actor去执行，那么就必须去监管相应的Actor，当Actor出现了失败，比如系统环境错误，各种异常，能根据我们制定的相应监管策略进行错误恢复，也就是容错。

### 监管者
看下ActorSystem中的顶级监管者：
![image](https://user-gold-cdn.xitu.io/2017/4/19/db8468f7af36435c7327f11855c37883?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

一个actor系统在其创建过程中至少要启动三个actor，如上图所示，下面来说说这三个Actor的功能：

#### `/`管理者
它监管着ActorSystem中所有的顶级Actor，顶级Actor有以下几种：
- `/user`： 是所有由用户创建的顶级actor的监管者；用ActorSystem.actorOf创建的actor在其下。
- `/system`： 是所有由系统创建的顶级actor的监管者，如日志监听器，或由配置指定在actor系统启动时自动部署的actor。
- `/deadLetters`： 是死信actor，所有发往已经终止或不存在的actor的消息会被重定向到这里。
- `/temp`：是所有系统创建的短时actor的监管者，例如那些在ActorRef.ask的实现中用到的actor。
- `/remote`： 是一个人造虚拟路径，用来存放所有其监管者是远程actor引用的actor。

平常用的最多的就是/user，它是我们在程序中用ActorSystem.actorOf创建的actor的监管者。

根监管者监管着所有顶级Actor，对它们的各种失败情况进行处理，一般来说如果错误要上升到根监管者，整个系统就会停止。

#### `/user`： 顶级actor监管者
`/user`是所有由用户创建的顶级actor的监管者，即用ActorSystem.actorOf创建的actor，我们可以自己制定相应的监管策略，但由于它是actor系统启动时就产生的，所以我们需要在相应的配置文件里配置，具体的配置可以参考这里[Akka配置](https://doc.akka.io/docs/akka/current/general/configuration.html)

#### `/system`： 系统监管者
`/system`所有由系统创建的顶级actor的监管者,比如Akka中的日志监听器，因为在Akka中日志本身也是用Actor实现的，`/system`的监管策略如下：对收到的除ActorInitializationException和ActorKilledException之外的所有Exception无限地执行重启，当然这也会终止其所有子actor。所有其他Throwable被上升到根监管者，然后整个actor系统将会关闭。

Actor系统的组织结构是一种树形结构。这种结构对actor的监管是非常有利的，Akka实现的是一种叫“父监管”的形式，每一个被创建的actor都由其父亲所监管，这种限制使得actor的监管结构隐式符合其树形结构，所以我们可以得出一个结论：

> 一个被创建的Actor肯定是一个被监管者，也可能是一个监管者，它监管着它的子级Actor

### 监管策略


Akka中有以下4种策略：

- 恢复下属，保持下属当前积累的内部状态
- 重启下属，清除下属的内部状态
- 永久地停止下属
- 升级失败（沿监管树向上传递失败），由此失败自己

Akka中有两种类型的监管策略：`OneForOneStrategy`和`AllForOneStrategy`，它们的主要区别在于：

`OneForOneStrategy`： 该策略只会应用到发生故障的子actor上。
`AllForOneStrategy`： 该策略会应用到所有的子actor上。

一般都使用`OneForOneStrategy`来进行制定相关监管策略，当然可以根据具体需求选择合适的策略。另外k可以给策略配置相应参数，比如上面maxNrOfRetries，withinTimeRange等，这里的含义是每分钟最多进行多少次重启，若超出这个界限相应的Actor将会被停止，当然也可以使用策略的默认配置，具体配置信息可以参考源码。


### 监管容错示例

演示Actor在发生错误时，它的监管者会根据相应的监管策略进行不同的处理。



