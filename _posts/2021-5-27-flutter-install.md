---
title: "安装Flutter"
date: 2018-12-18T13:37:22+08:00
draft: true
---
## 1.安装SDK

1. [官网](https://flutter.dev/docs/development/tools/sdk/releases#macos)下载其最新可用的安装包并解压。

2. 设置环境变量

在`.bash_profile`文件中写入`export PATH=/Users/malk/Documents/flutter/bin:$PATH`，然后保存并执行`source .bash_profile`。

3. 运行`flutter doctor`命令查看是否需要安装其它依赖项来完成安装


该命令检查您的环境并在终端窗口中显示报告。`Dart SDK`已经在捆绑在`Flutter`里了，没有必要单独安装Dart。

![](/images/flutter-doctor-command-line.png)

## 2.配置Android Studio

需要安装两个插件:

* Flutter插件： 支持Flutter开发工作流 (运行、调试、热重载等).
* Dart插件： 提供代码分析 (输入代码时进行验证、代码补全等).

## 3.创建 Flutter Project

1. 选择 File>New Flutter Project
2. 选择 Flutter application 作为 project 类型, 然后点击 Next
3. 输入项目名称 (如 myapp), 然后点击 Next
4. 点击 Finish
5. 等待Android Studio安装SDK并创建项目.

## 4.运行项目

可以通过点击Android Studio的run按钮来运行。也可以通过flutter命令来运行。

```shell
flutter devices //查看连接设备
flutter run //运行项目
```

## 遇到的问题

* [Flutter配置好后在Android Studio中找不到设备](https://stackoverflow.com/questions/49222658/device-list-doesnt-shows-in-android-studio-using-flutter)
* [Waiting for another flutter command to release the startup lock](https://stackoverflow.com/questions/51679269/waiting-for-another-flutter-command-to-release-the-startup-lock)