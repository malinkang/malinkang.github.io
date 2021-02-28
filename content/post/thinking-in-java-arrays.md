---
title: 《Java编程思想》第16章数组
date: 2013-06-25 13:39:36
tags: [Thinking in Java]
---

## 1.数组为什么特殊

数组与其他种类的容器之间的区别有三方面：效率、类型和保存基本类型的能力。在`Java`中，数组是一种效率最高的存储和随机访问对象引用序列的方式。数组就是一个简单的线性序列，这使得元素访问非常快速。但是为这种速度所付出的代价是数组对象的大小被固定，并且在其生命周期中不可改变。

数组之所以优于泛型之前的容器，就是因为你可以创建一个数组去持有某种具体类型。这意味着你可以通过编译期检查，来防止插入错误类型和抽取不当类型。

数组可以持有基本类型，而泛型之前的容器则不能，但是有了泛型，容器就可以指定并检查它们所持有对象的类型，并且有了自动包装机制，容器看起来还能够持有基本类型。

```java
public class ContainerComparison {
    public static void main(String[] args) {
        BerylliumSphere[] spheres = new BerylliumSphere[10];
        for (int i = 0; i < 5; i++) {
            spheres[i] = new BerylliumSphere();
        }
        System.out.println(Arrays.toString(spheres));
        System.out.println(spheres[4]);
        List<BerylliumSphere> sphereList = new ArrayList<BerylliumSphere>();
        for (int i = 0; i < 5; i++) {
            sphereList.add(new BerylliumSphere());
        }
        System.out.println(sphereList);
        System.out.println(sphereList.get(4));
        int[] integers = {0,1,2,3,4,5};
        System.out.println(Arrays.toString(integers));
        System.out.println(integers[4]);
        List<Integer> intList =new ArrayList<Integer>(Arrays.asList(0,1,2,3,4,5));
        intList.add(97);
        System.out.println(intList);
        System.out.println(intList.get(4));
    }

}
/*
输出
[Sphere 0, Sphere 1, Sphere 2, Sphere 3, Sphere 4, null, null, null, null, null]
Sphere 4
[Sphere 5, Sphere 6, Sphere 7, Sphere 8, Sphere 9]
Sphere 9
[0, 1, 2, 3, 4, 5]
4
[0, 1, 2, 3, 4, 5, 97]
4
 */
```

## 2.数组是第一级对象

无论使用哪种类型的数组，数组标识符其实只是一个引用，指向在堆中创建的一个真实对象，**这个对象用以保存指向其他对象的引用**。

对象数组和基本类型数组在使用上几乎是相同的；唯一的区别就是对象数组保存的是引用，基本类型数组直接保存基本类型的值。

```java
public class ArrayOptions {
    public static void main(String[] args) {
        BerylliumSphere[] a;
        BerylliumSphere[] b = new BerylliumSphere[5];
        System.out.println("b: " + Arrays.toString(b));
        BerylliumSphere[] c = new BerylliumSphere[4];
        for (int i = 0; i < c.length; i++) {
            if (c[i] == null) {
                c[i] = new BerylliumSphere();
            }
        }
        BerylliumSphere[] d = {new BerylliumSphere(), new BerylliumSphere()
                , new BerylliumSphere()};
        a = new BerylliumSphere[]{
                new BerylliumSphere(), new BerylliumSphere()
        };
        System.out.println("a.length = " + a.length);
        System.out.println("b.length = " + b.length);
        System.out.println("c.length = " + c.length);
        System.out.println("d.length = " + d.length);
        a = d;
        System.out.println("a.length = " + a.length);
        int[] e;
        int[] f = new int[5];
        System.out.println("f: " + Arrays.toString(f));
        int[] g = new int[4];
        for (int i = 0; i < g.length; i++) {
            g[i] = i * i;
        }
        int[] h = {11, 47, 93};
        // 编译失败
        // System.out.println("e.length = "+e.length);
        System.out.println("f.length = " + f.length);
        System.out.println("g.length = " + g.length);
        System.out.println("h.length = " + h.length);
        e = h;
        System.out.println("e.length = " + e.length);
        e = new int[]{1,2};
        System.out.println("e.length = " + e.length);
    }
}
/*
输出
b: [null, null, null, null, null]
a.length = 2
b.length = 5
c.length = 4
d.length = 3
a.length = 3
f: [0, 0, 0, 0, 0]
f.length = 5
g.length = 4
h.length = 3
e.length = 3
e.length = 2
 */
```

