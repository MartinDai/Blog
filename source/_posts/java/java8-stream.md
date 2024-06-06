---
title: Java8 Stream完全使用指南
date: 2020-05-30 20:44:00
categories: 
- Java
---

## 什么是Stream

`Stream`是Java 1.8版本开始提供的一个接口，主要提供对数据集合使用流的方式进行操作，流中的元素不可变且只会被消费一次，所有方法都设计成支持链式调用。使用Stream API可以极大生产力，写出高效率、干净、简洁的代码。

## 如何获得Stream实例

`Stream`提供了静态构建方法，可以基于不同的参数创建返回Stream实例
使用`Collection`的子类实例调用`stream()`或者`parallelStream()`方法也可以得到Stream实例，两个方法的区别在于后续执行`Stream`其他方法的时候是单线程还是多线程

```java
Stream<String> stringStream = Stream.of("1", "2", "3");
//无限长的偶数流
Stream<Integer> evenNumStream = Stream.iterate(0, n -> n + 2);

List<String> strList = new ArrayList<>();
strList.add("1");
strList.add("2");
strList.add("3");
Stream<String> strStream = strList.stream();
Stream<String> strParallelStream = strList.parallelStream();
```

## filter

`filter`方法用于根据指定的条件做过滤，返回符合条件的流

```java
Stream<Integer> numStream = Stream.of(-2, -1, 0, 1, 2, 3);
//获得只包含正数的流，positiveNumStream -> (1，2，3)
Stream<Integer> positiveNumStream = numStream.filter(num -> num > 0);
```

## map

`map`方法用于将流中的每个元素执行指定的转换逻辑，返回其他类型元素的流
```java
Stream<Integer> numStream = Stream.of(-2, -1, 0, 1, 2, 3);
//转换成字符串流
Stream<String> strStream = numStream.map(String::valueOf);
```

<!--more-->

## mapToInt mapToLong mapToDouble

这三个方法是对`map`方法的封装，返回的是官方为各个类型单独定义的Stream，该Stream还提供了适合各自类型的其他操作方法

```java
Stream<String> stringStream = Stream.of("-2", "-1", "0", "1", "2", "3");
IntStream intStream = stringStream.mapToInt(Integer::parseInt);
LongStream longStream = stringStream.mapToLong(Long::parseLong);
DoubleStream doubleStream = stringStream.mapToDouble(Double::parseDouble);
```

## flatMap

`flatMap`方法用于将流中的每个元素转换成其他类型元素的流，比如，当前有一个订单(Order)列表，每个订单又包含多个商品(itemList)，如果要得到所有订单的所有商品汇总，就可以使用该方法，如下：

```java
Stream<Item> allItemStream = orderList.stream().flatMap(order -> order.itemList.stream());
```

## flatMapToInt flatMapToLong flatMapToDouble

这三个方法是对`flatMap`方法的封装，返回的是官方为各个类型单独定义的Stream，使用方法同上

## distinct

`distinct`方法用于对流中的元素去重，判断元素是否重复使用的是`equals`方法

```java
Stream<Integer> numStream = Stream.of(-2, -1, 0, 0, 1, 2, 2, 3);
//不重复的数字流，uniqueNumStream -> (-2, -1, 0, 1, 2, 3)
Stream<Integer> uniqueNumStream = numStream.distinct();
```

## sorted

`sorted`有一个无参和一个有参的方法，用于对流中的元素进行排序。无参方法要求流中的元素必须实现`Comparable`接口，不然会报`java.lang.ClassCastException`异常

```java
Stream<Integer> unorderedStream = Stream.of(5, 6, 32, 7, 27, 4);
//按从小到大排序完成的流，orderedStream -> (4, 5, 6, 7, 27, 32)
Stream<Integer> orderedStream = unorderedStream.sorted();
```

有参方法`sorted(Comparator<? super T> comparator)`不需要元素实现`Comparable`接口，通过指定的元素比较器对流内的元素进行排序

```java
Stream<String> unorderedStream = Stream.of("1234", "123", "12", "12345", "123456", "1");
//按字符串长度从小到大排序完成的流，orderedStream -> ("1", "12", "123", "1234", "12345", "123456")
Stream<String> orderedStream = unorderedStream.sorted(Comparator.comparingInt(String::length));
```

## peek

