---
title: "Handler的使用和原理分析"
date: 2019-04-26T11:22:09+08:00
draft: false
tags: ["Android","技术"]
toc: true
---

## 使用Handler

每个应用程序都有自己的特殊线程来运行UI对象，例如View对象;这个线程称为UI线程。在`Android`中，不允许非UI线程访问和修改UI对象。但是在实际的开发中，很多地方要在非UI线程中修改UI对象。比如下载网络图片并将图片设置给`ImageView`。

`Handler`允许您发送和处理与线程的MessageQueue关联的Message和Runnable对象。每个`Handler`实例都与一个线程和该线程的消息队列相关联。当您创建一个新的Handler时，它被绑定到正在创建它的线程的线程/消息队列 - 从那时起，它将消息和runnables传递给该消息队列并在消息出来时执行它们队列。将`Handler`连接到UI线程时，处理消息的代码在UI线程上运行。

Handler有两个主要用途：

1. 发送消息; 
2. 在不同于发送的线程上处理消息。

发送消息通过以下方法来完成：

* post(Runnable)
* postAtTime(java.lang.Runnable, long)
* postDelayed(Runnable, Object, long)
* sendEmptyMessage(int)
* sendMessage(Message)
* sendMessageAtTime(Message, long)
* sendMessageDelayed(Message, long)

post版本允许您将Runnable对象排入队列，以便在收到它们时由消息队列调用;sendMessage版本允许您将包含将由Handler的handleMessage(Message)方法处理的数据包的Message对象排入队列。

## 原理解析

![](/images/handler.png)




### 2.3 MessageQueue

## 参考

