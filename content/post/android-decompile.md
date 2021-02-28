---
title: android反编译
date: 2015-12-07 15:48:40 +0800
draft: false
---


### Apktool 

Apktool 能够从APK中解析出资源文件，xml文件以及生成smail文件。

<!--more--> 

mac系统下Apktool安装步骤：

1. 打开终端进入 `/usr/local/bin`目录下执行命令`touch apktool`创建`apktool`文件。
2. copy[脚本](https://raw.githubusercontent.com/iBotPeaches/Apktool/master/scripts/osx/apktool)到`apktool`。
3. [下载](https://bitbucket.org/iBotPeaches/apktool/downloads)最新的jar，修改名字为`apktool.jar`并放到`/usr/local/bin`目录下。
4. 执行命令` chmod +x /usr/local/bin/apktool`

apktool使用：

切换到apk所在目录下执行`apktool d test.apk`命令对apk进行反编译。


### dex2jar

dex2jar可以将APK中的dex文件转换为jar文件。

mac系统下dex2jar安装步骤：

1. [下载](http://sourceforge.net/projects/dex2jar/)并解压。
2. 将解压后的文件夹copy到`/Applications`目录下
3. 修改文件为可执行文件`  chmod +x /Applications/dex2jar-2.0/d2j-dex2jar.sh /Applications/dex2jar-2.0/d2j_invoke.sh`

使用：

切换到apk所在目录下执行`sh /Applications/dex2jar-2.0/d2j-dex2jar.sh test.apk`命令,会在相同目录下生成一个jar文件。

### JD-GUI

JD-GUI文件可以查看jar包中的.class文件。可以利用该工具查看通过dex2jar工具获得的jar文件。

下载地址：<http://jd.benow.ca/>


### 更多阅读

* [ Android安全攻防战，反编译与混淆技术完全解析（上）](http://blog.csdn.net/guolin_blog/article/details/49738023)
* [jadx:更好的Android反编译工具](https://liuzhichao.com/2016/jadx-decompiler.html)
* [android-classyshark](https://github.com/google/android-classyshark)






