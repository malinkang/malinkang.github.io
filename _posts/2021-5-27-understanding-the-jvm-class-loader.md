---
title: 《深入理解JVM》第7章虚拟机类加载机制
date: 2016-08-27 13:39:36
tags: [深入理解JVM]
categories: [Java,JVM]
---

## 类加载的时机

类从被加载到虚拟机内存中开始，到卸载出内存为止，它的整个生命周期包括：加载（Loading）、验证（Verification）、准备（Preparation）、解析（Resolution）、初始化（Initialization）、使用（Using）和卸载（Unloading）7个阶段。其中验证、准备、解析3个部分统称为连接（Linking）。


加载、验证、准备、初始化和卸载这5个阶段的顺序是确定的，类的加载过程必须按照这种顺序按部就班地开始，而解析阶段则不一定：它在某些情况下可以在初始化阶段之后再开始，这是为了支持Java语言的运行时绑定（也称为动态绑定或晚期绑定）。

什么情况下需要开始类加载过程的第一个阶段：加载？Java虚拟机规范中并没有进行强制约束，这点可以交给虚拟机的具体实现来自由把握。但是对于初始化阶段，虚拟机规范则是严格规定了有且只有5种情况必须立即对类进行“初始化”。

1. 遇到new、getstatic、putstatic或invokestatic这4条字节码指令时，如果类没有进行过初始化，则需要先触发其初始化。生成这4条指令的最常见的Java代码场景是：使用new关键字实例化对象的时候、读取或设置一个类的静态字段（被final修饰、已在编译期把结果放入常量池的静态字段除外）的时候，以及调用一个类的静态方法的时候。
2. 使用java.lang.reflect包的方法对类进行反射调用的时候，如果类没有进行过初始化，则需要先触发其初始化。
3. 当初始化一个类的时候，如果发现其父类还没有进行过初始化，则需要先触发其父类的初始化。
4. 当虚拟机启动时，用户需要指定一个要执行的主类（包含main()方法的那个类），虚拟机会先初始化这个主类。
5. 当使用JDK 1.7的动态语言支持时，如果一个java.lang.invoke.MethodHandle实例最后的解析结果REF_getStatic、REF_putStatic、REF_invokeStatic的方法句柄，并且这个方法句柄所对应的类没有进行过初始化，则需要先触发其初始化。

对于这5种会触发类进行初始化的场景，虚拟机规范中使用了一个很强烈的限定语：“有且只有”，这5种场景中的行为称为对一个类进行主动引用。除此之外，所有引用类的方式都不会触发初始化，称为被动引用。



被动引用例子一：
```java
public class SuperClass {
    static {
        System.out.println("SuperClass init!");
    }
    public static int value = 123;
}
```

```java
public class SubClass extends SuperClass{
    static {
        System.out.println("SubClass init!");
    }
}
```

```java
public class NotInitialization {
    public static void main(String[] args) {
        System.out.println(SubClass.value);
    }
}
```

上述代码运行之后，只会输出“SuperClass init!”，而不会输出“SubClass init!”。对于静态字段，只有直接定义这个字段的类才会被初始化，因此通过其子类来引用父类中定义的静态字段，只会触发父类的初始化而不会触发子类的初始化。至于是否要触发子类的加载和验证，在虚拟机规范中并未明确规定，这点取决于虚拟机的具体实现。对于Sun HotSpot虚拟机来说，可通过`-XX:+TraceClassLoading`参数观察到此操作会导致子类的加载。


被动引用例子二：

```java
public class NotInitialization {
    public static void main(String[] args) {
        SuperClass[] sca = new SuperClass[10];
    }
}
```

