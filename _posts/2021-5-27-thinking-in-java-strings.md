---
title: 《Java编程思想》第13章字符串
date: 2013-06-04 13:39:36
tags: [Thinking in Java]
---
# 第13章 字符串

## 13.1 不可变String

```java
public class Immutable {
    public static String upcase(String s){
        return s.toUpperCase();
    }

    public static void main(String[] args) {
        String q = "howdy";
        System.out.println(q);
        String qq = upcase(q);
        System.out.println(qq);
        System.out.println(q);
    }
}
```

当把`q`传给`upcase()`方法时，实际传递的是引用的一份拷贝。其实，**每当把**`String`**对象作为方法的参数时，都会复制一份引用**，而该引用所指的对象其实一直待在单一的物理位置上，从未动过。

回到`upcase()`的定义，传入其中的引用有了名字`s`,只有`upcase()`运行的时候，局部引用`s`才存在。一旦`upcase()`运行结束，`s`就消失了。当然了，`upcase()`的返回值，其实只是最终结果的引用。这足以说明，`upcase()`返回的引用已经指向了一个新的对象，而原本的`q`则还在原地。

## 13.2 重载“+”与StringBuilder

操作符“+”可以用来连接`String`。

```java
public class Concatenation {
    public static void main(String[] args) {
        String mango =  "mango";
        String s = "abc"+mango+"def"+47;
        System.out.println(s);//abcmangodef47
    }
}
```

用`JDK`自带的工具`javap`来反编译以上代码。命令如下

```text
javap -c Concatenation
```

`-c`标志表示将生成`JVM`字节码。剔除不感兴趣的部分，有了以下的字节码。

```java
public static void main(java.lang.String[]);
Code:
   0: ldc           #2                  // String mango
   2: astore_1
   3: new           #3                  // class java/lang/StringBuilder
   6: dup
   7: invokespecial #4                  // Method java/lang/StringBuilder."<init>":()V
  10: ldc           #5                  // String abc
  12: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
  15: aload_1
  16: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
  19: ldc           #7                  // String def
  21: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
  24: bipush        47
  26: invokevirtual #8                  // Method java/lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
  29: invokevirtual #9                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
  32: astore_2
  33: getstatic     #10                 // Field java/lang/System.out:Ljava/io/PrintStream;
  36: aload_2
  37: invokevirtual #11                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
  40: return
```

`dup`与`invokevirtual`语句相当于`Java`虚拟机上的汇编语句。

在这个例子中，编译器创建了一个`StringBuilder`对象，用以构建最终的`String`，并为每个字符串调用一次`StringBuilder`的`append()`方法，总计四次。最后调用`toString()`生成结果，并存为`s`。

```java
public class WhitherStringBuilder {
    public String implicit(String[] fields) {
        String result = "";
        for (int i = 0; i < fields.length; i++) {
            result += fields[i];
        }
        return result;
    }

    public String explicit(String[] fields) {
        StringBuilder result = new StringBuilder();
        for (int i = 0; i < fields.length; i++) {
            result.append(fields[i]);
        }
        return result.toString();
    }

}
```

运行`javap -c WhitherStringBuilder`。

`implicit()`方法对应的字节码：

```java
public java.lang.String implicit(java.lang.String[]);
Code:
   0: ldc           #2                  // String
   2: astore_2
   3: iconst_0
   4: istore_3
   5: iload_3
   6: aload_1
   7: arraylength
   8: if_icmpge     38
  11: new           #3                  // class java/lang/StringBuilder
  14: dup
  15: invokespecial #4                  // Method java/lang/StringBuilder."<init>":()V
  18: aload_2
  19: invokevirtual #5                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
  22: aload_1
  23: iload_3
  24: aaload
  25: invokevirtual #5                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
  28: invokevirtual #6                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
  31: astore_2
  32: iinc          3, 1
  35: goto          5
  38: aload_2
  39: areturn
```

从第8行到第35行构成了一个循环体。第8行：对堆栈中的操作数进行“大于或等于的整数比较运算”，循环结束时跳到第38行。第35行：返回循环体的起始点第5行。要注意的重点是：`StringBuilder`是在循环之内构造的，这意味着每经过循环一次，就会创建一个新的`StringBuilder`对象。

`explicit()`方法对应的字节码：

```java
public java.lang.String explicit(java.lang.String[]);
Code:
   0: new           #3                  // class java/lang/StringBuilder
   3: dup
   4: invokespecial #4                  // Method java/lang/StringBuilder."<init>":()V
   7: astore_2
   8: iconst_0
   9: istore_3
  10: iload_3
  11: aload_1
  12: arraylength
  13: if_icmpge     30
  16: aload_2
  17: aload_1
  18: iload_3
  19: aaload
  20: invokevirtual #5                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
  23: pop
  24: iinc          3, 1
  27: goto          10
  30: aload_2
  31: invokevirtual #6                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
  34: areturn
```

可以看到，不仅循环部分的代码更简短、更简单，并且它只生成了一个`StringBuilder`对象。显式地创建`StringBuilder`还允许你预先指定`StringBuilder`的大小，避免多次重新分配缓存。

