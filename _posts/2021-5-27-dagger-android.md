---
title: "Dagger2在Android中的使用"
date: 2018-07-25T11:56:12+08:00
draft: false
---

与其他大多数依赖注入框架相比，`Dagger2`的主要优点之一是其严格生成的实现（无反射）意味着它可以在`Android`应用程序中使用。但是，在`Android`应用程序中使用`Dagger`时仍有一些注意事项。

<!--more-->

原文：https://google.github.io/dagger/android


使用`Dagger`编写`Android`应用程序的主要困难之一是很多`Android`框架类都是由操作系统本身实例化的，例如`Activity`和`Fragment`，但是如果`Dagger`可以创建所有注入的对象，则`Dagger`的效果最好。相反，您必须在生命周期方法中执行成员注入。这意味着许多类最终看起来像：

```cpp
public class FrombulationActivity extends Activity {
  @Inject Frombulator frombulator;

  @Override
  public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    // DO THIS FIRST. Otherwise frombulator might be null!
    ((SomeApplicationBaseType) getContext().getApplicationContext())
        .getApplicationComponent()
        .newActivityComponentBuilder()
        .activity(this)
        .build()
        .inject(this);
    // ... now you can write the exciting code
  }
}
```

### 注入Activity对象



1. 在您的应用程序组件中安装`AndroidInjectionModule`以确保这些基本类型所需的所有绑定都可用。

```java
@Singleton
@Component(
    modules = [
        AndroidInjectionModule::class, //安装AndroidInjectionModule
        AppModule::class,
        MainActivityModule::class]
)
interface AppComponent {
    @Component.Builder
    interface Builder {
        @BindsInstance
        fun application(application: Application): Builder

        fun build(): AppComponent
    }
    //注入App
    fun inject(githubApp: GithubApp)
}
```

2. 首先编写一个实现`AndroidInjector<YourActivity>`的`@Subcomponent`和一个继承`AndroidInjector.Builder<YourActivity>`的`@Subcomponent.Builder`。


```java
@Subcomponent(modules = ...)
public interface YourActivitySubcomponent extends AndroidInjector<YourActivity> {
  @Subcomponent.Builder
  public abstract class Builder extends AndroidInjector.Builder<YourActivity> {}
}
```

3. 定义子组件后，通过定义一个绑定子组件层次结构并将其添加到注入应用程序的组件的模块，将其添加到组件层次结构中。

```java
@Module(subcomponents = YourActivitySubcomponent.class)
abstract class YourActivityModule {
  @Binds
  @IntoMap
  @ActivityKey(YourActivity.class)
  abstract AndroidInjector.Factory<? extends Activity>
      bindYourActivityInjectorFactory(YourActivitySubcomponent.Builder builder);
}

@Component(modules = {..., YourActivityModule.class})
interface YourApplicationComponent {}
```



专业提示：如果您的子组件及其构建器没有第2步中提到的其他方法或超类型，您可以使用`@ContributesAndroidInjector`为您生成它们。添加一个抽象模块方法，该方法返回您的活动，使用@ContributesAndroidInjector对其进行注释，并指定要安装到子组件中的模块，而不是步骤2和3。如果子组件需要作用域，则也可以将范围注释应用于该方法。

```java
@ActivityScope
@ContributesAndroidInjector(modules = { /* modules to install into the subcomponent */ })
abstract YourActivity contributeYourActivityInjector();
```



4. 接下来，让您的`Application`实现`HasActivityInjector`并且`@Inject`一个从`activityInjector()`方法返回的`DispatchingAndroidInjector<Activity>`。

```java
public class YourApplication extends Application implements HasActivityInjector {
  @Inject DispatchingAndroidInjector<Activity> dispatchingActivityInjector;

  @Override
  public void onCreate() {
    super.onCreate();
    DaggerYourApplicationComponent.create()
        .inject(this);
  }

  @Override
  public AndroidInjector<Activity> activityInjector() {
    return dispatchingActivityInjector;
  }
}
```
5. 最终，在你的`Activity.onCreate()`方法中，在调用`super.onCreate();`之前调用`AndroidInjection.inject(this)`。

```java
public class YourActivity extends Activity {
  public void onCreate(Bundle savedInstanceState) {
    AndroidInjection.inject(this);
    super.onCreate(savedInstanceState);
  }
}
```

#### 如何工作

`AndroidInjection.inject()`从`Application`中获取一个`DispatchingAndroidInjector<Activity>`并且传递你的`activity`到`inject(this)`。`DispatchingAndroidInjector`为您的Activity类（它是YourActivitySubcomponent.Builder）查找`AndroidInjector.Factory`，创建`AndroidInjector（它是YourActivitySubcomponent）`，并将您的Activity传递给`inject（YourActivity）`。

