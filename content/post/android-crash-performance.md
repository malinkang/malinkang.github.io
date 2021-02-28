---
title: Android崩溃优化
date: 2019-03-05T10:08:28+08:00
draft: false
toc: true
tags: ["Android"]
---

## Android的两种崩溃

Android崩溃分为Java崩溃和Native崩溃。Java崩溃就是在Java代码中，出现了未捕获异常，导致程序异常退出。Native崩溃一般都是因为在Native代码中访问非法地址，也可能是地址对齐出现的问题，或者发生了程序主动abort，这些都会产生相应的signal信号，导致程序异常退出。

### 1.Native崩溃的捕获流程

![完整的Native崩溃从捕获到解析要经历的流程](/images/android-performance/native-exception-catch-sequence.jpeg)

### 2.Native崩溃捕获的难点


### 3.选择合适的崩溃服务

* [腾讯Bugly](https://bugly.qq.com/v2/)
* [UC啄木鸟](https://wpk.uc.cn/)
* [网易云捕](http://crash.163yun.com/)
* [Google Firbase](http://crash.163yun.com/)


## 崩溃现场

### 1.崩溃信息

### 2.系统信息

### 3.内存信息

### 4.资源信息

## 崩溃分析

### 第一步：确定重点

#### 1.确认严重程度

#### 2.崩溃基本信息

#### 3.Logcat

#### 4.各个资源情况

### 第二步：查找共性

### 第三步：尝试复现