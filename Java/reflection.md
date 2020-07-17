# Reflection

通过对象获取私有的变量、方法、甚至实际指向的子类对象的变量，方法的技术。

## 获取类

如果类是可以访问权限是可以直接访问的，可以直接通过字节码创建对象

```Java
Class myClass = JavaClass.class;
Object object = myClass.newInstance();
```

如果 Class 被声明的权限导致不能够获取，则可以通过类加载器加载。

```
Class myClass = Class.forName("tech.yaowen.test_reflection.Utils");
myClass.getConstructors();
Constructor constructor = myClasgetConstructors()[0];
constructor.setAccessible(true);
Object object = constructor.newInstance();
```


## 字节码能获取的内容

要首先通过 `getClass()` 获取类的字节码，然后调用

- getDeclaredFields(): 获取变量
- getCanonicalName(): 用 `#` 分割的外部类和内部类完整类名。
- getName(): 用 `.` 分割外部类和内部类的完整类名。

## 变量

首先要获取的 `Field` 对象

- ` <T extends Annotation> T getAnnotation(Class<T> var1)` 按类名获取到注解对象。
- `setAccessible(true)` 将私有的变量设置为可访问。否则会抛出访问异常。
- `set(obj, bind.value())` 设置 obj 对象的　field 的值。