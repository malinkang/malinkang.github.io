---
title: "《算法》读书笔记 第3章 查找"
date: 2016-03-01T14:33:57+08:00
tags: ["算法","读书笔记"]
toc: true
draft: true
---

## 3.1 符号表

### 3.1.1 API

### 3.1.2 有序符号表

### 3.1.3 用例举例

### 3.1.4 无序链表中的顺序查找

### 3.1.5 有序数组中的二分查找

```java
import edu.princeton.cs.algs4.BinarySearch;

/**
 * Created by malk on 2018/12/25.
 */
public class BinarySearchST<Key extends Comparable<Key>, Value> {
    private Key[] keys;
    private Value[] values;
    private int N;

    public BinarySearchST(int capacity) {
        keys = (Key[]) new Comparable[capacity];
        values = (Value[]) new Object[capacity];
    }

    public void delete(Key key) {
        put(key, null);
    }

    boolean contains(Key key) {
        return get(key) != null;
    }

    public int size() {
        return N;
    }

    public boolean isEmpty() {
        return size() == 0;
    }

    public Value get(Key key) {
        if (isEmpty()) return null;
        int i = rank(key);
        if (i < N && keys[i].compareTo(key) == 0) return values[i];
        else return null;
    }

    //基于有序数组的二分查找
    public int rank(Key key) {
        int lo = 0, hi = N - 1;
        while (lo <= hi) {
            int mid = lo + (hi - lo) / 2;
            int cmp = key.compareTo(keys[mid]);
            if (cmp < 0) hi = mid - 1;
            else if (cmp > 0) lo = mid + 1;
            else return mid;
        }
        //如果未找到则返回lo，lo是一个临界值，lo位置的数据比key大，lo+1位置的数据比key小
        //可以看下图的二分查找排名
        return lo;
    }

    public void put(Key key, Value value) {
        //通过二分查找查找key
        int i = rank(key);
        //存在key
        if (i < N && keys[i].compareTo(key) == 0) {
            values[i] = value;
            return;
        }
        //插入新元素前将所有较大的键向后移动一格
        for (int j = N; j > i; j--) {
            keys[j] = keys[j - 1];
            values[j] = values[j - 1];
        }
        keys[i] = key;
        values[i] = value;
        N++;
    }

}

```