运行之后发现没有输出“SuperClass init!”，说明并没有触发类SuperClass的初始化阶段。但是这段代码里面触发了另外一个名为“[LSuperClass”的类的初始化阶段，对于用户代码来说，这并不是一个合法的类名称，它是一个由虚拟机自动生成的、直接继承于java.lang.Object的子类，创建动作由字节码指令newarray触发。

这个类代表了一个元素类型为SuperClass的一维数组，数组中应有的属性和方法（用户可直接使用的只有被修饰为public的length属性和clone()方法）都实现在这个类里。Java语言中对数组的访问比C/C++相对安全是因为这个类封装了数组元素的访问方法，而C/C++直接翻译为对数组指针的移动。在Java语言中，当检查到发生数组越界时会抛出java.lang.ArrayIndexOutOfBoundsException异常。

被动引用例子三：

```java
public class ConstClass {
    static {
        System.out.println("ConstClass init!");
    }
    public static final String HELLO_WORLD = "hello world";
}
public class NotInitialization {
    public static void main(String[] args) {
        System.out.println(ConstClass.HELLO_WORLD);
    }
}
```

上述代码运行之后，也没有输出“ConstClass init!”，这是因为虽然在Java源码中引用了ConstClass类中的常量HELLOWORLD，但其实在编译阶段通过常量传播优化，已经将此常量的值“hello world”存储到了NotInitialization类的常量池中，以后NotInitialization对常量ConstClass.HELLOWORLD的引用实际都被转化为NotInitialization类对自身常量池的引用了。也就是说，实际上NotInitialization的Class文件之中并没有ConstClass类的符号引用入口，这两个类在编译成Class之后就不存在任何联系了。


接口的加载过程与类加载过程稍有一些不同，针对接口需要做一些特殊说明：接口也有初始化过程，这点与类是一致的，上面的代码都是用静态语句块“static{}”来输出初始化信息的，而接口中不能使用“static{}”语句块，但编译器仍然会为接口生成“<clinit>()”类构造器[插图]，用于初始化接口中所定义的成员变量。接口与类真正有所区别的是前面讲述的5种“有且仅有”需要开始初始化场景中的第3种：当一个类在初始化时，要求其父类全部都已经初始化过了，但是一个接口在初始化时，并不要求其父接口全部都完成了初始化，只有在真正使用到父接口的时候（如引用接口中定义的常量）才会初始化。

## 类的加载过程

“加载”是“类加载”（Class Loading）过程的一个阶段，希望读者没有混淆这两个看起来很相似的名词。在加载阶段，虚拟机需要完成以下3件事情：

1. 通过一个类的全限定名来获取定义此类的二进制字节流。
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。
3. 在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口。

相对于类加载过程的其他阶段，一个非数组类的加载阶段（准确地说，是加载阶段中获取类的二进制字节流的动作）是开发人员可控性最强的，因为加载阶段既可以使用系统提供的引导类加载器来完成，也可以由用户自定义的类加载器去完成，开发人员可以通过定义自己的类加载器去控制字节流的获取方式（即重写一个类加载器的loadClass()方法）。

加载阶段完成后，虚拟机外部的二进制字节流就按照虚拟机所需的格式存储在方法区之中，方法区中的数据存储格式由虚拟机实现自行定义，虚拟机规范未规定此区域的具体数据结构。然后在内存中实例化一个java.lang.Class类的对象（并没有明确规定是在Java堆中，对于HotSpot虚拟机而言，Class对象比较特殊，它虽然是对象，但是存放在方法区里面），这个对象将作为程序访问方法区中的这些类型数据的外部接口。

加载阶段与连接阶段的部分内容（如一部分字节码文件格式验证动作）是交叉进行的，加载阶段尚未完成，连接阶段可能已经开始，但这些夹在加载阶段之中进行的动作，仍然属于连接阶段的内容，这两个阶段的开始时间仍然保持着固定的先后顺序。

## 验证

## 准备

## 解析

## 类加载器





一个完整的 Java 程序是由多个 .class 文件组成的，在程序运行过程中，需要将这些 .class 文件加载到 JVM 中才可以使用。而负责加载这些 .class 文件的就是类加载器（ClassLoader）。

在 Java 程序启动的时候，并不会一次性加载程序中所有的 .class 文件，而是在程序的运行过程中，动态地加载相应的类到内存中。


通常情况下,Java 程序中的 .class 文件会在以下 2 种情况下被 ClassLoader 主动加载到内存中：

* 调用类构造器

* 调用类中的静态（static）变量或者静态方法

Java虚拟机的角度来讲，只存在两种不同的类加载器：一种是启动类加载器（Bootstrap ClassLoader），这个类加载器使用C++语言实现，是虚拟机自身的一部分；另一种就是所有其他的类加载器，这些类加载器都由Java语言实现，独立于虚拟机外部，并且全都继承自抽象类java.lang.ClassLoader。

从Java开发人员的角度来看，类加载器还可以划分得更细致一些，绝大部分Java程序都会使用到以下3种系统提供的类加载器。

* 启动类加载器（Bootstrap ClassLoader）：前面已经介绍过，这个类将器负责将存放在<JAVA_HOME>\lib目录中的，或者被-Xbootclasspath参数所指定的路径中的，并且是虚拟机识别的（仅按照文件名识别，如rt.jar，名字不符合的类库即使放在lib目录中也不会被加载）类库加载到虚拟机内存中。启动类加载器无法被Java程序直接引用，用户在编写自定义类加载器时，如果需要把加载请求委派给引导类加载器，那直接使用null代替即可，如代码清单7-9所示为java.lang.ClassLoader.getClassLoader()方法的代码片段。
* 扩展类加载器（Extension ClassLoader）：这个加载器由`sun.misc.Launcher$ExtClassLoader`实现，它负责加载\lib\ext目录中的，或者被java.ext.dirs系统变量所指定的路径中的所有类库，开发者可以直接使用扩展类加载器。

* 应用程序类加载器（Application ClassLoader）：这个类加载器由`sun.misc.Launcher$App-ClassLoader`实现。由于这个类加载器是ClassLoader中的getSystemClassLoader()方法的返回值，所以一般也称它为系统类加载器。它负责加载用户类路径（ClassPath）上所指定的类库，开发者可以直接使用这个类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。
我们的应用程序都是由这3种类加载器互相配合进行加载的，如果有必要，还可以加入自己定义的类加载器。这些类加载器之间的关系一般如图所示。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/79019407ad4d4506b332f1d263ca0c1b~tplv-k3u1fbpfcp-zoom-1.image)

