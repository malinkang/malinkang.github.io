---
title: MultiType和AdapterDelegates源码分析
date: 2017-11-03 09:03:48
draft: true
---


[MultiType](https://github.com/drakeet/MultiType)和[AdapterDelegates](https://github.com/sockeqwe/AdapterDelegates)两个库都是对`RecyclerView`的`Adapter`进行封装的库，可以快速实现多种布局类型的`RecyclerView`。两个库的核心思想都是封装一个实体我们暂时命名为`ItemViewBinder`，用来提供布局资源和绑定数据。对于多种类型的Adapter存在多个`ItemViewBinder`对象，如`AItemViewBinder`、`BItemViewBinder`，我们可以使用一个集合来存放这些`ItemViewBinder`此外，还需要建立viewtype和绑定实体之间的一一对应关系，在`onCreateViewHolder`和`onBindViewHolder`中通过viewtype来获取对应的`ItemViewBinder`类实现提供布局资源和数据绑定。下面我们来分别看看两个库的实现方式。


<!--more-->


### MultiType

MultiType的`ItemViewBinder`类就是封装的实体类。

```java
public abstract class ItemViewBinder<T, VH extends ViewHolder> {
    protected abstract @NonNull VH onCreateViewHolder(@NonNull LayoutInflater inflater, @NonNull ViewGroup parent);
    protected abstract void onBindViewHolder(@NonNull VH holder, @NonNull T item);
}

```

TypePool用来存放多个`ItemViewBinder`类，并绑定数据的Model类型实现viewtype和`ItemViewBinder`一一对应的关系。TypePool是一个接口，MultiTypePool是他的直接子类。

```java
   public class MultiTypePool implements TypePool {

    private @NonNull final List<Class<?>> classes; //用来存放model的Class
    private @NonNull final List<ItemViewBinder<?, ?>> binders; //存放ItemViewBinder
    private @NonNull final List<Linker<?>> linkers;//对于相同的model类型，可能有多个布局这时候需要自己提供viewtype


    /**
     * Constructs a MultiTypePool with default lists.
     */
    public MultiTypePool() {
        this.classes = new ArrayList<>();
        this.binders = new ArrayList<>();
        this.linkers = new ArrayList<>();
    }


    /**
     * Constructs a MultiTypePool with default lists and a specified initial capacity.
     *
     * @param initialCapacity the initial capacity of the list
     */
    public MultiTypePool(int initialCapacity) {
        this.classes = new ArrayList<>(initialCapacity);
        this.binders = new ArrayList<>(initialCapacity);
        this.linkers = new ArrayList<>(initialCapacity);
    }


    /**
     * Constructs a MultiTypePool with specified lists.
     *
     * @param classes the list for classes
     * @param binders the list for binders
     * @param linkers the list for linkers
     */
    public MultiTypePool(
        @NonNull List<Class<?>> classes,
        @NonNull List<ItemViewBinder<?, ?>> binders,
        @NonNull List<Linker<?>> linkers) {
        this.classes = classes;
        this.binders = binders;
        this.linkers = linkers;
    }


    @Override
    public <T> void register(
        @NonNull Class<? extends T> clazz,
        @NonNull ItemViewBinder<T, ?> binder,
        @NonNull Linker<T> linker) {
        classes.add(clazz);
        binders.add(binder);
        linkers.add(linker);
    }


    @Override
    public boolean unregister(@NonNull Class<?> clazz) {
        boolean removed = false;
        while (true) {
            int index = classes.indexOf(clazz);
            if (index != -1) {
                classes.remove(index);
                binders.remove(index);
                linkers.remove(index);
                removed = true;
            } else {
                break;
            }
        }
        return removed;
    }


    @Override
    public int size() {
        return classes.size();
    }

    //获取索引
    @Override
    public int firstIndexOf(@NonNull final Class<?> clazz) {
        int index = classes.indexOf(clazz);
        if (index != -1) {
            return index;
        }
        for (int i = 0; i < classes.size(); i++) {
            if (classes.get(i).isAssignableFrom(clazz)) {
                return i;
            }
        }
        return -1;
    }


    @Override
    public @NonNull Class<?> getClass(int index) {
        return classes.get(index);
    }


    @Override
    public @NonNull ItemViewBinder<?, ?> getItemViewBinder(int index) {
        return binders.get(index);
    }


    @Override
    public @NonNull Linker<?> getLinker(int index) {
        return linkers.get(index);
    }
}


```

MultiTypeAdapter：

```java

/**
 * @author drakeet
 */
public class MultiTypeAdapter extends RecyclerView.Adapter<ViewHolder> {

    private static final String TAG = "MultiTypeAdapter";

    private @NonNull List<?> items; //使用集合存放各种类型的数据
    private @NonNull TypePool typePool;


    /**
     * Constructs a MultiTypeAdapter with an empty items list.
     */
    public MultiTypeAdapter() {
        this(Collections.emptyList());
    }


    /**
     * Constructs a MultiTypeAdapter with a items list.
     *
     * @param items the items list
     */
    public MultiTypeAdapter(@NonNull List<?> items) {
        this(items, new MultiTypePool());
    }


    /**
     * Constructs a MultiTypeAdapter with a items list and an initial capacity of TypePool.
     *
     * @param items the items list
     * @param initialCapacity the initial capacity of TypePool
     */
    public MultiTypeAdapter(@NonNull List<?> items, int initialCapacity) {
        this(items, new MultiTypePool(initialCapacity));
    }


    /**
     * Constructs a MultiTypeAdapter with a items list and a TypePool.
     *
     * @param items the items list
     * @param pool the type pool
     */
    public MultiTypeAdapter(@NonNull List<?> items, @NonNull TypePool pool) {
        this.items = items;
        this.typePool = pool;
    }


    /**
     * Registers a type class and its item view binder. If you have registered the class,
     * it will override the original binder(s). Note that the method is non-thread-safe
     * so that you should not use it in concurrent operation.
     * <p>
     * Note that the method should not be called after
     * {@link RecyclerView#setAdapter(RecyclerView.Adapter)}, or you have to call the setAdapter
     * again.
     * </p>
     *
     * @param clazz the class of a item
     * @param binder the item view binder
     * @param <T> the item data type
     */
    public <T> void register(@NonNull Class<? extends T> clazz, @NonNull ItemViewBinder<T, ?> binder) {
        checkAndRemoveAllTypesIfNeeded(clazz);
        register(clazz, binder, new DefaultLinker<T>());
    }


    <T> void register(
        @NonNull Class<? extends T> clazz,
        @NonNull ItemViewBinder<T, ?> binder,
        @NonNull Linker<T> linker) {
        typePool.register(clazz, binder, linker);
        binder.adapter = this;
    }


    /**
     * Registers a type class to multiple item view binders. If you have registered the
     * class, it will override the original binder(s). Note that the method is non-thread-safe
     * so that you should not use it in concurrent operation.
     * <p>
     * Note that the method should not be called after
     * {@link RecyclerView#setAdapter(RecyclerView.Adapter)}, or you have to call the setAdapter
     * again.
     * </p>
     *
     * @param clazz the class of a item
     * @param <T> the item data type
     * @return {@link OneToManyFlow} for setting the binders
     * @see #register(Class, ItemViewBinder)
     */
    @CheckResult
    public @NonNull <T> OneToManyFlow<T> register(@NonNull Class<? extends T> clazz) {
        checkAndRemoveAllTypesIfNeeded(clazz);
        return new OneToManyBuilder<>(this, clazz);
    }


    /**
     * Registers all of the contents in the specified type pool. If you have registered a
     * class, it will override the original binder(s). Note that the method is non-thread-safe
     * so that you should not use it in concurrent operation.
     * <p>
     * Note that the method should not be called after
     * {@link RecyclerView#setAdapter(RecyclerView.Adapter)}, or you have to call the setAdapter
     * again.
     * </p>
     *
     * @param pool type pool containing contents to be added to this adapter inner pool
     * @see #register(Class, ItemViewBinder)
     * @see #register(Class)
     */
    public void registerAll(@NonNull final TypePool pool) {
        final int size = pool.size();
        for (int i = 0; i < size; i++) {
            registerWithoutChecking(
                pool.getClass(i),
                pool.getItemViewBinder(i),
                pool.getLinker(i)
            );
        }
    }


    /**
     * Sets and updates the items atomically and safely. It is recommended to use this method
     * to update the items with a new wrapper list or consider using {@link CopyOnWriteArrayList}.
     *
     * <p>Note: If you want to refresh the list views after setting items, you should
     * call {@link RecyclerView.Adapter#notifyDataSetChanged()} by yourself.</p>
     *
     * @param items the new items list
     * @since v2.4.1
     */
    public void setItems(@NonNull List<?> items) {
        this.items = items;
    }


    public @NonNull List<?> getItems() {
        return items;
    }


    /**
     * Set the TypePool to hold the types and view binders.
     *
     * @param typePool the TypePool implementation
     */
    public void setTypePool(@NonNull TypePool typePool) {
        this.typePool = typePool;
    }


    public @NonNull TypePool getTypePool() {
        return typePool;
    }


    @Override
    public final int getItemViewType(int position) {
        Object item = items.get(position); //获取item
        return indexInTypesOf(position, item); //
    }


    @Override
    public final ViewHolder onCreateViewHolder(ViewGroup parent, int indexViewType) {
        LayoutInflater inflater = LayoutInflater.from(parent.getContext());
        ItemViewBinder<?, ?> binder = typePool.getItemViewBinder(indexViewType);
        return binder.onCreateViewHolder(inflater, parent);
    }


    /**
     * This method is deprecated and unused. You should not call this method.
     * <p>
     * If you need to call the binding, use {@link RecyclerView.Adapter#onBindViewHolder(ViewHolder,
     * int, List)} instead.
     * </p>
     *
     * @param holder The ViewHolder which should be updated to represent the contents of the
     * item at the given position in the data set.
     * @param position The position of the item within the adapter's data set.
     * @throws IllegalAccessError By default.
     * @deprecated Call {@link RecyclerView.Adapter#onBindViewHolder(ViewHolder, int, List)}
     * instead.
     */
    @Override @Deprecated
    public final void onBindViewHolder(ViewHolder holder, int position) {
        onBindViewHolder(holder, position, Collections.emptyList());
    }


    @Override @SuppressWarnings("unchecked")
    public final void onBindViewHolder(ViewHolder holder, int position, List<Object> payloads) {
        Object item = items.get(position);
        ItemViewBinder binder = typePool.getItemViewBinder(holder.getItemViewType());
        binder.onBindViewHolder(holder, item, payloads);
    }


    @Override
    public final int getItemCount() {
        return items.size();
    }


    /**
     * Called to return the stable ID for the item, and passes the event to its associated binder.
     *
     * @param position Adapter position to query
     * @return the stable ID of the item at position
     * @see ItemViewBinder#getItemId(Object)
     * @see RecyclerView.Adapter#setHasStableIds(boolean)
     * @since v3.2.0
     */
    @Override @SuppressWarnings("unchecked")
    public final long getItemId(int position) {
        Object item = items.get(position);
        int itemViewType = getItemViewType(position);
        ItemViewBinder binder = typePool.getItemViewBinder(itemViewType);
        return binder.getItemId(item);
    }


    /**
     * Called when a view created by this adapter has been recycled, and passes the event to its
     * associated binder.
     *
     * @param holder The ViewHolder for the view being recycled
     * @see RecyclerView.Adapter#onViewRecycled(ViewHolder)
     * @see ItemViewBinder#onViewRecycled(ViewHolder)
     */
    @Override @SuppressWarnings("unchecked")
    public final void onViewRecycled(@NonNull ViewHolder holder) {
        getRawBinderByViewHolder(holder).onViewRecycled(holder);
    }


    /**
     * Called by the RecyclerView if a ViewHolder created by this Adapter cannot be recycled
     * due to its transient state, and passes the event to its associated item view binder.
     *
     * @param holder The ViewHolder containing the View that could not be recycled due to its
     * transient state.
     * @return True if the View should be recycled, false otherwise. Note that if this method
     * returns <code>true</code>, RecyclerView <em>will ignore</em> the transient state of
     * the View and recycle it regardless. If this method returns <code>false</code>,
     * RecyclerView will check the View's transient state again before giving a final decision.
     * Default implementation returns false.
     * @see RecyclerView.Adapter#onFailedToRecycleView(ViewHolder)
     * @see ItemViewBinder#onFailedToRecycleView(ViewHolder)
     */
    @Override @SuppressWarnings("unchecked")
    public final boolean onFailedToRecycleView(@NonNull ViewHolder holder) {
        return getRawBinderByViewHolder(holder).onFailedToRecycleView(holder);
    }


    /**
     * Called when a view created by this adapter has been attached to a window, and passes the
     * event to its associated item view binder.
     *
     * @param holder Holder of the view being attached
     * @see RecyclerView.Adapter#onViewAttachedToWindow(ViewHolder)
     * @see ItemViewBinder#onViewAttachedToWindow(ViewHolder)
     */
    @Override @SuppressWarnings("unchecked")
    public final void onViewAttachedToWindow(@NonNull ViewHolder holder) {
        getRawBinderByViewHolder(holder).onViewAttachedToWindow(holder);
    }


    /**
     * Called when a view created by this adapter has been detached from its window, and passes
     * the event to its associated item view binder.
     *
     * @param holder Holder of the view being detached
     * @see RecyclerView.Adapter#onViewDetachedFromWindow(ViewHolder)
     * @see ItemViewBinder#onViewDetachedFromWindow(ViewHolder)
     */
    @Override @SuppressWarnings("unchecked")
    public final void onViewDetachedFromWindow(@NonNull ViewHolder holder) {
        getRawBinderByViewHolder(holder).onViewDetachedFromWindow(holder);
    }


    private @NonNull ItemViewBinder getRawBinderByViewHolder(@NonNull ViewHolder holder) {
        return typePool.getItemViewBinder(holder.getItemViewType());
    }

//这样写不存在问题吗？待验证。如果第一个有linker返回索引的0+1 第二个没有linker索引刚好是1 岂不是有问题？？
    int indexInTypesOf(int position, @NonNull Object item) throws BinderNotFoundException {
        int index = typePool.firstIndexOf(item.getClass());
        if (index != -1) {
            @SuppressWarnings("unchecked")
            Linker<Object> linker = (Linker<Object>) typePool.getLinker(index);
            return index + linker.index(position, item);
        }
        throw new BinderNotFoundException(item.getClass());
    }


    private void checkAndRemoveAllTypesIfNeeded(@NonNull Class<?> clazz) {
        if (typePool.unregister(clazz)) {
            Log.w(TAG, "You have registered the " + clazz.getSimpleName() + " type. " +
                "It will override the original binder(s).");
        }
    }


    /** A safe register method base on the TypePool's safety for TypePool. */
    @SuppressWarnings("unchecked")
    private void registerWithoutChecking(@NonNull Class clazz, @NonNull ItemViewBinder binder, @NonNull Linker linker) {
        checkAndRemoveAllTypesIfNeeded(clazz);
        register(clazz, binder, linker);
    }
}
```

### AdapterDelegate

AdapterDelegate的封装实体类为`AdapterDelegate`

```java
/**
 * This delegate provide method to hook in this delegate to {@link RecyclerView.Adapter} lifecycle.
 * This "hook in" mechanism is provided by {@link AdapterDelegatesManager} and that is the
 * component
 * you have to use.
 *
 * @param <T> The type of the data source
 * @author Hannes Dorfmann
 * @since 1.0
 */
public abstract class AdapterDelegate<T> {

  /**
   * Called to determine whether this AdapterDelegate is the responsible for the given data
   * element.
   *
   * @param items The data source of the Adapter
   * @param position The position in the datasource
   * @return true, if this item is responsible,  otherwise false
   */
  protected abstract boolean isForViewType(@NonNull T items, int position);

  /**
   * Creates the  {@link RecyclerView.ViewHolder} for the given data source item
   *
   * @param parent The ViewGroup parent of the given datasource
   * @return The new instantiated {@link RecyclerView.ViewHolder}
   */
  @NonNull abstract protected RecyclerView.ViewHolder onCreateViewHolder(@NonNull ViewGroup parent);

  /**
   * Called to bind the {@link RecyclerView.ViewHolder} to the item of the datas source set
   *
   * @param items The data source
   * @param position The position in the datasource
   * @param holder The {@link RecyclerView.ViewHolder} to bind
   * @param payloads A non-null list of merged payloads. Can be empty list if requires full update.
   */
  protected abstract void onBindViewHolder(@NonNull T items, int position,
      @NonNull RecyclerView.ViewHolder holder, @NonNull List<Object> payloads);

  /**
   * Called when a view created by this adapter has been recycled.
   *
   * <p>A view is recycled when a {@link RecyclerView.LayoutManager} decides that it no longer
   * needs to be attached to its parent {@link RecyclerView}. This can be because it has
   * fallen out of visibility or a set of cached views represented by views still
   * attached to the parent RecyclerView. If an item view has large or expensive data
   * bound to it such as large bitmaps, this may be a good place to release those
   * resources.</p>
   * <p>
   * RecyclerView calls this method right before clearing ViewHolder's internal data and
   * sending it to RecycledViewPool. This way, if ViewHolder was holding valid information
   * before being recycled, you can call {@link RecyclerView.ViewHolder#getAdapterPosition()} to
   * get
   * its adapter position.
   *
   * @param viewHolder The ViewHolder for the view being recycled
   */
  protected void onViewRecycled(@NonNull RecyclerView.ViewHolder viewHolder) {
  }

  /**
   * Called by the RecyclerView if a ViewHolder created by this Adapter cannot be recycled
   * due to its transient state. Upon receiving this callback, Adapter can clear the
   * animation(s) that effect the View's transient state and return <code>true</code> so that
   * the View can be recycled. Keep in mind that the View in question is already removed from
   * the RecyclerView.
   * <p>
   * In some cases, it is acceptable to recycle a View although it has transient state. Most
   * of the time, this is a case where the transient state will be cleared in
   * {@link RecyclerView.Adapter#onBindViewHolder(RecyclerView.ViewHolder, int)} call when View is
   * rebound to a new position.
   * For this reason, RecyclerView leaves the decision to the Adapter and uses the return
   * value of this method to decide whether the View should be recycled or not.
   * <p>
   * Note that when all animations are created by {@link RecyclerView.ItemAnimator}, you
   * should never receive this callback because RecyclerView keeps those Views as children
   * until their animations are complete. This callback is useful when children of the item
   * views create animations which may not be easy to implement using an {@link
   * RecyclerView.ItemAnimator}.
   * <p>
   * You should <em>never</em> fix this issue by calling
   * <code>holder.itemView.setHasTransientState(false);</code> unless you've previously called
   * <code>holder.itemView.setHasTransientState(true);</code>. Each
   * <code>View.setHasTransientState(true)</code> call must be matched by a
   * <code>View.setHasTransientState(false)</code> call, otherwise, the state of the View
   * may become inconsistent. You should always prefer to end or cancel animations that are
   * triggering the transient state instead of handling it manually.
   *
   * @param holder The ViewHolder containing the View that could not be recycled due to its
   * transient state.
   * @return True if the View should be recycled, false otherwise. Note that if this method
   * returns <code>true</code>, RecyclerView <em>will ignore</em> the transient state of
   * the View and recycle it regardless. If this method returns <code>false</code>,
   * RecyclerView will check the View's transient state again before giving a final decision.
   * Default implementation returns false.
   */
  protected boolean onFailedToRecycleView(@NonNull RecyclerView.ViewHolder holder) {
    return false;
  }

  /**
   * Called when a view created by this adapter has been attached to a window.
   *
   * <p>This can be used as a reasonable signal that the view is about to be seen
   * by the user. If the adapter previously freed any resources in
   * {@link RecyclerView.Adapter#onViewDetachedFromWindow(RecyclerView.ViewHolder)
   * onViewDetachedFromWindow}
   * those resources should be restored here.</p>
   *
   * @param holder Holder of the view being attached
   */
  protected void onViewAttachedToWindow(@NonNull RecyclerView.ViewHolder holder) {
  }

  /**
   * Called when a view created by this adapter has been detached from its window.
   *
   * <p>Becoming detached from the window is not necessarily a permanent condition;
   * the consumer of an Adapter's views may choose to cache views offscreen while they
   * are not visible, attaching an detaching them as appropriate.</p>
   *
   * @param holder Holder of the view being detached
   */
  protected void onViewDetachedFromWindow(RecyclerView.ViewHolder holder) {
  }
}
```

AdapterDelegatesManager用来存储AdapterDelegate和建立与viewtype的对应关系。


```java

public class AdapterDelegatesManager<T> {
//...
	  /**
   * Map for ViewType to AdapterDelegate
   */
//使用Map建立	  ViewType和adapter的对应关系
protected SparseArrayCompat<AdapterDelegate<T>> delegates = new SparseArrayCompat();
//....

public AdapterDelegatesManager<T> addDelegate(@NonNull AdapterDelegate<T> delegate) {
    // algorithm could be improved since there could be holes,
    // but it's very unlikely that we reach Integer.MAX_VALUE and run out of unused indexes
    int viewType = delegates.size();
    //在上面情况下会走这里？
    while (delegates.get(viewType) != null) {
      viewType++;
      if (viewType == FALLBACK_DELEGATE_VIEW_TYPE) {
        throw new IllegalArgumentException(
            "Oops, we are very close to Integer.MAX_VALUE. It seems that there are no more free and unused view type integers left to add another AdapterDelegate.");
      }
    }
    return addDelegate(viewType, false, delegate);
  }

}
  /**
   * Adds an {@link AdapterDelegate}.
   *
   * @param viewType The viewType id
   * @param allowReplacingDelegate if true, you allow to replacing the given delegate any previous
   * delegate for the same view type. if false, you disallow and a {@link IllegalArgumentException}
   * will be thrown if you try to replace an already registered {@link AdapterDelegate} for the
   * same view type.
   * @param delegate The delegate to add
   * @throws IllegalArgumentException if <b>allowReplacingDelegate</b>  is false and an {@link
   * AdapterDelegate} is already added (registered)
   * with the same ViewType.
   * @throws IllegalArgumentException if viewType is {@link #FALLBACK_DELEGATE_VIEW_TYPE} which is
   * reserved
   * @see #addDelegate(AdapterDelegate)
   * @see #addDelegate(int, AdapterDelegate)
   * @see #setFallbackDelegate(AdapterDelegate)
   */
  public AdapterDelegatesManager<T> addDelegate(int viewType, boolean allowReplacingDelegate,
      @NonNull AdapterDelegate<T> delegate) {

    if (delegate == null) {
      throw new NullPointerException("AdapterDelegate is null!");
    }

    if (viewType == FALLBACK_DELEGATE_VIEW_TYPE) {
      throw new IllegalArgumentException("The view type = "
          + FALLBACK_DELEGATE_VIEW_TYPE
          + " is reserved for fallback adapter delegate (see setFallbackDelegate() ). Please use another view type.");
    }

    if (!allowReplacingDelegate && delegates.get(viewType) != null) {
      throw new IllegalArgumentException(
          "An AdapterDelegate is already registered for the viewType = "
              + viewType
              + ". Already registered AdapterDelegate is "
              + delegates.get(viewType));
    }

    delegates.put(viewType, delegate);

    return this;
  }
    /**
   * Must be called from {@link RecyclerView.Adapter#getItemViewType(int)}. Internally it scans all
   * the registered {@link AdapterDelegate} and picks the right one to return the ViewType integer.
   *
   * @param items Adapter's data source
   * @param position the position in adapters data source
   * @return the ViewType (integer). Returns {@link #FALLBACK_DELEGATE_VIEW_TYPE} in case that the
   * fallback adapter delegate should be used
   * @throws NullPointerException if no {@link AdapterDelegate} has been found that is
   * responsible for the given data element in data set (No {@link AdapterDelegate} for the given
   * ViewType)
   * @throws NullPointerException if items is null
   */
  public int getItemViewType(@NonNull T items, int position) {

    if (items == null) {
      throw new NullPointerException("Items datasource is null!");
    }

    int delegatesCount = delegates.size();
    //遍历所有的delegate 
    for (int i = 0; i < delegatesCount; i++) {
      AdapterDelegate<T> delegate = delegates.valueAt(i);
      //根据类型判断 获取key
      if (delegate.isForViewType(items, position)) {
        return delegates.keyAt(i); 
      }
    }

    if (fallbackDelegate != null) {
      return FALLBACK_DELEGATE_VIEW_TYPE;
    }

    throw new NullPointerException(
        "No AdapterDelegate added that matches position=" + position + " in data source");
  }

```