![&#x4E8C;&#x5206;&#x67E5;&#x627E;&#x6392;&#x540D;&#x8F68;&#x8FF9;](../.gitbook/assets/image%20%289%29.png)

### 3.1.6 对二分查找的分析

### 3.1.7 预览

## 3.2 二叉查找树

```java
import edu.princeton.cs.algs4.Queue;

/**
 * Created by malk on 2018/12/25.
 */
public class BST<Key extends Comparable<Key>, Value> {
    private Node root; //二叉查找树的根节点

    private class Node {
        private Key key; //键
        private Value value; //值
        private Node left, right; //指向子树的链接
        private int N; //以该 节点为跟的子树中的结点总数

        public Node(Key key, Value value, int N) {
            this.key = key;
            this.value = value;
            this.N = N;
        }

    }

    public int size() {
        return size(root);
    }

    private int size(Node x) {
        if (x == null) return 0;
        else return x.N;
    }

    public Value get(Key key) {
        return get(root, key);
    }

    private Value get(Node x, Key key) {
        if (x == null) return null;
        int cmp = key.compareTo(x.key);
        if (cmp < 0) return get(x.left, key);
        else if (cmp > 0) return get(x.right, key);
        else return x.value;
    }

    public void put(Key key, Value value) {
        root = put(root, key, value);
    }

    private Node put(Node x, Key key, Value value) {
        //如果key存在于以x为根节点的子树中则更新它的值;
        //否则将一key和value为键值对的新节点插入到该子树中
        if (x == null) return new Node(key, value, 1);
        int cmp = key.compareTo(x.key);
        if (cmp < 0) x.left = put(x.left, key, value);
        else if (cmp > 0) x.right = put(x.right, key, value);
        else x.value = value;
        x.N = size(x.left) + size(x.right) + 1;
        return x;
    }

    public Key min() {
        return min(root).key;
    }

    private Node min(Node x) {
        if (x.left == null) return x;
        return min(x.left);
    }

    public Key max() {
        return max(root).key;
    }

    private Node max(Node x) {
        if (x.right == null) return x;
        return max(x.right);
    }

    public Key floor(Key key) {
        Node x = floor(root, key);
        if (x == null) return null;
        return x.key;
    }

    private Node floor(Node x, Key key) {
        if (x == null) return null;
        int cmp = key.compareTo(x.key);
        if (cmp == 0) return x;
        if (cmp < 0) return floor(x.left, key);
        Node t = floor(x.right, key);
        if (t != null) return t;
        else return x;
    }

    public Key select(int k) {
        return select(root, k).key;
    }

    private Node select(Node x, int k) {
        if (x == null) return null;
        int t = size(x.left);
        if (t > k) return select(x.left, k);
        else if (t < k) return select(x.right, k - t - 1);
        else return x;
    }

    public int rank(Key key) {
        return rank(key, root);
    }

    private int rank(Key key, Node x) {
        if (x == null) return 0;
        int cmp = key.compareTo(x.key);
        if (cmp < 0) return rank(key, x.left);
        else if (cmp > 0) return 1 + size(x.left) + rank(key, x.right);
        else return size(x.left);
    }

    public void deleteMin() {
        root = deleteMin(root);
    }

    private Node deleteMin(Node x) {
        if (x.left == null) return x.right;
        x.left = deleteMin(x.left);
        x.N = size(x.left) + size(x.right) + 1;
        return x;
    }

    public void delete(Key key) {
        root = delete(root, key);
    }

    private Node delete(Node x, Key key) {
        if (x == null) return null;
        int cmp = key.compareTo(x.key);
        if (cmp < 0) x.left = delete(x.left, key);
        else if (cmp > 0) x.right = delete(x.right, key);
        else {
            if (x.right == null) return x.left;
            if (x.left == null) return x.right;
            Node t = x;
            x = min(t.right);
            x.right = deleteMin(t.right);
            x.left = t.left;
        }
        x.N = size(x.left) + size(x.right) + 1;
        return x;
    }

    public Iterable<Key> keys() {
        return keys(min(), max());
    }

    public Iterable<Key> keys(Key lo, Key hi) {
        Queue<Key> queue = new Queue<Key>();
        keys(root, queue, lo, hi);
        return queue;

    }

    private void keys(Node x, Queue<Key> queue, Key lo, Key hi) {
        if (x == null) return;
        int cmplo = lo.compareTo(x.key);
        int cmphi = hi.compareTo(x.key);
        if (cmplo < 0) keys(x.left, queue, lo, hi);
        if (cmplo <= 0 && cmphi >= 0) queue.enqueue(x.key);
        if (cmphi > 0) keys(x.right, queue, lo, hi);
    }

}

```

### 3.2.1 基本实现

#### 3.2.1.1 数据表示

#### 3.2.1.2 查找

## 3.3 平衡查找树

### 3.3.1 2-3查找树

#### 3.3.1.1 查找

#### 3.3.1.2 向2-结点中插入新键

### 3.3.2 红黑二叉查找树

### 3.3.3 实现

### 3.3.4 删除操作

### 3.3.5 红黑树的性质


## 3.4 散列表

如果所有的键都是小整数，我们可以用一个数组来实现无序的符号表，将键作为数组的索引而数组中键`i`处存储的就是它对应的值。这样我们就可以快速访问任意键的值。散列表是这种简易方法的扩展并能够处理更加复杂的类型的键。我们需要用算术操作将键转化为数组的索引来访问数组中的键值对。

使用散列的查找算法分为两步。第一步是用散列函数将被查找的键转化为数组的一个索引。理想情况下，不同的键都能转化为不同的索引值。当然，这只是理想情况，所以我们需要面对两个或者多个键都会散列到相同的索引值的情况。因此，散列查找的第二步就是一个`处理碰撞冲突`的过程。解决碰撞的方法有两种：拉链法和线性探测法。



### 3.4.1 散列函数

如果我们有一个能够保存M个键值对的数组，那么我们就需要一个能够将任意键转换为该数组范围内的索引的散列函数。我们要找的散列函数因应该易于计算并且能够均匀分布所有的键，即对于任意键，0到M-1之间的每个整数都相等的可能性与之对应。

散列函数和键的类型有关。严格地说，对于每种类型的键我们都需要一个与之对应的散列函数。


#### 3.4.1.1 典型例子

略

#### 3.4.1.2 正整数

将整数散列最常用方法是`除留余数法`。我们选择大小为素数M的数组，对于任意正整数`k`，计算`k`除以`M`的余数。如果`M`不是素数，我们可能无法利用键中包含的所有信息，这可能导致我们无法均匀地散列散列值。

#### 3.4.1.3 浮点数

如果键是0到1之间的实数，我们可以将它乘以M并四舍五入得到一个0到`M-1`之间的索引值。尽管这个方法很容易理解，但它是有缺陷的，因为这种情况下高位起的作用更大，最低位对散列的结果没有影响（被四舍五入忽略掉了）。修正这个问题的办法是将键表示位二进制然后再使用除留余数法。

#### 3.4.1.4 字符串

除留余数法也可以处理较长的键，例如字符串，我们只需将它们当做大整数即可。例如，下面的代码就能够用除留余数法计算字符串`S`的散列值：

```java
int hash = 0;
for (int i = 0; i < s.length(); i++)
    hash = (R * hash + s.charAt(i)) % M;
```

`Java`的`chartAt()`函数能够返回一个`char`值，即一个非负16位整数。如果R比任何字符的值都大，这种计算相当于将字符串当作一个N位的R进制值，将它除以M并取余。一种叫`[Horner方法](https://en.wikipedia.org/wiki/Horner%27s_method)`的经典算法用N次乘法、加法和取余来计算一个字符串的散列值。只要R最够小，不造成溢出，那么结果就能够如我们所愿，落在0至M-1之内。使用一个较小的素数，例如31，可以保证字符串中所有字符都能发挥作用。`Java`的`String`的默认实现使用了一个类似的方法。（这部分没看明白）

#### 3.4.1.5 组合键

#### 3.4.1.6 Java的约定

每种数据类型都需要相应的散列函数，于是Java令所有数据类型都继承了一个能够返回一个32位整数的`hashCode()`方法。

#### 3.4.1.7 将hashCode()的返回值转换为一个数组索引

因为我们需要的是数组的索引而不是一个32位的整数，我们在实现中会将默认的`hashCode()`方法和除留余数法结合起来产生一个0到`M-1`的整数，方法如下：

```java
private int hash(Key x)
{ return (x.hashCode() & 0x7fffffff) % M;}
```
这段代码会将符号位屏蔽（将一个32位整数变为一个31位非负整数），然后用除留余数法计算它除以M的余数。在使用这样的代码时我们一般会将数组的大小M取为素数以充分利用原散列值的所有位。

#### 3.4.1.8 自定义的hashCode()方法

#### 3.4.1.9 软缓存

如果散列值的计算很耗时，那么我们或许可以将每个键的散列值缓存起来，即在每个键中使用一个hash变量来保存它的`hashCode()`的返回值。第一次调用`hashCode()`方法时，我们需要计算对象的散列值，但之后对`hashCode()`方法的调用会直接返回`hash`变量的值。

### 3.4.2 基于拉链法的散列表

#### 3.4.2.1 散列表的大小

#### 3.4.2.2 删除操作

#### 3.4.2.3 有序性相关的操作

### 3.4.3 基于线性探测法的散列表 

#### 3.4.3.1 删除操作

#### 3.4.3.2 键簇

#### 3.4.3.3 线性探测法的性能分析


### 3.4.4 调整数组大小

#### 3.4.4.1 拉链法

#### 3.4.4.2 均摊分析

### 3.4.5 内存分析