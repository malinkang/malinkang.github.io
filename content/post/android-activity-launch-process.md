---
title: "Activity的启动流程"
date: 2019-09-24T15:53:26+08:00
draft: false
---

App启动的整体流程分为以下7个阶段。

1. `Launcher`通知AMS，要启动`App`，而且指定要启动`App`的哪个页面（也就是首页）。
2. `AMS`通知Launcher, “好了我知道了，没你什么事了”。同时，把要启动的首页记下来。
3. Launcher当前页面进入Paused状态，然后通知AMS, “我睡了，你可以去找App了”。
4. AMS检查App是否已经启动了。是，则唤起App即可。否，就要启动一个新的进程。AMS在新进程中创建一个ActivityThread对象，启动其中的main函数。
5. App启动后，通知AMS, “我启动好了”。
6. AMS翻出之前在2中存的值，告诉App，启动哪个页面。
7. App启动首页，创建Context并与首页Activity关联。然后调用首页`Activity`的onCreate函数。

至此启动流程完成，可分成两部分：第1～3阶段，Launcher和AMS相互通信；第4～7阶段，App和AMS相互通信。

<!--more-->

[ActivityManagerService](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/marshmallow-release/services/core/java/com/android/server/am/ActivityManagerService.java)

## 第1阶段：Launcher通知AMS

