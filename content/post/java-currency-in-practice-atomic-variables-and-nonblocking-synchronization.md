---
title: 《Java并发编程实战》第15章原子变量与非阻塞同步机制
date: 2015-11-12 10:53:22
tags: ["Java并发编程实战"]
---

在`java.util.concurrent`包的许多类中，例如`Semaphore`和`ConcurrentLinkedQueue`，都提供了比`synchronized`机制更高的性能和可伸缩性。

<!--more-->

## 15.1 锁的劣势

## 15.2 硬件对并发的支持

### 15.2.1 比较并交换

### 15.2.2 非阻塞的计数器

### 15.2.3 JVM对CAS的支持

## 15.3 原子变量类

共有12个原子变量类，可分为4组：标量类（Scalar）、更新器类、数组类以及复合变量类。最常用的原子变量就是标量类：`AtomicInteger`、`AtomicLong`、`AtomicBoolean`以及`AtomicReference`。所有这些类都支持CAS，此外，`AtomicInteger`和`AtomicLong`还支持算数运算符。（要想模拟其他基本类型的原子变量，可以将short或byte等类型与int类型进行转换，以及使用floatToBits或doubleToLongBits来转换浮点数。）

### 15.3.1 原子变量是一种“更好的volatile”

### 15.3.2 性能比较：锁与原子变量

## 15.4 非阻塞算法

### 15.4.1 非阻塞的栈

### 15.4.2 非阻塞的链表

### 15.4.3 原子的域更新器

### 15.4.4 ABA问题

