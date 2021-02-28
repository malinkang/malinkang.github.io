---
title: 《Java编程思想》第11章持有对象
date: 2013-05-21 13:39:36
tags: [Thinking in Java,读书笔记]
---

# 第11章 持有对象

## 1.泛型和类型安全的容器

`Apple`和`Orange`都放置在了容器中，然后将它们取出。正常情况下，`Java`编译器会报告警告信息，因为这个示例没有使用泛型。在这里，我们使用`Java SE5`所特有的注解来抑制了警告信息。注解以“@”符号开头，可以接受参数，这里的`@SuppressWarnings`注解及其参数标识只有有关“不受检查的异常”的警告信息应该被压制。

```java
class Apple{
    private static long counter;
    private final long id = counter++;
    public long id(){ return id;}
}
class Orange{}
public class ApplesAndOrangesWithoutGenerics {
    @SuppressWarnings("unchecked")
    public static void main(String[] args) {
        ArrayList apples = new ArrayList();
        for (int i = 0; i < 3; i++) {
            apples.add(new Apple());
        }
        apples.add(new Orange());
        for (int i = 0; i < apples.size(); i++) {
            ((Apple)apples.get(i)).id();
            //Orange is detected only at run time
        }
    }
}
/*
java.lang.ClassCastException: Orange cannot be cast to Apple
 */
```

要想定义用来保存`Apple`对象的`ArrayList`，你可以声明`ArrayList<Apple>`，而不仅仅只是`ArrayList`。其中尖括号括起来的是`类型参数`，它指定了这个容器实例可以保存的类型。通过使用泛型，就可以在编译期防止将错误类型的对象放置到容器中。

```java
public class ApplesAndOrangesWithGenerics {
    @SuppressWarnings("unchecked")
    public static void main(String[] args) {
        ArrayList<Apple> apples = new ArrayList();
        for (int i = 0; i < 3; i++) {
            apples.add(new Apple());
        }
        //编译错误
        //apples.add(new Orange());
        for (int i = 0; i < apples.size(); i++) {
            System.out.println(apples.get(i).id());
        }
        for (Apple c : apples) {
            System.out.println(c.id());
        }
    }
}
/*
输出
0
1
2
0
1
2
 */
```

当你指定了某个类型作为泛型参数时，你并不仅限于只能将该确切类型的对象放置到容器中，向上转型也可以像作用域其他类型一样作用于泛型。

```java
class GrannySmith extends Apple{}
class Gala extends Apple{}
class Fuji extends Apple{}
class Braeburn extends Apple{}
public class GenericsAndUpcasting {
    public static void main(String[] args) {
        ArrayList<Apple> apples = new ArrayList<Apple>();
        apples.add(new GrannySmith());
        apples.add(new Gala());
        apples.add(new Fuji());
        apples.add(new Braeburn());
        for (Apple c :apples){
            System.out.println(c);
        }
    }
}
/*
输出
GrannySmith@60e53b93
Gala@5e2de80c
Fuji@1d44bcfa
Braeburn@266474c2
 */
```

## 2.基本概念

`Java`容器类类库用途是“保存对象”，并将其划分为两个不同的概念。

* Collection：一个独立元素的序列，这些元素都服从一条或多条规则。`List`必须按照插入的顺序保存元素。而`Set`不能有重复元素。`Queue`按照排队规则来确定对象产生的顺序。
* Map：一组成对的”键值对“对象，允许你使用键来查找值。

```java
public class SimpleCollection {
    public static void main(String[] args) {
        Collection<Integer> c = new ArrayList<Integer>();
        for (int i = 0; i < 10; i++) {
            c.add(i);
        }
        for(Integer i:c){
            System.out.print(i+", ");
        }
    }
}
/*
0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 
 */
```

## 3.添加一组元素

```java
public class AddingGoups {
    public static void main(String[] args) {
        Collection<Integer> collection = new ArrayList<Integer>(Arrays.asList(1,2,3,4,5));
        Integer[] moreInts = {6,7,8,9,10};
        collection.addAll(Arrays.asList(moreInts));
        Collections.addAll(collection,11,12,13,14,15);
        Collections.addAll(collection,moreInts);
        List<Integer> list = Arrays.asList(16,17,18,19,20);
        list.set(1,99);
        //list.add(21); 运行时异常 
    }
}
```

可以直接使用`Arrays.asList()`的输出，将其当做`List`，但是在这种情况下，其底层表示的是数组，因此不能调整尺寸，如果视图用`add()`或`delete()`方法在这种列表中添加或删除元素就有可能会引发去改变数组尺寸的尝试，因此你将在运行时获得`Unsupported Operation（不支持的操作）`错误。

```java
class Snow{}
class Powder extends Snow{}
class Light extends Powder{}
class Heavy extends Powder{}
class Crusty extends Snow{}
class Slush extends Snow{}
public class AsListInference {
    public static void main(String[] args) {
        List<Snow> snow1 = Arrays.asList(new Crusty(),new Slush(),new Powder());
        //编译错误
        //List<Snow> snow2 = Arrays.asList(new Light(),new Heavy());
        List<Snow> snow3 = new ArrayList<Snow>();
        Collections.addAll(snow3,new Light(),new Heavy());
        List<Snow> snow4 = Arrays.<Snow>asList(new Light(),new Heavy());
    }
}
```

