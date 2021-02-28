---
title: 《Java编程思想》第10章内部类
date: 2013-05-14 13:39:36
tags: [Thinking in Java]
---

## 1.创建内部类

创建内部类的方式就是把类的定义置于外围类的里面。

```java
public class Parcel1 {
    class Contents {
        private int i = 11;

        public int value() {
            return i;
        }
    }
    class Destination {
        private String label;

        Destination(String whereTo) {
            label = whereTo;
        }

        String readLabel() {
            return label;
        }
    }
    public void ship(String dest){
        Contents c = new Contents();
        Destination d = new Destination(dest);
        System.out.println(d.readLabel());
    }

    public static void main(String[] args) {
        Parcel1 p = new Parcel1();
        p.ship("Tasmania");
    }
}
/**
 * 输出Tasmania
 */
```

更常见的情况是，外部类将有一个方法，该方法返回一个指向内部类的引用，就像下面的`to()`和`content()`方法。

```java
public class Parcel2 {
    class Contents {
        private int i = 11;

        public int value() {
            return i;
        }
    }
    class Destination {
        private String label;

        Destination(String whereTo) {
            label = whereTo;
        }

        String readLabel() {
            return label;
        }
    }
    public Destination to(String s){
        return new Destination(s);
    }

    public Contents contents(){
        return new Contents();
    }

    public void ship(String dest){
        Contents c = new Contents();
        Destination d = new Destination(dest);
        System.out.println(d.readLabel());
    }

    public static void main(String[] args) {
        Parcel2 p = new Parcel2();
        p.ship("Tasmania");
        Parcel2 q = new Parcel2();
        //如果想从外部类的非静态方法之外的任意位置创建某个内部类的对象
        //必须具体地指定这个对象的类型OuterClassName.InnerClassName
        Parcel2.Contents c = q.contents();
        Parcel2.Destination d = q.to("Borneo");
    }
}
/**
 * 输出Tasmania
 */
```

## 2.链接到外部类

当某个外围类的对象创建了一个内部类对象时，此内部类对象必定会秘密地捕获一个指向那个外围类对象的引用。然后，在你访问此外围类的成员时，就是用那个引用来选择外围类的成员。

```java
public interface Selector {
    boolean end();
    Object current();
    void next();
}

public class Sequence {
    private Object[] items;
    private int next;

    public Sequence(int size) {
        items = new Object[size];
    }

    public void add(Object x) {
        if (next < items.length) {
            items[next++] = x;
        }
    }

    private class SequenceSelector implements Selector {
        private int i = 0;

        public boolean end() {
            return i == items.length;
        }

        public Object current() {
            return items[i]; //访问外围类的items
        }

        public void next() {
            if (i < items.length) {
                i++;
            }
        }
    }

    public Selector selector() {
        return new SequenceSelector();
    }

    public static void main(String[] args) {
        Sequence sequence = new Sequence(10);
        for (int i = 0; i < 10; i++) {
            sequence.add(Integer.toString(i));
        }
        Selector selector = sequence.selector();
        while (!selector.end()) {
            System.out.println(selector.current() + " ");
            selector.next();
        }
    }
}
```

## 3.使用.this与.new

如果需要生成对外部类对象的引用，可以使用外部类的名字后面紧跟圆点和`this`。这样产生的引用自动地具有正确的类型，这一点在编译期酒杯知晓并接受检查，因此没有任何运行时的开销。

```java
public class DotThis {
    void f(){
        System.out.println("DotThis.f()");
    }
    public class Inner{
        public DotThis outer(){
            return DotThis.this;
        }
    }
    public Inner inner(){
        return new Inner();
    }

    public static void main(String[] args) {
        DotThis dt = new DotThis();
        DotThis.Inner dti = dt.inner();
        dti.outer().f();
    }

}
/*
输出DotThis.f()
 */
```

要想直接创建内部类的对象，你不能按照你想象的方式，去引用外部类的名字，而是必须使用外部类的对象来创建内部类对象。

```java
public class DotNew {
    public class Inner{}

    public static void main(String[] args) {
        DotNew dn = new DotNew();
        DotNew.Inner dni = dn.new Inner();
    }
}
```

这也解决了内部类名字作用域的问题，因此你不必声明`dn.new DotNew.Inner()`。