点击图标会调用[Launcher](http://androidxref.com/6.0.0_r1/xref/packages/apps/Launcher3/src/com/android/launcher3/Launcher.java) 的`startAppShortcutOrInfoActivity`方法


```java
@Thunk void startAppShortcutOrInfoActivity(View v) {
    Object tag = v.getTag();
    final ShortcutInfo shortcut;
    final Intent intent;
    if (tag instanceof ShortcutInfo) {
        shortcut = (ShortcutInfo) tag;
        intent = shortcut.intent;
        int[] pos = new int[2];
        v.getLocationOnScreen(pos);
        intent.setSourceBounds(new Rect(pos[0], pos[1],
                pos[0] + v.getWidth(), pos[1] + v.getHeight()));

    } else if (tag instanceof AppInfo) {
        shortcut = null;
        intent = ((AppInfo) tag).intent;
    } else {
        throw new IllegalArgumentException("Input must be a Shortcut or AppInfo");
    }

    boolean success = startActivitySafely(v, intent, tag);
    mStats.recordLaunch(v, intent, shortcut);

    if (success && v instanceof BubbleTextView) {
        mWaitingForResume = (BubbleTextView) v;
        mWaitingForResume.setStayPressed(true);
    }
}
```

```java
@Thunk boolean startActivitySafely(View v, Intent intent, Object tag) {
    boolean success = false;
    if (mIsSafeModeEnabled && !Utilities.isSystemApp(this, intent)) {
        Toast.makeText(this, R.string.safemode_shortcut_error, Toast.LENGTH_SHORT).show();
       return false;
   }
   try {
       success = startActivity(v, intent, tag);
   } catch (ActivityNotFoundException e) {
       Toast.makeText(this, R.string.activity_not_found, Toast.LENGTH_SHORT).show();
        Log.e(TAG, "Unable to launch. tag=" + tag + " intent=" + intent, e);
    }
    return success;
}
```

[Activity](http://androidxref.com/6.0.0_r1/xref/frameworks/base/core/java/android/app/Activity.java)的`startActivityForResult`方法

```java
public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
    if (mParent == null) {
      //mMainThread是一个ActivityThread对象
      //通过ActivityThread对象的getApplicationThread方法获取一个Binder对象
      //这个对象的类型是ApplicationThread代表了Launcher所在的线程
      //mToken也是一个Binder对象，代表Launcher这个Activity也通过Instrumentation传给AMS
     
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);
        if (ar != null) {
            mMainThread.sendActivityResult(
                mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                ar.getResultData());
        }
        if (requestCode >= 0) {
            // If this start is requesting a result, we can avoid making
            // the activity visible until the result is received.  Setting
            // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
           // activity hidden during this time, to avoid flickering.
           // This can only be done when a result is requested because
           // that guarantees we will get information back when the
           // activity is finished, no matter what happens to it.
           mStartedActivity = true;
       }

       cancelInputsAndStartExitTransition(options);
       // TODO Consider clearing/flushing other event sources and events for child windows.
   } else {
       if (options != null) {
           mParent.startActivityFromChild(this, intent, requestCode, options);
       } else {
           // Note we want to go through this method for compatibility with
           // existing applications that may have overridden it.
           mParent.startActivityFromChild(this, intent, requestCode);
       }
   }
}

```

Instrumentation类的[execStartActivity](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/marshmallow-release/core/java/android/app/Instrumentation.java#1481)方法

```java
public ActivityResult execStartActivity(
         Context who, IBinder contextThread, IBinder token, Activity target,
         Intent intent, int requestCode, Bundle options) {
     IApplicationThread whoThread = (IApplicationThread) contextThread;
     Uri referrer = target != null ? target.onProvideReferrer() : null;
     if (referrer != null) {
         intent.putExtra(Intent.EXTRA_REFERRER, referrer);
     }
     if (mActivityMonitors != null) {
         synchronized (mSync) {
             final int N = mActivityMonitors.size();
             for (int i=0; i<N; i++) {
         
                 final ActivityMonitor am = mActivityMonitors.get(i);
                 if (am.match(who, null, intent)) {
                     am.mHits++;
                     if (am.isBlocking()) {
                         return requestCode >= 0 ? am.getResult() : null;
                     }
                     break;
                 }
             }
         }
     }
     try {
         intent.migrateExtraStreamToClipData();
         intent.prepareToLeaveProcess();
             //借助Instrumentation把数据传递给ActivityManagerNative
         int result = ActivityManagerNative.getDefault()
             .startActivity(whoThread, who.getBasePackageName(), intent,
                     intent.resolveTypeIfNeeded(who.getContentResolver()),
                     token, target != null ? target.mEmbeddedID : null,
                     requestCode, 0, null, options);
         checkStartActivityResult(result, intent);
     } catch (RemoteException e) {
         throw new RuntimeException("Failure from system", e);
     }
     return null;
}
```

`ServiceManager`是一个容器类。`AMN`通过`getDefault`方法，从`ServiceManager`中取得一个名为`activity`的对象，然后把它包装成一个`ActivityManagerProxy`（AMP）对象。

```java
private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
  protected IActivityManager create() {
      IBinder b = ServiceManager.getService("activity");
      if (false) {
          Log.v("ActivityManager", "default service binder = " + b);
      }
      IActivityManager am = asInterface(b);
      if (false) {
          Log.v("ActivityManager", "default service = " + am);
      }
      return am;
  }
};
//AMN的getDefault

```

`AMN`的`getDefault`方法返回类型为`IActivityManager`，而不是`AMP `。

```java
static public IActivityManager getDefault() {
      return gDefault.get();
}
```

## 第2阶段：AMS处理Launcher传过来的信息

`ActivityManagerProxy `的`startActivity`方法

```java
public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
        String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
   Parcel data = Parcel.obtain();
   Parcel reply = Parcel.obtain();
   data.writeInterfaceToken(IActivityManager.descriptor);
   data.writeStrongBinder(caller != null ? caller.asBinder() : null);
   data.writeString(callingPackage);
   intent.writeToParcel(data, 0);
   data.writeString(resolvedType);
   data.writeStrongBinder(resultTo);
   data.writeString(resultWho);
   data.writeInt(requestCode);
   data.writeInt(startFlags);
   if (profilerInfo != null) {
       data.writeInt(1);
       profilerInfo.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
   } else {
       data.writeInt(0);
   }
   if (options != null) {
       data.writeInt(1);
       options.writeToParcel(data, 0);
   } else {
       data.writeInt(0);
   }
   mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
   reply.readException();
   int result = reply.readInt();
   reply.recycle();
   data.recycle();
   return result;
}
```

那么当`AMS`想给Launcher发消息，又该怎么办呢？前面不是把`Launcher`以及它所在的进程给传过来了吗？它在AMS这边保存为一个ActivityRecord对象，这个对象里面有一个`ApplicationThreadProxy`，顾名思义，这就是一个`Binder`代理对象。它的`Binder`真身，也就是`ApplicationThread`。

`AMS`通过`ApplicationThreadProxy`发送消息，而`App`端则通过`ApplicationThread`来接收这个消息。

`ApplicationThread`接收到来自AMS的消息后，调用ActivityThread的sendMessage方法，向Launcher的主线程消息队列发送一个PAUSE_ACTIVITY消息。

## 第3阶段：Launcher去休眠，然后通知AMS:“我真的已经睡了”

[ActivityThread](http://androidxref.com/6.0.0_r1/xref/frameworks/base/core/java/android/app/ActivityThread.java)中的`H`的类

```java
public class H extends Handler {
   //...
    //处理AMS发送的消息
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case PAUSE_ACTIVITY:
                //...
                handlePauseActivity((IBinder) msg.obj, false, (msg.arg1 & 1) != 0, msg.arg2,
                        (msg.arg1 & 2) != 0);
                maybeSnapshot();
                //...
                break;
        }
    }
  //...
}
```

`ActivityThread`的`handlePauseActivity`方法



```java
private void handlePauseActivity(IBinder token, boolean finished,
        boolean userLeaving, int configChanges, boolean dontReport) {
    ActivityClientRecord r = mActivities.get(token);
    if (r != null) {
        //Slog.v(TAG, "userLeaving=" + userLeaving + " handling pause of " + r);
        if (userLeaving) {
            performUserLeavingActivity(r);
        }

        r.activity.mConfigChangeFlags |= configChanges;
        performPauseActivity(token, finished, r.isPreHoneycomb());

        // Make sure any pending writes are now committed.
        if (r.isPreHoneycomb()) {
            QueuedWork.waitToFinish();
        }

        // Tell the activity manager we have paused.
        if (!dontReport) {
            try {
                ActivityManagerNative.getDefault().activityPaused(token);
            } catch (RemoteException ex) {
            }
        }
        mSomeActivitiesChanged = true;
    }
}
```

`ActivityThread`里面有一个`mActivities`集合，保存当前App也就是Launcher中所有打开的Activity，把它找出来，让它休眠。

通过AMP通知AMS, “我真的休眠了。”
你可能会找不到H和APT这两个类文件，那是因为它们都是ActivityThread的内嵌类。

## 第4阶段：AMS启动新的进程

AMS接下来要启动App的首页，因为App不在后台进程中，所以要启动一个新的进程。这里调用的是[Process.start](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/marshmallow-release/services/core/java/com/android/server/am/ActivityManagerService.java#3320)方法，并且指定了ActivityThread的main函数为入口函数。代码如下：

```java
// Start the process.  It will either succeed and return a result containing
// the PID of the new process, or else throw a RuntimeException.
boolean isActivityProcess = (entryPoint == null);
if (entryPoint == null) entryPoint = "android.app.ActivityThread";
Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "Start proc: " +
        app.processName);
checkTime(startTime, "startProcess: asking zygote to start proc");
Process.ProcessStartResult startResult = Process.start(entryPoint,
        app.processName, uid, uid, gids, debugFlags, mountExternal,
        app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
        app.info.dataDir, entryPointArgs);
checkTime(startTime, "startProcess: returned from zygote!");
Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
```

## 第5阶段：新的进程启动，以ActivityThread的main函数作为入口

在启动新进程的时候，为这个进程创建ActivityThread对象，这就是我们耳熟能详的主线程（UI线程）。
创建好UI线程后，立刻进入ActivityThread的main函数，接下来要做两件具有重大意义的事情：

1. 创建一个主线程Looper，也就是MainLooper。注意，MainLooper就是在这里创建的。
2. 创建Application。注意，Application是在这里生成的。

主线程在收到BIND_APPLICATION消息后，根据传递过来的ApplicationInfo创建一个对应的LoadedApk对象（标志当前APK信息），然后创建ContextImpl对象（标志当前进程的环境），紧接着通过反射创建目标Application，并调用其attach方法，将ContextImpl对象设置为目标Application的上下文环境，最后调用Application的onCreate函数，做一些初始工作。
App开发人员对Application非常熟悉，因为我们可以在其中写代码，进行一些全局的控制，所以我们通常认为Application是掌控全局的，其实Application的地位在App中并没有那么重要，它就是一个Context上下文，仅此而已。
App中的灵魂是ActivityThread，也就是主线程，只是这个类对于App开发人员是访问不到的，但使用反射是可以修改这个类的一些行为的。
创建新App的最后就是告诉AMS“我启动好了”，同时把自己的ActivityThread对象发送给AMS。从此以后，AMS的电话簿中就多了这个新的App的登记信息，AMS以后就通过这个ActivityThread对象，向这个App发送消息。

## 第6阶段：AMS告诉新App启动哪个Activity

AMS把传入的ActivityThread对象转为一个ApplicationThread对象，用于以后和这个App跨进程通信。还记得APT和ATP的关系吗？参见图2-7。
在第1阶段，Launcher通知AMS，要启动斗鱼App的哪个Activity。在第2阶段，这个信息被AMS存下来。在第6阶段，AMS从过去的记录中翻出来要启动哪个Activity，然后通过ATP告诉App。

## 第7阶段：启动首页Activity

在Binder的另一端，App通过APT接收到AMS的消息，仍然在H的handleMessage方法的switch语句中处理，只不过，这次消息的类型是LAUNCH_ACTIVITY：

```java
case LAUNCH_ACTIVITY: {
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
    final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
    r.packageInfo = getPackageInfoNoCheck(
            r.activityInfo.applicationInfo, r.compatInfo);
    handleLaunchActivity(r, null);
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
} break;
```

ActivityClientRecord是什么？这是AMS传递过来的要启动的Activity。
我们仔细看那个getPackageInfoNoCheck方法，这个方法会提取apk中的所有资源，然后设置r的packageInfo属性。这个属性的类型很有名，叫做LoadedApk。注意，这个地方也是插件化技术渗入的一个点。
在H的这个分支中，又反过来回调ActivityThread的handleLaunchActivity方法。

重新看一下这个过程，每次都是APT执行ActivityThread的sendMessage方法，在这个方法中，把消息拼装一下，然后扔给H的swicth语句去分析，来决定要执行ActivityThread的那个方法。每次都是这样，习惯就好了。
handleLaunchActivity方法都做哪些事呢？

1. 通过Instrumentation的newActivity方法，创建要启动的Activity实例。
2. 为这个Activity创建一个上下文Context对象，并与Activity进行关联。
3. 通过Instrumentation的callActivityOnCreate方法，执行Activity的onCreate方法，从而启动Activity。看到这里是不是很熟悉很亲切？

至此，App启动完毕。这个流程经过了很多次握手，App和ASM频繁地向对方发送消息，而发送消息的机制，是建立在Binder的基础之上的。

## 参考