当试图创建`snow2`时，`Arrays.asList()`中只有`Powder`类型，因此它会创建`List<Powder>`而不是`List<Snow>`。`Collection.addAll()`工作的很好，因为它从第一个参数中了解到了目标类型是什么。

正如你从创建`snow4`的操作中所看到的，可以在`Arrays.asList()`中间插入一条“线索”，以告诉编译器对于由`Arrays.asList()`产生的`List`类型，实际的目标类型应该是什么。这称为`显式类型参考说明`。

## 4.容器的打印

```java
public class PrintingContainers {
    static Collection fill(Collection<String> collection){
        collection.add("rat");
        collection.add("cat");
        collection.add("dog");
        collection.add("dog");
        return collection;
    }

    static Map fill(Map<String,String> map){
        map.put("rat","Fuzzy");
        map.put("cat","Rags");
        map.put("dog","Bosco");
        map.put("dog","Spot");
        return map;
    }

    public static void main(String[] args) {
        System.out.println(fill(new ArrayList<String>()));
        System.out.println(fill(new LinkedList<String>()));
        System.out.println(fill(new HashSet<String>()));
        System.out.println(fill(new TreeSet<String>()));
        System.out.println(fill(new LinkedHashSet<String>()));
        System.out.println(fill(new HashMap<String, String>()));
        System.out.println(fill(new TreeMap<String, String>()));
        System.out.println(fill(new LinkedHashMap<String, String>()));
    }
}
/*
[rat, cat, dog, dog]
[rat, cat, dog, dog]
[rat, cat, dog]
[cat, dog, rat]
[rat, cat, dog]
{rat=Fuzzy, cat=Rags, dog=Spot}
{cat=Rags, dog=Spot, rat=Fuzzy}
{rat=Fuzzy, cat=Rags, dog=Spot}
 */
```

`ArrayList`和`LinkedList`都是List类型，从输出可以看出，它们都按照被插入的顺序保存元素。两者的不同之处不仅在于执行某些类型的操作时的性能，而且`LinkedList`包含的操作也多于`ArrayList`。

`TreeSet`按照比较结果的升序保存对象；`LinkedHashSet`按照被添加的顺序保存对象。

`TreeMap`按照比较结果的升序保存键，而`LinkedHashMap`则按照插入顺序保存键。

## 5.List

有两种类型的List：

* 基本的`ArrayList`，它长于随机访问元素，但是在`List`的中间插入和移除元素时比较慢。
* `LinkedList`，它通过代价较低的在`List`中间进行的插入和删除操作，提供了优化的顺序访问。`LinkedList`在随机访问方面相对比较慢，但是它的特性集较`ArrayList`更大。

`pets`:

