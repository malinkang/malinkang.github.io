---
layout: posts
title: WorkManager源码分析
tags: [源码分析]
date: 2020-12-07 20:40:56
cover: https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/blog/images/cover/指环王.JPG
---

## WorkManager

`WorkManger`是一个单例对象, 但是从例子代码中没有看的调用初始化的地方。通过文档可知 `WorkManager`有两种初始化方式:

1. 应用启动之后, 它自己自动初始化
2. 按需初始化, 到了需要用到的地方才初始化。可以避免初始化影响应用的启动速度。

### 自动初始化

`WorkManagerInitializer`继承`ContentProvider`。在应用启动之后会调用 `onCreate()`, 所以自动初始化就是在 `onCreate()` 中执行`WorkManager` 的初始化。

```java
public class WorkManagerInitializer extends ContentProvider {
    @Override
    public boolean onCreate() {
        // Initialize WorkManager with the default configuration.
      // 调用initialize方法进行初始化
        WorkManager.initialize(getContext(), new Configuration.Builder().build());
        return true;
    }
}
```

![WorkManager](https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/blog/images/WorkManager.png)

### initialize()

```java
//WorkManager.java
public static void initialize(@NonNull Context context, @NonNull Configuration configuration) {   //调用WorkManagerImpl的initialize()
    WorkManagerImpl.initialize(context, configuration);
}
```

```java
//WorkManagerImpl.java
public static void initialize(@NonNull Context context, @NonNull Configuration configuration) {
    synchronized (sLock) {
        if (sDelegatedInstance == null) {
            context = context.getApplicationContext();
            if (sDefaultInstance == null) {
                //创建WorkManagerImpl
                sDefaultInstance = new WorkManagerImpl(
                        context,
                        configuration,
                        new WorkManagerTaskExecutor(configuration.getTaskExecutor()));
            }
            sDelegatedInstance = sDefaultInstance;
        }
    }
}
```

### WorkManagerImpl

```java
public WorkManagerImpl(
        @NonNull Context context,
        @NonNull Configuration configuration,
        @NonNull TaskExecutor workTaskExecutor,
        boolean useTestDatabase) {
    this(context,
            configuration,
            workTaskExecutor,
           //创建数据库
            WorkDatabase.create(
                    context.getApplicationContext(),
                    workTaskExecutor.getBackgroundExecutor(),
                    useTestDatabase)
    );
}
```

```java
public WorkManagerImpl(
        @NonNull Context context,
        @NonNull Configuration configuration,
        @NonNull TaskExecutor workTaskExecutor,
        @NonNull WorkDatabase database) {
    Context applicationContext = context.getApplicationContext();
   //创建Scheduler
    List<Scheduler> schedulers =
            createSchedulers(applicationContext, configuration, workTaskExecutor);
    //创建Processor
    Processor processor = new Processor(
            context,
            configuration,
            workTaskExecutor,
            database,
            schedulers);
    //调用internalInit方法
    internalInit(context, configuration, workTaskExecutor, database, schedulers, processor);
}
```

### createSchedulers()

```java
public List<Scheduler> createSchedulers(
        @NonNull Context context,
        @NonNull Configuration configuration,
        @NonNull TaskExecutor taskExecutor) {
    return Arrays.asList(
            //调用Schedulers的createBestAvailableBackgroundScheduler方法
            Schedulers.createBestAvailableBackgroundScheduler(context, this),
            // Specify the task executor directly here as this happens before internalInit.
            // GreedyScheduler creates ConstraintTrackers and controllers eagerly.
            new GreedyScheduler(context, configuration, taskExecutor, this));
}
```

### enqueue()

```java
//提交请求
public Operation enqueue(
        @NonNull List<? extends WorkRequest> workRequests) {
    // This error is not being propagated as part of the Operation, as we want the
    // app to crash during development. Having no workRequests is always a developer error.
    if (workRequests.isEmpty()) {
        throw new IllegalArgumentException(
                "enqueue needs at least one WorkRequest.");
    }
    return new WorkContinuationImpl(this, workRequests).enqueue();
}
```

```java
@Override
public @NonNull Operation enqueue() {
    // Only enqueue if not already enqueued.
    //防止重复提交, mEnqueued 会在 EnqueueRunnable 中被标记为 true
    if (!mEnqueued) {
        // The runnable walks the hierarchy of the continuations
        // and marks them enqueued using the markEnqueued() method, parent first.
        EnqueueRunnable runnable = new EnqueueRunnable(this);
      	//EnqueueRunnable 被提交到线程池执行
        mWorkManagerImpl.getWorkTaskExecutor().executeOnBackgroundThread(runnable);
        mOperation = runnable.getOperation();
    } else {
       //...
    }
    return mOperation;
}
```

## WorkRequest

