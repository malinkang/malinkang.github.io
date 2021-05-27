---
title: "生成绑定类"
date: 2017-06-19T20:01:58+08:00
draft: false
tags: ["android","databinding"]
toc: true
---

数据绑定库生成用于访问布局的变量和视图的绑定类。生成的绑定类将布局变量与布局中的视图链接起来。绑定类的名称和包可以自定义。所有生成的绑定类都继承自`ViewDataBinding`类。

<!--more-->

为每个布局文件生成绑定类。默认情况下，类的名称基于布局文件的名称，将其转换为`Pascal`大小写并向其添加`Binding`后缀。布局文件名是`activity_main.xml`，因此相应的生成类是`ActivityMainBinding`。此类包含布局属性（例如，用户变量）到布局视图的所有绑定，并知道如何为绑定表达式指定值。

## 创建一个绑定对象
在对布局进行`inflate`之后，应该很快创建绑定对象，以确保在使用布局中的表达式绑定到视图之前不会修改视图层次结构。将对象绑定到布局的最常用方法是使用绑定类上的静态方法。您可以通过使用绑定类的`inflate()`方法来扩展视图层次结构并将对象绑定到该层次结构，如以下示例所示：

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
  super.onCreate(savedInstanceState);
  MyLayoutBinding binding = MyLayoutBinding.inflate(getLayoutInflater());
}
```

还有另外一个版本的`inflate()`方法，它除了`LayoutInflater`对象之外，还接受一个`ViewGroup`对象，如下例所示：

```java
MyLayoutBinding binding = MyLayoutBinding.inflate(getLayoutInflater(), viewGroup, false);
```

如果使用不同的机制对布局进行了`inflate`，则可以单独绑定，如下所示：
```java
MyLayoutBinding binding = MyLayoutBinding.bind(viewRoot);
```

有时不能事先知道绑定类型。在这种情况下，可以使用`DataBindingUtil`类创建绑定，如以下代码段所示：
```java
View viewRoot = LayoutInflater.from(this).inflate(layoutId, parent, attachToParent);
ViewDataBinding binding = DataBindingUtil.bind(viewRoot);
```

如果在`Fragment`，`ListView`或`RecyclerView`适配器中使用数据绑定项，则可能更喜欢使用绑定类或`DataBindingUtil`类的`inflate()`方法，如以下代码示例所示：

```java
ListItemBinding binding = ListItemBinding.inflate(layoutInflater, viewGroup, false);
// or
ListItemBinding binding = DataBindingUtil.inflate(layoutInflater, R.layout.list_item, viewGroup, false);
```

## 带Id的View

数据绑定库在绑定类中为每个在布局中具有ID的视图创建不可变字段。例如，数据绑定库从以下布局创建类型为`TextView`的`firstName`和`lastName`字段。

```xml
<layout xmlns:android="http://schemas.android.com/apk/res/android">
 <data>
   <variable name="user" type="com.example.User"/>
 </data>
 <LinearLayout
   android:orientation="vertical"
   android:layout_width="match_parent"
   android:layout_height="match_parent">
   <TextView android:layout_width="wrap_content"
     android:layout_height="wrap_content"
     android:text="@{user.firstName}"
     android:id="@+id/firstName"/>
   <TextView android:layout_width="wrap_content"
     android:layout_height="wrap_content"
     android:text="@{user.lastName}"
     android:id="@+id/lastName"/>
 </LinearLayout>
</layout>
```

该库在一次传递中从视图层次结构中提取包括ID的视图。此机制比为布局中的每个视图调用`findViewById()`方法更快。

ID没有必要，因为它们没有数据绑定，但仍有一些情况需要从代码访问视图。

## 变量

数据绑定库为布局中声明的每个变量生成访问器方法。例如，以下布局在绑定类中为变量`user`，`image`和注`image`的生成`setter`和`getter`方法：

```xml
<data>
 <import type="android.graphics.drawable.Drawable"/>
 <variable name="user" type="com.example.User"/>
 <variable name="image" type="Drawable"/>
 <variable name="note" type="String"/>
</data>
```

## ViewStubs

与普通`View`不同，`ViewStub`对象从一个不可见的视图开始。当它们被显示或被明确告知要`inflate`时，它们会通过`inflate`另一个布局来替换自己的布局。

由于`ViewStub`从视图层次结构中消失，因此绑定对象中的视图也必须消失以允许垃圾回收声明。因为视图是最终的，所以`ViewStubProxy`对象取代了生成的绑定类中的`ViewStub`，使您可以在`ViewStub`存在时访问它，并在`ViewStub` `inflate`时访问`inflate`的视图层次结构。

在`inflate`另一个布局时，必须为新布局建立绑定。因此，`ViewStubProxy`必须监听`ViewStub OnInflateListener`并在需要时建立绑定。由于在给定时间只能存在一个侦听器，因此`ViewStubProxy`允许您设置`OnInflateListener`，它在建立绑定后调用它。

## 立即绑定

当变量或可观察对象发生更改时，绑定计划在下一帧之前更改。但是，有时必须立即执行绑定。要强制执行，请使用`executePendingBindings()`方法。

## 高级绑定

### 动态变量

有时，特定的绑定类是未知的。例如，针对任意布局操作的`RecyclerView.Adapter`不知道特定的绑定类。它仍然必须在调用`onBindViewHolde()`方法期间分配绑定值。

在下面的例子中，`RecyclerView`绑定的所有布局都有一个`item`变量。 `BindingHolder`对象有一个`getBinding()`方法返回`ViewDataBinding`基类。

```java
public void onBindViewHolder(BindingHolder holder, int position) {
  final T item = items.get(position);
  holder.getBinding().setVariable(BR.item, item);
  holder.getBinding().executePendingBindings();
}
```

> 注意：数据绑定库在模块包中生成一个名为BR的类，其中包含用于数据绑定的资源的ID。在上面的示例中，库自动生成BR.item变量。

## 后台线程

您可以在后台线程中更改数据模型，只要它不是集合即可。数据绑定在评估期间本地化每个变量/字段以避免任何并发问题。

## 自定义绑定类名字

默认情况下，将根据布局文件的名称生成绑定类，以大写字母开头，删除下划线（_），大写以下字母，并为单词`Binding`添加后缀。该类放在模块包下的数据绑定包中。例如，布局文件`contact_item.xml`生成`ContactItemBinding`类。如果模块包是`com.example.my.app`，则绑定类放在`com.example.my.app.databinding`包中。

通过调整数据元素的class属性，可以重命名绑定类或将绑定类放在不同的包中。例如，以下布局在当前模块的数据绑定包中生成`ContactItem`绑定类：

```xml
<data class="ContactItem">
  …
</data>
```

您可以通过在类名前加一个句点来为不同的包生成绑定类。以下示例在模块包中生成绑定类：

```xml
<data class=".ContactItem">
  …
</data>
```
您还可以使用要在其中生成绑定类的完整包名称。以下示例在`com.example`包中创建`ContactItem`绑定类：

```xml
<data class="com.example.ContactItem">
  …
</data>
```