---
title: 《Java并发编程实战》第6章执行任务
date: 2015-10-15 10:53:22
tags: ["Java并发编程实战"]
---

大多数并发应用程序都是围绕“任务执行”来构造的，任务通常是一些抽象的且离散的工作单元。通过把应用程序的工作分解到多个任务中，可以简化程序的组织结构，提供一种自然的事务边界来优化错误恢复过程，以及提供一种自然的并行工作结构来提升并发性。

<!--more-->

## 6.1 在线程中执行任务

## 6.2 Executor框架

### 6.2.3 线程池

“在线程池中执行任务”比“为每个任务分配一个线程”优势更多。通过重用现有的线程而不是创建新的线程，可以在处理多个请求时分摊在创建和销毁过程中产生的巨大开销。另一个额外的好处是，当请求到达时，工作线程通常已经存在，因此不会由于等待创建线程而延迟任务的执行，从而提高了响应性。通过适当调整线程池的大小，可以创建足够多的线程以便使处理器保持忙碌状态，同时还可以防止多个线程相互竞争资源而使应用程序耗尽内存或失败。

类库提供了一个灵活的线程池以及一些有用的默认配置。可以通过调用`Executors`中的静态方法之一来创建一个线程池。

#### newFixedThreadPool

`newFixedThreadPool`将创建一个固定长度的线程池，每当提交一个任务时就创建一个线程，直到达到线程池的最大数量，这时线程池的规模将不再变化（如果某个线程由于发生了未预期的Exception而结束，那么线程池会补充一个新的线程）。

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads, //核心线程数和最大线程数相同
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```



#### newCachedThreadPool

newCachedThreadPool将创建一个可缓存的线程池，如果线程池的当前规模超过了处理需求时，那么将回收空闲的线程，而当需求增加时，则可以添加新的线程，线程池的规模不存在任何限制。

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,//核心线程数是Integer最大值
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```



#### newSingleThreadExecutor

newSingleThreadExecutor是一个单线程的Executor，它创建单个线程来执行任务，如果这个线程异常结束，会创建另一个线程来替代。`newSingleThreadExecutor`能确保依照任务在队列中的顺序来串行执行。

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1, //核心线程数和最大线程数都是1
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```



#### newScheduledThreadPool

`newScheduledThreadPool`创建了一个固定长度的线程池，而且以延迟或定时的方式来执行任务。ScheduledThreadPool提供了3个周期方法：

```
ScheduledExecutorService service = Executors.newScheduledThreadPool(10);
 
service.schedule(new Task(), 10, TimeUnit.SECONDS);
 
service.scheduleAtFixedRate(new Task(), 10, 10, TimeUnit.SECONDS);
 
service.scheduleWithFixedDelay(new Task(), 10, 10, TimeUnit.SECONDS);
```

那么这 3 种方法有什么区别呢？

* 第一种方法 schedule 比较简单，表示延迟指定时间后执行一次任务，如果代码中设置参数为 10 秒，也就是 10 秒后执行一次任务后就结束。

* 第二种方法 scheduleAtFixedRate 表示以固定的频率执行任务，它的第二个参数 initialDelay 表示第一次延时时间，第三个参数 period 表示周期，也就是第一次延时后每次延时多长时间执行一次任务。

* 第三种方法 scheduleWithFixedDelay 与第二种方法类似，也是周期执行任务，区别在于对周期的定义，之前的 scheduleAtFixedRate 是以任务开始的时间为时间起点开始计时，时间到就开始执行第二次任务，而不管任务需要花多久执行；而 scheduleWithFixedDelay 方法以任务结束的时间为下一次循环的时间起点开始计时。

举个例子，假设某个同学正在熬夜写代码，需要喝咖啡来提神，假设每次喝咖啡都需要花5s的时间，如果此时采用第2种方法 scheduleAtFixedRate，时间间隔设置为10s。

```java
public static void main(String[] args) {
    ScheduledExecutorService service = Executors.newScheduledThreadPool(1);
    service.scheduleAtFixedRate(new Task(), 0, 10, TimeUnit.SECONDS);//间隔10s
}
public static String now() {
    return new SimpleDateFormat("mm:ss").format(System.currentTimeMillis());
}

public static class Task implements Runnable {
    @Override
    public void run() {
        System.out.println(now() + ": Start drinking coffee");
        try {
            Thread.sleep(5000); //花费5s
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(now() + ": Finish drinking coffee");
    }
}
```

```
28:53: Start drinking coffee
28:58: Finish drinking coffee
29:03: Start drinking coffee
29:08: Finish drinking coffee
29:13: Start drinking coffee
29:18: Finish drinking coffee
```

上述代码替换成`scheduleWithFixedDelay`方法

```
33:18: Start drinking coffee
33:23: Finish drinking coffee
33:33: Start drinking coffee
33:38: Finish drinking coffee
33:48: Start drinking coffee
33:53: Finish drinking coffee
```

#### newSingleThreadScheduledExecutor

它实际和` ScheduledThreadPool`线程池非常相似，它只是 ScheduledThreadPool 的一个特例，内部只有一个线程。

### 6.2.4 Executor的生命周期

### 6.2.5 延迟任务与周期任务

## 6.3 找出可利用的并行性