### 双亲委派模型

图中展示的各种类加载器之间的层次关系被称为类加载器的“双亲委派模型（Parents Delegation Model）”。双亲委派模型要求除了顶层的启动类加载器外，其余的类加载器都应有自己的父类加载器。不过这里类加载器之间的父子关系一般不是以继承（Inheritance）的关系来实现的，而是通常使用组合。

双亲委派模型的工作过程是：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到最顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类）时，子加载器才会尝试自己去完成加载。

使用双亲委派模型来组织类加载器之间的关系，一个显而易见的好处就是Java中的类随着它的类加载器一起具备了一种带有优先级的层次关系。例如类java.lang.Object，它存放在rt.jar之中，无论哪一个类加载器要加载这个类，最终都是委派给处于模型最顶端的启动类加载器进行加载，因此Object类在程序的各种类加载器环境中都能够保证是同一个类。反之，如果没有使用双亲委派模型，都由各个类加载器自行去加载的话，如果用户自己也编写了一个名为java.lang.Object的类，并放在程序的ClassPath中，那系统中就会出现多个不同的Object类，Java类型体系中最基础的行为也就无从保证，应用程序将会变得一片混乱。

```java
//ClassLoader
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
        // First, check if the class has already been loaded
        //判断是否已经被加载
        //①
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            try {  //没有加载调用父加载器加载
                if (parent != null) {//②
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);//③
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }
            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                c = findClass(name);//④
            }
        }
        return c;
}
```

1. 判断该 Class 是否已加载，如果已加载，则直接将该 Class 返回。

2. 如果该 Class 没有被加载过，则判断 parent 是否为空，如果不为空则将加载的任务委托给parent。

3. 如果 parent == null，则直接调用 BootstrapClassLoader 加载该类。

4. 如果 parent 或者 BootstrapClassLoader 都没有加载成功，则调用当前 ClassLoader 的 findClass 方法继续尝试加载。

## Android中的ClassLoader

本质上，Android 和传统的 JVM 是一样的，也需要通过 ClassLoader 将目标类加载到内存，类加载器之间也符合双亲委派模型。但是在 Android 中， ClassLoader 的加载细节有略微的差别。



