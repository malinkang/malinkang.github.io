---
title: Paging 库概述
date: 2018-07-27 14:44:49
tags:
toc: true
---

分页库可帮助您一次加载和显示小块数据。按需加载部分数据可减少网络带宽和系统资源的使用。

<!--more-->

在应用程序或模块的`build.gradle`文件中添加所需的依赖项：

```groovy
dependencies {
    def paging_version = "2.1.0"

    implementation "androidx.paging:paging-runtime:$paging_version" // For Kotlin use paging-runtime-ktx

    // alternatively - without Android dependencies for testing
    testImplementation "androidx.paging:paging-common:$paging_version" // For Kotlin use paging-common-ktx

    // optional - RxJava support
    implementation "androidx.paging:paging-rxjava2:$paging_version" // For Kotlin use paging-rxjava2-ktx
}
```

# 库架构

## PagedList

分页库的关键组件是`PagedList`类，它加载应用程序数据或页面的块。由于需要更多数据，因此将其分页到现有的`PagedList`对象中。如果任何加载的数据发生更改，则会从`LiveData`或基于`RxJava2`的对象向可观察数据持有者发出新的`PagedList`实例。生成`PagedList`对象时，应用程序的UI会显示其内容，同时遵循UI控制器的生命周期。

以下代码段显示了如何使用`PagedList`对象的`LiveData`持有者配置应用程序的视图模型以加载和显示数据：

```java
public class ConcertViewModel extends ViewModel {
    private ConcertDao concertDao;
    public final LiveData<PagedList<Concert>> concertList;

    // Creates a PagedList object with 50 items per page.
    public ConcertViewModel(ConcertDao concertDao) {
        this.concertDao = concertDao;
        concertList = new LivePagedListBuilder<>(
                concertDao.concertsByDate(), 50).build();
    }
}
```



## 数据

PagedList的每个实例都从其DataSource加载应用程序数据的最新快照。数据从应用程序的后端或数据库流入PagedList对象。

以下示例使用Room持久性库来组织应用程序的数据，但如果要使用其他方法存储数据，还可以提供自己的数据源工厂。

```java
@Dao
public interface ConcertDao {
    // The Integer type parameter tells Room to use a
    // PositionalDataSource object.
    @Query("SELECT * FROM concerts ORDER BY date DESC")
    DataSource.Factory<Integer, Concert> concertsByDate();
}
```



## 界面

PagedList类与PagedListAdapter一起使用，可以将项目加载到RecyclerView中。这些类一起工作以在加载内容时获取和显示内容，预取视图外内容并动画内容更改。


分页库实现了应用程序体系结构指南中建议的观察者模式。特别是，库的核心组件创建了一个UI可以观察的LiveData <PagedList>（或等效的基于RxJava2的类）的实例。然后，您的应用程序的UI可以在生成PagedList对象时显示内容，同时尊重UI控制器的生命周期。

# 支持不同的数据架构



图1显示了每种架构方案中数据的流动方式。对于仅限网络或仅限数据库的解决方案，数据直接流向应用程序的UI模型。如果您使用的是组合方法，则数据会从后端服务器流入设备上的数据库，然后流入应用程序的UI模型。每隔一段时间，每个数据流的端点就会耗尽要加载的数据，此时它会从提供数据的组件请求更多数据。例如，当设备上数据库用完数据时，它会从服务器请求更多数据。

![](../assets/paging-library-data-flow.png)


我们提供了用于不同数据架构的推荐模式的示例。要查看它们，请参阅GitHub上的PagingWithNetwork示例。

## Network only

要显示来自后端服务器的数据，请使用Retrofit API的同步版本将信息加载到您自己的自定义DataSource对象中。

## Database only

设置RecyclerView以观察本地存储，最好使用Room persistence library。这样，无论何时在应用程序的数据库中插入或修改数据，这些更改都会自动反映在显示此数据的RecyclerView中。

## Network and database


在开始观察数据库之后，可以使用PagedList.BoundaryCallback监听数据库何时没有数据。然后，您可以从网络中获取更多项目并将其插入数据库。如果您的UI正在观察数据库，那就是您需要做的。

# 处理网络错误

