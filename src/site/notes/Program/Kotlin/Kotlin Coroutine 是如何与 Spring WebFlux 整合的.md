---
{"dg-publish":true,"permalink":"/Program/Kotlin/Kotlin Coroutine 是如何与 Spring WebFlux 整合的/","noteIcon":"","created":"2024-05-22T16:17:54.153+08:00"}
---


## 一、前言

上篇文章介绍了 Kotlin Coroutine 的实现原理。因为篇幅所限，并未介绍 Kotlin Coroutine 具体是如何与其它异步编程技术整合的。本文将向大家介绍 Kotlin Coroutine 是如何与 Spring Reactor 整合。

然本文的标题是关于 Kotlin Coroutine 与 Spring WebFlux 的，但其实讲的是 Kotlin Coroutine 是如何与 Spring Reactor 整合的。因为 Spring Reactor 是 Spring WebFlux 的基础，所以不管起哪个标题，内容都是类似的

为 Spring WebFlux 这个名字更容易吸引人，所以本文便做了回标题党。

Kotlin Coroutine 与 Spring Reactor 的整合主要是通过 `kotlinx-coroutines-reactive` 和 `kotlinx-coroutines-reactor` 实现的。本文提到的源码都能从这两个模块中找到。

## 二、示例

本文将继续使用《[[Program/Kotlin/Kotlin Coroutine 初探\|Kotlin Coroutine 初探]]》一文中的在 Spring WebFlux 中使用 Kotlin Coroutine 的示例：

```kotlin
@GetMapping("/coroutine/{personId}")
fun getNumberOfMessages(@PathVariable personId: String) = mono(Unconfined) { // ①
    val person = peopleRepository.findById(personId).awaitFirstOrDefault(null) // ②
            ?: throw NoSuchElementException("No person can be found by $personId")

    // ②
    val lastLoginDate = auditRepository.findByEmail(person.email).awaitSingle().eventDate

    val numberOfMessages =
            messageRepository.countByMessageDateGreaterThanAndEmail(lastLoginDate, person.email).awaitSingle() // ②

    "Hello ${person.name}, you have $numberOfMessages messages since $lastLoginDate"
}
```

本文将重点介绍上面代码标注的 ①、② 两点。

第 ① 点：`mono(Unconfined) { ... }` 的实现原理

第 ② 点：`awaitXXX` 方法的实现原理

理解了上面两点，就能理解 `Kotlin Coroutine` 是如何与 `Spring Reactor` 整合的了。

## 三、Spring Reactor 相关知识

本节会介绍一下后面会涉及到的 Spring Reactor 的概念和实现细节方面的内容。之所以将 Spring Reactor 的部分内容单拎出来，是因为预先了解 Spring Reactor 的部分内容对理解后面的内容非常重要。

### 1. Publisher/Mono/Flux

Reactive Streams 是 Spring、Netflix 等公司提出的一个反应式编程的一个规范。这个规范定义了必要的接口和对实现的要求。`Publisher` 是其中一个重要的接口：

```kotlin
public interface Publisher<T> {
    public void subscribe(Subscriber<? super T> s);
}
```

`Publisher` 顾名思义，发布者，等同于很多技术中的 `Observable`。`subscribe` 方法实际应该被看做是被订阅，即 `aPublisher.subscribe(aSubscriber)` 应该理解为 `aPublisher` 被 `aSubscriber` 订阅。

但不同于之前同样有订阅机制的技术，比方说一些消息队列。之前的订阅机制中，订阅和消息的发布是两个独立的环节。而 `Publisher.subscribe(Subscriber)` 方法，如果被调用，则会使这个 `Publisher` 开始发布消息。

`Mono` 和 `Flux` 是在实现了 Reactive Streams 规范的 Spring Reactor 中的两个类。这两个类实现了 `Publisher` 接口。不同于类似的 RxJava 2，其只有一个 `Publisher` 的实现类。Spring Reactor 中两个 `Publisher` 实现类，分别代表单个元素和多个元素两种场景。

在使用了 Spring WebFlux（基于 Spring Reactor 的新一代 Web 框架）的项目中，一个被 `@RequestMapping` 标注的方法需要返回 `Mono` 或 `Flux`。然后，当一个请求根据映射配置被转发到这个方法上时，一个 `Mono` 或 `Flux` 对象会根据这个方法的定义创建出一个 `Mono`。但这时真正的请求处理并未开始，方法返回的只是一个处理的步骤定义。

