# NPTL 锁实现（glibc-2.34版本）

Futex 是fast userspace mutex的缩写，意思是快速用户空间互斥体。Linux内核把它们作为快速的用户空间的锁和信号量的预制构件提供给开发者。Futex非常基础，借助其自身的优异性能，构建更高级别的锁的抽象，如POSIX互斥体。大多数程序员并不需要直接使用Futex，它一般用来实现像NPTL这样的系统库。

Futex 由一块能够被多个进程共享的内存空间(一个对齐后的整型变量)组成；这个整型变量的值能够通过汇编语言调用CPU提供的原子操作指令来增加或减少，并且一个进程可以等待直到那个值变成正数。Futex 的操作几乎全部在应用程序空间完成；只有当操作结果不一致从而需要仲裁时，才需要进入操作系统内核空间执行。这种机制允许使用 futex 的锁定原语有非常高的执行效率：由于绝大多数的操作并不需要在多个进程之间进行仲裁，所以绝大多数操作都可以在应用程序空间执行，而不需要使用(相对高代价的)内核系统调用。

futex保存在用户空间的共享内存中，并且通过原子操作进行操作。在大部分情况下，资源不存在争用的情况下，进程或者线程可以立刻获得资源成功，实际上就没有必要调用系统调用，陷入内核了。实际上，futex的作用就在于减少系统调用的次数，来提高系统的性能。



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

数据结构，简化后的版本

```C
// 64
struct __pthread_mutex_s {
  int __lock __LOCK_ALIGNMENT;
  unsigned int __count;
  int __owner;
  unsigned int __nusers;
  int __kind;
  int __spins;
  __pthread_list_t __list;
  # define __PTHREAD_MUTEX_HAVE_PREV      1
}

// 32 位
struct __pthread_mutex_s
{
  int __lock __LOCK_ALIGNMENT;
  unsigned int __count;
  int __owner;
  int __kind;
  unsigned int __nusers;
  __extension__ union
  {
    int __spins;
    __pthread_slist_t __list;
  };
  # define __PTHREAD_MUTEX_HAVE_PREV      0
};
```

// 为了理解方便，对定义原型有较大改动，在定义中有四个比较重要的成员，以下对其分别进行介绍：

lock表示当前mutex的状态，0表示初始化没有被持有的状态，此时可以对mutex执行lock操作，lock为1时表示当前mutex已经被持有，并且没有其他线程在等待它的释放，当lock > 1时，表示mutex被某个线程持有并且有另外的线程在等待它的释放。

count表示当前被持有的次数，一般来说对不可重入的锁，这个值只可能是0和1，对于可重入的锁，比如递归锁，这个值会大于1。

owner用来记录持有当前mutex的线程id，如果没有线程持有，这个值为0。

nusers用来记录当前有多少线程持有该互斥体，一般来说，这个值只能是0和1，但是对于读写锁来说，多个读线程是可以共同持有mutex的，因此用nusers来记录线程的数量。

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
     well. */
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
// glibc-2.34/sysdeps/nptl/lowlevellock.h

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
// cas_x86.c
int test_cas() {
    int x = 1;
    return __sync_bool_compare_and_swap(&x, 1, 0);
}
```
编译指令，`-S` 参数表示仅执行到汇编，而不生成可执行程序。`-masm=intel` 表示生成 Intel 格式的汇编，如果对 gas 汇编比较熟悉，可以去掉该参数。
```bash
clang -masm=intel -S cas_x86.c -o cas_x86.s
```

```ARM
_test_cas:                              ## @test_cas
	.cfi_startproc
## %bb.0:
	push	rbp
	.cfi_def_cfa_offset 16
	.cfi_offset rbp, -16
	mov	rbp, rsp
	.cfi_def_cfa_register rbp
	mov	dword ptr [rbp - 4], 1
	xor	edx, edx
	xor	ecx, ecx
	mov	eax, 1
	lock		cmpxchg	dword ptr [rbp - 4], edx  ## 执行 CAS 原子操作。
	sete	cl
	mov	eax, ecx
	pop	rbp
	ret
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

