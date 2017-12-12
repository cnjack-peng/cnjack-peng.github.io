---
title: Java内存分析
date: 2017-12-08 17:29:15
tags:
categories: 软件开发
---


## 内存分析
通过`top`命令定位占用大内存的应用，通过`jps`命令找到应用进程号。打印出某个java进程（使用pid）内存内的，所有‘对象’的情况（如：产生那些对象，及其数量）。


### jmap

jmap命令(Java Memory Map)

- `jmap -heap [pid]` 查看整个JVM内存状态;
```bash
Attaching to process ID 22999, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 24.80-b11

using thread-local object allocation.
Parallel GC with 4 thread(s)

Heap Configuration:
   MinHeapFreeRatio = 0
   MaxHeapFreeRatio = 100
   MaxHeapSize      = 2065694720 (1970.0MB)
   NewSize          = 1310720 (1.25MB)
   MaxNewSize       = 17592186044415 MB
   OldSize          = 5439488 (5.1875MB)
   NewRatio         = 2
   SurvivorRatio    = 8
   PermSize         = 21757952 (20.75MB)
   MaxPermSize      = 85983232 (82.0MB)
   G1HeapRegionSize = 0 (0.0MB)

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 24117248 (23.0MB)
   used     = 8397088 (8.008087158203125MB)
   free     = 15720160 (14.991912841796875MB)
   34.81777025305706% used
From Space:
   capacity = 524288 (0.5MB)
   used     = 98304 (0.09375MB)
   free     = 425984 (0.40625MB)
   18.75% used
To Space:
   capacity = 524288 (0.5MB)
   used     = 0 (0.0MB)
   free     = 524288 (0.5MB)
   0.0% used
PS Old Generation
   capacity = 85983232 (82.0MB)
   used     = 44148184 (42.102989196777344MB)
   free     = 41835048 (39.897010803222656MB)
   51.34510877655774% used
PS Perm Generation
   capacity = 39321600 (37.5MB)
   used     = 39033120 (37.224884033203125MB)
   free     = 288480 (0.275115966796875MB)
   99.266357421875% used

19166 interned Strings occupying 1688912 bytes.
```

- `jmap -histo [pid]` 查看JVM堆中对象详细占用情况
```bash
num     #instances         #bytes  class name
----------------------------------------------
 1:         10823        3072312  [B
 2:         16605        2318720  <constMethodKlass>
 3:         18687        1388088  [C
 4:         16605        1328608  <methodKlass>
 5:         27595        1296832  <symbolKlass>
 6:          1699         940392  <constantPoolKlass>
 7:          2520         883408  [I
 8:          1699         724944  <instanceKlassKlass>
 9:          1472         565136  <constantPoolCacheKlass>
10:           256         561152  [Lnet.sf.ehcache.store.chm.SelectableConcurrentHashMap$HashEntry;
11:         12148         291552  java.lang.String
12:          4505         288320  net.sf.ehcache.Element
13:          7290         233280  java.lang.ThreadLocal$ThreadLocalMap$Entry
14:          1946         186816  java.lang.Class
15:          4509         180360  net.sf.ehcache.store.chm.SelectableConcurrentHashMap$HashEntry
```
其中[开头表示数组，[C [I [B 分别是char[] int[] byte[]。constMethodKlass、都实现自sun.jvm.hotspot.oops.Klass，用于在永久代里保存类的信息。

- `jmap -dump:format=b,file=文件名 [pid]` 导出整个JVM 中内存信息,通过`jhat`启动一个Web Server（端口7000）查看分析结果：
```bash
jmap -dump:file=a.txt 22999
jhat -J-Xmx512m a.txt 
```

`kill -3 [pid]`
在Linux 上找到Java所在的进程号，然后执行以上命令，线程的相关信息就输出到console。inux的`kill -3`指令可以帮我们输出当前进程中所有线程的状态，如哪些线程在运行，哪些在等待，因为什么等待，代码哪一行等待。`kill -3`会将信息输出至控制台，所以使用时，被`kill -3`的进程最好是nohup启动的。`kill -3`并不会影响程序运行，不用担心他把程序杀死了。
PS: 需要`-Xrs JVM`选择被使用。


### jstack

jstack 是sun JDK 自带的工具，通过该工具可以看到JVM 中线程的运行状况，包括锁等待，线程是否在运行。