在 Android 虚拟机里是无法直接运行 .class 文件的，Android 会将所有的 .class 文件转换成一个 .dex 文件，并且 Android 将加载 .dex 文件的实现封装在 BaseDexClassLoader 中，而我们一般只使用它的两个子类：PathClassLoader 和 DexClassLoader。

### PathClassLoader

PathClassLoader 用来加载系统 apk 和被安装到手机中的 apk 内的 dex 文件。它的 2 个构造函数如下：


参数说明：

* dexPath：dex 文件路径，或者包含 dex 文件的 jar 包路径；

* librarySearchPath：C/C++ native 库的路径。

PathClassLoader 里面除了这 2 个构造方法以外就没有其他的代码了，具体的实现都是在 BaseDexClassLoader 里面，其 dexPath 比较受限制，一般是已经安装应用的 apk 文件路径。

### DexClassLoader

对比 PathClassLoader 只能加载已经安装应用的 dex 或 apk 文件，DexClassLoader 则没有此限制，可以从 SD 卡上加载包含 class.dex 的 .jar 和 .apk 文件

### BaseDexClassLoader

先来看一眼 BaseClassLoader 的结构：

其中有个重要的字段 private final DexPathList pathList ，其继承 ClassLoader 实现的 findClass() 、findResource() 均是基于 pathList 来实现的。

`DexPathList`构造函数