`peek`方法可以不调整元素顺序和数量的情况下消费每一个元素，然后产生新的流，按文档上的说明，主要是用于对流执行的中间过程做debug的时候使用，因为`Stream`使用的时候一般都是链式调用的，所以可能会执行多次流操作，如果想看每个元素在多次流操作中间的流转情况，就可以使用这个方法实现

```java
Stream.of("one", "two", "three", "four")
     .filter(e -> e.length() > 3)
     .peek(e -> System.out.println("Filtered value: " + e))
     .map(String::toUpperCase)
     .peek(e -> System.out.println("Mapped value: " + e))
     .collect(Collectors.toList());
     
输出：
Filtered value: three
Mapped value: THREE
Filtered value: four
Mapped value: FOUR
```

## limit(long maxSize)

`limit`方法会对流进行顺序截取，从第1个元素开始，保留最多`maxSize`个元素

```java
Stream<String> stringStream = Stream.of("-2", "-1", "0", "1", "2", "3");
//截取前3个元素，subStringStream -> ("-2", "-1", "0")
Stream<String> subStringStream = stringStream.limit(3);
```

## skip(long n)

`skip`方法用于跳过前n个元素，如果流中的元素数量不足n，则返回一个空的流

```java
Stream<String> stringStream = Stream.of("-2", "-1", "0", "1", "2", "3");
//跳过前3个元素，subStringStream -> ("1", "2", "3")
Stream<String> subStringStream = stringStream.skip(3);
```

## forEach

`forEach`方法的作用跟普通的for循环类似，不过这个可以支持多线程遍历，但是不保证遍历的顺序

```java
Stream<String> stringStream = Stream.of("-2", "-1", "0", "1", "2", "3");
//单线程遍历输出元素
stringStream.forEach(System.out::println);
//多线程遍历输出元素
stringStream.parallel().forEach(System.out::println);
```

## forEachOrdered

`forEachOrdered`方法可以保证顺序遍历，比如这个流是从外部传进来的，然后在这之前调用过`parallel`方法开启了多线程执行，就可以使用这个方法保证单线程顺序遍历

```java
Stream<String> stringStream = Stream.of("-2", "-1", "0", "1", "2", "3");
//顺序遍历输出元素
stringStream.forEachOrdered(System.out::println);
//多线程遍历输出元素，下面这行跟上面的执行结果是一样的
//stringStream.parallel().forEachOrdered(System.out::println);
```

## toArray

`toArray`有一个无参和一个有参的方法，无参方法用于把流中的元素转换成Object数组

```java
Stream<String> stringStream = Stream.of("-2", "-1", "0", "1", "2", "3");
Object[] objArray = stringStream.toArray();
```

有参方法`toArray(IntFunction<A[]> generator)`支持把流中的元素转换成指定类型的元素数组

```java
Stream<String> stringStream = Stream.of("-2", "-1", "0", "1", "2", "3");
String[] strArray = stringStream.toArray(String[]::new);
```

## reduce

`reduce`有三个重载方法，作用是对流内元素做累进操作

第一个`reduce(BinaryOperator<T> accumulator)`

`accumulator` 为累进操作的具体计算

单线程等下如下代码

```java
boolean foundAny = false;
T result = null;
for (T element : this stream) {
  if (!foundAny) {
      foundAny = true;
      result = element;
  }
  else
      result = accumulator.apply(result, element);
}
return foundAny ? Optional.of(result) : Optional.empty();
```

```java
Stream<Integer> numStream = Stream.of(-2, -1, 0, 1, 2, 3);
//查找最小值
Optional<Integer> min = numStream.reduce(BinaryOperator.minBy(Integer::compareTo));
//输出 -2
System.out.println(min.get());

//过滤出大于5的元素流
numStream = Stream.of(-2, -1, 0, 1, 2, 3).filter(num -> num > 5);
//查找最小值
min = numStream.reduce(BinaryOperator.minBy(Integer::compareTo));
//输出 Optional.empty
System.out.println(min);
```

第二个`reduce(T identity, BinaryOperator<T> accumulator)`

`identity` 为累进操作的初始值
`accumulator` 同上

单线程等价如下代码

```java
T result = identity;
for (T element : this stream)
  result = accumulator.apply(result, element)
return result;
```

```java
Stream<Integer> numStream = Stream.of(-2, -1, 0, 1, 2, 3);
//累加计算所有元素的和，sum=3
int sum = numStream.reduce(0, Integer::sum);
```

