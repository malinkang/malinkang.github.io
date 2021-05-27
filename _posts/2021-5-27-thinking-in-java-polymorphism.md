---
title: 《Java编程思想》第8章多态
date: 2013-04-30 13:39:36
tags: [Thinking in Java]
---

## 8.1 再论向上转型

对某个对象的引用视为对其基类型的引用的做法被称作`向上转型`。

单独创建一个奏乐符（Note）类：

```java
//: polymorphism/music/Note.java
// Notes to play on musical instruments.
package polymorphism.music;

public enum Note {
    MIDDLE_C, C_SHARP, B_FLAT; // Etc.
} ///:~
```

乐器类`Instrument`:

```java
//: polymorphism/music/Instrument.java
package polymorphism.music;
import static net.mindview.util.Print.*;

class Instrument {
  public void play(Note n) {
    print("Instrument.play()");
  }
}
 ///:~
```

```java
//: polymorphism/music/Wind.java
package polymorphism.music;

// Wind objects are instruments
// because they have the same interface:
public class Wind extends Instrument {
  // Redefine interface method:
  public void play(Note n) {
    System.out.println("Wind.play() " + n);
  }
} ///:~
```

```java
//: polymorphism/music/Music.java
// Inheritance & upcasting.
package polymorphism.music;

public class Music {
  public static void tune(Instrument i) {
    // ...
    i.play(Note.MIDDLE_C);
  }
  public static void main(String[] args) {
    Wind flute = new Wind();
    tune(flute); // Upcasting
  }
} /* Output:
Wind.play() MIDDLE_C
*///:~
```

### 8.1.1 忘记对象类型

```java
//: polymorphism/music/Music2.java
// Overloading instead of upcasting.
package polymorphism.music;
import static net.mindview.util.Print.*;

class Stringed extends Instrument {
  public void play(Note n) {
    print("Stringed.play() " + n);
  }
}

class Brass extends Instrument {
  public void play(Note n) {
    print("Brass.play() " + n);
  }
}

public class Music2 {
  public static void tune(Wind i) {
    i.play(Note.MIDDLE_C);
  }
  public static void tune(Stringed i) {
    i.play(Note.MIDDLE_C);
  }
  public static void tune(Brass i) {
    i.play(Note.MIDDLE_C);
  }
  public static void main(String[] args) {
    Wind flute = new Wind();
    Stringed violin = new Stringed();
    Brass frenchHorn = new Brass();
    tune(flute); // No upcasting
    tune(violin);
    tune(frenchHorn);
  }
} /* Output:
Wind.play() MIDDLE_C
Stringed.play() MIDDLE_C
Brass.play() MIDDLE_C
*///:~
```

## 8.2转机

### 8.2.1 方法调用绑定

将一个方法调用同一个方法主体关联起来被称作`绑定`。若在程序执行前进行绑定，叫做`前期绑定`。

运行时根据对象的类型进行绑定就是`后期绑定`。后期绑定也叫做`动态绑定`或`运行时绑定`。**如果一种语言想要实现后期绑定，就必须具有某种机制，以便在运行时能判断对象的类型，从而调用恰当的方法。也就是说，编译器一直不知道对象的类型，但是方法调用机制能找到正确的方法体，并加以调用**。

### 8.2.2 产生正确的行为

`Java`中所有方法都是通过动态绑定实现多态。

