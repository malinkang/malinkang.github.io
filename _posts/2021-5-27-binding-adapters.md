---
title: "绑定适配器使用"
date: 2017-06-13T18:14:02+08:00
draft: false
tags: ["android","databinding"]
toc: true
---




绑定适配器负责对设置值进行适当的框架调用。一个例子是设置一个属性值，如调用`setText()`方法。另一个例子是设置一个事件监听器，如调用`setOnClickListener()`方法。

<!--more-->

数据绑定库允许您指定调用的方法来设置值，提供自己的绑定逻辑，并使用适配器指定返回对象的类型。

# 设置属性值
每当绑定值发生更改时，生成的绑定类必须使用绑定表达式在视图上调用setter方法。您可以允许数据绑定库自动确定方法，显式声明方法或提供自定义逻辑来选择方法。

## 自动方法选择
对于名为`example`的属性，库自动尝试查找接受兼容类型作为参数的方法`setExample(arg)`。不考虑属性的命名空间，搜索方法时仅使用属性名称和类型。

例如，给定`android:text="@{user.name}"`表达式，库会查找接受`user.getName()`返回的类型的`setText(arg)`方法。如果`user.getName()`的返回类型是String，则库会查找接受String参数的`setText()`方法。如果表达式返回`int`，则库将搜索接受`int`参数的`setText()`方法。表达式必须返回正确的类型，如有必要，可以转换返回值。

即使没有给定名称的属性，数据绑定仍然有效。然后，您可以使用数据绑定为任何setter创建属性。例如，支持类`DrawerLayout`没有任何属性，但有很多setter。以下布局自动使用`setScrimColor(int)`和`setDrawerListener(DrawerListener)`方法作为`app:scrimColor`和`app:drawerListener`属性的setter：

```xml
<android.support.v4.widget.DrawerLayout
  android:layout_width="wrap_content"
  android:layout_height="wrap_content"
  app:scrimColor="@{@color/scrim}"
  app:drawerListener="@{fragment.drawerListener}">
```

## 指定自定义方法名称
某些属性具有名称不匹配的setter。在这些情况下，可以使用BindingMethods注释将属性与setter相关联。注释与类一起使用，可以包含多个BindingMethod注释，每个注释方法一个注释。绑定方法是可以添加到应用程序中任何类的注释。在以下示例中，`android:tint`属性与`setImageTintList(ColorStateList)`方法关联，而不是与`setTint()`方法关联：

```java
@BindingMethods({
   @BindingMethod(type = "android.widget.ImageView",
           attribute = "android:tint",
           method = "setImageTintList"),
})
```

大多数情况下，您不需要在Android框架类中重命名setter。已使用名称约定实现的属性可自动查找匹配方法。

## 提供自定义逻辑
某些属性需要自定义绑定逻辑。例如，`android:paddingLeft`属性没有关联的setter。相反，提供了`setPadding(left, top, right, bottom)`方法。使用BindingAdapter注释的静态绑定适配器方法允许您自定义如何调用属性的setter。

Android框架类的属性已经创建了BindingAdapter注释。例如，以下示例显示了`paddingLeft`属性的绑定适配器：

```java
@BindingAdapter("android:paddingLeft")
public static void setPaddingLeft(View view, int padding) {
 view.setPadding(padding,
         view.getPaddingTop(),
         view.getPaddingRight(),
         view.getPaddingBottom());
}
```

参数类型很重要。第一个参数确定与属性关联的`view`的类型。第二个参数确定给定属性的绑定表达式中接受的类型。

绑定适配器可用于其他类型的自定义。例如，可以从工作线程调用自定义加载程序来加载图像。

当发生冲突时，您定义的绑定适配器将覆盖Android框架提供的默认适配器。

您还可以使用接收多个属性的适配器，如以下示例所示：

```java
@BindingAdapter({"imageUrl", "error"})
public static void loadImage(ImageView view, String url, Drawable error) {
 Picasso.get().load(url).error(error).into(view);
}
```

您可以在布局中使用适配器，如以下示例所示。请注意，`@drawable/venueError`指的是您应用中的资源。使用`@ {}`在资源周围使其成为有效的绑定表达式。

```xml
<ImageView 
app:imageUrl="@{venue.imageUrl}" 
app:error="@{@drawable/venueError}" />
```

> 注意：数据绑定库会忽略自定义命名空间以进行匹配。

如果`imageUrl`和`error`都用于`ImageView`对象并且`imageUrl`是字符串且`error`是`Drawable`，则调用适配器。如果希望在设置任何属性时调用适配器，则可以将适配器的可选requireAll标志设置为false，如以下示例所示：

```java
@BindingAdapter(value={"imageUrl", "placeholder"}, requireAll=false)
public static void setImageUrl(ImageView imageView, String url, Drawable placeHolder) {
 if (url == null) {
  imageView.setImageDrawable(placeholder);
 } else {
  MyImageLoader.loadInto(imageView, url, placeholder);
 }
}
```

> 注意：发生冲突时，绑定适配器会覆盖默认数据绑定适配器。

绑定适配器方法可以选择在其处理程序中使用旧值。采用旧值和新值的方法应首先声明属性的所有旧值，然后是新值，如下例所示：
```java
@BindingAdapter("android:paddingLeft")
public static void setPaddingLeft(View view, int oldPadding, int newPadding) {
 if (oldPadding != newPadding) {
   view.setPadding(newPadding,
           view.getPaddingTop(),
           view.getPaddingRight(),
           view.getPaddingBottom());
 }
}
```

