---
title: "DataBinding绑定表达式使用"
date: 2017-06-11T18:14:02+08:00
draft: false
tags: ["android","databinding"]
toc: true
---


表达式语言允许您编写处理`View`分发的事件的表达式。数据绑定库自动生成将布局中的`View`与数据对象绑定所需的类。

<!--more-->

数据绑定布局文件略有不同，以`layout`根标签开头，后跟`data`元素和`view`根元素。此`view`元素是您的根在非绑定布局文件中的位置。以下代码显示了一个示例布局文件：

```xml
<?xml version="1.0" encoding="utf-8"?>
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
     android:text="@{user.firstName}"/>
   <TextView android:layout_width="wrap_content"
     android:layout_height="wrap_content"
     android:text="@{user.lastName}"/>
 </LinearLayout>
</layout>
```

`layout`标签中的`user`变量描述了可在此布局中使用的属性。

```xml
<variable name="user" type="com.example.User" />
```
布局中的表达式使用`@ {}`语法写入属性中。这里，`TextView`文本设置为用户变量的`firstName`属性：
```xml
<TextView android:layout_width="wrap_content"
     android:layout_height="wrap_content"
     android:text="@{user.firstName}" />
```

> 注意：布局表达式应保持小而简单，因为它们不能进行单元测试并且IDE支持有限。为了简化布局表达式，您可以使用自定义绑定适配器。

# 数据对象

我们现在假设您有一个普通的对象来描述`User`实体：
```java
public class User {
 public final String firstName;
 public final String lastName;
 public User(String firstName, String lastName) {
   this.firstName = firstName;
   this.lastName = lastName;
 }
}
```
这种类型的对象具有永不改变的数据。在应用程序中，通常会读取一次并且之后不会更改的数据。也可以使用遵循一组约定的对象，例如Java中的访问器方法的使用，如以下示例所示：
```java
public class User {
 private final String firstName;
 private final String lastName;
 public User(String firstName, String lastName) {
   this.firstName = firstName;
   this.lastName = lastName;
 }
 public String getFirstName() {
   return this.firstName;
 }
 public String getLastName() {
   return this.lastName;
 }
}
```

从数据绑定的角度来看，这两个类是等价的。用于`android:text`属性的表达式`@ {user.firstName}`访问前一个类中的`firstName`字段和后一类中的`getFirstName()`方法。或者，如果`firstName()`方法存在，它也会解析为该方法。

# 绑定数据

为每个布局文件生成绑定类。默认情况下，类的名称基于布局文件的名称，将其转换为`Pascal`大小写并向其添加`Binding`后缀。上面的布局文件名是`activity_main.xml`，因此相应的生成类是`ActivityMainBinding`。此类包含布局属性（例如，`User`变量）到布局视图的所有绑定，并知道如何为绑定表达式指定值。创建绑定的推荐方法是在扩展布局时执行此操作，如下例所示：
```java
@Override
protected void onCreate(Bundle savedInstanceState) {
 super.onCreate(savedInstanceState);
 ActivityMainBinding binding = DataBindingUtil.setContentView(this, R.layout.activity_main);
 User user = new User("Test", "User");
 binding.setUser(user);
}
```
在运行时，应用程序在UI中显示Test用户。或者，您可以使用LayoutInflater获取视图，如以下示例所示：
```java
ActivityMainBinding binding = ActivityMainBinding.inflate(getLayoutInflater());
```
如果在`Fragment`，`ListView`或`RecyclerView`适配器中使用数据绑定项，则可能更喜欢使用绑定类或`DataBindingUtil`类的`inflate()`方法，如以下代码示例所示：
```java
ListItemBinding binding = ListItemBinding.inflate(layoutInflater, viewGroup, false);
// or
ListItemBinding binding = DataBindingUtil.inflate(layoutInflater, R.layout.list_item, viewGroup, false);
```

# 表达式语言

## 常见特性

表达式语言看起来很像Java表达式。下面这些表达式用法是一样的：

* 数学运算符`+` `-` `*` `/` `%`
* 字符串连接符 `+`
* 逻辑运算符 `&&` `||`
* 位运算符 `&` `|` `^`
* 一元运算 `+` `-` `!` `~`
* 移位 `>>` `>>>` `<<`
* 比较运算符 `==` `>` `<` `>=` `<=`
* instanceof
* 分组()
* 字面量：字符、字符串、数字，null
* Cast
* 方法调用
* 数组访问 `[]`
* 三元运算 `?:`

```xml
android:text="@{String.valueOf(index + 1)}"
android:visibility="@{age > 13 ? View.GONE : View.VISIBLE}"
android:transitionName='@{"image_" + id}'
```

## 不支持的操作

* this
* super
* new
* 显式泛型调用

## Null合并操作