## 3.返回一个数组

```java
public class IceCream {
    private static Random rand = new Random(47);
    static final String[] FLAVORS = {
            "Chocolate", "Strawberry", "Vanilla Fudge Swirl",
            "Mint Chip", "Mocha Almond Fudge", "Rum Raisin",
            "Praline Cream", "Mud Pie"
    };

    public static String[] flavorSet(int n) {
        if (n > FLAVORS.length)
            throw new IllegalArgumentException("Set too big");
        String[] results = new String[n];
        boolean[] picked = new boolean[FLAVORS.length];
        for (int i = 0; i < n; i++) {
            int t;
            do {
                t = rand.nextInt(FLAVORS.length);
            } while (picked[t]);
            results[i] = FLAVORS[t];
            picked[t] = true;
        }
        return results;
    }

    public static void main(String[] args) {
        for (int i = 0; i < 7; i++) {
            System.out.println(Arrays.toString(flavorSet(3)));
        }
    }
}
/*
输出
[Rum Raisin, Mint Chip, Mocha Almond Fudge]
[Chocolate, Strawberry, Mocha Almond Fudge]
[Strawberry, Mint Chip, Mocha Almond Fudge]
[Rum Raisin, Vanilla Fudge Swirl, Mud Pie]
[Vanilla Fudge Swirl, Chocolate, Mocha Almond Fudge]
[Praline Cream, Strawberry, Mocha Almond Fudge]
[Mocha Almond Fudge, Strawberry, Mint Chip]
 */
```

## 4.多维数组

```java
public class MultidimensionalPrimitiveArray {
    public static void main(String[] args) {
        int[][] a = {
                {1, 2, 3},
                {4, 5, 6}
        };
        System.out.println(Arrays.deepToString(a));
    }
}
/*
[[1, 2, 3], [4, 5, 6]]
 */
```

`Arrays.deepToString()`方法可以将多维数组转换为多个`String`。

```java
public class ThreeDWithNew {
    public static void main(String[] args) {
        int[][][] a = new int[2][2][4];
        System.out.println(Arrays.deepToString(a));
    }
}
/*
[[[0, 0, 0, 0], [0, 0, 0, 0]], [[0, 0, 0, 0], [0, 0, 0, 0]]]

 */
```

基本类型数组的值在不进行显式初始化的情况下，会被自动初始化。对象数组会被初始化为`null`。

数组中构成矩阵的每个向量都可以具有任意的长度，这被称为`粗糙数组`。

```java
public class RaggedArray {
    public static void main(String[] args) {
        Random rand = new Random(47);
        int[][][] a = new int[rand.nextInt(7)][][];
        for (int i = 0; i < a.length; i++) {
            //第二纬的长度
            a[i] = new int[rand.nextInt(5)][];
            for (int j = 0; j < a[i].length; j++) {
                //第三纬的长度
                a[i][j] = new int[rand.nextInt(5)];
            }
        }
        System.out.println(Arrays.deepToString(a));
    }
}
/*
[[], [[0], [0], [0, 0, 0, 0]], [[], [0, 0], [0, 0]], [[0, 0, 0], [0], [0, 0, 0, 0]],
[[0, 0, 0], [0, 0, 0], [0], []], [[0], [], [0]]]

 */
```

