---
title: 《深入理解JVM》第10章编译期优化
date: 2016-09-03 09:09:28
tags: [深入理解JVM]
cover: https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/blog/images/cover/你的名字30.png
---
# 第10章 Java内存区域与内存溢出

## 10.1 概述

## 10.2 Javac编译器

### 10.2.1 Javac的源码与调试

从Sun Javac的代码来看，编译过程大致可以分为3个过程，分别是：

* 解析与填充符号表过程。
* 插入式注解处理器的注解处理过程。 ￼
* 分析与字节码生成过程。

这3个步骤之间的关系与交互顺序如图10-4所示。

  ![](https://github.com/malinkang/JavaNote/tree/05f7c6abd740c7af6029fb75682bad60b7d55521/images/understanding-the-jvm/10-4.png)

Javac编译动作的入口是com.sun.tools.javac.main.JavaCompiler类，上述3个过程的代码逻辑集中在这个类的compile\(\)和compile2\(\)方法中，其中主体代码如图10-5所示，整个编译最关键的处理就由图中标注的8个方法来完成，下面我们具体看一下这8个方法实现了什么功能。

![&#x56FE;10-5Javac&#x7F16;&#x8BD1;&#x8FC7;&#x7A0B;&#x7684;&#x4E3B;&#x4F53;&#x4EE3;&#x7801;](https://github.com/malinkang/JavaNote/tree/05f7c6abd740c7af6029fb75682bad60b7d55521/images/understanding-the-jvm/10-5.png)

### 10.2.2 解析与填充符号表

解析步骤由图10-5中的parseFiles\(\)方法（图10-5中的过程1.1）完成，解析步骤包括了经典程序编译原理中的词法分析和语法分析两个过程。

#### 1.词法、语法分析

词法分析是将源代码的字符流转变为标记（Token）集合，单个字符是程序编写过程的最小元素，而标记则是编译过程的最小元素，关键字、变量名、字面量、运算符都可以成为标记，如“int a=b+2”这句代码包含了6个标记，分别是int、a、=、b、+、2，虽然关键字int由3个字符构成，但是它只是一个Token，不可再拆分。在Javac的源码中，词法分析过程由com.sun.tools.javac.parser.Scanner类来实现。

语法分析是根据Token序列构造抽象语法树的过程，抽象语法树（Abstract Syntax Tree, AST）是一种用来描述程序代码语法结构的树形表示方式，语法树的每一个节点都代表着程序代码中的一个语法结构（Construct），例如包、类型、修饰符、运算符、接口、返回值甚至代码注释等都可以是一个语法结构。

### 10.2.3 注解处理器

### 10.2.4 语义分析与字节码生成

#### 1.标注检查

#### 2.数据及控制流分析

#### 3.解语法糖

语法糖（Syntactic Sugar），也称糖衣语法，是由英国计算机科学家彼得·约翰·兰达（Peter J.Landin）发明的一个术语，指在计算机语言中添加的某种语法，这种语法对语言的功能并没有影响，但是更方便程序员使用。通常来说，使用语法糖能够增加程序的可读性，从而减少程序代码出错的机会。

#### 4.字节码生成

## 10.3 Java语法糖的味道

几乎各种语言或多或少都提供过一些[语法糖](https://zh.wikipedia.org/wiki/%E8%AF%AD%E6%B3%95%E7%B3%96)来方便程序员的代码开发，这些语法糖虽然不会提供实质性的功能改进，但是它们或能提高效率，或能提升语法的严谨性，或能减少编码出错的机会。不过也有一种观点认为语法糖并不一定都是有益的，大量添加和使用“含糖”的语法，容易让程序员产生依赖，无法看清语法糖的糖衣背后，程序代码的真实面目。

总而言之，语法糖可以看做是编译器实现的一些“小把戏”，这些“小把戏”可能会使得效率“大提升”，但我们也应该去了解这些“小把戏”背后的真实世界，那样才能利用好它们，而不是被它们所迷惑。

### 10.3.1 泛型与类型擦除

### 10.3.2 自动装箱、拆箱与遍历循环

```java
List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);
int sum = 0;
for (int i:list){
    sum +=i;
}
System.out.println(sum);
```
编译生成
```java
List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);
int sum = 0;
int i;
//本质调用的迭代器
for(Iterator var3 = list.iterator(); var3.hasNext(); sum += i) {
    i = (Integer)var3.next();
}
System.out.println(sum);
```

### 10.3.3 条件编译

## 10.4 实战：插入式注解处理器

