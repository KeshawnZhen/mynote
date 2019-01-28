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


## 处理器是如何实现原子操作的（TODO）

## CAS实现原子操作带来的问题
### 1.ABA问题
因为CAS需要在操作值的时候，检查值有没有发生变化，如果没有发生变化则更新，但是如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时会发现它的值没有发生变化，但是实际上却变化了。   
ABA问题多的解决思路是使用版本号。在变量前追加上版本号，每次变量更新的时候把版本号加1，那么`A→B→C`就会变成`1A→2B→3C`。         
JDK的Atomic包里提供`AtomicStampedReference`来解决ABA问题。这个类的compareAndSet方法的作用是首先检查当前引用是否等于预期引用，并且检查当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给的的更新值。

### 2.循环时间长开销大
自旋CAS如果长时间不成功，会给CPU带来非常大的执行开销。如果JVM能支持处理器提供的pause指令，那么效率会有一定的提升。pause指令有两个作用：1.延长流水线执行指令，是CPU不会消耗过多的执行资源，延迟的时间取决于具体实现的版本，在一些处理器上延迟的时间是0；2.避免在退出循环时因内存顺序冲突而引起CPU流水线被清空，从而提高CPU的执行效率。

### 3.只能保证一个共享变量的原子操作
当对一个额共享变量执行操作时，可以使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性。