---
title: Java关键字通识
date: 2017-12-11 14:32:40
tags:
---

## strictfp
strictfp的意思是FP-strict，也就是说精确浮点的意思。在Java虚拟机进行浮点运算时，如果没有指定strictfp关键字时，Java的编译器以及运行环境在对浮点运算的表达式是采取一种近似于我行我素的行为来完成这些操作，以致于得到的结果往往无法令你满意。而一旦使用了strictfp来声明一个类、接口或者方法时，那么所声明的范围内Java的编译器以及运行环境会完全依照浮点规范IEEE-754来执行。因此如果你想让你的浮点运算更加精确，而且不会因为不同的硬件平台所执行的结果不一致的话，那就请用关键字strictfp。

strictfp不能解决所谓”精确“问题，是用来保证可移植性的。如果没有加strictfp关键字，在不同的平台下面，可能会得到不同的结果。使用strictfp关键字的目的，是保证平台移植之后，浮点运算结果是一致的。如果要进行精确的金额计算，还是需要使用BigDecimal。


## transient 
变量修饰符(只能修饰字段)。标记为transient的变量，在对象存储时，这些变量状态不会被持久化。当对象序列化的保存在存储器上时，不希望有些字段数据被保存，为了保证安全性，可以把这些字段声明为transient。

## volatile 
volatile修饰变量。在每次被线程访问时，都强迫从共享内存中重读该成员变量的值。而且，当成员变量发生变化时，强迫线程将变化值回写到共享内存。这样在任何时刻，两个不同的线程总是看到某个成员变量的同一个值。

Java内存模型规定了所有变量都存储在主内存中，每条线程都有自己的工作内存，线程的工作内存保存了被该线程使用到变量的主内存副本拷贝，线程对变量的所有操作(读取,赋值等)都必须在工作内存中进行，而不能直接读写主内存中的变量。不同线程也不能直接访问对方工作内存中的变量，线程间变量值的传递均需要通过主内存来完成。

当一个变量定义成volatile之后, 保证了此变量对所有线程的可见性,也就是说当一条线程修改了这个变量的值,新的值对于其它线程来说是可以立即得知的.此时,该变量的读写操作直接在主内存中完成。Volatile 变量具有 synchronized 的可见性特性，但是不具备原子特性。虽然增量操作（x++）看上去类似一个单独操作，实际上它是一个由读取－修改－写入操作序列组成的组合操作，必须以原子方式执行，而 volatile 不能提供必须的原子特性。 在多线程并发的环境下, 各个线程的读/写操作可能有重叠现象, 在这个时候, volatile并不能保证数据同步。

volatile适用于简单的标志量的场景中。如：
```java
public class CheesyCounter {

    private volatile int value;

    public int getValue() { return value; }

    public synchronized int increment() {
        return value++;
    }
}
```

## synchronized   
通过 synchronized 关键字来实现，所有加上synchronized 和 块语句，在多线程访问的时候，同一时刻只能有一个线程能够用。synchronized用来修饰方法或者代码块。