当使用网络获取或分页您正在使用分页库显示的数据时，重要的是不要将网络视为“可用”或“不可用”，因为许多连接是间歇性的或片状的：

* 特定服务器可能无法响应网络请求。

* 设备可能连接到缓慢或弱的网络。


相反，您的应用应检查每个失败请求，并在网络不可用的情况下尽可能优雅地恢复。例如，您可以提供“重试”按钮，供用户选择数据刷新步骤是否不起作用。如果在数据分页步骤期间发生错误，则最好自动重试分页请求。

# 更新已经存在的app

如果您的应用已经消耗了数据库或后端源中的数据，则可以直接升级到分页库提供的功能。本节介绍如何升级具有通用现有设计的应用程序。

## 定制分页解决方案

如果使用自定义功能从应用程序的数据源加载小的数据子集，则可以将此逻辑替换为PagedList类中的逻辑。 PagedList的实例提供与公共数据源的内置连接。这些实例还为您可能包含在应用程序UI中的RecyclerView对象提供适配器。

## 使用列表而不是页面加载数据

如果使用内存列表作为UI适配器的后备数据结构，请考虑如果列表中的项目数可能变大，则使用PagedList类观察数据更新。 `PagedList`的实例可以使用`LiveData <PagedList>`或`Observable <List>`将数据更新传递到应用程序的UI，从而最大限度地减少加载时间和内存使用量。更好的是，在应用程序中用`PagedList`对象替换`List`对象不需要对应用程序的UI结构或数据更新逻辑进行任何更改。

## 使用CursorAdapter将数据光标与列表视图相关联


您的应用程序可能使用CursorAdapter将Cursor中的数据与ListView相关联。在这种情况下，您通常需要从ListView迁移到RecyclerView作为应用程序的列表UI容器，然后将Cursor组件替换为Room或PositionalDataSource，具体取决于Cursor实例是否访问SQLite数据库。

在某些情况下，例如在处理Spinner实例时，您只提供适配器本身。然后，库将获取加载到该适配器中的数据并为您显示数据。在这些情况下，将适配器数据的类型更改为LiveData <PagedList>，然后在尝试让库类在UI中对这些项进行`inflate`之前，将此列表包装在ArrayAdapter对象中。

## 使用AsyncListUtil异步加载内容


如果您使用AyncListUtil对象异步加载和显示信息组，则分页库可让您更轻松地加载数据：

您的数据不需要是位置的。分页库允许您使用网络提供的密钥直接从后端加载数据。

您的数据可能非常大。使用分页库，您可以将数据加载到页面中，直到没有剩余数据。

您可以更轻松地观察数据。 Paging库可以显示应用程序的ViewModel在可观察数据结构中保存的数据。

# 数据库示例

以下代码片段显示了将所有部分协同工作的几种可能方法。

## 使用LiveData观察分页数据

以下代码段显示了所有协同工作。随着在数据库中添加，删除或更改音乐会事件，RecyclerView中的内容将自动且有效地更新：

```java
@Dao
public interface ConcertDao {
    // The Integer type parameter tells Room to use a PositionalDataSource
    // object, with position-based loading under the hood.
    @Query("SELECT * FROM concerts ORDER BY date DESC")
    DataSource.Factory<Integer, Concert> concertsByDate();
}

public class ConcertViewModel extends ViewModel {
    private ConcertDao mConcertDao;
    public final LiveData<PagedList<Concert>> concertList;

    public ConcertViewModel(ConcertDao concertDao) {
        mConcertDao = concertDao;
    }

    concertList = new LivePagedListBuilder<>(
            mConcertDao.concertsByDate(), /* page size */ 20).build();
}

public class ConcertActivity extends AppCompatActivity {
    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ConcertViewModel viewModel =
                ViewModelProviders.of(this).get(ConcertViewModel.class);
        RecyclerView recyclerView = findViewById(R.id.concert_list);
        ConcertAdapter adapter = new ConcertAdapter();
        viewModel.concertList.observe(this, adapter::submitList);
        recyclerView.setAdapter(adapter);
    }
}

public class ConcertAdapter
        extends PagedListAdapter<Concert, ConcertViewHolder> {
    protected ConcertAdapter() {
        super(DIFF_CALLBACK);
    }

    @Override
    public void onBindViewHolder(@NonNull ConcertViewHolder holder,
            int position) {
        Concert concert = getItem(position);
        if (concert != null) {
            holder.bindTo(concert);
        } else {
            // Null defines a placeholder item - PagedListAdapter automatically
            // invalidates this row when the actual object is loaded from the
            // database.
            holder.clear();
        }
    }

    private static DiffUtil.ItemCallback<Concert> DIFF_CALLBACK =
            new DiffUtil.ItemCallback<Concert>() {
        // Concert details may have changed if reloaded from the database,
        // but ID is fixed.
        @Override
        public boolean areItemsTheSame(Concert oldConcert, Concert newConcert) {
            return oldConcert.getId() == newConcert.getId();
        }

        @Override
        public boolean areContentsTheSame(Concert oldConcert,
                Concert newConcert) {
            return oldConcert.equals(newConcert);
        }
    };
}
```