`.new`应用于`Parcel`示例。

```java
public class Parcel3 {
    class Contents{
        private int i = 11;
        public int value(){
            return i;
        }
    }
    class Destination{
        private String label;
        Destination(String whereTo){
            label = whereTo;
        }
        String readLabel(){
            return label;
        }


    }
    public static void main(String[] args) {
        Parcel3 p = new Parcel3();
        Parcel3.Contents c = p.new Contents();
        Parcel3.Destination d = p.new Destination("Tasmania");
    }
}
```

## 4.内部类与向上转型

当将内部类向上转型为其基类，尤其是转型为一个接口的时候，内部类某个接口的实现能够完全不可见，并且不可用。所得到的只是指向基类或接口的引用，所以能够很方便地隐藏实现细节。

创建前一个示例的接口：

```java
public interface Destination {
    String readLabel();
}
public interface Contents {
    int value();
}
public class Parcel4 {
    private class PContents implements Contents{
        private int i = 11;
        public int value() {
            return i;
        }
    }
    protected class PDestination implements Destination{
        private String label;

        private PDestination(String whereTo){
            label = whereTo;
        }

        public String readLabel() {
            return label;
        }
    }

    public Destination destination(String s){
        return new PDestination(s);
    }
    public Contents contents(){
        return new PContents();
    }

}
public class TestParcel {
    public static void main(String[] args) {
        Parcel4 p = new Parcel4();
        Contents c = p.contents();
        Destination d = p.destination("Tasmania");

    }
}
```

## 5.在方法和作用域内的内部类

到目前为止，看到的都是内部类的典型用途。内部类的语法覆盖了大量其他的更加难以理解的技术。例如，可以在一个方法里面或者任意的作用域内定义内部类。

```java
public class Parcel5 {
    //PDestination类是destination方法的一部分
    public Destination destination(String s) {
        class PDestination implements Destination {
            private String label;

            public PDestination(String whereTo) {
                label = whereTo;
            }

            public String readLabel() {
                return label;
            }
        }
        return new PDestination("Tasmania");
    }

    public static void main(String[] args) {
        Parcel5 p = new Parcel5();
        Destination d = p.destination("Tasmania");
    }
}
```

下面的例子展示了如何在任意的作用域内嵌套一个内部类

```java
public class Parcel6 {
    private void internalTracking(boolean b){
        if(b){
            class TrackingSlip{
                private String id;
                TrackingSlip(String s){
                    id = s;
                }
                String getSlip(){
                    return id;
                }
            }
            TrackingSlip ts = new TrackingSlip("slip");
            String s = ts.getSlip();
        }
    }
    public void track(){
        internalTracking(true);
    }

    public static void main(String[] args) {
        Parcel6 p = new Parcel6();
        p.track();
    }
}
```

## 6.匿名内部类

```java
public class Parcel7 {
    public Contents contents(){
        return new Contents() {
            private int i = 11;
            public int value() {
                return i;
            }
        };
    }
}
```

在这个匿名内部类中，使用了默认的构造器来生成一个`Contents`。下面的代码展示的是，基类需要一个有参数的构造器。

```java
public class Wrapping {
    private int i;
    public Wrapping(int x){
        i = x;
    }
    public int value(){
        return i;
    }
}
public class Parcel8 {
    //只需简单地传递合适的参数给基类的构造函数即可
    public Wrapping wrapping(int x){
        return new Wrapping(x){
            public int value(){
                return super.value()*47;
            }
        };
    }

    public static void main(String[] args) {
        Parcel8 p = new Parcel8();
        Wrapping w = p.wrapping(10);
    }

}
```

在匿名类中定义字段时，还能够对其执行初始化操作。

```java
public class Parcel9 {
    public Destination destination(final String dest) {
        return new Destination() {
            private String label = dest; //对字段进行初始化

            public String readLabel() {
                return label;
            }
        };
    }

    public static void main(String[] args) {
        Parcel9 p = new Parcel9();
        Destination d = p.destination("Tasmania");
    }
}
```

如果定义一个匿名内部类，并且希望它使用一个在其外部定义的对象，那么编译器会要求其参数引用是`final`的。

