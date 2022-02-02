算法+ 数据结构

设计模式 多线程 JVM


[TOC]

## 数据一致性

数据一致性是寄存器的数据没有及时同步到内存，数据和不一致。其它线程读取内存的数据，和另一个线程持有的寄存器中的数据不一致。

## 保证数据一致性的手段

### 关中断

### 缓存一致性协议

缓存一致性是有硬件实现的。

在 CPU 和内存的访问速度速度之间差别太大，甚至有一百多倍，为了加快运算，在 CPU 和内存之间加入了访问速度快的缓存。

目前工业的实践基本是三级缓存，未来内存速度更快可能会减少，CPU 和 内存速度差别更大的话，可能会增加更多。


L1 L2 是和核共享的，每核一个，L3 是多核共享的，整个 CPU 只有一个。

每次读取数据到缓存中的数据块称为缓存行，目前三级缓存的情况下，工业实践缓存行大小为 64 字节为最佳。

不同核的缓存要保持一致。所以就出现了缓存一致性协议。最出名的就是Intel 的MESI协议，MESI协议保证了每个缓存中使用的共享变量的副本是一致的。它核心的思想是：当CPU写数据时，如果发现操作的变量是共享变量，即在其他CPU中也存在该变量的副本，会发出信号通知其他CPU将该变量的缓存行置为无效状态，因此当其他CPU需要读取这个变量时，发现自己缓存中缓存该变量的缓存行是无效的，那么它就会从内存重新读取。

JDK 1.7 LinkedBlockingQueue 使用了。

Disruptor RingBuffer 的实现。

JDK 1.8 增加了 @Contended, 保证注解的类的变量在单独一行中。同时要打开 JVM 的一项配置。

```JAVA
@Contended // 保证 x 位于单独缓存行中。
class T {
    public long x = 0L;
}
```

缓存一致性协议是硬件厂商决定的，有很多类型，例如 Intel 的 MESI。

### 系统屏障
### 总线/缓存锁



## synchronized 底层实现

synchronized 早期重量级，向系统申请。
后来改造，

执行时间长适合用系统锁/线程数量比较多。
执行时间短且线程数量比较少适合用自旋锁。

## Thread Executor, HandlerThread, AsyncTask 怎么选

如果想要执行后台任务，不与前台交互，可以使用, Thread 或者 Executor，而且推荐使用 Executor 线程池。

如果与前台频繁交互，例如下载进度条等则可以使用 AsyncTask，而 HandlerThread 基本没有什么使用场景。如果一次性交互，可以使用 Handler 或者其他切换线程的库。

Service 和 IntentService（带有单线程的后台一次性 Service，执行完后就自动退出了）。Servide 主要处理长时间的任务，除非用户为其他应用提供服务或者播放器等用户服务。否则都不建议使用，而是使用 DownloadManager 或者 TaskManager 代替。


## 实现 Rannable 接口的好处

1. 线程的任务从线程子类中分离处理，进行单独的封装，更符合面向对象的思想。
2. 避免 Java 单继承的局限性。


## 几种创建 Thread 的方法

1. 继承 Thread 类，重写 run 方法。
2. 实现 Runnable 接口。传递给 Thread 对象。同时覆盖和传递 runnable 将执行 Thread 自身覆盖的 `run` 方法。
3. ThreadFactory，其实内部还是自己 New Thread。只不过可以用与生产一批类似的线程。
4. Excturor 线程池，不用时及时关闭 shutdown。
    1. 单线程的线程池 singleThreadExecutor()。可以指定执行的顺序（FIFO,LIFO）/
    2. 固定数量线程池 fixedThreadExecutor()
    3. 可以动态增长的线程池 newCachePoolExecutor() 不限制数量
    4. newScheduledThreadPool 支持定时及周期执行任务。
5. Callable 有返回值的线程，通过 `Future.get()` 获取结果

## 线程间通信

包括 wait(), notify(). notifyAll()，join(), yield()。

