---
title: "创建应用程序进程"
date: 2020-02-27T09:55:01+08:00
lastmod: 2020-02-27T09:55:01+08:00
draft: false
keywords: []
description: ""
tags: ["系统源码分析"]
categories: []
author: ""

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: false
toc: true
autoCollapseToc: false
postMetaInFooter: false
hiddenFromHomePage: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false

# You unlisted posts you might want not want the header or footer to show
hideHeaderAndFooter: false

# You can enable or disable out-of-date content warning for individual post.
# Comment this out to use the global config.
#enableOutdatedInfoWarning: false

flowchartDiagrams:
  enable: false
  options: ""

sequenceDiagrams: 
  enable: false
  options: ""

---



一个App打开另一个App的Activity或者绑定另外一个App的服务的过程，如果另外一个App的进程不存在，都会先创建另外一个App的进程，再执行后续操作。

应用程序进程创建过程可以分为以下几部分：

1. AMS发送启动应用程序进程请求
2. Zygote接收请求并创建应用程序进程。
3. ActivityThread初始化

<!--more-->

## AMS发送启动应用程序进程请求

这里先给出AMS发送启动应用程序进程请求过程的时序图，然后对每一个步骤进行详细分析，如图所示。

![](https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/images/android/AMS发送请求流程.png)



### ATMS#startProcessAsync

在`Activity`的启动过程中，会调用`ATMS`的`startProcessAsync`方法来创建新的进程。`startProcessAsync`会调用`PooledLambda`的`obtainMessage`方法获得一个`Message`。

```java
//frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java
void startProcessAsync(ActivityRecord activity, boolean knownToBeDead, boolean isTop,
        String hostingType) {
    try {
        if (Trace.isTagEnabled(TRACE_TAG_WINDOW_MANAGER)) {
            Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "dispatchingStartProcess:"
                    + activity.processName);
        }
        // Post message to start process to avoid possible deadlock of calling into AMS with the
        // ATMS lock held.
        final Message m = PooledLambda.obtainMessage(ActivityManagerInternal::startProcess,
                mAmInternal, activity.processName, activity.info.applicationInfo, knownToBeDead,
                isTop, hostingType, activity.intent.getComponent());
        mH.sendMessage(m);
    } finally {
        Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
    }
}
```

### PooledLambda#obtainMessage

```java
//frameworks/base/core/java/com/android/internal/util/function/pooled/PooledLambda.java
static <A, B, C> Message obtainMessage(
        TriConsumer<? super A, ? super B, ? super C> function,
        A arg1, B arg2, C arg3) {
    synchronized (Message.sPoolSync) {
        PooledRunnable callback = acquire(PooledLambdaImpl.sMessageCallbacksPool,
                function, 3, 0, ReturnType.VOID, arg1, arg2, arg3, null, null, null, null, null,
                null, null, null);
        return Message.obtain().setCallback(callback.recycleOnUse());
    }
}
```

```java
//frameworks/base/core/java/com/android/internal/util/function/pooled/PooledLambdaImpl.java
static <E extends PooledLambda> E acquire(Pool pool, Object func,
        int fNumArgs, int numPlaceholders, int fReturnType, Object a, Object b, Object c,
        Object d, Object e, Object f, Object g, Object h, Object i, Object j, Object k) {
    PooledLambdaImpl r = acquire(pool);//获取PooledLambdaImpl
    //...
    return (E) r;
}
```

`PooledLambdaImpl`继承自`OmniFunction`,`OmniFunction`实现了`Runnuable`接口。

```java
//frameworks/base/core/java/com/android/internal/util/function/pooled/OmniFunction.java
public void run() {
    invoke(null, null, null, null, null, null, null, null, null, null, null);
}
```

```java
//frameworks/base/core/java/com/android/internal/util/function/pooled/PooledLambdaImpl.java
@Override
R invoke(Object a1, Object a2, Object a3, Object a4, Object a5, Object a6, Object a7,
        Object a8, Object a9, Object a10, Object a11) {
  //...
    try {
        return doInvoke(); //调用doInvoke
    } finally {
        if (isRecycleOnUse()) {
            doRecycle();
        } else if (!isRecycled()) {
            int argsSize = ArrayUtils.size(mArgs);
            for (int i = 0; i < argsSize; i++) {
                popArg(i);
            }
        }
    }
}
```

