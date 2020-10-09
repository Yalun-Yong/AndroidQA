# Android 配置 JNI 开发环境

## 使用 JNI 有三种方式。

1. 使用编译好的 so 库。这种方式可以是任意的编译方式，只需要将编译好的 so 库放到安卓项目中的 `src/main/jni` 目录下。

2. 使用 MakeFile 编译。Android Studio 支持，可以将源码放在项目中。特别适合需要不断添加新功能代码的 JNI.

3. 使用 CMake 编译，Android Studio 目前主推的编译，用于代替之前的 Makefile 方式。

在同一个模块中， 2 和 3 只能使用一种。


### 配置 Makefile 编译。

安卓为了简化 Makefile 的编写，内置了一个 Makefile 文件，同时留了一个 Android.mk 的接口文件，在这个文件内，只需要简单的配置参数，就能实现编译，而不必写复杂的依赖关系。Application.mk 文件实际上是对应用程序本身进行描述的文件，它描述了应用程序要针对哪些 CPU 架构打包动态 so 包、要构建的是 release 包还是 debug 包以及一些编译和链接参数等。





### 配置 CMake 编译






## 生成 Native 函数声明

### 指令

javah <full class name>


### 配置 Android studio 工具

https://github.com/YaowenGuo/SpeexDoc

在 Mac 上的 Android Studio 4.0 配置 External Tools 命令生成头文件。内置的 Macros 文件路径 `$ModuleFileDir$` 指向了 `.idea/module` 目录，又没有找到其他相应的路径。这种方法被废弃：

1. Macros 路径错误，无法使用。写死路径使命令仅限几处使用。非常麻烦。

2. 生成是整个文件生成，在开发中经常添加新方法，并不适用。

**推荐**

使用 Android Studio 错误的提示，然后 Alt + Enter 快速生成。更够根据方法逐个生成，方便灵活。

![生成 Native 方法签名](images/quily_create_native_function_declaration.png)

由于编译器还不支持 Rust，如果是使用 Rust 写本地方法，由于 Android Studio 检测不到，会有红色错误提醒。虽然不影响编译运行，但是太刺眼，可以使用 `@Suppress("KotlinJniMissingFunction")` 抑制提示。