```java
public class Individual implements Comparable<Individual> {
  private static long counter = 0;
  private final long id = counter++;
  private String name;
  public Individual(String name) { this.name = name; }
  // 'name' is optional:
  public Individual() {}
  public String toString() {
    return getClass().getSimpleName() +
      (name == null ? "" : " " + name);
  }
  public long id() { return id; }
  public boolean equals(Object o) {
    return o instanceof Individual &&
      id == ((Individual)o).id;
  }
  public int hashCode() {
    int result = 17;
    if(name != null)
      result = 37 * result + name.hashCode();
    result = 37 * result + (int)id;
    return result;
  }
  public int compareTo(Individual arg) {
    // Compare by class name first:
    String first = getClass().getSimpleName();
    String argFirst = arg.getClass().getSimpleName();
    int firstCompare = first.compareTo(argFirst);
    if(firstCompare != 0)
    return firstCompare;
    if(name != null && arg.name != null) {
      int secondCompare = name.compareTo(arg.name);
      if(secondCompare != 0)
        return secondCompare;
    }
    return (arg.id < id ? -1 : (arg.id == id ? 0 : 1));
  }
}

public class Pet extends Individual {
  public Pet(String name) { super(name); }
  public Pet() { super(); }
}
//🐱
public class Cat extends Pet {
  public Cat(String name) { super(name); }
  public Cat() { super(); }
}
//🐶
public class Dog extends Pet {
  public Dog(String name) { super(name); }
  public Dog() { super(); }
}
//鼠类
public class Rodent extends Pet {
  public Rodent(String name) { super(name); }
  public Rodent() { super(); }
}
//马恩岛🐱
public class Manx extends Cat {
  public Manx(String name) { super(name); }
  public Manx() { super(); }
}
//埃及🐱
public class EgyptianMau extends Cat {
  public EgyptianMau(String name) { super(name); }
  public EgyptianMau() { super(); }
}
//威尔士🐱
public class Cymric extends Manx {
  public Cymric(String name) { super(name); }
  public Cymric() { super(); }
}
//混血🐶
public class Mutt extends Dog {
  public Mutt(String name) { super(name); }
  public Mutt() { super(); }
}
//哈巴🐶
public class Pug extends Dog {
  public Pug(String name) { super(name); }
  public Pug() { super(); }
}
//大🐭
public class Rat extends Rodent {
  public Rat(String name) { super(name); }
  public Rat() { super(); }
} 
//仓鼠
public class Hamster extends Rodent {
  public Hamster(String name) { super(name); }
  public Hamster() { super(); }
}
//小🐭
public class Mouse extends Rodent {
  public Mouse(String name) { super(name); }
  public Mouse() { super(); }
}

public abstract class PetCreator {
  private Random rand = new Random(47);
  // The List of the different types of Pet to create:
  public abstract List<Class<? extends Pet>> types();
  public Pet randomPet() { // Create one random Pet
    int n = rand.nextInt(types().size());
    try {
      return types().get(n).newInstance();
    } catch(InstantiationException e) {
      throw new RuntimeException(e);
    } catch(IllegalAccessException e) {
      throw new RuntimeException(e);
    }
  }    
  public Pet[] createArray(int size) {
    Pet[] result = new Pet[size];
    for(int i = 0; i < size; i++)
      result[i] = randomPet();
    return result;
  }
  public ArrayList<Pet> arrayList(int size) {
    ArrayList<Pet> result = new ArrayList<Pet>();
    Collections.addAll(result, createArray(size));
    return result;
  }
}

public class ForNameCreator extends PetCreator {
  private static List<Class<? extends Pet>> types =
    new ArrayList<Class<? extends Pet>>();
  // Types that you want to be randomly created:
  private static String[] typeNames = {
    "pets.Mutt",
    "pets.Pug",
    "pets.EgyptianMau",
    "pets.Manx",
    "pets.Cymric",
    "pets.Rat",
    "pets.Mouse",
    "pets.Hamster"
  };    
  @SuppressWarnings("unchecked")
  private static void loader() {
    try {
      for(String name : typeNames)
        types.add(
          (Class<? extends Pet>)Class.forName(name));
    } catch(ClassNotFoundException e) {
      throw new RuntimeException(e);
    }
  }
  static { loader(); }
  public List<Class<? extends Pet>> types() {return types;}
}
public class LiteralPetCreator extends PetCreator {
  // No try block needed.
  @SuppressWarnings("unchecked")
  public static final List<Class<? extends Pet>> allTypes =
    Collections.unmodifiableList(Arrays.asList(
      Pet.class, Dog.class, Cat.class,  Rodent.class,
      Mutt.class, Pug.class, EgyptianMau.class, Manx.class,
      Cymric.class, Rat.class, Mouse.class,Hamster.class));
  // Types for random creation:
  private static final List<Class<? extends Pet>> types =
    allTypes.subList(allTypes.indexOf(Mutt.class),
      allTypes.size());
  public List<Class<? extends Pet>> types() {
    return types;
  }    
  public static void main(String[] args) {
    System.out.println(types);
  }
} 


public class Pets {
  public static final PetCreator creator =
    new LiteralPetCreator();
  public static Pet randomPet() {
    return creator.randomPet();
  }
  public static Pet[] createArray(int size) {
    return creator.createArray(size);
  }
  public static ArrayList<Pet> arrayList(int size) {
    return creator.arrayList(size);
  }
}
```

```java
public class ListFeatures {
    public static void main(String[] args) {
        Random random = new Random(47);
        List<Pet> pets = Pets.arrayList(7);
        System.out.println("1: " + pets);
        Hamster h = new Hamster();
        pets.add(h);
        System.out.println("2: " + pets);
        System.out.println("3: " + pets.contains(h));//判断某个对象是否在列表中
        pets.remove(h); //移除
        Pet p = pets.get(2);
        System.out.println("4: " + p + " " + pets.indexOf(p)); //获取索引
        //当确定一个元素是否属于某个List,发现某个元素的索引,
        // 以及从某个List中移除一个元素时,都会用到equals()方法
        Pet cymric = new Cymric();
        System.out.println("5: " + pets.indexOf(cymric));
        System.out.println("6: " + pets.remove(cymric));
        System.out.println("7: " + pets.remove(p));
        System.out.println("8: " + pets);
        pets.add(3, new Mouse());
        System.out.println("9: " + pets);
        List<Pet> sub = pets.subList(1, 4);
        System.out.println("subList: " + sub);
        System.out.println("10: " + pets.containsAll(sub));
        //对子列表打乱和排序都不影响containsAll()方法
        Collections.sort(sub);
        System.out.println("sorted subList:" + sub);
        System.out.println("11: " + pets.containsAll(sub));
        Collections.shuffle(sub, random);
        System.out.println("shuffled subList:");
        System.out.println("12: " + pets.containsAll(sub));
        List<Pet> copy = new ArrayList<Pet>(pets);
        sub = Arrays.asList(pets.get(1), pets.get(4));
        System.out.println("sub: " + sub);
        //retainAll()方法是一种有效的"交集"操作,保留了所有同时在copy与sub中的元素
        copy.retainAll(sub);
        System.out.println("13: " + copy);
        copy = new ArrayList<Pet>(pets);
        copy.remove(2);
        System.out.println("14: " + copy);
        copy.removeAll(sub);
        System.out.println("15: " + copy);
        copy.set(1, new Mouse());
        System.out.println("16: " + copy);
        copy.addAll(2, sub);
        System.out.println("17: " + copy);
        System.out.println("18: " + pets.isEmpty());
        pets.clear();
        System.out.println("19: " + pets);
        System.out.println("20: " + pets.isEmpty());
        pets.addAll(Pets.arrayList(4));
        System.out.println("21: " + pets);
        Object[] o = pets.toArray();
        System.out.println("22: " + o[3]);
        Pet[] pa = pets.toArray(new Pet[0]); //将集合转换为数组
        System.out.println("23: " + pa[3].id());
    }
}
```

