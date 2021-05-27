---
layout: posts
title: RxJava源码分析
date: 2020-11-27 23:47:34
tags: ["源码分析"]
cover: https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/blog/images/cover/罗马假日.JPG
---

## RxJava创建过程

![RxJava创建流程](https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/blog/images/android/RxJava创建流程.png)

### create()

调用`create()`创建`Observable`。

```java
Observable.create(new ObservableOnSubscribe<Integer>() {
    @Override
    public void subscribe(@NonNull ObservableEmitter<Integer> emitter) throws Throwable {
        emitter.onNext(2);
    }
}).subscribe(new Consumer<Integer>() {
    @Override
    public void accept(Integer integer) throws Throwable {
        System.out.println("integer = "+integer);
    }
});
```

`Observable`的`create`方法会创建一个`ObservableCreate`对象。

```java
public static <T> Observable<T> create(@NonNull ObservableOnSubscribe<T> source) {
    //创建ObservableCreate对象
    return RxJavaPlugins.onAssembly(new ObservableCreate<>(source));
}
```

### subscribe()

```java
public final Disposable subscribe(@NonNull Consumer<? super T> onNext) {
    return subscribe(onNext, Functions.ON_ERROR_MISSING, Functions.EMPTY_ACTION);
}
```

```java
public final Disposable subscribe(@NonNull Consumer<? super T> onNext, @NonNull Consumer<? super Throwable> onError,@NonNull Action onComplete) {
  	//创建LambdaObserver
    LambdaObserver<T> ls = new LambdaObserver<>(onNext, onError, onComplete, Functions.emptyConsumer());
    subscribe(ls);
    return ls;
}
```

```java
public final void subscribe(@NonNull Observer<? super T> observer) {
    Objects.requireNonNull(observer, "observer is null");
    try {
        observer = RxJavaPlugins.onSubscribe(this, observer);
      	//调用subscribeActual
        subscribeActual(observer);
    } catch (NullPointerException e) { // NOPMD
        throw e;
    } catch (Throwable e) {
      //...
    }
}
```

```java
@Override
protected void subscribeActual(Observer<? super T> observer) {
  //创建发射器
    CreateEmitter<T> parent = new CreateEmitter<>(observer);
    //调用onSubscribe方法
    observer.onSubscribe(parent);
    try {
      //调用subscribe方法 
      //在subscribe方法中会调用emitter的onNext()
      //emitter的onNext会调用observer的onNext()
      //observer的onNext会调用Consumer的accept()
        source.subscribe(parent);
    } catch (Throwable ex) {
        Exceptions.throwIfFatal(ex);
        parent.onError(ex);
    }
}
```

## 线程切换

线程控制的代码

```java
Observable.create(new ObservableOnSubscribe<Integer>() {
    @Override
    public void subscribe(@NonNull ObservableEmitter<Integer> emitter) throws Throwable {
        emitter.onNext(2);
    }
}).subscribeOn(Schedulers.io())
  .observeOn(AndroidSchedulers.mainThread())
  .subscribe(new Consumer<Integer>() {
    @Override
    public void accept(Integer integer) throws Throwable {
        System.out.println("integer = " + integer);
    }
});
```

### subscribeOn()

![subscribeOn流程](https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/blog/images/leetcode/subscribeOn流程.png)

```java
public final Observable<T> subscribeOn(@NonNull Scheduler scheduler) {
    //创建ObservableSubscribeOn
    return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<>(this, scheduler));
}
```

```java
//ObservableSubscribeOn.java
public void subscribeActual(final Observer<? super T> observer) {
  //创建观察者
    final SubscribeOnObserver<T> parent = new SubscribeOnObserver<>(observer);
    observer.onSubscribe(parent);
    parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
}
```



```java
final class SubscribeTask implements Runnable {
    private final SubscribeOnObserver<T> parent;
    SubscribeTask(SubscribeOnObserver<T> parent) {
        this.parent = parent;
    }
    @Override
    public void run() {
      	//调用subscribe方法
        source.subscribe(parent);
    }
}
```

### scheduleDirect()

```java
//Scheduler.java
public Disposable scheduleDirect(@NonNull Runnable run) {
    return scheduleDirect(run, 0L, TimeUnit.NANOSECONDS);
}
```

```java
@NonNull
public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
    final Worker w = createWorker(); //创建Worker
    final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);
    DisposeTask task = new DisposeTask(decoratedRun, w);
    //调用woker的schedule方法
    w.schedule(task, delay, unit);
    return task;
}
```

```java
//EventLoopWorker.java
public Disposable schedule(@NonNull Runnable action, long delayTime, @NonNull TimeUnit unit) {
    if (tasks.isDisposed()) {
        // don't schedule, we are unsubscribed
        return EmptyDisposable.INSTANCE;
    }
    return threadWorker.scheduleActual(action, delayTime, unit, tasks);
}
```

```java
@NonNull
public ScheduledRunnable scheduleActual(final Runnable run, long delayTime, @NonNull TimeUnit unit, @Nullable DisposableContainer parent) {
    Runnable decoratedRun = RxJavaPlugins.onSchedule(run);
    ScheduledRunnable sr = new ScheduledRunnable(decoratedRun, parent);
    if (parent != null) {
        if (!parent.add(sr)) {
            return sr;
        }
    }
    Future<?> f;
    try {
        if (delayTime <= 0) {
            f = executor.submit((Callable<Object>)sr);
        } else {
            f = executor.schedule((Callable<Object>)sr, delayTime, unit);
        }
        sr.setFuture(f);
    } catch (RejectedExecutionException ex) {
        if (parent != null) {
            parent.remove(sr);
        }
        RxJavaPlugins.onError(ex);
    }
    return sr;
}
```

### observeOn()





![observeOn流程](https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/blog/images/android/observeOn流程.png)



```java
public final Observable<T> observeOn(@NonNull Scheduler scheduler, boolean delayError, int bufferSize) {
    Objects.requireNonNull(scheduler, "scheduler is null");
    ObjectHelper.verifyPositive(bufferSize, "bufferSize");
    //创建ObservableObserveOn
    return RxJavaPlugins.onAssembly(new ObservableObserveOn<>(this, scheduler, delayError, bufferSize));
}
```

```java
//ObservableObserveOn
protected void subscribeActual(Observer<? super T> observer) {
    if (scheduler instanceof TrampolineScheduler) {
        source.subscribe(observer);
    } else {
        Scheduler.Worker w = scheduler.createWorker(); //获取worker
        //创建ObserveOnObserver 并调用subscribe
        source.subscribe(new ObserveOnObserver<>(observer, w, delayError, bufferSize));
    }
}
```

```java
//ObserveOnObserver
public void onNext(T t) {
    if (done) {
        return;
    }
    if (sourceMode != QueueDisposable.ASYNC) {
        queue.offer(t);
    }
    schedule();
}
void schedule() {
    if (getAndIncrement() == 0) {
        worker.schedule(this);
    }
}
```