## 使用RxJava2观察分页数据

如果您更喜欢使用RxJava2而不是LiveData，则可以创建一个Observable或Flowable对象：

```java
public class ConcertViewModel extends ViewModel {
    private ConcertDao mConcertDao;
    public final Flowable<PagedList<Concert>> concertList;

    public ConcertViewModel(ConcertDao concertDao) {
        mConcertDao = concertDao;

        concertList = new RxPagedListBuilder<>(
                mConcertDao.concertsByDate(), /* page size */ 50)
                        .buildFlowable(BackpressureStrategy.LATEST);
    }
}

```

然后，您可以使用以下代码段中的代码开始和停止观察数据：

```java
public class ConcertActivity extends AppCompatActivity {
    private ConcertAdapter mAdapter;
    private ConcertViewModel mViewModel;

    private CompositeDisposable mDisposable = new CompositeDisposable();

    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        RecyclerView recyclerView = findViewById(R.id.concert_list);

        mViewModel = ViewModelProviders.of(this).get(ConcertViewModel.class);
        mAdapter = new ConcertAdapter();
        recyclerView.setAdapter(mAdapter);
    }

    @Override
    protected void onStart() {
        super.onStart();
        mDisposable.add(mViewModel.concertList.subscribe(
                flowableList -> mAdapter.submitList(flowableList)
        ));
    }

    @Override
    protected void onStop() {
        super.onStop();
        mDisposable.clear();
    }
}
```

对于基于RxJava2的解决方案，ConcertDao和ConcertAdapter的代码与基于LiveData的解决方案的代码相同。



# 展示分页数据

## 将UI连接到ViewModel

您可以将`LiveData <PagedList>`的实例连接到`PagedListAdapter`，如以下代码段所示：

```java
public class ConcertActivity extends AppCompatActivity {
    private ConcertAdapter adapter = new ConcertAdapter();
    private ConcertViewModel viewModel;

    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        viewModel = ViewModelProviders.of(this).get(ConcertViewModel.class);
        viewModel.concertList.observe(this, adapter::submitList);
    }
}
```

当数据源提供PagedList的新实例时，`activity`会将这些对象发送到适配器。 PagedListAdapter实现定义了如何计算更新，并自动处理分页和列表差异。因此，您的ViewHolder只需要绑定到特定提供的`item`：

```java
public class ConcertAdapter
        extends PagedListAdapter<Concert, ConcertViewHolder> {
    protected ConcertAdapter() {
        super(DIFF_CALLBACK);
    }

    @Override
    public void onBindViewHolder(@NonNull ConcertViewHolder holder,
            int position) {
        Concert concert = getItem(position);

        // Note that "concert" can be null if it's a placeholder.
        holder.bindTo(concert);
    }

    private static DiffUtil.ItemCallback<Concert> DIFF_CALLBACK
            = ... // See Implement the diffing callback section.
}
```

PagedListAdapter使用PagedList.Callback对象处理页面加载事件。当用户滚动时，PagedListAdapter调用PagedList.loadAround()以向底层的PagedList提供关于它应该从DataSource获取哪些项的提示。

注意：PagedList是内容不可变的。这意味着，虽然可以将新内容加载到PagedList的实例中，但加载的项本身一旦加载就无法更改。因此，如果PagedList中的内容更新，则PagedListAdapter对象将接收包含更新信息的全新PagedList。

