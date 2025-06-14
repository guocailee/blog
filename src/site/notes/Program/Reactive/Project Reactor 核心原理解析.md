---
{"dg-publish":true,"permalink":"/Program/Reactive/Project Reactor 核心原理解析/","noteIcon":"","created":"2025-03-06T21:28:25.982+08:00"}
---

# Project Reactor 核心原理解析
# 一、开篇

本文将解析 Spring 的 Reactor 项目的源码。主要目的是让自己能深入理解 Reactor 这个项目，以及 Spring 5 和 Spring Boot 2。

Project Reactor 项目地址：[https://github.com/reactor](https://github.com/reactor)

Reactor 项目主要包含 Reactor Core 和 Reactor Netty 两部分。Reactor Core 实现了反应式编程的核心功能，Reactor Netty 则是 Spring WebFlux 等技术的基础。本文将介绍 Core 模块的核心原理。

本文从一个例子开始：

```java
public static void main(String[] args) {
  Flux.just("tom", "jack", "allen")
      .filter(s -> s.length() > 3)
      .map(s -> s.concat("@qq.com"))
      .subscribe(System.out::println);
} 
```

整个 Project Reactor 的核心调用分为下面几个阶段：

-   声明阶段
-   subscribe 阶段
-   onSubscribe 阶段
-   request 阶段
-   调用阶段

接下来对每个阶段详细介绍。

# 二、声明阶段

## 简介

之前的文章介绍过，反应式编程和传统风格的编程一个最大的不同点在于，开发人员编写的大部分代码都是在声明处理过程。即这些代码并不会被实际的执行，直到开始被订阅。这便是为何第一阶段是声明阶段的原因。

## 详解

先来看第一个方法 `Flux.just` 的源码：

```java
public static <T> Flux<T> just(T... data) {
  return fromArray(data);
}

public static <T> Flux<T> fromArray(T[] array) {
  // 省略不重要的部分
  return onAssembly(new FluxArray<>(array));
} 
```

> `just` 方法只是创建反应式流的众多方式的一个。在实际工作中，更常见的通过反应式 Repository 将数据库查询结果，或通过 Spring 5 的 `WebClient` 将 HTTP 调用结果最为流的开始。

`onAssembly` 是一个扩展机制，因为实例中并没有使用，所以可简单理解为将入参直接返回。

接下来 `new FluxArray<>(array)` 做的事情也很简单，仅仅是把入参赋给成员变量。

再接下来 `filter(s -> s.length() > 3)` 做的事情是把上一个 `Flux`，即 `FluxArray`，以及 `s -> s.length()> 3` 所表示的 `Predicate` 保存起来。

看一下源码

```java
public final Flux<T> filter(Predicate<? super T> p) {
  if (this instanceof Fuseable) {
    return onAssembly(new FluxFilterFuseable<>(this, p));
  }
  return onAssembly(new FluxFilter<>(this, p));
}

FluxFilterFuseable(Flux<? extends T> source, 
                   Predicate<? super T> predicate) {
  super(source);
  this.predicate = 
      Objects.requireNonNull(predicate, "predicate");
} 
```

这里有个逻辑判断，就是返回的值分为了 `Fuseable` 和非 `Fuseable` 两种。简单介绍一下，Fuseable 的意思就是可融合的。我理解就是 `Flux` 表示的是一个类似集合的概念，有一些集合类型可以将多个元素融合在一起，打包处理，而不必每个元素都一步步的处理，从而提高了效率。因为 `FluxArray` 中的数据一开始就都准备好了，因此可以打包处理，因此就是 `Fuseable`。

而接下来的 `map` 操作也对应一个 `FluxMapFusable`。原理与 `filter` 操作，这里不再重复。

# 三、subscribe 阶段

## 简介

subscribe 阶段同行会触发数据发送。在本例中，后面可以看到，对于 `FluxArray`，数据发送很简单，就是循环发送。而对于像数据库、RPC 这样的长久，则会触发请求的发送。

## 详解

当调用 `subscribe(System.out::println)` 时，整个执行过程便进入 subscribe 阶段。经过一系列的调用之后，subscribe 动作会代理给具体的 `Flux` 来实现。就本例来说，会代理给 `FluxMapFuseable`，方法实现如下（经过简化）：

```java
public void subscribe(CoreSubscriber<? super R> actual) {
  source.subscribe(
      new MapFuseableSubscriber<>(actual, mapper)
  );
} 
```

其中的 `source` 变量对应的是当前 `Flux` 的上一个。本例中，`FluxMapFuseable` 上一个是 `FluxFilterFuseable`。

`new MapFuseableSubscriber<>(actual, mapper)` 则是将订阅了 `FluxMapFuseable` 的 Subscriber 和映射器封装在一起，组成一个新的 Subscriber。然后再订阅 `source`，即 `FluxArray`。`source` 是在上一个阶段被保存下来的。

> 这里强调一下 `Publisher` 接口中的 `subscribe` 方法语义上有些奇特，它表示的不是订阅关系，而是被订阅关系。即 `aPublisher.subscribe(aSubscriber)` 表示的是 `aPublisher` 被 `aSubscriber` 订阅。

接下来调用的就是 `FluxFilterFuseable` 中的 `subscribe` 方法，类似于 `FluxMapFuseable`, 源码如下：

```java
public void subscribe(CoreSubscriber<? super T> actual) {
  source.subscribe(
      new FilterFuseableSubscriber<>(actual, predicate)
  );
} 
```

这时 `source` 就成了 `FluxArray`。于是，接下来是调用 `FluxArray` 的 `subscribe` 方法。

```java
public static <T> void subscribe(CoreSubscriber<? super T> s, 
                                 T[] array) {
  if (array.length == 0) {
    Operators.complete(s);
    return;
  }
  if (s instanceof ConditionalSubscriber) {
    s.onSubscribe(
      new ArrayConditionalSubscription<>(
        (ConditionalSubscriber<? super T>) s, array
      )
    );
  }
  else {
    s.onSubscribe(new ArraySubscription<>(s, array));
  }
} 
```

最重要的部分还是 `s.onSubscribe(new ArraySubscription<>(s, array))`。不同于 `FluxMapFuseable` 和 `FluxFilterFuseable`，`FluxArray` 没有再调用 `subscribe` 方法，因为它是数据源头。而 `FluxMapFuseable` 和 `FluxFilterFuseable` 则是中间过程。

可以这样简单理解，对于中间过程的 `Mono/Flux`，subscribe 阶段是订阅上一个 `Mono/Flux`；而对于源 `Mono/Flux`，则是要执行 `Subscriber.onSubscribe(Subscription s)` 方法。

# 四、onSubscribe 阶段

## 简介

在调用 `FluxArray` 的 `subscribe` 方法之后，执行过程便进入了 onSubscribe 阶段。onSubscribe 阶段指的是 `Subscriber#onSubscribe` 方法被依次调用的阶段。这个阶段会让各 `Subscriber` 知道 `subscribe` 方法已被触发，真正的处理流程马上就要开始。所以这一阶段的工作相对简单。

## 详解

`s.onSubscribe(new ArraySubscription<>(s, array));`

s 是 `FilterFuseableSubscriber`，看一下 `FilterFuseableSubscriber` 的 `onSubscribe(Subscription s)` 源码：

```java
public void onSubscribe(Subscription s) {
  if (Operators.validate(this.s, s)) {
    this.s = (QueueSubscription<T>) s;
    actual.onSubscribe(this);
  }
} 
```

actual 对应 `MapFuseableSubscriber`。`MapFuseableSubscriber` 的 `onSubscribe` 方法也是这样，但 actual 对于的则是代表 `System.out::println` 的 `LambdaSubscriber`。

调用过程：

`FluxArray.subscribe -> FilterFuseableSubscriber.onSubscribe -> MapFuseableSubscriber.onSubscribe -> LambdaSubscriber.onSubscribe`

# 五、request 阶段

## 含义

onSubscribe 阶段是表示订阅动作的方式，让各 `Subscriber` 知悉，准备开始处理数据。当最终的 `Subscriber` 做好处理数据的准备之后，它便会调用 `Subscription` 的 `request` 方法请求数据。

## 详解

下面是 `LambdaSubscriber` `onSubscribe` 方法的源码：

```java
public final void onSubscribe(Subscription s) {
  if (Operators.validate(subscription, s)) {
    this.subscription = s;
    if (subscriptionConsumer != null) {
      try {
        subscriptionConsumer.accept(s);
      }
      catch (Throwable t) {
        Exceptions.throwIfFatal(t);
        s.cancel();
        onError(t);
      }
    }
    else {
      s.request(Long.MAX_VALUE);
    }
  }
} 
```

由上可见 `request` 方法被调用。在本例中，这里的 `s` 是 `MapFuseableSubscriber`。

这里需要说明，如 `MapFuseableSubscriber`、`FilterFuseableSubscriber`，它们都有两个角色。一个角色是 `Subscriber`，另一个角色是 `Subscription`。因为它们都位于调用链的中间，本身并不产生数据，也不需要对数据暂存，但是需要对数据做各式处理。因此，在 onSubscribe、request 阶段，以及后面要讲到的调用阶段都需要起到代理的作用。这就解释了 `actual.onSubscribe(this)` onSubscribe 自己的原因。

下面是 `FilterFuseableSubscriber` 的 `request` 方法的源码。充分可见其代理的角色。

```java
public void request(long n) {
  s.request(n);
} 
```

最后 `ArraySubscription` 的 `request` 方法将被调用。

> 与 map、filter 等操作不同，flatMap 有比较大的差异，感兴趣的同学可以自己研究一下。本文先不详细介绍。

# 六、调用阶段

## 含义解释

这一阶段将会通过调用 `Subscriber` 的 `onNext` 方法，从而进行真正的反应式的数据处理。

## 流程分解

在 `ArraySubscription` 的 `request` 方法被调用之后，执行流程便开始了最后的调用阶段。

```java
public void request(long n) {
  if (Operators.validate(n)) {
    if (Operators.addCap(REQUESTED, this, n) == 0) {
      if (n == Long.MAX_VALUE) {
        fastPath();
      }
      else {
        slowPath(n);
      }
    }
  }
}

void fastPath() {
  final T[] a = array;
  final int len = a.length;
  final Subscriber<? super T> s = actual;
  for (int i = index; i != len; i++) {
      if (cancelled) {
         return;
      }

      T t = a[i];

      if (t == null) {
      s.onError(new NullPointerException("The " 
          + i 
          + "th array element was null")
      );
      return;
    }

    s.onNext(t);
  }
  if (cancelled) {
    return;
  }
  s.onComplete();
} 
```

`ArraySubscription` 会循环数据中的所有元素，然后调用 `Subscriber` 的 `onNext` 方法，将元素交由 Subscriber 链处理。

于是接下来由 `FilterFuseableSubscriber` 处理。下面是其 `onNext` 方法（做了一些简化）

```java
public void onNext(T t) {
    boolean b;

    try {
        b = predicate.test(t);
    }
    catch (Throwable e) {
        Throwable e_ = 
            Operators.onNextError(t, e, this.ctx, s);
        if (e_ != null) {
            onError(e_);
        }
        else {
            s.request(1);
        }
        Operators.onDiscard(t, this.ctx);
        return;
    }
    if (b) {
        actual.onNext(t);
    }
    else {
        s.request(1);
        Operators.onDiscard(t, this.ctx);
    }
} 
```

其最要部分是通过 `predicate` 进行判断，如果满足条件，则交由下一个 `Subscriber` 处理。

下一个要处理的是 `MapFuseableSubscriber`，原理类似，不再重复。

最终要处理数据的是 `LambdaSubscriber`，下面是它的 `onNext` 方法源码：

```java
public final void onNext(T x) {
  try {
    if (consumer != null) {
      consumer.accept(x);
    }
  }
  catch (Throwable t) {
    Exceptions.throwIfFatal(t);
    this.subscription.cancel();
    onError(t);
  }
} 
```

`consumer.accept(x)` 会使数据打印在命令行中。

整个流程的解释到这里基本结束。

# 七、总结

接下来我们用一个图来回顾一下整体的流程。

![](https://upload-images.jianshu.io/upload_images/3108769-cb4b773a4eb5b738.jpg)

Reactor Core

# 参考

需要感谢下面这篇文章，帮助我很好理解了 Project Reactor 的核心处理流程。

由表及里学 ProjectReactor：[http://blog.yannxia.top/2018/06/26/java/spring/projectreactor/](http://blog.yannxia.top/2018/06/26/java/spring/projectreactor/)

![](https://upload-images.jianshu.io/upload_images/3108769-de89907f87e56d19.jpg)

我的技术公众号 “编走编想” 
 [https://www.jianshu.com/p/df395eb28f69](https://www.jianshu.com/p/df395eb28f69)