```java
public class MultidimensionalObjectArrays {
    public static void main(String[] args) {
        BerylliumSphere[][] spheres = {
                {new BerylliumSphere(), new BerylliumSphere()},
                {new BerylliumSphere(), new BerylliumSphere(),
                        new BerylliumSphere(), new BerylliumSphere()},
                {new BerylliumSphere(), new BerylliumSphere(),
                        new BerylliumSphere(), new BerylliumSphere(),
                        new BerylliumSphere(), new BerylliumSphere(),
                        new BerylliumSphere(), new BerylliumSphere()},
        };
        System.out.println(Arrays.deepToString(spheres));
    }
}
/*
输出
[[Sphere 0, Sphere 1], [Sphere 2, Sphere 3, Sphere 4, Sphere 5],
[Sphere 6, Sphere 7, Sphere 8, Sphere 9, Sphere 10, Sphere 11, Sphere 12, Sphere 13]]
 */
```

自动包装机制对数组初始化器也起作用：

```java
public class AutoboxingArrays {
    public static void main(String[] args) {
        Integer[][] a = { //自动装箱
                {1, 2, 3, 4, 5, 6, 7, 8, 9, 10},
                {21,22,23,24,25,26,27,28,29,30},
                {51,52,53,54,55,56,57,58,59,60},
                {71,72,73,74,75,76,77,78,79,80}
        };
        System.out.println(Arrays.deepToString(a));
    }
}
/*
输出
[[1, 2, 3, 4, 5, 6, 7, 8, 9, 10],
[21, 22, 23, 24, 25, 26, 27, 28, 29, 30],
[51, 52, 53, 54, 55, 56, 57, 58, 59, 60],
[71, 72, 73, 74, 75, 76, 77, 78, 79, 80]]

 */
```

下面的示例展示了可以如何逐个地、部分地构建一个非基本类型的对象数组：

```java
public class AssemblingMultidimensionalArrays {
    public static void main(String[] args) {
        Integer[][] a ;
        a = new Integer[3][];
        for (int i = 0; i <a.length ; i++) {
            a[i]=new Integer[3];
            for (int j = 0; j < a[i].length; j++) {
                a[i][j] = i*j;
            }
        }
        System.out.println(Arrays.deepToString(a));
    }
}
/*
[[0, 0, 0], [0, 1, 2], [0, 2, 4]]
 */
```

```java
public class MultiDimWrapperArray {
    public static void main(String[] args) {
        Integer[][] a1 = {
                {1,2,3},
                {4,5,6}
        };
        Double[][][] a2 = {
                {{1.1,2.2},{3.3,4.4}},
                {{5.5,6.6},{7.7,8.8}},
                {{9.9,1.2},{2.3,3.4}}
        };
        String[][] a3 = {
                {"The","Quick","Sly","Fox"},
                {"Jumped","Over"},
                {"The","Lazy","Brown","Dog","and","friend"}
        };
        System.out.println("a1: " + Arrays.deepToString(a1));
        System.out.println("a2: " + Arrays.deepToString(a2));
        System.out.println("a3: " + Arrays.deepToString(a3));
    }
}
/*
a1: [[1, 2, 3], [4, 5, 6]]
a2: [[[1.1, 2.2], [3.3, 4.4]], [[5.5, 6.6], [7.7, 8.8]], [[9.9, 1.2], [2.3, 3.4]]]
a3: [[The, Quick, Sly, Fox], [Jumped, Over], [The, Lazy, Brown, Dog, and, friend]]
 */
```

## 5.数组与泛型

通常，数组与泛型不能很好地结合。你不能实例化具有参数化类型的数组：

```java
Pell<Banana>[] peels = new Pell<Banana>[10];//错误
```

擦除会移除参数类型信息，而数组必须知道它们所持有的确切类型，以强制保证类型安全。

但是，你可以参数化数组本身的类型：