在匿名类中不可能有命名构造器，因为它根本没名字！但通过`实例初始化`就能够达到为匿名内部类创建一个构造器的效果。

```java
public abstract class Base {
    public Base(int i){
        System.out.println("Base constructor.i = "+i);
    }
    public abstract void f();
}
public class AnonymousConstructor {
    //匿名内部类不可能有构造器,但通过实例初始化,能够达到为匿名内部类创建一个构造器的效果

    public static Base getBase(int i){
        return new Base(i) {
            {
                System.out.println("Inside instance initializer");
            }
            @Override
            public void f() {
                System.out.println("In anonymous f()");
            }
        };
    }

    public static void main(String[] args) {
        Base base = getBase(47);
        base.f();
    }
}
/*
输出
Base constructor.i = 47
Inside instance initializer
In anonymous f()
 */
```

### 6.1 再访工厂方法

```java
public interface Service {
    void method1();
    void method2();
}

public interface ServiceFactory {
    Service getService();
}

public class Implementation1 implements Service {
    private Implementation1() {
    }

    public void method1() {
        System.out.println("Implementation1 method1");

    }

    public void method2() {
        System.out.println("Implementation1 method2");
    }

    public static ServiceFactory factory =
            new ServiceFactory() {
                public Service getService() {
                    return new Implementation1();
                }
            };
}

public class Implementation2 implements Service {
    private Implementation2() {
    }

    public void method1() {
        System.out.println("Implementation2 method1");

    }

    public void method2() {
        System.out.println("Implementation2 method2");
    }

    public static ServiceFactory factory =
            new ServiceFactory() {
                public Service getService() {
                    return new Implementation2();
                }
            };
}

public class Factories {
    public static void serviceConsumer(ServiceFactory factory){
        Service s = factory.getService();
        s.method1();
        s.method2();
    }

    public static void main(String[] args) {
        serviceConsumer(Implementation1.factory);
        serviceConsumer(Implementation1.factory);
    }
}
/*
输出
Implementation1 method1
Implementation1 method2
Implementation1 method1
Implementation1 method2
 */
```

```java
public interface Game {
    boolean move();
}

public interface GameFactory {
    Game getGame();
}

public class Checkers implements Game {

    private Checkers(){}
    private int moves = 0;
    private static final int MOVES = 3;
    public boolean move() {
        System.out.println("Checkers move"+moves);
        return ++moves!=MOVES;
    }
    public static GameFactory factory = new GameFactory() {
        public Game getGame() {
            return new Checkers();
        }
    };

}
public class Chess implements Game {
    private Chess() {
    }

    private int moves = 0;
    private static final int MOVES = 4;

    public boolean move() {
        System.out.println("Chess move " + moves);
        return ++moves != MOVES;
    }

    public static GameFactory factory = new GameFactory() {
        public Game getGame() {
            return new Chess();
        }
    };
}

public class Games {
    public static void playGame(GameFactory factory){
        Game s = factory.getGame();
        while (s.move());
    }

    public static void main(String[] args) {
        playGame(Checkers.factory);
        playGame(Chess.factory);
    }
}
/*
输出
Checkers move0
Checkers move1
Checkers move2
Chess move 0
Chess move 1
Chess move 2
Chess move 3
 */
```

## 7.嵌套类

如果不需要内部类与外围类对象之间有联系，那么可以将内部类声明为`static`，这通常称为`嵌套类`。普通的内部类对象隐式地保存了一个引用，指向创建它的外围类对象。当内部类是static时，要创建嵌套类的对象，并不需要其外围类的对象。不能从嵌套类的对象中访问非静态的外围类对象。

嵌套类与普通的内部类还有一个区别。普通内部类的字段与方法，只能放在类的外部层次上，所以**普通的内部类不能有static数据和static字段，也不能包含嵌套类。**但嵌套类可以包含所有这些东西。

```java
public class Parcel11 {
    private static class ParcelContents implements Contents {
        private int i = 11;

        public int value() {
            return i;
        }
    }

    protected static class ParcelDestination implements Destination {
        private String label;

        private ParcelDestination(String whereTo) {
            label = whereTo;
        }

        public String readLabel() {
            return label;
        }

        public static void f() {

        }

        static int x = 10;

        static class AnotherLevel {
            public static void f() {

            }

            static int x = 10;
        }
    }

    public static Destination destination(String s) {
        return new ParcelDestination(s);
    }

    public static Contents contents() {
        return new ParcelContents();
    }

    public static void main(String[] args) {
        Contents c = contents();
        Destination d = destination("Tasmania");
    }

}
```

