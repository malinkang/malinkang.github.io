---
title: 《Effective Java》第4章类和接口
date: 2016-01-16 17:34:17
tags: ["Effective Java"]
cover: https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/blog/images/cover/你的名字50.png
---

## 第13条：使类和成员的可访问性最小化

区分一个组件设计得好不好，唯一重要的因素在于，它对于外部的其他组件而言，是否隐藏了其内部数据和其他实现细节。 设计良好的组件会隐藏所有的实现细节， 把 API与实现清晰地隔离开来。 然后，组件之间只通过API进行通信，一个模块不需要知道其他模块的内部工作情况。 这个概念被称为`信息隐藏( info1mation hiding)`或`封装( encapsulation)`, 是软件设计的基本原则之一。

`Java`程序设计语言提供了许多机制来协助信息隐藏。`访问控制（access control)`机制决定了类、接口和成员的`可访问性（accessibility）`。实体的可访问性是由该实体声明所在的位置，以及该实体声明中所出现的访问修饰符共同决定的。正确地使用这些修饰符对于实现信息隐藏是非常关键的。

第一规则很简单：**尽可能地使每个类或者成员不被外界访问**。

## 第14条：在共有类中使用访问方法而非共有域

## 第15条：使可变性最小化

## 第18条：复合优先于继承

继承（inheritance）是实现代码重用的有力手段，但它并非永远是完成这项工作的最佳工具。使用不当会导致软件变得很脆弱。在包的内部使用继承是非常安全的，在那里子类和超类的实现都处在同一个程序员的控制之下。对于专门为了继承而设计并且具有很好的文档说明的类来说，使用继承也是非常安全的。然而，对普通的具体类（concrete class）进行跨越包边界的继承，则是非常危险的。

与方法调用不同的是，继承打破了封装性。换句话说，子类依赖于其超类中特定功能的实现细节。超类的实现有可能会随着发行版本的不同而有所变化，如果真的发生了变化，子类可能会遭到破坏，即使它的代码完全没有改变。因而，子类必须要跟着其超类的更新而演变，除非超类是专门为了扩展而设计的，并且具有很好的文档说明。

```java
public class InstrumentedHashSet<E> extends HashSet<E> {
    private int addCount = 0;
    public InstrumentedHashSet(){

    }
    public InstrumentedHashSet(int initCap,float loadFactor){
        super(initCap,loadFactor);
    }

    @Override
    public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        int i = addCount += c.size();
        return super.addAll(c);
    }

    public int getAddCount() {
        return addCount;
    }

    public static void main(String[] args) {
        InstrumentedHashSet<String> s = new InstrumentedHashSet<>();
        //List.of方法 Java9新增
        s.addAll(List.of("Snap","Crackle","Pop"));
    }
}
```

在新的类中增加一个私有域，它引用现有类的一个实例。这种设计被称为“复合”（composition），因为现有的类变成了新类的一个组件。新类中的每个实例方法都可以调用被包含的现有类实例中对应的方法，并返回它的结果。这种称为`转发（forwarding）`，新类中的方法被称为`转发方法（forwarding method）`。这样得到的类将会非常稳固，它不依赖于现有类的实现细节。即使现有的类添加了新的方法，也不会影响新的类。

只有当子类真正是超类的子类型（subtype）时，才适合用继承。换句话说，对于两个类A和B，只有当两者之间确实存在“is-a”关系的时候，类B才应该扩展类A。如果你打算让类B扩展类A，就应该问问自己：每个B确实也是A吗？如果你不能够确定这个问题的答案是肯定的，那么B就不应该扩展A。如果答案是否定的，通常情况下，B应该包含A的一个私有实例，并且暴露一个较小的、较简单的API：A本质上不是B的一部分，只是它的实现细节而已。

在Java平台类库中，有许多明显违反这条原则的地方。例如，栈（stack）并不是向量（vector），所以Stack不应该扩展Vector。属性列表也不是散列表，所以Properties不应该扩展Hashtable。

在决定使用继承而不是复合之前，还应该问自己最后一组问题。对于你正视图扩展的类，它的API中有没有缺陷呢？如果有，你是否愿意把那些缺陷传播到类的API中？继承机制会把超类API中的所有缺陷传播到子类中，而复合则允许设计新的AP来隐藏这些缺陷。

简而言之，继承的功能非常强大，但是也存在诸多问题，因为它违背了封装原则。只有当子类和超类之间确实存在子类型关系时，使用继承才是恰当的。即使如此，如果子类和超类处在不同的包中，并且超类并不是为了继承而设计的，那么继承将会导致脆弱性。为了避免这种脆弱性，可以用复合和转发机制来替代继承，尤其是当存在适当的接口可以实现包装类的时候。包装类不仅比子类更加健壮，而且功能也更加强大。

## 第19条：要么为继承而设计，并提供文档说明，要么就禁止继承

## 第20条：接口优于抽象类
