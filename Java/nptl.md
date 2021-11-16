# NPTL 锁实现（glibc-2.34版本）

## 1. 锁类型

```C
/* Mutex types.  */
enum
{
  PTHREAD_MUTEX_TIMED_NP, // 普通互斥锁，首先进行一次CAS，如果失败则陷入内核态然后挂起线程
  PTHREAD_MUTEX_RECURSIVE_NP, // 可重入锁，允许同一个线程对同一个锁成功获得多次，
  PTHREAD_MUTEX_ERRORCHECK_NP, // 检错锁，如果同一个线程请求同一个锁，则返回EDEADLK，否则与PTHREAD_MUTEX_TIMED_NP类型动作相同。这样就保证当不允许多次加锁时不会出现最简单情况下的死锁。

  PTHREAD_MUTEX_ADAPTIVE_NP // 适应锁，此锁在多核处理器下首先进行自旋获取锁，如果自旋次数超过配置的最大次数，则也会陷入内核态挂起。

#if defined __USE_UNIX98 || defined __USE_XOPEN2K8
  ...
#endif
#ifdef __USE_GNU
  ...
#endif
};
```

根据锁的类型，代码比较多，仅看普通的互斥锁  `PTHREAD_MUTEX_TIMED_NP`。具体的代码在 `nptl/pthread_mutex_lock.c`

## 2. 互斥锁

```c
int
PTHREAD_MUTEX_LOCK (pthread_mutex_t *mutex)
{
  ...
  if (__glibc_likely (type == PTHREAD_MUTEX_TIMED_NP))
    {
      FORCE_ELISION (mutex, goto elision);
    simple:
      /* Normal mutex.  */
      LLL_MUTEX_LOCK_OPTIMIZED (mutex);
      assert (mutex->__data.__owner == 0);
    }
#if ENABLE_ELISION_SUPPORT
  else if (__glibc_likely (type == PTHREAD_MUTEX_TIMED_ELISION_NP))
  {
  ....
	}
}

# define LLL_MUTEX_LOCK_OPTIMIZED(mutex) lll_mutex_lock_optimized (mutex)


static inline void
lll_mutex_lock_optimized (pthread_mutex_t *mutex)
{
  /* 针对单线程进程的优化(Linux 进程可以设置为单线程)，如果进程仅有一个线程，并且没有在进程见共享，则只需要设置标志位。
  如果线程在进程间共享，即使只有一个线程，也需要同步锁。
    单线程时，如果锁是锁定状态，因为普通锁不允许重入，跳过优化，仍然死锁。
    The single-threaded optimization is only valid for private
     mutexes.  For process-shared mutexes, the mutex could be in a
     shared mapping, so synchronization with another process is needed
     even without any threads.  If the lock is already marked as
     acquired, POSIX requires that pthread_mutex_lock deadlocks for
     normal mutexes, so skip the optimization in that case as
     well.  */
  // 线程仅在当前进程使用。
  int private = PTHREAD_MUTEX_PSHARED (mutex);
  if (private == LLL_PRIVATE && SINGLE_THREAD_P && mutex->__data.__lock == 0)
    mutex->__data.__lock = 1;
  else
    // 执行正常的加锁
    lll_lock (mutex->__data.__lock, private);
}
```

### likely 和 unlikely

> 预留标识符

**标准规定单下划线加大写字母和双下划线开头的标识符都是预留给实现/扩展的，标准库实现使用这些标识符，以避免和用户定义的宏撞上导致冲突。可移植的用户代码不该直接使用它们，也不该自行定义这种标识符。**


```c
Linux kernel
# define likely(x)	__builtin_expect(!!(x), 1)
# define unlikely(x)	__builtin_expect(!!(x), 0)

glibc
# define __glibc_likely(cond)	__builtin_expect ((cond), 1)
# define __glibc_unlikely(cond)	__builtin_expect ((cond), 0)
```

猜测 `!!` 是将数字转为 0 或者 1