1. 这些方法都必须在同步代码块内使用。因为这些方法都是用于操作线程状态的方法，必须要明确到底要操作的是哪个锁上的线程。
2. 为什么这些方法内定义在了 Object 类中？ 因为这些方法是监视器方法，监视器可以是任意对象，任意对象都有的方法一定是在 Object 中。


## 线程阻塞

该线程放弃 CPU 的使用，暂停执行。只有等到导致阻塞的原因消除之后才能运行。

1. sleep(), wait(), yield(), join(),(suspend(), resume() 已废弃)
2. 执行一段代码无法获得相关锁。
3. IO 操作等待相关资源。

有争议的地方，Java 中对阻塞的定义


```
BLOCKED：Thread state for a thread blocked waiting for a monitor lock.
A thread in the blocked state is waiting for a monitor lock to enter a synchronized block/method or reenter a synchronized block/method after calling ｀Object.wait｀
```

## 线程的状态图

![Thread status transform](images/thread_status_transform.png)



## interrupt 和 stop 的区别，为什么 stop 被废弃。

- interrupt 是在线程中设置了一个标志位，需要在 `run` 方法中自己判断标志位来终止。`Thread.isInterrupe()` 会在返回之后，把标志位置为 true，这样方便下次再次执行；而 `isInterrupt()` 不会设置标志位。
- stop 方法类似将线程 kill 掉，结果不可预期。释放它已经锁定的所有监视器。可能产生数据的不一致性。已经被废弃。

- 正在 `sleep` 的线程，被执行 `interrupt()` 将会终止休眠，同时抛出 `InterruptedException`，此时捕获异常可以做一些善后工作。
- Android 中有个 `SystenClock.sleep()` 不会抛出 ｀InterruptedException`，同时也不会被打断休眠状态，可以用于特殊情况。

```
try {
    Thread.sleep(1000)
} catch (ex: InterruptedException) {
    // 正在睡的时候，执行了 `interrupt` 将会直接被激活，然后抛出 `InterruptedException`
}
```

## 线程停止方法的比较。

1. stop 方法不安全，已经使用 interrupt 代替。
2. run 方法结束，或者判断`isInterrupt`标记位。线程可能被 wait 后进入到冻结动态，无法恢复或者判断标记位。可以使用 interrupt() 将线程中冻结或者休眠状态激活重新获得执行资格。

## 线程操作方法介绍

> join

在一个线程中调用另一个线程的 `join`，会等待另一个线程执行完毕之后，才执行后面的代码。join 的线程并不会立即执行，而是和其他具有执行权的线程进入分配队列等待线程调度。

> yield()

正在执行的线程获取到了执行时间片，执行 `yield()` 时，会主动让出时间片，然后到排队等待的队列中等待下一个时间片。

> setDeamon(true)

设置线程为守护线程。普通线程会阻塞进程，所有线程都结束后才会结束进程。而守护线程优先级比较低，不会阻塞进程结束。

1. 开启和任何操作都和普通线程一样。只需要设置线程 `setDeamon(true)`

> 优先级

获取 CPU 执行的记录，即线程调度的权重。1~10

MAX_PRIORITY = 10
NORM_PRIORITY = 5
MIN_PRIORITY = 1

> 线程组

在构造函数中指定，可以集体判断等操作处理。

## 多生产者多消费者关系

1. 为了防止被同类唤醒，而又不需要运行，要循环判断标记位。
2. 为了防止循环判断成立时有进入等待状态，从而全部进入等待状态，而进入死锁，要用 notifyAll 唤醒所有线程，一定会唤醒对象线程。



## wait 和 sleep 的区别

1. wait 可以指定时间，也可以不同指定。
2. 在同步中时，对 cup 的执行权的实例方式不同。wait 释放执行权，释放锁。sleep 释放执行权，但不释放锁。


## 加锁的类型

1. 同步代码快
2. 同步函数。
    1. 并不是所有内容都可以放在同步函数中。
    2. 同步函数的锁不是 Object，而是 this 对象。
3. 静态同步函数
    锁的是当前类的字节码。this.getClasss(). 或者 <类名>.class
    t.getClass() 和 Ticket.class 等价

## synchronized 修饰的类型

1. 修饰方法
2. 修饰代码块

两者等价

```
public synchronized void method()
{
   // todo
}