```java
private R doInvoke() {
    final int funcType = getFlags(MASK_FUNC_TYPE);
    final int argCount = LambdaType.decodeArgCount(funcType);
    final int returnType = LambdaType.decodeReturnType(funcType);

    switch (argCount) { //根据参数个数
        //...
        case 3: {
            switch (returnType) {
                case LambdaType.ReturnType.VOID: {
                    ((TriConsumer) mFunc).accept(popArg(0), popArg(1), popArg(2));
                    return null;
                }
                case LambdaType.ReturnType.BOOLEAN: {
                    return (R) (Object) ((TriPredicate) mFunc).test(
                            popArg(0), popArg(1), popArg(2));
                }
                case LambdaType.ReturnType.OBJECT: {
                    return (R) ((TriFunction) mFunc).apply(popArg(0), popArg(1), popArg(2));
                }
            }
        } break;
       //...
    }
    throw new IllegalStateException("Unknown function type: " + LambdaType.toString(funcType));
}
```

最终`startProcessAsync`方法会调用`mAmInternal`的`startProcess`。``mAmInternal``是`ActivityManagerInternal`类，在`onActivityManagerInternalAdded`方法中调用`LocalServices`的`getService`获取该对象。

### ATMS#onActivityManagerInternalAdded

```java
ActivityManagerInternal mAmInternal;
public void onActivityManagerInternalAdded() {
    synchronized (mGlobalLock) {
        mAmInternal = LocalServices.getService(ActivityManagerInternal.class);
        mUgmInternal = LocalServices.getService(UriGrantsManagerInternal.class);
    }
}
```


在`AMS`的构造函数中创建`LocalService`，并在start方法中注册到`LocalServices`中。

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
public ActivityManagerService(Context systemContext, ActivityTaskManagerService atm) {
  //LocalService是
  mInternal = new LocalService();//创建LocalService 
  mAtmInternal = LocalServices.getService(ActivityTaskManagerInternal.class);
}
```

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
private void start() {
    //...
    //添加ActivityManagerInternal
    LocalServices.addService(ActivityManagerInternal.class, mInternal);
    //...
}
```

### LocalService

`LocalService`是`ActivityManagerService`的内部类，是`ActivityManagerInternal`的子类。

```java
public void startProcess(String processName, ApplicationInfo info, boolean knownToBeDead,
        boolean isTop, String hostingType, ComponentName hostingName) {
    try {
        if (Trace.isTagEnabled(Trace.TRACE_TAG_ACTIVITY_MANAGER)) {
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "startProcess:"
                    + processName);
        }
        synchronized (ActivityManagerService.this) {
            // If the process is known as top app, set a hint so when the process is
            // started, the top priority can be applied immediately to avoid cpu being
            // preempted by other processes before attaching the process of top app.
            //调用startProcessLocked
            startProcessLocked(processName, info, knownToBeDead, 0 /* intentFlags */,
                    new HostingRecord(hostingType, hostingName, isTop),
                    ZYGOTE_POLICY_FLAG_LATENCY_SENSITIVE, false /* allowWhileBooting */,
                    false /* isolated */, true /* keepIfLarge */);
        }
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    }
}
```

### AMS#startProcessLocked

