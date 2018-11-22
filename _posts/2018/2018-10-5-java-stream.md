---
layout: post
title: java8 stream详解
category: java
tags: [java]
---



## 概览
Stream的出现，是弥补集合的不足，很多在集合中需要迭代的操作才能完成的功能，使用stream,可以很好的完成。

### 流和集合的区别
简单而言，流和集合的区别，在于何时计算，集合中的元素是必须确定后，才构成集合；流是按需计算的。流只能消费一次，如果想要再次遍历，就得重新获取流。流操作让迭代内部化，对于集合的操作就不需要面向单独的集合元素了

### 使用流对象去重
Stream提供的distinct()方法只能去除重复的对象，无法根据指定的对象属性进行去重，可以应付简单场景
```
List<User> us = users.stream().collect(
        collectingAndThen(toCollection(() -> new TreeSet<>(Comparator.comparing(u -> u.getId()))),ArrayList::new))
```
使用上述代码可以根据指定元素去重（User的id属性），原理是把id作为key放入set中，然后在转换为list,但是这种操作会执行终端操作。

下面代码是在网上看见的
```
List<User> us =  users.stream()
        .filter(distinctByKey(o -> o.getId()))
        .collect(Collectors.toList());
```
distinctByKey()是自定义的，如下
```
public static <T> Predicate<T> distinctByKey(Function<? super T, Object> keyExtractor) {
    Map<Object, Boolean> seen = new ConcurrentHashMap<>();
    return t -> seen.putIfAbsent(keyExtractor.apply(t), Boolean.TRUE) == null;
  }
```
首先是filter方法，返回一个流，需要一个Predicate类型的参数(Predicate这个函数式接口，接受一个参数，返回布尔值)
```
Stream<T> filter(Predicate<? super T> predicate)
```
filter根据Predicate返回的布尔值来判断是否要过滤掉，会过滤掉返回值为false的数据。而我们自己定义的distinctByKey返回值就是Predicate，所以可以作为参数传入filter。distinctByKey也需要一个Function的参数。distinctByKey先是定义了一个线程安全的Map(相比于Hashtable以及Collections.synchronizedMap()，ConcurrentHashMap在线程安全的基础上提供了更好的写并发能力，但同时降低了对读一致性的要求)，因为在流计算中是多线程处理的，需要线程安全。然后将值作为key,TRUE作为value put到map中。这里的put方法使用的是putIfAbsent()。putIfAbsent()方法是如果key不存在则put如map中，并返回null。若key存在，则直接返回key所对应的value值。文章末尾贴上putIfAbsent()源码。所以 
```
seen.putIfAbsent(keyExtractor.apply(t), Boolean.TRUE) == null;

```
这里用 == null判断是否存在于map了，若存在则返回false，不存在则为true。以用来达到去重的目的。

```
default V putIfAbsent(K key, V value) {
        V v = get(key);
        if (v == null) {
            v = put(key, value);
        }
 
        return v;
    }

```
注：putIfAbsent()定义在Map接口中，是默认方法。

### 使用filter对流进行判空

#### 操作集合对象
```
 List<User> manUsers = users.stream()
                // 此处自定义筛选条件
                .filter(u -> u.getName() != null)
                .collect(Collectors.toList());
```
#### 操作map
```
 Map<String, Object> collect = map.entrySet().stream()
                // 此处自定义筛选条件
                .filter((e) -> e.getValue() != null)
                .collect(Collectors.toMap(
                        (e) -> (String) e.getKey(),
                        (e) -> e.getValue()
                ));
```
### 使用map操作集合对象
```
   List<String> names = users.stream()
                // 只取名字
                .map(User::getName)
                .collect(Collectors.toList());
```
### 统计集合重复元素出现次数，并且去重返回hashMap
```
  Map<String, Long> map = list.stream().
      collect(Collectors.groupingBy(Function.identity(),Collectors.counting()));

```


[参考链接](http://www.importnew.com/24029.html)