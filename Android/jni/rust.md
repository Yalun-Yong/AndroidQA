# 支持 Rust

#### 1. 创建源文件目录

Android Studio 的源码都在 main 目录下，创建 rust 目录:

右键点击 main -> Open in Terminal

在打开的终端中输入：

```shell
cargo new lib_name --lib
```

指令中的 `lib_name` 表示目录名，你应该改成你实际库的名字，方便后期阅读。 `--lib` 表示创建一个库，而不是用于直接运行的程序。

#### 2. 配置编译源文件目录

默认情况下 Android 的 gradle 只将 main 和 cpp 作为源文件目录，为了将自己的目录页加入到编译的目录中。在 app 的 build.gradle 中添加：

```
android {
    defaultConfig {
        sourceSets.main {
             // 添加源文件目录
            jni.srcDirs = ['src_main']
        }
    }
```