public void method()
{
   synchronized(this/object) {
      // todo
   }
}

```

3. 修饰静态方法
4. 修饰类 `synchronized(ClassName.class) {}`

两者是等价的


## 不能继承

synchronized关键字不能继承。
虽然可以使用synchronized来定义方法，但synchronized并不属于方法定义的一部分，因此，synchronized关键字不能被继承。如果在父类中的某个方法使用了synchronized关键字，而在子类中覆盖了这个方法，在子类中的这个方法默认情况下并不是同步的，而必须显式地在子类的这个方法中加上synchronized关键字才可以。当然，还可以在子类方法中调用父类中相应的方法，这样虽然子类中的方法不是同步的，但子类调用了父类的同步方法，因此，子类的方法也就相当于同步了。

## Lock 优点

原锁为一个块的封装体，对锁的操作是隐式的，无法进行灵活的操作。到了 v1.5 将锁对象化，将隐式操作显式化，可以灵活地获取释放锁。

```
Lock lock = new ReetrantLock(); 可重入的互斥锁。
```

Lock.unLock() 要放在 `finally` 块中。

Condation 将 Object 监视方法（wait(),notify,notifyAll) 分解为截然不同的对象，以便将这些对象与任意 lock 实现组合，为每个对象提供多个等待的 set, 其中 Lock 代替了原有 synchronized 关键字的使用。condation 代替了 Ojeect 监视器方法的使用。

方法名的改变

wait －> await
notify  －> signal
notifyAll －> signalAll


## 死锁

1. A 线程持有资源 x, 要获取 y。Ｂ 线程持有资源 y，要获取 x。

它们一定是双锁，同时产生了嵌套。一个获取了锁 a，代码块内部想要获取锁 b. 另一个线程获取了锁 b，内部同步代码执行时想要获取锁 a。由于是嵌套的，不获取的时候还释放不了 a。

## 单例模式的问题

单例的变量要使用 volatile 声明变量，保证基本数据类型的操作是线程同步的。例如虚拟机上 double 的赋值可能会被分成两步。防止变量还没初始化完成就返回引用供其他程序访问而出现 null 的情况。

volatile 不能保证对象，++,-- 等操作的线程安全。

对于`++`,`--` 这些操作，为了线程安全，如果使用 synchronized 则太重了。这时候可以使用带 `Atomic` 前缀的基本数据类型，它们的操作是原子性的。

1. 恶汉模式不存在线程安全。
2.


## 乐观锁和悲观锁

- 乐观锁是读数据的时候不会锁数据，假设数据不会被修改，而是等到写入的时候再检查是否有修改，有修改就会更新失败。没有修改就能够更新成功。但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，可以使用版本号等机制。乐观锁适用于多读的应用类型，这样可以提高吞吐量。

- 悲观锁总是假设最坏的情况，是觉得别人会修改数据，在读取的时候就先加锁，写入完毕的时候再释放锁。

乐观锁与悲观锁并不是特指中实现的锁，而是在并发情况下保证数据完整性的不同策略。乐观锁指是一种基于冲突检测的方法，检测到冲突时操作就会失败。因此需要在更新失败的时候，加载新数据就更新。乐观锁的实现依赖于 CAS 或者 TAS 操作或者数据版本机制（实现数据版本一般有两种，第一种是使用版本号，第二种是使用时间戳。遗留问题：数据版本机制在多内核中还能实现吗？有硬件支持吗？），也就是硬件上支持。实现可以使用”或“CAS操作”

悲观锁适合写操作非常多的场景，乐观锁适合读操作非常多的场景，不加锁会带来大量的性能提升。

乐观锁的具体实现有：Java Atomic 系列类，ReentrantLock 中的 FairSync 类等。


ABA问题
ABA问题是使用CAS时最常出现的一个问题。它的整个过程如下图所示：

```
A(线程2修改一次值，线程3改回原值)->>线程1: 读
A(线程2修改一次值，线程3改回原值)->>线程2: 读
线程2->>A(线程2修改一次值，线程3改回原值):写
A(线程2修改一次值，线程3改回原值)->>线程3: 读
线程3->>A(线程2修改一次值，线程3改回原值):写
线程1->>A(线程2修改一次值，线程3改回原值):写
```

通过上图我们可以看到虽然线程1仍然可以CAS写成功，但是它并没有感受到该A值在整个过程当中发生的问题。有可能它的值没有发生变化，而其含义却已经发生了变化。其实这种场景也非常常见，比如在我们的业务当中，我们需要在修改记录数据之前验证数据是否发生过变化，如果没有发生变化则进行写入，如果发生变化则放弃。这样可以在一定程度上提高并发度。

那么如何解决ABA问题？
1
对于ABA问题的常见解决思路即生成一个唯一可表示记录信息的标记值。例如我们可以新增一个自增字段，每次操作这个字段后该值加1，写写入数据之前比较该值是否与进入该方法时读取到的值相同。初次之外还可以记录版本号和时间戳(思路大同小异)。

对于自旋带来的CPU资源浪费问题

根据上面的分析我们知道，在CAS写入的过程当中，如果写入失败并不会挂起线程，而是会自旋并继续重试。在某些极端场景下，这可能会死循环或者造成CPU资源的白白浪费。在我们平时的编码过程当中，我们其实也可以考虑jdk1.6之后对synchronized进行锁升级的思路。自旋到一定次数还无法或者资源时，我们可以考虑放弃该任务返回null值或主动升级成重量级锁。


## 独享锁 & 共享锁
两种锁只是一种概念

独享锁：该锁一次只能被一个线程所持有

共享锁：该锁可以被多个线程所持有

举例：

synchronized是独享锁；

可重入锁ReentrantLock是独享锁；

读写锁ReentrantReadWriteLock中的读锁ReadLock是共享锁，写锁WriteLock是独享锁。

独享锁与共享锁通过AQS(AbstractQueuedSynchronizer)来实现的，通过实现不同的方法，来实现独享或者共享。

## 互斥锁 & 读写锁
上面讲的独享锁/共享锁就是一种概念，互斥锁/读写锁是具体的实现。

互斥锁的具体实现就是synchronized、ReentrantLock。ReentrantLock是JDK1.5的新特性，采用ReentrantLock可以完全替代替换synchronized传统的锁机制，更加灵活。

读写锁的具体实现就是读写锁ReadWriteLock。



## 读写锁

只有两方都是读的时候不会出问题，不用加锁。否则只要有一方写，就不安全。

读写锁要在 finally 中释放锁。

```
val readWriterLock = ReentrantReadWriteLock()
val readLock = readWriterLock.readLock()
val writeLock = readWriterLock.writeLock()

