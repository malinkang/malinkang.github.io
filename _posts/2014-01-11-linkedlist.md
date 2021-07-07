---
title: "LinkedList源码分析"
date: 2014-01-11 14:33:57 +0800
categories: [Java,JDK源码分析]
---

## 类图

![image-20210707120008429](https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/images/java/image-20210707120008429.png)

## 构造函数

```java
transient int size = 0;
transient Node<E> first; //记录第一个节点
transient Node<E> last; //记录最后一个节点
public LinkedList() {
}
public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);//调用addAll方法
}
```

## 添加

### add()

```java
public boolean add(E e) {
    linkLast(e);//调用linkLast
    return true;
}
```
### addLast()

```java
public void addLast(E e) {
    //add和addLast等价
    linkLast(e);
}
```

```java
private static class Node<E> {
    E item;//元素
    Node<E> next;//下一个
    Node<E> prev;//前一个
    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

```java
void linkLast(E e) {
    final Node<E> l = last;
    //创建一个新节点
    final Node<E> newNode = new Node<>(l, e, null);
    //赋值给最后一个节点
    last = newNode;
    //如果最后一个节点为空说明链表中还没有元素，将创建的节点赋值给第一个节点
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}
```

### addFirst()

```java
public void addFirst(E e) {
    linkFirst(e);
}
```



```java
private void linkFirst(E e) {
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null)
        last = newNode;
    else
        f.prev = newNode;
    size++;
    modCount++;
}
```

## 获取

```java
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}
```

```java
Node<E> node(int index) {
    // assert isElementIndex(index);
  //如果index<size/2从前面查否则从后面查找
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

## 删除

```java
public E remove() {
    return removeFirst();
}
```

```java
public E removeFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return unlinkFirst(f);
}
```

```java
//根据索引删除
public E remove(int index) {
    checkElementIndex(index);
    //先调用node方法查找然后调用unlink删除
    return unlink(node(index));
}
```

```java
E unlink(Node<E> x) {
    // assert x != null;
    final E element = x.item;
  //获取前一个和后一个
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;
    //pre.next指向next
    if (prev == null) {
        first = next;
    } else {
        prev.next = next;
        x.prev = null;
    }
    //next的prev指向prev
    if (next == null) {
        last = prev;
    } else {
        next.prev = prev;
        x.next = null;
    }
    x.item = null;
    size--;
    modCount++;
    return element;
}
```



