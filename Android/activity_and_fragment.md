## onSaveInstanceState 调用情况

该Activity　又被转到后台时，可能被销毁，则可能会调用，这种可能性有这么几种情况：


1. 配置发生改变，例如选装屏幕，从新创建。
2. 内存不足时，后台优先级较低的 Activity 容易被销毁。

1.当用户按下Home键时；

2.长按Home键，选择运行其他的程序时；

3.按下电源键（关闭屏幕显示）时；

4.从Activity A启动一个新的Activity时；或者点击消息进入另一个 app 时。

5.屏幕方向切换时，如从横屏切换到竖屏；

6.电话打入等情况发生时；

## Activity 的启动流程

1. 最终都是调用　`startActivityForResult`

2. 内部调用 `Instrumentation#execStartActivity`

3. 内部调用 `ActivityTaskManager.getService().startActivity`。`ActivityTaskManager.getService()` 返回的是一个 aidl, 我们知道 aidl 会被生成一个接口，接口的实例会继承 IBinder. 由此就清楚了，启动一个 Activity 其实也是获取了一个 `Context.ACTIVITY_TASK_SERVICE` Service 的接口，然后执行启动 Activity 的功能的。 接着调用 checkStartActivityResult() 来检查启动结果，如果返回了错误码，就根据错误码抛出相应的异常。

4. 启动 Acitivity 的 Service 是 ` ActivityTaskManagerService extends IActivityTaskManager.Stub` 它已经是 Framwork 层的内容了。startActivity 最终会调用 `startActivityAsUser`

```Java
    int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
            boolean validateIncomingUser) {
        enforceNotIsolatedCaller("startActivityAsUser");

        userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
                Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

        // TODO: Switch to user app stacks here.
        return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
                .setCaller(caller)
                .setCallingPackage(callingPackage)
                .setResolvedType(resolvedType)
                .setResultTo(resultTo)
                .setResultWho(resultWho)
                .setRequestCode(requestCode)
                .setStartFlags(startFlags)
                .setProfilerInfo(profilerInfo)
                .setActivityOptions(bOptions)
                .setMayWait(userId)
                .execute();

```

而 `execute()` 的内部很简单，就是将之前设置的这些参数，直接调用了 `ActivityStarter#startActivityMayWait` 内部做了许多参数检车和数据记录后，直接调用了自己的 `startActivity`， `startActivity` 才是检查 `ActivityStack` 或是创建新栈的实际方法。



5. 最终，消息返回后，会调用 ApplicationThread 的  `H.performLaunchActivity` 来启动一个 Activity。  `performLaunchActivity` 主要实现了如下几步：
    1．从 `ActivityClientRecord` 中获取待启动的　`Activity` 的组件信息。
    2. 调用　`Instrumentation.newActivity` 加载类并创建实例
    3. 通过 `LoadedApk.makeApplication` 尝试获取或创建单例的 application 对象。
    4. ContextImpl.setOuterContext 和 Activity.attatch 来完成一些重要数据的初始化。如上下文。
    5. 然后调用 `Instrumentation.callActivityOnCreate` 完成了 Activity `onCreate`　的调用。


## 横竖屏切换和　A　启动　B　的生命周期调用不一样

屏幕切换　A.onPause -> A.onStop -> A.onDestroy -> A.onCreate -> A.onStart -> A.onResume

A 启动B  A.onPause -> B.onCreate->B.onStart ->   B.onResume -> A.onStop


## taskAffinity属性

https://www.jianshu.com/p/947b5fb28db4

taskAffinity 属性和Activity的启动模式息息相关，而且taskAffinity属性比较特殊，在普通的开发中也是鲜有遇到，但是在有些特定场景下却有着出其不意的效果。

taskAffinity是Activity在mainfest中配置的一个属性，暂时可以理解为：taskAffinity为宿主Activity指定了存放的任务栈[不同于App中其他的Activity的栈]，为activity设置taskAffinity属性时不能和包名相同，因为Android团队为taskAffinity默认设置为包名任务栈。

taskAffinity只有和SingleTask启动模式匹配使用时，启动的Activity才会运行在名字和taskAffinity相同的任务栈中。