`AndroidInjection`的`inject`方法
```java
/**
 * Injects {@code activity} if an associated {@link AndroidInjector} implementation can be found,
 * otherwise throws an {@link IllegalArgumentException}.
 *
 * @throws RuntimeException if the {@link Application} doesn't implement {@link
 *     HasActivityInjector}.
 */
public static void inject(Activity activity) {
  checkNotNull(activity, "activity");
  Application application = activity.getApplication();
  if (!(application instanceof HasActivityInjector)) {
    throw new RuntimeException(
        String.format(
            "%s does not implement %s",
            application.getClass().getCanonicalName(),
            HasActivityInjector.class.getCanonicalName()));
  }
//
  AndroidInjector<Activity> activityInjector =
      ((HasActivityInjector) application).activityInjector();
  checkNotNull(activityInjector, "%s.activityInjector() returned null", application.getClass());

  activityInjector.inject(activity);
}

```

### 注入Fragment对象

>Injecting a Fragment is just as simple as injecting an Activity. Define your subcomponent in the same way, replacing Activity type parameters with Fragment, @ActivityKey with @FragmentKey, and HasActivityInjector with HasFragmentInjector.

注入一个`Fragment`像注入一个`Activity`一样简单。以相同的方式定义你的`subcomponent`，使用`Fragment`替换`Activity`，`@FragmentKey`替换`@ActivityKey`,`HasFragmentInjector`替换`HasActivityInjector`。

>Instead of injecting in onCreate() as is done for Activity types, inject Fragments to in onAttach().

不像在`Activity`类型中那样在`onCreate()`中注入，而是在`onAttach()`中注入`Fragment`。

> Unlike the modules defined for Activitys, you have a choice of where to install modules for Fragments. You can make your Fragment component a subcomponent of another Fragment component, an Activity component, or the Application component — it all depends on which other bindings your Fragment requires. After deciding on the component location, make the corresponding type implement HasFragmentInjector. For example, if your Fragment needs bindings from YourActivitySubcomponent, your code will look something like this:

与为`Activity`定义的模块不同，您可以选择在哪里为`Fragments`安装模块。你可以让你的`Fragment`组件成为另一个`Fragment`子组件，`Activity`组件或`Application`组件的一个子组件 - 这一切都取决于你的`Fragment`需要的其他绑定。确定组件位置后，使相应的类型实现`HasFragmentInjector`。例如，如果您的`Fragment`需要来自`YourActivitySubcomponent`的绑定，那么您的代码将如下所示：

```java
public class YourActivity extends Activity
    implements HasFragmentInjector {
  @Inject DispatchingAndroidInjector<Fragment> fragmentInjector;

  @Override
  public void onCreate(Bundle savedInstanceState) {
    AndroidInjection.inject(this);
    super.onCreate(savedInstanceState);
    // ...
  }

  @Override
  public AndroidInjector<Fragment> fragmentInjector() {
    return fragmentInjector;
  }
}

public class YourFragment extends Fragment {
  @Inject SomeDependency someDep;

  @Override
  public void onAttach(Activity activity) {
    AndroidInjection.inject(this);
    super.onAttach(activity);
    // ...
  }
}

@Subcomponent(modules = ...)
public interface YourFragmentSubcomponent extends AndroidInjector<YourFragment> {
  @Subcomponent.Builder
  public abstract class Builder extends AndroidInjector.Builder<YourFragment> {}
}

@Module(subcomponents = YourFragmentSubcomponent.class)
abstract class YourFragmentModule {
  @Binds
  @IntoMap
  @FragmentKey(YourFragment.class)
  abstract AndroidInjector.Factory<? extends Fragment>
      bindYourFragmentInjectorFactory(YourFragmentSubcomponent.Builder builder);
}

@Subcomponent(modules = { YourFragmentModule.class, ... }
public interface YourActivityOrYourApplicationComponent { ... }

```


### 基本框架类型

> Because DispatchingAndroidInjector looks up the appropriate AndroidInjector.Factory by the class at runtime, a base class can implement HasActivityInjector/HasFragmentInjector/etc as well as call AndroidInjection.inject(). All each subclass needs to do is bind a corresponding @Subcomponent. Dagger provides a few base types that do this, such as DaggerActivity and DaggerFragment, if you don’t have a complicated class hierarchy. Dagger also provides a DaggerApplication for the same purpose — all you need to do is to extend it and override the applicationInjector() method to return the component that should inject the Application.

因为`DispatchingAndroidInjector`在运行时按类查找适当的`AndroidInjector.Factory`，所以基类可以实现`HasActivityInjector`、`HasFragmentInjectoretc`等等以及调用`AndroidInjection.inject()`。每个子类都需要做的就是绑定一个对应的`@Subcomponent`。如果您没有复杂的类层次结构，`Dagger`会提供一些基本类型，例如`DaggerActivity`和`DaggerFragment`。Dagger还为同样的目的提供了一个DaggerApplication你需要做的就是扩展它并覆盖applicationInjector()方法来返回应该注入应用程序的组件。