因此，当为一个类编写`toString()`方法时，如果字符串操作比较简单，那就可以信赖编译器，他会为你合理地构造最终的字符串结果。但是，如果你要在`toString()`方法中使用循环，那么最好自己创建一个`StringBuilder`对象，用它来构造最终的结果。

```java
public class UsingStringBuilder {
    public static Random rand = new Random(47);

    @Override
    public String toString() {
        StringBuilder result = new StringBuilder("[");
        for (int i = 0; i < 25; i++) {
            result.append(rand.nextInt(100));
            result.append(", ");
        }
        result.delete(result.length() - 2, result.length());
        result.append("]");
        return result.toString();
    }
    public static void main(String[] args) {
        UsingStringBuilder usb = new UsingStringBuilder();
        System.out.println(usb);
    }
}
/*
[58, 55, 93, 61, 61, 29, 68, 0, 22, 7, 88, 28, 51, 89, 
9, 78, 98, 61, 20, 58, 16, 40, 11, 22, 4]
 */
```

## 13.3 无意识的递归

`Java`中的每个类从根本上都是继承自`Object`，标准容器类自然也不例外。因此容器类都有`toString()`方法，覆写了该方法，使得它生成的`String`结果能够表达容器自身，以及容器所包含的对象。例如`ArrayList.toString()`，它会遍历`ArrayList`中包含的所有对象，调用每个元素上的`toString()`方法：

```java
public class ArrayListDisplay {
    public static void main(String[] args) {
        ArrayList<Coffee> coffees = new ArrayList<Coffee>();
        for (Coffee coffee : new CoffeeGenerator(10)) {
            coffees.add(coffee);
        }
        System.out.println(coffees);
    }
}
/*
[Americano 0, Latte 1, Americano 2, Mocha 3, Mocha 4, Breve 5,
Americano 6, Latte 7, Cappuccino 8, Cappuccino 9]
 */
```

如果希望`toString()`方法打印出对象的内存地址，也许会考虑使用`this`关键字：

```java
public class InfiniteRecursion {
    @Override
    public String toString() {
        return "InfiniteRecursion address: " + this + "\n";
    }
    public static void main(String[] args) {
        List<InfiniteRecursion> v = new ArrayList<InfiniteRecursion>();
        for (int i = 0; i < 10; i++) {
            v.add(new InfiniteRecursion());
        }
        System.out.println(v);
    }
}
```

执行代码会得到一串非常长的异常。这里发生了自动类型转换，由`InfiniteRecursion`类型转换成`String`类型。因为编译器看到一个`String`对象后面跟着一个“+”，而在后面的对象不是`String`，于是编译器试着将`this`转换成一个`String`。调用`this`上的`toString()`方法进行转换，于是就发生了递归调用。所以不该使用`this`，而是应该调用`super.toString()`方法。

## 13.4 String上的操作

## 13.5 格式化输出

### 13.5.1 printf\(\)

```java
System.out.printf("Row 1:[%d %f] \n",10,2.5);// Row 1:[10 2.500000]
```

### 13.5.2 System.out.format\(\)

`Java SE5`引入的`format`方法可用于`PrintStream`或`PrintWriter`对象，其中也包括`System.out`对象。

```java
public class SimpleFormat {
    public static void main(String[] args) {
        int x = 5;
        double y = 5.332542;
        // The old way:
        System.out.println("Row 1: [" + x + " " + y + "]");
        // The new way:
        System.out.format("Row 1: [%d %f]\n", x, y);
        // OR
        System.out.printf("Row 1: [%d %f]\n", x, y);
    }
}
/*
Row 1: [5 5.332542]
Row 1: [5 5.332542]
Row 1: [5 5.332542]
 */
```

可以看到，`format()`与`printf()`是等价的。

### 13.5.3 Formatter类

```java
public class Turtle {
    private String name;
    private Formatter f;

    public Turtle(String name, Formatter f) {
        this.name = name;
        this.f = f;
    }

    public void move(int x, int y) {
        f.format("%s The Turtle is at (%d,%d)\n", name, x, y);
    }

    public static void main(String[] args) {
        PrintStream outAlias =System.out;
        Turtle tommy = new Turtle("Tommy",new Formatter(System.out));
        Turtle terry = new Turtle("Terry",new Formatter(outAlias));
        tommy.move(0,0);
        terry.move(4,8);
        tommy.move(3,4);
        terry.move(2,5);
        tommy.move(3,3);
        terry.move(3,3);
    }

}
/*
Tommy The Turtle is at (0,0)
Terry The Turtle is at (4,8)
Tommy The Turtle is at (3,4)
Terry The Turtle is at (2,5)
Tommy The Turtle is at (3,3)
Terry The Turtle is at (3,3)
 */
```

### 13.5.4 格式化说明符

在插入数据时，如果想要控制空格与对齐，你需要更精细复杂的格式修饰符，一下是其抽象的语法：

```text
%[argument_index$][flags][width][precision]conversion
```

通过指定`width`来实现控制一个域的最小尺寸。`Formatter`对象通过在必要时添加空格，来确保一个域至少达到某个长度。在默认情况下，数据时右对齐，不过可以通过使用“-”标志来改变对齐方向。

