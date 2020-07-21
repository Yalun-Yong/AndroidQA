### Android NDK: The armeabi ABI is no longer supported. Use armeabi-v7a.

ARM 处理器的版本

- armeabi-v7a: 第7代及以上的 ARM 处理器，使用硬件浮点运算。2010年起以后的生产的大部分Android设备都使用它.

- arm64-v8a: 第8代、64位ARM处理器，很少设备，三星 Galaxy S6是其中之一。
- armeabi: 第5代、第6代的ARM处理器，早期的手机用的比较多。使用软件浮点运算，通用性强，速度慢。
- x86: 平板、模拟器用得比较多。
- x86_64: 64位的平板。


[NDK 17 不再支持 armabi 和 mips，同时将默认的编译器从 gcc 改为了 clang](https://github.com/android/ndk/wiki/Changelog-r17)


[Adroid 4.0 (API 14)默认不再支持 armeabi。](https://www.jianshu.com/p/4b1c2dd3c87f)
[Android 4.4 (API 19)之后强制要求armv7处理器。](https://stackoverflow.com/questions/10920747/android-cpu-arm-architectures)


