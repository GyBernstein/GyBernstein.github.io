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
一般认为，forEach 和常规 for 循环的差异不涉及到性能，它们仅仅是函数式风格与传统 Java 风格的差别。个人认为，处理少量数据的时候，可能还是for循环比较快，对于大量数据，还是stream高效稳定。  

如果想要循环几遍，进行不同的操作，可以使用peek，peek对每个元素执行操作并返回一个新的stream：
~~~ java
Stream.of("one", "two", "three", "four")
 .filter(e -> e.length() > 3)
 .peek(e -> System.out.println("Filtered value: " + e)) // 可以替换为别的功能函数
 .map(String::toUpperCase)
 .peek(e -> System.out.println("Mapped value: " + e))
 .collect(Collectors.toList());
 ~~~

forEach 不能修改自己包含的本地变量值，也不能用 break/return 之类的关键字提前结束循环。  

__findFirst__  

这是一个 termimal 兼 short-circuiting 操作，它总是返回 Stream 的第一个元素，或者空。  

这里比较重点的是它的返回值类型：Optional。这也是一个模仿 Scala 语言中的概念，作为一个容器，它可能含有某值，或者不包含。使用它的目的是尽可能避免 NullPointerException。  

~~~ java
public static int getLength(String text) {
	// Java 8
	return Optional.ofNullable(text).map(String::length).orElse(-1);
	// Pre-Java 8
	// return if (text != null) ? text.length() : -1;
 };
 ~~~
 在更复杂的 if (xx != null) 的情况中，使用 Optional 代码的可读性更好，而且它提供的是编译时检查，能极大的降低 NPE 这种 Runtime Exception 对程序的影响，或者迫使程序员更早的在编码阶段处理空值问题，而不是留到运行时再发现和调试。
Stream 中的 findAny、max/min、reduce 等方法等返回 Optional 值。还有例如 IntStream.average() 返回 OptionalDouble 等等。


__reduce__  

这个方法的主要作用是把 Stream 元素组合起来。  
没有起始值的情况，这时会把 Stream 的前面两个元素组合起来，返回的是 Optional，有起始值时返回具体对象。  
可以返回一个值，或者对象（list等也是对象），对于reduce解决不了的，可以考虑自定义collector 

关于入参，套路差不多，第一个参数是初始值/返回类型容器，可不填。第二个参数是线程内两个元素合并的方法，返回新元素，必填。第三个参数是并行时不同线程运算结果合并的方法，可不填。  
定义collector时多了第四个参数，指定返回什么。

eg. reduce的用例
~~~ java
// 字符串连接，concat = "ABCD"
String concat = Stream.of("A", "B", "C", "D").reduce("", String::concat); 
// 求最小值，minValue = -3.0
double minValue = Stream.of(-1.5, 1.0, -3.0, -2.0).reduce(Double.MAX_VALUE, Double::min); 
// 求和，sumValue = 10, 有起始值
int sumValue = Stream.of(1, 2, 3, 4).reduce(0, Integer::sum);
// 求和，sumValue = 10, 无起始值
sumValue = Stream.of(1, 2, 3, 4).reduce(Integer::sum).get();
// 过滤，字符串连接，concat = "ace"
concat = Stream.of("a", "B", "c", "D", "e", "F").
 filter(x -> x.compareTo("Z") > 0).
 reduce("", String::concat);
 ~~~

__limit/skip__  

limit 返回 Stream 的前面 n 个元素；skip 则是扔掉前 n 个元素（它是由一个叫 subStream 的方法改名而来）。
目的有两种，一种是达到short-circuiting目的，减少操作次数；另一种是选取一部分结果。  
当然，先排序，再limit是不会减少操作次数的，但不要求排序后再取值，这在正常的业务逻辑中是不常见的。  
有一点需要注意的是，对一个 parallel 的 Steam 管道来说，如果其元素是有序的，那么 limit 操作的成本会比较大，因为它的返回对象必须是前 n 个也有一样次序的元素。取而代之的策略是取消元素间的次序，或者不要用 parallel Stream。

__sorted__  

对 Stream 的排序通过 sorted 进行，它比数组的排序更强之处在于你可以首先对 Stream 进行各类 map、filter、limit、skip 甚至 distinct 来减少元素数量后，再排序，这能帮助程序明显缩短执行时间。  

当然，这种优化是有 business logic 上的局限性的：即不要求排序后再取值。
eg. 排序前进行 limit 和 skip
~~~ java
List<Person> persons = new ArrayList();
 for (int i = 1; i <= 5; i++) {
	Person person = new Person(i, "name" + i);
	persons.add(person);
 }
List<Person> personList2 = persons.stream().limit(2).sorted((p1, p2) -> p1.getName().compareTo(p2.getName())).collect(Collectors.toList());
System.out.println(personList2);
~~~

__min/max/distinct__  

