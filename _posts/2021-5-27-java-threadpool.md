---
title: Java线程池
date: 2016-07-15 10:24:54
---
### 线程池介绍
合理利用线程池能够带来三个好处：

* 第一：降低资源消耗。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
* 第二：提高响应速度。当任务到达时，任务可以不需要等到线程创建就能立即执行。
* 第三：提高线程的可管理性。线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。但是要做到合理的利用线程池，必须对其原理了如指掌。

Java通过Executors提供四种线程池:

* newCachedThreadPool创建一个可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程，若无可回收，则新建线程。
* newFixedThreadPool 创建一个定长线程池，可控制线程最大并发数，超出的线程会在队列中等待。
* newScheduledThreadPool 创建一个定长线程池，支持定时及周期性任务执行。
* newSingleThreadExecutor 创建一个单线程化的线程池，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行。

### 使用execute提交任务

我们可以使用execute提交的任务，通过以下代码可知execute方法输入的任务是一个Runnable类的实例。

```java
private class TestRunnable implements Runnable{

    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName()+"线程被调用了");
    }
}
//创建一个线程池
ExecutorService executorService = Executors.newCachedThreadPool();
for (int i = 0; i < 5; i++) {
//提交Runnable任务
    executorService.execute(new TestRunnable());
    System.out.println("执行任务" + i);
}
executorService.shutdown();
```
### 使用submit提交任务

使用execute方法没有返回值，所以无法判断任务是否被线程池执行成功.
Java 1.5提供了Callable和Future，通过它们可以在任务执行完毕之后得到任务执行结果。

在`ExecutorService`接口中声明了若干个submit方法的重载：

* <T> Future<T> submit(Callable<T> task);
* <T> Future<T> submit(Runnable task, T result);
* Future<?> submit(Runnable task);

submit可以提交一个Callable对象，Callable是一个接口，里面只声明了一个方法call()。

```java
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```
Future就是对具体的Runnable或者Callable任务的执行结果进行取消，查询是否完成，还可以通过get方法获取执行结果。

