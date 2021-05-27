---
title: 实现navigation
date: 2018-06-07 16:13:46
tags:
toc: true
---



导航体系结构组件简化了应用程序中目标之间导航的实现。目的地是应用中的特定屏幕。默认情况下，导航体系结构组件包括支持`fragment`和`activity`作为目标，但您也可以添加对新类型目标的支持。一组目的地组成一个应用程序的“导航图”。

<!--more-->

除目的地之外，导航图在目标之间具有称为“操作”的连接。图1显示了包含6个目的地的示例应用程序的导航图的直观表示，该应用程序由5个操作连接。

![img](/images/navigation-graph.png)

### 在项目中设置navigation


在您创建导航图之前，您必须为您的项目设置导航结构组件。要在`Android Studio`中设置您的项目，请执行以下步骤。

1. 将`navigation`组件添加到您的应用程序或模块的`build.gradle`文件。

```groovy
dependencies {
    def nav_version = "1.0.0-alpha01"

    implementation "android.arch.navigation:navigation-fragment:$nav_version" // use -ktx for Kotlin
    implementation "android.arch.navigation:navigation-ui:$nav_version" // use -ktx for Kotlin

    // optional - Test helpers
    androidTestImplementation "android.arch.navigation:navigation-testing:$nav_version" // use -ktx for Kotlin
}
```

2. 在`Project`窗口中，右键单击`res`目录并选择`New> Android`资源文件。出现新资源对话框。

3. 在“文件名”字段中输入名称，例如“nav_graph”。
4. 从资源类型下拉列表中选择`Navigation`。

5. 点击确定。发生以下情况：

* 导航资源目录在res目录中创建。
* nav_graph.xml文件在导航目录中创建。
* nav_graph.xml文件将在导航编辑器中打开。这个XML文件包含您的导航图。

6. 点击文本标签切换到XML文本视图。空导航图的XML如下所示：

```xml
<?xml version="1.0" encoding="utf-8"?>
<navigation xmlns:android="http://schemas.android.com/apk/res/android">
</navigation>
```

7. 点击设计返回导航编辑器。

### 浏览导航编辑器

在导航编辑器中，您可以快速构建导航图，而无需手动构建图形的XML。如图2所示，导航编辑器有三个部分：

![img](/images/navigation-editor.png)

导航编辑器的部分是：

①目的地列表 - 列出当前在图表编辑器中的所有目的地。

②图表编辑器 - 包含您的导航图的可视化表示。

③属性编辑器 - 包含与导航图中的目的地和动作相关的属性。

### 识别目的地

创建导航图的第一步是确定您的应用的目的地。您可以在现有项目中创建空白目的地或创建`fragment`和`activity`的目的地。


要识别您的应用的目标，请使用以下步骤。

1. 在图表编辑器中，单击新建目标。出现新目标对话框。

2. 点击创建空白目的地或点击`fragment`或`activity`。出现新的`Android`组件对话框。

3. 在`Fragment Name`字段中输入一个名称。这个名字是片段类的名字。

4. 在`Fragment Layout Name`字段中输入名称。该名称是该片段的布局文件的名称。

5. 点击完成。代表目标的框出现在图形编辑器和目标列表中。发生以下情况：

* 如果您创建了空白目的地，图表编辑器会在目的地显示消息“Hello blank fragment”。如果您单击片段或活动，图表编辑器会显示该活动或片段的布局预览。
* 为您的项目创建一个Fragment子类。该课程具有您在第3步中分配的名称。
* 资源文件是为您的项目创建的。该文件具有您在步骤4中分配的名称。

![img](/images/navigation-newexisting.png)


6.点击新插入的目的地以突出显示目的地。以下属性出现在“属性”面板中：

* 类型字段包含“Fragment”或“Activity”，用于指示目标是否作为源代码中的片段或活动实现。
* 标签字段包含目标的XML布局文件的名称。
* ID字段包含将用于引用代码中的目的地的目的地的ID。
* 类字段包含目标的类的名称。



点击文本标签切换到XML视图。 XML现在包含基于现有类和布局文件的名称的id，名称（类名称），标签和布局属性：