min 和 max 的功能也可以通过对 Stream 元素先排序，再 findFirst 来实现，但前者的性能会更好，为 O(n)，而 sorted 的成本是 O(n log n)。同时它们作为特殊的 reduce 方法被独立出来也是因为求最大最小值是很常见的操作。

__Match__  

Stream 有三个 match 方法，从语义上说：
* allMatch：Stream 中全部元素符合传入的 predicate，返回 true
* anyMatch：Stream 中只要有一个元素符合传入的 predicate，返回 true
* noneMatch：Stream 中没有一个元素符合传入的 predicate，返回 true  

它们都不是要遍历全部元素才能返回结果。例如 allMatch 只要一个元素不满足条件，就 skip 剩下的所有元素，返回 false。

## 进阶
### 自己生成流
* Stream.generate  

通过实现 Supplier 接口，你可以自己来控制流的生成。这种情形通常用于随机数、常量的 Stream，或者需要前后元素间维持着某种状态信息的 Stream。由于它是无限的，在管道中，必须利用 limit 之类的操作限制 Stream 大小。

eg. 生成 10 个随机整数
~~~ java
Supplier<Integer> random = seed::nextInt;
Stream.generate(random).limit(10).forEach(System.out::println);
//Another way
IntStream.generate(() -> (int) (System.nanoTime() % 100)).
limit(10).forEach(System.out::println);
~~~

Stream.generate() 还接受自己实现的 Supplier  
eg. 自实现 Supplier
~~~ java
Stream.generate(new PersonSupplier()).
limit(10).
forEach(p -> System.out.println(p.getName() + ", " + p.getAge()));
private class PersonSupplier implements Supplier<Person> {
 private int index = 0;
 private Random random = new Random();
 @Override
 public Person get() {
 return new Person(index++, "StormTestUser" + index, random.nextInt(100));
 }
}
~~~

eg. jenetics的Engine

* Stream.iterate  

iterate 跟 reduce 操作很像，接受一个种子值，和一个 UnaryOperator（例如 f）。然后种子值成为 Stream 的第一个元素，f(seed) 为第二个，f(f(seed)) 第三个，以此类推。  
eg. 生成一个等差数列
~~~ java
Stream.iterate(0, n -> n + 3).limit(10). forEach(x -> System.out.print(x + " "));
~~~

### 用 Collectors 来进行 reduction 操作
java.util.stream.Collectors 类的主要作用就是辅助进行各类有用的 reduction 操作，例如转变输出为 Collection，把 Stream 元素进行归组。  
* groupingBy/partitioningBy  
eg. 按照年龄分组  
~~~ java 
Map<Integer, List<Person>> personGroups = Stream.generate(new PersonSupplier()).
 limit(100).
 collect(Collectors.groupingBy(Person::getAge));
Iterator it = personGroups.entrySet().iterator();
while (it.hasNext()) {
 Map.Entry<Integer, List<Person>> persons = (Map.Entry) it.next();
 System.out.println("Age " + persons.getKey() + " = " + persons.getValue().size());
}
~~~ 
eg. 按照未成年人和成年人归组
~~~ java
Map<Boolean, List<Person>> children = Stream.generate(new PersonSupplier()).
 limit(100).
 collect(Collectors.partitioningBy(p -> p.getAge() < 18));
System.out.println("Children number: " + children.get(true).size());
System.out.println("Adult number: " + children.get(false).size());
~~~
partitioningBy 其实是一种特殊的 groupingBy，它依照条件测试的是否两种结果来构造返回的数据结构，get(true) 和 get(false) 能即为全部的元素对象。

## 特性归纳
* 不是数据结构
* 它没有内部存储，它只是用操作管道从 source（数据结构、数组、generator function、IO channel）抓取数据。
* 它也绝不修改自己所封装的底层数据结构的数据。例如 Stream 的 filter 操作会产生一个不包含被过滤元素的新 Stream，而不是从 source 删除那些元素。
* 所有 Stream 的操作必须以 lambda 表达式为参数
* 不支持索引访问
* 你可以请求第一个元素，但无法请求第二个，第三个，或最后一个。不过请参阅下一项。
很容易生成数组或者 List
* 惰性化
* 很多 Stream 操作是向后延迟的，一直到它弄清楚了最后需要多少数据才会开始。
* Intermediate 操作永远是惰性化的。
* 并行能力
* 当一个 Stream 是并行化的，就不需要再写多线程代码，所有对它的操作会自动并行进行的。
* 可以是无限的
* 集合有固定大小，Stream 则不必。limit(n) 和 findFirst() 这类的 short-circuiting 操作可以对无限的 Stream 进行运算并很快完成。



参考链接：  
[Java 8 中的 Streams API 详解](https://www.ibm.com/developerworks/cn/java/j-lo-java8streamapi/)   
[Java中对List去重, Stream去重](https://www.cnblogs.com/woshimrf/p/java-list-distinct.html)  
[JAVA8-LAMBDA中reduce的用法](https://blog.csdn.net/zhang89xiao/article/details/77164866)