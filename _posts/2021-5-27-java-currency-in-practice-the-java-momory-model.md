---
title: 《Java并发编程实战》第16章Java内存模型
date: 2015-11-19 10:53:22
tags: ["Java并发编程实战"]
cover: https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/blog/images/cover/千与千寻02.png
---

本章将介绍Java内存模型的底层需求以及锁提供的保证，此外还将介绍在本书给出的一些高层设计原则背后的原理。

<!--more-->

## 16.1 什么是内存模型，为什么需要它

### 16.1.1 平台的内存模型

### 16.1.2 重排序

### 16.1.3 Java内存模型简介

### 16.1.4 借助同步

## 16.2 发布

### 16.2.1 不安全的发布

### 16.2.2 安全的发布

### 16.2.3 安全初始化模式

### 16.2.4 双重检查加锁

双重校验锁写法

```java
public class Singleton {
    private static volatile Singleton singleton;

    private Singleton() {
    }

    public static Singleton getInstance() {
        //①
        if (singleton == null) {
          	//②
            synchronized (Singleton.class) {
                //③
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```

第一个check避免多次进入同步效率低下。

第二个check，当A、B线程都处在②的位置，线程A进入同步代码块执行完成后，B进入如果没有第二次判断，仍然会创建一个新的实例。

使用volatile主要就在于` singleton = new Singleton() `，它并非是一个原子操作，事实上，在 JVM 中上述语句至少做了以下这 3 件事：

* 第一步是给 singleton 分配内存空间；
* 然后第二步开始调用 Singleton 的构造函数等，来初始化 singleton；
* 最后第三步，将 singleton 对象指向分配的内存空间（执行完这步 singleton 就不是 null 了）。

![img](https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/blog/images/Cgq2xl6BpWCAMBaVAACFIdffjfM852.png)

这里需要留意一下 1-2-3 的顺序，因为存在指令重排序的优化，也就是说第2 步和第 3 步的顺序是不能保证的，最终的执行顺序，可能是 1-2-3，也有可能是 1-3-2。

如果是 1-3-2，那么在第 3 步执行完以后，singleton 就不是 null 了，可是这时第 2 步并没有执行，singleton 对象未完成初始化，它的属性的值可能不是我们所预期的值。假设此时线程1还没执行完，线程 2 进入 getInstance 方法，由于 singleton 已经不是 null 了，所以会通过第一重检查并直接返回，但其实这时的 singleton 并没有完成初始化，所以使用这个实例的时候会报错，详细流程如下图所示：

![img](https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/blog/images/Cgq2xl6BpWCAB6QQAAEKacFd0CE542.png)

线程 1 首先执行新建实例的第一步，也就是分配单例对象的内存空间，由于线程 1 被重排序，所以执行了新建实例的第三步，也就是把 singleton 指向之前分配出来的内存地址，在这第三步执行之后，singleton 对象便不再是 null。

这时线程 2 进入 getInstance 方法，判断 singleton 对象不是 null，紧接着线程 2 就返回 singleton 对象并使用，由于没有初始化，所以报错了。最后，线程 1 “姗姗来迟”，才开始执行新建实例的第二步——初始化对象，可是这时的初始化已经晚了，因为前面已经报错了。

使用了 volatile 之后，相当于是表明了该字段的更新可能是在其他线程中发生的，因此应确保在读取另一个线程写入的值时，可以顺利执行接下来所需的操作。在 JDK 5 以及后续版本所使用的 JMM 中，在使用了 volatile 后，会一定程度禁止相关语句的重排序，从而避免了上述由于重排序所导致的读取到不完整对象的问题的发生。