[ARMv8.1 使用 CASA 原子操作](https://developer.arm.com/documentation/dui0801/g/A64-Data-Transfer-Instructions/CASA--CASAL--CAS--CASL--CASAL--CAS--CASL)。 编译验证。

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

```c
/**
 * val 和 old_value 相同，将 val 更新为 new_value 返回 true.
 * val 和 old_value 不相同，返回 false
 */
bool CAS(T* val, T new_value, T old_value) {
  if (*val == old_value) {
    *val = new_value;
    return true;
  } else {
    return false;
  }
}
```

### TAS

```C
// TestAndSet
int TAS(int *ptr, int new) {
  int old = *ptr; // fetch old value at ptr
  *ptr = new; // store ’new’ into ptr
  return old; // return the old value
}
```

### 获取不到锁，调用 __lll_lock_wait 休眠

```C
// glibc-2.34/nptl/lowlevellock.c
void
__lll_lock_wait (int *futex, int private)
{
  if (atomic_load_relaxed (futex) == 2)
    goto futex;

  // 在获取锁失败后，会将互斥量设置为2，然后进行系统调用进行挂起，这是为了让解锁线程发现有其它等待互斥量的线程需要被唤醒
  while (atomic_exchange_acquire (futex, 2) != 0)
    {
    futex:
      LIBC_PROBE (lll_lock_wait, 1, futex);
      futex_wait ((unsigned int *) futex, 2, private); /* Wait if *futex == 2.  */
    }
}

#  define atomic_load_relaxed(mem) \
   ({ __typeof ((__typeof (*(mem))) *(mem)) __atg100_val;		      \
   __asm ("" : "=r" (__atg100_val) : "0" (*(mem)));			      \
   __atg100_val; })

// glibc-2.34/sysdeps/nptl/futex-internal.h

/* Atomically wrt other futex operations on the same futex, this blocks iff
   the value *FUTEX_WORD matches the expected value.  This is
   semantically equivalent to:
     l = <get lock associated with futex> (FUTEX_WORD);
     wait_flag = <get wait_flag associated with futex> (FUTEX_WORD);
     lock (l);
     val = atomic_load_relaxed (FUTEX_WORD);
     if (val != expected) { unlock (l); return EAGAIN; }
     atomic_store_relaxed (wait_flag, true);
     unlock (l);
     // Now block; can time out in futex_time_wait (see below)
     while (atomic_load_relaxed(wait_flag) && !<spurious wake-up>);

   Note that no guarantee of a happens-before relation between a woken
   futex_wait and a futex_wake is documented; however, this does not matter
   in practice because we have to consider spurious wake-ups (see below),
   and thus would not be able to reliably reason about which futex_wake woke
   us.

   Returns 0 if woken by a futex operation or spuriously.  (Note that due to
   the POSIX requirements mentioned above, we need to conservatively assume
   that unrelated futex_wake operations could wake this futex; it is easiest
   to just be prepared for spurious wake-ups.)
   Returns EAGAIN if the futex word did not match the expected value.
   Returns EINTR if waiting was interrupted by a signal.

   Note that some previous code in glibc assumed the underlying futex
   operation (e.g., syscall) to start with or include the equivalent of a
   seq_cst fence; this allows one to avoid an explicit seq_cst fence before
   a futex_wait call when synchronizing similar to Dekker synchronization.
   However, we make no such guarantee here.  */
static __always_inline int
futex_wait (unsigned int *futex_word, unsigned int expected, int private)
{
  int err = lll_futex_timed_wait (futex_word, expected, NULL, private);
  switch (err)
    {
    case 0:
    case -EAGAIN:
    case -EINTR:
      return -err;

    case -ETIMEDOUT: /* Cannot have happened as we provided no timeout.  */
    case -EFAULT: /* Must have been caused by a glibc or application bug.  */
    case -EINVAL: /* Either due to wrong alignment or due to the timeout not
		     being normalized.  Must have been caused by a glibc or
		     application bug.  */
    case -ENOSYS: /* Must have been caused by a glibc bug.  */
    /* No other errors are documented at this time.  */
    default:
      futex_fatal_error ();
    }
}



// glibc-2.34/sysdeps/nptl/lowlevellock-futex.h

# define lll_futex_timed_wait(futexp, val, timeout, private)     \
  lll_futex_syscall (4, futexp,                                 \
		     __lll_private_flag (FUTEX_WAIT, private),  \
		     val, timeout)

// glibc-2.34/sysdeps/nptl/lowlevellock-futex.h
# define lll_futex_syscall(nargs, futexp, op, ...)                      \
  ({                                                                    \
    long int __ret = INTERNAL_SYSCALL (futex, nargs, futexp, op, 	\
				       __VA_ARGS__);                    \
    (__glibc_unlikely (INTERNAL_SYSCALL_ERROR_P (__ret))         	\
     ? -INTERNAL_SYSCALL_ERRNO (__ret) : 0);                     	\
  })

```

### 系统调用 INTERNAL_SYSCALL

以 ARM 64 位为例

```C
// glibc-2.34/sysdeps/unix/sysv/linux/aarch64/sysdep.h
# define INTERNAL_SYSCALL(name, nr, args...)			\
	INTERNAL_SYSCALL_RAW(SYS_ify(name), nr, args)
```

SYS_ify是个宏，用于将syscall name转换为syscall number。不同平台的syscall number是不同的，即使arm和arm64也不相同

```C
// glibc-2.34/sysdeps/unix/sysv/linux/aarch64/sysdep.h
/* For Linux we can use the system call table in the header file
	/usr/include/asm/unistd.h
   of the kernel.  But these symbols do not follow the SYS_* syntax
   so we have to redefine the `SYS_ify' macro here.  */
#undef SYS_ify
#define SYS_ify(syscall_name)	(__NR_##syscall_name)

// glibc-2.34/sysdeps/unix/sysv/linux/aarch64/arch-syscall.h
#define __NR_futex 98

```
这里将 `futex` 转化为 `__NR_futex`。再看 `INTERNAL_SYSCALL_RAW` 的实现：

```C
// glibc-2.34/sysdeps/unix/sysv/linux/aarch64/sysdep.h
# define INTERNAL_SYSCALL_RAW(name, nr, args...)		\
  ({ long _sys_result;						\
     {								\
       LOAD_ARGS_##nr (args)					\
       register long _x8 asm ("x8") = (name);			\
       asm volatile ("svc	0	// syscall " # name     \
		     : "=r" (_x0) : "r"(_x8) ASM_ARGS_##nr : "memory");	\
       _sys_result = _x0;					\
     }								\
     _sys_result; })
```

- nr 为 NumberArgument 参数个数，这里为 4。 LOAD_ARGS_##nr 为 LOAD_ARGS_4，这个宏用于将所有的参数转换为64bit， 因为 futex 是 int 并将相应的参数保存到 _x0/_x1/_x2/_x3/ 中。

```C
// glibc-2.34/sysdeps/unix/sysv/linux/aarch64/sysdep.h
# define LOAD_ARGS_0()				\
  register long _x0 asm ("x0");
# define LOAD_ARGS_1(x0)			\
  long _x0tmp = (long) (x0);			\
  LOAD_ARGS_0 ()				\
  _x0 = _x0tmp;
# define LOAD_ARGS_2(x0, x1)			\
  long _x1tmp = (long) (x1);			\
  LOAD_ARGS_1 (x0)				\
  register long _x1 asm ("x1") = _x1tmp;
# define LOAD_ARGS_3(x0, x1, x2)		\
  long _x2tmp = (long) (x2);			\
  LOAD_ARGS_2 (x0, x1)				\
  register long _x2 asm ("x2") = _x2tmp;
# define LOAD_ARGS_4(x0, x1, x2, x3)		\
  long _x3tmp = (long) (x3);			\
  LOAD_ARGS_3 (x0, x1, x2)			\
  register long _x3 asm ("x3") = _x3tmp;
```
ASM_ARGS_##nr 对应ASM_ARGS_4, 用于将x0~x3这4个寄存器作为内联汇编的输入寄存器列表

```C
// glibc-2.34/sysdeps/unix/sysv/linux/aarch64/sysdep.h
# define ASM_ARGS_0
# define ASM_ARGS_1	, "r" (_x0)
# define ASM_ARGS_2	ASM_ARGS_1, "r" (_x1)
# define ASM_ARGS_3	ASM_ARGS_2, "r" (_x2)
# define ASM_ARGS_4	ASM_ARGS_3, "r" (_x3)
```

简化 `futex_wait` 之后

```C
static __always_inline int futex_wait(unsigned int *futex_word, unsigned int expected, int private) {
  long _sys_result;
  long _x3tmp = (long)(NULL);
  long _x2tmp = (long)(expected);
  long _x1tmp = (long)((((0) | 128) ^ (private)));
  long _x0tmp = (long)(futex_word);
  register long _x0 asm("x0") = _x0tmp;
  register long _x1 asm("x1") = _x1tmp;
  register long _x2 asm("x2") = _x2tmp;
  register long _x3 asm("x3") = _x3tmp;
  register long _x8 asm("x8") = ((98));
  asm volatile(
      "svc	0	// syscall "
      "SYS_ify(futex)"
      : "=r"(_x0)
      : "r"(_x8), "r"(_x0), "r"(_x1), "r"(_x2), "r"(_x3)
      : "memory");
  long __ret = _x0;
  int err = __builtin_expect((((unsigned long int)(__ret) > -4096UL)), 0) ? -(-(__ret)) : 0;
  switch (err) {
    case 0:
    case -EAGAIN:
    case -EINTR:
      return -err;

    case -ETIMEDOUT: /* Cannot have happened as we provided no timeout.  */
    case -EFAULT:    /* Must have been caused by a glibc or application bug.  */
    case -EINVAL:    /* Either due to wrong alignment or due to the timeout not
                        being normalized.  Must have been caused by a glibc or
                        application bug.  */
    case -ENOSYS:    /* Must have been caused by a glibc bug.  */
    /* No other errors are documented at this time.  */
    default:
      futex_fatal_error();
  }
}
```
`"svc	0	// syscall " "SYS_ify(futex)"` 会被编译为 `svc	0	// syscall SYS_ify(futex)` 指令， `// syscall SYS_ify(futex)` 是注释，没任何影响。

- syscall number存储到了x8寄存器
- 最后svc 0 触发进入内核态

## pthread_mutex_unlock

```C
// glibc-2.34/nptl/pthread_mutex_unlock.c
int
___pthread_mutex_unlock (pthread_mutex_t *mutex)
{
  return __pthread_mutex_unlock_usercnt (mutex, 1);
}


int
__pthread_mutex_unlock_usercnt (pthread_mutex_t *mutex, int decr)
{
  /* See concurrency notes regarding mutex type which is loaded from __kind
     in struct __pthread_mutex_s in sysdeps/nptl/bits/thread-shared-types.h.  */
  int type = PTHREAD_MUTEX_TYPE_ELISION (mutex);

  ...
  if (__builtin_expect (type, PTHREAD_MUTEX_TIMED_NP)
      == PTHREAD_MUTEX_TIMED_NP)
    {
      /* Always reset the owner field.  */
    normal:
      mutex->__data.__owner = 0; // 解锁将 __owner 设置为 0
      if (decr) // 持有者数量减一
	      /* One less user.  */
	      --mutex->__data.__nusers;

      /* Unlock.  */
      lll_mutex_unlock_optimized (mutex);

      LIBC_PROBE (mutex_release, 1, mutex);

      return 0;
    }
  ....
}

/* lll_lock with single-thread optimization.  */
static inline void
lll_mutex_unlock_optimized (pthread_mutex_t *mutex)
{
  /* The single-threaded optimization is only valid for private
     mutexes.  For process-shared mutexes, the mutex could be in a
     shared mapping, so synchronization with another process is needed
     even without any threads.  */
  int private = PTHREAD_MUTEX_PSHARED (mutex);
  if (private == LLL_PRIVATE && SINGLE_THREAD_P)
    mutex->__data.__lock = 0;
  else
    lll_unlock (mutex->__data.__lock, private);
}

```

```C
// glibc-2.34/sysdeps/nptl/lowlevellock.h
#define lll_unlock(futex, private)	\
  __lll_unlock (&(futex), private)


/* This is an expression rather than a statement even though its value is
   void, so that it can be used in a comma expression or as an expression
   that's cast to void.  */
/* Unconditionally set FUTEX to 0 (not acquired), releasing the lock.  If FUTEX
   was >1 (acquired, possibly with waiters), then wake any waiters.  The waiter
   that acquires the lock will set FUTEX to >1.
   Evaluate PRIVATE before releasing the lock so that we do not violate the
   mutex destruction requirements.  Specifically, we need to ensure that
   another thread can destroy the mutex (and reuse its memory) once it
   acquires the lock and when there will be no further lock acquisitions;
   thus, we must not access the lock after releasing it, or those accesses
   could be concurrent with mutex destruction or reuse of the memory.  */
#define __lll_unlock(futex, private)					\
  ((void)								\
  ({									\
     int *__futex = (futex);						\
     int __private = (private);						\
     int __oldval = atomic_exchange_rel (__futex, 0);			\
     if (__glibc_unlikely (__oldval > 1))				\
       {								\
         if (__builtin_constant_p (private) && (private) == LLL_PRIVATE) \
           __lll_lock_wake_private (__futex);                           \
         else                                                           \
           __lll_lock_wake (__futex, __private);			\
       }								\
   }))

void
__lll_lock_wake (int *futex, int private)
{
  lll_futex_wake (futex, 1, private);
}

// glibc-2.34/sysdeps/nptl/lowlevellock-futex.h
/* Wake up up to NR waiters on FUTEXP.  */
# define lll_futex_wake(futexp, nr, private)                             \
  lll_futex_syscall (4, futexp,                                         \
		     __lll_private_flag (FUTEX_WAKE, private), nr, 0)

```
到 `lll_futex_syscall` 时，unlock 和 lock 就是一样的函数了。对比一下 lock 和 unlock 对 `lll_futex_syscall` 调用参数的差别：

```C
// lock
lll_futex_syscall (
  4,                                         // futex 调用的参数数量
  futexp,                                    // __lock 的指针
	__lll_private_flag (FUTEX_WAIT, private),  // futex 之后会转换为中断 号
	val,                                       // 2 __lock 的期望值，不满足并不会休眠，而是立即返回继续该线程。
  timeout                                    // 0 延时
)
// unlock
lll_futex_syscall (
  4,
  futexp,
	__lll_private_flag (FUTEX_WAKE, private),
  nr,                                        // 1 唤醒的线程数量
  0                                          // 延时
)
```

## 内核调用流程

### 中断向量表

一个系统调用以一个软中断的形式发生， 中断是和硬件相关的，不同芯片实现方式不同，因此在各个架构目录下有各自的实现。以 ARM64 为例，其在 `arch/arm64/kernel/entry.S` 目录下：


```ASM
SYM_CODE_START(vectors)
	kernel_ventry	1, t, 64, sync		// Synchronous EL1t
	kernel_ventry	1, t, 64, irq		// IRQ EL1t
	kernel_ventry	1, t, 64, fiq		// FIQ EL1h
	kernel_ventry	1, t, 64, error		// Error EL1t

	kernel_ventry	1, h, 64, sync		// Synchronous EL1h
	kernel_ventry	1, h, 64, irq		// IRQ EL1h
	kernel_ventry	1, h, 64, fiq		// FIQ EL1h
	kernel_ventry	1, h, 64, error		// Error EL1h

	kernel_ventry	0, t, 64, sync		// Synchronous 64-bit EL0
	kernel_ventry	0, t, 64, irq		// IRQ 64-bit EL0
	kernel_ventry	0, t, 64, fiq		// FIQ 64-bit EL0
	kernel_ventry	0, t, 64, error		// Error 64-bit EL0

	kernel_ventry	0, t, 32, sync		// Synchronous 32-bit EL0
	kernel_ventry	0, t, 32, irq		// IRQ 32-bit EL0
	kernel_ventry	0, t, 32, fiq		// FIQ 32-bit EL0
	kernel_ventry	0, t, 32, error		// Error 32-bit EL0
SYM_CODE_END(vectors)
```

具体含义查看
https://blog.csdn.net/liuhangtiant/article/details/90399374,
https://zhuanlan.zhihu.com/p/356968735

`kernel_ventry` 是一个宏，ARM 宏定义使用 `.macro` 开始，`.endm` 结束。

```ARM
	.macro kernel_ventry, el:req, ht:req, regsize:req, label:req
	.align 7
#ifdef CONFIG_UNMAP_KERNEL_AT_EL0
	.if	\el == 0
alternative_if ARM64_UNMAP_KERNEL_AT_EL0
	.if	\regsize == 64
	mrs	x30, tpidrro_el0
	msr	tpidrro_el0, xzr
	.else
	mov	x30, xzr
	.endif
alternative_else_nop_endif
	.endif
#endif

	sub	sp, sp, #PT_REGS_SIZE
#ifdef CONFIG_VMAP_STACK
	/*
	 * Test whether the SP has overflowed, without corrupting a GPR.
	 * Task and IRQ stacks are aligned so that SP & (1 << THREAD_SHIFT)
	 * should always be zero.
	 */
	add	sp, sp, x0			// sp' = sp + x0
	sub	x0, sp, x0			// x0' = sp' - x0 = (sp + x0) - x0 = sp
	tbnz	x0, #THREAD_SHIFT, 0f
	sub	x0, sp, x0			// x0'' = sp' - x0' = (sp + x0) - sp = x0
	sub	sp, sp, x0			// sp'' = sp' - x0 = (sp + x0) - x0 = sp
	b	el\el\ht\()_\regsize\()_\label

0:
	/*
	 * Either we've just detected an overflow, or we've taken an exception
	 * while on the overflow stack. Either way, we won't return to
	 * userspace, and can clobber EL0 registers to free up GPRs.
	 */

	/* Stash the original SP (minus PT_REGS_SIZE) in tpidr_el0. */
	msr	tpidr_el0, x0

	/* Recover the original x0 value and stash it in tpidrro_el0 */
	sub	x0, sp, x0
	msr	tpidrro_el0, x0

	/* Switch to the overflow stack */
	adr_this_cpu sp, overflow_stack + OVERFLOW_STACK_SIZE, x0

	/*
	 * Check whether we were already on the overflow stack. This may happen
	 * after panic() re-enables interrupts.
	 */
	mrs	x0, tpidr_el0			// sp of interrupted context
	sub	x0, sp, x0			// delta with top of overflow stack
	tst	x0, #~(OVERFLOW_STACK_SIZE - 1)	// within range?
	b.ne	__bad_stack			// no? -> bad stack pointer

	/* We were already on the overflow stack. Restore sp/x0 and carry on. */
	sub	sp, sp, x0
	mrs	x0, tpidrro_el0
#endif
	b	el\el\ht\()_\regsize\()_\label
	.endm

```
- `\()` 表示拼接前后的字符串
- `\el` 表示取 el 的值
- `\ht` 表示取 ht 的值

```
el\el\ht\()_\regsize\()_\label
kernel_ventry	1, t, 64, sync
.macro kernel_ventry, el:req, ht:req, regsize:req, label:req

el1t_64_sync

```

对应的处理程序

```ASM
	.macro entry_handler el:req, ht:req, regsize:req, label:req
SYM_CODE_START_LOCAL(el\el\ht\()_\regsize\()_\label)
	kernel_entry \el, \regsize
	mov	x0, sp
	bl	el\el\ht\()_\regsize\()_\label\()_handler
	.if \el == 0
	b	ret_to_user
	.else
	b	ret_to_kernel
	.endif
SYM_CODE_END(el\el\ht\()_\regsize\()_\label)
	.endm

/*
 * Early exception handlers
 */
	entry_handler	1, t, 64, sync
	entry_handler	1, t, 64, irq
	entry_handler	1, t, 64, fiq
	entry_handler	1, t, 64, error

	entry_handler	1, h, 64, sync
	entry_handler	1, h, 64, irq
	entry_handler	1, h, 64, fiq
	entry_handler	1, h, 64, error

	entry_handler	0, t, 64, sync
	entry_handler	0, t, 64, irq
	entry_handler	0, t, 64, fiq
	entry_handler	0, t, 64, error

	entry_handler	0, t, 32, sync
	entry_handler	0, t, 32, irq
	entry_handler	0, t, 32, fiq
	entry_handler	0, t, 32, error
```




## 总结

NPTL 锁并获取锁已经尽力避免了系统调用引起的消耗，在不竞争的情况下，其消耗和无锁性能是一样的，被精妙地设计为低竞争。为了减少系统调用引起的性能消耗，优化的方向是减少临界区的大小。比如对 Hash 表的访问，常用的减少临界区大小的方式比如哈希表分桶、对于非共享的数据使用TLS(Thread Local Storage)数据结构等等。**因此，对于互斥锁的使用优化就是尽量减少临界区的大小。对于单个简单数据类型的竞争（线程数量少时）可以使用 Atomic 类型的操作**


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



## 分析对错


互斥锁使用注意事项临界资源保护的使用场景，不要乱用，锁的使用是会消耗资源的(获取锁，如果获取不到会线程切换，切换需要将工作环境入栈，这就造成cpu的浪费。而且回到当前线程运行可能已经不止10ms了)

获取锁之后不要sleep太久

不要交叉使用两把锁，然后死锁

锁不要保护的太大，够用就行

业务需求是否能接受线程的切换所造成的实时性的损失(默认10ms)



2. spinlock的lock操作则是一个死循环，不断尝试trylock，直到成功。 对于一些很小的临界区，使用spinlock是很高效的。因为trylock失败时，可以预期持有锁的线程（进程）会很快退出临界区（释放锁）。所以死循环的忙等待很可能要比进程挂起等待更高效。


但是spinlock的应用场景有限，对于大的临界区，忙等待则是件很恐怖的事情，特别是当同步机制运用于等待某一事件时（比如服务器工作线程等待客户端发起请求）。所以很多情况下进程挂起等待是很有必要的。


3. 所有锁都要基于 CAS 实现吗？
https://www.zhihu.com/question/276921045

4. Linux 对线程的调度算法是怎样的？

https://www.zhihu.com/question/283318421/answer/431242454