## 6.迭代器

如果原本是对着`List`编码的，但是后来发现如果能够把相同的代码应用于`Set`，将会显得非常方便，此时应该怎么做？或者打算从头开始编写通用的代码，它们只是使用容器，不知道或者说不关心容器的类型，那么如何才能不重写代码就可以应用于不同类型的容器？

`迭代器`的概念可以用于达成此目的。迭代器是一个对象，它的工作是遍历并选择序列中的对象，而客户端程序员不必知道或关心该序列底层的结构。

`Java`的`Iterator`只能单向移动，这个`Iterator`只能用来：

* 使用方法`iterator()`要求容器返回一个`Iterator`。`Iterator`将准备好返回序列的第一个元素。
* 使用`next()`获得序列中的下一个元素。
* 使用`hasNext()`检查序列中是否还有元素。
* 使用`remove()`将迭代器新近返回的元素删除。

```java
public class SimpleIteration {
    public static void main(String[] args) {
        List<Pet> pets = Pets.arrayList(12);
        Iterator<Pet> it = pets.iterator();
        while (it.hasNext()) {
            Pet p = it.next();
            System.out.print(p.id() + ":" + p + " ");
        }
        System.out.println();
        for (Pet p : pets) {
            System.out.print(p.id() + ":" + p + " ");
        }
        System.out.println();
        it = pets.iterator();
        for (int i = 0; i < 6; i++) {
            it.next();
            it.remove();
        }
        System.out.println(pets);
    }

}
/*
0:Rat 1:Manx 2:Cymric 3:Mutt 4:Pug 5:Cymric 6:Pug 7:Manx 8:Cymric 9:Rat 10:EgyptianMau 11:Hamster
0:Rat 1:Manx 2:Cymric 3:Mutt 4:Pug 5:Cymric 6:Pug 7:Manx 8:Cymric 9:Rat 10:EgyptianMau 11:Hamster
[Pug, Manx, Cymric, Rat, EgyptianMau, Hamster]

 */
```

```java
public class CrossContainerIteration {
    public static void display(Iterator<Pet> it){
        while (it.hasNext()){
            Pet p = it.next();
            System.out.print(p.id() +":"+p +"");
        }
        System.out.println();
    }

    public static void main(String[] args) {
        ArrayList<Pet> pets = Pets.arrayList(8);
        LinkedList<Pet> petsLL = new LinkedList<Pet>(pets);
        HashSet<Pet> petsHS = new HashSet<Pet>(pets);
        TreeSet<Pet> petsTS = new TreeSet<Pet>(pets);
        display(pets.iterator());
        display(petsLL.iterator());
        display(petsHS.iterator());
        display(petsTS.iterator());
    }
}
/*
0:Rat1:Manx2:Cymric3:Mutt4:Pug5:Cymric6:Pug7:Manx
0:Rat1:Manx2:Cymric3:Mutt4:Pug5:Cymric6:Pug7:Manx
0:Rat1:Manx2:Cymric3:Mutt4:Pug5:Cymric6:Pug7:Manx
5:Cymric2:Cymric7:Manx1:Manx3:Mutt6:Pug4:Pug0:Rat
 */
```

`display()`方法不包含任何有关它所遍历的序列的类型信息，而这也展示了`Iterator`的真正威力：能够将遍历蓄力的操作与序列底层的结构分离。正由于此，我们有时会说：迭代器统一了对容器的访问方式。

### 6.1 ListIterator

`ListIterator`是一个更加强大的`Iterator`的子类型，它只能用于各种`List`类的访问。尽管`Iterator`只能向前移动，但是`ListIterator`可以双向移动。它还可以产生相对于迭代器在列表中指向的当前位置的前一个和后一个元素的索引，并且可以使用`set()`方法替换它访问过的最后一个元素。你可以通过调用`listIterator()`方法产生一个指向`List`开始处的`ListIterator`，并且还可以通过调用`listIterator(n)`方法创建一个一开始就指向列表索引为`n`的元素处的`ListIterator`。

