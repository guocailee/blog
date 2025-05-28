---
{"dg-publish":true,"permalink":"/Program/Kotlin/Kotlin Coroutine 初探/","noteIcon":"","created":"2024-05-22T16:17:54.152+08:00"}
---


## 一、前言

Kotlin 是近两年兴起的一门编程语言，最近一年的发展速度很快。在 2017 年，Google 宣布 Kotlin 成为 Android 的官方开发语言。同时，作为 Java 服务器端开发领域的带头大哥之一的 Spring 也对 Kotlin 提供了全面的支持。

在 Kotlin 众多的特性中，在 1.1 中作为实验特性加入的 Coroutine（协程，可以简单看作是轻量级线程）技术是非常值得关注的。因为大家都知道，近些年大红大紫的 Go 语言的流行同其协程特性就有着很大的关系，而这也让 Java 的线程技术看上去显得很落后。因为传统的 Java 应用提高并发的主要方式就是开启更多的线程，但线程太多会导致资源的浪费，太低又容易导致并发不够。虽然如 Netty 这样的技术能解决 IO 密集场景下的并发问题，但是使用门槛比较高，学习曲线比较陡，不易于大面积使用。

而 Kotlin 的 Coroutine 特性为 JVM 上的高并发应用开发带了一个非常有希望的新选择，因此是十分值得关注的。本篇文章将向大家介绍 Kotlin Coroutine 是什么、如何使用、以及能解决哪些问题等内容。

_本文的内容基于 Kotlin Coroutine 0.21.2 版本。_

## 二、Kotlin Coroutine 是什么

Kotlin Coroutine 是 Kotlin 为了实现更好的异步和并发程序所提供的一个特性，从 1.1 版本开始引入。不同于其它的编程语言，Kotlin 将其 Coroutine 特性的大部分内容作为了一个扩展库：`kolinx.coroutines`，语言层面仅提供了有限的支持。

例如，C#、ECMAScript 等语言都将 `async`、`await` 做为了关键字，但在 Kotlin 中，这些只是普通的方法。在 Kotlin 中，和 Coroutine 相关的关键字仅有 `suspend`。

不仅如此，Kotlin 还将 Coroutine 库做了进一步拆分，分成了核心模块 `kotlinx-coroutines-core` 和与其它异步、并发技术集成的模块，如：`kotlinx-coroutines-jdk8`、`kotlinx-coroutines-reactive`、`kotlinx-coroutines-rx1/rx2`、`kotlinx-coroutines-reactor` 等。

目前（Kotlin 1.2），Kotlin Coroutine 还只是一个实验特性。所以，Kotlin Coroutine 相关类的包名中包含了 `experimental` 的字样。但 Kotlin 将要正式包含 Coroutine 特性基本上是板上钉钉的事了。按照目前的计划，Kotlin 1.3 将会正式包含 Coroutine 特性。目前 Coroutine 的整体设计和使用方式也早已确定，不会发生明显变化。

## 三、Kotlin Coroutine 的使用

接下来我们看看 Kotlin Coroutine 在不同场景中如何解决我们在异步和并发编程中所遇到的问题和痛点。

### 场景一：延迟执行

我们在开发的时候，经常遇到需要等待一段时间之后在执行某些语句的场景。这时，我们常用 Thread.sleep 实现：

```kotlin
    @Test
    fun delay_with_thread() {
        log.info("Hello")
        Thread.sleep(1000)
        log.info("World")
    }
```

这样做效率比较低，因为线程白白地浪费了一秒钟。如果这段代码调用量比较大，那就很浪费资源了。

我们可以改进一下，使用 ScheduledThreadPool：

```kotlin
@Test
fun delay_with_scheduler() {
    val scheduler = Executors.newScheduledThreadPool(1)
    log.info("Hello")
    scheduler.schedule({
        log.info("World")
    }, 1, TimeUnit.SECONDS)
    scheduler.shutdown()
    scheduler.awaitTermination(1, TimeUnit.SECONDS)
}
```

这样做虽然效率高了，但是缺点也很明显，那就是代码变得很不直观了。如果代码再复杂，那就更加不易理解了。

如果用 Kotlin Coroutine，该怎么写呢？

```kotlin
@Test
fun delay_with_coroutine() {
    runBlocking {
        log.info("Hello")
        delay(1000)
        log.info("World")
    }
}
```

是不是很简单，和第一个版本唯一的区别就是把 `Thread.sleep(1000)` 换成了 `delay(1000)`。并且，`delay(1000)` 并不会挂起当前线程，这样代码执行效率就高的多了。

### 场景二：Completable Future

