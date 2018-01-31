---
layout: post
title: Java多线程常识
date: 2017-12-09 23:34:25
tags:
categories: 软件开发
---

多线程编程是进行企业级开发时常见的技术。

# 线程安全

给线程安全下定义比较困难。存在很多种定义，如：“一个类在可以被多个线程安全调用时就是线程安全的”。 

## 静态变量
使用static关键字定义的变量。static可以修饰变量和方法，也有static静态代码块。被static修饰的成员变量和成员方法独立于该类的任何对象。也就是说，它不依赖类特定的实例，被类的所有实例共享。只要这个类被加载，Java虚拟机就能根据类名在运行时数据区的方法区内定找到他们。因此，static对象可以在它的任何对象创建之前访问，无需引用任何对象。静态变量通常用于对象间共享值、方便访问变量等场景。
静态变量即类变量，位于方法区，为所有对象共享，共享一份内存，一旦静态变量被修改，其他对象均对修改可见，静态变量是*非线程安全的*。

## 实例变量
单例模式（只有一个对象实例存在）线程非安全，非单例线程安全。 实例变量为对象实例私有，在虚拟机的堆中分配，若在系统中只存在一个此对象的实例，在多线程环境下，“犹如”静态变量那样，被某个线程修改后，其他线程对修改均可见，故线程非安全；如果每个线程执行都是在不同的对象中，那对象与对象之间的实例变量的修改将互不影响，故线程安全。

## 局部变量
线程安全。每个线程执行时将会把局部变量放在各自栈帧的工作内存中，线程间不共享，故不存在线程安全问题。

# 线程安全的单例模式

单例模式是一种常用的软件设计模式。在它的核心结构中只包含一个被称为单例类的特殊类。通过单例模式可以保证系统中一个类只有一个实例而且该实例易于外界访问，从而方便对实例个数的控制并节约系统资源。如果希望在系统中某个类的对象只能存在一个，单例模式是最好的解决方案。

## Double Check

在 Effecitve Java 一书的第 48 条中提到了双重检查模式，并指出这种模式在 Java 中通常并不适用。该模式的结构如下所示：

```java
public Resource getResource() {
  if (resource == null) {
    synchronized(this){
      if (resource==null) {
        resource = new Resource(); 
      }
    }
  }
  return resource;
}
```
该模式是对下面的代码改进：
```java
public synchronized Resource getResource(){
  if (resource == null){ 
        resource = new Resource();  
  }
  return resource;
}
```
这段代码的目的是对 resource 延迟初始化。但是每次访问的时候都需要同步。为了减少同步的开销，于是有了双重检查模式。在 Java 中双重检查模式无效的原因是在不同步的情况下引用类型不是线程安全的。对于除了 long 和 double 的基本类型，双重检查模式是适用 的。比如下面这段代码就是正确的：

```java
private int count;
public int getCount(){
  if (count == 0){
    synchronized(this){
      if (count == 0){
        count = computeCount();  //一个耗时的计算
      }
    }
  }
  return count;
}
```

上面就是关于java中双重检查模式（double-check idiom）的一般结论。但是事情还没有结束，因为java的内存模式也在改进中。Doug Lea 在他的文章中写道：“根据最新的 JSR133 的 Java 内存模型，如果将引用类型声明为 volatile，双重检查模式就可以工作了”，参见 `http://gee.cs.oswego.edu/dl/cpj/updates.html` 。所以以后要在 Java 中使用双重检查模式，可以使用下面的代码：

```java
private volatile Resource resource;
public Resource getResource(){
  if (resource == null){
    synchronized(this){
      if (resource==null){
        resource = new Resource();
      }
    }
  }
  return resource;
}
```

当然了，得是在遵循 JSR133 规范的 Java 中。所以，double-check 在 J2SE 1.4 或早期版本在多线程或者 JVM 调优时由于 out-of-order writes，是不可用的。 这个问题在 J2SE 5.0 中已经被修复，可以使用 volatile 关键字来保证多线程下的单例。

```java
public class Singleton {
    private volatile Singleton instance = null;
    public Singleton getInstance() {
        if (instance == null) {
            synchronized(this) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

## IODH

推荐方法 是Initialization on Demand Holder（IODH），详见 `http://en.wikipedia.org/wiki/Initialization_on_demand_holder_idiom`

```java
public class Singleton {
    static class SingletonHolder {
        static Singleton instance = new Singleton();
    }

    public static Singleton getInstance(){
        return SingletonHolder.instance;
    }
}
```