## Fragment 的优点

1. 模块化，把不同块的代码放在不同的 Fragment 中，防止 Activity　中的代码堆积。

2. 重用，多个 Activity 的相同部分可以使用同一个　Fragment。

3. 适配：根据不同的屏幕尺寸，先是不同布局的页面，可以组合　Fragment 体验更好，实现简单。

4. 其他一些不太常用的：sharedPreference, 静态变量，file(数据库也算了), 广播，EventBus。

## Fragment 之间进行通信

这个说法很奇怪，应该说传递数据的方式更确切一些。

1. `getActivity` 获取所在的 Activity，然后调用它的　`findFragmentById` 即可获取到另一个　Fragment 进行调用。可以实现接口或者调用它的公共方法，但是这样不要的就是绑定比较死，很难实现逻辑的解耦。

2. 通过公用　LiveData 对象来获取数据。

3. Fragment 1.3 开始，FragmentManager 都实现了 `FragmentResultOwner` 接口用户传输数据。 可以通过　`setResultListener(key: String,  listener)` 设置一个监听器。然后通过 `setResultListener(key, bindle)` 发送数据

```Kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    // Use the Kotlin extension in the fragment-ktx artifact
    setResultListener("requestKey") { key, bundle ->
        // We use a String here, but any type that can be put in a Bundle is supported
        val result = bundle.getString("bundleKey")
        // Do something with the result...
    }
}

// 发送数据

button.setOnClickListener {
    val result = "result"
    // Use the Kotlin extension in the fragment-ktx artifact
    setResult("requestKey", bundleOf("bundleKey" to result))
}

```

如需将结果从子级 Fragment 传递到父级 Fragment，父级 Fragment 在调用 setFragmentResultListener() 时应使用 getChildFragmentManager() 而不是 getParentFragmentManager()。

```
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    // We set the listener on the child fragmentManager
    childFragmentManager.setResultListener("requestKey") { key, bundle ->
        val result = bundle.getString("bundleKey")
        // Do something with the result..
    }
}
```

## Activity 和　Fragment 的数据传输方式

跟 Fragment 之间传递数据一样，多了一个调用　Fragment 的　`setArgument` 方法传递数据的方式。但是需要注意的是，设置数据必须在 `onStart 之前调用`。

推荐使用　LiveData 和　setArgument 的方式，能够在重建时回复数据。不过这是个伪命题，其他方法也可以在重建的时候恢复数据，只是有没有在重建的时候写相应的逻辑。

官方组件ViewModel如何实现数据恢复，原理是什么？

ViewModel 只能在 config 变化引起重建的时候恢复,在内存不够等情况下 被销毁的时候并不能恢复,
原理请百度 onRetainNoConfigInstance,如果我方法名没写错的话=.=

内存不够的ondestory这种情况会执行onClear() clear掉, 配置变化得ondestory不会的



## 进程保活

在确定进程保活之前，首要要想为什么要保活，是否真的有保活的必要，用户需要该应用的时候，点击应用即可。接收通知，可以消息推送。为了做出对用户友好的软件，就要节省用户的资源，例如电量，网络数据。除了要自己做消息推送，其实并没有什么保活的必要。首先要考虑向系统注册监听器或定时的方法，而不是保活。

> 方法包括，1像素Activity，前台服务，账号同步，Jobscheduler,相互唤醒

https://juejin.im/entry/58acf391ac502e007e9a0a11

## Service的运行线程（生命周期方法全部在主线程）



## Service启动方式以及如何停止

startService  启动的(调用它的 onＳtart 和　onStartCommand)，调用多次startService，onCreate只有第一次会被执行，而onStartCommand会执行多次。可以通过在内部调用 `stopSelf` 或者外部调用　`stopService` 停止。生命周期执行onDestroy方法，并且多次调用stopService时，onDestroy只有第一次会被执行。

bindService 启动的，`unbindServide`，在所有绑定都解绑后，就会停止。

## ServiceConnection里面的回调方法运行在哪个线程？

在 Bindle 的线程池中获取的子线程中。因此想要更新 UI 需要切换线程。


## Application 声明周期的监听

