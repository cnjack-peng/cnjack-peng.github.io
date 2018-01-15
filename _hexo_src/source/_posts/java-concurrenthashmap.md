---
layout: post
title: Java - ConcurrentHashMap
date: 2018-01-15 11:38:14
tags:
categories: 软件开发
---


## 方法

### getOrDefault
如果指定的key存在，则返回该key对应的value，如果不存在，则返回指定的值。例子如下
```java
System.out.println(map.getOrDefault(4, "d"));
```

### forEach 
遍历Map中的所有Entry, 对key, value进行处理， 接收参数 (K, V) -> void, 例子如下
```java
// 输出1a, 2b, 3c
map.forEach((key, value) -> System.out.println(key + value));
```

### replaceAll
替换Map中所有Entry的value值，这个值由旧的key和value计算得出，接收参数 (K, V) -> V, 类似如下代码
```java
for (Map.Entry<K, V> entry : map.entrySet())
    entry.setValue(function.apply(entry.getKey(), entry.getValue()));
```
示例如下：
```java
map.replaceAll((key, value) -> (key + 1) + value);
// 输出 12a 23b 34c
map.forEach((key, value) -> System.out.println(key + value));
```

### putIfAbsent 
putIfAbsent用来原子性的获取键值，如果键值存在则返回原来的值，不存在设置为给定的值后返回null，类似如下代码：
```java
V v = map.get(key);
if (v == null)
    v = map.put(key, value);
return v;
```
这里需要注意的是，如果改key已经有值，则总是返回旧的值，并不会将你调用时传入的第二个参数set进去，这个方法通常用来初始化键值。示例代码如下：
```java
map.putIfAbsent(3, "d");
map.putIfAbsent(4, "d");
// 输出 c
System.out.println(map.get(3));
// 输出 d
System.out.println(map.get(4));
```

### remove 
接收2个参数，key和value，如果key关联的value值与指定的value值相等（equals)，则删除这个元素，类似代码如下：
```java
if (map.containsKey(key) && Objects.equals(map.get(key), value)) {
    map.remove(key);
    return true;
} else
    return false;
```
示例代码如下：
```java
map.remove(1, "b");
// 未删除成功， 输出 a
System.out.println(map.get(1));
map.remove(2, "b");
// 删除成功，输出 null
System.out.println(map.get(2));

```
在多线程编程中，可以根据remove的结果（boolean）判断是否删除成功，从而进行业务处理，这个删除方法是线程安全的。

### replace(K key, V oldValue, V newValue)
如果key关联的值与指定的oldValue的值相等，则替换成新的newValue，类似代码如下：
```java
if (map.containsKey(key) && Objects.equals(map.get(key), value)) {
    map.put(key, newValue);
    return true;
} else
    return false;
```

示例代码如下
```java
map.replace(3, "a", "z");
// 未替换成功，输出 c
System.out.println(map.get(3));
map.replace(1, "a", "z");
// 替换成功， 输出 z
System.out.println(map.get(1));

```

### replace(K key, V value) 
如果map中存在key，则替换成value值，否则返回null, 类似代码如下:
```java
if (map.containsKey(key)) {
    return map.put(key, value);
} else
    return null;
```

示例代码如下：
```java
// 输出旧的值， a
System.out.println(map.replace(1, "aa"));
// 替换成功，输出新的值， aa
System.out.println(map.get(1));
// 不存在key为4， 输出 null
System.out.println(map.replace(4, "d"));
// 不存在key为4， 输出 null
System.out.println(map.get(4));

```

### computeIfAbsent
如果指定的key不存在，则通过指定的K -> V计算出新的值设置为key的值，类似代码如下：
```java
if (map.get(key) == null) {
    V newValue = mappingFunction.apply(key);
    if (newValue != null)
        map.put(key, newValue);
}
```

示例代码如下：
```java
map.computeIfAbsent(1, key -> key + " computed");
// 存在key为1，则不进行计算，输出值 a
System.out.println(map.get(1));
map.computeIfAbsent(4, key -> key + " computed");
// 不存在key为4，则进行计算，输出值 4 computed
System.out.println(map.get(4));
```

### computeIfPresent 
如果指定的key存在，则根据旧的key和value计算新的值newValue, 如果newValue不为null，则设置key新的值为newValue, 如果newValue为null, 则删除该key的值，类似代码如下：
```java
if (map.get(key) != null) {
    V oldValue = map.get(key);
    V newValue = remappingFunction.apply(key, oldValue);
    if (newValue != null)
        map.put(key, newValue);
    else
        map.remove(key);
}
```

示例代码如下：
```java
map.computeIfPresent(1, (key, value) -> (key + 1) + value);
// 存在key为1， 则根据旧的key和value计算新的值，输出 2a
System.out.println(map.get(1));
map.computeIfPresent(2, (key, value) -> null);
// 存在key为2， 根据旧的key和value计算得到null，删除该值，输出 null
System.out.println(map.get(2));
```

### compute 
compute方法是computeIfAbsent与computeIfPresent的综合体。

### merge(K key, V value, BiFunction<? super V, ? super V, ? extends V> remappingFunction) 

如果指定的key不存在，则设置指定的value值，否则根据key的旧的值oldvalue，value计算出新的值newValue, 如果newValue为null, 则删除该key，否则设置key的新值newValue。类似如下代码：
```java
V oldValue = map.get(key);
V newValue = (oldValue == null) ? value :
        remappingFunction.apply(oldValue, value);
if (newValue == null)
    map.remove(key);
else
    map.put(key, newValue);
```

示例代码如下：

```java
// 存在key为1， 输出 a merge
System.out.println(map.merge(1, " merge", (oldValue, newValue) -> oldValue + newValue));
// 新值为null，删除key，输出 null
System.out.println(map.merge(1, " merge", (oldValue, newValue) -> null));
// 输出 " merge"
System.out.println(map.merge(4, " merge", (oldValue, newValue) -> oldValue + newValue));

```