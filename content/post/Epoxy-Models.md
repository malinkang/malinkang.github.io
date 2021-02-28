---
title: Epoxy Models
date: 2020-01-05 15:52:23
tags: ["Java", "读书笔记"]
toc: true
---

## 概览（Overview）

> Epoxy uses EpoxyModel objects to decide which views to display and how to bind data to them. This is similar to the popular ViewModel pattern. Models also allow you to control other aspects of the view, such as the grid span size, id, and saved state.


`Epoxy`使用`EpoxyModel`对象来确定要显示的`view`以及如何为这些`view`绑定数据。这类似于流行的`ViewModel`模式。`Models`也允许控制`view`的其他方面，例如网格范围大小，id和保存状态。

## 创建Model（Creating Models）

> There are several ways to have Epoxy generate your model classes.


有多种方法可以使`Epoxy`生成`Model`类。

* [Annotate custom views]()
* [Use Android databinding]()
* [Use the view holder pattern]()

> Generated model classes are suffixed with _ to indicate that they are generated.

生成的`model`类后缀带`_`表示已生成。

## Model IDs

> The RecyclerView concept of stable ideas is built into Epoxy, meaning each EpoxyModel should have a unique id to identify it. This allows diffing and saved state.

`Epoxy`中内置了`RecyclerView`稳定想法的概念，这意味着每个`EpoxyModel`应该有一个唯一的id来识别它。这允许差异和保存状态。

> Models are assigned ids via the EpoxyModel#id(long) method. This id value would generally come from a database entry, such as the id of a user.

`Model`通过`EpoxyModel`的`id(long)`方法分配id。这个id值一般来自数据库条目，如用户的id。

> However, there are many cases where an EpoxyModel is not backed by a database object and doesn't have a clearly assignable id. In these cases you may use a string as a model's id.

然而，在许多情况下，`EpoxyModel`没有数据库对象的支持，也没有一个明确的可分配的id。在这些情况下，您可以使用一个字符串作为`Model`的id。

```java
model.id("header")
```

> Alternatively, if you have EpoxyModel's represented by object's from different database tables there is a risk of colliding ids. If that is a concern you may use a string to namespace the id.

另外，如果您的EpoxyModel是由不同数据库表中的对象来表示的，那么就有可能发生id冲突。如果这是一个问题，你可以使用一个字符串来命名空间的ID。

```java
model.id("photo", photoId)
model.id("video", videoId)
```

> There are other overloads of id that accept multiple numbers or strings to help you create custom ids as needed. These alternative options are hashed into a 64 bit long to create an id for you. The downside to this approach is that, since the id is computed via a hash, there is the chance for id collisions which will cause an error. However, since a 64 bit hash is used the chance of a collision is extremely small. Assuming an even spread of hashcodes, and several hundred models in the adapter, there would be roughly 1 in 100 trillion chance of a collision. To protected against collisions, EpoxyController supports fallback behavior when duplicates are detected.

id还有其他的重载，可以接受多个数字或字符串，帮助你根据需要创建自定义的id。这些替代选项会被哈希成一个64位的长来为你创建一个id。这种方法的缺点是，由于id是通过哈希计算的，所以有可能出现id碰撞，从而导致错误。然而，由于使用的是64位哈希，所以发生碰撞的几率非常小。假设哈希码的均匀分布，以及适配器中的几百个模型，碰撞的几率大约为100万亿分之一。为了防止碰撞，EpoxyController支持当检测到重复时的回退行为。

### Automatic IDs in EpoxyController

> For models that represent static content (such as a header or loader) you can use the AutoModel annotation with the EpoxyController to automatically generate a unique id for you.

对于表示静态内容的模型（如头或加载器），您可以使用EpoxyController的AutoModel注解为您自动生成一个唯一的id。


## Model Properties

> Each model holds data that it eventually binds to a view. This data is represented by properties:


每个模型都拥有数据，并最终与视图绑定。这些数据由属性来表示。

