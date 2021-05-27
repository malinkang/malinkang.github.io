---
title: React Native for Android 入门
category: Program
date: 2016-02-29 9:38:05
draft: true
---


### 环境配置
* 安装Homebrew：Homebrew是Mac的一个包管理器。

* 安装Node.js

	* 通过nvm安装Node.js
	nvm是Node.js的版本管理器，首先安装使用Homebrew安装nvm
	
	```sh
	brew install nvm
	```
	配置nvm
	```sh
	  mkdir ~/.nvm
	  export NVM_DIR=~/.nvm. $(brew --prefix nvm)/nvm.sh
	```
	安装nodejs
	```sh
	nvm install node
	```
	安装Node.js的同时也安装了npm，npm是一个Node.js的包管理器。
	<!--more-->	
* 安装 watchman

```sh
brew install watchman
```

* 安装flow

```sh
brew install flow
```

### 安装React-Native 

通过npm安装

```sh
npm install -g react-native-cli
```

### 初始化一个项目
```sh
react-native init FirstProject
```

### 运行项目

在开发调试阶段，需要在Mac上启动一个HTTP服务，称为`Debug Server`，默认运行在8081的端口上，APP通过`Debug Server` 来加载js。

运行命令

```sh
react-native run-android
```
app就会被运行到真机或者模拟器上，但是js不会被加载，Android 5.0以下的手机，晃动手机，弹出菜单选择Dev Setting>Debug Server host for device,然后填入Mac的IP地址。如果是5.0以上的机型，可通过adb反向代理端口，将Mac端口反向代理到测试机上。

```sh
adb reverse tcp:8081 tcp:8081
```
然后晃动手机重新加载js即可。

### 参考

* [React Native: 配置和起步](http://liaohuqiu.net/cn/posts/react-native-1/)

