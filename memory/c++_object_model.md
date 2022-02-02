# C++ 对象内存模型

https://mp.weixin.qq.com/s/N3Zei6xO2PJ-CPw4A3Dt0g

以下内容对 C++ 的类和 C 中结构体同时适用，进演示 C++ 的类，不再分别举例。

1. 为了寻址，C++ 对象也必须占用内存空间，为此空类占用 1 字节。

```C++
class A {

}
// sizeof(A) = a
```

## 位域

C/C++ 可以对底层硬件编程，这时候数据的位宽有时候不一定全部用到，例如开关只有通电和断电两种状态，用 0 和 1 表示足以，只需要用一个二进位。C 语言使用位域的概念来对编译器加以指示。

位域的标识仅需要在变量的后面使用 `: <位宽>` 指示变量所占的位宽。位域的大小不能超过类型的位宽，例如 int 占 32 为，则变量位域不能超过 32（超过不会编译报错，但是却不会实际分配那么多字节，导致运行时数据溢出，要格外小心。）。

```C++
class BS {
  char a;
  char b: 2;
};
```
位域限定了变量所占的字节，但是由于 C/C++ 对于赋值时是否溢出没有检查，因此应格外小心溢出问题。

位域在分配内存时尽量压缩存储空间：
```C++
class BS {
public:
  char a: 1;
  char b: 2;
};
//
cout << sizeof(BS) << endl; // 1
```
此时 b、c 占用 `1 + 2 = 3` 位，b 不会从新的地址开始，而是和 a 公用一个字节。

**C语言标准并没有规定位域的具体存储方式，不同的编译器有不同的实现，因此尽量使用编译器 sizeof 计算内存，而不是记忆字节大小分析**

例如，当相邻成员的类型相同时，如果它们的位宽之和小于类型的 sizeof 大小，那么后面的成员紧邻前一个成员存储，直到不能容纳为止；如果它们的位宽之和大于类型的 sizeof 大小，那么后面的成员将从新的存储单元开始，其偏移量为类型大小的整数倍。但当相邻成员的类型不同时，不同的编译器有不同的实现方案，GCC 会压缩存储，而 VC/VS 不会。


### 位域导致 offsetof 无法使用

由于位域存在合并，会导致表示位域的成员变量不能再使用 offsetof 计算在对象中的偏移位置。


### 无名位域

位域成员可以没有名称，只给出数据类型和位宽，如下所示：

```C++
class BS {
public:
  char a: 1;
  char  : 7;
  char b: 2;
};
cout << sizeof(BS) << endl; // 2，因为 a 后面的无名位域调整了内存占用。
```
无名位域一般用来填充或者调整成员位置。因为没有名称，无名位域不能访问。


## 对齐

### 1. 数据有顺序

结构体和类会根据定义的顺序排列变量，不会调整声明的顺序优化占用。例如，声明的数据 a, b, c，实际的数据不会编程 c，a，b。

### 2. 数据对齐(data alignment)

组合数据开始的偏移位置为 0，其后的基本数据本身的字节是 x，总是以 nx (n ∈ N) 的位置对齐。例如，short 的占字节为 2，总是在 2n 的位置上。

```C++
class A {
    char a;
    int32_t b;
}
```

`a` 的位置从偏移位置为 0 的地址开始，占一个字节，但是 b 却不会从偏移位置 1 开始，由于 b 占 4 个字节，需要从 n * 4 的位置开始 i，也即偏移位置为 4，所以 `sizeof(A)` 是 8。

```
+---+---+
| 0 | a |
+---+---+
| 1 | - |
| 2 | - |
| 3 | - |
+---+---+
| 4 |   |
| 5 | b |
| 6 |   |
| 7 |   |
+---+---+
```

除了打印变量的地址外，我们可以使用 `sizeof()` 和 `offsetof()` 关键字验证。

```C++
cout << "sizeof: " << sizeof(A) << endl;
cout << "offsset: " << offsetof(A, a) << endl;
cout << "offsset: " << offsetof(A, b) << endl;
cout << "alignof: " << alignof(A) << endl;
// 结果
sizeof: 8
offsset: 0
offsset: 4
alignof: 4
```

