[TOC]

# 线程

## 线程上下文

一个整数线程 ID
栈
栈指针
程序计数器
通用目的的寄存器和条件码？

与同一进程中的其它线程共享整个进程的虚拟地址空间
包括代码、数据区域、堆、共享库、和打开的文件。

线程的上下文要比进程的上下文小得多，线程上线文切换要比进程上下文切换快的多。
不同于进程的严格父子层次关系组织，一个进程内的线程组成一个线程对等池。对等影响是，一个线程可以杀死任意的线程，或者等待其结束。每个对等线程都可以读写共享的数据。


**寄存器是从不共享的，而虚拟存储器总是共享的**





## Thread Executor, HandlerThread, AsyncTask 怎么选

如果想要执行后台任务，不与前台交互，可以使用, Thread　或者　Executor，而且推荐使用　Executor 线程池。

如果与前台频繁交互，例如下载进度条等则可以使用 AsyncTask，而　HandlerThread 基本没有什么使用场景。如果一次性交互，可以使用 Handler 或者其他切换线程的库。

Service 和　IntentService（带有单线程的后台一次性 Service，执行完后就自动退出了）。Servide 主要处理长时间的任务，除非用户为其他应用提供服务或者播放器等用户服务。否则都不建议使用，而是使用 DownloadManager 或者 TaskManager 代替。


## 实现 Rannable 接口的好处

1. 线程的任务从线程子类中分离处理，进行单独的封装，更符合面向对象的思想。
2. 避免 Java 单继承的局限性。


## 几种创建 Thread 的方法

1. 继承 Thread 类，重写　run 方法。
2. 实现 Runnable 接口。传递给　Thread 对象。同时覆盖和传递 runnable 将执行 Thread 自身覆盖的 `run` 方法。
3. ThreadFactory，其实内部还是自己 New Thread。只不过可以用与生产一批类似的线程。
4. Excturor 线程池，不用时及时关闭 shutdown。
    1. 单线程的线程池　singleThreadExecutor()。可以指定执行的顺序（FIFO,LIFO）/
    2. 固定数量线程池 fixedThreadExecutor()
    3. 可以动态增长的线程池 newCachePoolExecutor() 不限制数量
    4. newScheduledThreadPool 支持定时及周期执行任务。
5. Callable 有返回值的线程，通过 `Future.get()` 获取结果

