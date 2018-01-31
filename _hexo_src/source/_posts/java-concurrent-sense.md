---
layout: post
title: Java并发编程通识
date: 2018-01-15 15:45:05
tags:
categories: 软件开发
---
并发(Concurrency)在我们的现实世界中随处可见，以至于我们常常忽略了它的存在，比方说你在工作（假设你是一名程序员，你的工作就是编程）的时候也可以听听自己喜欢的音乐，并且你的耳朵并不会因为手头的工作而忽略了声音的存在（当然，除非你自己有意的去忽略它，但你还是能够听得见声音，只是你的大脑可能不会去感受音乐的节奏），此时你的大脑既要控制你的双手敲击键盘，也要控制你的耳朵去感受音乐。因此，在一定程度上，你的大脑就在并发地处理不同的事情，并且每个时刻都可能会侧重处理某件事情，比如某个时刻音乐达到高潮并且是你喜欢的旋律，你可能会放慢或者停止手边的工作，但在另外一个时刻你正在编写关键代码，需要全神贯注来避免 Bug 的出现，你可能会把声音调小一点或者干脆摘掉耳机。所以，我们的大脑就在并发地指导我们完成各种任务，或者换一种说法，我们需要处理的任务并发地征用我们的大脑，大脑就相当于计算机的 CPU，而待处理的任务就相当于计算机程序（更确切地说应该是进程或线程等执行实体）。

不过在现实世界中，我们并不会严格定义什么是并发。而在计算机程序世界中，为了编写高性能的代码，我们应该理解什么是并发，并发的基本特性是什么，哪些问题可以使用并发编程来（高效地）解决，哪些情况下又应该尽量避免使用并发编程，我们在使用并发编程时需要注意一些什么问题。

# 基本概念

## 并发与并行的联系和区别

与并发相近的另一个概念是并行(Parallel)。和并发所描述的情况一样，并行也是指两个或多个任务被同时执行。但是严格来讲，并发和并行的概念并是不等同的，两者存在很大的差别。下面我们来看看计算机科学家们是怎么区分并发和并行的。

**Erlang的发明者Joe Armstrong**在他的一篇博文中提到如何向一个 5 岁的小孩去介绍并发和并行的区别，并给出了下面一幅图：

{% asset_img concurrent.png 并发与并行 %}

直观来讲，并发是两个等待队列中的人同时去竞争一台咖啡机（当然，人是有理性懂礼貌的动物（也不排除某些很霸道的人插队的可能），两队列中的排队者也可能约定交替使用咖啡机，也可能是大家同时竞争咖啡机，谁先竞争到咖啡机谁使用，不过后一种的方法可能引发冲突，因为两个队列里面排在队列首位的人可能同时使用咖啡机），每个等待者在使用咖啡机之前不仅需要知道排在他前面那个人是否已经使用完了咖啡机，还需知道另一个队列中排在首位的人是否也正准备使用咖啡机；而并行是每个队列拥有自己的咖啡机，两个队列之间并没有竞争的关系，队列中的某个排队者只需等待队列前面的人使用完咖啡机，然后再轮到自己使用咖啡机。

因此，并发意味着多个执行实体（比方说上面例子中的人）可能需要竞争资源（咖啡机），因此就不可避免带来竞争和同步的问题；而并行则是不同的执行实体拥有各自的资源，相互之间可能互不干扰。

**Go发明者之一Rob Pike**认为并发是程序本身的一种特性，程序被分为多个可独立执行的部分，而各个可独立执行的片段通过通信手段进行协调，而并行则是程序的计算过程（不同的计算过程可能相关联）同时执行。
Rob Pike 的观点是： 并发是一次处理(dealing with)很多事情，而并行是一次做(doing)很多事情.(注: 英文词汇的表达也很微妙)原文是如下：

> Concurrency is about dealing with lots of things at once.
> Parallelism is about doing lots of things at once.

前者是关于程序结构的，而后者是关于程序执行的,Rob认为：

> Concurrency provides a way to structure a solution to solve a problem that may (but not necessarily) be parallelizable.

即我们可以利用并发的手段去构建一种解决方案来解决那些有可能被并行处理的问题。作者在本文中还提到，设计并发程序时应该将程序分为多个执行片段，使得每个片段可以独立执行。不同执行片段通过通信(Communication )来进行协调。因此Go的并发模型基于CSP: C. A. R. Hoare: Communicating Sequential Processes (CACM 1978)

**Intel中文网站**的一篇文章曾这样写道:

> 并发（Concurrence）：指两个或两个以上的事件或活动在同一时间间隔内发生。并发的实质是单个物理 CPU(也可以多个物理CPU) 在若干道程序之间多路复用，并发可以对有限物理资源强制行使多用户共享以提高效率，如下图所示：

{% asset_img intel-concurrence.png 并发与并行 %}

> 并行（Parallelism）指两个或两个以上事件或活动在同一时刻发生。在多道程序环境下，并行性使多个程序同一时刻可在不同CPU上同时执行，如下图所示：

{% asset_img intel-parallelism.png 并发与并行 %}

此，该文认为并发与并行的区别是：并发是一个处理器同时处理多个任务，而并行多个处理器或者是多核的处理器同时处理多个不同的任务。前者是逻辑上的同时发生（simultaneous），而后者是物理上的同时发生。而两者的联系是：并行的事件或活动一定是并发的，但反之并发的事件或活动未必是并行的。并行性是并发性的特例，而并发性是并行性的扩展。

## 互联网的高并发

高并发（High Concurrency）是互联网分布式系统架构设计中必须考虑的因素之一，它通常是指，通过设计保证系统能够同时并行处理很多请求。高并发相关常用的一些指标有响应时间（Response Time），吞吐量（Throughput），每秒查询率QPS（Query Per Second），并发用户数等。

|概念|说明|
|----|----|
|响应时间|系统对请求做出响应的时间。例如系统处理一个HTTP请求需要200ms，这个200ms就是系统的响应时间。|
|吞吐量|单位时间内处理的请求数量。|
|请求数|QPS（Query Per Second）/RPS（Request Per Second）每秒响应请求数。请求数指的是客户端在建立完连接后，向HTTP服务发出GET/POST/HEAD数据包。在互联网领域，这个指标和吞吐量区分的没有这么明显。|
|并发用户数|同时承载正常使用系统功能的用户数量。例如一个即时通讯系统，同时在线量一定程度上代表了系统的并发用户数。|
|并发连接数|SBC（Simultaneous Browser Connections）指的是客户端向服务器发起请求，并建立了TCP连接。每秒钟服务器链接的总TCP数量，就是并发连接数。|

面对互联网高并发的场景，常见的架构思路有负载均衡、异步处理、 限流阀（throttle）、批量处理、数据分区、数据镜像、缓存系统、CDN、排队系统等。

# 锁

并发编程意味着同一资源被并发的访问，对于需要保持数据一致性的场景中，需要引入额外的同步机制，锁就是用来解决并发编程中资源共享安全性的问题的。

## synchronized

使用synchronized关键字声明的方法或者包含的代码块只允许一个线程获得锁，其它线程只能等待锁的释放。在这里要释放锁有如下情况:

1. 正常情况下，当代码执行完毕会释放锁。
2. 当线程执行发生异常，JVM会让线程自动释放锁。

在JDK8中JVM对synchronized关键字做了大量优化，其性能已经有很大的提升。

## Lock

相对于synchronized，Lock可以知道是否获取到了锁、可以中断线程的等待,而synchronized不能。

1. synchronized是java语言的关键字，是java的内置特性，而lock只是一个类。
2. synchronzied是系统释放锁,而不需要手动地去释放锁。但是lock需要手动的调用unlock来释放锁，否则会出现死锁。

Lock接口如下：

```java
public interface Lock {
    void lock();
    void lockInterruptibly() throws InterruptedException;
    boolean tryLock();
    boolean tryLock(long time, TimeUnit unit) throwsInterruptedException;   
    void unlock();
    Condition newCondition();
}
```

从方法名字可以看出，lock(),tryLock(),tryLock(long time,TimeUnit unit),lockInterruptibly()是用来获取锁的。

### lock()

此方法就是用来获取一个锁的，如果锁已被别的线程获得，那就进行等待。前面讲到必须使用lock必须手动释放锁,并且在线程发生异常时，系统也不会自动释放的，所以我们就应该把任务代码放在try{}catch(){}中执行最后 在finally{}代码里面释放:

```java
lock.lock();
try{
    // do something
} catch() {
    // do something
}finally{
    lock.unlock()
}
```

### tryLock()&tryLock(long time,TimeUnit unit)

这个方法是有返回值的，它用来尝试获取锁，如果获取成功则返回true,反之则返回false,这个方法会立即返回，并不会等待。`tryLock(long time,TimeUnit unit)`和`tryLock`类似，只是前者如果一开始没有获取到锁会等待一段时间。如果在时间范围内拿到了锁就会返回true。

```java
if(lock.tryLock()) {
  try{
    //处理任务
  }catch(Exception ex){
  }finally{
    lock.unlock();   //释放锁
  }
}else {
  //如果不能获取锁，则直接做其他事情
}
```