Kotlin Coroutine 提供了与各种异步技术的集成，包括 JDK8 Completable Future、Google Guava 的 Listenable Future、Spring 的 Reactor、Netflix 的 RxJava 等，但不包括 JDK5 中的 Future。原因是传统的 Future 接口并没有提供任何回掉机制，所以 Kotlin Coroutine 无法与其集成。因此，本节主要介绍 Kotlin Coroutine 如何与 `CompletableFuture` 集成。

按照传统方式使用 `CompletableFuture` 需要调用 `thenApply`、`thenCompose`、`thenAccept` 这样的方法串联起异步调用：

```kotlin
val future = CompletableFuture.supplyAsync({ 1 })
future.thenApply { value -> "${value + 2}" }
        .thenAccept({ value ->
    log.info(value.toString())
})
```

Kotlin Coroutine 为 `CompletableFuture` 接口增加了 `await` 方法，可以将回调转换为传统的调用方式：

```kotlin
val future = CompletableFuture.supplyAsync({ 1 })
val value = future.await()
val result = value + 2
log.info(result.toString())
```

可见使用 Kotlin Coroutine 之后代码得到了明显简化。

### 场景三：反应式编程

接下来我们来看看 Kotlin Coroutine 是如何简化反应式编程的。

在 Spring 5 出现之后，开发人员可以在 Web 开发领域更容易地使用反应式编程，从而提高系统的并发性能和伸缩性。可是，虽然像 Spring Reactor 项目、Netflix RxJava 项目等反应式编程技术使得异步编程变得简单了许多，但是距离理想还是有一定距离。

接下来咱们就来看看现有的反应式编程技术存在的问题和 Kotlin Coroutine 是如何解决这些问题的。

#### 直接使用 Spring Reactor

下面这段代码的目的是根据人员 ID 查询在他上次登录之后，又有多少新消息。其中使用到了 Spring 5 的反应式编程特性，使用了 Reactor 的 API 和 Spring Data 中的 Reactive Repository。

```kotlin
@GetMapping("/reactive/{personId}")
fun getMessagesFor(@PathVariable personId: String): Mono<String> {
    return peopleRepository.findById(personId)
        .switchIfEmpty(Mono.error(NoSuchElementException()))
        .flatMap { person ->
            auditRepository.findByEmail(person.email)
                .flatMap { lastLogin ->
                    messageRepository.countByMessageDateGreaterThanAndEmail(lastLogin.eventDate, person.email)
                        .map { numberOfMessages ->
                            "Hello ${person.name}, you have $numberOfMessages messages since ${lastLogin.eventDate}"
                        }
                }
        }
}
```

看到上面这段代码之后，我想大部分人的直观感受就是 “好复杂”、“Callback Hell” 等。

等等，不是说好了 Reactive Stream 方式可以避免 Callback Hell 吗？为什么这里还是存在 Callback Hell。其实，像 RxJava、Reactor 这样的 Reactive Programming 框架，所能解决的 Callback Hell 问题的范围是有限的。一般来说，如果一系列的调用，每一步只依赖上一步的结果，那用 Reactive Stream 的方式可以完美的写成链式调用：

```kotlin
monoA.flatMap(valueA -> {
    returnMonoB(valueA);
}).flatMap(valueB -> {
    returnMonoC(valueB);
}).flatMap(valueC -> {
    returnMonoD(valueC);
});
```

上面代码中，`monoA` 中包含的值是 `valueA`，依次类推。

但问题是，现实中的业务需求哪里会这么简单和理想。以上面的查询新消息数的应用为例，`messageRepository.countByMessageDateGreaterThanAndEmail(lastLogin.eventDate, person.email)` 这一步依赖了上一步的结果 `lastLogin` 和上上步的结果 `person`。不满足我之前所说的 “每一步只依赖上一步的结果” 的条件，导致这个例子不太容易写成完美链式调用。

虽然通过一些小技巧可以对上面的代码进行一定程度的优化，但优化之后可读性还是不高。

#### 使用 Kotlin Coroutine

Spring 5 对 Kotlin 提供了完备的支持。同样，Kotlin 也增加了对 Spring 的支持。其中一个便是对 Spring Reactor 项目的支持。于是我们可以使用 Kotlin Coroutine 改造上面的代码：

```kotlin
    @GetMapping("/coroutine/{personId}")
    fun getNumberOfMessages(@PathVariable personId: String) = mono(Unconfined) {
        val person = peopleRepository.findById(personId).awaitFirstOrDefault(null)
                ?: throw NoSuchElementException("No person can be found by $personId")

        val lastLoginDate = auditRepository.findByEmail(person.email).awaitSingle().eventDate

        val numberOfMessages =
                messageRepository.countByMessageDateGreaterThanAndEmail(lastLoginDate, person.email).awaitSingle()

        "Hello ${person.name}, you have $numberOfMessages messages since $lastLoginDate"
    }
```

