---
title: 《Java编程思想》第9章接口
date: 2013-05-07 13:39:36
tags: [Thinking in Java]
---


## 1.抽象类和抽象方法

## 2.接口

## 3.完全解耦

## 4.Java的多重继承

```java
public interface CanFight {
    void fight();
}
public interface CanSwim {
    void swim();
}
public interface CanFly {
    void fly();
}
public class ActionCharacter {
    public void fight(){}
}
public class Hero extends ActionCharacter implements CanFight, CanSwim, CanFly {
    //CanFight接口和ActionCharacter类中的fight()方法的特征签名一样
    public void swim() {}

    public void fly() {}
}
public class Adventure {
    public static void t(CanFight x){x.fight();}
    public static void u(CanSwim x){x.swim();}
    public static void v(CanFly x){x.fly();}
    public static void w(ActionCharacter x){x.fight();}

    public static void main(String[] args) {
        Hero hero = new Hero();
        t(hero);
        u(hero);
        v(hero);
        w(hero);
    }
}
```

## 5.通过继承来扩展接口

通过继承，可以很容易地在接口中添加新的方法声明，可以通过继承在新接口中组合数个接口。这两种情况可以获得新的接口。

```java
public interface Monster {
    void menace();
}
public interface DangerousMonster extends Monster  {
    void destory();
}
public interface Lethal {
    void kill();
}
public class DragonZilla implements DangerousMonster {
    public void menace() {}
    public void destory() {}
}
//一般情况下，只可以将extends用于单一类，但是可以引用多个基类接口。
public interface Vampire extends DangerousMonster,Lethal {
    void drinkBlood();
}
public class VeryBadVampire implements Vampire {
    public void menace() {}
    public void destory() {}
    public void kill() {}
    public void drinkBlood() {}
}
public class HorrorShow {
    static void u(Monster b) {
        b.menace();
    }
    static void v(DangerousMonster d) {
        d.menace();
        d.destory();
    }
    static void w(Lethal l) {
        l.kill();
    }
    public static void main(String[] args) {
        DangerousMonster barney = new DragonZilla();
        u(barney);
        v(barney);
        Vampire vlad = new VeryBadVampire();
        u(vlad);
        v(vlad);
        w(vlad);
    }
}
```

### 9.5.1 组合接口时的名字冲突

实现多重继承时，相同的方法不会有什么问题，但是如果它们的签名或返回类型不同，又会怎么样呢？

```java
public interface I1 {
    void f();
}
public interface I2 {
    int f(int i);
}
public interface I3 {
    int f();
}
public class C {
    public int f(){
        return 1;
    }
}
//重载
public class C2 implements I1, I2 {
    public void f() {}
    public int f(int i) {
        return 1;
    }
}
//重载
public class C3 extends C implements I2 {
    public int f(int i) {
        return 0;
    }
}
public class C4 extends C implements I3 {
}
//下面代码会出错
//public class I4 extends I1,I3 {
//    public void f() {
//
//    }
//}
```

## 6.适配接口

## 7.接口中的域

因为你放入接口中的任何域都自动是`static`和`final`的，所以接口就成为了一种很便捷的用来创建常量组的工具。

### 7.1初始化接口中的域

在接口中定义的域不能是空`final`，但是可以被非常量表达式初始化。

既然域是`static`的，它们就可以在类第一次被加载时初始化，这发生在任何域首次被访问时。

```java
public interface RanVals {
    Random RAND = new Random(47);
    int RANDOM_INT = RAND.nextInt(10);
    long RANDOM_LONG = RAND.nextLong() * 10;
    float RANDOM_FLOAT = RAND.nextLong() * 10;
    double RANDOM_DOUBLE = RAND.nextDouble() * 10;
}
```

## 8.嵌套接口

接口可以嵌套在类或其他接口中。

## 9.接口与工厂

