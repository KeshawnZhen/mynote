# AtomicInteger理解

在进行并发编程的时候我们需要确保程序在被多个线程并发访问时可以得到正确的结果，也就是实现线程安全。

线程安全需要实现两点：
> 1. 保证线程的内存可见性
> 2. 保证原子性

而AtomicInteger类可以在无锁的条件下实现线程安全的自增、更新等操作。

## AtomicIngeter 重要的API
```java
//以原子方式输入的数组与实例中的值（AtomocInteger.value）相加并返回结果
public final int addAndGet(int delta);

//如果输入的值等于预期值，则以原子方式将该值设置为输入的值
public final boolean compareAndSet(int expect, int update);

//以原子方式将当前值加1，返回的是自增前的值
public final int getAndIncrement();

//最终会设置成newValue
public final void lazySet(int newValue);

//以原子方式设置为newValue的值，并返回旧的值
public final int getAndSet(int newValue);
```

## getAndIncrement原子操作的实现
ActomicInteger.java
```java
private static final Unsafe unsafe = Unsafe.getUnsafe();

public final int getAndIncrement() {
    return unsafe.getAndAddInt(this, valueOffset, 1);
}
```
Unsafe.java
```java
//Unsafe.getAndAddInt()
//CAS循环判断是否符合预期
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
}

public native int getIntVolatile(Object var1, long var2);
public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
```
从源码中可以看到，`getAndIncrement`调用`Unsafe`类中的方法来实现功能。   

`Unsafe`中的方法`getAndAddInt()`的循环体主要做了三件事：
1. 获取当前值。 （通过volatile关键字保证可见性）
2. 计算出目标值：
3. 进行CAS操作，如果成功则跳出循环，如果失败则重复上述步骤。     


`Unsafe`中的方法`getIntVolatile`和`compareAndSwapInt`都是native的，交由jvm来做一些和平台相关的实现。

`Unsafe`是JDK内部的工具类，主要实现了平台相关的操作。

> sun.misc.Unsafe是JDK内部用的工具类。它通过暴露一些Java意义上说“不安全”的功能给Java层代码，来让JDK能够更多的使用Java代码来实现一些原本是平台相关的、需要使用native语言（例如C或C++）才可以实现的功能。该类不应该在JDK核心类库之外使用。


## 处理器是如何实现原子操作的
32为IA-32处理器使用基于对缓存加锁或总线加锁的方式来实现多处理器之间的原子操作。首先处理器会自动保证基本的内存操作的原子性。处理器保证从系统内存中读取或者写入一个字节是原子的，意思是当一个处理器读取一个字节时，其他处理器不能访问这个字节的内存地址。Pentium 6 和最新的处理器能自动保证其原子性的，比如跨总线宽度、跨多个缓存行和跨页表的访问。但是，处理器提供总线锁定和缓存锁定两个机制来保证复杂内存操作的原子性。
### 基础知识
|名称|英文|解释
|:--|:-:|:--|
|缓存行|Cache Line|缓存的最小操作单位
|比较并交换| Compare And Swap| CAS操作需要输入两个数值，一个旧值（期望操作前的值）和一个新值，在操作期间先比较旧值有没有发生变化，如果没有发生变化，才交换成新值，发生了变化则不交换
|CPU流水线| CPU piple | CPU流水线的工作方式像工业生产线上的装配流水线，在CPU中由5~6个不同功能的电路单元组成一条指令处理流水线，然后将一条X86指令分成5~6步后在由这些电路单元分布执行，这样就能实现在一个CPU时钟周期完成一条指令，因此提高CPU的运算速度
|内存顺序冲突| Memory order violation| 内存顺序冲突一般是有假共享引起的，假共享是指多个CPU同时修改同一个缓存行的不同部分而引起其中一个CPU的操作无效，当出现这个内存顺序冲突时，CPU必须清空流水线

