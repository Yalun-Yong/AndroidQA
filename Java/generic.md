# Generic

参数化类型。泛型抽象一类操作，多用于一类对象。而在使用时可以具体化类型（主要看返回值是否有多种类型，通过返回具体类型，避免了强转的类型错误发生）。

类型安全，将类型检查提前到编译期。
潜在的性能收益，将类型转化提前到编译器，带来了编译期优化的可能。

## extends vs supper


dextends 在声明时(做形参)，表示限定接收的类型的范围。而在实例化(实参)的时候，由于泛型仅能接收同种类型的变量，即等号左右需要完全一样。否则要想向上转型，需要使用 `?`

```
class Fruit {}

class Apple extends Fruit {}

class Banana extends Fruit {}

ArrayList<? extends Fruit> fruitList = new ArrayList<Fruit>();

// 多层时
ArrayList<? extends List<? extends Fruit>> fruitList = new ArrayList<List<Apple>>();

```

付出的代价是所有的参数带该泛型的，都不能调用了。因为泛型擦除的原因，在运行的时候不确定要传入的具体类型。只能禁止传入参数。但是能够调用返回类型是 `Fruit` 的方法。


相反，supper 为向下转型，这时，仅能添加，不能获取。因为实际的对象是一个父类容器，所以可以添加对象被父类引用。


Type parameter: 形参
Type arguments: 实参

- 声明的地方是形参，其他都是实参。 例如 ｀class A<T> extends B<T>｀， `T` 对于 A 来说是形参，而对于 B 来说是实参。因为在确定了调用 B 时传入的类型。

`？`  仅能做参。仅能用在变量的声明部分。

in 和 out 是 kotlin 中的修饰符，它们等于

- out = ? extends 指定了输出类型的边界值，都会输出该类的类型。
- in = ? super 指定了输入类型的边界值。
- `*` 相当于两者的结合，用于从声明中直接继承使用。声明中使用的是 `in`，在定义变量的时候可以直接写 `*`，简略式的写法。


## 类型擦除

所谓类型擦除，就是泛型的类实例化的时候并没有实例化的泛型信息。而是只保存了定义的的类的信息。例如

```Java
List<String>
List<Integer>
```
在编译后，都是 `List` 类的，一样的。
Java 这么做，一是因为　1.5 版本才加入泛型，为了向上兼容，让新代码和老代码相互调用是不出现问题。二是效率考虑，不同的类型保存不同的信息，会使代码膨胀。

但是，为了能够反射，它有不是完全擦除的，它保存了类型的上线类型。例如

```Java
class Test<T extends List> {
    T value;
}
```

在字节码中，value 的类型有一个　`descroptor`用于描述，它保存的类型信息是　`List`, 如果没有声明类型上限，它保存的类型是 `Object`. 所以可以在反射的时候通过　getGenericParamType 或者　`getGenericReturnType` 来获得上线信息。

既然保存了上限类，那就可以做一些特殊用处。例如，gson 反序列化的时候，就是通过声明类型的子类来保存信息的。

```Java
TypeToken<List<String>> = new TypeToken<List<String>>() {}
```

此时，由于声明了子类，TypeToken 保存的上限信息就是 `List` 和 `String`,而不再是 `Object` 了，这样就能正确的通过反射实例化类。