事件处理程序只能与带有一个抽象方法的接口或抽象类一起使用，如以下示例所示：

```java
@BindingAdapter("android:onLayoutChange")
public static void setOnLayoutChangeListener(View view, View.OnLayoutChangeListener oldValue,
   View.OnLayoutChangeListener newValue) {
 if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.HONEYCOMB) {
  if (oldValue != null) {
   view.removeOnLayoutChangeListener(oldValue);
  }
  if (newValue != null) {
   view.addOnLayoutChangeListener(newValue);
  }
 }
}
```

在布局中使用此事件处理程序，如下所示：

```xml
<View android:onLayoutChange="@{() -> handler.layoutChanged()}"/>
```

当监听器具有多个方法时，必须将其拆分为多个监听器。例如，`View.OnAttachStateChangeListener`有两个方法：`onViewAttachedToWindow(View)`和`onViewDetachedFromWindow(View)`。该库提供了两个接口来区分它们的属性和处理程序：

```java
@TargetApi(VERSION_CODES.HONEYCOMB_MR1)
public interface OnViewDetachedFromWindow {
 void onViewDetachedFromWindow(View v);
}

@TargetApi(VERSION_CODES.HONEYCOMB_MR1)
public interface OnViewAttachedToWindow {
 void onViewAttachedToWindow(View v);
}
```

因为更改一个监听器也会影响另一个监听器，所以需要一个适用于任一属性或适用于两者的适配器。您可以在注释中将requireAll设置为false，以指定不是必须为每个属性分配绑定表达式，如以下示例所示：
```java
@BindingAdapter({"android:onViewDetachedFromWindow", "android:onViewAttachedToWindow"}, requireAll=false)
public static void setListener(View view, OnViewDetachedFromWindow detach, OnViewAttachedToWindow attach) {
  if (VERSION.SDK_INT >= VERSION_CODES.HONEYCOMB_MR1) {
    OnAttachStateChangeListener newListener;
    if (detach == null && attach == null) {
      newListener = null;
    } else {
      newListener = new OnAttachStateChangeListener() {
        @Override
        public void onViewAttachedToWindow(View v) {
          if (attach != null) {
            attach.onViewAttachedToWindow(v);
          }
        }
        @Override
        public void onViewDetachedFromWindow(View v) {
          if (detach != null) {
            detach.onViewDetachedFromWindow(v);
          }
        }
      };
    }

    OnAttachStateChangeListener oldListener = ListenerUtil.trackListener(view, newListener,
        R.id.onAttachStateChangeListener);
    if (oldListener != null) {
      view.removeOnAttachStateChangeListener(oldListener);
    }
    if (newListener != null) {
      view.addOnAttachStateChangeListener(newListener);
    }
  }
}
```

上面的示例比正常情况稍微复杂一些，因为`View`类使用`addOnAttachStateChangeListener()`和`removeOnAttachStateChangeListener()`方法而不是`OnAttachStateChangeListener`的`setter`方法。 `android.databinding.adapters.ListenerUtil`类有助于跟踪以前的监听器，以便可以在绑定适配器中删除它们。

通过使用`@TargetApi(VERSION_CODES.HONEYCOMB_MR1)`注释`OnViewDetachedFromWindow`和`OnViewAttachedToWindow`接口，数据绑定代码生成器知道只应在`Android 3.1(API级别12)`及更高版本上运行时生成监听器，这与`addOnAttachStateChangeListener` 方法支持的版本相同。

# 对象转换
## 自动对象转换
从绑定表达式返回Object时，库会选择用于设置属性值的方法。 Object被强制转换为所选方法的参数类型。在使用ObservableMap类存储数据的应用程序中，此行为很方便，如以下示例所示
```xml
<TextView
 android:text='@{userMap["lastName"]}'
 android:layout_width="wrap_content"
 android:layout_height="wrap_content" />
```

> 注意：您还可以使用object.key表示法引用map中的值。例如，上面示例中的@ {userMap [“lastName”]}可以替换为@ {userMap.lastName}。

表达式中的`userMap`对象返回一个值，该值自动转换为`setText(CharSequence)`方法中的参数类型，该方法用于设置`android:text`属性的值。如果参数类型不明确，则必须在表达式中强制转换返回类型。

## 自定义转换
在某些情况下，特定类型之间需要自定义转换。例如，视图的`android:background`属性需要`Drawable`，但指定的颜色值是整数。以下示例显示了一个期望`Drawable`的属性，但是提供了一个整数：

```java
<View
 android:background="@{isError ? @color/red : @color/white}"
 android:layout_width="wrap_content"
 android:layout_height="wrap_content"/>
```

每当需要Drawable并返回一个整数时，int应该转换为ColorDrawable。可以使用带有BindingConversion注解的静态方法完成转换，如下所示：
```java
@BindingConversion
public static ColorDrawable convertColorToDrawable(int color) {
  return new ColorDrawable(color);
}
```

但是，绑定表达式中提供的值类型必须一致。您不能在同一表达式中使用不同的类型，如以下示例所示：
```java
<View
 android:background="@{isError ? @drawable/error : @color/white}"
 android:layout_width="wrap_content"
 android:layout_height="wrap_content"/>
```