与`width`相对的是`precision`,它用来指明最大尺寸。`width`可以用用于各种类型的数据转换，并且其行为方式都一样。`precision`则不然，不是所有类型的数据都能使用`precision`，而且，应用于不同类型的数据转换时，`precision`的意义也不同。在将`precision`应用于`String`时，它表示打印`String`时输出字符的最大数量。而在将`precision`应用于浮点数时，它表示小数部分要显示出来的位数（默认是6位小数），如果小数位过多则舍入，太少则在尾部补零。由于整数没有小数部分，所以`precision`无法应用于整数，如果你对整数应用`precision`，则会触发异常。

```java
public class Receipt {
    private double total = 0;
    private Formatter f = new Formatter(System.out);

    public void printTitle() {
        f.format("%-15s %5s %10s \n", "Item", "Qty", "Price");
        f.format("%-15s %5s %10s \n", "----", "---", "-----");
    }

    public void print(String name, int qty, double price) {
        f.format("%-15.15s %5d %10.2f \n", name, qty, price);
        total += price;
    }

    public void printTotal() {
        f.format("%-15s %5s %10.2f \n", "Tax", "", total * 0.06);
        f.format("%-15s %5s %10s \n", "Tax", "", "-----");
        f.format("%-15s %5s %10.2f \n", "Total", "", total * 1.06);
    }

    public static void main(String[] args) {
        Receipt receipt = new Receipt();
        receipt.printTitle();
        receipt.print("Jack's Magic Beans",4,4.25);
        receipt.print("Princess Peas",3,5.1);
        receipt.print("Three Bears Porridge",1,14.29);
        receipt.printTotal();
    }
}
/*
Item              Qty      Price
----              ---      -----
Jack's Magic Be     4       4.25
Princess Peas       3       5.10
Three Bears Por     1      14.29
Tax                         1.42
Tax                        -----
Total                      25.06

 */
```

### 13.5.5 Formatter转换

类型转换字符：

* d 整数型（十进制）
* c Unicode字符
* b Boolean值
* s String
* f 浮点数（十进制）
* e 浮点数（科学计数）
* x 整数（十六进制）
* h 散列码（十六进制）
* % 字符"%"

```java
public class Conversion {
    public static void main(String[] args) {
        Formatter f = new Formatter(System.out);
        char u = 'a';
        System.out.println("u = 'a'");
        f.format("s: %s\n", u);
//        f.format("d: %d\n", u);
        f.format("c: %c\n", u);
        f.format("b: %b\n", u);
//        f.format("f: %f\n", u);
//        f.format("e: %e\n", u);
//        f.format("x: %x\n", u);
//        f.format("h: %h\n", u);
        int v = 121;
        System.out.println("v = 121");
        f.format("d: %d\n", v);
        f.format("c: %c\n", v);
        f.format("b: %b\n", v);
        f.format("s: %s\n", v);
//        f.format("f: %f\n", v);
//        f.format("e: %e\n", v);
        f.format("x: %x\n", v);
        f.format("h: %h\n", v);
        BigInteger w = new BigInteger("50000000000000");
        System.out.println("w = new BigInteger(\"50000000000000\")");
        f.format("d: %d\n", w);
       // f.format("c: %c\n", w);
        f.format("b: %b\n", w);
        f.format("s: %s\n", w);
//        f.format("f: %f\n", w);
//        f.format("e: %e\n", w);
        f.format("x: %x\n", w);
        f.format("h: %h\n", w);
        double x = 179.543;
        System.out.println("x = 179.543");
//        f.format("d: %d\n", x);
//        f.format("c: %c\n", x);
        f.format("b: %b\n", x);
        f.format("s: %s\n", x);
        f.format("f: %f\n", x);
        f.format("e: %e\n", x);
//        f.format("x: %x\n", x);
        f.format("h: %h\n", x);
        Conversion y = new Conversion();
        System.out.println("y = new Conversion()");
//        f.format("d: %d\n", y);
//        f.format("c: %c\n", y);
        f.format("b: %b\n", y);
        f.format("s: %s\n", y);
//        f.format("f: %f\n", y);
//        f.format("e: %e\n", y);
//        f.format("x: %x\n", y);
        f.format("h: %h\n", y);
        boolean z = false;
        System.out.println("z = false");

//        f.format("d: %d\n", z);
//        f.format("c: %c\n", z);
        f.format("b: %b\n", z);
        f.format("s: %s\n", z);
//        f.format("f: %f\n", z);
//        f.format("e: %e\n", z);
//        f.format("x: %x\n", z);
        f.format("h: %h\n", z);
    }
}
/*
u = 'a'
s: a
c: a
b: true
v = 121
d: 121
c: y
b: true
s: 121
x: 79
h: 79
w = new BigInteger("50000000000000")
d: 50000000000000
b: true
s: 50000000000000
x: 2d79883d2000
h: 8842a1a7
x = 179.543
b: true
s: 179.543
f: 179.543000
e: 1.795430e+02
h: 1ef462c
y = new Conversion()
b: true
s: strings.Conversion@7440e464
h: 7440e464
z = false
b: false
s: false
h: 4d5
 */
```