var x = 0

fun testRead() {
    readLock.lock()
    try {
        println(x)
    } finally {
        readLock.unlock()
    }
}

fun testWrite() {
    writeLock.lock()
    try {
        x++
    } finally {
        writeLock.unlock()
    }
}
```

## 可重入锁
定义：对于同一个线程在外层方法获取锁的时候，在进入内层方法时也会自动获取锁。

优点：避免死锁

举例：ReentrantLock、synchronized

## 公平锁 & 非公平锁
公平锁：多个线程相互竞争时要排队，多个线程按照申请锁的顺序来获取锁。

非公平锁：多个线程相互竞争时，先尝试插队，插队失败再排队，比如：synchronized、ReentrantLock

## 分段锁
分段锁并不是具体的一种锁，只是一种锁的设计。

分段锁的设计目的是细化锁的粒度，当操作不需要更新整个数组的时候，就仅仅针对数组中的一项进行加锁操作。CurrentHashMap底层就用了分段锁，使用Segment，就可以进行并发使用了，而HashMap确实非线程安全的，就差在了分段锁上。

## 偏向锁 & 轻量级锁 & 重量级锁
JDK 1.6 为了减少获得锁和释放锁所带来的性能消耗，在JDK 1.6里引入了4种锁的状态：无锁、偏向锁、轻量级锁和重量级锁，它会随着多线程的竞争情况逐渐升级，但不能降级。

研究发现大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，为了不让这个线程每次获得锁都需要CAS操作的性能消耗，就引入了偏向锁。当一个线程访问对象并获取锁时，会在对象头里存储锁偏向的这个线程的ID，以后该线程再访问该对象时只需判断对象头的Mark Word里是否有这个线程的ID，如果有就不需要进行CAS操作，这就是偏向锁。当线程竞争更激烈时，偏向锁就会升级为轻量级锁，轻量级锁认为虽然竞争是存在的，但是理想情况下竞争的程度很低，通过自旋方式等待一会儿上一个线程就会释放锁，但是当自旋超过了一定次数，或者一个线程持有锁，一个线程在自旋，又来了第三个线程访问时（反正就是竞争继续加大了），轻量级锁就会膨胀为重量级锁，重量级锁就是Synchronized,重量级锁会使除了此时拥有锁的线程以外的线程都阻塞。

# 协程

Kotlin 协程由于要和 Java 互操作和运行在 Jvm 之上，其本质上还是对线程的一个封装，底层是靠线程实现的。

## 挂起

挂起是挂起整个携程，挂起点之后的代码不再执行，而是转而执行其他携程。

## 挂起函数

并不是执行到这个函数就挂起了，而是函数内部可能包含有能够使携程挂起的代码，在函数内的代码执行到挂起代码的时候，携程被挂起。被 `suspend` 修饰的函数不应定有挂起代码，也不定会时携程挂起。由于挂起只能出现在携程内部，该修饰符就是为了给编译器标识函数万一有挂起函数，只能用于携程内部，用于安全检查的。


## 创建

1. luanch
2. sync
3. runBlocking

调度器

1. Dispatchers.Unconfined
2. Dispatchers.IO 对 IO 操作做了优化
3. Dispatchers.Default 适用于 CPU 密集型操作
5. Dispatchers.Main  安卓库独有

> 协称实现为什么也需要栈?

直观感觉上，单线程的协称只是从程序的一个点 jump 到另一个点执行，是可以使用同一个栈的。然而每个协称却需要单独的栈，这是为什么呢？

几乎所有的编程语言都是以函数（方法）组织代码片段的，而函数的调用是以栈作为保存局部变量的，栈成为程序流程必不可少的上下文。假如一个线程中有两个协称：

协程 1：funA ---> 挂起 ---> funB
协程 2：funX ---> funY ---> 挂起

协称 1 执行时，funA 先入栈。当协称 1 调用`挂起函数`时，调度将执行协称 2，此时 funX、funY 入栈。协称2 挂起返回协称 1 执行时，如果两个协称公用相同的栈，funB 入栈时，将覆盖 funX、funY 的局部变量。从而再次切换回协程 2 执行时，因为 funX、FunY 的栈被覆盖，函数出栈将出错。

这里有篇文章将协称分为[有栈、无栈协程](https://blog.csdn.net/weixin_39875941/article/details/110592519)。我看不过是栈是由系统在栈区分配，还是由用户在堆分配空间作为栈使用的区别。因为**函数调用的入栈规则不改变，就不可能没有栈。**


# Atomic 类

https://www.jianshu.com/p/84c75074fa03

## 线程多少合适？

如果是 CPU 密集型，线程数量等于 CPU 核数能达到最大的计算效率。例如 Retrofit 的 Scheulers.computation
根据是非 CUP 密集型，例如网络传输，可以任务多个。例如 Retrofit 的 Scheulers.io。
