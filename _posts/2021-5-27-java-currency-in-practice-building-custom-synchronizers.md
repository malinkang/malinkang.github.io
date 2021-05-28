---
title: 《Java并发编程实战》第14章构建自定义的同步工具
date: 2015-11-05 10:53:22
tags: ["Java并发编程实战"]
categories: ["Java"]
---

类库包含了许多存在状态依赖性的类，例如`FutureTask`、`Semaphore`和`BlockingQueue`等。在这些类的一些操作中有着基于状态的前提条件，例如，不能从一个空的队列中删除元素，或者获取一个尚未结束的任务的计算结果，在这些操作可以执行之前，必须等到队列进入“非空”状态，或者任务进入“已完成”状态。

创建状态依赖类的最简单方法通常是在类库中现有状态依赖类的基础上进行构造。如果类库没有提供你需要的功能，那么还可以使用`Java`语言和类库提供的底层机制来构造自己的同步机制，包括内置的条件队列、显式的`Condition`对象以及`AbstractQueuedSynchronizer`框架。


<!--more-->

## 14.1 状态依赖性的管理


## 14.4 Synchronizer剖析

在`ReentrantLock`和`Semaphore`这两个接口之间存在许多共同点。这两个类都可以用做一个“阀门”，即每次只允许一定数量的线程通过，并当线程到达阀门时，可以通过（在调用lock或acquire时成功返回），也可以等待（在调用lock或acquire时阻塞），还可以取消（在调用tryLock或tryAcquire时返回“假”，表示在指定的时间内锁是不可用的或者无法获得许可）。而且这两个接口都支持可中断的、不可中断的以及限时的获取操作，并且也都支持等待线程执行公平或非公平的队列操作。

列出了这种共性后，你或许会认为`Semaphore`是基于`ReentrantLock`实现的，或者认为`ReentrantLock`实际上是带有一个许可的`Semaphore`。

事实上，它们在实现时都使用了一个共同的基类，即`AbstarctQueuedSynchronizer(AQS)`，这个类也是其他许多同步类的基类。AQS是一个用于构建锁和同步器的框架，许多同步器都可以通过AQS很容易并且高效地构造出来。不仅`ReentrantLock`和`Semaphore`是基于AQS构建的，还包括`CountDownLatch`、`ReentrantReadWriteLOck`、`SynchronousQueue`和`FutureTask`。

AQS解决了在实现同步器时设计的大量细节问题，例如等待线程采用FIFO队列操作顺序。在不同的同步器中还可以定义一些灵活的标准来判断某个线程是应该通过还是需要等待。

基于AQS来构建同步器能带来许多好处。它不仅能极大地减少实现工作，而且也不必处理在多个位置上发生的竞争问题（这是在没有使用AQS来构建同步器时的情况）。在基于AQS构建的同步器中，只可能在一个时刻发生阻塞，从而降低上下文切换的开销，并提高吞吐量。在设计AQS时充分考虑了可伸缩性，因此`java.util.concurrent`中所有基于AQS构建的同步器都能获得这个优势。

## 14.5 AbstractQueuedSynchronizer

大多数开发者都不会直接使用AQS，标准同步器类的集合能够满足大多数情况的需求。但如果能了解标准同步器类的实现方式，那么对于理解它们的工作原理是非常有帮助的。

在基于AQS构建的同步器类中，最基本的操作包括各种形式的获取操作和释放操作。获取操作是一种依赖状态的操作，并且通常会阻塞。当使用锁或信号量时，“获取”操作的含义就很直观，即获取的是锁或者许可，并且调用者可能会一直等待直到同步器类处于可被获取的状态。在使用`CountDownLatch`时，“获取”操作意味着“等待并直到闭锁到达结束状态”，而在使用`FutureTask`时，则 意味着“等待并直到任务已经完成”。“释放”并不是一个可阻塞的操作，当执行“释放”操作时，所有在请求时被阻塞的线程都会开始执行。

如果一个类想成为状态依赖的类，那么它必须拥有一些状态。AQS负责管理同步器类中的状态，它管理了一个整数状态信息，可以通过getState、setState以及compareAndSetState等protected类型方法来进行操作。这个整数可以用于表示任意状态。例如ReentrantLock用它来表示所有者线程已经重复获取该锁的次数，`Semaphore`用它来表示剩余的许可数量，`FutureTask`用它来表示任务的状态（尚未开始、正在运行、已完成以及已取消）。在同步器类中还可以自行管理一些额外的状态变量，例如，`ReentrantLock`保存了锁的当前所有者的信息，这样就能区分某个获取操作是重入的还是竞争的。

根据同步器的不同，获取操作可以是一种独占操作（例如`ReentrantLock`），也可以是一种非独占操作（例如Semaphore和CountDownLatch）。一个获取操作包括两部分。首先，同步器判断当前状态是否允许获得操作，如果是，则允许线程执行，否则获取操作将阻塞或失败。这种判断是由同步器的语义决定的。

如果某个同步器支持独占的获取操作，那么需要实现一些保护方法，包括`tryAcquire`、`tryRelease`和`isHeldExclusively`等，而对于支持共享获取的同步器，则应该实现`tryAcquireShared`和`tryReleaseShared`等方法。AQS中的`acquire`、`acquireShared`、`release`和`releaseShared`等方法都将调用这些方法在子类中带有前缀try的版本来判断某个操作是否能执行。在同步器的子类中，可以根据其获取操作和释放操作的语义，使用`getSate`、`setState`以及`compareAndSetState`来检查和更新状态，并通过返回的状态值来告诉基类“获取”或“释放”同步器的操作是否成功。例如，如果`tryAcquireShared`返回一个负值，那么表示获取操作失败，返回零值表示同步器通过独占方式获取，返回正直表示同步器通过非独占方式被获取。对于`tryRelease`和`tryReleaseShared`方法来说，如果释放操作使得所有在获取同步器时被阻塞的线程恢复执行，那么这两个方法应该返回`true`。



## 14.6 java.util.concurrent同步器类中的AQS

### 14.6.1 ReentrantLock

### 14.6.2 Semaphore与CountDownLatch

CountDownLatch使用AQS的方式与Semaphore很相似：在同步状态中保存的是当前的计数值。`countDown`方法调用release，从而导致计数值递减，并且当计数值为零时，解除所有等待线程的阻塞。`await`调用`acquire`，当计数器为零时，`acquire`将立即返回，否则将阻塞。



#### CountDownLatch源码分析

内部类`Sync`

```java
private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;
    Sync(int count) {
        setState(count);//调用AQS的setState
    }
    int getCount() {
        return getState();//调用AQS的getState
    }
  	//判断是否获取
    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }
    //判断是否释放
    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)//如果状态等于0直接返回false
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}
private final Sync sync;
```

构造函数

```java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count); //创建sync
}
```

CountDownLatch的`countDown`方法：

```java
public void countDown() {
    sync.releaseShared(1); //调用AQS的releaseShared
}
```

AQS的`releaseShared`方法：

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) { //如果可以释放
        doReleaseShared(); //执行释放
        return true;
    }
    return false;
}
```

CountDownLatch的await方法

```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1); //调用AQS的方法
}
```

```java
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
```











### 14.6.3 FutureTask

### 14.6.4 ReentrantReadWriteLock