使用了gcc (version >= 2.96）的内建函数 `__builtin_expect()`。 该函数用来引导 gcc 进行条件分支预测。在一条指令执行时，由于流水线的作用，CPU 可以同时完成下一条指令的取指，这样可以提高CPU的利用率。在执行条件分支指令时，CPU也会预取下一条执行，但是如果条件分支的结果为跳转到了其他指令，那CPU预取的下一条指令就没用了，这样就降低了流水线的效率。

另外，跳转指令相对于顺序执行的指令会多消耗CPU时间，如果可以尽可能不执行跳转，也可以提高CPU性能。

使用__builtin_expect (long exp, long c) 函数可以帮助 gcc 优化程序编译后的指令序列，使汇编指令尽可能的顺序执行，从而提高CPU预取指令的正确率和执行效率。

_builtin_expect(exp, c)接受两个long型的参数，用来告诉gcc：exp==c的可能性比较大。例如，__builtin_expect(exp, 1) 表示程序执行过程中，exp取到1的可能性比较大。该函数的返回值为exp自身。

用作 if 的条件时，由于 `condation` 非 0 执行 if 中的内容，为 0 执行 else 的内容，因此，likely 会将 if 内容排在前面，而 unlikely 会将 else 的内容排在前面，以优化效率。



## 3. lll_lock

```c
// sysdeps/nptl/lovellock.c

/* This is an expression rather than a statement even though its value is
   void, so that it can be used in a comma expression or as an expression
   that's cast to void.  */
/* The inner conditional compiles to a call to __lll_lock_wait_private if
   private is known at compile time to be LLL_PRIVATE, and to a call to
   __lll_lock_wait otherwise.  */
/* If FUTEX is 0 (not acquired), set to 1 (acquired with no waiters) and
   return.  Otherwise, ensure that it is >1 (acquired, possibly with waiters)
   and then block until we acquire the lock, at which point FUTEX will still be
   >1.  The lock is always acquired on return.  */
#define __lll_lock(futex, private)                                      \
  ((void)                                                               \
   ({                                                                   \
     int *__futex = (futex);                                            \
     if (__glibc_unlikely                                               \
         (atomic_compare_and_exchange_bool_acq (__futex, 1, 0)))        \
       {                                                                \
         if (__builtin_constant_p (private) && (private) == LLL_PRIVATE) \
           __lll_lock_wait_private (__futex);                           \
         else                                                           \
           __lll_lock_wait (__futex, private);                          \
       }                                                                \
   }))
#define lll_lock(futex, private)	\
  __lll_lock (&(futex), private)
```

```
#define atomic_compare_and_exchange_bool_acq(mem, newval, oldval) \
  (! __sync_bool_compare_and_swap (mem, oldval, newval))
```

## 4. atomic_compare_and_exchange_bool_acq

对于 x86

```c
// sysdeps/x86/atomic-machine.h

#define atomic_compare_and_exchange_bool_acq(mem, newval, oldval) \
  (! __sync_bool_compare_and_swap (mem, oldval, newval))


```

X86 平台比较简单，直接执行 [__sync_bool_compare_and_swap](https://gcc.gnu.org/onlinedocs/gcc/_005f_005fsync-Builtins.html) 的编译器内置函数，完成 CAS 原子操作。关于 CAS 原子操作，请看下文。


对于 arm

```c
// sysdeps/aarch64/atomic-machine.h

/* Compare and exchange with "acquire" semantics, ie barrier after.  */

# define atomic_compare_and_exchange_bool_acq(mem, new, old)	\
  __atomic_bool_bysize (__arch_compare_and_exchange_bool, int,	\
			mem, new, old, __ATOMIC_ACQUIRE)
```

```c
// include/atomic.h

#define __atomic_bool_bysize(pre, post, mem, ...)			      \
  ({									      \
    int __atg2_result;							      \
    if (sizeof (*mem) == 1)						      \
      __atg2_result = pre##_8_##post (mem, __VA_ARGS__);		      \
    else if (sizeof (*mem) == 2)					      \
      __atg2_result = pre##_16_##post (mem, __VA_ARGS__);		      \
    else if (sizeof (*mem) == 4)					      \
      __atg2_result = pre##_32_##post (mem, __VA_ARGS__);		      \
    else if (sizeof (*mem) == 8)					      \
      __atg2_result = pre##_64_##post (mem, __VA_ARGS__);		      \
    else								      \
      abort ();								      \
    __atg2_result;							      \
  })
```

```
// sysdeps/aarch64/atomic-machine.h

# define __arch_compare_and_exchange_bool_32_int(mem, newval, oldval, model) \
  ({									\
    typeof (*mem) __oldval = (oldval);					\
    !__atomic_compare_exchange_n (mem, (void *) &__oldval, newval, 0,	\
				  model, __ATOMIC_RELAXED);		\
  })
```

ARM 的调用流程比较长，最后还是调用了 [`__atomic_compare_exchange_n`](https://gcc.gnu.org/onlinedocs/gcc/_005f_005fatomic-Builtins.html) 编译器内置函数。[查看 Gcc 内置函数文档](https://gcc.gnu.org/onlinedocs/gcc/_005f_005fatomic-Builtins.html)以及 [Gcc Atomic model](https://gcc.gnu.org/wiki/Atomic/GCCMM/AtomicSync)。


### CAS

无论 `__sync_bool_compare_and_swap` 还是 `__atomic_compare_exchange_n` 都是被称为 `CAS` 的原子操作。因为获取锁的代码也是进阶区代码，也要实现原子操作，此时的原子操作单靠软件无法实现，需要硬件支持。具体实现根据不同平台，甚至同一平台的不同版本实现也不一样。例如：

在现代 x86 处理骑上，基本算数运算、逻辑运算指令前添加 `LOCK` (80486)即可使用使用原子操作。比如x86处理器以及[ARMv8.1架构等处理器直接提供了CAS指令作为原子条件原语](https://blog.csdn.net/Roland_Sun/article/details/107552574)。而ARMv7、ARMv8处理器则使用了另一种LL-SC，这两种原子条件原语都可以作为Lock-free算法工具。

比 CAS 与 LL_SC 更早一些一些的原子条件有 SWAP(8086, ARMv5), Bit test adn set(80386. Blackfin SDP561) 等，这些同步原语只能用于 `同步锁`，而无法作为 `lock-free` 的原子对象进行操作。

没有搜到 `__sync_bool_compare_and_swap` 和 `__atomic_compare_exchange_n` 具体实现，但是可以自己写一个测试，然后编译，看一下具体的汇编代码。

For x86
```

```

For Arm
```c
// cas_arm.c
void test_cas() {
    int lock; // lock 假如不确定，是上一次的值。
    int old = 0;
    // lock 和 old 比较，相等，则写入 1，否则将 lock 值写入 old.
    // lock 写入新值（即等于 old）返回 true，否则返回 false.
    __atomic_compare_exchange_n(&lock, &old, 1 /* new value */, 0, __ATOMIC_ACQUIRE, __ATOMIC_RELAXED);
}
```

[ARMv8.1 使用 CASA 原子操作](https://developer.arm.com/documentation/dui0801/g/A64-Data-Transfer-Instructions/CASA--CASAL--CAS--CASL--CASAL--CAS--CASL)

```bash
$NDK_HOME/toolchains/llvm/prebuilt/darwin-x86_64/bin/clang \
-target aarch64-linux-android21 \
-march=armv8.1-a \
-S cas_arm.c \
-o cas_arm.s
```

```ARM
...
test_cas:                               // @test_cas
	.cfi_startproc
// %bb.0:
	sub	sp, sp, #16                     // =16
	.cfi_def_cfa_offset 16
	mov	w8, wzr                         // v8 = 0
	add	x9, sp, #12                     // x9 = &lock
	mov	w10, #1                         // w10 = 1, new value
	casa	w8, w10, [x9]                 // v8 == [x9] 即 0 == lock，先将[x9] 旧值保存到 w8, 然后将 w10 存入 [x9]，即将 1 写入 lock
  cmp	w8, #0                          // 比较 w8 的旧值是否跟 old 相等
  cset w0, eq                         // if (cond == true) W0 = 1, else W0 = 0 相等返回 1，否则返回 0。
	add	sp, sp, #16                     // =16
	ret
```

ARMv8.1 之前

|  位宽   |  获取  |  存贮  |
|------- | ------ | ------|
  64 位  |  ldaxr | stxr
  32 位  |  ldrex | strex



```ARM
test_cas:                               // @test_cas
	.cfi_startproc
// %bb.0:
	sub	sp, sp, #16                     // =16
	.cfi_def_cfa_offset 16
	add	x8, sp, #12                     // x8 = &lock
	mov	w9, #1                          // w9 = 1
.LBB0_1:                                // =>This Inner Loop Header: Depth=1
	ldaxr	w10, [x8]                   // 加载 lock 的值到 w10 中
	cbnz	w10, .LBB0_4                // w10 != 0 直接跳到 .LBB0_4:
// %bb.2:                               //   in Loop: Header=BB0_1 Depth=1
	stxr	w10, w9, [x8]               // 将 w9 的内容写入 [x8]，即 lock. 如果 ldaxr 设置的状态被改变了，则更新失败。
	cbnz	w10, .LBB0_1                // w10 保存了是否更新成功的状态，0 为成功，失败则跳到 .LBB0_1 重试
// %bb.3:
	add	sp, sp, #16                     // =16
	ret
.LBB0_4:
	clrex                               // 清除内存监测
	add	sp, sp, #16                     // =16
	ret
```

`ldaxr <Rt>, [<Rn>]`

ldaxr 将 [Rn] 的内容加载到 Rt (destination register)中。这些操作和ldr 的操作是一样的，那么如何体现exclusive呢？其实，在执行这条指令的时候，还放出两条“狗”来负责观察特定地址的访问（就是保存在 Rn 中的地址了），这两条狗一条叫做local monitor，一条叫做global monitor。

stxr <Rd>, <Rt>, [<Rn>]

和LDREX指令类似，<Rn>是base register，保存memory的address，
stxr 将 Rt (source register)中的内容加载到该 [Rn] 的内存中。Rd 保存了memeory 更新成功或者失败的结果，0表示更新成功，1表示失败。stxr 指令是否能成功执行是和 `local monitor` 和 `global monitor` 的状态相关的。对于 Non-shareable memory（该memory不是多个CPU之间共享的，只会被一个CPU访问），只需要放出该CPU的 local monitor这条狗就OK了。

以 Local monitor 为例，开始的时候，local monitor 处于 Open Access state的状态，thread 1 执行 ldaxr 命令后，local monitor 的状态迁移到 Exclusive Access state（标记本地CPU对xxx地址进行了 ldaxr 的操作），这时候，中断发生了，在中断 handler 中，又一次执行了 ldaxr ，这时候，local monitor 的状态保持不变，直到 stxr 指令成功执行，local monitor 的状态迁移到 Open Access state 的状态（清除xxx地址上的 ldaxr 的标记）。返回thread 1的时候，在Open Access state的状态下，执行 stxr 指令会导致该指令执行失败（没有 lsaxr 的标记），说明有其他的内核控制路径插入了。

对于 shareable memory，需要系统中所有的local monitor和global monitor共同工作，完成exclusive access，概念类似。


### 简化 CAS

通过硬件的支持，保证对内存变量的原子操作。这些操作仍然是一个复杂的流程，为了简化，将这些操作可以简化为一个 CAS 函数。

```
bool CAS(T* val, T new_value, T old_value) {
  if (*val == old_value) {
    *val = new_value;
    return true;
  } else {
    return false;
  }
}
```




## 评估时间

The time command in Linux is used to determine the duration of execution of a command. This command is useful when you want to know the exection time of a command or a script. By default, three times are displayed:

real time – the total execution time. This is the time elapsed between invocation and termination of the command.
user CPU time – the CPU time used by your process.
system CPU time – the CPU time used by the system on behalf of your process.

### mac查看自己cpu的具体型号、核数、线程数

终端输入，查看cpu具体型号：

```
sysctl machdep.cpu.brand_string
```
查看cpu核心数：
```
sysctl -n machdep.cpu.core_count
```
查看cpu线程数：
```
sysctl -n machdep.cpu.thread_count
```

### Linux 查看 CPU 信息

cat /proc/cpuinfo

## 遗留

内核互斥锁，内核锁是 Linux 内核实现的锁，和 NPTL 锁实现不一样。

https://blog.csdn.net/arm7star/article/details/77108301
