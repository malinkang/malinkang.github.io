---
title: 《Java编程思想》第2章一切都是对象
date: 2013-03-19 16:26:34
tags: [Thinking in Java]
---

## 2.2 必须由你创建所有对象

### 2.2.1 存储到什么地方

### 2.2.2 特例：基本类型

在程序设计中经常用到一系列类型，它们需要特殊对待。可以把它们想象成“基本”类型。之所以特殊对待，是因为new将对象存储在“堆”里，故用new创建一个对象，特别是小的、简单的变量，往往不是很有效。因此，对于这些类型，Java采取与C和C++相同的方法。也就是说，不用new来创建变量，而是创建一个并非是引用的“自动”变量。这个变量直接存储“值”，并置于栈中，因此更加高效。

Java要确定每种基本类型所占存储空间的大小。它们的大小并不像其他大多数语言那样随机器硬件架构的变化而变化。这种所占存储空间大小的不变性是Java程序比用其他大多数语言编写的程序更具有可移植性的原因之一。

| 基本类型 | 大小 | 最小值 | 最大值  | 包装器类型 |
| -------- | ---- | ------ | ------- | ---------- |
| boolean  |      |        |         | Boolean    |
| char     | 16位 |        |         | Character  |
| byte     | 8位  | -128   | +127    | Byte       |
| short    | 16位 | -2^15  | +2^15-1 | Short      |
| int      | 32位 |        |         | Integer    |
| long     | 64位 |        |         | Long       |
| float    | 32位 |        |         | Float      |
| double   | 64位 |        |         | Double     |
| void     |      |        |         | Void       |

所有数值类型都有正负号，所以不要去寻找无符号的数值类型。

boolean类型所占存储空间的大小没有明确指出，仅定义为能够取字面值true或false。

基本类型具有的包装器类，使得可以在堆中创建一个非基本对象，用来表示对应的基本类型。例如：

```java
int i = 100;
Integer i2 = Integer.valueOf(i);
int i3 = i2.intValue();
```

Java SE5的自动包装功能将自动地将基本类型转换为包装器类型：

```java
int i = 100;
Integer i2 = i;
int i3 = i2;
```

反编译可以看到装箱调用的`Integer.valueOf()`拆箱调用`Integer.intValue()`

![image-20201229203052098](https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/blog/images/leetcode/integer.png)



### Integer

```java
Integer i1 = 127;
Integer i2 = 127;
System.out.println(i1==i2); //true
Integer i3 = 128;
Integer i4 = 128;
System.out.println(i3==i4);//false
Integer i5 = new Integer(127); 
Integer i6 = new Integer(127);
System.out.println(i5 == i6); //false
```

编译器把`Integer x = 127;`自动变为`Integer x = Integer.valueOf(127);`，为了节省内存，`Integer.valueOf()`对于较小的数，始终返回相同的实例，因此，`==`比较“恰好”为`true`。

```java
public static Integer valueOf(int i) {
   // >= -128 <= 127 直接返回缓存
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

```java
private static class IntegerCache {
    static final int low = -128;
    static final int high;
    static final Integer cache[];

    static {
        // high value may be configured by property
        int h = 127;
        String integerCacheHighPropValue =
            sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
        if (integerCacheHighPropValue != null) {
            try {
                int i = parseInt(integerCacheHighPropValue);
                i = Math.max(i, 127);
                // Maximum array size is Integer.MAX_VALUE
                h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
            } catch( NumberFormatException nfe) {
                // If the property cannot be parsed into an int, ignore it.
            }
        }
        high = h;
        //会创建长度为256
        cache = new Integer[(high - low) + 1];
        int j = low;
        for(int k = 0; k < cache.length; k++)
            cache[k] = new Integer(j++);

        // range [-128, 127] must be interned (JLS7 5.1.7)
        assert IntegerCache.high >= 127;
    }

    private IntegerCache() {}
}
```

Java提供了两个用于高精度计算的类：BigInteger和BIgDecimal。虽然它们大体上属于“包装类”的范畴，但二者都没有对应的基本类型。

不过，这两个类包含的方法，提供的操作与对基本类型所能执行的操作相似。也就是说，能作用于int或float的操作，也同样能作用于与BigInteger或BigDecimal。只不过必须以方法调用方式取代运算符方式来实现。由于这么做复杂了许多，所以运算速度会比较慢。在这里，我们以速度换取了精度。

`BigInteger`支持任意精度的整数。也就是说，在运算中，可以准确地表示任何大小的整数值，而不会丢失任何信息。

`BigDecimal`支持任何精度的定位数，例如，可以用它进行精确的货币计算。



## 扩展阅读

* [Java中的自动装箱与拆箱](https://droidyue.com/blog/2015/04/07/autoboxing-and-autounboxing-in-java/)
* [深入剖析Java中的装箱和拆箱](https://www.cnblogs.com/dolphin0520/p/3780005.html)