### 7.1 接口内部的类

正常情况下，不能再接口内部放置任何代码，单嵌套类可以作为接口的一部分。你放到接口中的任何类都自动地是`public`和`static`的。因为类是`static`的，只是将嵌套类置于接口的命名空间内，这并不违法接口的规则。

```java
public interface ClassInInterface {
    void howdy();
    class Test implements ClassInInterface{

        public void howdy() {
            System.out.println("Howdy!");
        }

        public static void main(String[] args) {
            new Test().howdy();
        }
    }
}
/*
输出
Howdy!
*/
```

### 7.2 从多层嵌套类中访问外部类的成员

```java
public class MNA {
    private void f(){}
    class A{
        private void g(){}
        public class B{
            void h(){
                g();
                f();
            }
        }
    }
}
public class MultiNestingAccess {
    public static void main(String[] args){
        MNA mna = new MNA();
        MNA.A mnaa = mna.new A();
        MNA.A.B mnaab = mnaa.new B();
        mnaab.h();
    }
}
```

## 8.为什么需要内部类

每个内部类都能独立地集成自一个接口的实现，所以无论外围类是否已经集成了某个接口的实现，对于内部类都没有影响。

如果没有内部类提供的、可以继承多个具体的或抽象的类的能力，一些设计与编程问题就很难解决。从这个角度看，**内部类使得多重继承的解决方案变得完整。接口解决了部分问题，而内部类有效地实现了“多重继承”。**也就是说内部类允许继承多个非接口类型（类或抽象类）。

必须在一个类中以某种方式实现两个接口。由于接口的灵活性，你又两种选择：使用单一类，或者使用内部类。

```java
public interface A {}
public interface B {}
public class X implements A,B {}
public class Y implements A {
    B makeB(){
        return new B(){};
    }
}
public class MultiInterfaces {
    static void taskA(A a){}
    static void taskB(B b){}

    public static void main(String[] args) {
        X x = new X();
        Y y = new Y();
        taskA(x);
        taskA(y);
        taskB(x);
        taskB(y.makeB());
    }
}
```

如果拥有的是抽象的类或具体的类，而不是接口，那就只能使用内部类才能实现多重继承。

```java
public class D {}
public abstract class E {}
public class Z extends D{
    E makeE(){
        return new E() {};
    }
}
public class MultiImplementation {
    static void tasksD(D d){}
    static void tasksE(E e){}

    public static void main(String[] args) {
        Z z = new Z();
        tasksD(z);
        tasksE(z.makeE());
    }
}
```

### 8.1 闭包与回调

**闭包（closure）**是一个可调用的对象，它记录了一些信息，这些信息来自创建它的作用域。通过这个定义，可以看出内部类是面向对象的闭包，因为它不仅包含外围类对象的信息，还自动拥有一个指向外围类对象的引用，在此作用域内，内部类有权操作所有的成员，包括`private`成员。

```java
public interface Incrementable {
    void increment();
}
public class Callee1 implements Incrementable {

    private int i =0;
    public void increment() {
        i++;
        System.out.println(i);
    }
}
public class MyIncrement {
    public void increment() {
        System.out.println("Other operation");
    }

    static void f(MyIncrement mi) {
        mi.increment();
    }
}
public class Callee2 extends MyIncrement {

    private int i = 0;

    @Override
    public void increment() {
        super.increment();
        i++;
        System.out.println(i);
    }

    private class Closure implements Incrementable {
        public void increment() {
            Callee2.this.increment();
        }
    }

    Incrementable getCallbackReference() {
        return new Closure();
    }
}
public class Caller {
    private Incrementable callbackReference;
    Caller(Incrementable cbh){
        callbackReference = cbh;
    }
    void go(){
        callbackReference.increment();
    }
}
public class Callbacks {
    public static void main(String[] args) {
        Callee1 c1 = new Callee1();
        Callee2 c2 = new Callee2();
        MyIncrement.f(c2);
        Caller caller1 = new Caller(c1);
        Caller caller2 = new Caller(c2.getCallbackReference());
        caller1.go();
        caller1.go();
        caller2.go();
        caller2.go();
    }
}
/*
输出
Other operation
1
1
2
Other operation
2
Other operation
3
 */
```