```java
//: polymorphism/shape/Shape.java
package polymorphism.shape;

public class Shape {
  public void draw() {}
  public void erase() {}
} ///:~


//: polymorphism/shape/Circle.java
package polymorphism.shape;
import static net.mindview.util.Print.*;

public class Circle extends Shape {
  public void draw() { print("Circle.draw()"); }
  public void erase() { print("Circle.erase()"); }
} ///:~

//: polymorphism/shape/Square.java
package polymorphism.shape;
import static net.mindview.util.Print.*;

public class Square extends Shape {
  public void draw() { print("Square.draw()"); }
  public void erase() { print("Square.erase()"); }
} ///:~

//: polymorphism/shape/Triangle.java
package polymorphism.shape;
import static net.mindview.util.Print.*;

public class Triangle extends Shape {
  public void draw() { print("Triangle.draw()"); }
  public void erase() { print("Triangle.erase()"); }
} ///:~
//: polymorphism/shape/RandomShapeGenerator.java
// A "factory" that randomly creates shapes.
package polymorphism.shape;
import java.util.*;

public class RandomShapeGenerator {
  private Random rand = new Random(47);
  public Shape next() {
    switch(rand.nextInt(3)) {
      default:
      case 0: return new Circle();
      case 1: return new Square();
      case 2: return new Triangle();
    }
  }
} ///:~

//: polymorphism/Shapes.java
// Polymorphism in Java.
import polymorphism.shape.*;

public class Shapes {
  private static RandomShapeGenerator gen =
    new RandomShapeGenerator();
  public static void main(String[] args) {
    Shape[] s = new Shape[9];
    // Fill up the array with shapes:
    for(int i = 0; i < s.length; i++)
      s[i] = gen.next();
    // Make polymorphic method calls:
    for(Shape shp : s)
      shp.draw();
  }
} /* Output:
Triangle.draw()
Triangle.draw()
Square.draw()
Triangle.draw()
Square.draw()
Triangle.draw()
Square.draw()
Triangle.draw()
Circle.draw()
*///:~
```

每个`return`语句取得一个指向某个`Circle`、`Square`或者`Triangle`的引用，并将其以`Shape`类型从`next()`方法中发送出去。所以无论我们在上面时候调用`next()`方法时，是绝对不可能知道具体类型到底是什么，因为我们总是只能获得一个通用的`Shape`引用。

随机选择几何形状是为了让大家理解：在编译时，编译器不需要获得任何特殊信息就能进行正确的调用。对`draw()`方法的所有调用都是通过动态绑定进行的。

### 8.2.3 可扩展性

```java
//: polymorphism/music3/Music3.java
// An extensible program.
package polymorphism.music3;
import polymorphism.music.Note;
import static net.mindview.util.Print.*;

class Instrument {
  void play(Note n) { print("Instrument.play() " + n); }
  String what() { return "Instrument"; }
  void adjust() { print("Adjusting Instrument"); }
}

class Wind extends Instrument {
  void play(Note n) { print("Wind.play() " + n); }
  String what() { return "Wind"; }
  void adjust() { print("Adjusting Wind"); }
}    

class Percussion extends Instrument {
  void play(Note n) { print("Percussion.play() " + n); }
  String what() { return "Percussion"; }
  void adjust() { print("Adjusting Percussion"); }
}

class Stringed extends Instrument {
  void play(Note n) { print("Stringed.play() " + n); }
  String what() { return "Stringed"; }
  void adjust() { print("Adjusting Stringed"); }
}

class Brass extends Wind {
  void play(Note n) { print("Brass.play() " + n); }
  void adjust() { print("Adjusting Brass"); }
}

class Woodwind extends Wind {
  void play(Note n) { print("Woodwind.play() " + n); }
  String what() { return "Woodwind"; }
}    

public class Music3 {
  // Doesn't care about type, so new types
  // added to the system still work right:
  public static void tune(Instrument i) {
    // ...
    i.play(Note.MIDDLE_C);
  }
  public static void tuneAll(Instrument[] e) {
    for(Instrument i : e)
      tune(i);
  }    
  public static void main(String[] args) {
    // Upcasting during addition to the array:
    Instrument[] orchestra = {
      new Wind(),
      new Percussion(),
      new Stringed(),
      new Brass(),
      new Woodwind()
    };
    tuneAll(orchestra);
  }
} /* Output:
Wind.play() MIDDLE_C
Percussion.play() MIDDLE_C
Stringed.play() MIDDLE_C
Brass.play() MIDDLE_C
Woodwind.play() MIDDLE_C
*///:~
```

多态是一项让程序员“将改变的事物与未变的事物分离开来”的重要技术。

### 8.2.4 缺陷：“覆盖”私有方法

```java
//: polymorphism/PrivateOverride.java
// Trying to override a private method.
package polymorphism;
import static net.mindview.util.Print.*;

public class PrivateOverride {
  private void f() { print("private f()"); }
  public static void main(String[] args) {
    PrivateOverride po = new Derived();
    po.f();
  }
}

class Derived extends PrivateOverride {
  public void f() { print("public f()"); }
} /* Output:
private f()
*///:~
```

