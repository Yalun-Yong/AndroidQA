# 使用 Rust 代替 C/C++ 做安卓 native 开发

同 C/C++ 一样，Rust 做 android 的 native 开发也有两种方式： 

1. 可以使用编译好的 so，这种方式适合饮用已有的三方库，而不用对其进行更改。

2. 在 项目中使用 Rust 开发，打包时实时编译代码。这种方式比较适合自己开发的库，以及 JNI 层开发。

前提条件：安装配置好 [Rust 的开发环境](https://www.rust-lang.org/tools/install)和安卓开发环境。

## 方式一：编译好 so 库引入。

既然这种方式主要用于已有的库编译成 so 库，就不介绍编写 JNI 接口部分了，因为编写 JNI 使用这种方式开发和测试都很繁琐，没有实用价值。

### 创建 Rust lib 库。

在打开的终端中输入：

```shell
cargo new lib_name --lib
```

### 添加 JNI 支持 

在 Rust 库的根目录中有个 `Cargo.toml` 用于配置 Rust 库的依赖和库类型。添加如下配置

```
[dependencies]
# 添加 JNI 支持
jni = { version = "0.17.0", default-features = false }

[lib]
# 项目作为动态库使用
crate_type = ["cdylib"]
```

### 配置交叉编译工具链

要将 Rust 库编译为安卓上的动态库，就涉及到交叉编译工具链的设置。Rust 编译器天生就支持 arm 架构处理器的编译能力，我们仅需要这只一下交叉编译工具链即可。

在 Cargo 的安装目录（不知道的话可以通过 which cargo 查找）添加 `config` 文件，该配置将用于所有项目。如果仅想对某项目添加配置，可以在 Rust 项目内添加 `cargo-config.toml` 配置文件。在配置文件中添加如下内容，用于配置交叉编译工具链：

```
[target.aarch64-linux-android]
ar = "<NDK 的路径>/toolchains/llvm/prebuilt/darwin-x86_64/bin/aarch64-linux-android-ar"
linker = "<NDK 的路径>/toolchains/llvm/prebuilt/darwin-x86_64/bin/aarch64-linux-android30-clang"

[target.armv7-linux-androideabi]
ar = "<NDK 的路径>/toolchains/llvm/prebuilt/darwin-x86_64/bin/arm-linux-androideabi-ar"
linker = "<NDK 的路径>/toolchains/llvm/prebuilt/darwin-x86_64/bin/armv7a-linux-androideabi30-clang"

[target.i686-linux-android]
ar = "<NDK 的路径>/toolchains/llvm/prebuilt/darwin-x86_64/bin/i686-linux-android-ar"
linker = "<NDK 的路径>/toolchains/llvm/prebuilt/darwin-x86_64/bin/i686-linux-android30-clang"

[target.x86_64-linux-android]
ar = "<NDK 的路径>/toolchains/llvm/prebuilt/darwin-x86_64/bin/x86_64-linux-android-ar"
linker = "<NDK 的路径>/toolchains/llvm/prebuilt/darwin-x86_64/bin/x86_64-linux-android30-clang"
```

在 windows 上应该是如下的路径格式。

```
[target.aarch64-linux-android]
ar = "E:\\路径\\toolchains\\llvm\\prebuilt\\darwin-x86_64\\bin\\aarch64-linux-android-ar.exe"
linker = "<NDK 的路径>\\toolchains\\llvm\\prebuilt\\darwin-x86_64\\bin\\aarch64-linux-android30-clang.cmd"

[target.armv7-linux-androideabi]
ar = "<NDK 的路径>\\toolchains\\llvm\\prebuilt\\darwin-x86_64\\bin\\arm-linux-androideabi-ar.exe"
linker = "<NDK 的路径>\\toolchains\\llvm\\prebuilt\\darwin-x86_64\\bin\\armv7a-linux-androideabi30-clang.cmd"

[target.i686-linux-android]
ar = "<NDK 的路径>\\toolchains\\llvm\\prebuilt\\darwin-x86_64\\bin\\i686-linux-android-ar.exe"
linker = "<NDK 的路径>\\toolchains\\llvm\\prebuilt\\darwin-x86_64\\bin\\i686-linux-android30-clang.cmd"

[target.x86_64-linux-android]
ar = "<NDK 的路径>\\toolchains\\llvm\\prebuilt\\darwin-x86_64\\bin\\x86_64-linux-android-ar.exe"
linker = "<NDK 的路径>\\toolchains\\llvm\\prebuilt\\darwin-x86_64\\bin\\x86_64-linux-android30-clang.ar"
```

需要注意的是：

- 从 NDK 19 开始，NDK 中默认带有 `toolchains` 可供使用，与任意构建系统进行交互时不再需要使用。使用 python `make_standalone_toolchain.py` 或者 shell 构建 toolchain 没有必要了，虽然也提供了该脚本，仅仅是为了兼容，强烈见识使用新的方式构建 so 库。

- [Adroid 4.0 (API 14)默认不再支持 armeabi。](https://www.jianshu.com/p/4b1c2dd3c87f)
[Android 4.4 (API 19)之后强制要求armv7处理器。](https://stackoverflow.com/questions/10920747/android-cpu-arm-architectures)。所以这里近提供了 armv7 的编译配置，除非你需要适配 4.3 以下的手机，否则不需要在编译 armabi 库了（想要编译 armabi 库，可以使用 NDK 17 之前的版本）。

### 定义 JNI 接口

在 Rust 目录下的 src 中添加 lib.rs 文件，并添加如下代码。

```Rust
#![cfg(target_os = "android")]
#![allow(non_snake_case)]

use std::ffi::{CString, CStr};
use jni::JNIEnv;
use jni::objects::{JObject, JString};
use jni::sys::{jstring};

#[no_mangle]
pub unsafe extern "C"  fn Java_tech_yaowen_androidrust_Hello_stringFromJNI(env: JNIEnv, _: JObject, j_recipient: JString) -> jstring {
    let recipient = CString::from(
        CStr::from_ptr(
            env.get_string(j_recipient).unwrap().as_ptr()
        )
    );

    let output = env.new_string("Hello ".to_owned() + recipient.to_str().unwrap());
    output.into_inner()
}
```

### 编译

Cargo 默认支持 arm 架构参数，在 Rust 项木目录下执行如下命令编译 so：
```
cargo build --target aarch64-linux-android --release
cargo build --target armv7-linux-androideabi --release
cargo build --target i686-linux-android --release
cargo build --target x86_64-linux-android --release
```

生成的 so 库位于 target目录下各个根据各个目标平台的目录下，例如 `target/aarch64-linux-android/Release/libtest_rust.so`。

将对应的平台的 so 拷贝到 Android `src/main/jniLibs`目录下 `armeabi-v7a`、`arm64-v8a`、`x86`、`x86_64`的对应目录。

添加 Kotlin 代码

```Kotlin
package tech.yaowen.androidrust

class Hello {
    companion object {
        // Used to load the 'native-lib' library on application startup.
        init {
            System.loadLibrary("test_rust")
        }
    }

    /**
     * A native method that is implemented by the 'native-lib' native library,
     * which is packaged with this application.
     */
    @Suppress("KotlinJniMissingFunction")
    external fun stringFromJNI(str: String): String


    @Suppress("KotlinJniMissingFunction")
    external fun callNativeFun(): String
}
```

## 方式二：直接在项目中编译

### Gradle 添加 Rust 编译 Task

在项目的 build.gradle 中添加：

```gradle
buildscript {
    repositories {
        maven {
            url "https://plugins.gradle.org/m2/"
        }
    }
    dependencies {
        classpath 'gradle.plugin.org.mozilla.rust-android-gradle:plugin:0.8.3'
    }
}
```
在 module 中的 build.gradle 的末尾添加

```gradle
android { ... }

apply plugin: 'org.mozilla.rust-android-gradle.rust-android'

cargo {
    module  = "./src/main/test_rust"       // Or whatever directory contains your Cargo.toml
    libname = "test_rust"          // Or whatever matches Cargo.toml's [package] name.
    targets = ["arm", "arm64", "x86", "x86_64"]  // See bellow for a longer list of options
}

afterEvaluate {
    // The `cargoBuild` task isn't available until after evaluation.
    android.libraryVariants.all { variant ->
        def productFlavor = ""
        variant.productFlavors.each {
            productFlavor += "${it.name.capitalize()}"
        }
        def buildType = "${variant.buildType.name.capitalize()}"
        tasks["generate${productFlavor}${buildType}Assets"].dependsOn(tasks["cargoBuild"])
    }
}
```
这是一个为 Rust 编写的 Android 的 Gradle 插件，用于编译 Rust。更多文档可以查看：https://github.com/mozilla/rust-android-gradle


### 创建源文件目录

Android Studio 的源码都在 main 目录下，创建 rust 目录。右键点击 main -> Open in Terminal

在打开的终端中输入：

```shell
cargo new <lib_name> --lib
```

指令中的 `lib_name` 表示目录名，你应该改成你实际库的名字，方便后期阅读。 `--lib` 表示创建一个库，而不是用于直接运行的程序。

在 Rust 库的根目录中有个 `Cargo.toml` 用于配置 Rust 库的依赖和库类型。添加如下配置

```
[dependencies]
# 添加 JNI 支持
jni = { version = "0.17.0", default-features = false }

[lib]
# 项目作为动态库使用
crate_type = ["cdylib"]
```

在同一目录下添加 `cargo-config.toml` 配置文件。文件内容跟上面配置 `so` 交叉工具链的 `cargo-config.toml` 文件一样。

Rust 代码和 Java 代码添加方法和上面一样，即可编译运行。


参考文档：

https://mozilla.github.io/firefox-browser-architecture/experiments/2017-09-21-rust-on-android.html
https://www.jianshu.com/p/6353cb691e6b?utm_campaign=hugo
https://blog.csdn.net/u012067469/article/details/104013445