`??`- 左边的对象如果它不是`null`，选择左边的对象；或者如果它是`null`，选择右边的对象。例如下面的例子中如果`displayName`不为空则将`user.displayName`设置给`text`属性，如果为空则将`user.lastName`设置给`text`属性。

```java
android:text="@{user.displayName ?? user.lastName}"
```
上面表达式等价于下面的表达式

```java
android:text="@{user.displayName != null ? user.displayName : user.lastName}"
```

## 避免空指针

生成的数据绑定代码会自动检查空值并避免空指针异常。例如，在表达式`@{user.name}`中，如果`user`为`null`，则为`user.name`分配其默认值`null`。如果引用`user.age`，其中`age`的类型为`int`，则数据绑定使用默认值0。

## 集合

常用的集合包括`Arrays`、`List`、`SparseArray`和`Map`，这些都可以使用`[]`操作符来访问

```xml
<data>
    <import type="android.util.SparseArray"/>
    <import type="java.util.Map"/>
    <import type="java.util.List"/>
    <variable name="list" type="List&lt;String>"/>
    <variable name="sparse" type="SparseArray&lt;String>"/>
    <variable name="map" type="Map&lt;String, String>"/>
    <variable name="index" type="int"/>
    <variable name="key" type="String"/>
</data>
…
android:text="@{list[index]}"
…
android:text="@{sparse[index]}"
…
android:text="@{map[key]}"

```

> 注意：要使XML在语法上正确，您必须转义`<`字符。例如：您必须编写`List＆lt;String>`而不是`List<String>`。


您还可以使用`object.key`表示法引用`map`中的值。例如，上面示例中的`@{map[key]}`可以替换为`@{map.key}`。

## 字符串字面量


您可以使用单引号括起属性值，这允许您在表达式中使用双引号，如以下示例所示：

```xml
android:text='@{map["firstName"]}'
```
也可以使用双引号来包围属性值。这样做时，字符串文字应该用后引号`包围：

```xml
android:text="@{map[`firstName`}"
```

## 资源


您可以使用以下语法访问表达式中的资源：

```xml
android:padding="@{large? @dimen/largePadding : @dimen/smallPadding}"
```

支持字符串的格式化

```xml
<string name="info">first name is %1$s last name is %2$s age is %3$d</string>

<TextView
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text='@{@string/info(user.firstName,user.lastName,user.age)}'
    />
```

在表达式中引用一些资源与直接引用资源有所不同，需要显式判断类型


| 类型   |      正常引用     |  表达式引用 |
|----------|:-------------:|------:|
|String[] |  @array | @stringArray |
| int[]   |   @array  |   @intArray |
| TypedArray | @array |   @typedArray|
| Animator | @animator|   @animator|
| StateListAnimator | @animator |   @stateListAnimator|
| color int | @color |   @color|
| ColorStateList | @color |   @color|

# 事件处理

数据绑定允许编写处理`view`分发的事件的表达式（例如，`onClick()`方法）。事件属性名称由监听器方法的名字决定，但有一些例外。例如，`View.OnClickListener`有一个方法`onClick()`，因此该事件的属性是`android：onClick`。

点击事件有一些专门的事件处理器需要一个除`android:onClick`以外的属性以避免冲突。您可以使用以下属性来避免这些类型的冲突：

| Class                                                        | Listener setter                                              | Attribute             |
| ------------------------------------------------------------ | ------------------------------------------------------------ | --------------------- |
| [SearchView](https://developer.android.com/reference/android/widget/SearchView.html) | [setOnSearchClickListener(View.OnClickListener)](https://developer.android.com/reference/android/widget/SearchView.html#setOnSearchClickListener(android.view.View.OnClickListener)) | android:onSearchClick |
| ZoomControls                                                 | [setOnZoomInClickListener(View.OnClickListener)](https://developer.android.com/reference/android/widget/ZoomControls.html#setOnZoomInClickListener(android.view.View.OnClickListener)) | android:onZoomIn      |
| [ZoomControls](https://developer.android.com/reference/android/widget/ZoomControls.html) | [setOnZoomOutClickListener(View.OnClickListener)](https://developer.android.com/reference/android/widget/ZoomControls.html#setOnZoomOutClickListener(android.view.View.OnClickListener)) | android:onZoomOut     |

你可以使用以下机制来处理事件：

* 方法参考：在表达式中，您可以引用符合监听方法签名的方法。当表达式求值为方法引用时，`Data`绑定将方法引用和所有者对象包装在监听器中，并在目标`View`上设置该监听器。如果表达式求值为`null`，则数据绑定不会创建监听器并改为设置空监听器。

* 监听器绑定：这些是在事件发生时计算的`lambda`表达式。数据绑定总是创建一个监听器，它在`View`上设置。分发事件时，监听器将计算lambda表达式。

## 方法引用

事件可以直接绑定到处理方法上，类似于`android:onClick`的方式可以指定一个`activity`中的方法。与`View `的`onClick`属性相比，一个主要优点是表达式在编译时处理，因此如果该方法不存在或其签名不正确，则会收到编译时错误。

方法引用和监听器绑定之间的主要区别在于实际的监听器实现是在绑定数据时创建的，而不是在触发事件时创建的。如果您希望在事件发生时计算表达式，则应使用监听器绑定。

要将事件分配给其处理程序，请使用普通绑定表达式，其值为要调用的方法名称。例如，请考虑以下示例布局数据对象：

```java
public class MyHandlers {
    public void onClickFriend(View view) { ... }
}
```

绑定表达式可以将`view`的点击监听器分配给`onClickFriend()`方法，如下所示：

```java
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android">
   <data>
       <variable name="handlers" type="com.example.MyHandlers"/>
       <variable name="user" type="com.example.User"/>
   </data>
   <LinearLayout
       android:orientation="vertical"
       android:layout_width="match_parent"
       android:layout_height="match_parent">
       <TextView android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           android:text="@{user.firstName}"
           android:onClick="@{handlers::onClickFriend}"/>
   </LinearLayout>
