# 生成 Native 函数声明

## 指令

javah <full class name>


## 配置 Android studio 工具

https://github.com/YaowenGuo/SpeexDoc

在 Mac 上的 Android Studio 4.0 配置 External Tools 命令生成头文件。内置的 Macros 文件路径 `$ModuleFileDir$` 指向了 `.idea/module` 目录，又没有找到其他相应的路径。这种方法被废弃：

1. Macros 路径错误，无法使用。写死路径使命令仅限几处使用。非常麻烦。

2. 生成是整个文件生成，在开发中经常添加新方法，并不适用。

**推荐**

使用 Android Studio 错误的提示，然后 Alt + Enter 快速生成。更够根据方法逐个生成，方便灵活。

![生成 Native 方法签名](images/quily_create_native_function_declaration.png)

由于编译器还不支持 Rust，如果是使用 Rust 写本地方法，由于 Android Studio 检测不到，会有红色错误提醒。虽然不影响编译运行，但是太刺眼，可以使用 `@Suppress("KotlinJniMissingFunction")` 抑制提示。