除此之外 `alignof()` 用于计算某个元素类型的对齐大小。除非 `#pragma pack(<对齐字节>)` 指定，alignof 是成员中变量最大的字节。例如此时 A 中 b 占用字节最大，为 4，则 `alignof(A) = 4`，另外 **`alignof` 不受位域的影响**。


### 3. 数据填充(data padding)

但是为了下一个对象的对齐，对象占用的字节会出现补齐，即整个对象的字节数是 **`n * alignof(<类型>)`** （成员中字节的最大的变量的字节整数倍）。例如将 a 和 b 的顺序换一下，占用的内存大小任仍然是 `8`。

```C++
class A {
    char a;
    int32_t b;
}
```
这是因为 `A` 占用最多字节的成员变量是 `a`，占 4 个字节，所以 A 占用只能是 `4n`，需要在最后补齐 3 字节，占 8 字节。
```
+---+---+
| 0 |   |
| 1 |   |
| 2 | b |
| 3 |   |
+---+---+
| 4 | a |
+---+---+
| 5 | - |
| 6 | - |
| 7 | - |
+---+---+
```

同理

```C++
class A {
  int32_t a;  // 0
  short b;    // 4
  int16_t c;  // 6
  char d;     // 8
};
```
将补齐 3 位，sizeof(A) 为 `12`。

```C++
class A {
  int32_t a;  // 0
  int64_t b;  // 8
  short c;    // 16
  char d;     // 18
};
```
后面补 5 字节，占 `24` 字节。

### 4. 指定对齐字节

可以使用预处理指令 `#pragma pack(2)` 为编译器指定对齐字节，可以设置的值为 1, 2, 4, 8。例如：
```C++
#pragma pack(2)
class A {
  int32_t a;  // 0
  int64_t b;  // 4
  short c;    // 12
  char d;     // 14
};
```
将占用 `16` 字节。`b` 的字节是 8，但是此时 `b` 不再从 `8n` 的偏移位置开始，而是从 `2n`，以也即第一个满足的 "4 = 2 * `2`"。

```
#pragma pack()      //设置预设对齐大小为默认规则
#pragma pack(4)     //设置预设对齐大小为4
```

### 5. 关于位域

位域会影响占用的字节数量和对齐的偏移，但是对象的整体字节，仍然需要按照成员变量中原字节的最小整数倍来补齐。例如：
```C++
class A {
public:
  int32_t a;  // 0
  int64_t b: 2;// 4
  short c;    // 6
  char d;     // 8
};
```

占中的字节是 `16`，而不是 `14`。这是因为 A 中的 `b` 原有占用的字节是 `8`, A 占用的字节需要是 `8` 的整数倍 `16`，而不是 `4` 整数倍 `12`。
```
+----+---+
|  0 |   |
|  1 |   |
|  2 | a |
|  3 |   |
+----+---+
|  4 | b |
+----+---+
|  5 | - |
+----+---+
|  6 | c |
|  7 |   |
+----+---+
|  8 | d |
+----+---+
|  9 | - |
| 10 | - |
| 11 | - |
| 12 | - |
| 13 | - |
| 14 | - |
| 15 | - |
+----+---+
```
C++ 的对象开发者显示声明的变量外，如果类中包含虚函数，则会隐式的包含指向虚函数表的指针。

## 虚函数

C++ 的对象开发者显示声明的变量外，如果类中包含虚函数，则会隐式的包含指向虚函数表的指针（__vptr）。这个指向虚函数的地址列表称为虚函数表。

```C++
class A {
public:
  int a;
  void f();
};
// sizeof(A) = 4
// 添加虚函数作为对比
class A {
public:
  int a;
  void f();
  virtual void f1();
  virtual void f2();
};
// 带虚函数的内存占用
// sizeof(A) = 24
```

可以给 clang++ 添加 `-Xclang -fdump-record-layouts` 编译参数输出内存布局。
clang 为了每一个类生成一个 vtable 虚函数表，放在程序的.rodata段，其他编译器（平台），实现可能不同.