</layout>
```

> 注意：表达式中方法的签名必须与监听器对象中方法的签名完全匹配。

监听器绑定是在事件发生时运行的绑定表达式。它们与方法引用类似，但它们允许您运行任意数据绑定表达式。此功能适用于`Gradle`版本`2.0`及更高版本的`Android Gradle`插件。

在方法引用中，方法的参数必须与事件监听器的参数匹配。在监听器绑定中，只有您的返回值必须与侦听器的预期返回值匹配（除非它期望无效）。例如，考虑以下具有`onSaveClick()`方法的`presenter`类：

```java
public class Presenter {
    public void onSaveClick(Task task){}
}
```

然后，您可以将`click`事件绑定到`onSaveClick()`方法，如下所示：

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android">
    <data>
        <variable name="task" type="com.android.example.Task" />
        <variable name="presenter" type="com.android.example.Presenter" />
    </data>
    <LinearLayout android:layout_width="match_parent" android:layout_height="match_parent">
        <Button android:layout_width="wrap_content" android:layout_height="wrap_content"
        android:onClick="@{() -> presenter.onSaveClick(task)}" />
    </LinearLayout>
</layout>
```

在表达式中使用回调时，数据绑定会自动创建必要的监听器并为事件注册它。当视图触发事件时，数据绑定会评估给定的表达式。与常规绑定表达式一样，在评估这些监听器表达式时，仍然可以获得数据绑定的null和线程安全性。

在上面的示例中，我们尚未定义传递给`onClick(View)`的`view`参数。监听器绑定为监听器参数提供了两种选择：您可以忽略方法的所有参数，也可以命名所有参数。如果您希望为参数命名，可以在表达式中使用它们。例如，上面的表达式可以写成如下：

```xml
android:onClick="@{(view) -> presenter.onSaveClick(task)}"
```

或者，如果要在表达式中使用该参数，则可以按如下方式工作：

```java
public class Presenter {
    public void onSaveClick(View view, Task task){}
}
```

```java
android:onClick="@{(theView) -> presenter.onSaveClick(theView, task)}"
```

您可以使用带有多个参数的`lambda`表达式：

```java
public class Presenter {
    public void onCompletedChanged(Task task, boolean completed){}
}
```

```xml
<CheckBox 
      android:layout_width="wrap_content" 
			android:layout_height="wrap_content"
      android:onCheckedChanged="@{(cb, isChecked) -> presenter.completeChanged(task, isChecked)}" />
```

如果您正在监听的事件返回类型不为`void`的值，则表达式也必须返回相同类型的值。例如，如果要监听长按事件，则表达式应返回布尔值。

```java
public class Presenter {
    public boolean onLongClick(View view, Task task) { }
}
```

```xml
android:onLongClick="@{(theView) -> presenter.onLongClick(theView, task)}"
```

如果由于`null`对象而无法计算表达式，则数据绑定将返回该类型的默认值。例如，`null`表示引用类型，0表示`int`，`false`表示布尔值等。

如果需要使用带谓词的表达式（例如，三元组），则可以使用`void`作为符号。

```xml
android:onClick="@{(v) -> v.isVisible() ? doSomething() : void}"
```

## 避免复杂的监听器

监听器表达式非常强大，可以使您的代码非常容易阅读。另一方面，包含复杂表达式的监听器使您的布局难以阅读和维护。这些表达式应该像将可用数据从UI传递到回调方法一样简单。您应该在从监听器表达式调用的回调方法中实现任何业务逻辑。