```java
public class ClassParameter<T> {
    public T[] f(T[] arg){
        return arg;
    }
}
public class MethodParameter {
    public static <T> T[] f(T[] arg){
        return arg;
    }
}
//使用参数化方法而不使用参数化类的方便之处在于：你不必为需要应用的
//每种不同的类型都使用一个参数去实例化这个类，并且你可以将其定义为静态的
public class ParameterizedArrayType {
    public static void main(String[] args) {
        Integer[] ints = {1,2,3,4,5};
        Double[] doubles = {1.1,2.2,3.3,4.4,5.5};
        Integer[] ints2 = new ClassParameter<Integer>().f(ints);
        Double[] doubles2 = new ClassParameter<Double>().f(doubles);
        ints2 = MethodParameter.f(ints);
        doubles2 = MethodParameter.f(doubles);
    }
}
```

编译器不允许实例化泛型数组，但是，它允许你创建对这种数组的引用。例如：

```java
List<String>[] ls;
```

这条语句可以顺利地通过编译器而不报任何错误，而且，尽管你不能创建实际的持有泛型的数组对象，但是你可以创建非泛型的数组，然后将其转型：

```java
public class ArrayOfGenerics {
    public static void main(String[] args) {
        List<String>[] ls;
        List[] la = new List[10];
        ls = la;
        ls[0] = new ArrayList<String>();
        //编译出错
        //ls[1] = new ArrayList<Integer>();
        Object[] objects = ls;
        objects[1] = new ArrayList<Integer>();
        List<BerylliumSphere>[] spheres = new List[10];
        for (int i = 0; i < spheres.length; i++) {
            spheres[i]=new ArrayList<BerylliumSphere>();
        }
    }
}
```

一般而言，你会发现泛型在类或方法的边界处很有效，而在类或方法的内部，擦除通常会使泛型变得不适用。例如，你不能创建泛型数组：

```java
public class ArrayOfGenericType<T> {
    T[] array;

    public ArrayOfGenericType(int size) {
        //编译错误
        //  array = new T[size];
        array = (T[]) new Object[size];
    }
//编译错误
//    public <U> U[] makeArray(){
//        return new U[10];
//    }
}
```

## 6.创建测试数据

通常，在试验数组和程序时，能够很方便地生成填充了测试数据的数组，将会很有帮助。

### 6.1 Arrays.fill\(\)

Java标准类库`Arrays`有一个作用十分有限的`fill()`方法：只能用同一个值填充各个位置，而针对对象而言，就是复制同一个引用进行填充。

```java
public class FillingArrays {
    public static void main(String[] args) {
        int size = 6;
        boolean[] a1 = new boolean[size];
        byte[] a2 = new byte[size];
        char[] a3 = new char[size];
        short[] a4 = new short[size];
        int[] a5 = new int[size];
        long[] a6 = new long[size];
        float[] a7 = new float[size];
        double[] a8 = new double[size];
        String[] a9 = new String[size];
        Arrays.fill(a1, true);
        System.out.println("a1 = " + Arrays.toString(a1));
        Arrays.fill(a2, (byte) 11);
        System.out.println("a2 = " + Arrays.toString(a2));
        Arrays.fill(a3, 'x');
        System.out.println("a3 = " + Arrays.toString(a3));
        Arrays.fill(a4, (short) 17);
        System.out.println("a4 = " + Arrays.toString(a4));
        Arrays.fill(a5, 19);
        System.out.println("a5 = " + Arrays.toString(a5));
        Arrays.fill(a6, 23);
        System.out.println("a6 = " + Arrays.toString(a6));
        Arrays.fill(a7, 29);
        System.out.println("a7 = " + Arrays.toString(a7));
        Arrays.fill(a8, 47);
        System.out.println("a8 = " + Arrays.toString(a8));
        Arrays.fill(a9, "Hello");
        System.out.println("a9 = " + Arrays.toString(a9));
        //3开始索引,5结束索引
        Arrays.fill(a9, 3, 5, "World");
        System.out.println("a9 = " + Arrays.toString(a9));
    }
}
/*
输出
a1 = [true, true, true, true, true, true]
a2 = [11, 11, 11, 11, 11, 11]
a3 = [x, x, x, x, x, x]
a4 = [17, 17, 17, 17, 17, 17]
a5 = [19, 19, 19, 19, 19, 19]
a6 = [23, 23, 23, 23, 23, 23]
a7 = [29.0, 29.0, 29.0, 29.0, 29.0, 29.0]
a8 = [47.0, 47.0, 47.0, 47.0, 47.0, 47.0]
a9 = [Hello, Hello, Hello, Hello, Hello, Hello]
a9 = [Hello, Hello, Hello, World, World, Hello]
 */
```