> * In custom views a property is declared with a @ModelProp annotation on a setter.
> * In databinding layouts each variable represents a model property.
> * In view holders the @EpoxyAttribute field annotation declares a property.


* 在自定义视图中，一个属性是用一个`@ModelProp`注解在`setter`上声明的。
* 在数据绑定布局中，每个变量代表一个模型属性。
* 在视图持有者中，`@EpoxyAttribute`字段注解声明了一个属性。

> Every property must be a type that implements equals and hashcode. These implementations must correctly identify when the property value changes. Epoxy relies on this for diffing. Your properties may not be updated on the view if Epoxy doesn't know they have changed. The exception to this is callbacks, such as click listeners, which you should generally apply the DoNotHash option to.

每个属性必须是实现`equals`和`hashcode`的类型。这些实现必须正确识别属性值的变化。Epoxy依靠这一点来实现差异化。如果Epoxy不知道你的属性发生了变化，你的属性可能不会在视图上更新。例外的情况是回调，比如点击监听器，你一般应该应用`DoNotHash`选项。

> On the generated EpoxyModels each property has a setter and getter; when you create a new model you build it by setting the value for each property.

在生成的`EpoxyModels`上，每个属性都有一个`setter`和`getter`；当你创建一个新的模型时，你通过设置每个属性的值来建立它。

```java
new MyModel_()
       .id(1)
       .title("title")
```
> Id is a required property for every model.

Id是每一个模型的必备属性。

## Model State

> A model's state is determined by its equals and hashCode implementations, which is based on the value of all of the model's properties.

一个模型的状态是由它的equals和hashCode实现的，它是基于模型的所有属性的值来决定的。

> This state is used in diffing to determine when a model has changed so Epoxy can update the view.

这个状态在diffing中用来判断一个模型何时发生了变化，以便Epoxy可以更新视图。

> These methods are generated so you don't have to created them manually.

这些方法是生成的，所以你不必手动创建它们。


## Creating Views

> Each model is typed with the View that it supports. The model implements buildView to create a new instance of that view type. Additionally, getViewType returns an int representing that view type for use in view recycling.

 每个模型都有它所支持的视图类型。模型实现 buildView 来创建该视图类型的新实例。此外，getViewType 返回一个代表该视图类型的 int，用于视图回收。

> When Epoxy receives the calls RecyclerView.Adapter#getItemViewType and RecyclerView.Adapter#onCreateViewHolder it delegates to these methods on the model.

当Epoxy接收到RecyclerView.Adapter#getItemViewType和RecyclerView.Adapter#onCreateViewHolder的调用时，它会将这些方法委托给模型。


> These methods are generated for you unless you are creating models manually.

这些方法是为您生成的，除非您是手动创建模型。

## Binding Views

> When RecyclerView.Adapter#onBindViewHolder is called, Epoxy delegates to the model at the requested position with EpoxyModel#bind(View). This is the model's chance to populate the view with the properties in the model.

当RecyclerView.Adapter#onBindViewHolder被调用时，Epoxy会在请求的位置用EpoxyModel#bind(View)委托给模型。这是模型用模型中的属性填充视图的机会。

> Similarly, Epoxy delegates RecyclerView.Adapter#onViewRecycled to EpoxyModel#unbind(View), giving the model a chance to release any resources associated with the view. This is a good opportunity to clear the view of large or expensive data such as bitmaps.

类似地，Epoxy将RecyclerView.Adapter#onViewRecycled委托给EpoxyModel#unbind(View)，让模型有机会释放与视图相关的任何资源。这是一个很好的机会来清除视图中的大型或昂贵的数据，如位图。

> Since RecyclerView reuses views when possible, a view may be bound multiple times, without unbind necessarily being called in between. You should make sure that your usage of EpoxyModel#bind(View) completely updates the view according to the data in your model.

