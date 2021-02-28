---
layout: posts
title: 绑定服务
date: 2020-12-14 19:18:41
tags: ["Android"]
cover: https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/blog/images/cover/千与千寻05.png
---

绑定服务是 `Service` 类的实现，可让其他应用与其进行绑定和交互。如要为服务提供绑定，您必须实现 `onBind()` 回调方法。该方法会返回 `IBinder` 对象，该对象定义的编程接口可供客户端用来与服务进行交互。

客户端通过调用 `bindService()` 绑定到服务。调用时，它必须提供 `ServiceConnection` 的实现，后者会监控与服务的连接。`bindService()` 的返回值表明所请求的服务是否存在，以及是否允许客户端访问该服务。当创建客户端与服务之间的连接时，Android 系统会调用 `ServiceConnection` 上的 `onServiceConnected()`。`onServiceConnected()` 方法包含 `IBinder` 参数，客户端随后会使用该参数与绑定服务进行通信。

您可以**同时将多个客户端连接到服务**。但是，系统会缓存 `IBinder` 服务通信通道。换言之，只有在第一个客户端绑定服务时，系统才会调用服务的 `onBind()` 方法来生成 `IBinder`。然后，系统会将同一 `IBinder` 传递至绑定到相同服务的所有其他客户端，无需再次调用 `onBind()`。

当**最后一个客户端取消与服务的绑定时**，系统会销毁服务（除非 `startService()` 也启动了该服务）。

针对您的绑定服务实现，其最重要的环节是定义 `onBind()` 回调方法所返回的接口。

## 绑定到服务

应用组件（客户端）可通过调用 `bindService()` 绑定到服务。然后，Android 系统会调用服务的 `onBind()` 方法，该方法会返回用于与服务交互的 `IBinder`。

绑定为异步操作，并且 `bindService()` *无需*将 `IBinder` 返回至客户端即可立即返回。如要接收 `IBinder`，客户端必须创建一个 `ServiceConnection` 实例，并将其传递给 `bindService()`。`ServiceConnection` 包含一个回调方法，系统通过调用该方法来传递 `IBinder`。

如要从您的客户端绑定到服务，请执行以下步骤：

1. 实现 `ServiceConnection`。

   您的实现必须重写两个回调方法：

   - `onServiceConnected()`

     系统会调用该方法，进而传递服务的 `onBind()` 方法所返回的 `IBinder`。

   - `onServiceDisconnected()`

     当与服务的连接意外中断（例如服务崩溃或被终止）时，Android 系统会调用该方法。**当客户端取消绑定时，系统不会调用该方法**。

2. 调用 `bindService()`，从而传递 `ServiceConnection` 实现。

3. 当系统调用 `onServiceConnected()` 回调方法时，您可以使用接口定义的方法开始调用服务。

4. 如要断开与服务的连接，请调用 `unbindService()`。

   当应用销毁客户端时，如果该客户端仍与服务保持绑定状态，则该销毁会导致客户端取消绑定。更好的做法是在客户端与服务交互完成后，立即取消与该客户端的绑定。这样可以关闭空闲服务。如需详细了解有关绑定和取消绑定的适当时机，请参阅[附加说明](https://developer.android.com/guide/components/bound-services#Additional_Notes)。

以下示例通过[扩展 Binder 类](https://developer.android.com/guide/components/bound-services#Binder)将客户端连接到上文创建的服务，因此它只需将返回的 `IBinder` 转换为 `LocalService` 类并请求 `LocalService` 实例：

```kotlin
var mService: LocalService

val mConnection = object : ServiceConnection {
    // Called when the connection with the service is established
    override fun onServiceConnected(className: ComponentName, service: IBinder) {
        // Because we have bound to an explicit
        // service that is running in our own process, we can
        // cast its IBinder to a concrete class and directly access it.
        val binder = service as LocalService.LocalBinder
        mService = binder.getService()
        mBound = true
    }

    // Called when the connection with the service disconnects unexpectedly
    override fun onServiceDisconnected(className: ComponentName) {
        Log.e(TAG, "onServiceDisconnected")
        mBound = false
    }
}
```

如以下示例所示，客户端可将此 `ServiceConnection` 传递至 `bindService()`，从而绑定到服务：

```kotlin
Intent(this, LocalService::class.java).also { intent ->
    bindService(intent, connection, Context.BIND_AUTO_CREATE)
}
```

* `bindService()` 的第一个参数是一个 `Intent`，用于显式命名要绑定的服务。
* 第二个参数是 `ServiceConnection` 对象。
* 第三个参数是指示绑定选项的标记。如要创建尚未处于活动状态的服务，此参数应为 `BIND_AUTO_CREATE`。其他可能的值为 `BIND_DEBUG_UNBIND` 和 `BIND_NOT_FOREGROUND`，或者 `0`（表示无此参数）。



## 参考

* [官方文档：绑定服务概览](https://developer.android.com/guide/components/bound-services)