---
title: "Android Build"
date: 2019-03-14T15:18:46+08:00
draft: false
---

## D8

Android Studio 3.0 推出了d8，并在3.1正式成为默认工具。它的作用是将“.class”文件编译为Dex文件，取代之前的dx工具。

![](/images/d8.png)

## R8 

R8 在 Android Studio 3.1 中引入，志向更加高远，它的目标是取代 ProGuard 和 d8。我们可以直接使用 R8 把“.class”文件变成 Dex。
同时，R8 还支持 ProGuard 中混淆、裁剪、优化这三大功能。

## 参考
* [配置构建](https://developer.android.com/studio/build/?hl=zh-cn)
* [签署您的应用](https://developer.android.com/studio/publish/app-signing.html?hl=zh-cn#sign-manually)
* [打包流程梳理](http://mouxuejie.com/blog/2016-08-04/build-and-package-flow-introduction/)
* [腾讯多渠道打包VasDolly](https://github.com/Tencent/VasDolly)
* [微信Android资源混淆打包工具](https://mp.weixin.qq.com/s/6YUJlGmhf1-Q-5KMvZ_8_Q)
* [Android安装包相关知识汇总](https://mp.weixin.qq.com/s/QRIy_apwqAaL2pM8a_lRUQ)
* [美团多渠道打包Walle](https://github.com/Meituan-Dianping/walle)
* [apksigner](https://developer.android.com/studio/command-line/apksigner.html)
* [APK 签名方案 v2](https://source.android.com/security/apksigning/v2.html)
* [APK 签名方案 v3](https://source.android.com/security/apksigning/v3)
* [AAPT2](https://developer.android.com/studio/command-line/aapt2)
* [How to execute the dex file in android with command?](https://stackoverflow.com/questions/10199863/how-to-execute-the-dex-file-in-android-with-command)
* [d8](https://developer.android.com/studio/command-line/d8)
* [Android Studio switching to D8 dexer](https://android-developers.googleblog.com/2018/04/android-studio-switching-to-d8-dexer.html)
* [Next-generation Dex Compiler Now in Preview](https://android-developers.googleblog.com/2017/08/next-generation-dex-compiler-now-in.html)
* [The D8 dexer](https://proandroiddev.com/the-d8-dexer-6736deb55fb8)
* [ProGuard and R8: a comparison of optimizers](https://www.guardsquare.com/en/blog/proguard-and-r8)
* [redex](https://github.com/facebook/redex)
* [R8, the new code shrinker from Google, is available in Android studio 3.3 beta](https://android-developers.googleblog.com/2018/11/r8-new-code-shrinker-from-google-is.html)
* [Android 8.0 中的 ART 功能改进](https://source.android.com/devices/tech/dalvik/improvements)