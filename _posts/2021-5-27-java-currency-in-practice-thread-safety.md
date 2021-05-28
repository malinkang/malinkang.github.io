---
title: 《Java并发编程实战》第2章线程安全性
date: 2015-09-24 10:53:22
tags: ["Java并发编程实战"]
categories: ["Java"]
---

一个对象是否需要是线程安全的，取决于它是否被多个线程访问。这指的是在程序中访问对象的方式，而不是对象要实现的功能。要使得对象时线程安全的，需要采用同步机制来协同对对象可变状态的访问。如果无法实现协同，那么可能会导致数据破坏以及其他不该出现的结果。

<!--more-->

当多个线程访问某个状态变量并且其中有一个线程执行写入操作时，必须采用同步机制来协同这些线程对变量的访问。`Java`中的主要同步机制是关键字`synchronized`，它提供了一种独占的加锁方式，但“同步”这个术语还包括volatile类型的变量，显式锁（Explicit Lock）以及原子变量。

## 2.1 什么是线程安全性

### 2.1 什么是线程安全性

## 2.2 原子性

### 2.2.1 竞态条件

### 2.2.2 延迟初始化中的竞态条件

### 2.2.3 复合操作

## 2.3 加锁机制

### 2.3.1 内置锁

`Java`提供了一种内置的锁机制来支持原子性：`同步代码块（Synchronized Block）`。同步代码块包括两部分：一个作为锁的对象引用，一个作为由这个锁保护的代码块。以关键字`synchronized`来修饰的方法就是一种横跨整个方法体的同步代码块，其中该同步代码块的锁就是方法所在的对象。静态的`synchronized`方法以`Class`对象作为锁。

```java
synchronized(lock){
  //访问或修改由锁保护的共享状态
}
```

每个Java对象都可以用做一个实现同步的锁，这些锁被称为`内置锁（Intrinsic Lock）`或`监视器锁（Monitor Lock）`。线程在进入同步代码块之前会自动获得锁，**并且在退出同步代码块时自动释放锁，而无论是通过正常的控制路径退出，还是通过从代码块中抛出异常退出**。获得内置锁的唯一途径就是进入由这个锁保护的同步代码块或方法。

Java的内置锁相当于一种互斥体，这意味着最多只有一个线程能持有这种锁。当线程A尝试获取一个由线程B持有的锁时，线程A必须等待或者阻塞，直到线程B释放这个锁。如果B永远不释放锁，那么A也将永远地等下去。

由于每次只能有一个线程执行内置锁保护的代码块。因此，由这个锁保护的同步代码块会以原子方式执行，多个线程在执行该代码块时也不会互相干扰。并发环境的原子性与事务应用程序中的原子性有着相同的含义，**一组语句作为一个不可分割的单元被执行**。任何一个执行同步代码块的线程，都不可能看到有其他线程正在执行由同一个锁保护的同步代码块。

#### 修饰实例方法

这种情况下的锁对象是`this`即当前实例对象，因此只有同一个实例对象调用此方法才会产生互斥效果，不同实例对象之间不会有互斥效果。

```java
public class Counter {
    private int count;

    public synchronized void add() {
        for (int i = 0; i < 5; i++) {
            count++;
            System.out.println("add " + hashCode() + ":" + Thread.currentThread().getName() + " : " + count);
        }
    }
    public static void main(String[] args) {
        Counter c1 = new Counter();
        Counter c2 = new Counter();
        new Thread(c1::add).start();
        new Thread(c1::add).start();
        new Thread(c2::add).start();
        new Thread(c2::add).start();
    }
}
```



#### 修饰静态类方法

```java
public class Counter {
    private static int count;

    public static synchronized void add() {
        for (int i = 0; i < 5; i++) {
            count++;
            System.out.println("add " + Thread.currentThread().getName() + " : " + count);
        }
    }
    public static void main(String[] args) {
        new Thread(Counter::add).start();
        new Thread(Counter::add).start();
    }
}
```



#### 修饰代码块

```java
public class Counter {
    private int count;

    public void add() {
        synchronized (this) {
            for (int i = 0; i < 5; i++) {
                count++;
                System.out.println("add " + hashCode() + ":" + Thread.currentThread().getName() + " : " + count);
            }
        }
    }

    public static void main(String[] args) {
        Counter c1 = new Counter();
        Counter c2 = new Counter();
        new Thread(c1::add).start();
        new Thread(c1::add).start();
        new Thread(c2::add).start();
        new Thread(c2::add).start();
    }
}
```

#### 实现细节

```java
public class Counter {
    public void add() {
        synchronized (this) {

        }
    }
}
```

使用javap查看上述代码的字节码

![image-20210127144341866](https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/blog/images/image-20210127144341866.png)

上面字节码中有 1 个 monitorenter 和 2 个 monitorexit。这是因为虚拟机需要保证当异常发生时也能释放锁。因此 2 个 monitorexit 一个是代码正常执行结束后释放锁，一个是在代码执行异常时释放锁。

修改代码

```
public class Counter {
    public synchronized void add() {
    }
}
```

![image-20210127144700271](https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/blog/images/image-20210127144700271.png)

从图中可以看出，被 synchronized 修饰的方法在被编译为字节码后，在方法的 flags 属性中会被标记为 ACC_SYNCHRONIZED 标志。当虚拟机访问一个被标记为 ACC_SYNCHRONIZED 的方法时，会自动在方法的开始和结束（或异常）位置添加 monitorenter 和 monitorexit 指令。

关于 monitorenter 和 monitorexit，可以理解为一把具体的锁。在这个锁中保存着两个比较重要的属性：计数器和指针。

* 计数器代表当前线程一共访问了几次这把锁；
* 指针指向持有这把锁的线程。

![Cgq2xl6X-COAEskYAABd1Qkprak432](https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/blog/images/Cgq2xl6X-COAEskYAABd1Qkprak432.png)

锁计数器默认为0，当执行monitorenter指令时，如锁计数器值为0 说明这把锁并没有被其它线程持有。那么这个线程会将计数器加1，并将锁中的指针指向自己。当执行monitorexit指令时，会将计数器减1。

### 2.3.2 重入

可重入锁指的是线程当前已经持有这把锁了，能在不释放这把锁的情况下，再次获取这把锁。同理，不可重入锁指的是虽然线程当前持有了这把锁，但是如果想再次获取这把锁，也必须要先释放锁后才能再次尝试获取。

当某个线程请求一个由其他线程持有的锁时，发出请求的线程就会阻塞。然而，由于内置锁是可重入的，因此如果某个线程试图获得一个已经由它自己持有的锁，那么这个请求就会成功。重入的一种实现方法是，为每个锁关联一个获取计数值和一个所有者线程。当计数值为0时，这个锁就被认为是没有被任何线程持有。当线程请求一个未被持有的锁时，JVM将记下锁的持有者，并且将获取计数值置为1。如果同一个线程再次获取这个锁，计数值将递增，而当线程退出同步代码块时，计数器会相应地递减。当计数值为0时，这个锁将被释放。

## 2.4 用锁来保护状态

## 2.5 活跃性与性能



## 参考

* [不可不说的Java“锁”事](https://tech.meituan.com/2018/11/15/java-lock.html)
* [synchronized 实现原理](https://xiaomi-info.github.io/2020/03/24/synchronized/)
* [关键字: synchronized详解](https://www.pdai.tech/md/java/thread/java-thread-x-key-synchronized.html)