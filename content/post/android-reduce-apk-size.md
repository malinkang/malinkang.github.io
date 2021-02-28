---
title: Android优化之安装包大小优化
date: 2019-03-05T16:00:24+08:00
draft: false
toc: true
tags: ["Android"]
---
## 1.为什么要优化包体积

* 下载转化率
* 推广成本
* 应用市场



## 2.包体积与应用性能

包体积除了转化率的影响，它对我们应用性能还有哪些影响呢？
* 安装时间。文件拷贝、Library 解压、编译 ODEX、签名校验，特别对于 Android 5.0 和 6.0 系统来说（Android 7.0 之后有了混合编译），微信 13 个 Dex 光是编译 ODEX 的时间可能就要 5 分钟。
* 运行内存。在内存优化的时候我们就说过，Resource 资源、Library 以及 Dex 类加载这些都会占用不少的内存。
* ROM 空间。100MB 的安装包，启动解压之后很有可能就超过 200MB 了。对低端机用户来说，也会有很大的压力。

![](/images/apk-size-reduce-1.png)

事实上安装包中无非就是 Dex、Resource、Assets、Library 以及签名信息这五部分。


## 代码优化

### ProGuard

* [译 ProGuard 在 Android 上的使用姿势](https://dieyidezui.com/troubleshooting-proguard-issues-on-android/)
* [实用 ProGuard 规则示例](https://github.com/xitu/gold-miner/blob/master/TODO1/practical-proguard-rules-examples.md)


## 资源优化

### 使用WebP

* [创建 WebP 图片](https://developer.android.com/studio/write/convert-webp)
* [booster-task-compression-cwebp](https://booster.johnsonlee.io/feature/shrink/webp-compression.html)


### PNG压缩

PNG压缩工具：
* [pngcrush](https://pmt.sourceforge.io/pngcrush/)
* [pngquant](https://pngquant.org/)
* [zopfli](https://github.com/google/zopfli)
* [tinypng](https://tinypng.com/)
* [booster-task-compression-pngquant](https://booster.johnsonlee.io/feature/shrink/png-compression.html)
* [McImage](https://github.com/smallSohoSolo/McImage)

## 包体积监控

* 大小监控
* 依赖监控
* 规则监控


## 参考
* [22 | 包体积优化（上）：如何减少安装包大小？](https://time.geekbang.org/column/article/81202)
* [23 | 包体积优化（下）：资源优化的进阶实践](https://time.geekbang.org/column/article/81483)
* [Reduce the APK size](https://developer.android.com/topic/performance/reduce-apk-size)
* [Android App包瘦身优化实践 - 美团技术团队](https://tech.meituan.com/2017/04/07/android-shrink-overall-solution.html)
* [ByteX](https://github.com/bytedance/ByteX)
* [支付宝 App 构建优化解析：Android 包大小极致压缩](https://mp.weixin.qq.com/s/_gnT2kjqpfMFs0kqAg4Qig)
* [AndResGuard](https://github.com/shwenzhang/AndResGuard)
* [抖音包大小优化-资源优化](https://juejin.im/post/6844904106696376334)
* [GitHub - eleme/Mess: a gradle plugin for minifying activities, services, receivers, providers and custom view](https://github.com/eleme/Mess)
* [ProGuard 最全混淆规则说明 - 简书](https://www.jianshu.com/p/b471db6a01af)