### 8.2 内部类与控制框架

## 9.内部类的集成

创建构造器，不能只是传递一个指向外围类对象的引用。必须在构造器内部使用如下语法。

```java
enclosingClassReference.super();
```

```java
public class WithInner {
    class Inner{}
}
public class InheritInner extends WithInner.Inner {
    InheritInner(WithInner wi){
        wi.super();
    }
}
```

## 10.内部类可以被覆盖吗

```java
public class Egg {
    private Yolk y;
    protected class Yolk{
        public Yolk(){
            System.out.println("Egg.Yolk()");
        }
    }
    public Egg(){
        System.out.println("New Egg()");
        y = new Yolk();
    }
}
public class BigEgg extends Egg {
    public class Yolk{
        public Yolk(){
            System.out.println("BigEgg.Yolk()");
        }
    }

    public static void main(String[] args) {
        new BigEgg();
    }
}
/*
New Egg()
Egg.Yolk()
 */
```

```java
public class Egg2 {
    protected class Yolk{
        public Yolk(){
            System.out.println("Egg2.Yolk()");
        }
        public void f(){
            System.out.println("Egg2.Yolk.f()");
        }
    }
    private Yolk y = new Yolk();
    public Egg2(){
        System.out.println("new Egg2()");
    }
    public void insertYolk(Yolk yy){
        y = yy;
    }
    public void g(){
        y.f();
    }

}
public class BigEgg2 extends Egg2 {
    public class Yolk extends Egg2.Yolk {
        public Yolk() {
            System.out.println("BigEgg2.Yolk()");
        }

        @Override
        public void f() {
            System.out.println("BigEgg2.Yolk.f()");
        }
    }

    public BigEgg2() {
        insertYolk(new Yolk());
    }

    public static void main(String[] args) {
        Egg2 e2 = new BigEgg2();
        e2.g();
    }

}
/*
输出
Egg2.Yolk()
new Egg2()
Egg2.Yolk()
BigEgg2.Yolk()
BigEgg2.Yolk.f()
 */
```

## 11.局部内部类

```java
public interface Counter {
    int next();
}
public class LocalInnserClass {
    private int count;

    Counter getCounter(final String name) {
        class LocalCounter implements Counter {
            public LocalCounter() {
                System.out.println("LocalCounter()");
            }

            public int next() {
                System.out.print(name);
                return count++;
            }
        }
        return new LocalCounter();
    }

    Counter getCounter2(final String name) {
        return new Counter() {
            {
                System.out.println("Counter");
            }

            public int next() {
                System.out.print(name);
                return count++;
            }
        };
    }

    public static void main(String[] args) {
        LocalInnserClass lic = new LocalInnserClass();
        Counter c1 = lic.getCounter("Local innser"),
                c2 = lic.getCounter2("Anonymous inner");
        for (int i = 0; i < 5; i++) {
            System.out.println(c1.next());
        }
        for (int i = 0; i < 5; i++) {
            System.out.println(c2.next());
        }
    }
}
/*
LocalCounter()
Counter
Local innser0
Local innser1
Local innser2
Local innser3
Local innser4
Anonymous inner5
Anonymous inner6
Anonymous inner7
Anonymous inner8
Anonymous inner9
 */
```

## 12.内部类标识符

每个类都会产生一个`.class`文件，其中包含了如何创建该类型的对象的全部信息。内部类也必须生成一个`.class`文件以包含它们的Class对象信息。这些类文件的命名有严格的规则：外围类的名字，加上`$`,再加上内部类的名字。例如，`LocalInnerClass.java`生成的`.class`文件包括：

```text
Counter.class
LocalInnerClass$1.class
LocalInnerClass$1LocalCounter.class
LocalInnserClass.class
```

如果内部类是匿名的，编译器会简单地产生一个数字作为其标识符。如果内部类是嵌套在别的内部类之中，只需直接将它们的名字加载其外围类标识符与`$`的后面。

## 参考

* Java编程思想

