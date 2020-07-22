# NDK

NDK 是安卓提供的一套用于开发、编译、链接、调试 C/C++ 程序的工具集合。NDK 开发主要用于以下场景：

- 实现软件低延迟或计算密集型程序的性能。例如游戏或者物理模拟程序。

- 复用已有的 C/C++ 库。

- 开发跨平台的软件库。

```
C/C++ 源码 --(ndk编译) ------┐
                            ├- 原生库 -- (gradle 打包) --> Java 调用
定义 JNI 代码 --(ndk 编译) ---┘
```

## 编译

Android Studio 提供了多种方式编译 C/C++ 代码的方式。

- `CMake` 是 Andriod Studio 的默认方式，对于支持 CMake 的库和新建库，尽量以 CMake 的方式编译。
- `ndk-build` 适用于已经使用 make 的软件库。可以使用这种方式编译。
- 辅助工具，例如 `CCache`很少使用，根据特殊需要可以使用。

使用 NDK 开发，你甚至可以完全使用 Native 开发应用（完全不使用 Java 和 Kotlin），但是你需要慎重的考虑为什么这样做，毕竟安卓提供的组件很方便的实现 UI 的操作和控制。跟多的是需要考虑的哪些部分使用 Native 开发，哪些使用 Java 开发。



## Flow

The general flow for developing a native app for Android is as follows:

1. Design your app, deciding which parts to implement in Java, and which parts to implement as native code.

Note: While it is possible to completely avoid Java, you are likely to find the Android Java framework useful for tasks including controlling the display and UI.

2. Create an Android app Project as you would for any other Android project.

3. If you are writing a native-only app, declare the NativeActivity class in AndroidManifest.xml. For more information, see the Native Activities and Applications.

4. Create an Android.mk file describing the native library, including name, flags, linked libraries, and source files to be compiled in the "JNI" directory.

5. Optionally, you can create an Application.mk file configuring the target ABIs, toolchain, release/debug mode, and STL. For any of these that you do not specify, the following default values are used, respectively:

    - ABI: all non-deprecated ABIs
    - Toolchain: Clang
    - Mode: Release
    - STL: system

6. Place your native source under the project's jni directory.

7. Use ndk-build to compile the native (.so, .a) libraries.

8. Build the Java component, producing the executable .dex file.

9. Package everything into an APK file, containing .so, .dex, and other files needed for your app to run.