![image-20201208162244365](https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/blog/images/WorkRequest.png)

`WorkRequest` 的`Buidler`是一个抽象类, 一次性和周期任务分别实现了自己的` Buidler`。

```java
public abstract static class Builder<B extends Builder<?, ?>, W extends WorkRequest> {
    boolean mBackoffCriteriaSet = false;
    UUID mId;
    WorkSpec mWorkSpec;
    Set<String> mTags = new HashSet<>();
    Class<? extends ListenableWorker> mWorkerClass;
    Builder(@NonNull Class<? extends ListenableWorker> workerClass) {
        mId = UUID.randomUUID(); //分配一个uuid
        mWorkerClass = workerClass;
        mWorkSpec = new WorkSpec(mId.toString(), workerClass.getName());//创建WorkSpec
        addTag(workerClass.getName());//
    }
  	public final @NonNull W build() {
    	W returnValue = buildInternal();//子类实现buildInternal()
    	// Create a new id and WorkSpec so this WorkRequest.Builder can be used multiple times.
    	mId = UUID.randomUUID();
    	mWorkSpec = new WorkSpec(mWorkSpec);
    	mWorkSpec.id = mId.toString();
    	return returnValue;
		}
}
```

## WorkSpec

查看`WorkSpec`的定义能发现其利用 `Room` 实现的数据库字段和类字段的映射。

在创建任务的时候, 每个任务会:

- 分配一个 uuid
- 创建一个 WorkSpec
- 添加 Worker 的类名为 tag
- 一次性任务设置 inputMergerClassName
- 周期任务设置周期执行的间隔时间

## EnqueueRunnable

### run()

```java
//EnqueueRunnable
@Override
public void run() {
    try { // 判断 Worker 是否有循环引用
        if (mWorkContinuation.hasCycles()) {
            throw new IllegalStateException(
                    String.format("WorkContinuation has cycles (%s)", mWorkContinuation));
        }
      	//储存到数据库, 并返回是否需要执行
        boolean needsScheduling = addToDatabase();
        if (needsScheduling) {
          	//需要执行:
          	//启用 RescheduleReceiver: 一个广播接收器
            // Enable RescheduleReceiver, only when there are Worker's that need scheduling.
            final Context context =
                    mWorkContinuation.getWorkManagerImpl().getApplicationContext();
            PackageManagerHelper.setComponentEnabled(context, RescheduleReceiver.class, true);
          // 在后台执行 work
            scheduleWorkInBackground();
        }
        mOperation.setState(Operation.SUCCESS);
    } catch (Throwable exception) {
        mOperation.setState(new Operation.State.FAILURE(exception));
    }
}
```

### addToDatabase()

```java
//添加到数据库
public boolean addToDatabase() {
    WorkManagerImpl workManagerImpl = mWorkContinuation.getWorkManagerImpl();
    WorkDatabase workDatabase = workManagerImpl.getWorkDatabase();
    workDatabase.beginTransaction();
    try {
        boolean needsScheduling = processContinuation(mWorkContinuation);
        workDatabase.setTransactionSuccessful();
        return needsScheduling;
    } finally {
        workDatabase.endTransaction();
    }
}
```

### scheduleWorkInBackground()

```java
public void scheduleWorkInBackground() {
    WorkManagerImpl workManager = mWorkContinuation.getWorkManagerImpl();
    //遍历Scheduler执行Scheduler的schedule()
    Schedulers.schedule(
            workManager.getConfiguration(),
            workManager.getWorkDatabase(),
            workManager.getSchedulers());
}
```



## Schedulers

```java
@NonNull
static Scheduler createBestAvailableBackgroundScheduler(
        @NonNull Context context,
        @NonNull WorkManagerImpl workManager) {
    Scheduler scheduler;
  // >=23 创建SystemJobScheduler
    if (Build.VERSION.SDK_INT >= WorkManagerImpl.MIN_JOB_SCHEDULER_API_LEVEL) {
        scheduler = new SystemJobScheduler(context, workManager);
      //调用PackageManagerHelper的静态方法
        setComponentEnabled(context, SystemJobService.class, true);
    } else {
        //创建GcmScheduler
        scheduler = tryCreateGcmBasedScheduler(context);
        if (scheduler == null) {
          //如果GcmScheduler为空则创建SystemAlarmScheduler
            scheduler = new SystemAlarmScheduler(context);
            setComponentEnabled(context, SystemAlarmService.class, true);
        }
    }
    return scheduler;
}
```

## Scheduler





![Scheduler](https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/blog/images/WorkManager-Scheduler.png)



## 参考

* [理解 WorkManager 的实现](https://blog.jiyang.site/posts/2019-12-07-%E7%90%86%E8%A7%A3-workmanager-%E7%9A%84%E5%AE%9E%E7%8E%B0/)