```
*** Dumping AST Record Layout
         0 | class A
         0 |   int a
           | [sizeof=4, dsize=4, align=4,
           |  nvsize=4, nvalign=4]

*** Dumping AST Record Layout
         0 | class A
         0 |   (A vtable pointer)
         8 |   int a
           | [sizeof=16, dsize=12, align=8,
           |  nvsize=12, nvalign=8]
```
clang 添加 `-Xclang -fdump-vtable-layouts` 输出虚函数表：

```
Vtable for 'A' (4 entries).
   0 | offset_to_top (0)
   1 | A RTTI
       -- (A, 0) vtable address --
   2 | void A::f1()
   3 | void A::f2()
```

虚函数表包含三部分信息：
- offset_to_top: 表示 this 指针对子类的偏移，用于子类和继承类之间 dynamic_cast 转换
vbase_offset 表示this指针对基类的偏移，用于共享基类；
- vbase_offset 表示this指针对基类的偏移，用于共享基类；
- typeinfo
- 虚函数的地址： 本质是函数指针，多个函数就有多个指针。

```C++
A* a = new A();
```
```
                                         Vtable
                               +-----------------------+
                               | 0 (offset_to_top)     |
                               +-----------------------+
                               | ptr to typeinof for A |
a --> +-------------+          +-----------------------+
      |  __vpter    | -------> |    ptr to A::f1()     |  ---------> A::f1()
      +-------------+          +-----------------------+
      |  a   |  -   |          |    ptr to A::f2()     |  ---------> A::f2()
      +------+------+          +-----------------------+
                                                                     A::f()
```
- 非虚函数不影响对象内存大小，仅在代码区有一个函数。

- 带虚函数的类，会添加一个指向虚函数表的指针，多个虚函数也仅有一个指针，这个指针在对象的起始位置。虚函数表中的函数指针再指向虚函数本身。

- 指向虚函数的指针并不是指向表头，而知指向表中虚函数指针列表开始的位置。

为了验证虚函数表的指针，如下代码是等价的。

```C++
using Fun = void(*)(void);
// 或
// typedef void(*Fun)(void);

A* a = new A();
a->f1();
a->f2();
// 等价于
(**(Fun**)a)();
((*(*(Fun**)a + 1)))();
```

### 虚函数表是实现多态的原理

C++ 要实现多态比较严苛，必须满足
- 父类中声明虚函数，子类中覆盖虚函数
- 使用指针来调用虚函数。直接使用对象不行。

```C++
class A {
public:
  int a;
  virtual void f1();
  virtual void f2();
};

class B: public A{
public:
  int b;
  void f1() override;
};

int main() {
  A* a = new B();
  a->f1();
  a->f2();
  // pFun();
  return 0;
}

void A::f1() {
  cout << "A::f1()" << endl;
}

void A::f2() {
  cout << "A::f2()" << endl;
}

void B::f1() {
  cout << "B::f1()" << endl;
}
```
输出结果为：
```
B::f1()
A::f2()
```

```
                                     Vtable for A
                               +-----------------------+
                               | 0 (offset_to_top)     |
                               +-----------------------+
            A obj              | ptr to typeinof for A |
      +-------------+          +-----------------------+
      |  __vpter    | -------> |    ptr to A::f1()     |  ---------> A::f1()
      +-------------+          +-----------------------+
      |  a   |  -   |          |    ptr to A::f2()     |  ----┬----> A::f2()
      +------+------+          +-----------------------+      |
                                                              |
                                                              |
                                      Vtable for B            |
                               +-----------------------+      |
                               | 0 (offset_to_top)     |      |
                               +-----------------------+      |
            B obj              | ptr to typeinof for B |      |
a --> +-------------+          +-----------------------+      |
      |  __vpter    | -------> |    ptr to A::f1()     |  ---------> B::f1()
      +-------------+          +-----------------------+      |
      |  a   |  b   |          |    ptr to A::f2()     |  ----┘
      +------+------+          +-----------------------+
```


多继承时，虚函数表变得复杂

