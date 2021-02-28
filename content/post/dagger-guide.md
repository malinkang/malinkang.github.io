---
title: "Dagger使用指南"
date: 2016-07-25T09:47:31+08:00
draft: false
toc: true
---



任何应用程序中最好的类是那些做事情的类：`BarcodeDecoder`，`KoopaPhysicsEngine`和`AudioStreamer`。 这些类具有依赖性; 也许是一个`BarcodeCameraFinder`，`DefaultPhysicsEngine`和一个`HttpStreamer`。

<!--more-->

相比之下，任何应用程序中最糟糕的类是那些占用空间而又没有做太多事情类：`BarcodeDecoderFactory`，`CameraServiceLoader`和`MutableContextWrapper`。 这些类是将有趣的东西连接在一起。

`Dagger`是这些`FactoryFactory`类的替代品，它们实现了依赖注入设计模式，无需编写样板文件。 它可以让你专注于有趣的类。 声明依赖关系，指定如何满足它们，并发布您的应用程序。

通过构建在标准的`javax.inject`注解（JSR 330）上，每个类都很容易测试。 您不需要大量的样板就可以将`RpcCreditCardService`交换为`FakeCreditCardService`。

依赖注入不仅仅用于测试。 它还可以轻松创建可重复使用的可互换模块。 您可以在所有应用程序中共享相同的`AuthenticationModule`。 您可以在开发过程中运行`DevLoggingModule`，在生产中运行`ProdLoggingModule`以在每种情况下获得正确的行为。

# 为什么Dagger2不同