程序中的每个变量都用到了`b`转换。虽然它对各种类型都是合法的，但其行为却不一定与你想象的一致。对于`boolean`基本类型或`Boolean`对象，其转换结果是对应的`true`或`false`。但是，对其他类型的参数，只要该参数不为`null`，那转换的结果就永远都是`true`。即使是数字0，转换结果依然是`true`，而这在其他语言中，往往转换为`false`。

### 5.6 String.format\(\)

```java
public class DatabaseException extends Exception {
    public DatabaseException(int transactionID,int queryID,String message){
        super(String.format("(t%d, q%d) %s",transactionID,queryID,message));
    }

    public static void main(String[] args) {
        try{
            throw new DatabaseException(3,7,"Write failed");
        }catch (Exception e){
            System.out.println(e);
        }
    }
}
/*
strings.DatabaseException: (t3, q7) Write failed

 */
```

**一个十六进制转储（dump）工具**

```java
public class Hex {
    public static String format(byte[] data) {
        StringBuilder result = new StringBuilder();
        int n = 0;
        for (byte b : data) {
            if (n % 16 == 0) {
                result.append(String.format("%05X: ", n));
            }
            result.append(String.format("%02X", b));
            n++;
            if (n % 16 == 0) {
                result.append("\n");
            }
        }
        result.append("\n");
        return result.toString();
    }

    public static void main(String[] args) throws Exception {
        if(args.length==0){
            System.out.println(format(BinaryFile.read("Hex.class")));
        }else{
            System.out.println(BinaryFile.read(new File(args[0])));
        }
    }
}
```

## 13.6 正则表达式

### 13.6.1 基础

```java
public class Splitting {
    public static String knights =
            "Then, when you have found the shrubbery, you must " +
                    "cut down the mightiest tree in the forest... " +
                    "with... a herring!";

    public static void split(String regex) {
        System.out.println(Arrays.toString(knights.split(regex)));
    }

    public static void main(String[] args) {
        split(" ");
        split("\\W+"); //非单词字符
        split("n\\W+"); //n后面跟着一个或多个非单词字符
    }
}
/*
[Then,, when, you, have, found, the, shrubbery,, you, must, cut, down,
the, mightiest, tree, in, the, forest..., with..., a, herring!]
[Then, when, you, have, found, the, shrubbery, you, must, cut, down,
the, mightiest, tree, in, the, forest, with, a, herring]
[The, whe, you have found the shrubbery, you must cut dow,
 the mightiest tree i, the forest... with... a herring!]

 */
```

```java
public class Replacing {
    static String s = Splitting.knights;

    public static void main(String[] args) {
        System.out.println(s.replaceFirst("f\\w+","located"));//以f开头后面跟一个或多个字母
        System.out.println(s.replaceAll("shrubbery|tree|herring","banana"));//匹配三个单词中的任意一个
    }
}
```

### 13.6.2 创建正则表达式

字符类（character classes）:

* · ：任意字符
* \[abc\]：包含a、b和c的任何字符
* ：除了a、b和c之外的任何字符
* \[a-zA-Z\]：从a到z或从A到Z的任何字符
* \[abc\[hij\]\]：任意a、b、c、h、i和j
* \[a-z&&\[hij\]\]：任意h、i或j
* \s：空白符
* \S：非空白符
* \d：数字\[0-9\]
* \D：非数字
* \w：词字符\[a-zA-Z0-9\]
* \W：非词字符

逻辑操作符：

* XY：Y跟在X后面
* X\|Y：X或Y
* \(X\): 捕获组，可以在表达式中用\i引用第i个捕获组

边界匹配符

* ^：一行的开始
* $：一行的结束
* \b：词的边界
* \B：非词的边界
* \G：前一个匹配的结束

```java
public class Rudolph {
    public static void main(String[] args) {
        for (String pattern : new String[]{"Rudolph", "[rR]udolph",
                "[rR][aeiou][a-z]ol.*", "R.*"}) {
            System.out.println("Rudolph".matches(pattern));
        }
    }
}
/*
true
true
true
true
 */
```

### 13.6.3 量词

量词描述了一个模式吸收输入文本的方式：

* `贪婪型`：量词总是贪婪的，除非有其他的选项被设置。贪婪表达式会为所有可能的模式发现尽可能多的匹配。导致此问题的一个典型理由就是假定我们的模式仅能匹配第一个可能的字符组，如果它是贪婪的，那么它就会继续往下匹配。
* `勉强型`：用问好来指定，这个量词匹配满足模式所需要的最少字符数。因此也称作懒惰的、最少匹配的、非贪婪的、或不贪婪的。
* `占有型`：目前，这种类型的量词只有在`Java`语言中才可用（在其他语言中不可用），并且也更高级，因此我们大概不会立刻用到它。当正则表达式被应用于字符串时，它会产生相当多的状态，以便在匹配失败时可以回溯。而“占有的”量词并不保存这些中间状态，因此它们可以防止回溯。它们常常用于防止正则表达式失控，因此可以使正则表达式执行起来更有效。

### 13.6.4 Pattern和Matcher

