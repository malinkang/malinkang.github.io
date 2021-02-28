---
title: LiveData概述
date: 2018-05-18 16:47:47
toc: true
tags:
draft: flase
---

`LiveData`是一个可观察的数据持有者类。与常规可观察性不同，`LiveData`具有生命周期感知能力，这意味着它遵守其他应用程序组件（例如`activity`，`fragment`或`service`）的生命周期。 这确保`LiveData`只更新处于活动生命周期状态的应用程序组件观察者。

<!--more-->

`LiveData`作为一个用`Observer`类表示观察者，如果它的生命周期处于`STARTED`或`RESUMED`状态，则处于活动状态。`LiveData`仅通知活动状态的观察者更新。 非活动观察者注册观察`LiveData`对象不会收到有关更改的通知。

您可以注册与实现`LifecycleOwner`接口的对象配对的观察者。此关系允许在相应生命周期对象的状态更改为DESTROYED时删除观察者。这对于`activity`和`fragment`尤其有用，因为它们可以安全地观察`LiveData`对象，而不必担心泄漏 - 在其生命周期被销毁时，`activity`和`fragment`会立即取消订阅。

### 使用LiveData的优点

使用LiveData提供以下优点：

#### 确保您的用户界面匹配您的数据状态

`LiveData`遵循观察者模式。当生命周期状态改变时，`LiveData`会通知观察者的对象。你可以合并你的代码以更新这些观察者对象中的UI。不是每一次数据改变时都更新UI，而是每次发生变化时，您的观察者可以更新UI。


#### 没有内存泄漏

观察者绑定到生命周期对象，并在其相关生命周期被摧毁后自行清理。

#### 由于停止活动而没有崩溃


如果观察者的生命周期处于非活动状态，例如在后退堆栈中的活动，则它不会收到任何LiveData事件。


#### 没有更多的手动生命周期处理

UI组件只是观察相关数据，不要停止或恢复观察。 `LiveData`自动管理所有这些，因为它在观察时感知到相关的生命周期状态更改。


#### 始终保持最新的数据


如果生命周期变为非活动状态，它将在再次变为活动状态时收到最新数据。例如，后台活动在返回到前台后立即收到最新数据。


#### 合适的配置更改


如果由于配置更改（如设备旋转）而重新创建`activity`或`fragment`，它会立即收到最新的可用数据。

#### 共享资源

您可以使用单例模式扩展`LiveData`对象以包装系统服务，以便它们可以在应用程序中共享。`LiveData`对象连接到系统服务一次，然后任何需要该资源的观察者都可以观看`LiveData`对象。


### 使用LiveData对象

按照以下步骤使用`LiveData`对象。

1.创建一个`LiveData`实例来保存某种类型的数据。这通常在您的`ViewModel`类中完成。

2.创建一个`Observer`对象，该对象定义`onChanged()`方法，该方法控制`LiveData`对象保存的数据更改时发生的情况。您通常在UI控制器中创建`Observer`对象，如`activity`或`fragment`。

3.使用`observe()`方法将`Observer`对象附加到`LiveData`对象。`observe()`方法使用`LifecycleOwner`对象。这将`Observer`对象订阅到`LiveData`对象，以便通知其更改。您通常将`Observer`对象附加到UI控制器中，如`activity`或`fragment`。


当您更新存储在`LiveData`对象中的值时，只要附加的`LifecycleOwner`处于活动状态，它就会触发所有已注册的观察者。

`LiveData`允许UI控制器观察者订阅更新。当`LiveData`对象持有的数据更改时，UI会自动更新以作为响应。


#### 创建LiveData对象


`LiveData`是一个可用于任何数据的包装器，包括实现集合的对象（如List）。`LiveData`对象通常存储在`ViewModel`对象中，并通过`getter`方法访问，如以下示例所示：

```java
public class NameViewModel extends ViewModel {

// Create a LiveData with a String
	private MutableLiveData<String> mCurrentName;

	public MutableLiveData<String> getCurrentName() {
	    if (mCurrentName == null) {
	        mCurrentName = new MutableLiveData<String>();
	    }
	    return mCurrentName;
	}

// Rest of the ViewModel...
}
```

最初，LiveData对象中的数据未设置。


#### 观察LiveData对象

在大多数情况下，出于以下原因，应用程序组件的`onCreate()`方法是开始观察`LiveData`对象的正确位置：

确保系统不会从`activity`或`fragment`的`onResume()`方法进行多余的调用。


确保`activity`或`fragment`具有一旦它变为活动状态即可显示的数据。只要应用程序组件处于`STARTED`状态，它就会从它所观察的`LiveData`对象中接收最新的值。这仅在要设置要观察的`LiveData`对象时才会发生。

通常，`LiveData`仅在数据更改时传递更新，并且仅传递给活动观察者。此行为的一个例外是，观察者在从非活动状态变为活动状态时也会收到更新。此外，如果观察者第二次从非激活状态变为激活状态，则只有在自上一次变为活动状态以来该值发生变化时才会收到更新。