## 总线锁保证原子性
**第一个机制是通过总线锁来保证原子性。** 如果多个处理器同时对共享变量进行读改写操作（i++就是经典的读改写操作），那么共享变量就会被多个处理器同时进行操作，这样读改写操作就不是原子的，操作完之后共享变量的值会个期望的不一致。例如，如果i=1，进行两次i++操作，期望的结果是3，但是可能结果是2。  
原因可能是多个处理器同时从各自的缓存中读取变量i,分别进行加1操作，然后分别写入系统内存中。那么，想要保住读改写共享变量的操作时原子的，就必须保证CPU1读改写共享变量的时候，CPU2不能操作缓存了该共享变量内存地址的缓存。   
处理器使用总线锁就是来解决这个问题的。所谓总线锁就是使用处理器来提供的一个LOCK#信号，当一个处理器在总线上输出此信号时，其他处理器的请求将被阻塞住，那么该处理器可以独占共享内存。
## 缓存锁保证原子性
**第二个机制是通过缓存锁来保证原子性。**  在同一时刻，只需要保证对某个内存地址的操作是原子性即可，d按总线锁把CPU和内存之间的通信锁住了，这使得锁定期间，其他处理器不能操作其他内存地址的数据，使用总线锁定的开销比较大，目前处理器在某些场合下使用缓存锁代替总线锁来进行优化。          
频繁使用的内存会缓存在处理器的L1、L2和L3高速缓存里，那么原子操作就可以直接在处理器内部缓存中进行，并不需要声明总线锁，在Pentium 6和目前的处理器中可以使用“缓存锁定”的方式来实现复杂的原子性。所谓“缓存锁定”是指内存区域如果被缓存在处理器的缓存中，并且在LOCK操作期间被锁定，那么当它执行锁操作回写到内存时，处理器不在总线上声言LOCK#信号，而是修改内部的内存地址，并允许他的缓存一致性机制来保证操作的原子性，因为缓存一致性机制会阻止同时修改两个以上处理器缓存的内存区域数据，当其他处理器回写已被锁定的缓存行的数据时，会使缓存行无效。对于共享变量i,当CPU1修改缓存行中的i时使用了缓存锁定，那么CPU2就不能同时缓存i的缓存行。

**但是有两种情况下处理器不会使用缓存锁定。**   
1. 当操作的数据不能被缓存在处理器内部，或操作的数据跨多个缓存行时，处理器会调用总线锁定。
2. 有些处理器不支持缓存锁定。对于Intel 484 和 Pentium处理器，就是锁定的内存区域在处理器的缓存行中也会调用总线锁定。


## CAS实现原子操作带来的问题
### 1.ABA问题
因为CAS需要在操作值的时候，检查值有没有发生变化，如果没有发生变化则更新，但是如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时会发现它的值没有发生变化，但是实际上却变化了。   
ABA问题多的解决思路是使用版本号。在变量前追加上版本号，每次变量更新的时候把版本号加1，那么`A→B→C`就会变成`1A→2B→3C`。         
JDK的Atomic包里提供`AtomicStampedReference`来解决ABA问题。这个类的compareAndSet方法的作用是首先检查当前引用是否等于预期引用，并且检查当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给的的更新值。

### 2.循环时间长开销大
自旋CAS如果长时间不成功，会给CPU带来非常大的执行开销。如果JVM能支持处理器提供的pause指令，那么效率会有一定的提升。pause指令有两个作用：1.延长流水线执行指令，是CPU不会消耗过多的执行资源，延迟的时间取决于具体实现的版本，在一些处理器上延迟的时间是0；2.避免在退出循环时因内存顺序冲突而引起CPU流水线被清空，从而提高CPU的执行效率。

### 3.只能保证一个共享变量的原子操作
当对一个额共享变量执行操作时，可以使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性。


参考文档：          
 1. [Java并发编程-无锁CAS与Unsafe类及其并发包Atomic](https://blog.csdn.net/javazejian/article/details/72772470)
 2. [深入解析Java AtomicInteger原子类型](https://juejin.im/post/5c22e5f46fb9a049f1543d0e)
 3. [Atomic*实现原理](https://juejin.im/post/5b9e5279f265da0af7750c3f)
 4. [伪共享](https://www.cnblogs.com/cyfonly/p/5800758.html)
 5. 《Java并发编程的艺术》