依赖注入框架已经很多年，有多种用于配置和注入的API。那么，为什么要重新发明轮子？`Dagger2`是第一个使用生成的代码实现完整堆栈。 指导原则是生成模仿用户可能手写的代码的代码，以确保依赖注入尽可能简单，可追踪和高性能。 有关设计的更多背景，请观看Gregory Kick的[演讲](https://www.youtube.com/watch?v=oK_XtfXPkqw&feature=youtu.be)（[幻灯片](https://docs.google.com/presentation/d/1fby5VeGU9CN8zjw4lAb2QPPsKRxx6mSwCe9q7ECNSJQ/pub?start=false&loop=false&delayms=3000&slide=id.p)）。

# 使用Dagger

我们将通过构建咖啡机来演示依赖注入和Dagger。 有关可以编译和运行的完整示例代码，请参阅Dagger的[coffee example](https://github.com/google/dagger/tree/master/examples/simple/src/main/java/coffee).

## 声明依赖

`Dagger`构造应用程序类的实例并满足它们的依赖关系。 它使用`javax.inject.Inject`注释来标识它感兴趣的构造函数和字段。

使用`@Inject`来注释`Dagger`应该用来创建类的实例的构造函数。当一个新的实例被请求时，Dagger将获得所需的参数值并调用这个构造函数。

```java
class Thermosiphon implements Pump {
  private final Heater heater;

  @Inject
  Thermosiphon(Heater heater) {
    this.heater = heater;
  }
  ...
}
```
`Dagger`可以直接注入字段。 在这个例子中，它为`heater`段获得一个`Heater`实例，为`pump`字段获得一个`Pump`实例。

```java
class CoffeeMaker {
  @Inject Heater heater;
  @Inject Pump pump;
  ...
}
```

如果您的类具有`@Inject`注释的字段，但没有`@Inject`注释的构造函数，Dagger会根据请求注入这些字段，但不会创建新的实例。 使用`@Inject`注释添加一个无参数构造函数，以指示`Dagger`也可以创建实例。

Dagger也支持方法注入，虽然通常建议使用构造函数或字段注入。

缺少`@Inject`注释的类不能由`Dagger`构造。

## 满足依赖

默认情况下，Dagger通过构建上面描述的请求类型的实例来满足每个依赖。当你申请一个`CoffeeMaker`时，它会通过调用`new CoffeeMaker()`并设置它的可注射字段来获得一个。

但是`@Inject`不能作用于每个地方

* 接口不能构造
* 第三方类不能被注解。
* 必须配置可配置的对象

对于`@Inject`不够或不方便的情况，请使用 [`@Provides`](https://dagger.dev/api/latest/dagger/Provides.html)注解方法来满足依赖关系。 该方法的返回类型定义了它满足哪个依赖关系。

例如，只要需要加热器，就调用`provideHeater()`：

```java
@Provides static Heater provideHeater() {
  return new ElectricHeater();
}
```

`@Provides`方法有可能拥有自己的依赖关系。 无论何时需要`Pump`，该设备都会返回`Thermosiphon`：

```java
@Provides static Pump providePump(Thermosiphon pump) {
  return pump;
}
```



所有`@Provides`方法都必须属于一个模块。 这些只是具有`@Module`注释的类。

```java
@Module
class DripCoffeeModule {
  @Provides static Heater provideHeater() {
    return new ElectricHeater();
  }

  @Provides static Pump providePump(Thermosiphon pump) {
    return pump;
  }
}
```



按照惯例，`@Provides`方法以`provide`为前缀命名，模块类以`Module`为后缀命名。

## 构建图

`@Inject`和`@Provide`注解的类形成了由它们的依赖关系链接的对象的图形。调用代码像应用程序的`main`方法或`Android`应用程序通过一个定义良好的根访问该图形。在`Dagger2`中，该集合由一个接口定义，该接口的方法没有参数并返回所需的类型。通过将`@Component`注解应用于这样的接口并将模块类型传递给`modules`参数，`Dagger2`将完全生成该合同的实现。

```java
@Component(modules = DripCoffeeModule.class)
interface CoffeeShop {
  CoffeeMaker maker();
}
```

该实现名字是接口名称加上`Dagger`前缀。通过调用该实现的`builder()`方法获取实例，并使用返回的构建器设置依赖关系并构建一个新的实例。

```java
CoffeeShop coffeeShop = DaggerCoffeeShop.builder()
    .dripCoffeeModule(new DripCoffeeModule())
    .build();
```

注意：如果您的`@Component`不是顶级类型，生成的组件的名称将包含其封闭类型的名称，并加上下划线。例如，这段代码：

```java
class Foo {
  static class Bar {
    @Component
    interface BazComponent {}
  }
}
```
会生成一个名为`DaggerFoo_Bar_BazComponent`的组件。

任何具有可访问默认构造函数的模块都可以省略，因为如果没有设置，构建器将自动构造一个实例。对于任何其`@Provides`方法都是静态的模块，实现根本不需要实例。如果所有的依赖都可以在用户创建依赖实例的情况下构建，那么生成的实现也会有一个create()方法，可用于获取新实例而无需处理构建器。

```java
CoffeeShop coffeeShop = DaggerCoffeeShop.create();
```

现在，我们的CoffeeApp可以使用Dagger生成的CoffeeShop实现来获得完全注入的CoffeeMaker。

```java
public class CoffeeApp {
  public static void main(String[] args) {
    CoffeeShop coffeeShop = DaggerCoffeeShop.create();
    coffeeShop.maker().brew();
  }
}
```

现在图形被构建并且入口点被注入，我们运行我们的咖啡机应用程序。

```java
$ java -cp ... coffee.CoffeeApp
~ ~ ~ heating ~ ~ ~
=> => pumping => =>
 [_]P coffee! [_]P
```

### 图中的绑定

上面的例子显示了如何用一些更典型的绑定来构建一个组件，但是有多种机制可以为图形提供绑定。以下是依赖关系，可用于生成格式良好的组件：

* 那些@Module中由`@Provides`声明的方法直接由@Component.modules引用或者通过@Module.includes传递
* 任何带有@Inject构造函数的类型，它都是unscoped或具有与组件范围之一相匹配的@Scope注释
* 组件依赖关系的组件提供方法
* 组件本身
* 任何包含的子组件均为不合格的建设者
* 提供者或惰性包装的任何上述绑定
* 任何上述绑定的提供者（例如，`Provider<Lazy<CoffeeMaker>>`）
* 任何类型的MembersInjector

## 单例和范围绑定

使用`@Singleton`注释一个`@Provides`方法或注入类。 该图将为所有客户端使用该值的单个实例。

```java
@Provides @Singleton static Heater provideHeater() {
  return new ElectricHeater();
}
```

注射类上的`@Singleton`注解也可以作为文档。 它提醒潜在的维护者，这个类可以被多个线程共享。

```java
@Singleton
class CoffeeMaker {
  ...
}
```


由于`Dagger2`将图中的范围实例与组件实现的实例相关联，所以组件本身需要声明它们想要表示的范围。 例如，在同一个组件中拥有`@Singleton`绑定和`@RequestScoped`绑定没有任何意义，因为这些范围具有不同的生命周期，因此必须生活在具有不同生命周期的组件中。要声明组件与给定范围关联，只需将范围注释应用到组件接口即可。

```java
@Component(modules = DripCoffeeModule.class)
@Singleton
interface CoffeeShop {
  CoffeeMaker maker();
}
```

组件可能会应用多个范围注释。 这声明它们都是相同范围的别名，并且组件可以包含它声明的任何范围的范围绑定。

## 复用范围

有时候，你想限制一个`@Inject`构造类被实例化或者`@Provides`方法被调用的次数，但是你不需要保证在任何特定组件或子组件的生命周期中使用完全相同的实例。 这对于像Android这样的分配可能很昂贵的环境很有用。



对于这些绑定，您可以应用`@Reusable`范围。其他范围不同，可重用范围的绑定不与任何单个组件关联;相反，实际使用绑定的每个组件都会缓存返回的或实例化的对象。


这意味着如果您在组件中安装带有`@Reusable`绑定的模块，但只有一个子组件实际使用绑定，那么只有该子组件才会缓存绑定的对象。如果不共享祖先的两个子组件都使用绑定，则它们中的每一个都将缓存它自己的对象。 如果组件的祖先已经缓存了该对象，则该子组件将重新使用它。

不能保证组件只会调用一次绑定，因此将@Reusable应用于返回可变对象的绑定或引用同一实例的重要对象是危险的。使用@Reusable作为不可变对象是很安全的，如果你不关心它们被分配了多少次，那么你就不会放大。

```java
@Reusable // It doesn't matter how many scoopers we use, but don't waste them.
class CoffeeScooper {
  @Inject CoffeeScooper() {}
}

@Module
class CashRegisterModule {
  @Provides
  @Reusable // DON'T DO THIS! You do care which register you put your cash in.
            // Use a specific scope instead.
  static CashRegister badIdeaCashRegister() {
    return new CashRegister();
  }
}

@Reusable // DON'T DO THIS! You really do want a new filter each time, so this
          // should be unscoped.
class CoffeeFilter {
  @Inject CoffeeFilter() {}
}
```

##  可释放的引用

当绑定使用范围注释时，这意味着组件对象持有对绑定对象的引用，直到组件对象本身被垃圾收集为止。在Android等内存敏感的环境中，当应用程序处于内存压力下时，您可能希望在垃圾回收期间删除当前未使用的范围对象。

在这种情况下，您可以定义一个范围并使用@CanReleaseReferences对其进行注释。

```java
@Documented
@Retention(RUNTIME)
@CanReleaseReferences
@Scope
public @interface MyScope {}
```

如果您确定要允许在垃圾回收期间保留在该范围内的对象（如果它们当前未被某个其他对象使用），则可以为您的范围注入一个`ReleasableReferenceManager`对象并在其上调用`releaseStrongReferences()`，它将使该组件对该对象持有一个`WeakReference`，而不是一个强引用：

```java
@Inject @ForReleasableReferences(MyScope.class)
ReleasableReferenceManager myScopeReferenceManager;

void lowMemory() {
  myScopeReferenceManager.releaseStrongReferences();
}
```

如果确定内存压力已降低，则可以通过调用`restoreStrongReferences()`来恢复在垃圾回收期间尚未删除的任何缓存对象的强引用：

```java
void highMemory() {
  myScopeReferenceManager.restoreStrongReferences();
}
```

## 延迟注入

有时你需要一个对象懒惰地实例化。 对于任何绑定T，您可以创建一个`Lazy <T>`，它延迟实例化，直到首次调用`Lazy <T>`的`get()`方法。 如果T是一个单例，那么Lazy <T>将成为ObjectGraph内所有注入的相同实例。 否则，每个注入站点将获得它自己的Lazy <T>实例。 无论如何，对Lazy <T>的任何给定实例的后续调用将返回相同的T的底层实例。

```java
class GrindingCoffeeMaker {
  @Inject Lazy<Grinder> lazyGrinder;

  public void brew() {
    while (needsGrinding()) {
      // Grinder created once on first call to .get() and cached.
      lazyGrinder.get().grind();
    }
  }
}
```

## 提供注入

有时你需要返回多个实例，而不是只注入一个值。 虽然有几个选项（工厂，构建器等），但一种选择是注入一个`Provider<T>`而不是T。一个`Provider<T>`在每次调用`.get()`时调用T的绑定逻辑。 如果该绑定逻辑是@Inject构造函数，则会创建一个新实例，但@Provides方法没有这种保证。

```java
class BigCoffeeMaker {
  @Inject Provider<Filter> filterProvider;

  public void brew(int numberOfPots) {
  ...
    for (int p = 0; p < numberOfPots; p++) {
      maker.addFilter(filterProvider.get()); //new filter every time.
      maker.addCoffee(...);
      maker.percolate();
      ...
    }
  }
}
```



注意：注入`Provider <T>`可能会产生混淆的代码，并且可能是图形中存在错误或错误结构对象的设计气味。 通常你会想要使用工厂或Lazy <T>或者重新组织代码的生命周期和结构，以便能够注入T.注入提供者<T>可以在某些情况下成为救命。 当您必须使用与您的对象的自然生命周期不一致的遗留体系结构时（例如，servlet是按设计单身，但仅在请求特定数据的上下文中有效），常见用法是。

## 限定符

有时仅仅这种类型不足以识别依赖性。 例如，一个复杂的咖啡机应用程序可能需要为水和热板分开加热器。

在这种情况下，我们添加一个限定符注释。 这是任何注释本身有一个@Qualifier注释。以下是@Named的声明，它是javax.inject中包含的限定符注释：

```java
@Qualifier
@Documented
@Retention(RUNTIME)
public @interface Named {
  String value() default "";
}
```



您可以创建自己的限定符注释，或者仅使用`@Named`。 通过注释感兴趣的字段或参数来应用限定符。 类型和限定符注释都将用于标识依赖关系。

```java
class ExpensiveCoffeeMaker {
  @Inject @Named("water") Heater waterHeater;
  @Inject @Named("hot plate") Heater hotPlateHeater;
  ...
}
```

通过注释相应的`@Provides`方法来提供合格的值。

```java
@Provides @Named("hot plate") static Heater provideHotPlateHeater() {
  return new ElectricHeater(70);
}

@Provides @Named("water") static Heater provideWaterHeater() {
  return new ElectricHeater(93);
}
```

依赖项可能没有多个限定符注释。

## 可选绑定

```java
@BindsOptionalOf abstract CoffeeCozy optionalCozy();
```

如果你想让绑定工作，即使组件中没有绑定某个依赖关系，也可以在模块中添加一个@BindsOptionalOf方法：

这意味着@Inject构造函数和成员和@Provides方法可以依赖于一个可选的<CoffeeCozy>对象。 如果组件中存在CoffeeCozy的绑定，则可选将存在; 如果CoffeeCozy没有绑定，则可选将不存在。

具体而言，您可以注入以下任何一项：

* `Optional<CoffeeCozy> `(unless there is a @Nullable binding for CoffeeCozy; see below)
* `Optional<Provider<CoffeeCozy>>`
* `Optional<Lazy<CoffeeCozy>>`
* `Optional<Provider<Lazy<CoffeeCozy>>>`

（你也可以注入一个提供者或懒惰或者任何这些懒惰的提供者，但这不是非常有用。）

如果CoffeeCozy有一个绑定，并且该绑定是@Nullable，那么注入可选<CoffeeCozy>是一个编译时错误，因为Optional不能包含null。 您可以随时注入其他表单，因为Provider和Lazy总是可以从get()方法返回null。

如果子组件包含对基础类型的绑定，则可以在子组件中存在一个组件中不存在的可选绑定。

您可以使用`Guava`的`Optional`或Java 8的Optional。

## 绑定实例

通常，您在构建组件时可以使用数据。例如，假设你有一个使用命令行参数的应用程序;您可能希望在组件中绑定这些参数。

也许您的应用程序需要一个参数，表示您要以@UserName String注入的用户名。您可以向组件构建器添加方法注解[@BindsInstance](https://google.github.io/dagger/api/latest/dagger/BindsInstance.html)，以允许将该实例注入到组件中。

```java
@Component(modules = AppModule.class)
interface AppComponent {
  App app();

  @Component.Builder
  interface Builder {
    @BindsInstance Builder userName(@UserName String userName);
    AppComponent build();
  }
}
```

你的应用程序可能看起来像

```java
public static void main(String[] args) {
  if (args.length > 1) { exit(1); }
  App app = DaggerAppComponent
      .builder()
      .userName(args[0])
      .build()
      .app();
  app.run();
}
```

在上面的示例中，在组件中注入`@UserName`字符串时将使用调用此方法时提供给构建器的实例。 在构建组件之前，必须调用所有@BindsInstance方法，并传递一个非空值（除了下面的@Nullable绑定外）。

如果@BindsInstance方法的参数被标记为@Nullable，那么绑定将被认为是“可空的”，就像@Provides方法是可空的那样：注入站点也必须将其标记为@Nullable，并且null是可接受的值 绑定。 而且，Builder的用户可能会省略调用该方法，并且该组件会将该实例视为null。

`@BindsInstance`方法应该优先于用构造函数参数编写@Module并立即提供这些值。

## 编译时检查

`Dagger`注释处理器是严格的，如果任何绑定无效或不完整，将导致编译器错误。 例如，该模块安装在缺少Executor绑定的组件中：

```java
@Module
class DripCoffeeModule {
  @Provides static Heater provideHeater(Executor executor) {
    return new CpuHeater(executor);
  }
}
```
编译时，javac会拒绝缺少的绑定：

```java
[ERROR] COMPILATION ERROR :
[ERROR] error: java.util.concurrent.Executor cannot be provided without an @Provides-annotated method.
```


通过为Executor添加一个@ Provide-annotated方法来修复组件中的任何模块。虽然@Inject，@Module和@Provides注释是单独验证的，但绑定之间关系的所有验证都发生在@Component级别。Dagger1严格依赖于@模块级验证（可能或可能没有反映运行时行为），但Dagger2不支持这种验证（以及@Module上的配置参数），以支持完整的图验证。

## 编译时代码生成

Dagger的注释处理器也可以生成带有CoffeeMaker_Factory.java或CoffeeMaker_MembersInjector.java等名称的源文件。 这些文件是Dagger实现细节。 您不需要直接使用它们，但通过注入进行分步调试时，它们可能非常方便。 您应该在您的代码中引用的唯一生成的类型是为您的组件添加了Dagger的前缀。

##  在你的构建中使用Dagger

您需要在应用程序的运行时包含dagger-2.X.jar。 为了激活代码生成，您需要在编译时在您的构建中包含dagger-compiler-2.X.jar。 请参阅自述文件以获取更多信息。

|      |      |
| ---- | ---- |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |
|      |      |



# 参考

* 

  

