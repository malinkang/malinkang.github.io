---
layout: posts
title: Java动态代理
date: 2014-11-29 17:36:33
tags: ["Java"]
---

`代理`是基本的设计模式之一，它是你为了提供额外的或不同的操作，而插入的用来代替”实际“对象的对象。这些操作通常涉及与“实际”对象的通信，因此代理通常充当着中间人的角色。下面是一个用来展示代理结构的简单示例：

```java
interface Interface {
    void doSomething();

    void somethingElse(String arg);
}
```

```java
//实现Interface
class RealObject implements Interface {

    @Override
    public void doSomething() {
        System.out.println("doSomething");
    }

    @Override
    public void somethingElse(String arg) {
        System.out.println("somethingElse " + arg);
    }
}
```

```java
class SimpleProxy implements Interface {
    private Interface proxied;

    public SimpleProxy(Interface proxied) {
        this.proxied = proxied;
    }

    @Override
    public void doSomething() {
        System.out.println("SimpleProxy doSomething");
        proxied.doSomething();
    }

    @Override
    public void somethingElse(String arg) {
        System.out.println("SimpleProxy somethingElse" + arg);
        proxied.somethingElse(arg);
    }
}
```

```java
public class SimpleProxyDemo {
    public static void consumer(Interface iface) {
        iface.doSomething();
        iface.somethingElse("bonobo");
    }

    public static void main(String[] args) {
        consumer(new RealObject());
        consumer(new SimpleProxy(new RealObject()));
    }
}
```

在任何时刻，只要你想要将额外的操作从“实际”对象中分离到不同的地方，特别是当你希望能够很容易地做出修改，**从**没有使用额外操作**转为**使用这些操作，或者反过来时，代理就显得很有用。例如，如果你希望跟踪对`RealObject`中的方法的调用，或者希望度量这些调用的开销，这些代码肯定是你不希望将其合并到应用中的代码，因此代理使得你可以很容易地添加或者移除它们。（总结：额外的操作与实际的对象分离，可以很容易地添加或者移除这些额外的操作）。

`Java`的动态代理比代理的思想更向前迈进了一步，因为它可以动态地创建代理并动态地处理对所代理方法的调用。在动态代理上所做的所有调用都会被重定向到单一的`调用处理器`上，它的工作是揭示调用的类型并确定相应的对策。下面是用动态代理重写的`SimpleDynamicProxy`：

```java
public class DynamicProxyHandler implements InvocationHandler {
    private Object proxied;
    public DynamicProxyHandler(Object proxied) {
        this.proxied = proxied;
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("**** proxy: " + proxy.getClass() +
                ", method: " + method + ", args: " + args);
        if(args != null)
            for(Object arg : args)
                System.out.println("  " + arg);
        return method.invoke(proxied,args);
    }
}
```

```java
public class SimpleDynamicProxy {
    public static void consumer(Interface iface) {
        iface.doSomething();
        iface.somethingElse("bonobo");
    }

    public static void main(String[] args) {
        RealObject real = new RealObject();
        consumer(real);
        Interface proxy = (Interface) Proxy.newProxyInstance(
                Interface.class.getClassLoader(),
                new Class[]{Interface.class},//你的代理类可能实现多个接口
                new DynamicProxyHandler(real)
        );
        consumer(proxy);
    }
}
```

通过调用静态方法`Proxy.newProxyInstance()`可以创建动态代理，这个方法需要得到一个类加载器（你通常可以从已经被加载的对象中获取其类加载器，然后传递给它），一个你希望**该代理实现的接口列表**（不是类或抽象类），以及`InvocationHandler`接口的一个实现。动态代理可以将所有调用重定向到调用处理器，因此通常会向调用处理器的构造器传递给一个“实际”对象的引用，从而使得调用处理器在执行其中介任务时，可以将请求转发。

![动态代理原理](https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/image/androidjava-dynamic-proxy.png)

`invoke()`方法中传递进来的代理对象，以防你需要区分请求的来源，但是在许多情况下，你并不关心这一点。然后，在`invoke()`内部，在代理上调用方法时需要格外当心，因为对接口的调用将被重定向为对代理的调用。

你可以通过传递某些参数，来过滤某些方法调用：

```java
class MethodSelector implements InvocationHandler {
    private Object proxied;

    public MethodSelector(Object proxied) {
        this.proxied = proxied;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (method.getName().equals("interesting"))
            System.out.println("Proxy detected the interesting method");
        return method.invoke(proxied, args);
    }
}
```

```java
interface SomeMethods {
  void boring1();
  void boring2();
  void interesting(String arg);
  void boring3();
}
```

```java
class Implementation implements SomeMethods {
  public void boring1() { System.out.println("boring1"); }
  public void boring2() { System.out.println("boring2"); }
  public void interesting(String arg) {
    System.out.println("interesting " + arg);
  }
  public void boring3() { System.out.println("boring3"); }
} 
```



```java
class SelectingMethods {
  public static void main(String[] args) {
    SomeMethods proxy= (SomeMethods) Proxy.newProxyInstance(
      SomeMethods.class.getClassLoader(),
      new Class[]{ SomeMethods.class },
      new MethodSelector(new Implementation()));
    proxy.boring1();
    proxy.boring2();
    proxy.interesting("bonobo");
    proxy.boring3();
  }
}
```

## 参考

* 《Java编程思想》