### lockInterruptibly()

这个方法比较特殊。当通过这个方法获取锁时，如果此钱程正在等待获取锁，则这个钱程可以响应等待中断，即中断钱程等待状态。如当A,B线程都试图使用lockInterruptibly()获取锁时，如果A获得了锁，B线程正在等待获取锁，则可以调用threadB.interrupt()能够中断线程B的等待。因为该方法抛出了异常。所以应该放在try代码块或者在方法申明时抛出异常。一般使用格式:

```java
try{
  lock.lockInterruptibly();
}catch(InterruptedException  e){
}finally{
  lock.unlock();
}
```

注意的是，如果线程已经获得了锁是不会中断的。并且正在执行的线程是不会被中断的，只有在阻塞中的线程才会被中断。而用synchronized修饰的话，当一个线程处于等待某个锁的状态，是无法被中断的，只有一直等待下去。

### newCondition()

此方法返回一个Condition 对象。Condition是为解决Object.wait/notify/notifyAll难以使用的问题。

synchronized锁配合的是线程等待（Object.wait）与线程通知(Object.notify),那么对于JDK1.5的`java.util.concurrent.locks.ReentrantLock`锁，JDK也为我们提供了与此功能相应的`java.util.concurrent.locks.Condition`。在Condition中，用await()替换wait()，用signal()替换notify()，用signalAll()替换notifyAll()，传统线程的通信方式，Condition都可以实现，这里注意Condition是被绑定到Lock上的，要创建一个Lock的Condition必须用newCondition()方法。接口方法名称的改变是为了区分Object中的方法，因为Condition也是Object的子类。在调用await()方法前线程必须获得重入锁，调用await()方法后线程会释放当前占用的锁。同理在调用`signal()`方法时当前线程也必须获得相应重入锁，调用`signal()`方法后系统会从`condition.await()`等待队列中唤醒一个线程。当线程被唤醒后，它就会尝试重新获得与之绑定的重入锁，一旦获取成功将继续执行。所以调用`signal()`方法后一定要释放当前占用的锁，这样被唤醒的线程才能有获得锁的机会，才能继续执行。

Condition接口中的方法:

```java
void await() throws InterruptedException;
void awaitUninterruptibly();
long awaitNanos(long nanosTimeout) throws InterruptedException;
boolean await(long time, TimeUnit unit) throws InterruptedException;
boolean awaitUntil(Date deadline) throws InterruptedException;
void signal();
void signalAll();
```

Condition是与Lock绑定的，所以也有Lock的公平性：如果是公平锁，线程为按照FIFO的顺序从Condition.await中释放，如果是非公平锁，那么后续的锁竞争就不保证FIFO顺序了。

 一个使用Condition实现生产者消费者的例子：

```java
final Lock lock = new ReentrantLock();//锁对象
final Condition notFull  = lock.newCondition();//写线程条件
final Condition notEmpty = lock.newCondition();//读线程条件

final Object[] items = new Object[100];//缓存队列
int putptr/*写索引*/, takeptr/*读索引*/, count/*队列中存在的数据个数*/;

public void put(Object x) throws InterruptedException {
  lock.lock();
  try {
    while (count == items.length) //如果队列满了
      notFull.await();//阻塞写线程
    items[putptr] = x;//赋值
    if (++putptr == items.length) putptr = 0;//如果写索引写到队列的最后一个位置了，那么置为0
    ++count;//个数++
    notEmpty.signal();//唤醒读线程

  } finally {
    lock.unlock();
  }
}

public Object take() throws InterruptedException {
  lock.lock();
  try {
       while (count == 0)//如果队列为空
         notEmpty.await();//阻塞读线程
       Object x = items[takeptr];//取值 
       if (++takeptr == items.length) takeptr = 0;//如果读索引读到队列的最后一个位置了，那么置为0
       --count;//个数--
       notFull.signal();//唤醒写线程
       return x;
  } finally {
       lock.unlock();
  }
}

```

### ReentrantLock

ReentrantLock类实现了Lock，它拥有与synchronized相同的并发性和内存语义，但是添加了类似锁投票、定时锁等候和可中断锁等候的一些特性。此外，它还提供了在激烈争用情况下更佳的性能。这意味着当许多线程都在争用同一个锁时，使用ReentrantLock的总体开支通常要比 synchronized少得多

