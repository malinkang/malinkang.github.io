---
title: "Improve App Performance With Kotlin Coroutines"
date: 2019-10-15T19:02:36+08:00
draft: false
---

 [原文](https://developer.android.com/kotlin/coroutines) 

协程是一种并发设计模式，您可以在`Android`上使用它来简化异步执行的代码。 `Coroutines`在版本1.3中添加到`Kotlin`，并基于其他语言的既定概念。

在`Android`上，协同程序有助于解决两个主要问题：

* 管理长时间运行的任务，否则可能会阻止主线程并导致应用冻结。
* 提供主安全性，或从主线程安全地调用网络或磁盘操作。


<!--more-->

本主题描述了如何使用Kotlin协同程序解决这些问题，使您能够编写更清晰，更简洁的应用程序代码。

## 管理长时间运行的任务

在Android上，每个应用程序都有一个主线程来处理用户界面并管理用户交互。如果您的应用程序为主线程分配了太多工作，那么应用程序可能会明显卡顿或运行缓慢。网络请求，JSON解析，从数据库读取或写入，甚至只是迭代大型列表都可能导致应用程序运行缓慢，导致可见的缓慢或冻结的UI对触摸事件响应缓慢。这些长时间运行的操作应该在主线程之外运行。

以下示例显示了假设的长期运行任务的简单协同程序实现：

```kotlin
suspend fun fetchDocs() {               // Dispatchers.Main
  val result = get("https://developer.android.com") // Dispatchers.IO for `get`
  show(result)                   // Dispatchers.Main
}

suspend fun get(url: String) = withContext(Dispatchers.IO) { /* ... */ }
```

协同程序通过添加两个操作来处理长时间运行的任务，从而构建常规功能。除了`invoke`（或`call`）和返回之外，协同程序还添加了`suspend`和`resume`：

* `suspend`暂停当前协同程序的执行，保存所有局部变量。
* `resume`恢复从暂停的协同处继续执行暂停的协同程序。

您只能从其他`suspend`函数调用`suspend`函数，或者使用诸如启动之类的协程构建器来启动新的协程。

在上面的示例中，`get()`仍然在主线程上运行，但它在启动网络请求之前挂起协同程序。当网络请求完成时，`get`恢复暂停的协程，而不是使用回调来通知主线程。

`Kotlin`使用堆栈框架来管理与任何局部变量一起运行的函数。挂起协程时，将复制并保存当前堆栈帧以供以后使用。恢复时，堆栈帧将从保存位置复制回来，并且该函数将再次开始运行。即使代码看起来像普通的顺序阻塞请求，协程也可以确保网络请求避免阻塞主线程。

## Use coroutines for main-safety
`Kotlin`协程使用调度程序来确定哪些线程用于协程执行。要在主线程之外运行代码，您可以告诉Kotlin协程在Default或IO调度程序上执行工作。在Kotlin中，所有协同程序必须在调度程序中运行，即使它们在主线程上运行。协同程序可以暂停，调度程序负责恢复它们。

要指定协程应该运行的位置，Kotlin提供了三个可以使用的调度程序：

* Dispatchers.Main  - 使用此调度程序在主Android线程上运行协同程序。 这应该仅用于与UI交互并执行快速工作。 示例包括调用挂起函数，运行Android UI框架操作以及更新`LiveData`对象。
* Dispatchers.IO  - 此调度程序已经过优化，可在主线程外执行磁盘或网络I / O. 示例包括使用Room组件，读取或写入文件以及运行任何网络操作。
* Dispatchers.Default  - 此调度程序已经过优化，可以在主线程之外执行CPU密集型工作。 示例用例包括对列表进行排序和解析JSON。

继续前面的示例，您可以使用调度程序重新定义get函数。 在get的主体内部，调用withContext(Dispatchers.IO)来创建一个在IO线程池上运行的块。 放在该块中的任何代码总是通过IO调度程序执行。 由于`withContext`本身是一个挂起函数，因此函数get也是一个挂起函数。

使用协同程序，您可以调度具有细粒度控制的线程。 因为`withContext()`允许您控制任何代码行的线程池而不引入回调，所以您可以将它应用于非常小的函数，例如从数据库读取或执行网络请求。 一个好的做法是使用`withContext()`来确保每个函数都是主安全的，这意味着您可以从主线程调用该函数。 这样，调用者永远不需要考虑应该使用哪个线程来执行该函数。

在前面的示例中，`fetchDocs()`在主线程上执行; 但是，它可以安全地调用get，后者在后台执行网络请求。 因为协同程序支持挂起和恢复，所以只要`withContext`块完成，主线程上的协程就会以`get`结果恢复。

> 重要说明：使用`suspend`并不能告诉`Kotlin`在后台线程上运行函数。 暂停函数在主线程上运行是正常的。 在主线程上启动协同程序也很常见。 当您需要主安全时，例如在读取或写入磁盘，执行网络操作或运行CPU密集型操作时，应始终在挂起函数内使用`withContext()`。

与等效的基于回调的实现相比，`withContext()`不会增加额外的开销。 此外，在某些情况下，可以优化`withContext()`调用，而不是基于等效的基于回调的实现。 例如，如果一个函数对网络进行十次调用，则可以通过使用外部`withContext()`告诉Kotlin只切换一次线程。 然后，即使网络库多次使用`withContext()`，它仍然停留在同一个调度程序上，并避免切换线程。 此外，Kotlin优化了Dispatchers.Default和Dispatchers.IO之间的切换，以尽可能避免线程切换。

> 要点：使用使用Dispatchers.IO或Dispatchers.Default等线程池的调度程序并不能保证该块从上到下在同一个线程上执行。 在某些情况下，Kotlin协程可能会在暂停和恢复后将执行移动到另一个线程。 这意味着线程局部变量可能不会指向整个`withContext()`块的相同值。

## 指定CoroutineScope

定义协程时，还必须指定其CoroutineScope。 CoroutineScope管理一个或多个相关协程。 您还可以使用CoroutineScope在该范围内启动新协程。 但是，与调度程序不同，CoroutineScope不会运行协同程序。

CoroutineScope的一个重要功能是当用户离开应用程序中的内容区域时停止协程执行。 使用CoroutineScope，您可以确保正确停止任何正在运行的操作。

### 将CoroutineScope与Android架构组件配合使用

在Android上，您可以将CoroutineScope实现与组件生命周期相关联。这样可以避免泄漏内存或为与用户不再相关的`activity`或`fragment`执行额外的工作。使用Jetpack组件，它们自然适合ViewModel。由于ViewModel在配置更改（例如屏幕旋转）期间不会被销毁，因此您不必担心协同程序被取消或重新启动。

范围知道他们开始的每个协同程序。这意味着您可以随时取消在作用域中启动的所有内容。范围传播自己，所以如果一个协程开始另一个协同程序，两个协同程序具有相同的范围。这意味着即使其他库从您的范围启动协程，您也可以随时取消它们。如果您在ViewModel中运行协同程序，这一点尤为重要。如果因为用户离开了屏幕而导致ViewModel被销毁，则必须停止它正在执行的所有异步工作。否则，您将浪费资源并可能泄漏内存。如果您在销毁ViewModel后应该继续进行异步工作，则应该在应用程序架构的较低层中完成。

> 警告：通过抛出CancellationException协同取消协同程序。 在协程取消期间触发捕获异常或Throwable的异常处理程序。

使用适用于Android体系结构的KTX库组件，您还可以使用扩展属性viewModelScope来创建可以运行的协同程序，直到ViewModel被销毁。

## 启动一个协程
您可以通过以下两种方式之一启动协同程序：

* `launch`会启动一个新的协程，并且不会将结果返回给调用者。 任何被认为是“发射并忘记”的工作都可以使用`launch`来开始。
* `async`启动一个新的协同程序，并允许您使用名为`await`的挂起函数返回结果。

通常，您应该从常规函数启动新协程，因为常规函数无法调用等待。 仅在另一个协同程序内部或在挂起函数内部执行并行分解时才使用异步。

在前面的示例的基础上，这里是一个带有viewModelScope KTX扩展属性的协程，它使用launch从常规函数切换到协同程序：

```kotlin
fun onDocsNeeded() {
  viewModelScope.launch {  // Dispatchers.Main
    fetchDocs()      // Dispatchers.Main (suspend function call)
  }
}
```

> 警告：启动和异步处理异常的方式不同。 由于async期望在某个时刻最终调用await，它会保留异常并在await调用中重新抛出它们。 这意味着如果您使用await从常规函数启动新的协同程序，则可能会以静默方式删除异常。 这些丢弃的异常不会出现在崩溃指标中，也不会出现在logcat中。

### 并行分解

当函数返回时，必须停止由挂起函数启动的所有协同程序，因此您可能需要保证这些协程在返回之前完成。 通过Kotlin中的结构化并发，您可以定义一个启动一个或多个协同程序的coroutineScope。 然后，使用`await()`（对于单个协同程序）或`awaitAll()`（对于多个协程），可以保证这些协程在从函数返回之前完成。

例如，让我们定义一个以异步方式获取两个文档的coroutineScope。 通过在每个延迟引用上调用`await()`，我们保证在返回值之前两个异步操作都完成：

```kotlin
suspend fun fetchTwoDocs() =
  coroutineScope {
    val deferredOne = async { fetchDoc(1) }
    val deferredTwo = async { fetchDoc(2) }
    deferredOne.await()
    deferredTwo.await()
  }
```

即使fetchTwoDocs（）使用异步启动新的协同程序，该函数也会使用awaitAll（）等待那些启动的协同程序在返回之前完成。 但请注意，即使我们没有调用awaitAll（），coroutineScope构建器也不会恢复调用fetchTwoDocs的协程，直到所有新的协程完成。

此外，coroutineScope捕获协程抛出的任何异常并将它们路由回调用者。

有关并行分解的更多信息，请参阅编写挂起函数。

## 具有内置支持的架构组件
一些体系结构组件（包括ViewModel和Lifecycle）通过其自己的CoroutineScope成员包含对协同程序的内置支持。

例如，ViewModel包含一个内置的viewModelScope。 这提供了在ViewModel范围内启动协同程序的标准方法，如以下示例所示：

```kotlin
class MyViewModel : ViewModel() {

  fun launchDataLoad() {
    viewModelScope.launch {
      sortList()
      // Modify UI
    }
  }

  /**
  * Heavy operation that cannot be done in the Main Thread
  */
  suspend fun sortList() = withContext(Dispatchers.Default) {
    // Heavy work
  }
}

```

LiveData还使用带有liveData块的协同程序：
```kotlin
liveData {
  // runs in its own LiveData-specific scope
}
```

#