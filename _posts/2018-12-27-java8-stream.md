---
layout: post
title:  "Java8 Stream使用"
date:   2018-12-27
excerpt: "专注于对集合对象进行各种非常便利、高效的聚合操作（aggregate operation），或者大批量数据操作 (bulk data operation)。"
tag:
- java 
- stream
- blog
comments: true
---

## 什么是Stream
Stream 不是集合元素，它不是数据结构并不保存数据，它是有关算法和计算的，更像一个高级版本的Iterator。  
单向，不可往复，数据只能遍历一次，像流水不复返。  
stream可以并行化操作。  
stream的另外一大特点，数据源本身可以是无限的，看完你就知道为什么了。

## 流的构成
通常包括三个基本步骤  
获取一个数据源 -> 数据转换 -> 执行操作获取想要的结果  
每次转换原有stream对象不改变，返回一个新的stream对象，这样对stream的操作可以像链条一样排列，变成一个管道，如下图。
<figure>
    <img src="../assets/img/img001.png">
</figure>

## 获取数据源
### 从collection和数组
* Collection.stream()
* Collection.parallelStream()
* Arrays.stream(T array) or Stream.of()

### 从BufferReader
* java.io.BufferedReader.lines()

### 静态工厂
* java.util.stream.IntStream.range()
* java.nio.file.Files.walk()

### 自己构建
* java.util.Spliterator
* Stream.generate(java.util.Supplier<T>)
* Stream.iterate(seed, f())

### 其他
* Random.ints()
* BigSet.stream()
* Pattern.splitAsStream(Java.lang.CharSequence)
* JarFile.stream()

## 数据转换
### 流的操作类型分为两三种：
* __Intermediate__: 一个流可以后面跟随零个或多个 intermediate 操作。其目的主要是打开流，做出某种程度的数据映射/过滤，然后返回一个新的流，交给下一个操作使用。这类操作都是惰性化的（lazy），就是说，仅仅调用到这类方法，并没有真正开始流的遍历。
* __Terminal__: 一个流只能有一个 terminal 操作，当这个操作执行后，流就被使用“光”了，无法再被操作。所以这必定是流的最后一个操作。Terminal 操作的执行，才会真正开始流的遍历，并且会生成一个结果，或者一个 side effect。
* __Short-circuiting__:   
  * 对于一个 intermediate 操作，如果它接受的是一个无限大（infinite/unbounded）的 Stream，但返回一个有限的新 Stream。  
  * 对于一个 terminal 操作，如果它接受的是一个无限大的 Stream，但能在有限的时间计算出结果。  

当操作一个无限大的 Stream，而又希望在有限时间内完成操作，则在管道内拥有一个 short-circuiting 操作是必要非充分条件。

## 流使用详解
### 构造
对于基本数值型，目前有三种对应的包装类型 Stream：IntStream、LongStream、DoubleStream。当然也可以使用诸如Stream<Integer>，但boxing和unboxing很耗时。  

~~~ java
IntStream.of(new int[]{1, 2, 3}).forEach(System.out::println);
IntStream.range(1, 3).forEach(System.out::println);  
IntStream.rangeClosed(1, 3).forEach(System.out::println);  
~~~

流转换为其他结构
~~~ java
// 1. Array
String[] strArray1 = stream.toArray(String[]::new);
// 2. Collection
List<String> list1 = stream.collect(Collectors.toList());
List<String> list2 = stream.collect(Collectors.toCollection(ArrayList::new));
Set set1 = stream.collect(Collectors.toSet());
Stack stack1 = stream.collect(Collectors.toCollection(Stack::new));
// 3. String
String str = stream.collect(Collectors.joining()).toString();
~~~

### 流的操作
常见操作归类如下：
* Intermediate:  
map (mapToInt, flatMap 等)、 filter、 distinct、 sorted、 peek、 limit、 skip、 parallel、 sequential、 unordered
* Terminal:  
forEach、 forEachOrdered、 toArray、 reduce、 collect、 min、 max、 count、 anyMatch、 allMatch、 noneMatch、 findFirst、 findAny、 iterator
* Short-circuiting:  
anyMatch、 allMatch、 noneMatch、 findFirst、 findAny、 limit

典型用法：  

__map/flatMap__  
map: 作用就是把input Stream的每一个元素，映射成output Stream的另外一个元素。  

eg. 转换大写  
~~~ java
List<String> output = wordList.stream().
map(String::toUpperCase).
collect(Collectors.toList());
~~~

flatMap: 把input Stream中的层级结构扁平化，将最底层元素抽出来放到一起。  
eg. 一对多，也可以推行到Stream\<Object>的情况
~~~ java
Stream<List<Integer>> inputStream = Stream.of(
 Arrays.asList(1),
 Arrays.asList(2, 3),
 Arrays.asList(4, 5, 6)
 );
Stream<Integer> outputStream = inputStream.
flatMap((childList) -> childList.stream());
~~~
最终 output 的新 Stream 里面已经没有 List 了，都是直接的数字。  

__filter__  

filter 对原始 Stream 进行某项测试，通过测试的元素被留下来生成一个新 Stream。  
eg. 把单词挑出来
~~~ java
List<String> output = reader.lines().
 flatMap(line -> Stream.of(line.split(REGEXP))).
 filter(word -> word.length() > 0).
 collect(Collectors.toList());
 ~~~

 __forEach__  

forEach 方法接收一个 Lambda 表达式，然后在 Stream 的每一个元素上执行该表达式。
 ~~~ java
 // Java 8
roster.stream()
 .filter(p -> p.getGender() == Person.Sex.MALE)
 .forEach(p -> System.out.println(p.getName()));
// Pre-Java 8
for (Person p : roster) {
    if (p.getGender() == Person.Sex.MALE) {
        System.out.println(p.getName());
    }
}
~~~