只有非`private`方法才可以被覆盖；在导出类中，对于基类中的`private`方法，最好采用不同的名字。

### 8.2.5 缺陷：域与静态方法

```java
//: polymorphism/FieldAccess.java
// Direct field access is determined at compile time.

class Super {
  public int field = 0;
  public int getField() { return field; }
}

class Sub extends Super {
  public int field = 1;
  public int getField() { return field; }
  public int getSuperField() { return super.field; }
}

public class FieldAccess {
  public static void main(String[] args) {
    Super sup = new Sub(); // Upcast
    System.out.println("sup.field = " + sup.field +
      ", sup.getField() = " + sup.getField());
    Sub sub = new Sub();
    System.out.println("sub.field = " +
      sub.field + ", sub.getField() = " +
      sub.getField() +
      ", sub.getSuperField() = " +
      sub.getSuperField());
  }
} /* Output:
sup.field = 0, sup.getField() = 1
sub.field = 1, sub.getField() = 1, sub.getSuperField() = 0
*///:~
```

当`Sub`对象转型为`Super`引用时，任何域访问操作都将由编译器解析，因此不是多态的。在本例中，为`Super.field`和`Sub.field`分配了不同的存储空间。这样，`Sub`实际上包含两个称为`field`的域：它自己的和它从`Super`处得到的。然而，在引用`Sub`中的`field`时所产生的默认域并非`Super`版本的`field`域。因此，为了得到`Super.field`，必须显式地指明`super.field`。

如果某个方法是静态的，它的行为就不具有多态性：

```java
//: polymorphism/StaticPolymorphism.java
// Static methods are not polymorphic.

class StaticSuper {
  public static String staticGet() {
    return "Base staticGet()";
  }
  public String dynamicGet() {
    return "Base dynamicGet()";
  }
}

class StaticSub extends StaticSuper {
  public static String staticGet() {
    return "Derived staticGet()";
  }
  public String dynamicGet() {
    return "Derived dynamicGet()";
  }
}

public class StaticPolymorphism {
  public static void main(String[] args) {
    StaticSuper sup = new StaticSub(); // Upcast
    System.out.println(sup.staticGet());
    System.out.println(sup.dynamicGet());
  }
} /* Output:
Base staticGet()
Derived dynamicGet()
*///:~
```

静态方法是与类，而并非与单个对象相关联

## 8.3构造器和多态

构造器不具有多态性（它们实际上是`static`方法，只不过该`static`声明是隐式的）。

### 8.3.1 构造器的调用顺序

基类的构造器总是在导出类的构造过程中被调用，而且按照继承层次逐渐向上链接，以使每个基类的构造器都能得到调用。这样做是有意义的，因为构造器具有一项特殊任务：检查对象是否被正确地构造。导出类只能访问它自己的成员，不能访问基类中的成员。只有基类的构造器才具有恰当的知识和权限来对自己的元素进行初始化。因此，必须令所有构造器都得到调用，否则就不可能正确构造完整对象。这正是编译器为什么要强制每个导出类部分都必须调用构造器的原因。

```java
//: polymorphism/Sandwich.java
// Order of constructor calls.
package polymorphism;
import static net.mindview.util.Print.*;

class Meal {
  Meal() { print("Meal()"); }
}

class Bread {
  Bread() { print("Bread()"); }
}

class Cheese {
  Cheese() { print("Cheese()"); }
}

class Lettuce {
  Lettuce() { print("Lettuce()"); }
}

class Lunch extends Meal {
  Lunch() { print("Lunch()"); }
}

class PortableLunch extends Lunch {
  PortableLunch() { print("PortableLunch()");}
}
//先调用静态方法main() 再初始化字段
public class Sandwich extends PortableLunch {
  private Bread b = new Bread();
  private Cheese c = new Cheese();
  private Lettuce l = new Lettuce();
  public Sandwich() { print("Sandwich()"); }
  public static void main(String[] args) {
    new Sandwich();
  }
} /* Output:
Meal()
Lunch()
PortableLunch()
Bread()
Cheese()
Lettuce()
Sandwich()
*///:~
```

### 8.3.2 继承与清理