当 Spring WebFlux 框架得到这个方法返回的 `Mono` 或 `Flux` 之后，会调用它们的 `subscribe(Subcriber)` 方法。此时，真正的请求处理便开始了。

后面的内容为了简便，会省略 `Flux`，仅会提到 `Mono`，但两者的原理基本类似。

### 2. Mono.create(ConsumerltMonoSink) 方法

Spring WebFlux 和传统 Spring MVC 最大的不同就是要求方法返回 `Mono` 或 `Flux`。当 Spring WebFlux 与 Kotlin Coroutine 整合后，我们需要将 Coroutine 转换成一个 `Mono`（或者 `Flux`，后面将省略 `Flux`）。

如何做呢？Kotlin Coroutine 使用的是 `Mono.create(Consumer<MonoSink<T>>)` 方法。

从方法签名看，`Mono.create` 方法涉及到最主要的接口是 `MonoSink`（此处不解释 `Consumer` 接口）。

`MonoSink` 是什么呢？其 API 文档是这么解释的：

> Wrapper API around an actual downstream Subscriber for emitting nothing, a single value or an error (mutually exclusive).

简单理解就是对后续 Subscriber 的封装。

可能有些同学对 Sink 这个词有些陌生，我起初也是这种感觉。但是对于一些做过流处理相关开发的同学，这个词应该不陌生。原因是 Sink 这个词经常出现在流处理相关技术中（比如 Flink、Flume）。在 Spring Cloud Stream 中，也能看到这个词。同样，Spring Reactor 是对 Reactive Streams 规范的实现，也可以看做是另一种形式的流技术，所以，出现 Sink 这个词也不足为奇了。

**因为 `MonoSink` 是对后续 Subscriber 的封装，所以可以利用 `MonoSink` 向后续的 Subscriber 输出一些东西的。在 Kotlin Coroutine 与 Spring Reactor 整合的过程中，Kotlin Coroutine 将开启一个 Coroutine，并将执行结果通过 `MonoSink` 输出给 Subscriber。**

在 Spring WebFlux 应用中，Subscriber 会将 Mono（或 Flux）以 HTTP 数据的形式输出。

这样就完成了 Kotlin Coroutine 向 Mono 转换的主要工作。更多细节将在下面的内容介绍。

## 四、整合的两个关键点

接下来将向大家介绍 Kotlin Coroutine 与 Spring Reactor 整合的两个关键点：`mono` 方法和 `await` 系列方法。

### 1. mono 方法

`mono` 方法连接了 Spring Reactor 环境与 Kotlin Coroutine 环境，可以看做是一个将 Kotlin Coroutine 装换为 Spring Reactor `Mono` 的工厂方法。我们先来看 `mono` 方法的源码：

```kotlin
fun <T> mono(
    context: CoroutineContext = DefaultDispatcher,
    parent: Job? = null,
    block: suspend CoroutineScope.() -> T?
): Mono<T> = Mono.create { sink ->
    // 创建一个新的 Coroutine Context
    val newContext = newCoroutineContext(context, parent)
    // 创建一个新的 MonoCoroutine，MonoCoroutine 会实现 Disposable 接口
    val coroutine = MonoCoroutine(newContext, sink)
    sink.onDispose(coroutine)
    coroutine.start(CoroutineStart.DEFAULT, coroutine, block)
}
```

`mono` 方法最主要的部分都集中在对 `Mono.create` 方法的调用。这也是为什么在前面的部分着重介绍 `Mono.create` 方法和 `MonoSink` 接口的原因。

`Mono.create` 会创建出一个 `Mono` 对象，当这个 `Mono` 对象的 `subscribe` 方法被执行的时候。传入 `Mono.create` 的 `Consumer` 就会被调用。此时下面的代码就会被执行：

```kotlin
{ sink ->
    // 创建一个新的 Coroutine Context
    val newContext = newCoroutineContext(context, parent)
    // 创建一个新的 MonoCoroutine，MonoCoroutine 会实现 Disposable 接口，用于关闭和异常处理。
    val coroutine = MonoCoroutine(newContext, sink)
    sink.onDispose(coroutine)
    coroutine.start(CoroutineStart.DEFAULT, coroutine, block)
}
```