```java
DexPathList(ClassLoader definingContext, String dexPath,
        String librarySearchPath, File optimizedDirectory, boolean isTrusted) {
        if (definingContext == null) {
            throw new NullPointerException("definingContext == null");
        }

        if (dexPath == null) {
            throw new NullPointerException("dexPath == null");
        }

        if (optimizedDirectory != null) {
            if (!optimizedDirectory.exists())  {
                throw new IllegalArgumentException(
                        "optimizedDirectory doesn't exist: "
                        + optimizedDirectory);
            }

            if (!(optimizedDirectory.canRead()
                            && optimizedDirectory.canWrite())) {
                throw new IllegalArgumentException(
                        "optimizedDirectory not readable/writable: "
                        + optimizedDirectory);
            }
        }

        this.definingContext = definingContext;

        ArrayList<IOException> suppressedExceptions = new ArrayList<IOException>();
        // save dexPath for BaseDexClassLoader
        this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
                                           suppressedExceptions, definingContext, isTrusted);

        // Native libraries may exist in both the system and
        // application library paths, and we use this search order:
        //
        //   1. This class loader's library path for application libraries (librarySearchPath):
        //   1.1. Native library directories
        //   1.2. Path to libraries in apk-files
        //   2. The VM's library path from the system property for system libraries
        //      also known as java.library.path
        //
        // This order was reversed prior to Gingerbread; see http://b/2933456.
        this.nativeLibraryDirectories = splitPaths(librarySearchPath, false);
        this.systemNativeLibraryDirectories =
                splitPaths(System.getProperty("java.library.path"), true);
        this.nativeLibraryPathElements = makePathElements(getAllNativeLibraryDirectories());

        if (suppressedExceptions.size() > 0) {
            this.dexElementsSuppressedExceptions =
                suppressedExceptions.toArray(new IOException[suppressedExceptions.size()]);
        } else {
            dexElementsSuppressedExceptions = null;
        }
}
```
接受之前传进来的包含 dex 的 apk/jar/dex 的路径集、native 库的路径集和缓存优化的 dex 文件的路径，然后调用 makePathElements() 方法生成一个 Element[] dexElements 数组，Element 是 DexPathList 的一个嵌套类：
`makeDexElements`方法。
```java
 private static Element[] makeDexElements(List<File> files, File optimizedDirectory,
         List<IOException> suppressedExceptions, ClassLoader loader, boolean isTrusted) {
   Element[] elements = new Element[files.size()]; //创建数组
   int elementsPos = 0;
   /*
    * Open all files and load the (direct or contained) dex files up front.
    */
   for (File file : files) { //遍历文件
       if (file.isDirectory()) {//如果是文件夹直接创建Element给数组赋值
           // We support directories for looking up resources. Looking up resources in
           // directories is useful for running libcore tests.
           elements[elementsPos++] = new Element(file);
       } else if (file.isFile()) {
           String name = file.getName();
           DexFile dex = null;
           if (name.endsWith(DEX_SUFFIX)) {//如果是dex文件
               // Raw dex file (not inside a zip/jar).
               try {//加载dex文件
                   dex = loadDexFile(file, optimizedDirectory, loader, elements);
                   if (dex != null) {
                       elements[elementsPos++] = new Element(dex, null);
                   }
               } catch (IOException suppressed) {
                   System.logE("Unable to load dex file: " + file, suppressed);
                   suppressedExceptions.add(suppressed);
               }
           } else {
               try {
                   dex = loadDexFile(file, optimizedDirectory, loader, elements);
               } catch (IOException suppressed) {
                   /*
                    * IOException might get thrown "legitimately" by the DexFile constructor if
                    * the zip file turns out to be resource-only (that is, no classes.dex file
                    * in it).
                    * Let dex == null and hang on to the exception to add to the tea-leaves for
                    * when findClass returns null.
                    */
                   suppressedExceptions.add(suppressed);
               }
               if (dex == null) {
                   elements[elementsPos++] = new Element(file);
               } else {
                   elements[elementsPos++] = new Element(dex, file);
               }
           }
           if (dex != null && isTrusted) {
             dex.setTrusted();
           }
       } else {
           System.logW("ClassLoader referenced unknown path: " + file);
       }
   }
   if (elementsPos != elements.length) {
       elements = Arrays.copyOf(elements, elementsPos);
   }
   return elements;
 }
```
 `BaseDexClassLoader`的`findClass`方法
 ```java
 @Override
protected Class<?> findClass(String name) throws ClassNotFoundException {
    // First, check whether the class is present in our shared libraries.
    if (sharedLibraryLoaders != null) {
        for (ClassLoader loader : sharedLibraryLoaders) {
            try {
                return loader.loadClass(name);
            } catch (ClassNotFoundException ignored) {
            }
        }
    }
    // Check whether the class in question is present in the dexPath that
    // this classloader operates on.
    List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
    Class c = pathList.findClass(name, suppressedExceptions);
    if (c == null) {
        ClassNotFoundException cnfe = new ClassNotFoundException(
                "Didn't find class \"" + name + "\" on path: " + pathList);
        for (Throwable t : suppressedExceptions) {
            cnfe.addSuppressed(t);
        }
        throw cnfe;
    }
    return c;
}
 ```
 接下来看以下 DexPathList 的 findClass() 方法，其根据传入的完整的类名来加载对应的 class，源码如下：
 ```java
 public Class findClass(String name, List<Throwable> suppressed) {
	// 遍历 dexElements 数组，依次寻找对应的 class，一旦找到就终止遍历
    for (Element element : dexElements) {
        DexFile dex = element.dexFile;
        if (dex != null) {
            Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
            if (clazz != null) {
                return clazz;
            }
        }
    }
    // 抛出异常
    if (dexElementsSuppressedExceptions != null) {
        suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
    }
    return null;
} 
 ```

 这里有关于热修复实现的一个点，就是将补丁 dex 文件放到 dexElements 数组前面，这样在加载 class 时，优先找到补丁包中的 dex 文件，加载到 class 之后就不再寻找，从而原来的 apk 文件中同名的类就不会再使用，从而达到修复的目的，虽然说起来较为简单，但是实现起来还有很多细节需要注意，本文先热身，后期再分析具体实现。

至此，BaseDexClassLader 寻找 class 的路线就清晰了：

1. 当传入一个完整的类名，调用 BaseDexClassLader 的 findClass(String name) 方法
2. BaseDexClassLader 的 findClass 方法会交给 DexPathList 的 findClass(String name, List<Throwable> suppressed 方法处理
3. 在 DexPathList 方法的内部，会遍历 dexFile ，通过 DexFile 的 dex.loadClassBinaryName(name, definingContext, suppressed) 来完成类的加载

##  参考

* [热修复入门：Android 中的 ClassLoader](https://jaeger.itscoder.com/android/2016/08/27/android-classloader.html)