以下示例代码说明了如何开始观察`LiveData`对象：

```java
public class NameActivity extends AppCompatActivity {

    private NameViewModel mModel;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        // Other code to setup the activity...

        // Get the ViewModel.
        mModel = ViewModelProviders.of(this).get(NameViewModel.class);


        // Create the observer which updates the UI.
        final Observer<String> nameObserver = new Observer<String>() {
            @Override
            public void onChanged(@Nullable final String newName) {
                // Update the UI, in this case, a TextView.
                mNameTextView.setText(newName);
            }
        };

        // Observe the LiveData, passing in this activity as the LifecycleOwner and the observer.
        mModel.getCurrentName().observe(this, nameObserver);
    }
}
```


使用`nameObserver`作为参数传递`observe()`后，立即调用`onChanged()`，以提供存储在`mCurrentName`中的最新值。如果`LiveData`对象未在`mCurrentName`中设置值，则不调用`onChanged()`。

#### 更新LiveData对象

`LiveData`没有公开可用的方法来更新存储的数据。 MutableLiveData类公开`setValue(T)`和`postValue(T)`方法，如果需要编辑存储在`LiveData`对象中的值，则必须使用这些方法。通常在`ViewModel`中使用`MutableLiveData`，然后`ViewModel`只向观察者公开不可变的`LiveData`对象。

在建立观察者关系后，可以更新`LiveData`对象的值，如以下示例所示，当用户点击按钮时触发所有观察者：

```java
mButton.setOnClickListener(new OnClickListener() {
    @Override
    public void onClick(View v) {
        String anotherName = "John Doe";
        mModel.getCurrentName().setValue(anotherName);
    }
});
```


在示例中调用`setValue(T)`会导致观察者用值`John Doe`调用它们的`onChanged()`方法。该示例显示按钮按下，但`setValue()`或`postValue()`可能因多种原因被调用来更新`mName`，包括响应网络请求或数据库加载完成;在所有情况下，调用`setValue()`或`postValue()`都会触发观察者并更新UI。

#### 和Room一起使用LiveData

`Room`持久性库支持返回`LiveData`对象的可观察查询。可观察查询是作为数据库访问对象（DAO）的一部分写入的。

当更新数据库时，会生成所有必要的代码以更新`LiveData`对象。生成的代码在需要时在后台线程上异步运行查询。这种模式对于保持UI中显示的数据与存储在数据库中的数据保持同步很有用。您可以在`Room`持久库指南中阅读关于`Room`和`DAO`的更多信息。


### 扩展LiveData


如果观察者的生命周期处于`STARTED`或`RESUMED`状态，则`LiveData`将认为观察者处于活动状态。以下示例代码说明了如何扩展`LiveData`类：

```java
public class StockLiveData extends LiveData<BigDecimal> {
    private StockManager mStockManager;

    private SimplePriceListener mListener = new SimplePriceListener() {
        @Override
        public void onPriceChanged(BigDecimal price) {
            setValue(price);
        }
    };

    public StockLiveData(String symbol) {
        mStockManager = new StockManager(symbol);
    }

    @Override
    protected void onActive() {
        mStockManager.requestPriceUpdates(mListener);
    }

    @Override
    protected void onInactive() {
        mStockManager.removeUpdates(mListener);
    }
}
```

本示例中的价格监听器的实现包括以下重要方法：


当`LiveData`对象具有活动观察者时，将调用`onActive()`方法。这意味着您需要开始观察此方法的股价更新。

当`LiveData`对象没有任何活动观察者时，将调用`onInactive()`方法。由于没有观察员在监听，因此没有理由保持连接到`StockManager`服务。

`setValue(T)`方法更新`LiveData`实例的值并通知任何活动观察者有关更改。


您可以使用`StockLiveData`类如下所示：

```java
public class MyFragment extends Fragment {
    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        LiveData<BigDecimal> myPriceListener = ...;
        myPriceListener.observe(this, price -> {
            // Update the UI.
        });
    }
}
```

`observe()`方法将作为LifecycleOwner`实例的`fragment`作为第一个参数传递。这样做表示此观察者被绑定到与所有者关联的生命周期对象，这意味着：


如果生命周期对象不处于活动状态，则即使值发生更改，也不会调用观察者。

生命周期对象被销毁后，观察者被自动删除。

LiveData对象支持生命周期意味着您可以在多个`activity`，`fragment`和`service`之间共享它们。为了保持示例简单，您可以按如下方式将`LiveData`类实现为单例：


```java
public class StockLiveData extends LiveData<BigDecimal> {
    private static StockLiveData sInstance;
    private StockManager mStockManager;

    private SimplePriceListener mListener = new SimplePriceListener() {
        @Override
        public void onPriceChanged(BigDecimal price) {
            setValue(price);
        }
    };

    @MainThread
    public static StockLiveData get(String symbol) {
        if (sInstance == null) {
            sInstance = new StockLiveData(symbol);
        }
        return sInstance;
    }