上面这段代码就是 `Consumer` 的 Lambda 形式。

最主要的部分是下面两行代码：

```kotlin
val coroutine = MonoCoroutine(newContext, sink)
coroutine.start(CoroutineStart.DEFAULT, coroutine, block)
```

首先，一个 Coroutine —— `MonoCoroutine` 被创建。同时，`sink` 对象被传入到这个 `MonoCoroutine` 中。

接下来，这个 Coroutine 被执行，接下来的重点变成了 `MonoCoroutine`：

```kotlin
private class MonoCoroutine<in T>(
    parentContext: CoroutineContext,
    private val sink: MonoSink<T>
) : AbstractCoroutine<T>(parentContext, true), Disposable {
    var disposed = false

    override fun onCompleted(value: T) {
        if (!disposed) {
            if (value == null) sink.success() else sink.success(value)
        }
    }

    override fun onCompletedExceptionally(exception: Throwable) {
        if (!disposed) sink.error(exception)
    }

    override fun dispose() {
        disposed = true
        cancel(cause = null)
    }

    override fun isDisposed(): Boolean = disposed
}
```

从上面的代码可以看到，`MonoCoroutine` 继承了 `AbstractCoroutine`，同时构造函数入参传入了 `MonoSink`。

`MonoCoroutine` 实现了两个在 `AbstractCoroutine` 声明的重要方法：

- `onCompleted`
- `onCompletedExceptionally`

**这两个方法都是回调方法。从上面的代码可以明显看出。当 `MonoCoroutine` 在执行完毕之后，即这两个回调方法被调用时，会通过调用 `MonoSink` 将结果输出给 Subscriber。从而完成 Kotlin Coroutine 向 Mono 的转换工作。**

### 2. await 系列方法

接下来要介绍的是一系列以 `await` 开头的方法，比如示例中的 `awaitFirstOrDefault`、`awaitSingle`，等等。这些方法定义在 `kotlinx-coroutines-reactive` 模块中的 `Await.kt` 文件。

这些方法都是 suspending 方法，能够用命令式的代码风格获取 `Publisher` （`Mono` 或 `Flux`）中的结果。

因为上一篇文章《Kotlin Coroutine 原理解析》已经介绍了 suspending 方法的工作原理，所以这里就不重复了。本文只介绍 Kotlin Coroutine 是如何与 Reactive Streams 中的 `Publisher` 接口整合的。

await 系列方法可以看作是将 `Mono` 或 `Flux` 转换为 `Coroutine` 的方法。这些方法真正的实现集中在了 `awaitOne` 方法中。接下来我们看看 `awaitOne` 方法的源代码：

```kotlin
private suspend fun <T> Publisher<T>.awaitOne(
    mode: Mode,
    default: T? = null
): T = suspendCancellableCoroutine { cont ->
    subscribe(object : Subscriber<T> {
        private lateinit var subscription: Subscription
        private var value: T? = null
        private var seenValue = false

        override fun onSubscribe(sub: Subscription) {
            subscription = sub
            cont.invokeOnCompletion { sub.cancel() }
            sub.request(if (mode == Mode.FIRST) 1 else Long.MAX_VALUE)
        }

        override fun onNext(t: T) {
            when (mode) {
                Mode.FIRST, Mode.FIRST_OR_DEFAULT -> {
                    if (!seenValue) {
                        seenValue = true
                        cont.resume(t)
                        subscription.cancel()
                    }
                }
                Mode.LAST, Mode.SINGLE -> {
                    if (mode == Mode.SINGLE && seenValue) {
                        if (cont.isActive)
                            cont.resumeWithException(IllegalArgumentException("More that one onNext value for $mode"))
                        subscription.cancel()
                    } else {
                        value = t
                        seenValue = true
                    }
                }
            }
        }

        @Suppress("UNCHECKED_CAST")
        override fun onComplete() {
            if (seenValue) {
                if (cont.isActive) cont.resume(value as T)
                return
            }
            when {
                mode == Mode.FIRST_OR_DEFAULT -> {
                    cont.resume(default as T)
                }
                cont.isActive -> {
                    cont.resumeWithException(NoSuchElementException("No value received via onNext for $mode"))
                }
            }
        }

        override fun onError(e: Throwable) {
            cont.resumeWithException(e)
        }
    })
}
```

逐行解释一下关键代码：