比起功能有限的`String`类。我们更愿意构建功能强大的正则表达式对象。用`Pattern.compile()`方法来编译你的正则表达式。它会根据你的`String`类型的正则表达式生成一个`Pattern`对象。接下来，把你想要检索的字符串传入`Pattern`对象的`matcher()`方法。`matcher()`方法会生成一个`Matcher`对象。

```java
public class TestRegularExpression {
    public static void main(String[] args) {
        if (args.length < 2) {
            System.out.println("Usage:\njava TestRegularExpression characterSequence regularExpression+");
            System.exit(0);
        }
        System.out.println("Input: \"" + args[0] + "\"");
        for (String arg : args) {
            System.out.println("Regular expression: \"" + arg + "\"");
            Pattern p = Pattern.compile(arg);
            Matcher m = p.matcher(args[0]);
            while (m.find()) {
                System.out.println("Match \"" + m.group() + "\" at positions " + m.start() + "-" + (m.end() - 1));
            }
        }
    }
}
```

```text
java strings.TestRegularExpression "abcabcabcdefabc" "abc+" "(abc)+" "(abc){2,}"
```

输出：

```java
Input: "abcabcabcdefabc"
Regular expression: "abcabcabcdefabc"
Match "abcabcabcdefabc" at positions 0-14
Regular expression: "abc+"
Match "abc" at positions 0-2
Match "abc" at positions 3-5
Match "abc" at positions 6-8
Match "abc" at positions 12-14
Regular expression: "(abc)+"
Match "abcabcabc" at positions 0-8
Match "abc" at positions 12-14
Regular expression: "(abc){2,}"
Match "abcabcabc" at positions 0-8
```

`Pattern`对象表示编译后的正则表达式。从例子中可以看到，使用已编译的`Pattern`对象上的`matcher()`方法，加上一个输入字符串，从而共同构造了一个`Matcher`对象。同时，`Pattern`类还提供了`static`方法。

```java
static boolean matches(String regex,CharSequence input)
```

该方法用以检查`regex`是否匹配整个`CharSequence`类型的`input`参数。编译后的`Pattern`对象还提供了`split()`方法，它从匹配了`regex`的地方分割输入字符串，返回分割后的字符串`String`数组。

通过调用`Pattern.matcher()`方法，并传入一个字符串参数，我们得到了一个`Matcher`对象。使用`Matcher`上的方法，我们将能够判断各种不同类型的匹配是否成功：

* boolean matches\(\)
* boolean lookingAt\(\)
* boolean find\(\)
* boolean find\(int start\)

其中`matches()`方法用来判断整个输入字符串是否匹配正则表达式模式，而`lookingAt()`则用来判断该字符串（不必是整个字符串）的始部分是否能够匹配模式。

#### find\(\)

`Matcher.find()`方法可用来在`CharSequence`中查找多个匹配。

```java
public class Finding {
    public static void main(String[] args) {
        Matcher m = Pattern.compile("\\w+").matcher("Evening is full of the linnet's wings");
        while (m.find()) {
            System.out.print(m.group() + ", ");
        }
        System.out.println();
        int i = 0;
        while (m.find(i)) {//不断重新设定搜索的起始位置
            System.out.print(m.group() + ", ");
            i++;
        }
    }
}
/*

Evening, is, full, of, the, linnet, s, wings, 

Evening, vening, ening, ning, ing, ng, g, is, is, s, full, full, ull, ll, l, of, of, f, 
the, the, he, e, linnet, linnet, innet, nnet, net, et, t, s, s, wings, wings, ings, ngs, gs, s, 

 */
```

#### 组（Groups）

组是用括号划分的正则表达式，可以根据组的编号来引用某个组。组号为0表示整个表达式，组号1表示被第一对括号括起来的组，依次类推。因此，在下面这个表达式,

```text
A(B(C))D
```

中有三个组：组0是ABCD，组1是BC，组2是C。

`Matcher`对象提供了一系列方法，用以获取与组相关的信息：

* groupCount\(\)：返回该匹配器的模式中的分组数目，第0组不包括在内。
* group\(\)：返回前一次匹配操作（例如find\(\)\)的第0组（整个匹配）。
* group\(int i\)：返回前一次匹配操作的第i组。指定的组没有匹配输入字符串的任何部分，则将会返回`null`。
* start\(int group\)：返回在前一次匹配操作中寻找到的组的起始索引。
* end\(int group\)：返回在前一次匹配操作中寻找到的组的最后一个字符索引加一的值。

```java
public class Groups {
    static public final String POEM =
            "Twas brillig, and the slithy toves\n" +
                    "Did gyre and gimble in the wabe.\n" +
                    "All mimsy were the borogoves,\n" +
                    "And the mome raths outgrabe.\n\n" +
                    "Beware the Jabberwock, my son,\n" +
                    "The jaws that bite, the claws that catch.\n" +
                    "Beware the Jubjub bird, and shun\n" +
                    "The frumious Bandersnatch.";

    public static void main(String[] args) {
        Matcher m = Pattern.compile("(?m)(\\S+)\\s+((\\S+)\\s+(\\S+))$").matcher(POEM);
        while (m.find()) {
            for (int i = 0; i <= m.groupCount(); i++) {
                System.out.print("[" + m.group(i) + "]");
            }
            System.out.println();
        }
    }
}
/*
[the slithy toves][the][slithy toves][slithy][toves]
[in the wabe.][in][the wabe.][the][wabe.]
[were the borogoves,][were][the borogoves,][the][borogoves,]
[mome raths outgrabe.][mome][raths outgrabe.][raths][outgrabe.]
[Jabberwock, my son,][Jabberwock,][my son,][my][son,]
[claws that catch.][claws][that catch.][that][catch.]
[bird, and shun][bird,][and shun][and][shun]
[The frumious Bandersnatch.][The][frumious Bandersnatch.][frumious][Bandersnatch.]
 */
```