### 6.2 数据生成器

```java
public interface Generator<T>  {
     T next();
}
public class CountingGenerator {
    public static class Boolean implements Generator<java.lang.Boolean> {
        private boolean value = false;

        public java.lang.Boolean next() {
            value = !value;
            return value;
        }
    }

    public static class Byte implements Generator<java.lang.Byte> {
        private byte value = 0;

        public java.lang.Byte next() {
            return value++;
        }
    }

    static char[] chars = ("abcdefghijklmnopqrstuvwxyz" +
            "ABCDEFGHIJKLMNOPQRSTUVWXYZ").toCharArray();

    public static class Character implements Generator<java.lang.Character> {
        int index = -1;

        public java.lang.Character next() {
            index = (index + 1) % chars.length;
            return chars[index];
        }
    }

    public static class String implements Generator<java.lang.String>{
        private int length = 8;
        Generator<java.lang.Character> cg= new Character();
        public String(){}
        public String(int length){
            this.length = length;
        }
        public java.lang.String next() {
            char[] buf = new char[length];
            for (int i = 0; i < length; i++) {
                buf[i] = cg.next();
            }
            return new java.lang.String(buf);
        }
    }
    public static class Short implements Generator<java.lang.Short>{
        private short value = 0;

        public java.lang.Short next() {
            return value++;
        }
    }
    public static class Integer implements Generator<java.lang.Integer>{
        private int value = 0;

        public java.lang.Integer next() {
            return value++;
        }
    }
    public static class Long implements Generator<java.lang.Long>{
        private long value = 0;
        public java.lang.Long next() {
            return value++;
        }
    }

    public static class Float implements Generator<java.lang.Float>{
        private float value =0;
        public java.lang.Float next() {
            float result = value;
            value += 1.0;
            return result;
        }
    }


    public static class Double implements Generator<java.lang.Double>{
        private double value =0;
        public java.lang.Double next() {
            double result = value;
            value += 1.0;
            return result;        }
    }

}
```

```java
public class GeneratorsTest {
    public static int size = 10;
    public static void test(Class<?> sourroundingClass){
        for (Class<?> type:sourroundingClass.getClasses()){
            System.out.print(type.getSimpleName()+":");
            try{
                Generator<?> g = (Generator<?>) type.newInstance();
                for (int i = 0; i < size; i++) {
                    System.out.print(g.next()+" ");
                }
                System.out.println();
            }catch (Exception e){
                throw new RuntimeException(e);
            }
        }
    }

    public static void main(String[] args) {
        test(CountingGenerator.class);
    }
}
/*
输出
Double:0.0 1.0 2.0 3.0 4.0 5.0 6.0 7.0 8.0 9.0 
Float:0.0 1.0 2.0 3.0 4.0 5.0 6.0 7.0 8.0 9.0 
Long:0 1 2 3 4 5 6 7 8 9 
Integer:0 1 2 3 4 5 6 7 8 9 
Short:0 1 2 3 4 5 6 7 8 9 
String:abcdefgh ijklmnop qrstuvwx yzABCDEF GHIJKLMN OPQRSTUV WXYZabcd efghijkl mnopqrst uvwxyzAB 
Character:a b c d e f g h i j 
Byte:0 1 2 3 4 5 6 7 8 9 
Boolean:true false true false true false true false true false 
 */
```