由于RecyclerView在可能的情况下会重用视图，所以一个视图可能会被多次绑定，中间不一定会被调用unbind。你应该确保EpoxyModel#bind(View)的用法会根据模型中的数据完全更新视图。


> Generated models have an onBind method added to them, which you can use to register a callback for when the model is bound to a view.

生成的模型有一个onBind方法，您可以用它来注册一个回调，当模型绑定到视图时，就可以使用这个方法。

```java
model.onBind((model, view, position) -> // Do something!);
```
> Similarly, onUnbind can be used for an unbind callback.

同样，onUnbind也可用于解除绑定的回调。

> These are useful if you need to take action when a view enters or leaves the screen.

如果您需要在视图进入或离开屏幕时进行操作，这些都很有用。

## View Updates

> When a model's state changes, and the model is bound to a view on screen, Epoxy rebinds the view with only the properties that changed. This partial rebind is more efficient than rebinding the whole view.

当模型的状态发生变化，并且模型被绑定到屏幕上的视图上时，Epoxy仅用变化的属性重新绑定该视图，这种部分重新绑定比重新绑定整个视图更有效。这种部分重新绑定比重新绑定整个视图更有效。

> If you are using models generated from custom views or databinding, then the partial rebind is done automatically for you.

如果你使用从自定义视图或数据绑定生成的模型，那么部分重新绑定将自动为你完成。


> If you are using manually created models you can implement EpoxyModel#bind(T view, EpoxyModel<?> previouslyBoundModel) to check which properties changed and update the view as needed.

 如果你使用的是手动创建的模型，你可以实现EpoxyModel#bind(T view, EpoxyModel<?> previouslyBoundModel)来检查哪些属性改变了，并根据需要更新视图。

> Note: This only works with EpoxyController, not the legacy EpoxyAdapter class.

注意：这只适用于EpoxyController，而不是传统的EpoxyAdapter类。


## Click Listeners

> If a model has a property of type View.OnClickListener then the generated model will include an additional, overloaded method for setting that click listener. The overloaded method will take a OnModelClickListener parameter.

如果一个模型有一个View.OnClickListener类型的属性，那么生成的模型将包含一个额外的、重载的方法来设置该点击监听器。该重载方法将取一个OnModelClickListener参数。


> This listener is typed with two parameters: the model's type and the model's view type. Implementations must implement void onClick(T model, V parentView, View clickedView, int position). This callback gives you the model whose view was clicked, the top level view (or view holder if you are using EpoxyModelWithHolder), the view that received the clicked, and the position of the model in the adapter.

这个监听器的类型有两个参数：模型的类型和模型的视图类型。实现必须实现void onClick(T model, V parentView, View clickedView, int position)。这个回调会给你提供视图被点击的模型、顶层视图（如果你使用的是EpoxyModelWithHolder，则是视图支架）、收到点击的视图以及模型在适配器中的位置。


> You should use the position provided by this listener instead of saving a reference to the model's position when it was added to the adapter/controller. This is because if the model is moved RecyclerView does an optimization to not rebind the new model since it knows the data did not change. This also means that the model returned by onClick is not necessarily the most recently created model, if the model did not need to be rebound to the view.


你应该使用这个监听器提供的位置，而不是保存模型被添加到适配器/控制器时的位置引用。这是因为如果模型被移动，RecyclerView会做一个优化，不重新绑定新的模型，因为它知道数据没有改变。这也就意味着，如果模型不需要重新绑定到视图，那么onClick返回的模型不一定是最近创建的模型。



## OnCheckedChangeListener

> Similarly, if a model has a property of type CompoundButton.OnCheckedChangeListener then an overloaded method will be generated on the model of type OnModelCheckedChangeListener - to provide access to the model and position when the listener is triggered.

同样，如果一个模型有一个类型为CompoundButton.OnCheckedChangeListener的属性，那么将在模型上生成一个类型为`OnModelCheckedChangeListener`的重载方法--以在触发监听器时提供对模型和位置的访问。