```xml
<?xml version="1.0" encoding="utf-8"?>
<navigation xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    xmlns:android="http://schemas.android.com/apk/res/android"
    app:startDestination="@id/blankFragment">
    <fragment
        android:id="@+id/blankFragment"
        android:name="com.example.cashdog.cashdog.BlankFragment"
        android:label="Blank"
        tools:layout="@layout/fragment_blank" />
</navigation>
```

### 连接目的地



您必须有多个目的地才能连接目的地。以下是带有两个空白目标的导航图的XML：

```xml
<?xml version="1.0" encoding="utf-8"?>
<navigation xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    xmlns:android="http://schemas.android.com/apk/res/android"
    app:startDestination="@id/blankFragment">
    <fragment
        android:id="@+id/blankFragment"
        android:name="com.example.cashdog.cashdog.BlankFragment"
        android:label="fragment_blank"
        tools:layout="@layout/fragment_blank" />
    <fragment
        android:id="@+id/blankFragment2"
        android:name="com.example.cashdog.cashdog.BlankFragment2"
        android:label="Blank2"
        tools:layout="@layout/fragment_blank_fragment2" />
</navigation>
```

目的地使用`action`连接。连接两个目的地：


从图形编辑器中，将光标悬停在您希望用户从中导航的目标的右侧。目的地上会出现一个圆圈。

![img](/images/navigation-actioncircle.png)

单击并按住，将光标拖到您希望用户导航到的目标上，然后释放。绘制一条线以指示两个目的地之间的导航。




3.点击箭头突出显示该操作。以下属性出现在“属性”面板中：

* 类型字段包含“Action”。
* ID字段包含为`action`系统分配的ID。
* 目标字段包含目标`fragment`或`activity`的ID。


4.点击文本标签切换到XML视图。操作元素已添加到父目标。该操作具有系统分配的ID和目标属性，其中包含下一个目标的ID。例如：

```xml
<?xml version="1.0" encoding="utf-8"?>
<navigation xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    xmlns:android="http://schemas.android.com/apk/res/android"
    app:startDestination="@id/blankFragment">
    <fragment
        android:id="@+id/blankFragment"
        android:name="com.example.cashdog.cashdog.BlankFragment"
        android:label="fragment_blank"
        tools:layout="@layout/fragment_blank" >
        <action
            android:id="@+id/action_blankFragment_to_blankFragment2"
            app:destination="@id/blankFragment2" />
    </fragment>
    <fragment
        android:id="@+id/blankFragment2"
        android:name="com.example.cashdog.cashdog.BlankFragment2"
        android:label="fragment_blank_fragment2"
        tools:layout="@layout/fragment_blank_fragment2" />
</navigation>
```

### 指定一个屏幕作为起始目的地




图形编辑器会在您的应用中放置第一个目标名称旁边的房屋图标。此图标表示这是导航图中的起始目标。您可以使用以下步骤将另一个目标指定为起始目标：

5.从图形编辑器中，单击目标。目标被突出显示。

6.在“属性”面板中单击`Click Set Start Destination`。目标现在是开始目标。


### 修改一个Activity去持有navigation

一个`activity`通过将一个`NavHost`实现添加到你的`activity`的布局中为app持有`navigation`。`NavHost`是一个空白的视图，当用户在您的应用中导航时，目的地被换入和换出。

导航体系结构组件的默认`NavHost`实现是`NavHostFragment`。

在包含`NavHost`后，必须使用`navGraph`属性将导航图与`NavHostFragment`关联起来。以下`fragment`显示了如何在活动的布局文件中包含`NavHostFragment`并将导航图与`NavHostFragment`相关联：

```xml
?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <fragment
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:id="@+id/my_nav_host_fragment"
        android:name="androidx.navigation.fragment.NavHostFragment"
        app:navGraph="@navigation/nav_graph"
        app:defaultNavHost="true"
        />

</android.support.constraint.ConstraintLayout>
```

前面的示例包含一个应用程序：`defaultNavHost =“true”`属性。该属性可确保您的`NavHostFragment`拦截系统后退按钮。您还将覆盖`AppCompatActivity.onSupportNavigateUp()`并调用`NavController.navigateUp`，如下所示：

```java
@Override
public boolean onSupportNavigateUp() {
    return Navigation.findNavController(this, R.id.nav_host_fragment).navigateUp();
}

```