最终会调用`AMS`的`startProcessLocked`方法。

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
//在构造函数中调用Injector的getProcessList方法中创建ProcessList
final ProcessList mProcessList; 
final ProcessRecord startProcessLocked(String processName,
      ApplicationInfo info, boolean knownToBeDead, int intentFlags,
      HostingRecord hostingRecord, int zygotePolicyFlags, boolean allowWhileBooting,
      boolean isolated, boolean keepIfLarge) {
  return mProcessList.startProcessLocked(processName, info, knownToBeDead, intentFlags,
          hostingRecord, zygotePolicyFlags, allowWhileBooting, isolated, 0 /* isolatedUid */,
          keepIfLarge, null /* ABI override */, null /* entryPoint */,
          null /* entryPointArgs */, null /* crashHandler */);
}
```

`AMS`的`startProcessLocked`方法会调用`ProcessList`的`startProcessLocked`方法。

### ProcessList#startProcessLocked

```java
//frameworks/base/services/core/java/com/android/server/am/ProcessList.java
boolean startProcessLocked(ProcessRecord app, HostingRecord hostingRecord,
        int zygotePolicyFlags, boolean disableHiddenApiChecks, boolean disableTestApiChecks,
        boolean mountExtStorageFull, String abiOverride) {
        //...
    try {
        try {
            final int userId = UserHandle.getUserId(app.uid);//
            AppGlobals.getPackageManager().checkPackageStartable(app.info.packageName, userId);
        } catch (RemoteException e) {
            throw e.rethrowAsRuntimeException();
        }
        //获取要创建的应用程序进程的用户ID
        int uid = app.uid;//①
        //
        // Start the process.  It will either succeed and return a result containing
        // the PID of the new process, or else throw a RuntimeException.
        final String entryPoint = "android.app.ActivityThread";//应用线程类名

        return startProcessLocked(hostingRecord, entryPoint, app, uid, gids,
                runtimeFlags, zygotePolicyFlags, mountExternal, seInfo, requiredAbi,
                instructionSet, invokeWith, startTime);
    } catch (RuntimeException e) {
       //...
    }
}
```

```java
//frameworks/base/services/core/java/com/android/server/am/ProcessList.java
boolean startProcessLocked(HostingRecord hostingRecord, String entryPoint, ProcessRecord app,
      int uid, int[] gids, int runtimeFlags, int zygotePolicyFlags, int mountExternal,
      String seInfo, String requiredAbi, String instructionSet, String invokeWith,
      long startTime) {
  app.pendingStart = true;
  app.killedByAm = false;
  app.removed = false;
  app.killed = false;
 //...

  if (mService.mConstants.FLAG_PROCESS_START_ASYNC) {
      if (DEBUG_PROCESSES) Slog.i(TAG_PROCESSES,
              "Posting procStart msg for " + app.toShortString());
      mService.mProcStartHandler.post(() -> handleProcessStart(
              app, entryPoint, gids, runtimeFlags, zygotePolicyFlags, mountExternal,
              requiredAbi, instructionSet, invokeWith, startSeq));
      return true;
  } else {
      try {
      //调用startProcess方法
          final Process.ProcessStartResult startResult = startProcess(hostingRecord,
                  entryPoint, app,
                  uid, gids, runtimeFlags, zygotePolicyFlags, mountExternal, seInfo,
                  requiredAbi, instructionSet, invokeWith, startTime);
          handleProcessStartedLocked(app, startResult.pid, startResult.usingWrapper,
                  startSeq, false);
      } catch (RuntimeException e) {
                  + app.processName, e);
          app.pendingStart = false;
          mService.forceStopPackageLocked(app.info.packageName, UserHandle.getAppId(app.uid),
                  false, false, true, false, false, app.userId, "start failure");
      }
      return app.pid > 0;
  }
}
```

### Process#startProcess

```java
//frameworks/base/services/core/java/com/android/server/am/ProcessList.java
private Process.ProcessStartResult startProcess(HostingRecord hostingRecord, String entryPoint,
      ProcessRecord app, int uid, int[] gids, int runtimeFlags, int zygotePolicyFlags,
      int mountExternal, String seInfo, String requiredAbi, String instructionSet,
      String invokeWith, long startTime) {
  try {
      //...
      final Process.ProcessStartResult startResult;
      if (hostingRecord.usesWebviewZygote()) {
        //...
      } else {
      //调用Process启动进程
          startResult = Process.start(entryPoint,
                  app.processName, uid, uid, gids, runtimeFlags, mountExternal,
                  app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                  app.info.dataDir, invokeWith, app.info.packageName, zygotePolicyFlags,
                  isTopApp, app.mDisabledCompatChanges, pkgDataInfoMap,
                  whitelistedAppDataInfoMap, bindMountAppsData, bindMountAppStorageDirs,
                  new String[]{PROC_START_SEQ_IDENT + app.startSeq});
      }
      checkSlow(startTime, "startProcess: returned from zygote!");
      return startResult;
  } finally {
      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
  }
}
```

### Process#start

```java
//frameworks/base/core/java/android/os/Process.java
public static final ZygoteProcess ZYGOTE_PROCESS = new ZygoteProcess();
public static ProcessStartResult start(@NonNull final String processClass,
                                       @Nullable final String niceName,
                                       int uid, int gid, @Nullable int[] gids,
                                       int runtimeFlags,
                                       int mountExternal,
                                       int targetSdkVersion,
                                       @Nullable String seInfo,
                                       @NonNull String abi,
                                       @Nullable String instructionSet,
                                       @Nullable String appDataDir,
                                       @Nullable String invokeWith,
                                       @Nullable String packageName,
                                       int zygotePolicyFlags,
                                       boolean isTopApp,
                                       @Nullable long[] disabledCompatChanges,
                                       @Nullable Map<String, Pair<String, Long>>
                                               pkgDataInfoMap,
                                       @Nullable Map<String, Pair<String, Long>>
                                               whitelistedDataInfoMap,
                                       boolean bindMountAppsData,
                                       boolean bindMountAppStorageDirs,
                                       @Nullable String[] zygoteArgs) {
    return ZYGOTE_PROCESS.start(processClass, niceName, uid, gid, gids,
                runtimeFlags, mountExternal, targetSdkVersion, seInfo,
                abi, instructionSet, appDataDir, invokeWith, packageName,
                zygotePolicyFlags, isTopApp, disabledCompatChanges,
                pkgDataInfoMap, whitelistedDataInfoMap, bindMountAppsData,
                bindMountAppStorageDirs, zygoteArgs);
}
```

### ZygoteProcess#start

在`Process`的`start`方法中只调用了`ZygoteProcess`的start方法，其中ZygoteProcess类用于保持与Zygote进程的通信状态。该start方法如下所示：

```java
//frameworks/base/core/java/android/os/ZygoteProcess.java
public final Process.ProcessStartResult start(@NonNull final String processClass,
                                              final String niceName,
                                              int uid, int gid, @Nullable int[] gids,
                                              int runtimeFlags, int mountExternal,
                                              int targetSdkVersion,
                                              @Nullable String seInfo,
                                              @NonNull String abi,
                                              @Nullable String instructionSet,
                                              @Nullable String appDataDir,
                                              @Nullable String invokeWith,
                                              @Nullable String packageName,
                                              int zygotePolicyFlags,
                                              boolean isTopApp,
                                              @Nullable long[] disabledCompatChanges,
                                              @Nullable Map<String, Pair<String, Long>>
                                                      pkgDataInfoMap,
                                              @Nullable Map<String, Pair<String, Long>>
                                                      whitelistedDataInfoMap,
                                              boolean bindMountAppsData,
                                              boolean bindMountAppStorageDirs,
                                              @Nullable String[] zygoteArgs) {
    // TODO (chriswailes): Is there a better place to check this value?
    if (fetchUsapPoolEnabledPropWithMinInterval()) {
        informZygotesOfUsapPoolStatus();
    }

    try {
        return startViaZygote(processClass, niceName, uid, gid, gids,
                runtimeFlags, mountExternal, targetSdkVersion, seInfo,
                abi, instructionSet, appDataDir, invokeWith, /*startChildZygote=*/ false,
                packageName, zygotePolicyFlags, isTopApp, disabledCompatChanges,
                pkgDataInfoMap, whitelistedDataInfoMap, bindMountAppsData,
                bindMountAppStorageDirs, zygoteArgs);
    } catch (ZygoteStartFailedEx ex) {
        Log.e(LOG_TAG,
                "Starting VM process through Zygote failed");
        throw new RuntimeException(
                "Starting VM process through Zygote failed", ex);
    }
}
```

### ZygoteProcess#startViaZygote

```java
private Process.ProcessStartResult startViaZygote(@NonNull final String processClass,
                                                  @Nullable final String niceName,
                                                  final int uid, final int gid,
                                                  @Nullable final int[] gids,
                                                  int runtimeFlags, int mountExternal,
                                                  int targetSdkVersion,
                                                  @Nullable String seInfo,
                                                  @NonNull String abi,
                                                  @Nullable String instructionSet,
                                                  @Nullable String appDataDir,
                                                  @Nullable String invokeWith,
                                                  boolean startChildZygote,
                                                  @Nullable String packageName,
                                                  int zygotePolicyFlags,
                                                  boolean isTopApp,
                                                  @Nullable long[] disabledCompatChanges,
                                                  @Nullable Map<String, Pair<String, Long>>
                                                          pkgDataInfoMap,
                                                  @Nullable Map<String, Pair<String, Long>>
                                                          whitelistedDataInfoMap,
                                                  boolean bindMountAppsData,
                                                  boolean bindMountAppStorageDirs,
                                                  @Nullable String[] extraArgs)
                                                  throws ZygoteStartFailedEx {
    //创建字符串列表argsForZygote，并将应用进程的启动参数保存在argsForZygote中
    ArrayList<String> argsForZygote = new ArrayList<>();

    // --runtime-args, --setuid=, --setgid=,
    // and --setgroups= must go first
    argsForZygote.add("--runtime-args");
    argsForZygote.add("--setuid=" + uid);
    argsForZygote.add("--setgid=" + gid);
    argsForZygote.add("--runtime-flags=" + runtimeFlags);
    //...
    synchronized(mLock) {
        // The USAP pool can not be used if the application will not use the systems graphics
        // driver.  If that driver is requested use the Zygote application start path.
        return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi),
                                          zygotePolicyFlags,
                                          argsForZygote);//启动参数
    }
}
```

### ZygoteProcess#openZygoteSocketIfNeeded

```java
private ZygoteState openZygoteSocketIfNeeded(String abi) throws ZygoteStartFailedEx {
    try {
        attemptConnectionToPrimaryZygote();

        if (primaryZygoteState.matches(abi)) {
            return primaryZygoteState;
        }

        if (mZygoteSecondarySocketAddress != null) {
            // The primary zygote didn't match. Try the secondary.
            attemptConnectionToSecondaryZygote();

            if (secondaryZygoteState.matches(abi)) {
                return secondaryZygoteState;
            }
        }
    } catch (IOException ioe) {
        throw new ZygoteStartFailedEx("Error connecting to zygote", ioe);
    }

    throw new ZygoteStartFailedEx("Unsupported zygote ABI: " + abi);
}
```

### ZygoteProcess#attemptConnectionToPrimaryZygote

```java
private void attemptConnectionToPrimaryZygote() throws IOException {
    if (primaryZygoteState == null || primaryZygoteState.isClosed()) {
        primaryZygoteState =
                ZygoteState.connect(mZygoteSocketAddress, mUsapPoolSocketAddress);

        maybeSetApiDenylistExemptions(primaryZygoteState, false);
        maybeSetHiddenApiAccessLogSampleRate(primaryZygoteState);
    }
}
```

### ZygoteState#connect

```java
static ZygoteState connect(@NonNull LocalSocketAddress zygoteSocketAddress,
        @Nullable LocalSocketAddress usapSocketAddress)
        throws IOException {
    DataInputStream zygoteInputStream;
    BufferedWriter zygoteOutputWriter;
    final LocalSocket zygoteSessionSocket = new LocalSocket();
    if (zygoteSocketAddress == null) {
        throw new IllegalArgumentException("zygoteSocketAddress can't be null");
    }
    try {
        zygoteSessionSocket.connect(zygoteSocketAddress);
        zygoteInputStream = new DataInputStream(zygoteSessionSocket.getInputStream());
        zygoteOutputWriter =
                new BufferedWriter(
                        new OutputStreamWriter(zygoteSessionSocket.getOutputStream()),
                        Zygote.SOCKET_BUFFER_SIZE);
    } catch (IOException ex) {
        try {
            zygoteSessionSocket.close();
        } catch (IOException ignore) { }

        throw ex;
    }

    return new ZygoteState(zygoteSocketAddress, usapSocketAddress,
                           zygoteSessionSocket, zygoteInputStream, zygoteOutputWriter,
                           getAbiList(zygoteOutputWriter, zygoteInputStream));
}
```

### ZygoteProcess#zygoteSendArgsAndGetResult

`zygoteSendArgsAndGetResult`方法的主要作用就是将传入的应用进程的启动参数argsForZygote写入`ZygoteState`中

```java
private Process.ProcessStartResult zygoteSendArgsAndGetResult(
        ZygoteState zygoteState, int zygotePolicyFlags, @NonNull ArrayList<String> args)
        throws ZygoteStartFailedEx {
   //...
    String msgStr = args.size() + "\n" + String.join("\n", args) + "\n";
   //...
    return attemptZygoteSendArgsAndGetResult(zygoteState, msgStr);
}
```

### ZygoteState#attemptZygoteSendArgsAndGetResult

```java
private Process.ProcessStartResult attemptZygoteSendArgsAndGetResult(
        ZygoteState zygoteState, String msgStr) throws ZygoteStartFailedEx {
    try {
        final BufferedWriter zygoteWriter = zygoteState.mZygoteOutputWriter;
        final DataInputStream zygoteInputStream = zygoteState.mZygoteInputStream;
        //写入数据
        zygoteWriter.write(msgStr);
        zygoteWriter.flush();
        //创建ProcessStartResult
        Process.ProcessStartResult result = new Process.ProcessStartResult();
        //读取数据
        result.pid = zygoteInputStream.readInt(); 
        result.usingWrapper = zygoteInputStream.readBoolean();
        if (result.pid < 0) {
            throw new ZygoteStartFailedEx("fork() failed");
        }

        return result;
    } catch (IOException ex) {
        zygoteState.close();
        Log.e(LOG_TAG, "IO Exception while communicating with Zygote - "
                + ex.toString());
        throw new ZygoteStartFailedEx(ex);
    }
}
```

## Zygote接收请求并创建应用程序进程

启动`Zygote`进程，会注册`Socket`，并调用`ZygoteServer`的`runSelectloop`进入无限循环。

### ZygoteServer#runSelectloop

```java
//frameworks/base/core/java/com/android/internal/os/ZygoteServer.java
Runnable runSelectLoop(String abiList) {
  ArrayList<FileDescriptor> socketFDs = new ArrayList<>();
  ArrayList<ZygoteConnection> peers = new ArrayList<>();

  socketFDs.add(mZygoteSocket.getFileDescriptor());
  peers.add(null);

  mUsapPoolRefillTriggerTimestamp = INVALID_TIMESTAMP;

  while (true) {
      //...

      if (pollReturnValue == 0) {
          // The poll returned zero results either when the timeout value has been exceeded
          // or when a non-blocking poll is issued and no FDs are ready.  In either case it
          // is time to refill the pool.  This will result in a duplicate assignment when
          // the non-blocking poll returns zero results, but it avoids an additional
          // conditional in the else branch.
          mUsapPoolRefillTriggerTimestamp = INVALID_TIMESTAMP;
          mUsapPoolRefillAction = UsapPoolRefillAction.DELAYED;

      } else {
          boolean usapPoolFDRead = false;

          while (--pollIndex >= 0) {
              if ((pollFDs[pollIndex].revents & POLLIN) == 0) {
                  continue;
              }

              if (pollIndex == 0) {
                  // Zygote server socket
                  //创建ZygoteConnection
                  ZygoteConnection newPeer = acceptCommandPeer(abiList);
                  //添加到集合中
                  peers.add(newPeer);
                  socketFDs.add(newPeer.getFileDescriptor());
              } else if (pollIndex < usapPoolEventFDIndex) {
                  // Session socket accepted from the Zygote server socket

                  try {
                      ZygoteConnection connection = peers.get(pollIndex);
                      boolean multipleForksOK = !isUsapPoolEnabled()
                              && ZygoteHooks.indefiniteThreadSuspensionOK();
                      //调用ZygoteConnection的processCommand方法
                      final Runnable command =
                              connection.processCommand(this, multipleForksOK);
                      //...
                  } catch (Exception e) {
                      //...
                  } finally {
                      //...
                  }

              } else {
                  //。。。
              }
          }
          //...
      }
      //...
  }
}
```

### ZygoteConnection#processCommand

```java
Runnable processCommand(ZygoteServer zygoteServer, boolean multipleOK) {
    ZygoteArguments parsedArgs;

    try (ZygoteCommandBuffer argBuffer = new ZygoteCommandBuffer(mSocket)) {
        while (true) {
            try {
                //获取参数
                parsedArgs = ZygoteArguments.getInstance(argBuffer);
                // Keep argBuffer around, since we need it to fork.
            } catch (IOException ex) {
                throw new IllegalStateException("IOException on command socket", ex);
            }

            if (parsedArgs.mInvokeWith != null || parsedArgs.mStartChildZygote
                    || !multipleOK || peer.getUid() != Process.SYSTEM_UID) {
                // Continue using old code for now. TODO: Handle these cases in the other path.
                //创建进程
                pid = Zygote.forkAndSpecialize(parsedArgs.mUid, parsedArgs.mGid,
                        parsedArgs.mGids, parsedArgs.mRuntimeFlags, rlimits,
                        parsedArgs.mMountExternal, parsedArgs.mSeInfo, parsedArgs.mNiceName,
                        fdsToClose, fdsToIgnore, parsedArgs.mStartChildZygote,
                        parsedArgs.mInstructionSet, parsedArgs.mAppDataDir,
                        parsedArgs.mIsTopApp, parsedArgs.mPkgDataInfoList,
                        parsedArgs.mWhitelistedDataInfoList, parsedArgs.mBindMountAppDataDirs,
                        parsedArgs.mBindMountAppStorageDirs);

                try {
                    if (pid == 0) {
                        // in child
                        zygoteServer.setForkChild();

                        zygoteServer.closeServerSocket();
                        IoUtils.closeQuietly(serverPipeFd);
                        serverPipeFd = null;

                        return handleChildProc(parsedArgs, childPipeFd,
                                parsedArgs.mStartChildZygote);
                    } else {
                        // In the parent. A pid < 0 indicates a failure and will be handled in
                        // handleParentProc.
                        IoUtils.closeQuietly(childPipeFd);
                        childPipeFd = null;
                        handleParentProc(pid, serverPipeFd);
                        return null;
                    }
                } finally {
                    IoUtils.closeQuietly(childPipeFd);
                    IoUtils.closeQuietly(serverPipeFd);
                }
            } else {
              //...
            }
        }
    }
    //...
}
```

### ZygoteConnection#handleChildProc

```java
private Runnable handleChildProc(ZygoteArguments parsedArgs,
        FileDescriptor pipeFd, boolean isZygote) {
    /*
     * By the time we get here, the native code has closed the two actual Zygote
     * socket connections, and substituted /dev/null in their place.  The LocalSocket
     * objects still need to be closed properly.
     */

    closeSocket();

    Zygote.setAppProcessName(parsedArgs, TAG);

    // End of the postFork event.
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    if (parsedArgs.mInvokeWith != null) {
        WrapperInit.execApplication(parsedArgs.mInvokeWith,
                parsedArgs.mNiceName, parsedArgs.mTargetSdkVersion,
                VMRuntime.getCurrentInstructionSet(),
                pipeFd, parsedArgs.mRemainingArgs);

        // Should not get here.
        throw new IllegalStateException("WrapperInit.execApplication unexpectedly returned");
    } else {
        if (!isZygote) {
            return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion,
                    parsedArgs.mDisabledCompatChanges,
                    parsedArgs.mRemainingArgs, null /* classLoader */);
        } else {
            return ZygoteInit.childZygoteInit(
                    parsedArgs.mRemainingArgs  /* classLoader */);
        }
    }
}
```

### ZygoteInit#zygoteInit

```java
//frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
public static Runnable zygoteInit(int targetSdkVersion, long[] disabledCompatChanges,
        String[] argv, ClassLoader classLoader) {
    if (RuntimeInit.DEBUG) {
        Slog.d(RuntimeInit.TAG, "RuntimeInit: Starting application from zygote");
    }

    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ZygoteInit");
    RuntimeInit.redirectLogStreams();

    RuntimeInit.commonInit();
    ZygoteInit.nativeZygoteInit();
    return RuntimeInit.applicationInit(targetSdkVersion, disabledCompatChanges, argv,
            classLoader);
}
```

### ZygoteInit#childZygote

```java
static Runnable childZygoteInit(String[] argv) {
    RuntimeInit.Arguments args = new RuntimeInit.Arguments(argv);
    return RuntimeInit.findStaticMain(args.startClass, args.startArgs, /* classLoader= */null);
}
```

### RuntimeInit#applicationInit

```java
//frameworks/base/core/java/com/android/internal/os/RuntimeInit.java
protected static Runnable applicationInit(int targetSdkVersion, long[] disabledCompatChanges,
        String[] argv, ClassLoader classLoader) {
    // If the application calls System.exit(), terminate the process
    // immediately without running any shutdown hooks.  It is not possible to
    // shutdown an Android application gracefully.  Among other things, the
    // Android runtime shutdown hooks close the Binder driver, which can cause
    // leftover running threads to crash before the process actually exits.
    nativeSetExitWithoutCleanup(true);

    VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);
    VMRuntime.getRuntime().setDisabledCompatChanges(disabledCompatChanges);

    final Arguments args = new Arguments(argv);

    // The end of of the RuntimeInit event (see #zygoteInit).
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

    // Remaining arguments are passed to the start class's static main
    return findStaticMain(args.startClass, args.startArgs, classLoader);
}
```



### RuntimeInit#findStaticMain

```java
//frameworks/base/core/java/com/android/internal/os/RuntimeInit.java
protected static Runnable findStaticMain(String className, String[] argv,
        ClassLoader classLoader) {
    Class<?> cl;

    try {
        cl = Class.forName(className, true, classLoader);
    } catch (ClassNotFoundException ex) {
        throw new RuntimeException(
                "Missing class when invoking static main " + className,
                ex);
    }

    Method m;
    try {
        //反射获取main方法
        m = cl.getMethod("main", new Class[] { String[].class });
    } catch (NoSuchMethodException ex) {
        throw new RuntimeException(
                "Missing static main on " + className, ex);
    } catch (SecurityException ex) {
        throw new RuntimeException(
                "Problem getting static main on " + className, ex);
    }

    int modifiers = m.getModifiers();
    if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
        throw new RuntimeException(
                "Main method is not public and static on " + className);
    }

    /*
     * This throw gets caught in ZygoteInit.main(), which responds
     * by invoking the exception's run() method. This arrangement
     * clears up all the stack frames that were required in setting
     * up the process.
     */
    return new MethodAndArgsCaller(m, argv);
}
```

### MethodAndArgsCaller

```java
static class MethodAndArgsCaller implements Runnable {
    /** method to call */
    private final Method mMethod;