```java
public class ListIteration {
    public static void main(String[] args) {
        List<Pet> pets = Pets.arrayList(8);
        ListIterator<Pet> it = pets.listIterator();
        while (it.hasNext()) {
            System.out.print(it.next() + ", " + it.nextIndex() + ", " + it.previousIndex() + "; ");
        }
        System.out.println();
        while (it.hasPrevious()) {
            System.out.print(it.previous().id() + " ");
        }
        System.out.println();
        System.out.println(pets);
        it = pets.listIterator(3);
        while (it.hasNext()) {
            it.next();
            it.set(Pets.randomPet());
        }
        System.out.println(pets);
    }

}
/*
Rat, 1, 0; Manx, 2, 1; Cymric, 3, 2; Mutt, 4, 3; Pug, 5, 4; Cymric, 6, 5; Pug, 7, 6; Manx, 8, 7; 
7 6 5 4 3 2 1 0 
[Rat, Manx, Cymric, Mutt, Pug, Cymric, Pug, Manx]
[Rat, Manx, Cymric, Cymric, Rat, EgyptianMau, Hamster, EgyptianMau]
 */
```

## 7.LinkedList

`LinkedList`添加了可以使其用作栈、队列或双端队列的方法。

这些方法中有些彼此之间只是名字有些差异，或者只存在些许差异，以使得这些名字在特定用法的上下文环境中更加适用。例如，`getFirst()`和`element()`完全一样，它们都返回列表的头，而并不移除它，如果`List`为空，则抛出`NoSuchElementException`。`peek()`方法与这两个方式只是稍有差异，它在列表为空时返回`null`。

`removeFirst()`和`remove()`也是完全一样的，它们移除并返回列表的头，而在列表为空时抛出`NoSuchElementException`。`poll()`稍有差异，它在列表为空时返回`null`。

`offer()`与`add()`和`addLast()`相同，它们都将某个元素插入到列表的尾部。

`removeLast()`移除并返回列表的最后一个元素。

```java
public class LinkedListFeatures {
    public static void main(String[] args) {
        LinkedList<Pet> pets = new LinkedList<Pet>(Pets.arrayList(5));
        System.out.println(pets);
        System.out.println("pets.getFirst(): " + pets.getFirst());
        System.out.println("pets.element(): " + pets.element());
        System.out.println("pets.peek(): " + pets.peek());
        System.out.println("pets.remove(): " + pets.remove());
        System.out.println("pets.removeFirst(): " + pets.removeFirst());
        System.out.println("pets.poll(): " + pets.poll());
        System.out.println(pets);
        pets.addFirst(new Rat());
        System.out.println("After addFirst(): " + pets);
        pets.offer(Pets.randomPet());
        System.out.println("After offer(): " + pets);
        pets.add(Pets.randomPet());
        System.out.println("After add(): " + pets);
        pets.addLast(Pets.randomPet());
        System.out.println("After addLast(): " + pets);
        System.out.println("pets.removeLast(): " + pets.removeLast());
    }
}
/*
[Rat, Manx, Cymric, Mutt, Pug]
pets.getFirst(): Rat
pets.element(): Rat
pets.peek(): Rat
pets.remove(): Rat
pets.removeFirst(): Manx
pets.poll(): Cymric
[Mutt, Pug]
After addFirst(): [Rat, Mutt, Pug]
After offer(): [Rat, Mutt, Pug, Cymric]
After add(): [Rat, Mutt, Pug, Cymric, Pug]
After addLast(): [Rat, Mutt, Pug, Cymric, Pug, Manx]
pets.removeLast(): Manx
 */
```

## 8.Stack

”栈“通常是指“后进先出”（LIFO）的容器。有时候也被称为`叠加栈`，因为最后“压入”栈的元素，第一个”弹出“栈。

`LinkedList`具有能够直接实现栈的所有功能的方法，因此可以直接将`LinkedList`作为栈使用。

```java
public class Stack<T> {
    private LinkedList<T> storage = new LinkedList<T>();

    public void push(T v) {
        storage.addFirst(v);
    }

    public T peek() {
        return storage.getFirst();
    }

    public T pop() {
        return storage.removeFirst();
    }

    public boolean empty() {
        return storage.isEmpty();
    }

    @Override
    public String toString() {
        return storage.toString();
    }
}
```

```java
public class StackTest {
    public static void main(String[] args) {
        Stack<String> stack = new Stack<String>();
        for (String s : "My dog has fleas".split(" ")) {
            stack.push(s);
        }
        while (!stack.empty()){
            System.out.print(stack.pop()+" ");
        }
    }
}
/*
fleas has dog My
 */
```

`java.util`包中的`Stack`使用。

```java
public class StackTest {
    public static void main(String[] args) {
        java.util.Stack<String> stack = new java.util.Stack<String>();
        for (String s : "My dog has fleas".split(" ")) {
            stack.push(s);
        }
        while (!stack.empty()){
            System.out.print(stack.pop()+" ");
        }
    }
}
/*
fleas has dog My
 */
```

## 9.Set

`Set`具有与`Collection`完全一样的接口，因此没有任何额外的功能，不像前面有两个不同的`List`。

