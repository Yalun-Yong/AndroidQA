Android系统中，ActivityManagerService(简称AMS)和WindowManagerService(简称WMS)会检测App的响应时间，如果App在特定时间无法响应屏幕触摸或键盘输入时间，或者特定事件没有处理完毕，就会出现ANR。


## 出现场景



> 以下四种情况

- InputDispatching Timeout：应用5秒内未响应用户的输入事件，如屏幕触摸事件或键盘输入事件。
    - 主线程被IO操作（从4.0之后网络IO不允许在主线程中）阻塞。
    - 主线程中存在耗时的计算
    - 主线程中错误的操作，比如Thread.wait或者Thread.sleep等
    - Android系统会监控程序的响应状况，一旦出现下面两种情况，则弹出ANR对话框
    - 主线程与另一线程发生死锁，无论是在进程中还是 binder 间调用。

- BroadcastQueue Timeout ：在执行前台广播（BroadcastReceiver）的 onReceive()函数时10秒没有处理完成，后台为60秒。
    - BroadcastReceiver未在10秒内完成相关的处理

- Service Timeout ：前台服务20秒内，后台服务在200秒内没有执行完毕。

- ContentProvider Timeout ：ContentProvider的publish在10s内没进行完。



## 如何避免

基本的思路就是将IO操作在工作线程来处理，减少其他耗时操作和错误操作

- 使用 AsyncTask/WorkManager 处理耗时IO操作。
- 使用Thread或者HandlerThread时，调用Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND)设置优先级，否则仍然会降低程序响应，因为默认Thread的优先级和主线程相同。
- 使用Handler处理工作线程结果，而不是使用Thread.wait()或者Thread.sleep()来阻塞主线程。
- Activity的onCreate和onResume回调中尽量避免耗时的代码
- BroadcastReceiver中onReceive代码也要尽量减少耗时，建议使用IntentService处理。

## 如何定位

如果开发机器上出现问题，我们可以通过查看 `/data/anr/traces.txt` 即可

导出文件

```
adb pull /data/anr/traces.txt
```

**特别注意：产生新的ANR，原来的 traces.txt 文件会被覆盖。**

对于 BroadcastReceiver 的 ANR 默认不提示，被称为 `background ANR`. 但是可以在 `traces` 文件中找到。

提问:可以更容易了解background ANR么？

回答:当然可以，在Android开发者选项—>高级—>显示所有”应用程序无响应“勾选即可对后台ANR也进行弹窗显示，方便查看了解程序运行情况。


## 原因分析

https://www.jianshu.com/p/388166988cef

## CrashHandler 处理没有捕获的异常。