```java
//: polymorphism/Frog.java
// Cleanup and inheritance.
package polymorphism;
import static net.mindview.util.Print.*;

class Characteristic {
  private String s;
  Characteristic(String s) {
    this.s = s;
    print("Creating Characteristic " + s);
  }
  protected void dispose() {
    print("disposing Characteristic " + s);
  }
}

class Description {
  private String s;
  Description(String s) {
    this.s = s;
    print("Creating Description " + s);
  }
  protected void dispose() {
    print("disposing Description " + s);
  }
}

class LivingCreature {
  private Characteristic p =
    new Characteristic("is alive"); //1 
  private Description t =
    new Description("Basic Living Creature"); //2
  LivingCreature() {
    print("LivingCreature()"); //3
  }
  protected void dispose() {
    print("LivingCreature dispose");
    t.dispose();
    p.dispose();
  }
}

class Animal extends LivingCreature {
  private Characteristic p =
    new Characteristic("has heart"); //4
  private Description t =
    new Description("Animal not Vegetable"); //5
  Animal() { print("Animal()"); } //6 
  protected void dispose() {
    print("Animal dispose");
    t.dispose();
    p.dispose();
    super.dispose();
  }
}

class Amphibian extends Animal {
  private Characteristic p =
    new Characteristic("can live in water"); // 7
  private Description t =
    new Description("Both water and land"); // 8
  Amphibian() {
    print("Amphibian()"); //9
  }
  protected void dispose() {
    print("Amphibian dispose");
    t.dispose();
    p.dispose();
    super.dispose();
  }
}

public class Frog extends Amphibian {
  private Characteristic p = new Characteristic("Croaks"); //10
  private Description t = new Description("Eats Bugs"); // 11
  public Frog() { print("Frog()"); } //12
  protected void dispose() {
    print("Frog dispose"); //14
    t.dispose(); //15
    p.dispose(); //16
    super.dispose();
  }
  public static void main(String[] args) {
    Frog frog = new Frog();
    print("Bye!"); //13
    frog.dispose();
  }
} /* Output:
Creating Characteristic is alive
Creating Description Basic Living Creature
LivingCreature()
Creating Characteristic has heart
Creating Description Animal not Vegetable
Animal()
Creating Characteristic can live in water
Creating Description Both water and land
Amphibian()
Creating Characteristic Croaks
Creating Description Eats Bugs
Frog()
Bye!
Frog dispose
disposing Description Eats Bugs
disposing Characteristic Croaks
Amphibian dispose
disposing Description Both water and land
disposing Characteristic can live in water
Animal dispose
disposing Description Animal not Vegetable
disposing Characteristic has heart
LivingCreature dispose
disposing Description Basic Living Creature
disposing Characteristic is alive
*///:~
```

```java
//: polymorphism/ReferenceCounting.java
// Cleaning up shared member objects.
import static net.mindview.util.Print.*;

class Shared {
  private int refcount = 0;
  private static long counter = 0;
  private final long id = counter++;
  public Shared() {
    print("Creating " + this);
  }
  public void addRef() { refcount++; }
  protected void dispose() {
    if(--refcount == 0)
      print("Disposing " + this);
  }
  public String toString() { return "Shared " + id; }
}

class Composing {
  private Shared shared;
  private static long counter = 0;
  private final long id = counter++;
  public Composing(Shared shared) {
    print("Creating " + this);
    this.shared = shared;
    this.shared.addRef();
  }
  protected void dispose() {
    print("disposing " + this);
    shared.dispose();
  }
  public String toString() { return "Composing " + id; }
}

public class ReferenceCounting {
  public static void main(String[] args) {
    Shared shared = new Shared();
    Composing[] composing = { new Composing(shared),
      new Composing(shared), new Composing(shared),
      new Composing(shared), new Composing(shared) };
    for(Composing c : composing)
      c.dispose();
  }
} /* Output:
Creating Shared 0
Creating Composing 0
Creating Composing 1
Creating Composing 2
Creating Composing 3
Creating Composing 4
disposing Composing 0
disposing Composing 1
disposing Composing 2
disposing Composing 3
disposing Composing 4
Disposing Shared 0
*///:~
```

### 8.3.3 构造器内部的多态方法的行为

如果要调用构造器内部的一个动态绑定方法，就要用到那个方法的被覆盖后的定义。然而，这个调用的效果可能相当难以预料，因为被覆盖的方法在对象完全构造之前就会被调用。这可能会造成一些难于发现的隐藏错误。

