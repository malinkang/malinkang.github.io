---
title: Android自定义属性
date: 2017-03-21 17:37:26
---

创建自定义属性需要以下步骤：

* 创建一个自定义View。
* 创建values/attrs.xml文件，并定义属性。
* 在View中获取属性值并使用。

<!--more--> 

### 自定义属性

```xml
<resources>
   <declare-styleable name="PieChart">
       <attr name="showText" format="boolean" />
       <attr name="labelPosition" format="enum">
           <enum name="left" value="0"/>
           <enum name="right" value="1"/>
       </attr>
   </declare-styleable>
</resources>
```
declare-styleable： 表示一个属性组。它的name必须和你自定义view的名字相同。

attr：表示单独的一个属性。format代表属性的格式，格式包括很多种。

```xml
<attr name="background" format="reference" /> <!--引用类型，比如drawable-->
<attr name="textColor" format="color" /> <!--颜色值-->
<attr name="focusable" format="boolean" /> <!--布尔值-->
<attr name="layout_width" format="dimension" /> <!--尺寸-->
<attr name="fromAlpha" format="float" /> <!--浮点值-->
<attr name="frameDuration" format="integer" /> <!--整型值-->
<attr name="apiKey" format="string" /> <!--字符串-->
<attr name="pivotX" format="fraction" /> <!--百分数-->
<attr name="orientation">  <!-枚举-->
    <enum name="horizontal" value="0" />
    <enum name="vertical" value="1" />
</attr>
<attr name="windowSoftInputMode"> <!-flag-->
    <flag name="stateUnspecified" value="0" />
    <flag name="stateUnchanged" value="1" />
    <flag name="stateHidden" value="2" />
    <flag name="stateAlwaysHidden" value="3" />
    <flag name="stateVisible" value="4" />
    <flag name="stateAlwaysVisible" value="5" />
    <flag name="adjustUnspecified" value="0x00" />
    <flag name="adjustResize" value="0x10" />
    <flag name="adjustPan" value="0x20" />
    <flag name="adjustNothing" value="0x30" />
</attr>
```
### 获取属性值


1. 通过context的obtainStyledAttributes方法获取TypedArray。


```java

TypedArray a = context.obtainStyledAttributes(attrs, R.styleable.CircleImageView, defStyle, 0);

```

2. TypedArray提供如下方法获取属性值


```java

public int getDimensionPixelSize(@StyleableRes int index, int defValue)//获取尺寸
public int getColor(@StyleableRes int index, @ColorInt int defValue)//获取颜色值
public boolean getBoolean(@StyleableRes int index, boolean defValue)//获取布尔值
public float getFloat(@StyleableRes int index, float defValue)//获取float值

``` 

### 参考

* [深入理解Android 自定义attr Style styleable以及其应用](http://www.jianshu.com/p/61b79e7f88fc)
