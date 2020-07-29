# 构建已有的三方库

从 NDK 18 开始，NDK 移除了 gcc 编译器，只能使用 Clang 编译器编译。Clang 编译器有更好的错误提示和语法检查。

从 NDK 19 开始，NDK 中默认带有 `toolchains` 可供使用，与任意构建系统进行交互时不再需要使用 make_standalone_toolchain.py 脚本。

想要为自己的构建系统添加原生NDK支持的构建系统维护人员应该阅读[《构建系统维护人员指南》](https://android.googlesource.com/platform/ndk/+/master/docs/BuildSystemMaintainers.md)。


要针对不同的 CPU 架构进行编译，要么使用 Clang 时使用 `-targe` 传入对应的目标架构，要么使用对应目标前缀的 Clang 编译文件，例如编译 `minSdkVersion` 21 的 ARM 64 位安卓目标，可以使用以下任意合适的一种。

```shell
$ $NDK/toolchains/llvm/prebuilt/$HOST_TAG/clang++ \
    -target aarch64-linux-android21 foo.cpp
```

```
$ $NDK/toolchains/llvm/prebuilt/$HOST_TAG/aarch64-linux-android21-clang++ \
    foo.cpp
```

`$NDK` 替换为为安装 `NDK` 环境的路径。`$HOST_TAG` 替换为根据下表你下载的 NDK 平台而对应的不同路径：

| NDK OS Variant | Host Tag |
| ---- | ---- |
| macOS	| darwin-x86_64 |
| Linux	| linux-x86_64 |
| 32-bit | Windows	windows |
| 64-bit Windows | windows-x86_64 |

这里的前缀或目标参数的格式是目标三元组，带有表示 minSdkVersion 的后缀。该后缀仅与 clang/clang++ 一起使用；binutils 工具（例如 ar 和 strip）不需要后缀，因为它们不受 minSdkVersion 影响。Android 支持的目标三元组如下：

| ABI | 三元组 |
| ---- | ---- |
| armeabi-v7a | armv7a-linux-androideabi |
| arm64-v8a | aarch64-linux-android |
| x86 | i686-linux-android |
| x86-64 | x86_64-linux-android |

**注意：对于 32 位 ARM，编译器会使用前缀 armv7a-linux-androideabi，但 binutils 工具会使用前缀 arm-linux-androideabi。对于其他架构，所有工具的前缀都相同。**

许多构建脚本都仅接受 GCC 格式的交叉编译，每个编译器仅针对一个 `OS/架构` 组合，因此可能不能正确的处理 `-target`，因此更好的做法是使用目标前缀的 Clang 编译器。


## Autoconf

***注意：通常无法在 Windows 上构建 Autoconf 项目。Windows 用户可以使用适用于 Linux 的 Windows 子系统或 Linux 虚拟机来构建这些项目。***

Autoconf 使用项目目录下的 `configure` 配置编译参数。Autoconf 允许指定不同的参数来配置编译过程和裁剪编译目标，而且它允许使用环境变量指定 `toolchain`。