```java
//: polymorphism/PolyConstructors.java
// Constructors and polymorphism
// don't produce what you might expect.
import static net.mindview.util.Print.*;

class Glyph {
  void draw() { print("Glyph.draw()"); }
  Glyph() {
    print("Glyph() before draw()");
    draw(); //？？为什么调用子类的方法
    print("Glyph() after draw()");
  }
}    

class RoundGlyph extends Glyph {
  private int radius = 1;
  RoundGlyph(int r) {
    radius = r;
    print("RoundGlyph.RoundGlyph(), radius = " + radius);
  }
  void draw() {
    print("RoundGlyph.draw(), radius = " + radius);
  }
}    

public class PolyConstructors {
  public static void main(String[] args) {
    new RoundGlyph(5);
  }
} /* Output:
Glyph() before draw()
RoundGlyph.draw(), radius = 0
Glyph() after draw()
RoundGlyph.RoundGlyph(), radius = 5
*///:~
```

很早之前看了篇文章[JAVA构造时成员初始化的陷阱](https://coolshell.cn/articles/1106.html#comment-345)觉得很神奇，重读《Java编程思想》才发现，书中早有讲解。

## 8.4 协变返回类型

`Java SE5`添加了协变返回类型，它表示在导出类中的被覆盖方法可以返回基类方法的返回类型的某种导出类型：

```java
//: polymorphism/CovariantReturn.java

class Grain {
  public String toString() { return "Grain"; }
}

class Wheat extends Grain {
  public String toString() { return "Wheat"; }
}

class Mill {
  Grain process() { return new Grain(); }
}

class WheatMill extends Mill {
  Wheat process() { return new Wheat(); }
}

public class CovariantReturn {
  public static void main(String[] args) {
    Mill m = new Mill();
    Grain g = m.process();
    System.out.println(g);
    m = new WheatMill();
    g = m.process();
    System.out.println(g);
  }
} /* Output:
Grain
Wheat
*///:~
```

`Java SE5`与`Java`较早版本之间的主要差异就是焦躁的版本将强制`process()`的覆盖版本必须返回`Grain`，而不能返回`Wheat`，尽管`Wheat`是从`Grain`导出的，因而也应该是一种合法的返回类型。协变返回类型允许返回更具体的`Wheat`类型。

## 8.5 用继承进行设计

```java
//: polymorphism/Transmogrify.java
// Dynamically changing the behavior of an object
// via composition (the "State" design pattern).
import static net.mindview.util.Print.*;

class Actor {
  public void act() {}
}

class HappyActor extends Actor {
  public void act() { print("HappyActor"); }
}

class SadActor extends Actor {
  public void act() { print("SadActor"); }
}

class Stage {
  private Actor actor = new HappyActor();
  public void change() { actor = new SadActor(); }
  public void performPlay() { actor.act(); }
}

public class Transmogrify {
  public static void main(String[] args) {
    Stage stage = new Stage();
    stage.performPlay();
    stage.change();
    stage.performPlay();
  }
} /* Output:
HappyActor
SadActor
*///:~
```

## 扩展阅读

#### [\[老人的讨论帖\]你知道为什么嘛？](https://bbs.csdn.net/topics/310238132)

```java
//属性不存在动态绑定
class A {
   public int i = 10;
}

class B extends A{
   public int i = 20;
}

public class TestPolm1{
   public static void main(String args[]){
      B b = new B();
      A a = b;
      System.out.println(b.i); //20 
      System.out.println(a.i); //10
   }
}
```

```java
class A {
   private int i = 10;

   public void printI(){
      System.out.println(i);
   }
}

class B extends A{
   private int i = 20;
}

public class TestPolm2{
   public static void main(String args[]){
      B b = new B();
      A a = b;
      //都是调用的B的printI方法
      b.printI();//20
      a.printI();//20
   }
}
```

```java
class A {
   private int i = 10;

   public void printI(){
      System.out.println(i);
   }
}

class B extends A{
   private int i = 20;

   public void printI(){
      System.out.println(i);
   }
}
//这个其实和上个例子一样，都是访问的B的方法
public class TestPolm3{
   public static void main(String args[]){
      B b = new B();
      A a = b;
      b.printI(); //20
      a.printI(); //20 
   }
}
```