```java
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
Future中声明了5个方法：

* cancel方法用来取消任务，如果取消任务成功则返回true，如果取消任务失败则返回false。参数mayInterruptIfRunning表示是否允许取消正在执行却没有执行完毕的任务，如果设置true，则表示可以取消正在执行过程中的任务。如果任务已经完成，则无论mayInterruptIfRunning为true还是false，此方法肯定返回false，即如果取消已经完成的任务会返回false；如果任务正在执行，若mayInterruptIfRunning设置为true，则返回true，若mayInterruptIfRunning设置为false，则返回false；如果任务还没有执行，则无论mayInterruptIfRunning为true还是false，肯定返回true。

* isCancelled方法表示任务是否被取消成功，如果在任务正常完成前被取消成功，则返回 true。

* isDone方法表示任务是否已经完成，若任务完成，则返回true；
* get()方法用来获取执行结果，这个方法会产生阻塞，会一直等到任务执行完毕才返回；
* get(long timeout, TimeUnit unit)用来获取执行结果，如果在指定时间内，还没获取到结果，就直接返回null。

```java
 public static void main(String[] args) {

        ExecutorService executor= Executors.newCachedThreadPool();
        Task task=new Task();

        Future<?> result=executor.submit(task);

        System.out.println("主线程在执行任务...");
        try {
            System.out.println("task运行结果"+result.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
        System.out.println("执行完毕...");
    }


class Task implements Callable<Integer>{

    @Override
    public Integer call() throws Exception {
        System.out.println("子线程在进行计算...");
        int sum=0;
        for (int i = 0; i < 1000; i++) {
            sum+=i;
        }
        return sum;
    }
}

```
执行结果

```
主线程在执行任务...
子线程在进行计算...
task运行结果499500
执行完毕...

```

#### FutureTask

```java
public class FutureTask<V> implements RunnableFuture<V>

public interface RunnableFuture<V> extends Runnable, Future<V>

```

通过查看源码，可以知道`FutureTask`实现了`Future`和`Runnable`接口。所以它既可以作为`Runnable`被线程执行，又可以作为`Future`得到`Callable`的返回值。

`FutureTask`提供了两个构造函数

```java
public FutureTask(Callable<V> callable) {
}
public FutureTask(Runnable runnable, V result) {
}
```

使用`Callable`和`FutureTask`来获取执行结果。

```java
public static void main(String[] args) {
    ExecutorService executor= Executors.newCachedThreadPool();
    Task task=new Task();
    FutureTask<Integer> futureTask=new FutureTask<Integer>(task);
    executor.submit(futureTask);

    System.out.println("主线程在执行任务...");
    try {
        System.out.println("task运行结果"+result.get());
    } catch (InterruptedException e) {
        e.printStackTrace();
    } catch (ExecutionException e) {
        e.printStackTrace();
    }
    System.out.println("执行完毕...");
}

```

执行结果

```
主线程在执行任务...
子线程在进行计算...
task运行结果499500
执行完毕...

```

### 创建线程池

我们可以通过`ThreadPoolExecutor`类来创建一个线程池。

```java
public ThreadPoolExecutor(int corePoolSize,
                            int maximumPoolSize,
                            long keepAliveTime,
                            TimeUnit unit,
                            BlockingQueue<Runnable> workQueue,
                            ThreadFactory threadFactory,
                            RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.acc = System.getSecurityManager() == null ?
            null :
            AccessController.getContext();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}

```
创建一个线程池需要输入几个参数：

* corePoolSize（线程池的基本大小）：当提交一个任务到线程池时，线程池会创建一个线程来执行任务，即使其他空闲的基本线程能够执行新任务也会创建线程，等到需要执行的任务数大于corePoolSize时就不再创建。如果调用了线程池的prestartAllCoreThreads方法，线程池会提前创建并启动所有基本线程。
* runnableTaskQueue（任务队列）：用于保存等待执行的任务的阻塞队列。 可以选择以下几个阻塞队列。
	* ArrayBlockingQueue：是一个基于数组结构的有界阻塞队列，此队列按 FIFO（先进先出）原则对元素进行排序。
	* LinkedBlockingQueue：一个基于链表结构的阻塞队列，此队列按FIFO （先进先出） 排序元素，吞吐量通常要高于ArrayBlockingQueue。静态工厂方法Executors.newFixedThreadPool()使用了这个队列。
	* SynchronousQueue：一个不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于LinkedBlockingQueue，静态工厂方法Executors.newCachedThreadPool使用了这个队列。
	* PriorityBlockingQueue：一个具有优先级的无限阻塞队列。
* maximumPoolSize（线程池最大大小）：线程池允许创建的最大线程数。如果队列满了，并且已创建的线程数小于最大线程数，则线程池会再创建新的线程执行任务。值得注意的是如果使用了无界的任务队列这个参数就没什么效果。
* ThreadFactory：用于设置创建线程的工厂，可以通过线程工厂给每个创建出来的线程设置更有意义的名字。
* RejectedExecutionHandler（饱和策略）：当队列和线程池都满了，说明线程池处于饱和状态，那么必须采取一种策略处理提交的新任务。这个策略默认情况下是AbortPolicy，表示无法处理新任务时抛出异常。
	以下是JDK1.5提供的四种策略:
	* AbortPolicy：直接抛出异常。
	* CallerRunsPolicy：只用调用者所在线程来运行任务。
	* DiscardOldestPolicy：丢弃队列里最近的一个任务，并执行当前任务。
	* DiscardPolicy：不处理，丢弃掉。
当然也可以根据应用场景需要来实现RejectedExecutionHandler接口自定义策略。如记录日志或持久化不能处理的任务。
* keepAliveTime（线程活动保持时间）：线程池的工作线程空闲后，保持存活的时间。所以如果任务很多，并且每个任务执行的时间比较短，可以调大这个时间，提高线程的利用率。
* TimeUnit（线程活动保持时间的单位）：可选的单位有天（DAYS），小时（HOURS），分钟（MINUTES），毫秒(MILLISECONDS)，微秒(MICROSECONDS, 千分之一毫秒)和毫微秒(NANOSECONDS, 千分之一微秒)。

### 参考
* [聊聊并发（三）——JAVA线程池的分析和使用](http://www.infoq.com/cn/articles/java-threadPool)
* [Java并发编程：线程池的使用
](http://www.cnblogs.com/dolphin0520/p/3932921.html)
* [浅谈Java两种并发类型——计算密集型与IO密集型](http://www.blogjava.net/bolo/archive/2015/01/20/422296.html)