改造之后代码最明显的变化就是代码可读性提高了很多。代码的可读性对所有的软件系统都是十分重要，如果代码很难让人理解，那软件系统的维护、升级工作的成本就会很高。因此，Kotlin Coroutine 对异步编程的代码可读性的提升是非常有价值的。

**说明：如果查询结果为空，调用 `awaitSingle` 会导致程序抛出 `NoSuchElementException`，并无法直接通过 try...catch 捕获（只能通过 `Mono` 的错误处理回调方法处理，如 `doOnError`、`onErrorCosume` 等）。为了除了查询结果可能为空的情况，使用了 `awaitFirstOrDefault` 方法。**

## 四、解释

上面介绍了使用 Kotlin Coroutine 所带来的一些好处。接下来将对上面的代码和 Kotlin Coroutine 中的重要概念进行介绍。

### suspending 方法

用一句话概括 Kotlin Coroutine 的特点可以是 “以同步之名，行异步之实”。那这个“实” 是怎么行的？关键就是 suspending 方法。上面几个 Kotlin Coroutine 的例子出现了多个 suspending 方法：`delay`、`await`、`awaitSingle` 等。这些 suspending 方法能够使程序执行过程暂停，但又不挂起线程。从而可以让程序既高效，又易懂。

suspending 方法的声明很简单，只需在方法或 Lambda 定义前面加 `suspend` 关键字即可。下面以 `awaitSingle` 为例：

```kotlin
    public suspend fun <T> Publisher<T>.awaitSingle(): T = awaitOne(Mode.SINGLE)
```

suspending 方法声明容易，但 suspending 方法的使用却有限制，并不是在任何地方都可以调用 suspending 方法。suspending 方法只能两种地方被调用，一是在另一个 suspending 方法中，二是在 Coroutine Builder 中被调用。所以，我们接下来看看什么是 Coroutine Builder。

### Coroutine Builder

Coroutine Builder，顾名思义，是用来创建 Coroutine 的。对于 Coroutine Builder 究竟是如何创建 Coroutine 的，我放在后面的文章再讲。我们先来看看 Coroutine Builder 具体都有哪些，以及它们如何使用。

常见的 Coroutine Builder 有 `runBlocking`、`launch`、`async`，以及用于和 Spring Reactor 配合使用的 `mono`、`flux`。

简单来说，Coroutine Builder 就是一些方法，这些方法接受 suspending lambda 作为参数，并将其放入 Coroutine 中执行。一个完整的 Coroutine 调用的开始都是一个 Coroutine Builder。

简单说一下几个常见的 Coroutine Builder 的用法：

#### runBlocking

这个 Coroutine Builder 的作用是阻塞调用它的线程。例如，在上面 delay 的例子中，就使用了 runBlocking。

#### launch

这个 Coroutine Builder 会创建一个 Coroutine 并执行它，并返回一个 Job 对象，用于控制这个 Coroutine 的执行，但没有结果的返回。

例如，之前 delay 例子也可以这么写
  ```kotlin
    fun main(args: Array<String>) {
        launch { // launch new coroutine in background and continue
            delay(1000L) // non-blocking delay for 1 second (default time unit is ms)
            println("World!") // print after delay
        }
        println("Hello,") // main thread continues while coroutine is delayed
        Thread.sleep(2000L) // block main thread for 2 seconds to keep JVM alive
    }
```

（说明：因为 `launch` 不会挂起线程，所以需要使用 `Thread.sleep` 避免主线程提前退出）

`launch` 方法会返回一个 Job 对象：

```kotlin
@Test
fun delay_with_coroutine_launch() {
    runBlocking {
        log.info("Hello")
        val job = launch {
            // launch new coroutine and keep a reference to its Job
            delay(5000L)
            log.info("World")
        }
        job.cancel()
        job.join() // wait until child coroutine completes
    }
}
```

Job 对象提供了 `cancel()`、`join()` 等方法，用来控制 Job 的执行（因为 `join()` 方法也是一个 suspending 方法，所以外面有加了一层 `runBlocking`）

#### async

和 `launch` 类似，`async` 也可以用来启动一个 Coroutine。不同的是，`launch` 返回的 `Job`，其只能控制 Coroutine 的执行，但是不能得到任何返回结果。`async` 返回的是 `Deferred`，你可以通过调用其提供的 `await()` 方法得到 Coroutine 的运行结果：

```kotlin
@Test
fun delay_with_async() {
    log.info("Start to demo async")

    val one = async {
        delay(1000)
        1
    }

    val two = async {
        delay(2000)
        2
    }

    runBlocking { log.info("${one.await() + two.await()}") }
}
```

#### mono 和 flux

