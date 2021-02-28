---
title: "【译】避免Android中内存泄漏的9种方法"
date: 2019-06-10T14:33:57+08:00
tags: ["Android"]
draft: false
toc: true
---


[原文](https://android.jlelse.eu/9-ways-to-avoid-memory-leaks-in-android-b6d81648e35e)，本文介绍了，引发内存泄露的常见情况，并给出了对应的修复方法。



> I have been an android developer for quite some time now. And I **realised（意识到）** that most of that time, I tend to spend on adding new features to an app or working on visually enhancements of the app, rather than focusing on the core issues like performance and quality.

我做安卓开发者已经有一段时间了。我意识到，大部分的时间，我倾向于花在为应用添加新功能，或者在应用的视觉增强上，而不是关注性能和质量等核心问题。


> This was me a while back when I would be asked to optimise or refactor code

 这是我前段时间被要求优化或重构代码时的情景

 ![](/images/9-ways-to-avoid-memory-leaks-in-android-1.gif)

 > But these things have a way of catching up to you and I spent a long time wishing I had not ignored the important stuff. So in this article, I thought I would focus on one of the most important optimisation techniques in android: Memory leaks.

 但是这些东西有办法让你追上，我花了很长时间希望自己没有忽略重要的东西。所以在这篇文章中，我想我会关注`android`中最重要的优化技术之一：内存泄漏。

 > There are a lot of articles on memory leaks and how to fix them. But when I was learning myself, I found that none of them were in one place and it was hard to keep track of it all. So I thought I would collectively post an article on it so that it might help people in the future.

 关于内存泄漏以及如何解决内存泄漏的文章很多。但是我自己在学习的时候，发现这些文章都不在一个地方，很难全部记录下来。所以我想，我就集体发一篇文章，这样可能会对以后的人有所帮助。

 > So let’s begin.

那么我们开始吧。


## 什么是内存泄露？ （What are memory leaks?）

> A memory leak happens when memory is allocated but never freed. This means the garbage collector is not able to take out the trash once we are done with the takeout. Initially, this might not be a problem. But imagine if you don’t take the trash out for 2 weeks! The house starts to smell right?


当内存被分配但从未被释放时，就会发生内存泄漏。这意味着一旦我们吃完了外卖，垃圾回收器就无法回收垃圾。最初，这可能不是一个问题。但试想一下，如果你2个星期都不倒垃圾的话! 家里就开始有味道了吧?

> Similarly, as the user keeps on using our app, the memory also keeps on increasing and if the memory leaks are not plugged, then the unused memory cannot be freed up by the Garbage Collection. So the memory of our app will constantly increase until no more memory can be allocated to our app, leading to OutOfMemoryError which ultimately crashes the app.

同样，随着用户不断的使用我们的app，内存也会不断的增加，如果不堵住内存泄漏的地方，那么未使用的内存就无法被垃圾回收释放出来。所以我们的应用的内存会不断的增加，直到不能再给我们的应用分配更多的内存，导致`OutOfMemoryError`，最终导致应用崩溃。

## 那么，如何检查我的应用是否泄露？(So how do I check if my app is leaking?)

> I am sure everyone on the planet is aware of it but just in case, there is an awesome library called LeakyCanary that is great for finding out the leaks in our app along with the stack trace.

我相信地球上的每个人都知道它，但为了以防万一，有一个很棒的库叫[LeakyCanary](https://github.com/square/leakcanary)，它可以很好地找出我们应用程序中的泄漏以及堆栈跟踪。

## 那么，导致内存泄露的常见错误有哪些呢？(So what are some of the common mistakes that lead to memory leaks?)

### 1.广播接受者（Broadcast Receivers:）

> Consider this scenario - you need to register a local broadcast receiver in your activity. If you don’t unregister the broadcast receiver, then it still holds a reference to the activity, even if you close the activity.

考虑这种情况--你需要在你的活动中注册一个本地广播接收器。如果你不取消注册广播接收器，那么即使你关闭了`activity`，它仍然持有对`activity`的引用。

```java
public class BroadcastReceiverLeakActivity extends AppCompatActivity {

    private BroadcastReceiver broadcastReceiver;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);
    }

    private void registerBroadCastReceiver() {
        broadcastReceiver = new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                //your receiver code goes here!
            }
        };
        registerReceiver(broadcastReceiver, new IntentFilter("SmsMessage.intent.MAIN"));
    }
    
    
    @Override
    protected void onStart() {
        super.onStart();
        registerBroadCastReceiver();
    }    


    @Override
    protected void onStop() {
        super.onStop();

        /*
         * Uncomment this line in order to avoid memory leak.
         * You need to unregister the broadcast receiver since the broadcast receiver keeps a reference of the activity.
         * Now when its time for your Activity to die, the Android framework will call onDestroy() on it
         * but the garbage collector will not be able to remove the instance from memory because the broadcastReceiver
         * is still holding a strong reference to it.
         * */

        if(broadcastReceiver != null) {
            unregisterReceiver(broadcastReceiver);
        }
    }
}
```

> How to solve this? Always remember to call unregister receiver in onStop() of the activity.

如何解决这个问题？一定要记得在活动的`onStop()`中调用`unregisterReceiver`。

> Note: As Artem Demyanov pointed out, if the broadcast Receiver is registered in onCreate(), then when the app goes into the background and resumed again, the receiver will not be registered again. So it is always good to register the broadcastReceiver in onStart() or onResume() of the activity and unregister in onStop().

注意：正如Artem Demyanov所指出的，如果广播接收器是在`onCreate()`中注册的，那么当应用程序进入后台并再次恢复时，接收器将不会被再次注册。所以最好在`activity`的`onStart()`或`onResume()`中注册广播接收器，并在`onStop()`中取消注册。

### 2.静态Activity或者静态View引用（2. Static Activity or View Reference:）

> Consider the below example — You are declaring a TextView as static (for whatever reason). If you reference an activity or view directly or indirectly from a static reference, the activity would not be garbage collected after it is destroyed.

考虑下面的例子--你正在将一个`TextView`声明为静态（无论出于什么原因）。如果你直接或间接地从静态引用中引用一个`activity`或`view`，该`activity`在被销毁后不会被垃圾回收。

```java
public class StaticReferenceLeakActivity extends AppCompatActivity {

    /*  
     * This is a bad idea! 
     */
    private static TextView textView;
    private static Activity activity;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);
        
        
        textView = findViewById(R.id.activity_text);
        textView.setText("Bad Idea!");
           
        activity = this;
    }
}
```
> How to solve this? Always remember to NEVER use static variables for views or activities or contexts.

如何解决这个问题？永远记住，千万不要使用`view`或`activity`或`context`静态变量。

### 3.单例类引用（3. Singleton Class Reference:）

> Consider the below example — you have defined a Singleton class as displayed below and you need to pass the context in order to fetch some files from the local storage.

考虑下面的例子--你已经定义了一个`Singleton`类，如下图所示，你需要传递上下文，以便从本地存储中获取一些文件。

```java
public class SingletonLeakExampleActivity extends AppCompatActivity {

    private SingletonSampleClass singletonSampleClass;
    
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        
    /* 
     * Option 1: Do not pass activity context to the Singleton class. Instead pass application Context
     */      
        singletonSampleClass = SingletonSampleClass.getInstance(this);
    }
  
    
   @Override
   protected void onDestroy() {
        super.onDestroy();

    /* 
     * Option 2: Unregister the singleton class here i.e. if you pass activity context to the Singleton class, 
     * then ensure that when the activity is destroyed, the context in the singleton class is set to null.
     */
     singletonSampleClass.onDestroy();
   }
}
```

`SingletonSampleClass.java`

```java
public class SingletonSampleClass {
  
    private Context context;
    private static SingletonSampleClass instance;
  
    private SingletonSampleClass(Context context) {
        this.context = context;
    }

    public synchronized static SingletonSampleClass getInstance(Context context) {
        if (instance == null) instance = new SingletonSampleClass(context);
        return instance;
    }
  
    public void onDestroy() {
       if(context != null) {
          context = null; 
       }
    }
}
```

> How to solve this?
Option 1: Instead of passing activity context i.e. this to the singleton class, you can pass applicationContext().
Option 2: If you really have to use activity context, then when the activity is destroyed, ensure that the context you passed to the singleton class is set to null.

如何解决这个问题？

选项一：

如何解决这个问题？
* 方案1：可以不传递活动上下文即这个给单子类，而是传递`applicationContext()`。
* 方案2：如果你真的要使用`activity`上下文，那么当`activity`被销毁时，确保你传递给单例类的上下文被设置为`null`。

### 4.内部类引用（4. Inner Class Reference:）

> Consider the below example — you have defined a inner class called LeakyClass.java as displayed below and you need to pass the activity in order to redirect to a new activity.

考虑下面的例子--你定义了一个名为`LeakyClass.java`的内部类，如下所示，你需要传递`activity`来跳转到一个新的`activity`。

```java
public class InnerClassReferenceLeakActivity extends AppCompatActivity {

  /* 
   * Mistake Number 1: 
   * Never create a static variable of an inner class
   * Fix I:
   * private LeakyClass leakyClass;
   */
  private static LeakyClass leakyClass;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);
        
        new LeakyClass(this).redirectToSecondScreen();

        /*
         * Inner class is defined here
         * */
         leakyClass = new LeakyClass(this);
         leakyClass.redirectToSecondScreen();
    }
    
  /* 
   * Mistake Number 2: 
   * 1. Never create a inner variable of an inner class
   * 2. Never pass an instance of the activity to the inner class
   */       
    private class LeakyClass {
        
        private Activity activity;
        public LeakyClass(Activity activity) {
            this.activity = activity;
        }
        
        public void redirectToSecondScreen() {
            this.activity.startActivity(new Intent(activity, SecondActivity.class));
        }
    }
}
```

> How to solve this?
Option 1: As mentioned before never create a static variable of an inner class.
Option 2: The class should be set to static. Instances of anonymous classes do not hold an implicit reference to their outer class when they are “static”.
Option 3: Use a weakReference of any view/activity. Garbage collector can collect an object if only weak references are pointing towards it.


如何解决这个问题？

* 方案1：如前所述，千万不要创建一个内类的静态变量。
* 方案2：应该将类设置为静态。匿名类的实例在 "静态 "时不会对其外类持有隐式引用。
* 选项3：使用任何视图/活动的弱引用。如果只有弱引用指向一个对象，垃圾收集器可以收集这个对象。


```java
public class InnerClassReferenceLeakActivity extends AppCompatActivity {

  /* 
   * Mistake Number 1: 
   * Never create a static variable of an inner class
   * Fix I:
   */
  private LeakyClass leakyClass;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);
        
        new LeakyClass(this).redirectToSecondScreen();

        /*
         * Inner class is defined here
         * */
         leakyClass = new LeakyClass(this);
         leakyClass.redirectToSecondScreen();
    }
  
  
    /*  
     * How to fix the above class:
     * Fix memory leaks:
     * Option 1: The class should be set to static
     * Explanation: Instances of anonymous classes do not hold an implicit reference to their outer class 
     * when they are "static".
     *
     * Option 2: Use a weakReference of the textview or any view/activity for that matter
     * Explanation: Weak References: Garbage collector can collect an object if only weak references 
     * are pointing towards it.
     * */
    private static class LeakyClass {
        
        private final WeakReference<Activity> messageViewReference;
        public LeakyClass(Activity activity) {
            this.activity = new WeakReference<>(activity);
        }
        
        public void redirectToSecondScreen() {
            Activity activity = messageViewReference.get();
            if(activity != null) {
               activity.startActivity(new Intent(activity, SecondActivity.class));
            }
        }
    }  
}
```

### 5.匿名内部类引用（5. Anonymous Class Reference:）

> This follows the same theory as above. Sample implementation for fixing memory leak is given below:

这与上面的理论相同。下面给出了修复内存泄漏的实现示例。

```java
public class AnonymousClassReferenceLeakActivity extends AppCompatActivity {

    private TextView textView;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);


        textView = findViewById(R.id.activity_text);
        textView.setText(getString(R.string.text_inner_class_1));
        findViewById(R.id.activity_dialog_btn).setVisibility(View.INVISIBLE);

        /*
         * Runnable class is defined here
         * */
         new Thread(new LeakyRunnable(textView)).start();
    }



    private static class LeakyRunnable implements Runnable {

        private final WeakReference<TextView> messageViewReference;
        private LeakyRunnable(TextView textView) {
            this.messageViewReference = new WeakReference<>(textView);
        }

        @Override
        public void run() {
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            TextView textView = messageViewReference.get();
            if(textView != null) {
                textView.setText("Runnable class has completed its work");
            }
        }
    }
}
```

### 6.AsyncTask引用（6. AsyncTask Reference:）

> Consider the following example — You are using an asyncTask to get a string value which is used to update the textView in OnPostExecute().

考虑下面的例子--你正在使用`AsyncTask`来获取一个字符串值，这个字符串值在`OnPostExecute()`中用来更新`textView`。

```java
public class AsyncTaskReferenceLeakActivity extends AppCompatActivity {

    private TextView textView;
    private BackgroundTask backgroundTask;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);

        /*
         * Executing AsyncTask here!
         * */
        backgroundTask = new BackgroundTask(textView);
        backgroundTask.execute();
    }

    /*
     * Couple of things we should NEVER do here:
     * Mistake number 1. NEVER reference a class inside the activity. If we definitely need to, we should set the class as static as static inner classes don’t hold
     *    any implicit reference to its parent activity class
     * Mistake number 2. We should always cancel the asyncTask when activity is destroyed. This is because the asyncTask will still be executing even if the activity
     *    is destroyed.
     * Mistake number 3. Never use a direct reference of a view from acitivty inside an asynctask.
     * */
 private class BackgroundTask extends AsyncTask<Void, Void, String> {    
        @Override
        protected String doInBackground(Void... voids) {

            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "The task is completed!";
        }

        @Override
        protected void onPostExecute(String s) {
            super.onPostExecute(s);
            textView.setText(s);
        }
    }
}
```

> How to solve this?
Option 1: We should always cancel the asyncTask when activity is destroyed. This is because the asyncTask will still be executing even if the activity is destroyed.
Option 2: NEVER reference a class inside the activity. If we definitely need to, we should set the class as static as static inner classes don’t hold any implicit reference to its parent activity class.
Option 3: Use a weakReference of the textview.


如何解决这个问题？
* 方案1：当activity被销毁时，我们应该总是取消asyncTask。因为即使activity被销毁，asyncTask仍然会执行。
* 方案2：永远不要在activity内部引用类。如果我们一定要这样做，我们应该把类设置为静态的，因为静态的内部类不会对它的父`activity`类持有任何隐式引用。
* 方案3：使用textview的弱引用。

```java
public class AsyncTaskReferenceLeakActivity extends AppCompatActivity {

    private TextView textView;
    private BackgroundTask backgroundTask;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);

        /*
         * Executing AsyncTask here!
         * */
        backgroundTask = new BackgroundTask(textView);
        backgroundTask.execute();
    }


    /*
     * Fix number 1
     * */
    private static class BackgroundTask extends AsyncTask<Void, Void, String> {

        private final WeakReference<TextView> messageViewReference;
        private BackgroundTask(TextView textView) {
            this.messageViewReference = new WeakReference<>(textView);
        }


        @Override
        protected String doInBackground(Void... voids) {

            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "The task is completed!";
        }

        @Override
        protected void onPostExecute(String s) {
            super.onPostExecute(s);
          /*
           * Fix number 3
           * */          
            TextView textView = messageViewReference.get();
            if(textView != null) {
                textView.setText(s);
            }
        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();

        /*
         * Fix number 2
         * */
        if(backgroundTask != null) {
            backgroundTask.cancel(true);
        }
    }
}
```

### 7.Handler引用（7. Handler Reference:）

> Consider the following example — You are using a Handler to redirect to a new screen after 5 seconds.

考虑以下例子--你正在使用一个处理程序在5秒后跳转到一个新的界面。

```java
public class HandlersReferenceLeakActivity extends AppCompatActivity {

    private TextView textView;

    /*
     * Mistake Number 1
     * */
     private Handler leakyHandler = new Handler();


    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);

        /*
         * Mistake Number 2
         * */
        leakyHandler.postDelayed(new Runnable() {
            @Override
            public void run() {
                textView.setText(getString(R.string.text_handler_1));
            }
        }, 5000);
    }
}
```

> How to solve this?
Option 1: NEVER reference a class inside the activity. If we definitely need to, we should set the class as static. This is because when a Handler is instantiated on the main thread, it is associated with the Looper’s message queue. Messages posted to the message queue will hold a reference to the Handler so that the framework can call Handler#handleMessage(Message) when the Looper eventually processes the message.
Option 2: Use a weakReference of the Activity.

如何解决这个问题？
* 方案一：永远不要在活动里面引用类。如果一定要引用，我们应该将类设置为静态。这是因为当在主线程上实例化一个Handler时，它会与Looper的消息队列相关联。发布到消息队列中的消息将持有对Handler的引用，这样当Looper最终处理消息时，框架可以调用Handler#handleMessage(Message)。
* 方案2：使用Activity的弱引用。

```java
public class HandlersReferenceLeakActivity extends AppCompatActivity {

    private TextView textView;

    /*
     * Fix number I
     * */
    private final LeakyHandler leakyHandler = new LeakyHandler(this);

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);

        leakyHandler.postDelayed(leakyRunnable, 5000);
    }

    /*
     * Fix number II - define as static
     * */
    private static class LeakyHandler extends Handler {
      
    /*
     * Fix number III - Use WeakReferences
     * */      
        private WeakReference<HandlersReferenceLeakActivity> weakReference;
        public LeakyHandler(HandlersReferenceLeakActivity activity) {
            weakReference = new WeakReference<>(activity);
        }

        @Override
        public void handleMessage(Message msg) {
            HandlersReferenceLeakActivity activity = weakReference.get();
            if (activity != null) {
                activity.textView.setText(activity.getString(R.string.text_handler_2));
            }
        }
    }

    private static final Runnable leakyRunnable = new Runnable() {
        @Override
        public void run() { /* ... */ }
    }
}
```

### 8.线程引用（8. Threads Reference:）

> We can repeat this same mistake again with both the Thread and TimerTask classes.

我们可以在Thread和TimerTask两个类中再次重复这个错误。

```java
public class ThreadReferenceLeakActivity extends AppCompatActivity {

    /*  
     * Mistake Number 1: Do not use static variables
     * */    
    private static LeakyThread thread;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);

        createThread();
        redirectToNewScreen();
    }


    private void createThread() {
        thread = new LeakyThread();
        thread.start();
    }

    private void redirectToNewScreen() {
        startActivity(new Intent(this, SecondActivity.class));
    }


    /*
     * Mistake Number 2: Non-static anonymous classes hold an 
     * implicit reference to their enclosing class.
     * */
    private class LeakyThread extends Thread {
        @Override
        public void run() {
            while (true) {
            }
        }
    }
}
```

> How to solve this?
Option 1: Non-static anonymous classes hold an implicit reference to their enclosing class.
Option 2: Close thread in activity onDestroy() to avoid thread leak.

如何解决这个问题？
* 方案1：非静态匿名类对其包围类持有一个隐式引用。
* 方案2：在onDestroy()活动中关闭线程，避免线程泄漏。

```java
public class ThreadReferenceLeakActivity extends AppCompatActivity {

    /*
     * FIX I: make variable non static
     * */
    private LeakyThread leakyThread = new LeakyThread();

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);

        createThread();
        redirectToNewScreen();
    }


    private void createThread() {
        leakyThread.start();
    }

    private void redirectToNewScreen() {
        startActivity(new Intent(this, SecondActivity.class));
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        // FIX II: kill the thread
        leakyThread.interrupt();
    }


    /*
     * Fix III: Make thread static
     * */
    private static class LeakyThread extends Thread {
        @Override
        public void run() {
            while (!isInterrupted()) {
            }
        }
    }
}
```

## 9.TimeTask引用（9. TimerTask Reference:）

> The same principle is as Threads can be followed for TimerTask as well. Sample implementation for fixing memory leak is given below:

TimerTask也可以遵循与Threads相同的原理。下面给出了修复内存泄漏的实现示例。

```java
public class TimerTaskReferenceLeakActivity extends Activity {

    private CountDownTimer countDownTimer;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);

        startTimer();
    }

    /*  
     * Mistake 1: Cancel Timer is never called 
     * even though activity might be completed
     * */
    public void cancelTimer() {
        if(countDownTimer != null) countDownTimer.cancel();
    }

    
    private void startTimer() {
        countDownTimer = new CountDownTimer(1000, 1000) {
            @Override
            public void onTick(long millisUntilFinished) {
                final int secondsRemaining = (int) (millisUntilFinished / 1000);
                //update UI
            }

            @Override
            public void onFinish() {
                //handle onFinish
            }
        };
        countDownTimer.start();
    }
}
```
> How to solve this?
Option 1: Cancel timer in activity onDestroy() to avoid memory leak.

如何解决这个问题？
方案1：在`activity`的`onDestroy()`中取消定时器，避免内存泄漏。

```java
public class TimerTaskReferenceLeakActivity extends Activity {

    private CountDownTimer countDownTimer;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);

        startTimer();
    }


    public void cancelTimer() {
        if(countDownTimer != null) countDownTimer.cancel();
    }


    private void startTimer() {
        countDownTimer = new CountDownTimer(1000, 1000) {
            @Override
            public void onTick(long millisUntilFinished) {
                final int secondsRemaining = (int) (millisUntilFinished / 1000);
                //update UI
            }

            @Override
            public void onFinish() {
                //handle onFinish
            }
        };
        countDownTimer.start();
    }
  
  
    /*
     * Fix 1: Cancel Timer when 
     * activity might be completed
     * */  
   @Override
    protected void onDestroy() {
        super.onDestroy();
        cancelTimer();
    }
}
```

> So basically to summarise:
1. Use applicationContext() instead of activity context when possible. If you really have to use activity context, then when the activity is destroyed, ensure that the context you passed to the class is set to null.
2. Never use static variables to declare views or activity context.
3. Never reference a class inside the activity. If we need to, we should declare it as static, whether it is a thread or a handler or a timer or an asyncTask.
4. Always make sure to unregister broadcastReceivers or timers inside the activity. Cancel any asyncTasks or Threads inside onDestroy().
5. Always use a weakReference of the activity or view when needed.


所以基本上要总结一下。

1. 尽可能使用applicationContext()来代替活动上下文。如果你真的要使用活动上下文，那么当活动被销毁时，确保你传递给类的上下文被设置为null。
2. 千万不要使用静态变量来声明view或activity上下文。
3. 千万不要在活动内部引用类。如果需要，我们应该将其声明为静态，不管它是线程还是处理程序，还是定时器或asyncTask。
4. 一定要确保在activity里面取消注册broadcastReceivers或定时器。在onDestroy()里面取消任何asyncTask或Threads。
5. 在需要的时候，一定要使用活动或视图的弱引用。


> And that’s it folks! Thanks for reading! I hope you enjoyed this article and found it useful, if so please hit the Clap button. Let me know your thoughts in the comments section.
Happy coding!

就这样了，朋友们! 谢谢大家的阅读! 我希望你喜欢这篇文章，并发现它是有用的，如果是这样，请点击拍手按钮。在评论区告诉我你的想法。
祝你编码愉快。


## 更多资料

* [DO NOT LEAK VIEWS (Android Performance Patterns Season 3 ep6)](https://www.youtube.com/watch?v=BkbHeFHn8JY)