```java
public class RandomGenerator {
    private static Random r = new Random(47);

    public static class Boolean implements Generator<java.lang.Boolean> {
        public java.lang.Boolean next() {
            return r.nextBoolean();
        }
    }

    public static class Byte implements Generator<java.lang.Byte> {
        public java.lang.Byte next() {
            return (byte) r.nextInt();
        }
    }


    public static class Character implements Generator<java.lang.Character> {
        public java.lang.Character next() {
            return CountingGenerator.chars[r.nextInt(CountingGenerator.chars.length)];
        }
    }

    public static class String extends CountingGenerator.String {
        {cg = new Character();}
        public String() {}
        public String(int length) {
            super(length);
        }
    }

    public static class Short implements Generator<java.lang.Short> {
        public java.lang.Short next() {
            return (short) r.nextInt();
        }
    }

    public static class Integer implements Generator<java.lang.Integer> {
        private int mod = 10000;
        public Integer() {}

        public Integer(int modulo) {
            mod = modulo;
        }
        public java.lang.Integer next() {
            return r.nextInt(mod);
        }
    }

    public static class Long implements Generator<java.lang.Long> {
        private int mod = 10000;
        public Long() {}
        public Long(int modulo) {
            mod = modulo;
        }

        public java.lang.Long next() {
            return new java.lang.Long(r.nextInt(mod));
        }
    }

    public static class Float implements Generator<java.lang.Float> {
        public java.lang.Float next() {
            int trimmed = Math.round(r.nextFloat() * 100);
            return ((float) trimmed) / 100;
        }
    }


    public static class Double implements Generator<java.lang.Double> {
        public java.lang.Double next() {
            long trimmed = Math.round(r.nextDouble() * 100);
            return ((double) trimmed) / 100;        
        }
    }
}
```

```java
public class RandomGeneratorsTest {
    public static void main(String[] args) {
        GeneratorsTest.test(RandomGenerator.class);
    }
}
/*
输出
Double:0.73 0.53 0.16 0.19 0.52 0.27 0.26 0.05 0.8 0.76
Float:0.53 0.16 0.53 0.4 0.49 0.25 0.8 0.11 0.02 0.8
Long:7674 8804 8950 7826 4322 896 8033 2984 2344 5810
Integer:8303 3141 7138 6012 9966 8689 7185 6992 5746 3976
Short:3358 20592 284 26791 12834 -8092 13656 29324 -1423 5327
String:bkInaMes btWHkjUr UkZPgwsq PzDyCyRF JQAHxxHv HqXumcXZ JoogoYWM NvqeuTpn Xsgqiaxx EAJJmzMs
Character:s l J r L v p f F v
Byte:-70 78 4 -83 113 99 -125 61 -110 -121
Boolean:false true true true true false false true true false
 */
```

### 6.3 从Generator中创建数组

## 7.Arrays实用功能

在`java.util`类库中可以找到`Arrays`类，它有一套用于数组的`static`实用方法，其中有6个基本方法：

* equals用于比较两个数组是否相等；
* fill\(\)
* sort\(\)用于对数组排序；
* binary\(\)用于在已经排序的数组中查找元素；
* toString\(\)产生数组的`String`表示。
* hashCode\(\)产生数组的散列码

此外，`Arrays.asList()`接受任意的序列或数组作为其参数，并将其转变为`List`容器。

### 7.1复制数组

`Java`标准类库提供有`static`方法`System.arraycopy()`，用它复制数组比用`for`循环复制要快很多。`System.arraycopy()`针对所有类型做了重载。

`arraycopy()`需要的参数有：

* 源数组
* 从源数组中的什么问题开始复制的偏移量
* 目标数组
* 目标数组的什么位置开始复制的偏移量
* 需要复制的元素个数