最后介绍的两个 Coroutine Builder 是 kotlinx-coroutines-reactor 所提供的，用于和 Spring 的 Reactor 项目集成。从上面的示例看，`mono` 与前几个 Coroutine Builder 有所不同。最明显的区别是在后面的括号了带了一个 `Unconfined`。简单说，这个 `Unconfined` 是一个 `CoroutineDispatcher`，用来限定使用什么线程来执行 Coroutine。

在 Github 的 Kotlin 项目的文档中，对 `CoroutineDispatcher` 有着详细的描述（链接在最后给出）。接下来我对文档里的内容做一些解释，方便大家理解。

所有的 Coroutine Builder 方法的第一参数都是 `CoroutineContext`。那为什么可以把 `CoroutineDispatcher` 作为参数传给 Coroutine Builder 呢？

原来 `CoroutineDispatcher` 实现了 `CoroutineContext.Element` 接口，而 `Element` 又是一个特殊的 `CoroutineContext`，其是只存放了一个元素的 `CoroutineContext`。所以，`CoroutineDispatcher` 也是一个 `CoroutineContext`。这个 `CoroutineContext` 仅包含一个元素，而这个元素就是 `CoroutineDispatcher` 自己。

当 Coroutine 执行的时候，Kotlin 会看当前 Coroutine 的 `CoroutineContext` 里面是否有 `CoroutineDispatcher`。如果有，则使用 `CoroutineDispatcher` 限定 Coroutine 所使用的线程。

当不给 Coroutine Builder 制定参数时，`launch`、 `async`，以及 `mono` 和 `flux` 默认使用的 `CoroutineDispatcher` 是 `CommonPool`，一个公共的线程池实现。`runBlocking` 默认使用的是 `BlockingEventLoop`。另一个常见的 `CoroutineDispatcher` 实现就是 `mono` 例子中的 `Unconfined`。

`Unconfined` 意思就是不限定。在第一个暂停点之前，Coroutine 的执行线程都是调用的线程。在第一个暂停点之后，用哪个线程执行就是由 suspending 方法决定了。

例如，在 “反应式编程” 这个示例中，`peopleRepository.findById(personId)` 的执行是使用的调用线程。之后的执行是使用 Mongo 异步客户端回调线程（其中的 Repository 基于 Mongo 异步客户端）。

## 五、小结

如今面对高并发应用开发场景，Java 传统的线程模型显得越来越力不从心。Java 社区也意识到了这个问题，于是出现了一批提供轻量级线程解决方案的项目，如 Quasar 项目、Alibaba JDK 的协程解决方案、Open JDK Project Loom 提案，也包括反应式编程技术。但这些方案都存在这样或那样的问题。

Kotlin Coroutine 的出现为解决 Java 高并发应用开发提供了新的选择，带来了新的希望。但我们也需要看到，Kotlin Coroutine 只是刚刚起步，还有很长的路要走。同时，Kotlin Coroutine 虽然在形式上简化了异步代码的开发，但也对使用者提出了相当的要求。如果对 Java 并发、NIO、反应式编程，以及 Kotlin 本身等技术缺乏足够的了解，那恐怕还是难以顺畅使用 Kotlin Coroutine 的。这可能也是 Java 程序开发难以摆脱的历史包袱。

本篇文章介绍了简要介绍了 Kotlin Coroutine 的概念和使用场景、使用 Kotlin Coroutine 的好处，以及一些关键概念。后续的文章将会给大家详细介绍 Kotlin Coroutine、其实现原理和 Kotlin Coroutine 同其它类似技术的比较。

**本文一些示例使用了日志，这里提醒大家需要注意在实际项目中要避免日志阻塞线程问题。避免线程阻塞是几乎所有高性能异步应用开发都需要注意的。**

## 附：参考

- [Coroutine context and dispatchers](https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html)
- Kotlin CoroutineDispatcher Javadoc：[https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-dispatcher/index.html](https://links.jianshu.com/go?to=https%3A%2F%2Fkotlin.github.io%2Fkotlinx.coroutines%2Fkotlinx-coroutines-core%2Fkotlinx.coroutines.experimental%2F-coroutine-dispatcher%2Findex.html)
- [Quasar 纤程技术文档](http://docs.paralleluniverse.co/quasar/)
- Alibaba JDK 协程：[http://www.infoq.com/cn/presentations/free-performance-lunch-alibaba-jdk-association/](https://links.jianshu.com/go?to=http%3A%2F%2Fwww.infoq.com%2Fcn%2Fpresentations%2Ffree-performance-lunch-alibaba-jdk-association%2F)
- Open JDK Project Loom 提案：[http://cr.openjdk.java.net/~rpressler/loom/Loom-Proposal.html](https://links.jianshu.com/go?to=http%3A%2F%2Fcr.openjdk.java.net%2F%7Erpressler%2Floom%2FLoom-Proposal.html)
  [https://www.jianshu.com/p/2d2e21941461](https://www.jianshu.com/p/2d2e21941461)
