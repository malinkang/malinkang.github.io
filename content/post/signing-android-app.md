---
title: 签署Android应用
date: 2017-10-12 16:20:06
draft: true
---

Android 要求所有 APK 必须先使用证书进行数字签署，然后才能安装。


### 从命令行构建和签署您的应用

#### 1.使用keytool生成一个私钥

使用以下命令可以创建一个有效期为10000天的密钥：

```
keytool -genkeypair -keyalg RSA -keysize 2048 -sigalg SHA1withRSA -validity 10000 -alias test -keystore test.keystore
```

通常我们在申请第三方分享Key时，例如微信和微博需要填写一个签名，这里的签名就是keystore的MD5值。除了使用他们提供的获取签名工具之外，还可以通过如下命令获取keystore的md5值。

```
keytool -list -v -keystore test.keystore
```
![](https://i.imgur.com/yR2BuF3.png)


使用微信获取签名工具获取：

![](https://i.imgur.com/QuRTFlX.png)


[keytool](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/keytool.html)详细使用可以参考[这里](https://ieroot.com/2013/07/29/1129.html)。


### 参考

* [签署您的应用](https://developer.android.com/studio/publish/app-signing.html#signing-manually)
* [微信开放平台Android应用签名的本质及如何获取](http://blog.csdn.net/sapce_fish/article/details/51636578)
