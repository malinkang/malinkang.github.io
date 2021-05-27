---
title: "Recyclerview源码分析三"
date: 2020-09-04T16:39:09+08:00
lastmod: 2020-09-04T16:39:09+08:00
tags: ["Android","RecyclerView"]
draft: false
---

上一篇分析`RecyclerView`整体流程的时候提到，`RecyclerView`缓存机制比较复杂，所以这一篇单独分析一下`RecyclerView`的缓存机制。


## Recycler中的缓存

`Recycler`中定义了三个list来缓存`ViewHolder`，那这几个缓存有什么区别呢。

我们先来看`mAttachedScrap`和`mChangedScrap`。`Recycler`的`scrapView`中调用`mAttachedScrap`和`mChangedScrap`的`add`方法。

```java
//LayoutManager的scrapOrRecycleView方法
private void scrapOrRecycleView(Recycler recycler, int index, View view) {
    final ViewHolder viewHolder = getChildViewHolderInt(view);
    if (viewHolder.shouldIgnore()) {
        if (DEBUG) {
            Log.d(TAG, "ignoring view " + viewHolder);
        }
        return;
    }
    if (viewHolder.isInvalid() && !viewHolder.isRemoved()
            && !mRecyclerView.mAdapter.hasStableIds()) {
        removeViewAt(index);
        recycler.recycleViewHolderInternal(viewHolder);
    } else {
        detachViewAt(index);
        //调用Recycler的scrapView的方法
        recycler.scrapView(view);
        mRecyclerView.mViewInfoStore.onViewDetached(viewHolder);
    }
}
//LayoutManager的detachAndScrapAttachedViews方法
public void detachAndScrapAttachedViews(@NonNull Recycler recycler) {
    final int childCount = getChildCount();
    for (int i = childCount - 1; i >= 0; i--) {
        final View v = getChildAt(i);

        scrapOrRecycleView(recycler, i, v);
    }
}
```
在`detachAndScrapAttachedViews`方法中可以看到RecyclerView中的所有Child也就是屏幕中的所有View都被添加到`mAttachedScrap`中。`detachAndScrapAttachedViews`被`onLayoutChildren`方法调用。

所以当我们调用`adapter`的`notifyDataSetChanged`方法时。调用`requestLayout`触发`onLayout`，然后会回收屏幕中所有的`holder`。再次获取时复用这些`holder`。


```java
//RecyclerViewDataObserver onChanged方法
@Override
public void onChanged() {
    assertNotInLayoutOrScroll(null);
    mState.mStructureChanged = true;

    processDataSetCompletelyChanged(true);
    if (!mAdapterHelper.hasPendingUpdates()) {
        requestLayout();
    }
}
```

`mCacheViews`用于存放哪些缓存呢，我们同样可以查看`mCacheViews`的`add`方法调用位置，以及一系列的调用链