编译并运行上述代码，运行结果为：true，即创建的单例对象s1和s2为同一对象。由于静态单例对象没有作为Singleton的成员变量直接实例化，因此类加载时不会实例化Singleton，第一次调用getInstance()时将加载内部类SingletonHolder，在该内部类中定义了一个static类型的变量instance，此时会首先初始化这个成员变量，由Java虚拟机来保证其线程安全性，确保该成员变量只能初始化一次。由于getInstance()方法没有任何线程锁定，因此其性能不会造成任何影响。通过使用IoDH，我们既可以实现延迟加载，又可以保证线程安全，不影响系统性能，不失为一种最好的Java语言单例模式实现方式（其缺点是与编程语言本身的特性相关，很多面向对象语言不支持IoDH）。

# 线程池

Java通过Executors提供四种线程池，分别为：

1. newCachedThreadPool创建一个可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程，若无可回收，则新建线程。
2. newFixedThreadPool 创建一个定长线程池，可控制线程最大并发数，超出的线程会在队列中等待。
3. newScheduledThreadPool 创建一个定长线程池，支持定时及周期性任务执行。
4. new SingleThreadExecutor 创建一个单线程化的线程池，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行。

## ExecutorService

ExecutorServcie中执行一个Runnable有两个方法，两个分别是：

```java
public void execute(Runnable command);
public <T> Future<T> submit(Runnable task, T result);
```

其实submit最后是调用的execute的，而且在调用execute前，对task进行了一次封装，变成了RunnableFuture。

### execute

### submit

### newCachedThreadPool

创建一个可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程，若无可回收，则新建线程。示例代码如下：

```java
package test;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
public class ThreadPoolExecutorTest {
 public static void main(String[] args) {
  ExecutorService cachedThreadPool = Executors.newCachedThreadPool();  
  for (int i = 0; i < 10; i++) {
   final int index = i;
   try {
    Thread.sleep(index * 1000);
   } catch (InterruptedException e) {
    e.printStackTrace();
   }
   cachedThreadPool.execute(new Runnable() {
    public void run() {
     System.out.println(index);
    }
   });
  }
 }
}
```

线程池为无限大，当执行第二个任务时第一个任务已经完成，会复用执行第一个任务的线程，而不用每次新建线程。

### newFixedThreadPool

创建一个定长线程池，可控制线程最大并发数，超出的线程会在队列中等待。示例代码如下：

```java
package test;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
public class ThreadPoolExecutorTest {
 public static void main(String[] args) {
  ExecutorService fixedThreadPool = Executors.newFixedThreadPool(3);
  for (int i = 0; i < 10; i++) {
   final int index = i;
   fixedThreadPool.execute(new Runnable() {
    public void run() {
     try {
      System.out.println(index);
      Thread.sleep(2000);
     } catch (InterruptedException e) {
      e.printStackTrace();
     }
    }
   });
  }
 }
}
```

因为线程池大小为3，每个任务输出index后sleep 2秒，所以每两秒打印3个数字。
定长线程池的大小最好根据系统资源进行设置。如Runtime.getRuntime().availableProcessors()

### newScheduledThreadPool

创建一个定长线程池，支持定时及周期性任务执行。延迟执行示例代码如下：

```java
package test;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;
public class ThreadPoolExecutorTest {
 public static void main(String[] args) {
  ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(5);
  scheduledThreadPool.schedule(new Runnable() {
   public void run() {
    System.out.println("delay 3 seconds");
   }
  }, 3, TimeUnit.SECONDS);
 }
}
```

表示延迟3秒执行。定期执行示例代码如下：

```java
package test;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;
public class ThreadPoolExecutorTest {
 public static void main(String[] args) {
  ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(5);
  scheduledThreadPool.scheduleAtFixedRate(new Runnable() {
   public void run() {
    System.out.println("delay 1 seconds, and excute every 3 seconds");
   }
  }, 1, 3, TimeUnit.SECONDS);
 }
}
```

表示延迟1秒后每3秒执行一次。

### newSingleThreadExecutor

创建一个单线程化的线程池，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行。示例代码如下：

```java
package test;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
public class ThreadPoolExecutorTest {
 public static void main(String[] args) {
  ExecutorService singleThreadExecutor = Executors.newSingleThreadExecutor();
  for (int i = 0; i < 10; i++) {
   final int index = i;
   singleThreadExecutor.execute(new Runnable() {
    public void run() {
     try {
      System.out.println(index);
      Thread.sleep(2000);
     } catch (InterruptedException e) {
      e.printStackTrace();
     }
    }
   });
  }
 }
}
```

