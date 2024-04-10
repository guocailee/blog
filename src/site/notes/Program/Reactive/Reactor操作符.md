---
{"dg-publish":true,"permalink":"/Program/Reactive/Reactor操作符/","noteIcon":""}
---

Reactor 是一个基于 Java 8 的响应式编程库，它提供了丰富的操作符来处理反应式流（Reactor中的`Mono`和`Flux`）。以下是一些常用的Reactor操作符及其用途：
## map
 **`map`** - 对`Mono`或`Flux`中的每个元素应用一个函数，并将结果发出。
```java
Flux.just("1", "2", "3")
   .map(String::toUpperCase)
   .subscribe(System.out::println); // 输出大写字符
```

## flatMap
 **`flatMap`** - 与`map`类似，但可以返回一个`Mono`或`Flux`，并将结果连接起来。
```java
Flux.just("1", "2", "3")
   .flatMap(s -> Flux.fromArray(s.split("")))
   .subscribe(System.out::println); // 输出单个字符
```

## filter
 **`filter`** - 根据给定的谓词函数过滤`Flux`中的元素。
```java
Flux.just(1, 2, 3, 4, 5)
   .filter(i -> i % 2 == 0)
   .subscribe(System.out::println); // 输出偶数
```

## take
 **`take`** - 从`Flux`中取出前N个元素。
```java
Flux.range(1, 10)
   .take(5)
   .subscribe(System.out::println); // 只输出前5个数字
```

## skip
 **`skip`** - 跳过`Flux`中的前N个元素。
```java
Flux.range(1, 10)
   .skip(5)
   .subscribe(System.out::println); // 跳过前5个数字
```
## reduce
 **`reduce`** - 将`Flux`中的所有元素组合成一个单一的值。
```java
Flux.range(1, 3)
   .reduce((a, b) -> a + b)
   .subscribe(System.out::println); // 输出和
```
## concat

 **`concat`** - 将两个`Flux`序列连接起来，第一个`Flux`完成后，第二个`Flux`开始发出元素。
```java
Flux.concat(Flux.just("A"), Flux.just("B", "C"))
   .subscribe(System.out::println); // 输出ABC
```

## merge
 **`merge`** - 将多个`Flux`合并成一个，但有背压支持。
```java
Flux.merge(Flux.just("A"), Flux.just("B", "C"))
   .subscribe(System.out::println); // 输出ABC，无特定顺序
```
## zip
 **`zip`** - 将多个`Mono`或`Flux`的元素组合在一起，创建一个包含所有元素的集合。
```java
Mono.just("A")
   .zipWith(Mono.just("B"), (a, b) -> a + b)
   .subscribe(System.out::println); // 输出AB
```
## delay

 **`delay`** - 延迟`Mono`或`Flux`的发出，可以用于控制流的时序。
```java
Flux.just("Hello")
       .delayElements(Duration.ofSeconds(1))
       .subscribe(System.out::println); // 延迟1秒后输出
```

 ## Retry
 **`retry`** - 当`Mono`或`Flux`发生错误时，重新尝试操作。
```java
Mono.justOrEmpty("Error")
       .retry()
       .subscribe(System.out::println); // 发生错误时重试
```
## doOnNext

 **`doOnNext`** - 在`Flux`的每个元素被订阅者接收之前执行一个操作。
```java
Flux.just("1", "2", "3")
      .doOnNext(s -> System.out.println("Emitting: " + s))
      .subscribe(); // 输出每个元素的发射信息
```

## switchMap
 **`switchMap`** - 将`Flux`中的每个元素转换为一个新的`Mono`或`Flux`，并将这些`Mono`或`Flux`连接起来，但与`flatMap`不同，它会取消之前没有完成的`Mono`或`Flux`。
   ```java
   Flux.just(1, 2, 3)
       .switchMap(i -> Mono.just("String " + i))
       .subscribe(System.out::println); // 输出每个转换后的字符串
   ```

## flatMapSequential
 **`flatMapSequential`** - 类似于`flatMap`，但会顺序执行每个`Mono`或`Flux`，而不是并行。
```java
Flux.just(1, 2, 3)
   .flatMapSequential(i -> Mono.just("String " + i))
   .subscribe(System.out::println); // 顺序输出每个转换后的字符串
```
## publishOn

**`publishOn`** - 指定`Mono`或`Flux`发出元素时使用的`Scheduler`，这有助于控制并发和执行环境。
```java
Mono.just("Hello")
   .publishOn(Schedulers.elastic())
   .subscribe(System.out::println); // 在指定的Scheduler上发出元素
```
## cache

**`cache`** - 缓存`Flux`的元素，使得后续的订阅可以直接获取缓存的元素，而不是重新发出。
```java
Flux.just("1", "2", "3")
   .cache()
   .subscribe(System.out::println); // 输出元素并缓存
   subscribeAgain(); // 再次订阅时，直接获取缓存的元素
```
## distinct
 **`distinct`** - 根据给定的键提取函数，只发出不同的元素。
```java
Flux.just(1, 2, 2, 3, 3, 1)
   .distinct(i -> i)
   .subscribe(System.out::println); // 只输出唯一的元素
```

 ## takeWhile
 **`takeWhile`** - 只要满足给定的谓词函数，就发出`Flux`中的元素。
```java
Flux.range(1, 5)
   .takeWhile(i -> i < 4)
   .subscribe(System.out::println); // 输出小于4的元素
```
## skipWhile

**`skipWhile`** - 跳过`Flux`中满足给定的谓词函数的元素，直到遇到不满足的元素为止。
```java
Flux.range(1, 5)
   .skipWhile(i -> i < 3)
   .subscribe(System.out::println); // 跳过小于3的元素
```

## timeout
**`timeout`** - 如果在指定的时间内没有发出元素，就发出一个错误信号。
```java
Mono.just("Hello")
   .delayElement(Duration.ofSeconds(5))
   .timeout(Duration.ofSeconds(1))
   .subscribe(System.out::println, System.err::println); // 超时后输出错误信息
```

 ## debounce
 
 **`debounce`** - 过滤掉连续发出的元素，只发出经过指定时间间隔后的第一个元素。
```java
Flux.interval(Duration.ofMillis(100))
   .debounce(Duration.ofMillis(200))
   .take(5)
   .subscribe(System.out::println); // 每200毫秒输出一个元素，共输出5个
```

## groupBy
**`groupBy`** - 根据给定的函数将`Flux`的元素分组，每个分组都是一个`Flux`。

```java
Flux.just("one", "two", "three", "four")
      .groupBy(String::length)
      .flatMap(group -> group.map(s -> "Group " + group.key() + " : " + s))
      .subscribe(System.out::println); // 按长度分组并输出
```
