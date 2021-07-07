---
title: "ArrayList源码分析"
date: 2014-01-04 14:33:57 +0800
categories: [Java,JDK源码分析]
---

## 类图

![image-20210707101448097](https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/images/java/image-20210707101448097.png)



## 构造函数

```java
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}

//无参构造函数
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
public ArrayList(Collection<? extends E> c) {
    Object[] a = c.toArray(); //先转换为数组
    if ((size = a.length) != 0) {
        if (c.getClass() == ArrayList.class) {
            elementData = a;
        } else {
            elementData = Arrays.copyOf(a, size, Object[].class);
        }
    } else {
        // replace with empty array.
        elementData = EMPTY_ELEMENTDATA;
    }
}
```

## 添加

```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```
```java
private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}
```
```java
//默认容量为10
private static final int DEFAULT_CAPACITY = 10;
//计算容量
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    //如果是空的 取DEFAULT_CAPACITY和minCapacity的最大值
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}
```

```java
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;
    // overflow-conscious code
    //minCapacity容量可以理解为需要的最小容量
    //当需要的容量大于数组的大小则进行扩容
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
```

```java
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
private void grow(int minCapacity) {
    // overflow-conscious code
    //老的容量
    int oldCapacity = elementData.length;
    //扩容为原来的1.5倍
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    //如果扩容1.5倍还是不能满足，则扩展至需要的容量
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    //如果容量大于数组的最大容量则调用hugeCapacity()
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

```java
private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    //如果期望的大于最大数组size则返回Integer最大值，否则返回最大数组size
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```

## 获取

```java
public E get(int index) {
    //范围检查
    rangeCheck(index);
    return elementData(index);
}
```

```java
private void rangeCheck(int index) {
    //index大于size抛出异常
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

## 删除

```java
public boolean remove(Object o) {
    //遍历寻找目标
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}
```

```java

//[1,2,3,4,5,6,7]
//假设index = 3 size = 7 要移动5,6,7 
private void fastRemove(int index) {
    modCount++;
    //计算移动的长度
    int numMoved = size - index - 1;
    //移动数组
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    //最后一位设置为空
    //数组长度并没有收缩
    elementData[--size] = null; // clear to let GC do its work
}
```

## contains()

```java
public boolean contains(Object o) {
    return indexOf(o) >= 0;
}
```

```java
//遍历获取符合条件的索引
public int indexOf(Object o) {
    if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
```

## retainAll()


```java
public boolean retainAll(Collection<?> c) {
    Objects.requireNonNull(c);
    return batchRemove(c, true);
}
```

```java
//批量删除
private boolean batchRemove(Collection<?> c, boolean complement) {
    final Object[] elementData = this.elementData;
    int r = 0, w = 0;
    boolean modified = false;
    try {
        //这里使用双指针将包含的元素移动到数组的前面
        for (; r < size; r++)
            if (c.contains(elementData[r]) == complement)
                elementData[w++] = elementData[r];
    } finally {
        // Preserve behavioral compatibility with AbstractCollection,
        // even if c.contains() throws.
        if (r != size) {
            System.arraycopy(elementData, r,
                             elementData, w,
                             size - r);
            w += size - r;
        }
        //如果w与size不相等
        if (w != size) {
            // clear to let GC do its work
            //将索引>=w的元素设置为null
            for (int i = w; i < size; i++)
                elementData[i] = null;
            modCount += size - w;
            size = w;
            modified = true;
        }
    }
    return modified;
}
```

## toArray()

```java
public <T> T[] toArray(T[] a) {
    //如果传入的数组长度小于size 调用copyOf创建一个新数组大小为size
    if (a.length < size)
        // Make a new array of a's runtime type, but my contents:
        return (T[]) Arrays.copyOf(elementData, size, a.getClass());
    System.arraycopy(elementData, 0, a, 0, size);
    if (a.length > size)
        a[size] = null;
    return a;
}
```