```java
public class SetOfInteger {
    public static void main(String[] args) {
        Random rand = new Random(47);
        Set<Integer> intset = new HashSet<Integer>();
        for (int i = 0; i < 10000; i++) {
            intset.add(rand.nextInt(30));
        }
        System.out.println(intset);
    }
}
/*
[0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17,
18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29]
 */
```

## 10.Map

```java
public class Statistics {
    public static void main(String[] args) {
        Random rand = new Random(47);
        Map<Integer,Integer> m = new HashMap<Integer, Integer>();
        for (int i = 0; i < 10000; i++) {
            int r = rand.nextInt(20);
            Integer freq = m.get(r);
            m.put(r,freq==null?1:freq+1);
        }
        System.out.println(m);
    }
}
/*
{0=481, 1=502, 2=489, 3=508, 4=481, 5=503, 6=519, 7=471, 8=468, 9=549, 10=513,
 11=531, 12=521, 13=506, 14=477, 15=497, 16=533, 17=509, 18=478, 19=464}

 */
```

```java
public class PetMap {
    public static void main(String[] args) {
        Map<String, Pet> petMap = new HashMap<String, Pet>();
        petMap.put("My Cat", new Cat("Molly"));
        petMap.put("My Dog", new Dog("Ginger"));
        petMap.put("My Hamster", new Hamster("Bosco"));
        System.out.println(petMap);
        Pet dog = petMap.get("My Dog");
        System.out.println(dog);
        System.out.println(petMap.containsKey("My Dog"));
        System.out.println(petMap.containsValue(dog));
    }
}
/*
{My Dog=Dog Ginger, My Cat=Cat Molly, My Hamster=Hamster Bosco}
Dog Ginger
true
true
 */
```

## Queue

队列是一种典型的`先进先出（FIFO）`的容器。

`LinkedList`提供了方法以支持队列的行为，并且它实现了`Queue`接口，因此`LinkedList`可以用作`Queue`的一种实现。

```java
public class QueueDemo {
    public static void printQ(Queue queue){
        while (queue.peek()!=null){
            System.out.print(queue.remove() + " ");
        }
        System.out.println();
    }

    public static void main(String[] args) {
        Queue<Integer> queue = new LinkedList<Integer>();
        Random rand = new Random(47);
        for (int i = 0; i < 10; i++) {
            queue.offer(rand.nextInt(i+10));
        }
        printQ(queue);
        Queue<Character> qc = new LinkedList<Character>();
        for (char c : "Brontosaurus".toCharArray()) {
            qc.offer(c);
        }
        printQ(qc);
    }
}
/*
8 1 1 1 5 14 3 1 0 1
B r o n t o s a u r u s
 */
```

`offer()`方法是`Queue`相关的方法之一，它在允许的情况下，将一个元素插入到队尾，或者返回`false`。`peek()`和`element()`都将在不移除的情况下返回队头，但是`peek()`方法在队列为空时返回`null`，而`element()`会抛出`NoSuchElementException`异常。`poll()`和`remove()`方法将移除并返回队头，但是`poll()`在队列为空时返回`null`，而`remove()`会抛出`NoSuchElementException`异常。

### 11.1 PriorityQueue

`优先级队列`声明下一个弹出元素是最需要的元素。

当在`PriorityQueue`上调用`offer()`方法来插入一个对象时，这个对象会在队列中被排序。默认的排序将使用对象在队列中的`自然顺序`，但是你可以通过提供自己的`Comparator`来修改这个顺序。`PriorityQueue`可以确保你调用`peek()`、`poll()`和`remove()`方法时，获取的元素将是队列中优先级最高的元素。

```java
public class PriorityQueueDemo {
    public static void main(String[] args) {
        PriorityQueue<Integer> priorityQueue = new PriorityQueue<Integer>();
        Random rand = new Random();
        for (int i = 0; i < 10; i++) {
            priorityQueue.offer(rand.nextInt(i + 10));
        }
        QueueDemo.printQ(priorityQueue);
        List<Integer> ints = Arrays.asList(25, 22, 20, 18, 14, 9, 3, 1, 1, 2, 3, 9, 14, 18, 21, 23, 25);
        priorityQueue = new PriorityQueue<Integer>(ints);
        QueueDemo.printQ(priorityQueue);
        //反序
        priorityQueue = new PriorityQueue<Integer>(ints.size(), Collections.<Integer>reverseOrder());
        priorityQueue.addAll(ints);
        QueueDemo.printQ(priorityQueue);
        //空格也可以算作值，并且比字母的优先级高
        String fact = "EDUCATION SHOULD ESCHEW OBFUSCATION";
        List<String> strings = Arrays.asList(fact.split(""));
        PriorityQueue<String> stringPQ = new PriorityQueue<String>(strings);
        QueueDemo.printQ(stringPQ);
        stringPQ = new PriorityQueue<String>(
                strings.size(), Collections.<String>reverseOrder());
        stringPQ.addAll(strings);
        QueueDemo.printQ(stringPQ);
        Set<Character> charSet = new HashSet<Character>();
        for (char c : fact.toCharArray()) {
            charSet.add(c);
        }
        PriorityQueue<Character> characterPQ = new PriorityQueue<Character>(charSet);
        QueueDemo.printQ(characterPQ);
    }
}
/*
2 3 3 4 4 5 5 7 7 9 
1 1 2 3 3 9 9 14 14 18 18 20 21 22 23 25 25 
25 25 23 22 21 20 18 18 14 14 9 9 3 3 2 1 1 
      A A B C C C D D E E E F H H I I L N N O O O O S S S T T U U U W 
W U U U T T S S S O O O O N N L I I H H F E E E D D C C C B A A       
  A B C D E F H I L N O S T U W 
 */
```