```java
public class CopyingArrays {
    public static void main(String[] args) {
        int[] i = new int[8];
        int[] j = new int[10];
        Arrays.fill(i, 47);
        Arrays.fill(j, 99);
        System.out.println("i = " + Arrays.toString(i));
        System.out.println("j = " + Arrays.toString(j));
        System.arraycopy(i,0,j,0,i.length);
        System.out.println("j = "+Arrays.toString(j));
        int[] k = new int[5];
        Arrays.fill(k,103);
        System.arraycopy(i,0,k,0,k.length);
        System.out.println("k = " + Arrays.toString(k) );
        Arrays.fill(k,103);
        System.arraycopy(k,0,i,0,k.length);
        System.out.println("i = " + Arrays.toString(i));
        //Objects:
        Integer[] u = new Integer[10];
        Integer[] v = new Integer[5];
        Arrays.fill(u,new Integer(47));
        Arrays.fill(v,new Integer(99));
        System.out.println("u = " + Arrays.toString(u));
        System.out.println("v = " + Arrays.toString(v));
        System.arraycopy(v,0,u,u.length/2,v.length);
        System.out.println("u = " + Arrays.toString(u));
    }
}
/*
输出
i = [47, 47, 47, 47, 47, 47, 47, 47]
j = [99, 99, 99, 99, 99, 99, 99, 99, 99, 99]
j = [47, 47, 47, 47, 47, 47, 47, 47, 99, 99]
k = [47, 47, 47, 47, 47]
i = [103, 103, 103, 103, 103, 47, 47, 47]
u = [47, 47, 47, 47, 47, 47, 47, 47, 47, 47]
v = [99, 99, 99, 99, 99]
u = [47, 47, 47, 47, 47, 99, 99, 99, 99, 99]
 */
```

这个例子说明基本类型数组与对象数组都可以复制。然而，如果复制对象数组，那么只是复制了对象的引用，而不是对象本身的拷贝。这被称作`浅复制（shallow copy）`。

`Syste.arraycopy()`不会执行自动包装和自动拆包，两个数组必须具有相同的确切类型。

### 7.2数组的比较

`Arrays`类提供了重载后的`equals`方法，用来比较整个数组。同样，此方法针对所有基本类型与`Object`都做了重载。数组相等的条件是元素个数必须相等，并且对应位置的元素也相等，这可以通过对每个元素使用`equals()`作比较来判断，对于基本类型，需要使用基本类型的包装器类的`equals()`方法。

```java
public class ComparingArrays {
    public static void main(String[] args) {
        int[] a1 = new int[10];
        int[] a2 = new int[10];
        Arrays.fill(a1, 47);
        Arrays.fill(a2, 47);
        System.out.println(Arrays.equals(a1, a2));
        a2[3] = 11;
        System.out.println(Arrays.equals(a1, a2));
        String[] s1 = new String[4];
        Arrays.fill(s1, "Hi");
        String[] s2 = {new String("Hi"), new String("Hi"),
                new String("Hi"), new String("Hi")
        };
        System.out.println(Arrays.equals(s1, s2));
    }
}
/*
输出
true
false
true
 */
```

### 7.3数组元素的比较

排序必须根据对象的实际类型执行比较操作。一种自然的解决方案是位每种不同的类型各编写一个不同的排序方法，但是这样的代码难以被新的类型所复用。

`Java`有两种方式来提供比较功能。第一是实现`java.lang.Comparable`接口，使你的类具有"天生"的比较能力。此接口很简单，只有`compareTo()`一个方法。此方法接收另一个`Object`为参数，如果当前对象小于参数则返回负值，如果相等则返回零，如果当前对象大于参数则返回正值。

也可以编写自己的`Comparator`。

### 7.4数组排序

使用内置的排序方法，就可以对任意的基本类型数组排序；也可以对任意的对象数组进行排序，只要该对象实现了`Comparable`接口或具有相关联的`Comparator`。

### 7.5 在已排序的数组中查找

如果数组已经排好序了，就可以使用`Arrays.binarySearch()`执行快速查找。如果要对未排序的数组使用`binarySearch()`，那么将产生不可预料的结果。