```java
/**
* Remove a child view and recycle it using the given Recycler.
*
* @param index    Index of child to remove and recycle
* @param recycler Recycler to use to recycle child
*/
public void removeAndRecycleViewAt(int index, @NonNull Recycler recycler) {
    final View view = getChildAt(index);
    removeViewAt(index);
    recycler.recycleView(view);
}
//LinearLayoutManager的recycleChildren方法
private void recycleChildren(RecyclerView.Recycler recycler, int startIndex, int endIndex) {
    if (startIndex == endIndex) {
        return;
    }
    if (DEBUG) {
        Log.d(TAG, "Recycling " + Math.abs(startIndex - endIndex) + " items");
    }
    if (endIndex > startIndex) {
        for (int i = endIndex - 1; i >= startIndex; i--) {
            removeAndRecycleViewAt(i, recycler);
        }
    } else {
        for (int i = startIndex; i > endIndex; i--) {
            removeAndRecycleViewAt(i, recycler);
        }
    }
}
//LinearLayoutManager的recycleViewsFromStart方法
//recycleViewsFromStart 和 recycleViewsFromEnd两个方法其实差不多只不过开始回收的位置不一样
//如果向上滑动会调用recycleViewsFromStart 优先回收底部的view，如果向下滑动则相反
private void recycleViewsFromStart(RecyclerView.Recycler recycler, int scrollingOffset,
        int noRecycleSpace) {
    if (scrollingOffset < 0) {
        if (DEBUG) {
            Log.d(TAG, "Called recycle from start with a negative value. This might happen"
                    + " during layout changes but may be sign of a bug");
        }
        return;
    }
    // ignore padding, ViewGroup may not clip children.
    final int limit = scrollingOffset - noRecycleSpace;
    final int childCount = getChildCount();
    if (mShouldReverseLayout) {
        for (int i = childCount - 1; i >= 0; i--) {
            View child = getChildAt(i);
            //这是是判断是否可见来回收
            if (mOrientationHelper.getDecoratedEnd(child) > limit
                    || mOrientationHelper.getTransformedEndWithDecoration(child) > limit) {
                // stop here
                recycleChildren(recycler, childCount - 1, i);
                return;
            }
        }
    } else {
        for (int i = 0; i < childCount; i++) {
            View child = getChildAt(i);
            if (mOrientationHelper.getDecoratedEnd(child) > limit
                    || mOrientationHelper.getTransformedEndWithDecoration(child) > limit) {
                // stop here
                recycleChildren(recycler, 0, i);
                return;
            }
        }
    }
}
private void recycleByLayoutState(RecyclerView.Recycler recycler, LayoutState layoutState) {
    if (!layoutState.mRecycle || layoutState.mInfinite) {
        return;
    }
    int scrollingOffset = layoutState.mScrollingOffset;
    int noRecycleSpace = layoutState.mNoRecycleSpace;
    if (layoutState.mLayoutDirection == LayoutState.LAYOUT_START) {
        recycleViewsFromEnd(recycler, scrollingOffset, noRecycleSpace);
    } else {
        recycleViewsFromStart(recycler, scrollingOffset, noRecycleSpace);
    }
}
```
`recycleByLayoutState`会被`fill`方法调用。除了`onLayoutChildren`方法会调用`fill`方法之外，`scrollBy`也会调用。在滑动过程中，会判断该View是否可见，如果不可见，则会判断`mCacheViews`是否已经达到最大容量。如果没达到直接添加。
如果已经满了，则会移除第一条，并把移除的添加到`RecycledViewPool`里面，再进行添加。

## 获取缓存

`tryGetViewHolderForPositionByDeadline`方法会依次尝试`从mAttachedScrap`、`mCachedViews`和`RecycledViewPool`中获取`ViewHolder`。


