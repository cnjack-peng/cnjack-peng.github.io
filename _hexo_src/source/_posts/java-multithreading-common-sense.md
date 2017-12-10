---
layout: post
title: Java多线程常识
date: 2017-12-09 23:34:25
tags:
categories: 软件开发
---

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
上面就是关于java中双重检查模式（double-check idiom）的一般结论。但是事情还没有结束，因为java的内存模式也在改进中。Doug Lea 在他的文章中写道：“根据最新的 JSR133 的 Java 内存模型，如果将引用类型声明为 volatile，双重检查模式就可以工作了”，参见 http://gee.cs.oswego.edu/dl/cpj/updates.html 。所以以后要在 Java 中使用双重检查模式，可以使用下面的代码：
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
推荐方法 是Initialization on Demand Holder（IODH），详见 http://en.wikipedia.org/wiki/Initialization_on_demand_holder_idiom
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


