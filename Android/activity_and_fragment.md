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



5. 最终，消息返回后，会调用 ApplicationThread 的 `performLaunchActivity` 来启动一个 Activity。  `performLaunchActivity` 主要实现了如下几步：
    1．从 `ActivityClientRecord` 中获取待启动的　`Activity` 的组件信息。
    2. 调用　`Instrumentation.newActivity` 加载类并创建实例
    3. 通过 `LoadedApk.makeApplication` 尝试获取或创建单例的 application 对象。
    4. ContextImpl.setOuterContext 和 Activity.attatch 来完成一些重要数据的初始化。如上下文。
    5. 然后调用 `Instrumentation.callActivityOnCreate` 完成了 Activity `onCreate`　的调用。


## 横竖屏切换和　A　启动　B　的生命周期调用不一样

屏幕切换　A.onPause -> A.onStop -> A.onDestroy -> A.onCreate -> A.onStart -> A.onResume

A 启动B  A.onPause -> B.onCreate->B.onStart ->   B.onResume -> A.onStop


## taskAffinity属性

taskAffinity 属性和Activity的启动模式息息相关，而且taskAffinity属性比较特殊，在普通的开发中也是鲜有遇到，但是在有些特定场景下却有着出其不意的效果。

taskAffinity是Activity在mainfest中配置的一个属性，暂时可以理解为：taskAffinity为宿主Activity指定了存放的任务栈[不同于App中其他的Activity的栈]，为activity设置taskAffinity属性时不能和包名相同，因为Android团队为taskAffinity默认设置为包名任务栈。

taskAffinity只有和SingleTask启动模式匹配使用时，启动的Activity才会运行在名字和taskAffinity相同的任务栈中。


## Fragment 的优点

1. 模块化，把不同块的代码放在不同的 Fragment 中，防止 Activity　中的代码堆积。

2. 重用，多个 Activity 的相同部分可以使用同一个　Fragment。

3. 适配：根据不同的屏幕尺寸，先是不同布局的页面，可以组合　Fragment 体验更好，实现简单。


## Fragment 之间进行通信

1. `getActivity` 获取所在的 Activity，然后调用它的　`findFragmentById` 即可获取到另一个　Fragment 进行调用。可以实现接口或者调用它的公共方法，但是这样不要的就是绑定比较死，很难实现逻辑的解耦。

2. 更好的方法是调用相同的 ViewModel　然后获取相同的　`LiveData` 来共享数据。
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