    /** argument array */
    private final String[] mArgs;

    public MethodAndArgsCaller(Method method, String[] args) {
        mMethod = method;
        mArgs = args;
    }

    public void run() {
        try {
          	//执行方法
            mMethod.invoke(null, new Object[] { mArgs });
        } catch (IllegalAccessException ex) {
            throw new RuntimeException(ex);
        } catch (InvocationTargetException ex) {
            Throwable cause = ex.getCause();
            if (cause instanceof RuntimeException) {
                throw (RuntimeException) cause;
            } else if (cause instanceof Error) {
                throw (Error) cause;
            }
            throw new RuntimeException(ex);
        }
    }
}
```

## ActivityThread初始化

### AT#main

`Zygote`进程创建`ActivityThread`之后会执行它的`main`方法。`ActivityThread`的`main`方法会创建`ActivityThread`，并调用`attach`方法。

```java
public static void main(String[] args) {
  
    //创建主线程 looper
    Looper.prepareMainLooper();
		//创建ActivityThread
    ActivityThread thread = new ActivityThread();
    thread.attach(false, startSeq);//调用attach

    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    if (false) {
        Looper.myLooper().setMessageLogging(new
                LogPrinter(Log.DEBUG, "ActivityThread"));
    }

    // End of event ActivityThreadMain.
    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

### AT#attach

```java
private void attach(boolean system, long startSeq) {
    sCurrentActivityThread = this;
    mSystemThread = system;
    if (!system) {
        android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                UserHandle.myUserId());
        RuntimeInit.setApplicationObject(mAppThread.asBinder());
        //获取ams
        final IActivityManager mgr = ActivityManager.getService();
        try {
            //调用ams的attachApplication方法
            mgr.attachApplication(mAppThread, startSeq);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
        //...
    } else {
       //...
    }
    //...
}
```

### AMS#attachApplication

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
public final void attachApplication(IApplicationThread thread, long startSeq) {
    if (thread == null) {
        throw new SecurityException("Invalid application interface");
    }
    synchronized (this) {
        int callingPid = Binder.getCallingPid();
        final int callingUid = Binder.getCallingUid();
        final long origId = Binder.clearCallingIdentity();
        attachApplicationLocked(thread, callingPid, callingUid, startSeq);
        Binder.restoreCallingIdentity(origId);
    }
}
```

### AMS#attachApplicationLocked

`attachApplicationLocked`会调用`mAtmInternal`的`attachApplication`方法。`mAtmInternal`是`ActivityTaskManagerInternal`类型，它的子类是`ATMS`的内部类`LocalService`。在`ATMS`创建并添加到`LocalServices`中，并在`AMS`中获取。`LocalService`的`attachApplication`方法最终会调用`ATMS`的`attachApplication`方法

```java
@GuardedBy("this")
private boolean attachApplicationLocked(@NonNull IApplicationThread thread,
        int pid, int callingUid, long startSeq) {

    // See if the top visible activity is waiting to run in this process...
    if (normalMode) {
        try { //调用mAtmInternal的方法。
            didSomething = mAtmInternal.attachApplication(app.getWindowProcessController());
        } catch (Exception e) {
            Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
            badApp = true;
        }
    }
    return true;
}
```

### ATMS#attachApplication

```java
public boolean attachApplication(WindowProcessController wpc) throws RemoteException {
    synchronized (mGlobalLockWithoutBoost) {
        if (Trace.isTagEnabled(TRACE_TAG_WINDOW_MANAGER)) {
            Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "attachApplication:" + wpc.mName);
        }
        try {
            return mRootWindowContainer.attachApplication(wpc);
        } finally {
            Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
        }
    }
}
```



### RWC#attachApplication

```java
//frameworks/base/services/core/java/com/android/server/wm/RootWindowContainer.java
boolean attachApplication(WindowProcessController app) throws RemoteException {
    boolean didSomething = false;
    for (int displayNdx = getChildCount() - 1; displayNdx >= 0; --displayNdx) {
        mTmpRemoteException = null;
        mTmpBoolean = false; // Set to true if an activity was started.

        final DisplayContent display = getChildAt(displayNdx);
        for (int areaNdx = display.getTaskDisplayAreaCount() - 1; areaNdx >= 0; --areaNdx) {
            final TaskDisplayArea taskDisplayArea = display.getTaskDisplayAreaAt(areaNdx);
            for (int taskNdx = taskDisplayArea.getStackCount() - 1; taskNdx >= 0; --taskNdx) {
                final ActivityStack rootTask = taskDisplayArea.getStackAt(taskNdx);
                if (rootTask.getVisibility(null /*starting*/) == STACK_VISIBILITY_INVISIBLE) {
                    break;
                }
                 //调用startActivityForAttachedApplicationIfNeeded方法
                final PooledFunction c = PooledLambda.obtainFunction(
                        RootWindowContainer::startActivityForAttachedApplicationIfNeeded, this,
                        PooledLambda.__(ActivityRecord.class), app,
                        rootTask.topRunningActivity());
                rootTask.forAllActivities(c);
                c.recycle();
                if (mTmpRemoteException != null) {
                    throw mTmpRemoteException;
                }
            }
        }
        didSomething |= mTmpBoolean;
    }
    if (!didSomething) {
        ensureActivitiesVisible(null, 0, false /* preserve_windows */);
    }
    return didSomething;
}
```

### RWC#startActivityForAttachedApplicationIfNeeded

```java
private boolean startActivityForAttachedApplicationIfNeeded(ActivityRecord r,
        WindowProcessController app, ActivityRecord top) {
    if (r.finishing || !r.okToShowLocked() || !r.visibleIgnoringKeyguard || r.app != null
            || app.mUid != r.info.applicationInfo.uid || !app.mName.equals(r.processName)) {
        return false;
    }

    try { //调用realStartActivityLocked
        if (mStackSupervisor.realStartActivityLocked(r, app,
                top == r && r.isFocusable() /*andResume*/, true /*checkConfig*/)) {
            mTmpBoolean = true;
        }
    } catch (RemoteException e) {
        Slog.w(TAG, "Exception in new application when starting activity "
                + top.intent.getComponent().flattenToShortString(), e);
        mTmpRemoteException = e;
        return true;
    }
    return false;
}
```





## 参考

* [Android 11 Activity 启动分析](https://calmcenter.club/2020/android-framework-activity.html)