    private StockLiveData(String symbol) {
        mStockManager = new StockManager(symbol);
    }

    @Override
    protected void onActive() {
        mStockManager.requestPriceUpdates(mListener);
    }

    @Override
    protected void onInactive() {
        mStockManager.removeUpdates(mListener);
    }
}
```
你可以在片段中使用它，如下所示：


```java
public class MyFragment extends Fragment {
    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        StockLiveData.get(getActivity()).observe(this, price -> {
            // Update the UI.
        });
    }
}
```

多个`fragment`和`activity`可以观察`MyPriceListener`实例。如果`LiveData`只有一个或多个可见并处于活动状态，则`LiveData`仅连接到系统服务。


### 转换LiveData

您可能希望在存储在`LiveData`对象中的值被更改为观察者之前，或者您可能需要基于另一个`LiveData`实例的值返回不同的`LiveData`实例。 `Lifecycle`软件包提供`Transformations`类，其中包括支持这些场景的帮助程序方法。

`Transformations.map()`

在存储在`LiveData`对象中的值上应用函数，并向下游传播结果。

```java
LiveData<User> userLiveData = ...;
LiveData<String> userName = Transformations.map(userLiveData, user -> {
    user.name + " " + user.lastName
});
```

`Transformations.switchMap()`


与`map()`类似，将函数应用于存储在`LiveData`对象中的值，并展开并向下游分派结果。传递给`switchMap()`的函数必须返回一个`LiveData`对象，如下例所示：

```java
private LiveData<User> getUser(String id) {
  ...;
}

LiveData<String> userId = ...;
LiveData<User> user = Transformations.switchMap(userId, id -> getUser(id) );
```


您可以使用转换方法在观察者的生命周期中传递信息。除非观察者正在观看返回的`LiveData`对象，否则不会计算转换。由于转换是懒散计算的，因此与生命周期相关的行为会隐式传递，而不需要额外的显式调用或依赖关系。

如果您认为您需要`ViewModel`对象中的`Lifecycle`对象，则转换可能是更好的解决方案。例如，假设您有一个接受地址并返回该地址的邮政编码的UI组件。您可以为此组件实现朴素的ViewModel，如以下示例代码所示：

```java
class MyViewModel extends ViewModel {
    private final PostalCodeRepository repository;
    public MyViewModel(PostalCodeRepository repository) {
       this.repository = repository;
    }

    private LiveData<String> getPostalCode(String address) {
       // DON'T DO THIS
       return repository.getPostCode(address);
    }
}
```

UI组件随后需要从以前的`LiveData`对象注销，并在每次调用`getPostalCode()`时注册到新实例。另外，如果UI组件被重新创建，它会触发对`repository.getPostCode()`方法的另一次调用，而不是使用先前调用的结果。


相反，您可以将邮政编码查找实现为地址输入的转换，如以下示例所示：

```java
class MyViewModel extends ViewModel {
    private final PostalCodeRepository repository;
    private final MutableLiveData<String> addressInput = new MutableLiveData();
    public final LiveData<String> postalCode =
            Transformations.switchMap(addressInput, (address) -> {
                return repository.getPostCode(address);
             });

  public MyViewModel(PostalCodeRepository repository) {
      this.repository = repository
  }

  private void setInput(String address) {
      addressInput.setValue(address);
  }
}
```

在这种情况下，邮政编码字段用`public`和`final`修饰，因为该字段永远不会改变。`postalCode`字段定义为`addressInput`的转换，这意味着`addressInput`发生更改时将调用`repository.getPostCode()`方法。如果存在活动观察者，那么这是真实的，如果在`repository.getPostCode()`被调用时没有活动观察者，则在添加观察者之前不进行计算。

该机制允许较低级别的应用程序创建按需延迟计算的`LiveData`对象。`ViewModel`对象可以轻松获得对`LiveData`对象的引用，然后在其上定义转换规则。

#### 创建新的transformations

有十几种不同的特定转换可能在您的应用中很有用，但它们不是默认提供的。要实现自己的转换，您可以使用`MediatorLiveData`类，它监听其他`LiveData`对象并处理它们发出的事件。 `MediatorLiveData`将其状态正确传播到源`LiveData`对象。要了解有关此模式的更多信息，请参阅`Transformations`类的参考文档。


### 合并多个LiveData源

`MediatorLiveData`是`LiveData`的一个子类，允许您合并多个`LiveData`源。 `MediatorLiveData`对象的观察者随后会在任何原始`LiveData`源对象更改时触发。


例如，如果您的UI中有一个可从本地数据库或网络更新的`LiveData`对象，则可以将以下源添加到`MediatorLiveData`对象：

与存储在数据库中的数据关联的`LiveData`对象。

与从网络访问的数据关联的`LiveData`对象。


您的`activity`只需观察`MediatorLiveData`对象即可从两个来源接收更新。有关详细示例，请参阅应用程序体系结构指南的附录：展示网络状态部分。