**第 4 行：调用 `suspendCancellableCoroutine` 方法**

这个方法在上一篇文章中解释过了。它是一个特殊的 suspending 方法。不同于普通的 suspending 方法，通过这个方法，开发人员可以获得 `Continuation` 引用，用来与第三方技术进行集成。

**第 5 行：调用 `Publisher` 的 `subscribe` 方法**

因为 `awaitOne` 方法的定义是使用 Kotlin 扩展方法语法添加进了 `Publisher` 及其所有的子类，所以，可以在 `awaitOne` 方法中调用 `Publisher` 中的方法。

同时，调用 `subscribe` 方法的时候传入一个 `Subscriber` 接口的匿名内部类。后面大部分代码都是关于如何实现这个匿名内部类的。

在介绍这个 `Subscriber` 匿名内部类之前，需要先说明，对 `subscribe` 方法的调用会出发 `Publisher` 的执行。这也就是说当你调用一个如 `aMono.awaitSingle()` 方法的时候，就会使 `aMono` 开始执行。

但是，结合上面一节的内容，这一切的发生，需要当 Spring WebFlux 去调用如本例中 `getNumberOfMessages` 所返回的 `Mono` 的 `subscribe` 方法才会开始。所以，整个执行过程和一个普通的 Spring WebFlux 方法并没有大的区别。

**第 21 行：调用 `cont.resume(t)`**

这行调用发生在 `Subscriber` 的 `onNext` 方法中。在反应式编程体系中，`Publisher` 的每个结果都会通过回调 `onNext` 方法通知给 `Subscriber`。

在 `onNext` 方法里面，当获取到 `Publisher` 的结果之后，需要将结果传递给 `Continuation`。方法就是通过 `Continuation` 的 `resume` 方法。通过这种方法，`Publisher` 的结果便传递给了 Kotlin Coroutine。

## 五、总结

从上面的内容看，Kotlin Coroutine 与 Spring Reactor 的整合的原理并不复杂。主要是实现两个方向的转换：Kotlin Coroutine 向 Mono 的转换和 Mono 向 Kotlin Coroutine 的转换。

Kotlin Coroutine 向 Mono 的转换是通过 `Mono.create` 方法以及 `MonoSink` 接口实现的。Kotlin Coroutine 通过 `MonoSink` 接口，将执行结果输出给 `Subscriber`。

Mono 向 Kotlin Coroutine 的转换是通过使用 `suspendCancellableCoroutine` 方法获取到 `Continuation` 引用。再通过调用 `Publisher.subscribe` 方法，传入一个自定义的 `Subscriber`。通过 `Subscriber.onNext` 方法获取到 `Publisher` 的执行结果，并将这个执行结果传递给 `Continuation`。从而是 Kotlin Coroutine 获得了 Mono 的执行结果，完成了转换过程。

接下来，Kotlin Coroutine 系列的文章将向大家介绍 JVM 领域其它的协程技术（Quasar Fiber、AliJDK 协程等）与 Kotlin Coroutine 技术的对比。

# 附：名词解释

为方便大家理解，先向大家介绍本文将会涉及的名词及其含义：

1. **Reactive Streams：** 一个由多个技术社区共同提出的反应式编程方面的规范
2. **Spring Reactor：** Spring 社区提出的反应式编程解决方案，实现 Reactive Streams 规范。
3. **Spring WebFlux：** Spring 5 提供了反应式的 Web 开发解决方案，以 Spring Reactor 为基础。用法同 Spring MVC 类似。
4. **Publisher：** 在本文中指 Reactive Streams 中的一个重要接口。在其它技术中，也被称为 Observable。Publisher 中有一个重要方法 `subscribe`。Subscriber 可以通过这个方法订阅一个 Publisher，并使 Publisher 开始执行。
5. **Mono/Flux：** Spring Reactor 中对 Publisher 接口的实现，分别代表一个元素和多个元素两种场景。
6. **Continuation：** 异步编程中的一个概念，可以简单理解为 Callback。在 Kotlin Coroutine 中，Continuation 也表示一个具体的回调接口。
7. **Coroutine：** 协程。在 Kotlin 中，有很多以 Coroutine 命名的类，比如 `CoroutineImpl`、`AbstractCoroutine`。可以简单理解为 Continuation 是一个概念、规范，而 Coroutine 是一种实现机制。