```C++
class A {
public:
  int a;
  virtual void a1();
  virtual void a2();
};
class B {
public:
  int b;
  virtual void b1();
};
class C : public A, public B {
public:
  int c;
  void c1();
};
```

```
                                     Vtable for D
                               +-----------------------+
                               |   offset_to_top (0)   |
                               +-----------------------+
            C obj              | ptr to typeinof for C |
      +-------------+          +-----------------------+
      |  __vpter    | -------> |    ptr to A::a1()     |
      +-------------+          +-----------------------+
      |  a   |  -   |          |    ptr to A::a2()     |
      +------+------+          +-----------------------+
      |  __vpter    |----┐     |  offset_to_top (-16)  |
      +------+------+    |     +-----------------------+
      |  b   |  c   |    |     | ptr to typeinof for C |
      +------+------+    |     +-----------------------+
                         └---> |    ptr to B::b1()     |
                               +-----------------------+
```

- 基类成员变量聚在一起，因为 C++ 允许将子类对象转成父类使用。一次一块结构必须和父类结构完全相同。

### 非虚继承的菱形继承

```
class A {
public:
  int a;
  virtual void v();
};

class B : public virtual A {
public:
  int b;
  virtual void w();
};

class C : public virtual A {
public:
  int c;
  virtual void x();
};

class D : public B, public C {
public:
  int d;
  virtual void y();
};
```

非虚继承会让父类的成员在所有子类都包含该成员，当孙类同时继承不同的子类时，就会导致成员重复。

```
                                     Vtable for D
                               +-----------------------+
                               |   offset_to_top (0)   |
                               +-----------------------+
           D obj               |ptr to typeinof for ABD|
      +-------------+          +-----------------------+
      |  __vpter    | -------> |    ptr to A::a1()     |
      +-------------+          +-----------------------+
      |  a   |  b   |          |    ptr to A::a2()     |
      +------+------+          +-----------------------+
      |  __vpter    | ----┐    |    ptr to B::b1()     |
      +------+------+     |    +-----------------------+
      |  a   |  c   |     |    |    ptr to D::d1()     |
      +------+------+     |    +-----------------------+
      |  d   |  -   |     |    |  offset_to_top (-16)  |
      +------+------+     |    +-----------------------+
                          |    | ptr to typeinof for AC|
                          |    +-----------------------+
                          └--> |    ptr to A::a1()     |
                               +-----------------------+
                               |    ptr to A::a2()     |
                               +-----------------------+
                               |    ptr to C::c1()     |
                               +-----------------------+
```

可以看到，A 中的成员会在 B、C 中个各有一个，为了消除这种重复，可以使用虚继承

### 虚继承的菱形继承

```C++
class A {
public:
  int a;
  virtual void a1();
  virtual void a2();
};
class B: public virtual A {
public:
  int b;
  virtual void b1();
};
class C : public virtual A {
public:
  int c;
  virtual void c1();
};

class D : public B, public C {
public:
  int d;
  virtual void d1();
};
```

```
                                     Vtable for D
                               +-----------------------+
                               |   vbase_offset (32)   |
                               +-----------------------+
                               |   offset_to_top (0)   |
                               +-----------------------+
           D obj               |ptr to typeinof for BD |
      +-------------+          +-----------------------+
      |  __vpter B  | -------> |    ptr to B::b1()     |
      +-------------+          +-----------------------+
      |   b  |      |          |    ptr to D::d1()     |
      +------+------+          +-----------------------+
      |  __vpter C  | ----┐    |    vbase_offset (16)  |
      +------+------+     |    +-----------------------+
      |  c   |  d   |     |    |  offset_to_top (-16)  |
      +------+------+     |    +-----------------------+
      |  __vpter A  |---┐ |    | ptr to typeinof for C |
      +------+------+   | |    +-----------------------+
      |  a   |      |   | └--> |    ptr to C::c1()     |
      +------+------+   |      +-----------------------+
                        |      |    vcall_offset (0)   |
                        |      +-----------------------+
                        |      |    vcall_offset (0)   |
                        |      +-----------------------+
                        |      |  offset_to_top (-32)  |
                        |      +-----------------------+
                        └----> |    ptr to A::a1()     |
                               +-----------------------+
                               |    ptr to A::a2()     |
                               +-----------------------+
```

