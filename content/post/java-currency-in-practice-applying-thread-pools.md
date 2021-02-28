---
title: 《Java并发编程实战》第8章线程池的使用
date: 2015-10-22 10:53:22
tags: ["Java并发编程实战"]
---

本章将介绍对线程池进行配置与调优的一些高级选项，并分析在使用任务执行框架时需要注意的各种危险，以及一些使用`Executor`的高级示例。

<!--more-->

## 8.1 在任务与执行策略之间的隐性耦合

## 8.2 设置线程池的大小

## 8.3 配置ThreadPoolExecutor

### 8.3.1 线程的创建于销毁

### 8.3.2 管理队列任务

`ThreadPoolExecutor`允许提供一个`BlockingQueue`来保存等待执行的任务。基本的任务排队方法有3种：无界队列、有界队列和同步移交（Synchronous Handoff）。队列的选择与其他的配置参数有关，例如线程池的大小等。

`newFixedThreadPool`和`newSingleThreadExecutor`在默认情况下使用一个无界队列`LinkedBlockingQueue`。如果所有工作线程都处于忙碌状态，那么任务将在队列中等候。如果任务持续快速地到达，并且超过了线程池处理它们的速度，那么队列将无限制地增加。

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
public LinkedBlockingQueue() {
    this(Integer.MAX_VALUE); //默认容量为integer的最大值
}
```



一种更稳妥的资源管理策略是使用有界队列，例如`ArrayBlockingQueue`、有界的`LinkedBlockingQueue`、`PriorityBlockingQueue`。有界队列有助于避免资源耗尽的情况发生，但它又带来了新的问题：当队列填满后，新的任务该怎么办？在使用有界的工作队列时，队列的大小与线程池的大小必须一起调节。如果线程池较小而队列较大，那么有助于减少内存使用量，降低CPU的使用率，同时还可以减少上下文切换，但付出的代价是可能会限制吞吐量。

对于非常大的或者无界的线程池，可以通过使用`SynchronousQueue`来避免任务排队，以及直接将任务从生产者移交给工作线程。`SynchronousQueue`不是一个真正的队列，而是一种在线程之间进行移交的机制。要将一个元素放入`SynchronousQueue`中，必须有另一个线程正在等待接受这个元素。如果没有线程正在等待，并且线程池的当前大小小于最大值，那么`ThreadPoolExecutor`将创建一个新的线程，否则根据饱和策略，这个任务将被拒绝。使用直接移交将更高效，因为任务会直接移交给执行它的线程，而不是被首先放在队列中，然后由工作线程从队列中提取该任务。只有当线程池是无界的或者可以拒绝任务时，`SynchronousQueue`才有实际价值。在`newCachedThreadPool`工厂方法中就使用了`SynchronousQueue`。

当使用像`LinkedBlockingQueue`或`ArrayBlockingQueue`这样的FIFO队列时，任务的执行顺序与它们的到达顺序相同。如果想进一步控制任务执行顺序，还可以使用`PriorityBlockingQueue`，这个队列将根据优先级来安排任务。任务的优先级是通过自然顺序或`Comparator`来定义的。

### 8.3.3 饱和策略

### 8.3.4 线程工厂

每当线程池需要创建一个线程池时，都是通过线程工厂方法来完成的。

## 8.4 扩展ThreadPoolExecutor

## 参考