ReentrantLock构造器的一个参数是boolean值，它允许您选择想要一个公平（fair）锁，还是一个不公平（unfair）锁。公平锁使线程按照请求锁的顺序依次获得锁；而不公平锁则允许讨价还价，在这种情况下，线程有时可以比先请求锁的其他线程先得到锁。当然，公平总是好的，但是你需要保证它公平那你就需要牺牲性能。花掉性能成本来保证它是公平锁。作为默认设置，除非公平对您的算法至关重要，需要严格按照线程排队的顺序对其进行服务。否则我们都应该把公平设置为false，默认的无参构造函数就是非公平锁。

传统的synchronized是不公平的，而且永远都不公平。但是没有人抱怨过线程饥渴，因为JVM保证了所有线程最终都会得到它们所等候的锁。确保统计上的公平性，对多数情况来说，这就已经足够了，而这花费的成本则要比绝对的公平保证的低得多。

从ReentrantLock的名字知道，它是一个可重入锁。也就是说支持同一个线程对同一资源的重复加锁。另外synchronized关键字隐式的支持重进入。

> 可重入锁指的是在一个线程中可以多次获取同一把锁，比如：一个线程在执行一个带锁的方法，该方法中又调用了另一个需要相同锁的方法，则该线程可以直接执行调用的方法，而无需重新获得锁；

```java
public class ReentrantLockTest {
  private Lock lock = new ReentrantLock();
  private int count;

  public static void main(String[] args) {
    ReentrantLockTest test = new ReentrantLockTest();
    for (int i = 0; i < 3; i++) {
      new Thread() {
      @Override
      public void run() {
        test.test(this);
      }
      }.start();
    }
  }

  protected void test(Thread thread) {
    lock.lock();
    try {
      for (int i = 0; i < 5; i++) {
        count++;
        System.out.println("当前线程" + thread.getName() + "之后" + count);    
      }
    } finally {
      lock.unlock();
    }
  }

}
```

还有为保证读写效率提高的读写锁:ReentrantReadWriteLock ,它与ReentrantLock是相互独立的实现。没有继承和实现关系。

## ReentrantLock与synchronized的区别

1. 与synchronized相比，ReentrantLock提供了更多，更加全面的功能，具备更强的扩展性。例如：时间锁等候，可中断锁等候，锁投票。
2. ReentrantLock还提供了条件Condition，对线程的等待、唤醒操作更加详细和灵活，所以在多个条件变量和高度竞争锁的地方，ReentrantLock更加适合。
3. ReentrantLock提供了可轮询的锁请求。它会尝试着去获取锁，如果成功则继续，否则可以等到下次运行时处理，而synchronized则一旦进入锁请求要么成功要么阻塞，所以相比synchronized而言，ReentrantLock会不容易产生死锁些。
4. ReentrantLock支持更加灵活的同步代码块，但是使用synchronized时，只能在同一个synchronized块结构中获取和释放。(注：ReentrantLock的锁释放一定要在finally中处理，否则可能会产生严重的后果。)
5. ReentrantLock支持中断处理，且性能较synchronized会好些。

> 看起来ReentrantLock功能比synchronized强大得多，那是不是我们以后在做线程同步的时候都应该用ReentrantLock而抛弃synchronized呢？

当然不是，前面 我们说过使用Lock的时候，我们必须在finally块中手动调用lock.unlock()。如果我们忘记了，那么会为程序埋下很大的隐患。还有一个原因就是当JVM用synchronized管理锁定请求和释放时，JVM 在生成线程转储时能够包括锁定信息。这些对调试非常有价值，因为它们能标识死锁或者其他异常行为的来源。Lock类只是普通的类，JVM不知道具体哪个线程拥有Lock对象。而且，几乎每个开发人员都熟悉synchronized，它可以在JVM的所有版本中工作。在JDK 5.0成为标准（从现在开始可能需要两年之前，使用 Lock类将意味着要利用的特性不是每个JVM都有的，而且不是每个开发人员都熟悉的。

那我们什么时候应该使用Lock而不是synchronized呢?既然如此，我们什么时候才应该使用 ReentrantLock 呢？答案非常简单: 在确实需要一些synchronized所没有的特性的时候，比如时间锁等候、可中断锁等候、无块结构锁、多个条件变量或者锁投票。ReentrantLock还具有可伸缩性的好处，应当在高度争用的情况下使用它，但是请记住，大多数synchronized块几乎从来没有出现过争用，所以可以把高度争用放在一边。我建议用 synchronized开发，直到确实证明synchronized不合适，而不要仅仅是假设如果使用ReentrantLock “性能会更好”。请记住，这些是供高级用户使用的高级工具。（而且，真正的高级用户喜欢选择能够找到的最简单工具，直到他们认为简单的工具不适用为止。）。

## JVM锁的优化

锁粗化、轻量级锁、偏向锁、锁消除、适应自旋锁。
