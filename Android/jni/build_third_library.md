# 构建已有的三方库

从 NDK 18 开始，NDK 移除了 gcc 编译器，只能使用 Clang 编译器编译。Clang 编译器有更好的错误提示和语法检查。

从 NDK 19 开始，NDK 中默认带有 `toolchains` 可供使用，与任意构建系统进行交互时不再需要使用 make_standalone_toolchain.py 脚本。如果是 NDK 19 之前的版本，请查看 [NDK 18 及之前编译](https://developer.android.com/ndk/guides/standalone_toolchain)。

想要为自己的构建系统添加原生NDK支持的构建系统维护人员应该阅读[《构建系统维护人员指南》](https://android.googlesource.com/platform/ndk/+/master/docs/BuildSystemMaintainers.md)。


要针对不同的 CPU 架构进行编译，要么使用 Clang 时使用 `-targe` 传入对应的目标架构，要么使用对应目标前缀的 Clang 编译文件，例如编译 `minSdkVersion` 21 的 ARM 64 位安卓目标，可以使用以下任意合适的一种。

```shell
$ $NDK/toolchains/llvm/prebuilt/$HOST_TAG/bin/clang++  -target aarch64-linux-android21 foo.cpp
```

```shell
$ $NDK/toolchains/llvm/prebuilt/$HOST_TAG/bin/aarch64-linux-android21-clang++ foo.cpp
```

`$NDK` 替换为为安装 `NDK` 环境的路径。`$HOST_TAG` 替换为根据下表你下载的 NDK 平台而对应的不同路径：

| NDK OS Variant | Host Tag |
| ---- | ---- |
| macOS	| darwin-x86_64 |
| Linux	| linux-x86_64 |
| 32-bit Windows |	windows |
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


- --arch 参数是必填项，但 API 级别将默认设为指定架构支持的最低级别（目前，级别 16 适用于 32 位架构，级别 21 适用于 64 位架构）。

- 自 r18 开始，所有独立工具链都使用 Clang 和 libc++。除非构建静态可执行文件，否则将默认使用 libc++ 共享库。如需强制使用静态库，请在链接时传递 -static-libstdc++。此行为与普通主机工具链的行为相符。

- 如 C++ 库支持中所提到的那样，在链接到 libc ++ 时通常需要传递 -latomic。


## Autoconf

***注意：通常无法在 Windows 上构建 Autoconf 项目。Windows 用户可以使用适用于 Linux 的 Windows 子系统或 Linux 虚拟机来构建这些项目。***

Autoconf 使用项目目录下的 `configure` 配置编译参数。Autoconf 允许指定不同的参数来配置编译过程和裁剪编译目标，而且它允许使用环境变量指定 `toolchain`。

# NDK 18 及之前编译

`make_standalone_toolchain.py` 用于替换之前的 `make-standalone-toolchain.sh` 用于在 windows 不用配置 bash 环境也能编译，但是实际情况是许多使用 `Autoconf` 配置编译的三方库仍旧无法在 Windows 上编译。

`make_standalone_toolchain.py` 不再接收 `--abis` 参数，因为 `NDK 17` 开始就不再支持 `armabi` 了， 通过 `archs` 即可区分。而为了让老版本的写的编译脚本在版本 NDK 也能运行，`make-standalone-toolchain.sh` 最终也是调用 `make_standalone_toolchain.py` 来处理。虽然接收  `--abis` 参数，但却什么也没做，应该是为了兼容老版本的脚本运行。

高版本的 NDK（应该是 19 开始） 虽然 `make_standalone_toolchain.py` 仍在，由于 NDK 自带的 toolchain 已可用，基本都是拷贝 `$NDK_HOME/toolchains/llvm/prebuilt/darwin-x86_64/` 目录下的内容。 `sysroot` 也使用的是这个目录下的。 `$NDK_HOME` 目录下也有个 `sysroot`, 估计也是用来兼容老的编译脚本，冗余的。


从使用 `make_standalone_toolchain.py` 的警告信息中可以分析出两个目录基本一致。

```
$NDK/build/tools/make_standalone_toolchain.py \
    --arch arm --api 16 --install-dir /tmp/my-android-toolchain
```

`--install-dir` 将直接生成在指定目录，如果使用该参数，将生成一个压缩包，并且可以使用 `--package-dir` 指定压缩包的位置，方便解压到任意目录。

```
WARNING:__main__:make_standalone_toolchain.py is no longer necessary. The
$NDK/toolchains/llvm/prebuilt/darwin-x86_64/bin directory contains target-specific scripts that perform
the same task. For example, instead of:

    $ python $NDK/build/tools/make_standalone_toolchain.py \
        --arch arm --api 16 --install-dir toolchain
    $ toolchain/bin/clang++ src.cpp

Instead use:

    $ $NDK/toolchains/llvm/prebuilt/darwin-x86_64/bin/armv7a-linux-androideabi16-clang++ src.cpp
```