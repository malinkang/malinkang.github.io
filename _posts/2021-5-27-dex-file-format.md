---
title: "Dex文件格式分析"
date: 2019-09-27T10:04:55+08:00
draft: false
---

在刚学习Java的时候我们都会写一个`HelloWorld`的示例。

```java
public class HelloWorld{
    public static void main(String[] args){
        System.out.println("Hello,world!");
    }
}
```

然后通过`javac`命令编译成字节码，然后调用`java`命令机执行字节码。我们如何像`java`一样在命令行里直接在命令行输出`Hello,world!`呢。具体要执行如下操作：

<!--more-->

将`.class`文件转换成`.dex`文件

```shell
dx --dex --output=HelloWorld.dex HelloWorld.class
```

将`dex`文件推送到`sd`卡上。

```shell
adb push HelloWorld.dex /sdcard/
```

执行字节码

```shell
adb shell
dalvikvm -cp /sdcard/HelloWorld.dex HelloWorld
```



## Dex文件格式概貌

Dex文件结构如下图所示。

![img](/images/Center.png)

Dex文件对应的对象是[DexFile](https://android.googlesource.com/platform/dalvik/+/android-4.4.2_r2/libdex/DexFile.h#500)，定义如下。

```c
struct DexFile {
    /* directly-mapped "opt" header */
    const DexOptHeader* pOptHeader;
    /* pointers to directly-mapped structs and arrays in base DEX */
    const DexHeader*    pHeader;
    const DexStringId*  pStringIds;
    const DexTypeId*    pTypeIds;
    const DexFieldId*   pFieldIds;
    const DexMethodId*  pMethodIds;
    const DexProtoId*   pProtoIds;
    const DexClassDef*  pClassDefs;
    const DexLink*      pLinkData;
    /*
     * These are mapped out of the "auxillary" section, and may not be
     * included in the file.
     */
    const DexClassLookup* pClassLookup;
    const void*         pRegisterMapPool;       // RegisterMapClassPool
    /* points to start of DEX file data */
    const u1*           baseAddr;
    /* track memory overhead for auxillary structures */
    int                 overhead;
    /* additional app-specific data structures associated with the DEX */
    //void*               auxData;
};
```

首先从宏观上来说dex的文件结果很简单，实际上是由多个不同结构的数据体以首尾相接的方式拼接而成。各个成员的解释如下：

| 数据名称   | 解释                                                         |
| ---------- | ------------------------------------------------------------ |
| header     | dex文件头部，记录整个dex文件的相关属性                       |
| string_ids | 字符串数据索引，记录了每个字符串在数据区的偏移量             |
| type_ids   | 类似数据索引，记录了每个类型的字符串索引                     |
| proto_ids  |                                                              |
| field_ids  | 存储成员变量信息，包括变量名、类型等                         |
| method_ids | 存储成员函数信息包括函数名、参数和返回值类型等               |
| class_defs | 存储类的信息                                                 |
| data       | Dex文件重要的数据内容都存在data区域里。一些数据结构会通过如xx_off这样的成员变量指向文件的某个位置，从该位置开始，存储了对应数据结构的内容，而xx_off的位置一般落在data区域里 |
| link_data  | 理论上是预留区域，没有特别的作用。                           |



## 头部信息



查看刚才生成的`dex`的十六进制

![image-20190927104628357](/images/image-20190927104628357.png)

`Header`的大小固定为`0x70`,偏移地址从`0x00`到`0x70`。

`Header`包含的字段和各个字段的长度如下表所示：

| address | Name         | size/byte | Value |
| ------- | ------------ | --------- | ----- |
| 0       | Magic Number | 8         |       |
|         |              |           |       |
|         |              |           |       |
|         |              |           |       |
|         |              |           |       |
|         |              |           |       |
|         |              |           |       |
|         |              |           |       |
|         |              |           |       |
|         |              |           |       |
|         |              |           |       |
|         |              |           |       |
|         |              |           |       |
|         |              |           |       |
|         |              |           |       |
|         |              |           |       |
|         |              |           |       |
|         |              |           |       |
|         |              |           |       |
|         |              |           |       |
|         |              |           |       |
|         |              |           |       |
|         |              |           |       |
|         |              |           |       |



## string_ids数据结构

## type_ids数据结构

## proto_ids数据结构

## field_ids数据结构

## method_ids数据结构

## class_defs数据结构



## 参考

* [Android逆向之旅---解析编译之后的Dex文件格式](https://blog.csdn.net/jiangwei0910410003/article/details/50668549)
* [Dex文件格式详解](https://www.jianshu.com/p/f7f0a712ddfe)

* [Android应用安全防护和逆向分析第7章](https://book.douban.com/subject/27617785/)
* [深入理解Android第3章](https://book.douban.com/subject/33390277/)