注意点：

1.top_offset 表示this指针对子类的偏移，用于子类和继承类之间dynamic_cast转换（还需要typeinfo数据），实现多态，
vbase_offset 表示this指针对基类的偏移，用于共享基类；

2.gcc为了每一个类生成一个vtable虚函数表，放在程序的.rodata段，其他编译器（平台）比如vs，实现不太一样.

3.gcc还有VTT表，里面存放了各个基类之间虚函数表的关系，最大化利用基类的虚函数表，专门用来为构建最终类vtable；
4.在构造函数里面设置对象的vtptr指针。

5.虚函数表地址的前面设置了一个指向type_info的指针，RTTI（Run Time Type Identification）运行时类型识别是有编译器在编译器生成的特殊类型信息，包括对象继承关系，对象本身的描述，RTTI是为多态而生成的信息，所以只有具有虚函数的对象在会生成。

6.在C++类中有两种成员数据：static、nonstatic；三种成员函数：static、nonstatic、virtual。

C++成员非静态数据需要占用动态内存，栈或者堆中，其他static数据存在全局变量区（数据段）,编译时候确定。虚函数会增加用虚函数表大小，也是存储在数据区的.rodada段，编译时确定，其他函数不占空间。

7.G++选项 -fdump-class-hierarchy 可以生成C++类层结构，虚函数表结构，VTT表结构。

8.GDB调试选项：
set p obj <on/off> ：在C++中，如果一个对象指针指向其派生类， 如果打开这个选项，GDB会现在类对象结构的规则显示输出。
set p pertty <on/off>:   按照层次打印结构体。

思考问题：

1 Why don't we have virtual constructors?
From Bjarne Stroustrup's C++ Style and Technique FAQ
A virtual call is a mechanism to get work done given partial information. In particular, "virtual" allows us to call a function knowing only any interfaces and not the exact type of the object. To create an object you need complete information. In particular, you need to know the exact type of what you want to create. Consequently, a "call to a constructor" cannot be virtual.

2  为什么不要在构造函数或者析构函数中调用虚函数？

对于构造函数：此时子类的对象还没有完全构造，编译器会去虚函数化，只会用当前类的函数， 如果是纯虚函数，就会调用到纯虚函数，会导致构造函数抛异常：pure virtual method calle；对于析构函数：同样，由于对象不完整，编译器会去虚函数化，函数调用本类的虚函数，如果本类虚函数是纯虚函数，就会到账析构函数抛出异常：  pure virtual method called；

3  C++对象构造顺序？

1．构造子类构造函数的参数
2．子类调用基类构造函数
3．基类设置vptr
4．基类初始化列表内容进行构造
5.  基类函数体调用
6.  子类设置vptr
7.  子类初始化列表内容进行构造
8.  子类构造函数体调用

4  为什么虚函数会降低效率？

是因为虚函数调用执行过程中会跳转两次，首先找到虚函数表，然后再查找对应函数地址，这样CPU指令就会跳转两次，而普通函数指跳转一次，CPU每跳转一次，预取指令都可能作废，这会导致分支预测失败，流水线排空，所以效率会变低。设想一下，如果说不是虚函数，那么在编译时期，其相对地址是确定的，编译器可以直接生成jmp/invoke指令；如果是虚函数，多出来的一次查找vtable所带来的开销，倒是次要的，关键在于，这个函数地址是动态的，譬如 取到的地址在eax里，则在call eax之后的那些已经被预取进入流水线的所有指令都将失效。流水线越长，一次分支预测失败的代价也就越大。

## RTTI





参考：
[对齐](https://blog.csdn.net/lizi_stdio/article/details/77203335)
[C++ 对象模型](https://mp.weixin.qq.com/s/N3Zei6xO2PJ-CPw4A3Dt0g)
[虚函数表](https://blog.csdn.net/m0_37595954/article/details/102689725)
[RTTI](https://blog.csdn.net/ljianhui/article/details/46487951)