### 将目标绑定到UI小部件


导航到目的地是使用`NavController`类完成的。可以使用以下静态方法之一来获取`NavController`：

* NavHostFragment.findNavController(Fragment)
* Navigation.findNavController(Activity, @IdRes int viewId)
* Navigation.findNavController(View)

在您获取`NavController`后，使用其`navigate()`方法导航到目标。 `navigate()`方法接受资源ID。该ID可以是导航图形或动作中特定目的地的ID。使用操作的ID而不是目标的资源ID具有优势，例如将过渡与导航关联。有关转场的更多信息，请参阅在目的地之间创建转场。

以下代码片段显示了如何导航到`ViewTransactionsFragment`：

```java
viewTransactionsButton.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View view) {
        Navigation.findNavController(view).navigate(R.id.viewTransactionsAction);
    }
});
```

Android系统维护一个包含上次访问目标的后台堆栈。当用户打开应用程序时，应用程序的第一个目标位于堆栈上。每次调用`navigate()`方法时，都会在堆栈顶部放置另一个目标。相反，按“上”或“后”按钮分别调用`NavController.navigateUp()`和`NavController.popBackStack()`方法，以将顶层目标弹出堆栈。

对于按钮，还可以使用`Navigation`类的`createNavigateOnClickListener()`便捷方法导航到目标：

```java
button.setOnClickListener(Navigation.createNavigateOnClickListener(R.id.next_fragment, null));
```

#### 将目标绑定到菜单驱动的UI组件




您可以使用目标的id作为XML中导航抽屉或溢出菜单项的相同ID，将目标绑定到导航抽屉和溢出菜单。以下代码片段显示了其详细信息屏幕目标，其ID为`details_page_fragment`：

```xml
<fragment android:id="@+id/details_page_fragment"
     android:label="@string/details"
     android:name="com.example.android.myapp.DetailsFragment" />
```

对目的地和菜单项使用相同的ID会自动将目的地与菜单项相关联。以下XML显示了如何将片段目标与导航抽屉中的菜单项相关联（例如，menu_nav_drawer.xml）：


```xml
<item
    android:id="@id/details_page_fragment"
    android:icon="@drawable/ic_details"
    android:title="@string/details" />
```


以下XML显示了如何将详细信息目标绑定到溢出菜单（例如menu_overflow.xml）：

```xml
<item
    android:id="@id/details_page_fragment"
    android:icon="@drawable/ic_details"
    android:title="@string/details"
    android:menuCategory:"secondary" />
```




导航体系结构组件包含一个`NavigationUI`类。这个类有几个静态方法，你可以使用连接菜单项和导航目的地。例如，以下代码显示如何使用`setupWithNavController()`方法将菜单抽屉中的项目连接到`NavigationView`：

```java
NavigationView navigationView = (NavigationView) findViewById(R.id.nav_view);
NavigationUI.setupWithNavController(navigationView, navController);
```

有必要使用这些`NavigationUI`方法设置菜单驱动的导航组件，以便这些UI元素的状态与对`NavController`的更改保持同步。


### 在目标之间传递数据


您可以通过两种方式在目标之间传递数据：使用`Bundle`对象或使用`safeargs Gradle插件`以类型安全的方式传递数据。使用以下步骤使用`Bundle`对象在目标之间传递数据。如果您使用的是Gradle，请考虑按照类型安全的方式在目标之间传递数据中的说明。

1. 从`Graph Editor`中，单击接受参数的目标。目标亮点。

2. 单击“属性”面板的“参数”部分中的添加（+）。出现空名称和默认值字段

3. 双击名称并输入参数的名称。

4. 按Tab并输入参数的默认值。

5. 点击此目的地之前的操作。参数默认值应包含您新添加的参数。

6. 点击文本标签切换到XML视图。具有`name`和`defaultValue`属性的参数元素已添加到目标中：

```xml
<fragment
   android:id="@+id/confirmationFragment"
   android:name="com.example.cashdog.cashdog.ConfirmationFragment"
   android:label="fragment_confirmation"
   tools:layout="@layout/fragment_confirmation">
   <argument android:name="amount" android:defaultValue=”0” />
```

7.在您的代码中，使用`navigate()`法创建一个包并将其传递到目标：