第三个`reduce(U identity, BiFunction<U, ? super T, U> accumulator, BinaryOperator<U> combiner)`

`identity`和`accumulator`同上
`combiner`用于多线程执行的情况下合并最终结果

```java
Stream<Integer> numStream = Stream.of(-2, -1, 0, 1, 2, 3);
int sum = numStream.parallel().reduce(0, (a, b) -> {
    System.out.println("accumulator执行:" + a + " + " + b);
    return a + b;
}, (a, b) -> {
    System.out.println("combiner执行:" + a + " + " + b);
    return a + b;
});
System.out.println("最终结果："+sum);

输出：
accumulator执行:0 + -1
accumulator执行:0 + 1
accumulator执行:0 + 0
accumulator执行:0 + 2
accumulator执行:0 + -2
accumulator执行:0 + 3
combiner执行:2 + 3
combiner执行:-1 + 0
combiner执行:1 + 5
combiner执行:-2 + -1
combiner执行:-3 + 6
最终结果：3
```

## collect

`collect`有两个重载方法，主要作用是把流中的元素作为集合转换成其他`Collection`的子类，其内部实现类似于前面的累进操作

第一个`collect(Supplier<R> supplier, BiConsumer<R, ? super T> accumulator, BiConsumer<R, R> combiner)`

`supplier` 需要返回开始执行时的默认结果
`accumulator` 用于累进计算用
`combiner` 用于多线程合并结果

单线程执行等价于如下代码

```java
R result = supplier.get();
for (T element : this stream)
  accumulator.accept(result, element);
return result;
```

第二个`collect(Collector<? super T, A, R> collector)`

`collector`其实是对上面的方法参数的一个封装，内部执行逻辑是一样的，只不过JDK提供了一些默认的`Collector`实现

```java
Stream<Integer> numStream = Stream.of(-2, -1, 0, 1, 2, 3);
List<Integer> numList = numStream.collect(Collectors.toList());
Set<Integer> numSet = numStream.collect(Collectors.toSet());
```

## min

`min`方法用于计算流内元素的最小值

```java
Stream<Integer> numStream = Stream.of(-2, -1, 0, 1, 2, 3);
Optional<Integer> min = numStream.min(Integer::compareTo);
```

## max

`min`方法用于计算流内元素的最大值

```java
Stream<Integer> numStream = Stream.of(-2, -1, 0, 1, 2, 3);
Optional<Integer> max = numStream.max(Integer::compareTo);
```

## count

`count`方法用于统计流内元素的总个数

```java
Stream<Integer> numStream = Stream.of(-2, -1, 0, 1, 2, 3);
//count=6
long count = numStream.count();
```

## anyMatch

`anyMatch`方法用于匹配校验流内元素是否有符合指定条件的元素

```java
Stream<Integer> numStream = Stream.of(-2, -1, 0, 1, 2, 3);
//判断是否包含正数，hasPositiveNum=true
boolean hasPositiveNum = numStream.anyMatch(num -> num > 0);
```

## allMatch

`allMatch`方法用于匹配校验流内元素是否所有元素都符合指定条件

```java
Stream<Integer> numStream = Stream.of(-2, -1, 0, 1, 2, 3);
//判断是否全部是正数，allNumPositive=false
boolean allNumPositive = numStream.allMatch(num -> num > 0);
```

## noneMatch

`noneMatch`方法用于匹配校验流内元素是否都不符合指定条件

```java
Stream<Integer> numStream = Stream.of(-2, -1, 0, 1, 2, 3);
//判断是否没有小于0的元素，noNegativeNum=false
boolean noNegativeNum = numStream.noneMatch(num -> num < 0);
```

## findFirst

`findFirst`方法用于获取第一个元素，如果流是空的，则返回Optional.empty

```java
Stream<Integer> numStream = Stream.of(-2, -1, 0, 1, 2, 3);
//获取第一个元素，firstNum=-2
Optional<Integer> firstNum = numStream.findFirst();
```

## findAny

`findAny`方法用于获取流中的任意一个元素，如果流是空的，则返回Optional.empty，因为可能会使用多线程，所以不保证每次返回的是同一个元素

```java
Stream<Integer> numStream = Stream.of(-2, -1, 0, 1, 2, 3);
Optional<Integer> anyNum = numStream.findAny();
```