**start\(\)与end\(\)**

```java
public class StartEnd {
    public static String input =
            "As long as there is injustice, whenever a\n" +
                    "Targathian baby cries out, wherever a distress\n" +
                    "signal sounds among the stars... We'll be there.\n" +
                    "This fine ship, and this fine crew ...\n" +
                    "Never give up! Never surrender!";

    private static class Display {
        private boolean regexPrinted = false;
        private String regex;

        Display(String regex) {
            this.regex = regex;
        }

        void display(String message) {
            if (!regexPrinted) {
                System.out.println(regex);
                regexPrinted = true;
            }
            System.out.println(message);
        }
    }

    static void examine(String s, String regex) {
        Display d = new Display(regex);
        Pattern p = Pattern.compile(regex);
        Matcher m = p.matcher(s);
        while (m.find()) {
            d.display("find() '" + m.group() + "' start = "
                    + m.start() + " end = " + m.end());
        }
        if (m.lookingAt()) {
            d.display("lookingAt() start = " + m.start() + " end = " + m.end());
        }
        if (m.matches()) {
            d.display("matches() start = " + m.start() + " end = " + m.end());
        }
    }

    public static void main(String[] args) {
        for (String in : input.split("\n")) {
            System.out.println("input : " + in);
            for (String regex : new String[]{"\\w*ere\\w*",
                    "\\w*ever", "T\\w+", "Never.*?!"}) {
                examine(in, regex);
            }
        }
    }
}
/*
input : As long as there is injustice, whenever a
\w*ere\w*
find() 'there' start = 11 end = 16
\w*ever
find() 'whenever' start = 31 end = 39
input : Targathian baby cries out, wherever a distress
\w*ere\w*
find() 'wherever' start = 27 end = 35
\w*ever
find() 'wherever' start = 27 end = 35
T\w+
find() 'Targathian' start = 0 end = 10
lookingAt() start = 0 end = 10
input : signal sounds among the stars... We'll be there.
\w*ere\w*
find() 'there' start = 42 end = 47
input : This fine ship, and this fine crew ...
T\w+
find() 'This' start = 0 end = 4
lookingAt() start = 0 end = 4
input : Never give up! Never surrender!
\w*ever
find() 'Never' start = 0 end = 5
find() 'Never' start = 15 end = 20
lookingAt() start = 0 end = 5
Never.*?!
find() 'Never give up!' start = 0 end = 14
find() 'Never surrender!' start = 15 end = 31
lookingAt() start = 0 end = 14
matches() start = 0 end = 31
 */
```

`find()`可以在输入的任意位置定位正则表达式，而`lookingAt()`和`matches()`只有在正则表达式与输入的最开始处就开始匹配时才会成功。`matches()`只有在整个输入都匹配正则表达式时才会成功，而`lookingAt()`只要输入的第一部分匹配就会成功。

**Pattern标记**

`Pattern`类的`compile()`方法还有另一个版本，它接受一个标记参数，已调整匹配的行为：

```java
Pattern Pattern.compile(String regex,int flag)
```

其中的`flag`来自以下`Pattern`类中的常量。

* `Pattern.CANON_EQ`：两个字符当且仅当它们的完全规范分解相匹配时，就认为它们是匹配的。例如，如果我们指定这个标记，表达式`a\u030A`就会匹配字符串字符串`?`。在默认的情况下，匹配不考虑规范的等价性。
* `Pattern.CASE_INSENSITIVE(?i)`：默认情况下，大小写不敏感的匹配假定只有`US-ASCII`字符集中的字符才能进行。这个标记允许模式匹配不必考虑大小写（大写或小写）。通过指定`UNICODE_CASE`标记及结合此标记，基于`Unicode`的大小写不敏感的匹配就可以开启了。
* `Pattern.COMMENTS(?x)`：在这种模式下，空格符将被忽略掉，并且以`#`开始直到行末的注释也会被忽略掉。通过嵌入的标记表达式也可以开启`Unix`的行模式。
* `Pattern.DOTALL(?s)`：在`dotall`模式下，表达式“.”匹配所有字符，包括行终结符。默认情况下，“.”表达式不匹配行终结符。
* `Pattern.MuLTILINE(?m)`：在多行模式下，表达式`^`和`$`分别匹配一行的开始和结束。`^`还匹配输入字符串的开始，而`$`还匹配输入字符串的结尾。默认情况下，这些表达式仅匹配输入的完整字符串的开始和结束。
* `Pattern.UNICODE_CASE(?u)`：当指定这个标记，并且开启`CASE_INSENSITIVE`时，大小写不敏感的匹配将按照与`Unicode`标准相一致的方式进行。默认情况下，大小写不敏感的匹配假定只能在`US-ASCII`字符集中的字符才能进行。
* `Pattern.UNIX_LINES(?d)`：在这种模式下，在`.`、`^`和`$`行为中，只识别行终结符`\n`。

