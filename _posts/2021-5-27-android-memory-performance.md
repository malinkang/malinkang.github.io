---
title: "Android优化之内存优化"
date: 2019-03-05T15:10:02+08:00
draft: false
toc: true
tags: ["Android"]
---

## 目录

* [使用内存效率更高的代码结构](https://github.com/malinkang/AndroidInterview/blob/master/内存优化.md#使用内存效率更高的代码结构使用内存效率更高的代码结构)

  * 谨慎使用服务
  * 使用经过优化的数据容器

  * 谨慎对待代码抽象
  * 针对序列化数据使用精简版-protobuf
  * 避免内存抖动

* 移除会占用大量内存的资源和库

  * 缩减总体 apk大小
  * 使用-dagger-2-实现依赖注入
  * 谨慎使用外部库

* 减少bitmap占用的内存

  * 防止bitmap占用资源多大导致OOM
  * 图片按需加载
  * 统一的bitmap加载器
  * 图片存在像素浪费

## 使用内存效率更高的代码结构使用内存效率更高的代码结构

某些 Android 功能、Java 类和代码结构所使用的内存往往多于其他功能、类和结构。您可以在代码中选择效率更高的替代方案，以尽可能降低应用的内存使用量。

### 谨慎使用服务

在不需要某项服务时让其保持运行状态，是 Android 应用可能犯下的**最严重的内存管理错误之一**。如果您的应用需要某项[服务](https://developer.android.com/guide/components/services.html)在后台执行工作，请不要让其保持运行状态，除非其需要运行作业。请注意在服务完成任务后使其停止运行。否则，您可能会在无意中导致内存泄漏。

在您启动某项服务后，系统更倾向于让此服务的进程始终保持运行状态。这种行为会导致服务进程代价十分高昂，因为一旦服务使用了某部分 RAM，那么这部分 RAM 就不再可供其他进程使用。这会减少系统可以在 LRU 缓存中保留的缓存进程数量，从而降低应用切换效率。当内存紧张，并且系统无法维护足够的进程以托管当前运行的所有服务时，这甚至可能导致系统出现颠簸。

您通常应该避免使用持久性服务，因为它们会对可用内存提出持续性的要求。我们建议您采用 `JobScheduler`JobScheduler 等替代实现方式。要详细了解如何使用 `JobScheduler` 调度后台进程，请参阅[后台优化](https://developer.android.com/topic/performance/background-optimization.html)。

如果您必须使用某项服务，则限制此服务的生命周期的最佳方式是使用 `IntentService`，它会在处理完启动它的 intent 后立即自行结束。有关详情，请参阅[在后台服务中运行](https://developer.android.com/training/run-background-service/index.html)。

### 使用经过优化的数据容器

编程语言所提供的部分类并未针对移动设备做出优化。例如，常规 `HashMap` 实现的内存效率可能十分低下，因为每个映射都需要分别对应一个单独的条目对象。

Android 框架包含几个经过优化的数据容器，包括 `SparseArray`、`SparseBooleanArray` 和 `LongSparseArray`。 例如，`SparseArray` 类的效率更高，因为它们可以避免系统需要对键（有时还对值）进行自动装箱（这会为每个条目分别再创建 1-2 个对象）。

如果需要，您可以随时切换到原始数组以获得非常精简的数据结构

### 谨慎对待代码抽象

开发者往往会将抽象简单地当做一种良好的编程做法，因为抽象可以提高代码灵活性和维护性。不过，抽象的代价很高：通常它们需要更多的代码才能执行，需要更多的时间和更多的 RAM 才能将代码映射到内存中。因此，如果抽象没有带来显著的好处，您就应该避免使用抽象。

### 针对序列化数据使用精简版 Protobuf

[协议缓冲区](https://developers.google.com/protocol-buffers/docs/overview)是 Google 设计的一种无关乎语言和平台，并且可扩展的机制，用于对结构化数据进行序列化。该机制与 XML 类似，但更小、更快也更简单。如果您决定针对数据使用 Protobuf，则应始终在客户端代码中使用精简版 Protobuf。常规 Protobuf 会生成极其冗长的代码，这会导致应用出现多种问题，例如 RAM 使用量增多、APK 大小显著增加以及执行速度变慢。

有关详情，请参阅 [Protobuf 自述文件](https://android.googlesource.com/platform/external/protobuf/+/master/java/README.md#installation-lite-version-with-maven)中的“精简版”部分。

### 避免内存抖动

## 移除会占用大量内存的资源和库

### 缩减总体 APK 大小

您可以通过缩减应用的总体大小来显著降低应用的内存使用量。位图大小、资源、动画帧数和第三方库都会影响 APK 的大小。Android Studio 和 Android SDK 提供了可帮助您缩减资源和外部依赖项大小的多种工具。这些工具支持现代代码收缩方法，例如 [R8 编译](https://developer.android.com/studio/build/shrink-code)。（Android Studio 3.3 及更低版本使用 ProGuard，而不是 R8 编译。）

要详细了解如何缩减 APK 的总体大小，请参阅[有关如何缩减应用大小的指南](https://developer.android.com/topic/performance/reduce-apk-size.html)。

### 使用 Dagger 2 实现依赖注入

依赖注入框架可以简化您编写的代码，并提供一个可供您进行测试及其他配置更改的自适应环境。

如果您打算在应用中使用依赖注入框架，请考虑使用 [Dagger 2](http://dagger.dev/)。Dagger 不使用反射来扫描您应用的代码。Dagger 的静态编译时实现意味着它可以在 Android 应用中使用，而不会带来不必要的运行时代价或内存消耗量。

其他使用反射的依赖注入框架倾向于通过扫描代码中的注释来初始化进程。此过程可能需要更多的 CPU 周期和 RAM，并可能在应用启动时导致出现明显的延迟。

### 谨慎使用外部库

外部库代码通常不是针对移动环境编写的，在移动客户端上运行时可能效率低下。如果您决定使用外部库，则可能需要针对移动设备优化该库。在决定是否使用该库之前，请提前规划，并在代码大小和 RAM 消耗量方面对库进行分析。

即使是一些针对移动设备进行了优化的库，也可能因实现方式不同而导致问题。例如，一个库可能使用的是精简版 Protobuf，而另一个库使用的是 Micro Protobuf，导致您的应用出现两种不同的 Protobuf 实现。日志记录、分析、图像加载框架和缓存以及许多您意料之外的其他功能的不同实现都可能导致这种情况。

虽然 [ProGuard](https://developer.android.com/tools/help/proguard.html) 可以使用适当的标记移除 API 和资源，但无法移除库的大型内部依赖项。您所需要的这些库中的功能可能需要较低级别的依赖项。如果存在以下情况，这就特别容易导致出现问题：您使用某个库中的 `Activity` 子类（往往会有大量的依赖项）、库使用反射（这很常见，意味着您需要花费大量的时间手动调整 ProGuard 以使其运行）等。

此外，请避免仅针对数十个功能中的一两个功能使用共享库。您一定不希望产生大量您甚至根本用不到的代码和开销。在考虑是否使用某个库时，请查找与您的需求十分契合的实现。否则，您可以决定自己去创建实现。

## 设备分级

使用类似 device-year-class 的策略对设备分级，对于低端机用户可以关闭复杂的动画，或者是某些功能；使用 565 格式的图片，使用更小的缓存内存等。在现实环境下，不是每个用户的设备都跟我们的测试机一样高端，在开发过程我们要学会思考功能要不要对低端机开启、在系统资源吃紧的时候能不能做降级。

## 减少bitmap占用的内存

## 防止bitmap占用资源多大导致OOM
Android 2.x 系统 BitmapFactory.Options 里面隐藏的的inNativeAlloc反射打开后，申请的bitmap就不会算在external中。对于Android 4.x系统，可采用facebook的fresco库，即可把图片资源放于native中。

### 图片按需加载
即图片的大小不应该超过view的大小。在把图片载入内存之前，我们需要先计算出一个合适的inSampleSize缩放比例，避免不必要的大图载入。对此，我们可以重载drawable与ImageView，例如在Activity ondestroy时，检测图片大小与View的大小，若超过，可以上报或提示。

### 统一的bitmap加载器
Picasso、Fresco都是比较出名的加载库，同样微信也有自己的库ImageLoader。加载库的好处在于将版本差异、大小处理对使用者不感知。有了统一的bitmap加载器，我们可以在加载bitmap时，若发生OOM(try catch方式)，可以通过清除cache，降低bitmap format(ARGB8888/RBG565/ARGB4444/ALPHA8)等方式，重新尝试。

### 图片存在像素浪费
对于.9图，美工可能在出图时在拉伸与非拉伸区域都有大量的像素重复。通过获取图片的像素ARGB值，计算连续相同的像素区域，自定义算法判定这些区域是否可以缩放。关键也是需要将这些工作做到系统化，可及时发现问题，解决问题。



## 内存泄露的检测与修改



## 参考

* [内存管理概览](https://developer.android.com/topic/performance/memory-overview)
* [进程间的内存分配](https://developer.android.com/topic/performance/memory-management)
* [管理应用内存](https://developer.android.com/topic/performance/memory)
* [微信Android内存申请分析](https://mp.weixin.qq.com/s/b_lFfL1mDrNVKj_VAcA2ZA?)
* [微信Android内存优化杂谈](https://mp.weixin.qq.com/s/Z7oMv0IgKWNkhLon_hFakg)
* [微信 Android 终端内存优化实践](https://mp.weixin.qq.com/s/KtGfi5th-4YHOZsEmTOsjg?)
* [leakcanary](https://github.com/square/leakcanary)
*  [Android OOM案例分析 - 美团技术团队](https://tech.meituan.com/2017/04/14/oom-analysis.html)