```java
Bundle bundle = new Bundle();
bundle.putString("amount", amount);
Navigation.findNavController(view).navigate(R.id.confirmationAction, bundle);
```

在您的接收目标的代码中，使用`getArguments()`方法检索捆绑并使用其内容：

```java
TextView tv = view.findViewById(R.id.textViewAmount);
tv.setText(getArguments().getString("amount"));
```

### 以类型安全的方式在目标之间传递数据

导航架构组件有一个名为`safeargs`的`Gradle`插件，它生成简单的对象和构建器类，以便对目标和动作指定的参数进行类型安全访问。安全参数建立在`Bundle`方法的基础上，但需要一些额外的代码来换取更多的类型安全。如果您使用`Gradle`，则可以使用安全参数插件。要添加此插件，请将'androidx.navigation.safeargs'插件添加到您的`build.gradle`文件中。例如：

```groovy
apply plugin: 'com.android.application'
apply plugin: 'androidx.navigation.safeargs'

android {
   //...
}
```




在配置了Gradle插件之后，请按照以下步骤使用类型安全的参数。

1. 从Graph Editor中，单击收到参数的目标。目标高亮。

2. 在“属性”面板的“参数”部分中单击+。出现空名称和默认值字段。

3. 双击名称并输入参数的名称。

4. 按Tab键并从下拉列表中选择参数的类型。

5. 按Tab并输入参数的默认值。

6. 点击此目的地之前的操作。参数默认值应包含您新添加的参数。

7. 点击文本标签切换到XML视图。具有`name`和`defaultValue`属性的参数元素已添加到目标中：


```xml
<fragment
    android:id="@+id/confirmationFragment"
    android:name="com.example.buybuddy.buybuddy.ConfirmationFragment"
    android:label="fragment_confirmation"
    tools:layout="@layout/fragment_confirmation">
    <argument android:name="amount" android:defaultValue="1" app:type="integer"/>
</fragment>
```

当您使用`safeargs`插件生成代码时，会为动作和发送和接收目标创建简单的对象和构建器类。这些类是：一个动作发源地的类，附加单词“Directions”。

* 因此，如果源片段的标题为`SpecifyAmountFragment`，则生成的类称为`SpecifyAmountFragmentDirections`。这个类有一个方法，以用于传递参数的动作命名，以捆绑参数，如`confirmationAction()`。

* 一个内部类，其名称基于用于传递参数的操作。如果传递操作称为`confirmationAction`，则该类将被命名为`ConfirmationAction`。

* 传递参数的目的地的类，附带单词Args。

因此，如果目标片段的标题为`ConfirmationFragment`，则生成的类将称为ConfirmationFragmentArgs。使用此类的`fromBundle()`方法来检索参数。

以下代码显示如何使用这些方法来设置参数并将其传递给`navigate()`方法。

```java
@Override
public void onClick(View view) {
   EditText amountTv = (EditText) getView().findViewById(R.id.editTextAmount);
   int amount = Integer.parseInt(amountTv.getText().toString());
   ConfirmationAction action =
           SpecifyAmountFragmentDirections.confirmationAction()
   action.setAmount(amount)
   Navigation.findNavController(view).navigate(action);
}
```

在您的接收目标的代码中，使用`getArguments()`法检索捆绑并使用其内容：


```java
@Override
public void onViewCreated(View view, @Nullable Bundle savedInstanceState) {
    TextView tv = view.findViewById(R.id.textViewAmount);
    int amount = ConfirmationFragmentArgs.fromBundle(getArguments()).getAmount();
    tv.setText(amount + "")
}
```

### 将目的地组合到一个嵌套的导航图中

一系列目的地可以在导航图中组成一个子图。子图称为“嵌套图”，而包含的图称为“根图”。嵌套图有助于组织和重用应用程序UI的部分，如单独的登录流。

与根图一样，嵌套图必须将目标标识为起始目标。嵌套图封装其目的地;嵌套图形之外的目标（例如根图形上的目标）仅通过其起始目标访问嵌套图形。图6显示了简单转账应用程序的导航图。此图有两个流程：允许用户发送资金的流程和允许用户查看其余额的流程。

![img](/images/navigation-pre-nestedgraph.png)