# 导入、变量和引入

数据绑定库提供诸如导入，变量和引入之类的功能。导入使布局文件中的类很容易引用。变量允许您描述可用于绑定表达式的属性。包括让您在整个应用中重复使用复杂的布局。

## 导入



数据绑定的布局文件允许利用`import`元素像Java一样导入其他数据类型。例如下面代码就导入一个`View`。

```xml
<data>
    <import type="android.view.View"/>
</data>
```



现在，View对象可以在绑定表达式中使用。

```xml
<TextView
   android:text="@{user.lastName}"
   android:layout_width="wrap_content"
   android:layout_height="wrap_content"
   android:visibility="@{user.isAdult ? View.VISIBLE : View.GONE}"/>
```

## 类型别名

当存在类名冲突时，可以将其中一个类重命名为别名。以下示例将`com.example.real.estate`包中的`View`类重命名为`Vista`：

```xml
<import type="android.view.View"/>
<import type="com.example.real.estate.View"
        alias="Vista"/>
```

您可以使用`Vista`来引用`com.example.real.estate.View`，`View`可以用来引用布局文件中的`android.view.View`。

## 引入其他类

导入的类型可以用作变量和表达式中的类型引用。以下示例显示用作变量类型的`User`和`List`：

```java
<data>
    <import type="com.example.User"/>
    <import type="java.util.List"/>
    <variable name="user" type="User"/>
    <variable name="userList" type="List&lt;User>"/>
</data>
```

> 警告：Android Studio尚未处理导入，因此导入变量的自动完成功能可能无法在IDE中运行。您的应用程序仍在编译，您可以通过在变量定义中使用完全限定名称来解决IDE问题。

您还可以使用导入的类型来转换表达式的一部分。以下示例将连接属性强制转换为User类型：

```xml
<TextView
   android:text="@{((User)(user.connection)).lastName}"
   android:layout_width="wrap_content"
   android:layout_height="wrap_content"/>
```

在表达式中引用静态字段和方法时，也可以使用导入的类型。以下代码导入`MyStringUtils`类并引用其`capitalize`方法：

```xml
<data>
    <import type="com.example.MyStringUtils"/>
    <variable name="user" type="com.example.User"/>
</data>
…
<TextView
   android:text="@{MyStringUtils.capitalize(user.lastName)}"
   android:layout_width="wrap_content"
   android:layout_height="wrap_content"/>
```

就像在托管代码中一样，`java.lang.*`会自动导入。

## 变量

您可以在数据元素中使用多个变量元素。每个变量元素描述可以在布局上设置的属性，以在布局文件中的绑定表达式中使用。以下示例声明`user`，`drawable`和`note`变量：

```xml
<data>
    <import type="android.graphics.drawable.Drawable"/>
    <variable name="user" type="com.example.User"/>
    <variable name="image" type="Drawable"/>
    <variable name="note" type="String"/>
</data>
```

在编译时检查变量类型，因此如果变量实现`Observable`或是可观察集合，则应该在类型中响应。如果变量是未实现`Observable`接口的基类或接口，则不会观察变量。

当存在用于各种配置的不同布局文件（例如，横向或纵向）时，变量被合并。这些布局文件之间不得存在冲突的变量定义。

对于每个所描述的变量，生成的绑定类都具有该变量的`setter`和`getter`。`setter`方法调用之前，变量有一个默认值。引用类型的默认值是null，int默认值是0，boolean默认值是false等。

根据需要，生成一个名为`context`的特殊变量，用于绑定表达式。 `contex`t的值是来自根`View`的`getContext()`方法的`Context`对象。使用该名称的显式变量声明覆盖上下文变量。

## 引入

通过使用app命名空间和属性中的变量名，变量可以从包含的布局传递到包含的布局绑定中。以下示例显示了`name.xml`和`contact.xml`布局文件中包含的用户变量：

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:bind="http://schemas.android.com/apk/res-auto">
   <data>
       <variable name="user" type="com.example.User"/>
   </data>
   <LinearLayout
       android:orientation="vertical"
       android:layout_width="match_parent"
       android:layout_height="match_parent">
       <include layout="@layout/name"
           bind:user="@{user}"/>
       <include layout="@layout/contact"
           bind:user="@{user}"/>
   </LinearLayout>
</layout>
```

数据绑定不支持`include`作为`merge`元素的直接子元素。例如，不支持以下布局：

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:bind="http://schemas.android.com/apk/res-auto">
   <data>
       <variable name="user" type="com.example.User"/>
   </data>
   <merge><!-- Doesn't work -->
       <include layout="@layout/name"
           bind:user="@{user}"/>
       <include layout="@layout/contact"
           bind:user="@{user}"/>
   </merge>
</layout>
```