```java
@Nullable
ViewHolder tryGetViewHolderForPositionByDeadline(int position,
        boolean dryRun, long deadlineNs) {
    if (position < 0 || position >= mState.getItemCount()) {
        throw new IndexOutOfBoundsException("Invalid item position " + position
                + "(" + position + "). Item count:" + mState.getItemCount()
                + exceptionLabel());
    }
    boolean fromScrapOrHiddenOrCache = false;
    ViewHolder holder = null;
    // 0) If there is a changed scrap, try to find from there
    //首先先从 changed scrap 中获取
    if (mState.isPreLayout()) {
        holder = getChangedScrapViewForPosition(position);
        fromScrapOrHiddenOrCache = holder != null;
    }
    // 1) Find by position from scrap/hidden list/cache
    //通过position 从scrap/hidden list/cache中获取
    if (holder == null) {
        holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
        if (holder != null) {
            if (!validateViewHolderForOffsetPosition(holder)) {
                // recycle holder (and unscrap if relevant) since it can't be used
                if (!dryRun) {
                    // we would like to recycle this but need to make sure it is not used by
                    // animation logic etc.
                    holder.addFlags(ViewHolder.FLAG_INVALID);
                    if (holder.isScrap()) {
                        removeDetachedView(holder.itemView, false);
                        holder.unScrap();
                    } else if (holder.wasReturnedFromScrap()) {
                        holder.clearReturnedFromScrapFlag();
                    }
                    recycleViewHolderInternal(holder);
                }
                holder = null;
            } else {
                fromScrapOrHiddenOrCache = true;
            }
        }
    }
    if (holder == null) {
        final int offsetPosition = mAdapterHelper.findPositionOffset(position);
        if (offsetPosition < 0 || offsetPosition >= mAdapter.getItemCount()) {
            throw new IndexOutOfBoundsException("Inconsistency detected. Invalid item "
                    + "position " + position + "(offset:" + offsetPosition + ")."
                    + "state:" + mState.getItemCount() + exceptionLabel());
        }

        final int type = mAdapter.getItemViewType(offsetPosition);
        // 2) Find from scrap/cache via stable ids, if exists
        //通过 stable id从 scrap/cache中获取
        if (mAdapter.hasStableIds()) {
            holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),
                    type, dryRun);
            if (holder != null) {
                // update position
                holder.mPosition = offsetPosition;
                fromScrapOrHiddenOrCache = true;
            }
        }
        if (holder == null && mViewCacheExtension != null) {
            // We are NOT sending the offsetPosition because LayoutManager does not
            // know it.
            final View view = mViewCacheExtension
                    .getViewForPositionAndType(this, position, type);
            if (view != null) {
                holder = getChildViewHolder(view);
                if (holder == null) {
                    throw new IllegalArgumentException("getViewForPositionAndType returned"
                            + " a view which does not have a ViewHolder"
                            + exceptionLabel());
                } else if (holder.shouldIgnore()) {
                    throw new IllegalArgumentException("getViewForPositionAndType returned"
                            + " a view that is ignored. You must call stopIgnoring before"
                            + " returning this view." + exceptionLabel());
                }
            }
        }
        //尝试从RecycledViewPool中获取
        if (holder == null) { // fallback to pool
            if (DEBUG) {
                Log.d(TAG, "tryGetViewHolderForPositionByDeadline("
                        + position + ") fetching from shared pool");
            }
            holder = getRecycledViewPool().getRecycledView(type);
            if (holder != null) {
                holder.resetInternal();
                if (FORCE_INVALIDATE_DISPLAY_LIST) {
                    invalidateDisplayListInt(holder);
                }
            }
        }
        if (holder == null) {
            long start = getNanoTime();
            if (deadlineNs != FOREVER_NS
                    && !mRecyclerPool.willCreateInTime(type, start, deadlineNs)) {
                // abort - we have a deadline we can't meet
                return null;
            }
            //都没有获取到
            //创建viewholder
            holder = mAdapter.createViewHolder(RecyclerView.this, type);
            if (ALLOW_THREAD_GAP_WORK) {
                // only bother finding nested RV if prefetching
                RecyclerView innerView = findNestedRecyclerView(holder.itemView);
                if (innerView != null) {
                    holder.mNestedRecyclerView = new WeakReference<>(innerView);
                }
            }

            long end = getNanoTime();
            mRecyclerPool.factorInCreateTime(type, end - start);

        }
    }
    //...
    return holder;
}
```

## 缓存图解

上面已经分析了缓存个整个流程，为了更加直观，我做了几张图来梳理从进入列表，以及滑动过程中`Adapter`、`Recycler`以及`RecycledViewPool`是如何工作的。

1. 首次进入，`Adapter`创建9个`ViewHolder`并展示。

![](/images/recyclerview-cache-1.png)

2. 向上滑动列表，`GapWorker`的`prefetchPositionWithDeadline()`方法会调用`Recycler`的`tryGetViewHolderForPositionByDeadline()`预创建一个`ViewHolder`，创建的`ViewHolder`不可见，所以会调用`Recycler`的`recycleViewHolderInternal()`方法将`ViewHolder`存入到`mCachedViews`.

![](/images/recyclerview-cache-2.png)

滑动过程中`GapWorker`会多次调用`prefetchPositionWithDeadline()`方法。第一次缓存取不到缓存，调用`Adapter`的`createViewHolder()`，第二次会从缓存中，然后根据判断不可见，再存入缓存中。

![](/images/recyclerview-cache-3.png)

3. 继续向上滑动，`f6c081b`可见，所以停止回收。并新创建一个新的`ViewHoder`存到`mCachedViews`中。

![](/images/recyclerview-cache-4.png)

4. 继续向上滑，第一条不可见，回收。

![](/images/recyclerview-cache-5.png)

5. 继续上滑动，第二条将要消失。继续创建新的`ViewHoder`。

![](/images/recyclerview-cache-6.png)

6. 继续上滑动，第二条消失，回收。

![](/images/recyclerview-cache-7.png)

7. 继续上滑，第三条将要消失。这一步没有创建新的`ViewHoder`。


![](/images/recyclerview-cache-8.png)

8. 继续上滑动，第三条消失，回收。此时`mCachedViews`达到最大存储，此时会移除``mCachedViews`中的第一条，存放到`RecycledViewPool`中。

![](/images/recyclerview-cache-9.png)

9. 后面继续向上滑动，就会复用`RecycledViewPool`中的`holder`，而不是重新创建。在这个过程中`Adapter`一共创建的`holder`的个数是一屏展示的个数加上`mCacheViews`的最大个数。