在这些标记中，`Pattern.CASE_INSENSITIVE`、`Pattern.MULTILINE`以及`Pattern.COMMENTS`特别有用。

可以通过“或”\(\|\)操作符

```java
public class ReFlags {
    public static void main(String[] args) {
        Pattern p = Pattern.compile("^java", Pattern.CASE_INSENSITIVE | Pattern.MULTILINE);
        Matcher m = p.matcher(
                "java has regex\nJava has regex\n" +
                        "JAVA has pretty good regular expressions\n" +
                        "Regular expressions are in Java"
        );
        while (m.find()){
            System.out.println(m.group());
        }
    }
}
/*
java
Java
JAVA
 */
```

### 13.6.5 split\(\)

```java
public class SplitDemo {
    public static void main(String[] args) {
        String input = "THis!!unusual use!!of exclamation!!points";
        System.out.println(Arrays.toString(Pattern.compile("!!").split(input)));
        //限制将输入分割成字符串的数量
        System.out.println(Arrays.toString(Pattern.compile("!!").split(input, 3)));
    }
}
/*
[THis, unusual use, of exclamation, points]
[THis, unusual use, of exclamation!!points]
 */
```

### 13.6.6 替换操作

* replaceFirst\(String replacement\)
* replaceAll\(String replacement\)：替换所有匹配成功的部分。
* appendReplacement\(StringButter sbuf,String replacement\)：执行渐进式的替换，允许你调用其他方法来生成或处理`replacement`，使你能够以编程的方式将目标分割成组，从而具备更强大的替换功能。
* appendTail\(StringBuffer sbuf\)：在执行了一次货多次`appendReplacement()`之后，调用此方法可以将输入字符串余下的部分复制到`sbuf`中。

```java
/*!Here's a block of text to use as input to
   the regular expression matcher. Note that we'll
   first extract the block of text by looking for
   the special delimiters, then process the
   extracted block. !*/


import net.mindview.util.TextFile;

import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class TheReplacements {
    public static void main(String[] args) {
        String s = TextFile.read("src/main/java/strings/TheReplacements.java");
        Matcher mInput =
                Pattern.compile("/\\*!(.*)!\\*/", Pattern.DOTALL)
                        .matcher(s);
        if (mInput.find()) {
            s = mInput.group(1);
        }
        //将存在两个或两个以上空格的地方,缩减为一个空格,并且
        s = s.replaceAll(" {2,}", " ");
        //删除每行开头部分的所有空格
        s = s.replaceAll("(?m)^ +","");
        System.out.println(s);
        //replaceFirst只对找到的第一个匹配进行替换
        s = s.replaceFirst("[aeiou]", "(VOWEL1)");
        StringBuffer sbuf = new StringBuffer();
        Pattern p = Pattern.compile("[aeiou]");
        Matcher m = p.matcher(s);
        while (m.find()) {
            m.appendReplacement(sbuf, m.group().toUpperCase());
        }
        System.out.println(sbuf);
        m.appendTail(sbuf);
        System.out.println(sbuf);
    }
}
/*
Here's a block of text to use as input to
the regular expression matcher. Note that we'll
first extract the block of text by looking for
the special delimiters, then process the
extracted block.
H(VOWEL1)rE's A blOck Of tExt tO UsE As InpUt tO
thE rEgUlAr ExprEssIOn mAtchEr. NOtE thAt wE'll
fIrst ExtrAct thE blOck Of tExt by lOOkIng fOr
thE spEcIAl dElImItErs, thEn prOcEss thE
ExtrActEd blO
H(VOWEL1)rE's A blOck Of tExt tO UsE As InpUt tO
thE rEgUlAr ExprEssIOn mAtchEr. NOtE thAt wE'll
fIrst ExtrAct thE blOck Of tExt by lOOkIng fOr
thE spEcIAl dElImItErs, thEn prOcEss thE
ExtrActEd blOck. 
 */
```

### 13.6.7 reset\(\)

通过`reset()`方法，可以将现有的`Matcher`对象应用于一个新的字符序列：

```java
public class Resetting {
    public static void main(String[] args) {
        Matcher m = Pattern.compile("[frb][aiu][gx]")
                .matcher("fix the rug with bags");
        while (m.find()) {
            System.out.print(m.group() + " ");
        }
        System.out.println();
        m.reset("fix the rig with rags");
        while (m.find()) {
            System.out.print(m.group() + " ");
        }
    }
}
```

### 13.6.8 正则表达式与Java I/O

```java
public class JGrep {
    public static void main(String[] args) {
        if(args.length<2){
            System.out.println("Usage: java JGrep file regex");
            System.exit(0);
        }
        Pattern p = Pattern.compile(args[1]);
        int index = 0;
        Matcher m = p.matcher("");
        for (String line : new TextFile(args[0])) {
            m.reset(line);
            while (m.find()){
                System.out.println(index++ + ": "+m.group()+": "+m.start());
            }
        }
    }
}
```