## 实现DIFF CALLBACK

以下示例显示了`areContentsTheSame()`的手动实现，它比较了相关的对象字段：

```java
private static DiffUtil.ItemCallback<Concert> DIFF_CALLBACK =
        new DiffUtil.ItemCallback<Concert>() {

    @Override
    public boolean areItemsTheSame(Concert oldItem, Concert newItem) {
        // The ID property identifies when items are the same.
        return oldItem.getId() == newItem.getId();
    }

    @Override
    public boolean areContentsTheSame(Concert oldItem, Concert newItem) {
        // Don't use the "==" operator here. Either implement and use .equals(),
        // or write custom data comparison logic here.
        return oldItem.equals(newItem);
    }
};
```

由于适配器包含比较项的定义，因此适配器会在加载新的PagedList对象时自动检测对这些项的更改。因此，适配器会在RecyclerView对象中触发高效的`item`动画。

## 使用不同的适配器类型进行区分

如果您选择不从PagedListAdapter继承 - 例如当您使用提供自己的适配器的库时 - 您仍然可以通过直接使用AsyncPagedListDiffer对象来使用Paging Library适配器的diffing功能。

## 在UI中提供占位符

如果您希望UI在应用程序完成获取数据之前显示列表，您可以向用户显示占位符列表项。 PagedList通过将列表项数据显示为null来处理这种情况，直到加载数据。

> 注意：默认情况下，分页库启用此占位符行为。

占位符具有以下好处：

* 支持滚动条：PagedList为PagedListAdapter提供列表项的数量。此信息允许适配器绘制一个滚动条，传达列表的完整大小。加载新页面时，滚动条不会跳转，因为列表不会更改大小。
* 无需加载微调器：由于列表大小已知，因此无需提醒用户正在加载更多项目。占位符本身传达了这些信息。

但是，在添加对占位符的支持之前，请记住以下前提条件：

* 需要可数数据集：Room持久性库中的DataSource实例可以有效地计算其项目。但是，如果您使用的是自定义本地存储解决方案或仅限网络的数据体系结构，则确定数据集中包含的项目数量可能很昂贵甚至无法实现。

* 需要适配器来考虑卸载的项目：用于准备通胀列表的适配器或表示机制需要处理空列表项。例如，将数据绑定到ViewHolder时，需要提供默认值来表示卸载的数据。

* 需要相同大小的项目视图：如果列表项目大小可以根据其内容（例如社交网络更新）进行更改，则项目之间的交叉淡化看起来不太好。我们强烈建议在这种情况下禁用占位符。

# 收集分页数据

## 构造一个可观察的列表

通常，您的UI代码会观察LiveData <PagedList>对象（或者，如果您使用的是RxJava2，则为Flowable <PagedList>或Observable <PagedList>对象），该对象位于应用程序的ViewModel中。此可观察对象在应用程序列表数据的表示和内容之间形成连接。

为了创建这些可观察的PagedList对象之一，将DataSource.Factory的实例传递给LivePagedListBuilder或RxPagedListBuilder对象。 DataSource对象加载单个PagedList的页面。工厂类创建PagedList的新实例以响应内容更新，例如数据库表失效和网络刷新。 

Room持久性库可以为您提供DataSource.Factory对象，或者您可以构建自己的对象。
以下代码段显示了如何使用Room的DataSource.Factory构建功能在应用程序的ViewModel类中创建LiveData <PagedList>的新实例：

`ConcertDao`

```java
@Dao
public interface ConcertDao {
    // The Integer type parameter tells Room to use a PositionalDataSource
    // object, with position-based loading under the hood.
    @Query("SELECT * FROM concerts ORDER BY date DESC")
    DataSource.Factory<Integer, Concert> concertsByDate();
}
```

`ConcertViewModel`

```java
// The Integer type argument corresponds to a PositionalDataSource object.
DataSource.Factory<Integer, Concert> myConcertDataSource =
       concertDao.concertsByDate();

LiveData<PagedList<Concert>> concertList =
        LivePagedListBuilder(myConcertDataSource, /* page size */ 50).build();
```

## 自定义分页配置

