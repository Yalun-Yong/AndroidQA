# 【C11 Atomic 操作](https://www.it610.com/article/1279523769076957184.htm)

C/C++11 定义了原子类型 `atomic_<type>`，例如 `atomic_bool`, `atomic_char`, `atomic_int`...。
C 定义在 `stdatomic.h` 中，使用 `_Atomic()` 宏定义。使用 atomic 类型和函数需要引入该头文件。


## 原子对象的初始化

初始化有两种，分别应用于元基变量和局部变量的场景。**原子对象的初始化并不保证原子性，因此需要单独一个线程初始化好，再在多线程中使用。例如 ATOMIC_VAR_INIT 就是简单的将值返回的宏**

### 全局变量初始化

```c
static atomic_bool global_var = ATOMIC_VAR_INIT(0)
```

### 局部变量初始化

```c
atomic_int a;
atomic_init(&a, 1);
```

使用原子操作函数之前，必须保证该原子对象已经被初始化。


## 加载和存储

把原子对象的值加载到一个普通的对象中，或者讲一个太普通对象的值存储到原子对象中，应该使用原子操作函数，而不是 `=`。

### 使用 atomic_load 获取值

```c
int c = atomic_load(&global_var) // 将全局变量 global_var 赋值给 c。
```

### 使用 atomic_store 赋值

```c
atomic_store(&global_var, a);
```

**不能使用 atomic_store 对原子对象进行初始化。**


## 数学运算

C11标准是规定了五种可用以原子对象的算术逻辑操作，加、减、或、异或、与，在使用这五个操作的时候，操作数类型不能是atomic_bool类型，对这种类型不支持。

这五种操作运算对应到C11原子操作接口，使用宏定义为使用编译器的内置函数实现，不同的编译器对应不同硬件平台的实现不一样。

数学运算分别是atomic_fetch作为前缀，add就是加法，sub就是减法，or就是或，xor就是异或，and就是与。

**数学运算函数的返回值是运算之前的值。**


## 实现

C11 只一个标准，各个平台和编译器器实现的具体代码不同，但最终是有编译器和硬件来支持的。以 GCC 为例 `atomic_compare_exchange_strong` 定义为 [atomic_compare_exchange_strong](https://github.com/gcc-mirror/gcc/blob/master/gcc/ginclude/stdatomic.h)，并最终调用了 gcc 内置函数 [`__atomic_compare_exchange`]()

原子操作是无锁操作，需要硬件在底层做支持，也就是 `CAS` 或者 `TAS` 原语。各个处理器上支持的原子操作的种类和形式各不相同。在现代 x86 处理骑上，基本算数运算、逻辑运算指令前添加 `LOCK` (80486)即可使用使用原子操作。比如x86处理器以及[ARMv8.1架构等处理器直接提供了CAS指令作为原子条件原语](https://blog.csdn.net/Roland_Sun/article/details/107552574)。而ARMv7、ARMv8处理器则使用了另一种LL-SC，这两种原子条件原语都可以作为Lock-free算法工具。

比 CAS 与 LL_SC 更早一些一些的原子条件有 SWAP(8086, ARMv5), Bit test adn set(80386. Blackfin SDP561) 等，这些同步原语只能用于 `同步锁`，而无法作为 `lock-free` 的原子对象进行操作。另外，CAS 与 LL-SC 条件原语都能实现 SWAP 和 Bit test and set 指令的功能。

- 使用处理器提供的指令级别的原子条件原语，可用来对原子对象过更更夫的修改操作，比如浮点数的原子计算。C11 标准中提供了 CAS 形式的条件原语。

在C11标准当中，就只提供的CAS这种，宏函数接口名为 atomic_compare_exchange_strong 以及atomic_compare_exchange_weak，第一种是保证数据比较交换是成功还是失败，结果马上就会出来，而这个weak往往针对通过LL-SC指令模拟CAS，里面会产生一些副作用，我做一次比较和交换的时候，我这个结果确实已经交换成功了，但是返回结果可能是失败的，当然我们也可通过一次循环再一次迭代，然后直到它成功返回为止。


那么我们再介绍宏函数的时候，我以strong为例，函数原形是这个样子，返回bool类形，这个函数的语义就是我先比较object的原子对象指针所指的原子对象，与expected所指的内容对象是否相同，如果这两个指针所指的内容相同，我将desired的值存储到object，并且最终返回，我这次修改操作是成功的。否则，也就是expected和object两个内容不相同，这个时候会将object所指的值复制到expected，并且返回true为，我们使用接口的时候我们的操作步序往往是先将atomic原子对象指针的值先拿出来，放到一个普通的变量当中去，我们再去写我object原子对象值的时候我们要用desired，写进去的时候，我先比较expercted为和object是否相同，如果相同就说明我在做原子，从加载到做的过程当中外部没有干扰，也就是我没有存在另外一个线程也使用原子操作，对我当前的object对象进行修改，这个时候两个内容是完全相同的，这个时候显示成功。如果我先用desired对oject的值进行修改的时，这个值被其他线程修改了，也就是我在做atomic_load与atomic_compare_exchange_strong之间有一个缝隙，正好被另外一个线程抓住把柄，它在当前线程执行atomic_compare_exchange_strong前先修改了object的值，这样就会出现两个值不同，这个时候就会返回false。




例如

```
atomic_fetch_add(&a, 10);
```

x86 汇编
```
lock add	dword ptr [rbp - 4], 10
```
lock 和 add 其实是两条指令，我们使用 `objdump -d` 反汇编。

```shell
objdump --x86-asm-syntax=intel -d c11_atomic_x64.o
```

```
...
b: f0                           	lock
c: 83 45 fc 0a                  	add	dword ptr [rbp - 4], 10
...
```

很多处理器是不直接支持原子加法原子减法操作，但是这个时候我们可以通过更底层，更根源的原子操作指令实现。ARMv8.1 架构等处理器直接提供了CAS指令作为原子条件原语。而ARMv7、ARMv8处理器则使用了另一种LL-SC，这两种原子条件原语都可以作为Lock-free算法工具。

arm64
```asm
	ldaxr	w9, [x8]
	add	w9, w9, #10                     // =10
	stlxr	w10, w9, [x8]
```

armv7a
```
	ldrex	r1, [r0]
	add	r1, r1, #10
	strex	r2, r1, [r0]
```
ldr和str这两条指令大家都是非常的熟悉了，后缀的ex表示Exclusive，是ARMv7提供的为了实现同步的汇编指令。
```
LDREX  <Rt>, [<Rn>]
```
<Rn> 是base register，保存 memory 的address。LDREX指令从base register中获取memory address，并且将memory的内容加载到<Rt>(destination register)中。这些操作和ldr的操作是一样的，那么如何体现exclusive呢？其实，在执行这条指令的时候，还放出两条“狗”来负责观察特定地址的访问（就是保存在[<Rn>]中的地址了），这两条狗一条叫做local monitor，一条叫做global monitor。


STREX <Rd>, <Rt>, [<Rn>]

和LDREX指令类似，<Rn>是base register，保存memory的address，STREX指令从base register中获取memory address，并且将<Rt> (source register)中的内容加载到该memory中。这里的<Rd>保存了memeory 更新成功或者失败的结果，0表示memory更新成功，1表示失败。STREX指令是否能成功执行是和local monitor和global monitor的状态相关的。对于Non-shareable memory（该memory不是多个CPU之间共享的，只会被一个CPU访问），只需要放出该CPU的local monitor这条狗就OK了，下面的表格可以描述这种情况


ARM v8.1 CAS

```
    mov	w9, #10
	ldaddal	w9, w8, [x8]
```