## 13.7 扫描输入

从文件或标准输入读取数据时一件相当痛苦的事情。一般的解决之道就是读入一行文本，对其进行粉刺，然后使用`Integer`、`Double`等类的各种解析方法来解析数据：

```java
public class SimpleRead {
    public static BufferedReader input = new BufferedReader(new StringReader("Sir Robin of Camelot \n22 1.61803"));

    public static void main(String[] args) {
        try {
            System.out.println("What is your name?");
            String name = input.readLine();
            System.out.println(name);
            System.out.println("How old are you? What is your favorite double?");
            System.out.println("(input: <age> <double>)");
            String numbers = input.readLine();
            System.out.println(numbers);
            String[] numArray = numbers.split(" ");
            int age = Integer.parseInt(numArray[0]);
            double favorite = Double.parseDouble(numArray[1]);
            System.out.format("Hi %s.\n", name);
            System.out.format("In 5 years you will be %d.\n", age + 5);
            System.out.format("My favorite double is %f.", favorite / 2);
        } catch (IOException e) {
            System.out.println("I/O exception");
        }
    }
}
/*
What is your name?
Sir Robin of Camelot
How old are you? What is your favorite double?
(input: <age> <double>)
22 1.61803
Hi Sir Robin of Camelot .
In 5 years you will be 27.
My favorite double is 0.809015.
 */
```

`Java SE5`新增了`Scanner`类，可以大大减轻扫描输入的工作负担：

```java
public class BetterRead {
    public static void main(String[] args) {
        Scanner stdin = new Scanner(SimpleRead.input);
        System.out.println("What is your name?");
        String name = stdin.nextLine();
        System.out.println(name);
        System.out.println("How old are you? What is your favorite double?");
        System.out.println("input: <age> <double>");
        int age = stdin.nextInt();
        double favorite = stdin.nextDouble();
        System.out.format("Hi %s.\n", name);
        System.out.format("In 5 years you will be %d.\n", age + 5);
        System.out.format("My favorite double is %f.", favorite / 2);
    }
}
/*
What is your name?
Sir Robin of Camelot
How old are you? What is your favorite double?
input: <age> <double>
Hi Sir Robin of Camelot .
In 5 years you will be 27.
My favorite double is 0.809015.
 */
```

### 7.1 Scanner定界符

在默认情况下，`Scanner`根据空白字符对输入进行分词，但是你可以用正则表达式指定自己所需的定界符：

```java
public class ScannerDelimiter {
    public static void main(String[] args) {
        Scanner scanner = new Scanner("12,42,78,99,42");
        scanner.useDelimiter("\\s*,\\s*");
        while (scanner.hasNext()){
            System.out.println(scanner.nextInt());
        }
    }
}
/*
12
42
78
99
42
 */
```

### 7.2 用正则表达式扫描

除了能够扫描基本类型之外，还可以使用自定义的正则表达式进行扫描，这在扫描复杂数据的时候非常有用。

```java
public class ThreatAnalyzer {
    static String threatData =
            "58.27.82.161@02/10/2005\n"+
            "204.45.234.40@02/11/2005\n"+
            "58.27.82.161@02/11/2005\n"+
            "58.27.82.161@02/12/2005\n"+
            "58.27.82.161@02/12/2005\n"+
            "[Next log section with different data format]";

    public static void main(String[] args) {
        Scanner scanner = new Scanner(threatData);
        String pattern = "(\\d+[.]\\d+[.]\\d+[.]\\d+)@" +
                "(\\d{2}/\\d{2}/\\d{4})";
        while (scanner.hasNext(pattern)){
            scanner.next(pattern);
            MatchResult match = scanner.match();
            String ip = match.group(1);
            String date = match.group(2);
            System.out.format("Threat on %s from %s\n",date,ip);
        }
    }
}
/*
Threat on 02/10/2005 from 58.27.82.161
Threat on 02/11/2005 from 204.45.234.40
Threat on 02/11/2005 from 58.27.82.161
Threat on 02/12/2005 from 58.27.82.161
Threat on 02/12/2005 from 58.27.82.161
 */
```

## 13.8 StringTokenizer

在`Java`引入正则表达式和`Scanner`类之前，分割字符串的唯一方法是使用`StringTokenizer`来分词。

```java
public class ReplacingStringTokenizer {
    public static void main(String[] args) {
        String input = "But I'm not dead yet! I feel happy!";
        StringTokenizer stoke = new StringTokenizer(input);
        while (stoke.hasMoreElements()){
            System.out.print(stoke.nextToken()+" ");
        }
        System.out.println();
        System.out.println(Arrays.toString(input.split(" ")));
        Scanner scanner = new Scanner(input);
        while (scanner.hasNext()){
            System.out.print(scanner.next()+" ");
        }
    }
}
/*
But I'm not dead yet! I feel happy!
[But, I'm, not, dead, yet!, I, feel, happy!]
But I'm not dead yet! I feel happy!
 */
```

## String相关面试题

* 
* String、StringBuffer、StringBuilder区别

## 参考

* [正则表达式30分钟入门教程](https://deerchao.net/tutorials/regex/regex.htm)