## 12.Collection和Iterator

```java
public class InterfaceVsIterator {
    public static void display(Iterator<Pet> it) {
        while (it.hasNext()) {
            Pet p = it.next();
            System.out.print(p.id() + ":" + p + " ");
        }
        System.out.println();
    }

    public static void display(Collection<Pet> pets) {
        for (Pet pet : pets) {
            System.out.print(pet.id() + ":" + pet + " ");
        }
        System.out.println();
    }

    public static void main(String[] args) {
        List<Pet> petList = Pets.arrayList(8);
        Set<Pet> petSet = new HashSet<Pet>(petList);
        Map<String, Pet> petMap = new LinkedHashMap<String, Pet>();
        String[] names = ("Ralph, Eric, Robin, Lacey, " +
                "Britney, Sam, Spot, Fluffy").split(", ");
        for (int i = 0; i < names.length; i++) {
            petMap.put(names[i],petList.get(i));
        }
        display(petList);
        display(petSet);
        display(petList.iterator());
        System.out.println(petMap);
        System.out.println(petMap.keySet());
        display(petMap.values());
        display(petMap.values().iterator());
    }
}
/*
0:Rat 1:Manx 2:Cymric 3:Mutt 4:Pug 5:Cymric 6:Pug 7:Manx 
0:Rat 1:Manx 2:Cymric 3:Mutt 4:Pug 5:Cymric 6:Pug 7:Manx 
0:Rat 1:Manx 2:Cymric 3:Mutt 4:Pug 5:Cymric 6:Pug 7:Manx 
{Ralph=Rat, Eric=Manx, Robin=Cymric, Lacey=Mutt, Britney=Pug, Sam=Cymric, Spot=Pug, Fluffy=Manx}
[Ralph, Eric, Robin, Lacey, Britney, Sam, Spot, Fluffy]
0:Rat 1:Manx 2:Cymric 3:Mutt 4:Pug 5:Cymric 6:Pug 7:Manx 
0:Rat 1:Manx 2:Cymric 3:Mutt 4:Pug 5:Cymric 6:Pug 7:Manx 
 */
```

```java
public class CollectionSequence extends AbstractCollection<Pet> {
    private Pet[] pets = Pets.createArray(8);

    @Override
    public int size() {
        return pets.length;
    }

    @Override
    public Iterator<Pet> iterator() {
        return new Iterator<Pet>() {
            private int index = 0;
            public boolean hasNext() {
                return false;
            }

            public Pet next() {
                return pets[index++];
            }

            public void remove() {
                throw new UnsupportedOperationException();
            }
        };
    }

    public static void main(String[] args) {
        CollectionSequence c = new CollectionSequence();
        InterfaceVsIterator.display(c);
        InterfaceVsIterator.display(c.iterator());
    }
}
```

```java
class PetSequence{
    protected Pet[] pets = Pets.createArray(8);
}
public class NonCollectionSequence extends PetSequence {
    public Iterator<Pet> iterator(){
        return new Iterator<Pet>() {
            private int index =0;
            public boolean hasNext() {
                return index < pets.length;
            }

            public Pet next() {
                return pets[index++];
            }

            public void remove() {
                throw new UnsupportedOperationException();
            }
        };
    }

    public static void main(String[] args) {
        NonCollectionSequence nc = new NonCollectionSequence();
        InterfaceVsIterator.display(nc.iterator());
    }
}
/*
0:Rat 1:Manx 2:Cymric 3:Mutt 4:Pug 5:Cymric 6:Pug 7:Manx 

 */
```

## 13.Foreach与迭代器

```java
public class ForEachCollections {
    public static void main(String[] args) {
        Collection<String> cs = new LinkedHashSet<String>();
        Collections.addAll(cs, "Take the long way home".split(" "));
        for (String s : cs) {
            System.out.print("'" + s + "' ");
        }
    }
}
/*
'Take' 'the' 'long' 'way' 'home'
 */
```

`Java SE5`引入了新的被称为`Iterable`的接口，该接口包含一个能够产生`Iterator`的`iterator()`方法，并且`Iterable`接口被foreach用来在序列中移动。因此如果你创建了任何实现`Iterable`的类，都可以将它用于`foreach`语句中：

