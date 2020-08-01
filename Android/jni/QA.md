1. Android 不会挂起执行原生(Native)代码的线程。如果正在进行垃圾回收，或者调试程序已发出挂起请求，则在线程下次调用 JNI 时，Android 会将其挂起。(不是应当结束 JNI 调用后，或者在调用 JNI 前挂起？ ) （https://developer.android.com/training/articles/perf-jni?hl=zh-cn） 

2. 在取消加载类之前，类引用、字段 ID 和方法 ID 保证有效。只有在与 ClassLoader 关联的所有类可以进行垃圾回收时，系统才会取消加载类，这种情况很少见，但在 Android 中并非不可能。但请注意，jclass 是类引用，必须通过调用 NewGlobalRef 来保护它（请参阅下一部分）。 什么意思？

在执行 ID 查找的 C/C++ 代码中创建 nativeClassInit 方法。初始化类时，该代码会执行一次。如果要取消加载类之后再重新加载，该代码将再次执行。 创建的是本地方法名是固定的吗？