以下类型也包括在内：

* DaggerService和DaggerIntentService
* DaggerBroadcastReceiver
* DaggerContentProvider

> Note: DaggerBroadcastReceiver should only be used when the BroadcastReceiver is registered in the AndroidManifest.xml. When the BroadcastReceiver is created in your own code, prefer constructor injection instead.


注意：DaggerBroadcastReceiver只能在AndroidManifest.xml中注册BroadcastReceiver时使用。在您自己的代码中创建BroadcastReceiver时，请改为使用构造函数注入。

### 支持库

> For users of the Android support library, parallel types exist in the dagger.android.support package. Note that while support Fragment users have to bind AndroidInjector.Factory<? extends android.support.v4.app.Fragment>, AppCompat users should continue to implement AndroidInjector.Factory<? extends Activity> and not <? extends AppCompatActivity> (or FragmentActivity).

对于Android支持库的用户，`dagger.android.support`包中存在并行类型。请注意，尽管支持`Fragment`的用户必须绑定`AndroidInjector.Factory <？extends android.support.v4.app.Fragment>`，`AppCompat`用户应该继续实现`AndroidInjector.Factory <？extends Activity>`而不是`<？extends AppCompatActivity>`（或FragmentActivity）。

### 如何获取它

将以下内容添加到您的build.gradle中：

```groovy
dependencies {
  compile 'com.google.dagger:dagger-android:2.x'
  compile 'com.google.dagger:dagger-android-support:2.x' // if you use the support libraries
  annotationProcessor 'com.google.dagger:dagger-android-processor:2.x'
}
```

### 何时注入

> Constructor injection is preferred whenever possible because javac will ensure that no field is referenced before it has been set, which helps avoid NullPointerExceptions. When members injection is required (as discussed above), prefer to inject as early as possible. For this reason, DaggerActivity calls AndroidInjection.inject() immediately in onCreate(), before calling super.onCreate(), and DaggerFragment does the same in onAttach(), which also prevents inconsistencies if the Fragment is reattached.

只要有可能，构造函数注入是首选，因为javac将确保没有字段在被设置之前被引用，这有助于避免NullPointerException。 当需要成员注射（如上所述）时，倾向于尽早注射。 出于这个原因，DaggerActivity在调用`super.onCreate()`之前立即在`onCreate()`中调用AndroidInjection.inject()，并且DaggerFragment在onAttach()中也是这样做的，这也可以防止在重新连接片段时出现不一致。


> It is crucial to call AndroidInjection.inject() before super.onCreate() in an Activity, since the call to super attaches Fragments from the previous activity instance during configuration change, which in turn injects the Fragments. In order for the Fragment injection to succeed, the Activity must already be injected. For users of ErrorProne, it is a compiler error to call AndroidInjection.inject() after super.onCreate().

在Activity中的super.onCreate()之前调用AndroidInjection.inject()是非常重要的，因为超级调用在配置更改期间将前一个活动实例的碎片连接到Fragments，而这又会导致碎片。 为了使片段注入成功，该活动必须已经被注入。 对于ErrorProne的用户，在super.onCreate()之后调用AndroidInjection.inject()是一个编译器错误。

### 常见问题


> AndroidInjector.Factory is intended to be a stateless interface so that implementors don’t have to worry about managing state related to the object which will be injected. When DispatchingAndroidInjector requests a AndroidInjector.Factory, it does so through a Provider so that it doesn’t explicitly retain any instances of the factory. Because the AndroidInjector.Builder implementation that is generated by Dagger does retain an instance of the Activity/Fragment/etc that is being injected, it is a compile-time error to apply a scope to the methods which provide them. If you are positive that your AndroidInjector.Factory does not retain an instance to the injected object, you may suppress this error by applying @SuppressWarnings("dagger.android.ScopedInjectoryFactory") to your module method.

AndroidInjector.Factory旨在成为无状态接口，以便实现者不必担心管理与将被注入的对象相关的状态。 当DispatchingAndroidInjector请求一个AndroidInjector.Factory时，它通过一个Provider来这样做，以便它不明确地保留工厂的任何实例。 由于由Dagger生成的AndroidInjector.Builder实现确实保留了正在被注入的Activity / Fragment /等的实例，因此将范围应用于提供它们的方法时发生编译时错误。 如果你肯定你的AndroidInjector.Factory没有为注入对象保留一个实例，你可以通过在模块方法中应用@SuppressWarnings（“dagger.android.ScopedInjectoryFactory”）来消除这个错误。

### 有用链接
* [New Android Injector with Dagger 2 — part 1](https://medium.com/@iammert/new-android-injector-with-dagger-2-part-1-8baa60152abe)