```java
public class IterableClass implements Iterable<String> {

    protected String[] words = ("And that is how " +
            "we know the Earth to be banana-shaped.").split(" ");
    public Iterator<String> iterator() {
        return new Iterator<String>() {
            private int index = 0;

            public boolean hasNext() {
                return index<words.length;
            }

            public String next() {
                return words[index++];
            }

            public void remove() {
                throw new UnsupportedOperationException();
            }
        };
    }

    public static void main(String[] args) {
        for (String s : new IterableClass()) {
            System.out.println(s + " ");
        }
    }
}
/*
'Take' 'the' 'long' 'way' 'home' 
 */
```

显示所有的操作系统环境变量：

```java
public class EnvironmentVariables {
    public static void main(String[] args) {
        for (Map.Entry<String, String> entry : System.getenv().entrySet()) {
            System.out.println(entry.getKey()+":"+entry.getValue());
        }
    }
}
```

`foreach`语句可以用于数组或其他任何`Iterable`，但是这并不意味着数组肯定也是一个`Iterable`。

```java
public class ArrayIsNotIterable {
    static <T> void test(Iterable<T> ib) {
        for (T t : ib) {
            System.out.print(t + " ");
        }
    }

    public static void main(String[] args) {
        test(Arrays.asList(1, 2, 3));
        String[] strings = {"A", "B", "C"};
        //
       // test(strings);

        test(Arrays.asList(strings));
    }
}
/*
1 2 3 A B C 
 */
```

### 13.1 适配器方法惯用法

```java
public class ReversibleArrayList<T> extends ArrayList<T> {
    public ReversibleArrayList(Collection<T> c){
        super(c);
    }
    public Iterable<T> reversed(){
        return new Iterable<T>() {
            public Iterator<T> iterator() {
                return new Iterator<T>() {
                    int current = size() -1;

                    public boolean hasNext() {
                        return current > -1;
                    }

                    public T next() {
                        return get(current--);
                    }

                    public void remove() {
                        throw new UnsupportedOperationException();
                    }
                };
            }
        };
    }
}

public class AdapterMethodiom {
    public static void main(String[] args) {
        ReversibleArrayList<String> ral = new ReversibleArrayList<String>(
                Arrays.asList("To be or not to be".split(" ")));
        for (String s : ral) {
            System.out.print(s + " ");
        }
        System.out.println();
        for (String s : ral.reversed()) {
            System.out.print(s + " ");
        }
    }
}
/*
To be or not to be
be to not or be To
 */
```

```java
public class MultiIterableClass extends IterableClass {
    public Iterable<String> reversed() {
        return new Iterable<String>() {
            public Iterator<String> iterator() {
                return new Iterator<String>() {
                    int current = words.length - 1;

                    public boolean hasNext() {
                        return current > -1;
                    }

                    public String next() {
                        return words[current--];
                    }

                    public void remove() {
                        throw new UnsupportedOperationException();
                    }
                };
            }
        };
    }

    public Iterable<String> randomized() {
        return new Iterable<String>() {
            public Iterator<String> iterator() {
                List<String> shuffled = new ArrayList<String>(Arrays.asList(words));
                Collections.shuffle(shuffled, new Random(47));
                return shuffled.iterator();
            }
        };
    }

    public static void main(String[] args) {
        MultiIterableClass mic = new MultiIterableClass();
        for (String s : mic.reversed()) {
            System.out.print(s + " ");
        }
        System.out.println();
        for (String s : mic.randomized()) {
            System.out.print(s + " ");
        }
        System.out.println();
        for (String s : mic) {
            System.out.print(s + " ");
        }
    }
}
/*
banana-shaped. be to Earth the know we how is that And 
is banana-shaped. Earth that how the be And we know to 
And that is how we know the Earth to be banana-shaped. 
 */
```

```java
public class ModifyingArraysAsList {
    public static void main(String[] args) {
        Random rand = new Random(47);
        Integer[] ia = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
        List<Integer> list1 = new ArrayList<Integer>(Arrays.asList(ia));
        System.out.println("Before shuffling: " + list1);
        Collections.shuffle(list1, rand);
        System.out.println("After shuffling: " + list1);
        System.out.println("array: " + Arrays.toString(ia));
        List<Integer> list2 = Arrays.asList(ia);
        System.out.println("Before shuffling: " + list2);
        Collections.shuffle(list2, rand);
        System.out.println("After shuffling: " + list2);
        System.out.println("array: " + Arrays.toString(ia));

    }
}
/*
Before shuffling: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
After shuffling: [4, 6, 3, 1, 8, 7, 2, 5, 10, 9]
array: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
Before shuffling: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
After shuffling: [9, 1, 6, 3, 7, 2, 5, 10, 4, 8]
array: [9, 1, 6, 3, 7, 2, 5, 10, 4, 8]
 */
```

在第一种情况中，`Arrays.asList()`的输出被传递给了`ArrayList()`的构造器，这将创建一个引用`ia`的元素的`ArrayList`，因此打乱这些引用不会修改该数组。但是，如果直接使用`Arrays.asList(ia)`的结果，这种打乱就会修改`ia`的顺序。