## 进程间通信

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
A thread in the blocked state is waiting for a monitor lock　to enter a synchronized block/method or　reenter a synchronized block/method after calling　｀Object.wait｀
```

## 线程的状态图

![Thread status transform](images/thread_status_transform.png)



## interrupt 和 stop 的区别，为什么 stop 被废弃。

- interrupt 是在线程中设置了一个标志位，需要在 `run` 方法中自己判断标志位来终止。`Thread.isInterrupe()` 会在返回之后，把标志位置为 true，这样方便下次再次执行；而 `isInterrupt()` 不会设置标志位。
- stop　方法类似将线程　kill 掉，结果不可预期。释放它已经锁定的所有监视器。可能产生数据的不一致性。已经被废弃。

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



## wait 和　sleep　的区别

1. wait 可以制定时间，也可以不同指定。
2. 在同步中时，对　cup 的执行权的实例方式不同。wait 释放执行权，释放锁。sleep 释放执行权，但不释放锁。

## 线程安全

> 问题产生的原因

1. 多个线程在同时操作共享的数据
2. 操作共享数据的代码有多条

当一个线程执行操作共享数据的多条代码时，由于线程切换是随机的，会导致执行过程中，数据被其他线程更改。这样使数据产生预期之外的值。

为了解决这个问题，需要线程同步，即对同一个数据的一块操作必须整体要么执行完，要么不执行。不能被其它线程打断。实现操作的原子性。

使用 synchronized 或者　Lock 实现线程同步。

## 线程同步

即多个线程对共享数据的操作是顺序执行的，不能在一个线程执行过程中被其它线程打断。是为了解决线程安全提出的方案。多个线程执行的异步的，但对于共享数据的操作是线程同步的。

- 好处：解决了线程安全的问题

- 弊端：降低了线程的执行效率，因为同步外的线程都会判断锁，所得获取和释放都需要额外的操作。

实现同步的前提是，要实现同步的所有线程使用同一个锁。






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

Condation 将 Object 监视方法（wait(),notify,notifyAll) 分解为截然不同的对象，以便将这些对象与任意 lock 实现组合，为每个对象提供多个等待的　set, 其中 Lock 代替了原有 synchronized 关键字的使用。condation　代替了 Ojeect 监视器方法的使用。

方法名的改变

wait　－> await
notify  －> signal
notifyAll －> signalAll


## 死锁

1. A 线程持有资源 x, 要获取 y。Ｂ 线程持有资源 y，要获取 x。

它们一定是双锁，同时产生了嵌套。一个获取了锁 a，代码块内部想要获取锁 b. 另一个线程获取了锁 b，内部同步代码执行时想要获取锁 a。由于是嵌套的，不获取的时候还释放不了 a。

## 单例模式的问题

单例的变量要使用 volatile 声明变量，保证基本数据类型的操作是线程同步的。例如虚拟机上 double 的赋值可能会被分成两步。防止变量还没初始化完成就返回引用供其他程序访问而出现　null 的情况。

volatile 不能保证对象，++,-- 等操作的线程安全。

对于`++`,`--` 这些操作，为了线程安全，如果使用 synchronized 则太重了。这时候可以使用带 `Atomic` 前缀的基本数据类型，它们的操作是原子性的。

1. 恶汉模式不存在线程安全。
2.


## 乐观锁和悲观锁


乐观锁是读数据的时候先不锁数据，假设别人不会修改数据，等到写入的时候再检查是否有修改，有修改再加锁，然后写入。

悲观锁是觉得别人会修改数据，在读取的时候就先加锁，写入完毕的时候再释放锁。


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


## volatile

- 可见性：

　　可见性是一种复杂的属性，因为可见性中的错误总是会违背我们的直觉。通常，我们无法确保执行读操作的线程能适时地看到其他线程写入的值，有时甚至是根本不可能的事情。为了确保多个线程之间对内存写入操作的可见性，必须使用同步机制。

　　可见性，是指线程之间的可见性，一个线程修改的状态对另一个线程是可见的。也就是一个线程修改的结果。另一个线程马上就能看到。比如：用volatile修饰的变量，就会具有可见性。当把变量声明为volatile类型后，编译器与运行时都会注意到这个变量是共享的，因此不会将该变量上的操作与其他内存操作一起重排序。volatile变量不会被缓存在寄存器或者对其他处理器不可见的地方，因此在读取volatile类型的变量时总会返回最新写入的值。，即直接修改内存。所以对其他线程是可见的。但是这里需要注意一个问题，volatile只能让被他修饰内容具有可见性，但不能保证它具有原子性。比如 volatile int a = 0；之后有一个操作 a++；这个变量a具有可见性，但是a++ 依然是一个非原子操作，也就是这个操作同样存在线程安全问题。

volatile变量的内存可见性是基于内存屏障(Memory Barrier)实现的，什么是内存屏障?内存屏障，又称内存栅栏，是一个CPU指令。在程序运行时，为了提高执行性能，编译器和处理器会对指令进行重排序，通过插入特定类型的内存屏障来禁止特定类型的编译器重排序和处理器重排序，插入一条内存屏障会告诉编译器和CPU：不管什么指令都不能和这条Memory Barrier指令重排序。

https://www.jianshu.com/p/08a0a8c984ab

　　在 Java 中 volatile、synchronized 和 final 实现可见性。

- 原子性：

　　原子是世界上的最小单位，具有不可分割性。比如 a=0；（a非long和double类型） 这个操作是不可分割的，那么我们说这个操作时原子操作。再比如：a++； 这个操作实际是a = a + 1；是可分割的，所以他不是一个原子操作。非原子操作都会存在线程安全问题，需要我们使用同步技术（sychronized）来让它变成一个原子操作。一个操作是原子操作，那么我们称它具有原子性。java的concurrent包下提供了一些原子类，我们可以通过阅读API来了解这些原子类的用法。比如：AtomicInteger、AtomicLong、AtomicReference等。

　　在 Java 中 synchronized 和在 lock、unlock 中操作保证原子性。

- 有序性：

　　Java 语言提供了 volatile 和 synchronized 两个关键字来保证线程之间操作的有序性，volatile 是因为其本身包含“禁止指令重排序”的语义，synchronized 是由“一个变量在同一个时刻只允许一条线程对其进行 lock 操作”这条规则获得的，此规则决定了持有同一个对象锁的两个同步块只能串行执行。


> 当一个变量定义为 volatile 之后，将具备两种特性：

　　1.保证此变量对所有的线程的可见性，这里的“可见性”，当一个线程修改了这个变量的值，volatile 保证了新值能立即同步到主内存，以及每次使用前立即从主内存刷新。但普通变量做不到这点，普通变量的值在线程间传递均需要通过主内存（详见：Java内存模型）来完成。

　　2.禁止指令重排序优化。有volatile修饰的变量，赋值后多执行了一个“load addl $0x0, (%esp)”操作，这个操作相当于一个内存屏障（指令重排序时不能把后面的指令重排序到内存屏障之前的位置），只有一个CPU访问内存时，并不需要内存屏障；（什么是指令重排序：是指CPU采用了允许将多条指令不按程序规定的顺序分开发送给各相应电路单元处理）。

volatile 性能：
　　volatile 的读性能消耗与普通变量几乎相同，但是写操作稍慢，因为它需要在本地代码中插入许多内存屏障指令来保证处理器不发生乱序执行。


## Atomic 类

https://www.jianshu.com/p/84c75074fa03