结果依次输出，相当于顺序执行各个任务。你可以使用JDK自带的监控工具来监控我们创建的线程数量，运行一个不终止的线程，创建指定量的线程，来观察：工具目录：`C:\Program Files\Java\jdk1.6.0_06\bin\jconsole.exe` 运行程序做稍微修改：

```java
package test;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
public class ThreadPoolExecutorTest {
 public static void main(String[] args) {
  ExecutorService singleThreadExecutor = Executors.newCachedThreadPool();
  for (int i = 0; i < 100; i++) {
   final int index = i;
   singleThreadExecutor.execute(new Runnable() {
    public void run() {
     try {
      while(true) {
       System.out.println(index);
       Thread.sleep(10 * 1000);
      }
     } catch (InterruptedException e) {
      e.printStackTrace();
     }
    }
   });
   try {
    Thread.sleep(500);
   } catch (InterruptedException e) {
    e.printStackTrace();
   }
  }
 }
}
```

# Future

多线程开发中有几个痛点：

1. 主线程如何正确的关闭异步线程？
2. 主线程怎么知道异步线程是否执行完成？

当然已经又很好的方案来解决这些问题了，这就是Future模式。Future模式在请求发生时，会先产生一个Future对象给发出请求的客户，而真正的结果是由一个新线程执行，结果生成之后，将之设定至Future之中，而当客户端需要结果时，Future也已经准备好，可以让客户提取使用。

Future模式在请求发生时，会先产生一个Future对象给发出请求的客户，而真正的结果是由一个新线程执行，结果生成之后，将之设定至Future之中，而当客户端需要结果时，Future也已经准备好，可以让客户提取使用。

{% asset_img future1.jpg %}

一个简单的例子：

```java
public Future request() {
    final Future future = new Future();
    new Thread() {
        public void run() {
            // 下面这动作可能是耗时的
            RealSubject subject = new RealSubject();
            future.setRealSubject(subject);
        }
    }.start();
    return future;
}
```

DK里面也有对Future模式的实现，我们先来看看 Future 接口包含什么内容:

```java
package java.util.concurrent;
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

支持的功能：

1. 关闭线程
2. 判断是否关闭
3. 判断是否完成
4. 获取结果

## 线程中Future模式

```java
FutureTask<Integer> future = new FutureTask<Integer>(new Callable<Integer>() {
    public Integer call() throws Exception {
        return new Random().nextInt(100);
    }
});
new Thread(future).start();
Thread.sleep(5000); // do something
System.out.println(future.get());
```

这里的FutureTask的继承关系:

```java
public class FutureTask<V> implements RunnableFuture<V> {}
public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}
```

由于继承了Runnable和Future两个接口，所以可以被在线程中运行，同样具有Future功能，需要说明的是FutureTask在JDK中是唯一一个实现Future的类。

## 线程池中 Future 模式

```java
ExecutorService threadPool = Executors.newSingleThreadExecutor();
Future<Integer> future = threadPool.submit(new Callable<Integer>() {
    public Integer call() throws Exception {
        return new Random().nextInt(100);  
    }
});
Thread.sleep(5000); // do something
System.out.println(future.get());
```

通过submit(Callable)可以得到一个Future，之前我们说过FutureTask是JDK中是唯一的Future的实体类，所以submit返回的Future一定是一个FutureTask。其内部其实是通过newTaskFor方法来转换的。

```java
protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
    return new FutureTask<T>(callable);
}
```

## ThreadPool框架

这里我们来封装一下JDK中Future模式，使其更好用。现线程完成或关闭后的通知回调。

```java
List<ThreadTask<String>> results = new ArrayList<ThreadTask<String>>();
ThreadPool threadPool = new ThreadPool(4, 4, 10, TimeUnit.SECONDS, new LinkedBlockingQueue<Runnable>());
for (int i = 1; i < 5; i++) {
    ThreadTask<String> threadTask = threadPool.submit(new Task(i));
    threadTask.setOnTaskDoneListener(new OnTaskDoneListener() {
        @Override
        public void onTaskDone(ThreadTask<?> task) {
            System.out.println("onTaskDone " + task.get());
        }
    });
    threadTask.setOnTaskCancelListener(new OnTaskCancelListener() {
        @Override
        public void onTaskCancel(ThreadTask<?> task) {
            System.out.println(task + " onTaskCancel");
        }
    });
    results.add(threadTask);
}
for (ThreadTask<String> res : results) {
    res.cancel();
    System.out.println("result :" + res.get());
}
threadPool.shutdown();
```