https://mp.weixin.qq.com/s?__biz=MzAxMTI4MTkwNQ==&mid=2650831824&idx=2&sn=38ab22ab34993cf757efbbda343df6a5&chksm=80b7af4eb7c026580b2cd0dca5697b8ec661fd4a2dc9295e1952709dc4c765b46ba809df1b1b&mpshare=1&scene=1&srcid=0706fPVK7AAvWzCPaGecF1nE&sharer_sharetime=1594000246125&sharer_shareid=8697bdac93dcc42338af5348abed4ea0&exportkey=ATQLE2L%2F%2BCHhst%2BJyYYnCDE%3D&pass_ticket=iiylolXlHhc%2B0whn2ZRq9btgOPDoK6UgHC3bwqdTCSXoDEze1CRJZxmt9egpxrnG&wx_header=0#rd

2. 维护 Activity 栈来基数，要求所有的 Activity 继承 BaseActivity。繁琐而限制比较多。



## BroadcastReceiver 与 LocalBroadcastReceiver 有什么区别？

BroadcastReceiver是针对应用间、应用与系统间、应用内部进行通信的一种方式
LocalBroadcastReceiver仅在自己的应用内发送接收广播，也就是只有自己的应用能收到，数据更加安全广播只在这个程序里，而且效率更高。

BroadcastReceiver 使用

1.制作intent（可以携带参数）
2.使用sendBroadcast()传入intent;
3.制作广播接收器类继承BroadcastReceiver重写onReceive方法（或者可以匿名内部类啥的）
4.在java中（动态注册）或者直接在Manifest中注册广播接收器（静态注册）使用registerReceiver()传入接收器和intentFilter
5.取消注册可以在OnDestroy()函数中，unregisterReceiver()传入接收器
LocalBroadcastReceiver 使用
1.LocalBroadcastReceiver不能静态注册，只能采用动态注册的方式。
在发送和注册的时候采用，LocalBroadcastManager的sendBroadcast方法和registerReceiver方法


BroadcastReceiver 是跨应用广播，利用Binder机制实现，支持动态和静态两种方式注册方式。
LocalBroadcastReceiver 是应用内广播，利用Handler实现，利用了IntentFilter的match功能，提供消息的发布与接收功能，实现应用内通信，效率和安全性比较高，仅支持动态注册。

BroadcastReceiver（广播接收者）：是跨应用广播，利用Binder机制实现。
静态注册：
(1)在清单文件中，通过标签声明；
(2)在Android3.1开始，对于接收系统广播的BroadcastReceiver，App进程退出后，无法接收到广播；对于自定义的广播，可以通过重写flag的值，使得即使App进程退出，仍然可以接收到广播。
(3)静态注册的广播是由PackageManagerService负责。

动态注册：1.在代码中注册，程序运行的时候才能进行；2.跟随组件的生命周期；3.动态注册的广播是由AMS(ActivityManagerService)负责的。
注意：对于动态注册，最好在Activity的onResume（）中注册，在onPause（）中注销。在系统内存不足时，onStop()、onDestory()可能不会执行App就被销毁，onPause（）在App销毁前一定会被执行，保证广播在App销毁前注销。

LocalBroadcastManager实现原理：是应用内广播，利用Handler实现。
1.使用了单例模式，并且将外部传入的Context转换成了Application的Context，避免造成内存泄露。
2.在构造方法中创建了Handler，实质是通过Handler进行发送和接受消息的。
3.创建Handler时，传入了主线程的Looper，说明这个Handler是在主线程创建的，即广播接收者是在主线程接收消息的，所以不能在onReceiver（）中做耗时操作。
注意：对于LocalBroadcastManager发送的广播，只能通过LocalBroadcastManager动态注册，不能静态注册。

特别注意：
1.如果BroadcastReceiver在onReceiver（）方法中在10秒内没有执行完成，会造成ANR异常。
2.对于不同注册方式的广播接收者回调方法onReceive（）返回的Context是不一样的。
静态注册：context为ReceiverRestrictedContext。
动态注册：context为Activity的Context。
LocalBroadcastManager的动态注册：context为Application的Context。

