---
layout: posts
title: Android运行时权限
date: 2018-12-24 20:25:35
tags: ["Android"]
cover: https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/blog/images/cover/千与千寻17.png
---



权限的作用是保护 Android 用户的隐私。Android 应用必须请求权限才能访问敏感的用户数据（例如联系人和短信）以及某些系统功能（例如相机和互联网）。系统可能会自动授予权限，也可能会提示用户批准请求，具体取决于访问的功能。

Android 安全架构的设计主旨是：在默认情况下，任何应用都没有权限执行会对其他应用、操作系统或用户带来不利影响的任何操作。这包括读取或写入用户的私有数据（例如联系人或电子邮件）、读取或写入其他应用的文件、执行网络访问、使设备保持唤醒状态等。

## 权限审批

应用必须通过在[应用清单](https://developer.android.com/guide/topics/manifest/manifest-intro?hl=zh-cn)中添加 [`<uses-permission>`](https://developer.android.com/guide/topics/manifest/uses-permission-element?hl=zh-cn) 标记来公开所需的权限。例如，需要发送短信的应用会在清单中添加以下代码行：

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
          package="com.example.snazzyapp">

    <uses-permission android:name="android.permission.SEND_SMS"/>

    <application ...>
        ...
    </application>
</manifest>
```

如果您的应用在清单中列出普通权限（即不会给用户隐私或设备操作带来太大风险的权限），系统会自动将这些权限授予应用。

如果您的应用在清单中列出危险权限（即可能影响用户隐私或设备正常操作的权限），如上面的 `SEND_SMS` 权限，必须由用户明确同意授予这些权限。

### 危险权限的请求提示

仅危险权限需要用户同意。Android 请求用户授予危险权限的方式取决于用户设备上搭载的 Android 版本和应用的目标系统版本。

#### 运行时请求（Android 6.0 及更高版本）

如果设备搭载的是 Android 6.0（API 级别 23）或更高版本，并且应用的 [`targetSdkVersion`](https://developer.android.com/guide/topics/manifest/uses-sdk-element?hl=zh-cn#target) 是 23 或更高版本，用户在安装时不会收到任何应用权限的通知。您的应用必须在运行时请求用户授予危险权限。当应用请求权限时，用户会看到一个系统对话框（如图 1 左图所示），告知用户应用正在尝试访问的权限组。该对话框包括**拒绝**和**允许**按钮。

如果用户拒绝权限请求，当应用下次请求该权限时，该对话框将包含一个复选框，选中它即表示用户不想再收到权限提示（请参阅图 1 右图）。

![**图 1.** 初始权限对话框（左）和包含关闭进一步请求的选项的二次权限请求（右）](https://developer.android.com/images/permissions/runtime_permission_request_2x.png?hl=zh-cn)

如果用户选中**不再询问**复选框并点按**拒绝**，当您以后尝试请求相同权限时，系统不会再提示用户。

即使用户授予应用所请求的权限，您也不能指望始终拥有该权限。用户也可以选择在系统设置中逐一启用和停用权限。您应始终在运行时检查并请求权限，以防发生运行时错误 (`SecurityException`)。



#### 安装时请求（Android 5.1.1 及更低版本）

如果设备搭载的是 Android 5.1.1（API 级别 22）或更低版本，或者应用在任何版本的 Android 上运行时其 [`targetSdkVersion`](https://developer.android.com/guide/topics/manifest/uses-sdk-element?hl=zh-cn#target) 是 22 或更低版本，系统将在安装时自动请求用户向应用授予所有危险权限（请参见图 2）。

![**图 2.** 安装时权限对话框](https://developer.android.com/images/permissions/install_time_permissions_dialog_2x.png?hl=zh-cn)

如果用户点击**接受**，系统将授予应用请求的所有权限。如果用户拒绝权限请求，系统将取消安装应用。



## 保护级别

权限分为几个保护级别。保护级别影响着是否需要运行时权限请求。

有三种保护级别会影响第三方应用：普通、签名和危险权限。如需查看特定权限所拥有的保护级别，请访问[权限 API 参考页面](https://developer.android.com/reference/android/Manifest.permission?hl=zh-cn)。

### 普通权限

普通权限涵盖以下情况：应用需要访问其沙盒外部的数据或资源，但对用户隐私或其他应用的操作带来的风险很小。例如，设置时区的权限就是普通权限。

如果应用在清单中声明需要普通权限，系统会在安装时自动向应用授予该权限。系统不会提示用户授予普通权限，用户也无法撤消这些权限。

### 签名权限

系统在安装时授予这些应用权限，但仅会在尝试使用某权限的应用签名证书为定义该权限的同一证书时才会授予。

### 危险权限

危险权限涵盖以下情况：应用需要的数据或资源涉及用户隐私信息，或者可能对用户存储的数据或其他应用的操作产生影响。例如，能够读取用户的联系人属于危险权限。如果应用声明其需要危险权限，必须由用户向应用明确授予该权限。在用户批准该权限之前，应用无法提供依赖于该权限的功能。



## 特殊权限

有几项权限的行为与普通权限及危险权限都不同。`SYSTEM_ALERT_WINDOW` 和 `WRITE_SETTINGS` 特别敏感，因此大多数应用不应该使用它们。

在大多数情况下，应用必须在清单中声明 `SYSTEM_ALERT_WINDOW` 权限，并发送请求用户授权的 intent。系统将向用户显示详细管理屏幕，以响应该 intent。

从 Android 11（API 级别 30）开始，调用 `ACTION_MANAGE_OVERLAY_PERMISSION` intent 操作的应用不能指定软件包。当应用调用包含此 intent 操作的 intent 时，必须由用户先选择想要授予或撤消哪些应用的权限。此行为可以让权限的授予更有目的性，从而达到保护用户的目的。

## 检查权限

如果应用需要一项危险权限，那么每次执行需要该权限的操作时，您都必须检查是否具有该权限。在` Android 6.0`（API 级别 23）及更高版本中，用户可以随时从任何应用撤消危险权限。

### 确定应用是否已获得权限

如需检查用户是否已向您的应用授予特定权限，请将该权限传入 `ContextCompat.checkSelfPermission()`方法。根据您的应用是否具有相应权限，此方法会返回 `PERMISSION_GRANTED`或 `PERMISSION_DENIED`。

```kotlin
if (ContextCompat.checkSelfPermission(this,Manifest.permission.READ_PHONE_STATE) ==  PackageManager.PERMISSION_DENIED) {
    //权限被拒
}
```

### 说明您的应用为何需要获取权限

如果 `ContextCompat.checkSelfPermission()` 方法返回 `PERMISSION_DENIED`，请调用 `shouldShowRequestPermissionRationale()`。如果此方法返回 `true`，请向用户显示指导界面。请在此界面中说明用户希望启用的功能为何需要特定权限。

```kotlin
//用户授权成功或者点击禁止不再询问时返回false
ActivityCompat.shouldShowRequestPermissionRationale(this, Manifest.permission.READ_PHONE_STATE)
```

### 请求权限

用户查看指导界面后或者 `shouldShowRequestPermissionRationale()` 的返回值表明您这次不需要显示指导界面后，您可以请求权限。用户会看到系统权限对话框，并可在其中选择是否向您的应用授予特定权限。

按照历来的做法，您可以在权限请求过程中**自行管理请求代码**，并将此请求代码包含在您的权限回调逻辑中。另一种选择是使用 AndroidX 库中包含的 [`RequestPermission`](https://developer.android.com/reference/androidx/activity/result/contract/ActivityResultContracts.RequestPermission?hl=zh-cn) 协定类，您可在其中**允许系统代为管理权限请求代码**。由于使用 `RequestPermission` 协定类可简化逻辑，因此，**建议您尽可能使用该方法**。

#### 允许系统管理权限请求代码

如需允许系统管理与权限请求相关联的请求代码，请在您模块的 `build.gradle` 文件中[添加 `androidx.activity` 库的依赖项](https://developer.android.com/jetpack/androidx/releases/activity?hl=zh-cn#declaring_dependencies)。请使用该库的 1.2.0 版或更高版本。

```groovy
dependencies {
    implementation "androidx.activity:activity-ktx:1.2.0-beta01"
  	// 不依赖会报错 https://stackoverflow.com/questions/62771948/new-result-api-error-can-only-use-lower-16-bits-for-requestcode
    implementation 'androidx.fragment:fragment-ktx:1.3.0-beta01'
}
```

然后，您可以使用以下某个类：

- 如需请求一项权限，请使用 [`RequestPermission`](https://developer.android.com/reference/androidx/activity/result/contract/ActivityResultContracts.RequestPermission?hl=zh-cn)。
- 如需同时请求多项权限，请使用 [`RequestMultiplePermissions`](https://developer.android.com/reference/androidx/activity/result/contract/ActivityResultContracts.RequestMultiplePermissions?hl=zh-cn)。

以下步骤显示了如何使用 `RequestPermission` 协定类。使用 `RequestMultiplePermissions` 协定类的流程几乎与此相同。

1. 在 Activity 或 Fragment 的初始化逻辑中，将 [`ActivityResultCallback`](https://developer.android.com/reference/androidx/activity/result/ActivityResultCallback?hl=zh-cn) 的实现传入对 `registerForActivityResult()` 的调用。`ActivityResultCallback` 定义应用如何处理用户对权限请求的响应。保留对 `registerForActivityResult()`（类型为 [`ActivityResultLauncher`](https://developer.android.com/reference/androidx/activity/result/ActivityResultLauncher?hl=zh-cn)）的返回值的引用。

2. 如需在必要时显示系统权限对话框，请对您在上一步中保存的 `ActivityResultLauncher` 实例调用 [`launch()`](https://developer.android.com/reference/androidx/activity/result/ActivityResultLauncher?hl=zh-cn#launch(I)) 方法。

   调用 `launch()` 之后，系统会显示系统权限对话框。当用户做出选择后，系统会异步调用您在上一步中定义的 `ActivityResultCallback` 实现。

以下代码段展示了如何处理权限响应：

```kotlin
// Register the permissions callback, which handles the user's response to the
// system permissions dialog. Save the return value, an instance of
// ActivityResultLauncher. You can use either a val, as shown in this snippet,
// or a lateinit var in your onAttach() or onCreate() method.
val requestPermissionLauncher =
    registerForActivityResult(RequestPermission()
    ) { isGranted: Boolean ->
        if (isGranted) {
            // Permission is granted. Continue the action or workflow in your
            // app.
        } else {
            // Explain to the user that the feature is unavailable because the
            // features requires a permission that the user has denied. At the
            // same time, respect the user's decision. Don't link to system
            // settings in an effort to convince the user to change their
            // decision.
        }
    }
```

以下代码段演示了检查权限并根据需要向用户请求权限的建议流程：

```kotlin
when {
    ContextCompat.checkSelfPermission(
            CONTEXT,
            Manifest.permission.REQUESTED_PERMISSION
            ) == PackageManager.PERMISSION_GRANTED -> {
        // You can use the API that requires the permission.
    }
    shouldShowRequestPermissionRationale(...) -> {
        // In an educational UI, explain to the user why your app requires this
        // permission for a specific feature to behave as expected. In this UI,
        // include a "cancel" or "no thanks" button that allows the user to
        // continue using your app without granting the permission.
        showInContextUI(...)
    }
    else -> {
        // You can directly ask for the permission.
        // The registered ActivityResultCallback gets the result of this request.
        requestPermissionLauncher.launch(
                Manifest.permission.REQUESTED_PERMISSION)
    }
}
```

#### 自行管理权限请求代码

作为[允许系统管理权限请求代码](https://developer.android.com/training/permissions/requesting?hl=zh-cn#allow-system-manage-request-code)的替代方法，您可以自行管理权限请求代码。为此，请在对 `requestPermissions()` 的调用中添加请求代码。

以下代码段演示了如何使用请求代码来请求权限：

```kotlin
when {
    ContextCompat.checkSelfPermission(
            CONTEXT,
            Manifest.permission.REQUESTED_PERMISSION
            ) == PackageManager.PERMISSION_GRANTED -> {
        // You can use the API that requires the permission.
        performAction(...)
    }
    ActivityCompat.shouldShowRequestPermissionRationale(...) -> {
        // In an educational UI, explain to the user why your app requires this
        // permission for a specific feature to behave as expected. In this UI,
        // include a "cancel" or "no thanks" button that allows the user to
        // continue using your app without granting the permission.
        showInContextUI(...)
    }
    else -> {
        // You can directly ask for the permission.
        ActivityCompat.requestPermissions(CONTEXT,
                arrayOf(Manifest.permission.REQUESTED_PERMISSION),
                REQUEST_CODE)
    }
}
```

当用户响应系统权限对话框后，系统就会调用应用的 `onRequestPermissionsResult()` 实现。系统会传入用户对权限对话框的响应以及您定义的请求代码，如以下代码段所示：

```kotlin
override fun onRequestPermissionsResult(requestCode: Int,
        permissions: Array<String>, grantResults: IntArray) {
    when (requestCode) {
        PERMISSION_REQUEST_CODE -> {
            // If request is cancelled, the result arrays are empty.
            if ((grantResults.isNotEmpty() &&
                    grantResults[0] == PackageManager.PERMISSION_GRANTED)) {
                // Permission is granted. Continue the action or workflow
                // in your app.
            } else {
                // Explain to the user that the feature is unavailable because
                // the features requires a permission that the user has denied.
                // At the same time, respect the user's decision. Don't link to
                // system settings in an effort to convince the user to change
                // their decision.
            }
            return
        }

        // Add other 'when' lines to check for other
        // permissions this app might request.
        else -> {
            // Ignore all other requests.
        }
    }
}
```

## 单次授权

从 **Android 11（API 级别 30）**开始，每当您的应用请求与位置、麦克风或相机相关的权限时，面向用户的权限对话框都会包含**仅限这一次**选项，如图 1 所示。如果用户在对话框中选择此选项，系统会向应用授予临时的单次授权。

![在该对话框中有三个按钮，“仅限这一次”选项是其中的第二个按钮。](https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/blog/images/leetcode/one-time-prompt.svg)

然后，应用可以在一段时间内访问相关数据，具体时间取决于应用的行为和用户的操作：

- 当应用的 Activity 可见时，应用可以访问相关数据。
- 如果用户将应用转为后台运行，应用可以在短时间内继续访问相关数据。
- 如果您在 Activity 可见时启动了一项前台服务，并且用户随后将您的应用转到后台，那么您的应用可以继续访问相关数据，直到该前台服务停止。
- 如果用户撤消单次授权（例如在系统设置中撤消），无论您是否启动了前台服务，应用都无法访问相关数据。与任何权限一样，如果用户撤消了应用的单次授权，应用进程就会终止。

当用户下次打开应用并且应用中的某项功能请求访问位置信息、麦克风或摄像头时，系统会再次提示用户授予权限。

## 自动重置未使用的应用的权限

如果应用以 Android 11（API 级别 30）或更高版本为目标平台并且数月未使用，系统会通过自动重置用户已授予应用的运行时敏感权限保护用户数据。此操作与用户在系统设置中查看权限并将应用的访问权限级别更改为**拒绝**的做法效果一样。

如果应用遵循了有关[在运行时请求权限](https://developer.android.com/training/permissions/requesting?hl=zh-cn)的最佳做法，那么您不必对应用进行任何更改。

## 参考

* [权限概览](https://developer.android.com/guide/topics/permissions/overview?hl=zh-cn)
* [声明应用权限](https://developer.android.com/training/permissions/declaring?hl=zh-cn)
* [请求应用权限](https://developer.android.com/training/permissions/requesting?hl=zh-cn)
* [应用权限最佳做法](https://developer.android.com/training/permissions/usage-notes?hl=zh-cn)
* [仅在默认处理程序中使用的权限](https://developer.android.com/guide/topics/permissions/default-handlers?hl=zh-cn)
* [定义自定义应用权限](https://developer.android.com/guide/topics/permissions/defining?hl=zh-cn)

## 代码

* [permissions-samples](https://github.com/android/permissions-samples)
* [RxPermissions](https://github.com/tbruyelle/RxPermissions)
* [PermissionsDispatcher](https://github.com/permissions-dispatcher/PermissionsDispatcher)
* [easypermissions](https://github.com/googlesamples/easypermissions)

