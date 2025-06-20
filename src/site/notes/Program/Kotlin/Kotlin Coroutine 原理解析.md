---
{"dg-publish":true,"permalink":"/Program/Kotlin/Kotlin Coroutine 原理解析/","noteIcon":"","created":"2025-05-28T22:45:10.196+08:00"}
---


[[Program/Kotlin/Kotlin Coroutine 初探\|Kotlin Coroutine 初探]] 向大家介绍了 Kotlin Coroutine 的由来、重要概念、用法。为了能让消除大家对 Kotlin Coroutine 的疑惑，帮助大家更好地理解和使用 Kotlin Coroutine，本篇文章将向大家介绍在 Java 平台上 Kotlin Coroutine 的实现原理。

## 一、示例

我们先从一段代码示例开始，假设我们有如下一段代码：

```kotlin
fun postItem(item: Item): PostResult {
  val token = requestToken()
  val post = createPost(token, item)
  val postResult = processPost(post)
  return postResult
}
```

这段代码的含义我们不必深究，只需关注代码的形式。这段代码的形式是我们最为常见的，一个方法，调用若干子方法，最后返回结果。这种风格被称为 `Direct Style`，或 `Imperative Style`（命令式）。这种风格优点在于直观地反映了业务逻辑，但在执行效率方面存在问题。如果代码中包含 IO 密集型操作，因为 Direct Style 代码往往是线程同步执行，因此执行这段代码的线程就会被阻塞，导致效率不高。当这样的代码面对 IO 操作耗时较长，并发量较高的场景时，就会产生问题，进而影响整个系统的表现。

如果想让代码更加适合高并发、IO 密集的场景，就需要使用 Callback 风格的代码：

```kotlin
fun postItem(item: Item) {
  requestToken { token ->
	createPost(token, item) { post ->
	  processPost(post) { postResult ->
		handleResult(postResult)
	  }
	}
  }
}
```

但 Callback 风格代码的问题在于难看难写难调试。虽然提高了执行效率，但是大大降低了开发效率。这在面对复杂的业务场景是很严重的问题。理想的情况是能够用 Direct Style，编写出同 Callback 风格一样高效的代码。

而 Kotlin Coroutine 的出现为在 Java 平台上解决上述问提供了一个理想的方案，只需很小的改造，就能得到上面讲的理想结果。

▼ 示例 1：suspending 方法版本的 `postItem`（假设 `requestToken`、`createPost` 等方法也都是 suspending 方法）

```kotlin
suspend fun postItem(item: Item): PostResult {
  val token = requestToken()
  val post = createPost(token, item)
  val postResult = processPost(post)
  return postResult
}
```

从上面的示例可以看出，使用 Kotlin Coroutine，只需增加 `suspend` 关键字，就能达到同 Callback 风格相同的效率。

关于 Kotlin Coroutine 的使用，上一篇文章《Kotlin Coroutine 初探》已经有比较详细的介绍，不再赘述。今天就来谈谈 Kotlin Coroutine 是如何实现的，原理是什么。

## 二、原理

`suspending` 方法是使用 `Kotlin Coroutine` 的主要形式。suspending 方法的实现依赖于各种提供 Callback 机制的技术，如 JDK8 的 CompletableFuture、Google Guava 的 ListenableFuture、Spring Reactor、Netflix RxJava 等。这也是为什么只有这些技术才能和 Kotlin Coroutine 集成。

接下来解释 Kotlin Coroutine 是如何基于这些技术实现没有线程阻塞的执行暂停机制。这需要从 Kotlin Coroutine 的多个概念和原理说起：

1. suspending 方法与 Continuation
2. CPS 转换与 Switch 状态机
3. suspendCoroutine 方法
4. CoroutineBuilder 方法

## （一）Suspending 方法变形记

suspending 方法的定义非常简单，只需在普通方法前面加上 `suspend` 关键字即可。但是 Java 平台并没有 `suspend` 关键字，显然也没有 suspending 机制，那 suspending 方法是如何运行的呢？

原来 Kotlin 编译器会对 suspending 方法做特殊处理，对代码进行转换，从而实现 suspending 机制。

那 Kotlin 编译器做了哪些处理？简单说，主要做了下面这三项处理：

- 处理一：增加 `Continuation` 类型入参，返回值变为 Object
- 处理二：生成 `Continuation` 类型的匿名内部类
- 处理三：对 suspending 方法的调用变为 switch 形式的状态机

接下来详细介绍一下这三项处理

先来看一下示例 1 中 suspending 方法编译之后的样子，让大家有一个总体的印象（为方便演示，不使用字节码）

▼ 示例 2：suspending 版本 postItem 方法编译后的样子

```javascript
  switch (sm.label) {
  	case 0:
  	  sm.item = item
  	  sm.label = 1
  	  return requestToken(sm)
  	case 1:
  	  val item = sm.item
  	  val token = sm.result as Token
  	  sm.label = 2
  	  return createPost(token, item, sm)
  	case 2:
  	  val post = sm.result as Post
  	  sm.label = 3
  	  return processPost(post, sm)
  	case 3:
  	  return sm.result as PostResult
  }
```

### 1. `Continuation`方法参数和匿名内部类

从上面的代码可以看出第一、二项提到的变化。

▼ suspending 方法编译之后增加 `Continuation` 类型参数

```kotlin
fun postItem(item: Item, cont: Continuation): Any?
```

▼ suspending 方法编译之后增加 `Continuation` 类型的匿名内部类

```kotlin
val sm = cont as? ThisSM ?: object : ThisSM {
  fun resume(…) {
    postItem(null, this)
  }
}
```

这两项都提到一个概念 —— Continuation，所以接下来介绍一下。

Continuation 这个名字来源于 CPS（Continuation-Passing-Style）。CPS 指的是一种编程风格。CPS 这个名字看上去很酷炫，但说白了就是 Callback 风格。Continuation 直译是连续体，意思就是后续的部分。对于 `requestToken` 方法来说，Continuation 就是 `createPost` 和 `processPost` 方法。常见的 CPS 中，Continuation 部分会被放在回调接口中实现。

在 Kotlin Coroutine 中，Continuation 还有一个更加具体的含义 —— `Continuation` 接口。先来看看它的接口定义：

```kotlin
    public interface Continuation<in T> {
      public val context: CoroutineContext
      public fun resume(value: T)
      public fun resumeWithException(exception: Throwable)
    }
```

从上面的代码可以看出，`Continuation` 定义了一个回调接口。`resume` 方法用来恢复暂停的 Coroutine 的执行。

如何恢复暂停的 Coroutine 的执行？从上面的示例代码可以看到，`postItem` 方法对应的 `Continuation` 类型的匿名内部类的 `resume` 方法会去回调 `postItem` 方法自己（但入参发生了变化，后面会解释）。并且，在其调用的 suspending 方法的调用中会传递这个 `Continuation`，后续方法可以通过 `Continuation` 重新回调 `postItem` 方法。

**小结：每个 suspending 方法编译后会增加一个 `Continuation` 类型的参数。每个 suspending 方法都有一个回调自己的 `Continuation` 实现类，并且这个类会被传递给这个 suspending 方法所调用的其它 suspending 方法，这些子方法可以通过 `Continuation` 回调父方法以恢复暂停的程序。**

到这里会产生几个问题：

1. 暂停是什么？它是如何发生的？
2. `Continuation` 回调接口是如何以及何时被调用的？

对于这些问题，后续的章节会给出答案。

### 2. Switch 状态机

从示例 2 中的代码可以看到，suspending 方法编译之后，会将原来的方法体变为一个由 switch 语句构成的状态机：

```kotlin
    switch (sm.label) {
      case 0:
        sm.item = item
        sm.label = 1
        return requestToken(sm)
      case 1:
        val item = sm.item
        val token = sm.result as Token
        sm.label = 2
        return createPost(token, item, sm)
      case 2:
        val post = sm.result as Post
        sm.label = 3
        return processPost(post, sm)
      case 3:
        return sm.result as PostResult
    }
```

这么做的原因是什么呢？前面说到了，Kotlin Coroutine 的运行依赖于各种 Callback 机制。也就是说，一个 suspending 方法调用到最后，其实就是注册一个回调。方法的执行结果就是通过这个回调来处理。当回调注册完毕之后，当前的线程就没有必要再等待下去了。接下来就是方法返回，结束调用。所以，大家能看到这个 switch 语句中，每个 case 都会返回。

**所以，对于上一节中的问题 “暂停是什么？它是如何发生的？” 答案就是方法返回了。**

是不是很简单呢。但方法返回只是线程执行层面结束了，整个 suspending 方法的功能还没有完成，后续的方法还是需要调用，执行结果还是需要返回。这些工作都是如何实现呢？

在上面的示例代码中，**每个 case 都有调用 `sm.label = N` （除了最后一个 case）。这里的 N 表示的是当前 case 的下一个 case（下一步）所对应的 case 的值。这个值被记录在 `sm` 实例中，然后 `sm` 会做为 `Continuation` 类型的参数传递个当前 case 中的子 suspending 方法。**

**子 suspending 方法（本例中为 `requestToken`、`createPost` 等方法）会将 `sm` 设置进回调接口。当回调发生，并且子 suspending 方法完成执行时，`sm` 会回调它所对应的 suspending 方法（本例中为 `postItem`），并根据 `label` 中的值执行对应 case 中的语句。从而实现程序执行的恢复。**

上面这几段内容解释了 suspending 方法是如何暂停的，以及又是如何恢复的问题。

接下来逐行解释示例 2 的代码，以帮助大家更全面理解：

**case 0**

首先，在 case 0 中，通过语句 `sm.item = item`，将入参 `item` 保存在状态机实例 `sm` （类型为 ThisSM，实现 `Continuation` 接口）中，以使后续调用能够通过 `Continuaton` 获得入参。

然后通过 `sm.label = 1` 设置下一步的状态。从后续的代码中也可以看到，在每个 case 中，都会将 `sm.label` 设置为下一个 case 的值，这样，在通过 `Continuation` （就是 `sm`）回调时，就知道下一步要调用哪个方法了。

接下来就是调用 `requestToken` 方法，可以看到，在编译之后，`requestToken` 多了一个 `Continuation` 类型的入参。

**case 1**

当 `requestToken` 设置的回调被触发时（对应着 Direct Style 中方法返回），通过 `sm` 回调 `postItem` 方法。此时，`label=1`，因此执行 case 1。

通过调用 `val item = sm.item`，从 `sm` 中获取参数 `item`。

通过调用 `val token = sm.result as Token` 获取 `requestToken` 方法的返回值 `token`。

通过调用 `sm.label = 2` 将 label 设置为下一步的 case。

调用 `createPost(token, item, sm)`

**case 2**

同 case 1 的内容类似，略。

**case 3**

`return sm.result as PostResult` 从 `Continuation` 中获得返回值。

### 3. Continuation 的父子调用

上一节解释了 suspending 方法是如何暂停的，以及又是如何恢复的问题。但有一个细节没有解释：**一个 suspending 方法对应的 `Continuation` 是如何知道它是应该回调当前的 suspending 方法，还是上一级的 suspending 方法呢？**

要解释这个问题，需要讲解一个上面示例隐藏掉的细节。在一个 suspending 方法创建它所对应的 `Continuation` 时，会将从入参传入的 `Continuation` 作为父 `Continuation` 引入新创建的 `Continuation`。 因为每个 suspending 方法所创建的 `Continuation` 是基于 `CoroutineImpl` 的，所以看一下 `CoroutineImpl` 的源代码：

```kotlin
    abstract class CoroutineImpl(
        arity: Int,
        @JvmField
        protected var completion: Continuation<Any?>?
    ) : Lambda(arity), Continuation<Any?> {
      override fun resume(value: Any?) {
        processBareContinuationResume(completion!!) {
          doResume(value, null)
        }
      }
    }

    fun processBareContinuationResume(completion: Continuation<*>, block: () -> Any?) {
      try {
        val result = block()
        if (result !== COROUTINE_SUSPENDED) {
          @Suppress("UNCHECKED_CAST")
          (completion as Continuation<Any?>).resume(result)
        }
      } catch (t: Throwable) {
        completion.resumeWithException(t)
      }
    }
```

`CoroutineImpl` 构造函数有一个 `Continuation` 类型的入参 `completion`，这个 `completion` 代表的是父 `Continuation`。调用 `resume` 方法是会先调用 `processBareContinuationResume`。`processBareContinuationResume` 的第一个入参是父 `Continuation`，第二个入参 `block` 就是 `doResume` 方法，也就是对当前 suspending 方法的调用。如果当前 suspending 方法的返回结果不是 `COROUTINE_SUSPENDED`，即执行成功时，就会通过调用 `completion.resume(result)` 的方式回调父 `Continuation`，并返回执行结果。

看一下流程图：

![](https://upload-images.jianshu.io/upload_images/3108769-730fe1e81c30a4d0..jpg)

Kotlin Coroutine Suspending 方法父子调用

### 4. 小结

Kotlin Coroutine suspending 方法在编译之后会发生显著变化：

1. 首先，suspending 方法增加一个 `Continuation` 类型的入参，用于实现回调。返回值变为 Object 类型，既可以表示真实的结果，也可表示 Coroutine 的执行状态。
2. 然后，编译器会为这个 suspending 方法生产一个类型为 `Continuation` 的匿名内部类（扩展 `CoroutineImpl`），用于对这个 suspending 方法自身的回调，并可以在这个 suspending 方法执行完毕之后，回调这个 suspending 方法上一级的父方法。
3. 最后，这个 suspending 方法如果调用其它 suspending 方法，会将这些调用转换为一个 switch 形式的状态机，每个 case 表示对一个 suspending 子方法的调用或最后的 return。同时，生成的 `Continuation` 匿名内部类会保存下一步需要调用的 suspending 方法的 label 值，表示应该执行 switch 中的哪个 case，从而串联起整个调用过程。

## （二）suspendCoroutine 方法

前面的内容解释了 suspending 方法是如何实现没有线程阻塞的执行暂停，这是介绍了 Kotlin Coroutine 主干部分 —— suspending 方法的实现原理。但方法调用有头有尾，suspending 方法调用结束在哪里呢？

因为前面说到了，Kotlin Coroutine 还是基于 Callback 机制。所以，suspending 方法调用到最后，就应当是将 Kotlin Coroutine 自己的回调接口 `Continuation` 注册到某种 Future 技术的回调接口中。

但在普通的 Suspending 方法中压根访问不到 `Continuation`，那该如何做呢？

方法就是通过一个特殊的 suspending 方法 —— `suspendCoroutine` 实现。`suspendCoroutine` 方法是 Kotlin 标准库的一部分，它可以在 `kotlin-stdlib` 模块中的 `CoroutinesLibrary.kt` 中被找到。

`suspendCoroutine` 方法的签名如下：

`````kotlin
    suspend fun <T> suspendCoroutine(crossinline block: (Continuation<T>) -> Unit): T
```

`suspendCoroutine` 的入参是一个名称为 `block` 的 Lambda。这个 Lambda 可以有一个类型为 `Continuation` 的入参。能够拿到 `Continuation`，就可以将其注册到某种 Future 机制中了。

看一个 Kotlin Coroutine 官方文档中的示例，演示了如何使用 `suspendCoroutine` 使 `CompletableFuture` 同 Kotlin Coroutine 集成：

````kotlin
    suspend fun <T> CompletableFuture<T>.await(): T =
        suspendCoroutine<T> { cont: Continuation<T> ->
          whenComplete { result, exception ->
            if (exception == null) // the future has been completed normally
              cont.resume(result)
            else // the future has completed with an exception
              cont.resumeWithException(exception)
          }
        }
```

_注意：上面的这段代码只是一个演示 `suspendCoroutine` 以及如何与 Future 技术集成的的示例。虽然原理相同，但真实的代码会更为复杂。_

从上面的代码可以看出，正是因为 `suspendCoroutine` 的入参 `block` Lambda 拥有一个 `Continuation` 类型的入参，使得可以使用 `suspendCoroutine` 方法与各种 Future 机制集成。

进一步观察 `suspendCoroutine` 的实现原理，`suspendCoroutine` 调用了 `suspendCoroutineOrReturn` 方法，但直接观察源码无法了解 `suspendCoroutineOrReture` 的实现：

    inline suspend fun <T> suspendCoroutineOrReturn(crossinline block: (Continuation<T>) -> Any?): T =
        throw NotImplementedError("Implementation is intrinsic")

`suspendCoroutineOrReturn` 只起到一个标记的作用，实现细节隐藏在了编译阶段。但它的实现方式又和普通的 suspending 方法不同，所以要定义一个特殊方法，以区别对待。

## （三）Coroutine Builder 方法

`suspendCoroutine` 方法可以看做是 Kotlin Coroutine 调用的终点，接下来要讨论的是 Kotlin Coroutine 调用的起点。因为 suspending 方法不能直接被普通方法调用。如果普通方法要调用 suspending 方法，就必须通过 Coroutine Builder。

Kotlin Coroutine 核心和扩展模块提供了多种 Coroutine Builder。这些 Coroutine Builder 有着不同的作用。例如，`runBlocking` 能够挂起当前线程、`mono` 可以将 Coroutine 转换为 Spring Reactor Project 中的 `Mono` 类型。这些不同 Coroutine Builder 的作用不在本文的范围（后续文章将会介绍），而是介绍这些 Coroutine Builder 公共的部分 —— suspending Lambda。

以 `mono` 为例：

```kotlin
fun <T> mono(
	context: CoroutineContext = DefaultDispatcher,
	parent: Job? = null,
	block: suspend CoroutineScope.() -> T?
)
`````

最后一个入参 `block` 是一个 suspending Lambda。同 suspending 方法一样，suspending Lambda 在编译之后，其主体部分也会被转换为 switch 形式的状态机。不同于对 suspending 方法的处理，编译器并没有为 suspending Lambda 生产类型为 `Continuation` 的匿名内部类，而是 Lambda 自己作为 `Continuation` 实现（每个 Lambda 在编译之后会生成一个匿名内部类）。

除了对 suspending Lambda 的处理以外，Coroutine Builder 另外一个比较通用的处理是通过调用 `createCoroutineUnchecked` 方法创建一个新的 Coroutine。

# 三、总结

到这里 Kotlin Coroutine 的主要的实现原理已经介绍完毕。但还有很多其它的细节，大家可以 Kotlin Coroutine 官方文档（地址：[https://github.com/Kotlin/kotlin-coroutines/blob/master/kotlin-coroutines-informal.md#implementation-details](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2FKotlin%2Fkotlin-coroutines%2Fblob%2Fmaster%2Fkotlin-coroutines-informal.md%23implementation-details)）和视频《KotlinConf 2017 - Deep Dives into Coroutines on JVM》（地址：[https://www.youtube.com/watch?v=YrrUCSi72E8](https://link.jianshu.com?t=https%3A%2F%2Fwww.youtube.com%2Fwatch%3Fv%3DYrrUCSi72E8)）了解。

从业内的发展趋势看，反应式编程是 Java 社区应对高并发场景的主要选择，但直接使用反应式编程技术（Spring Reactor、RxJava）还是有很多不方便的地方（在上一篇文章《Kotlin Coroutine 初探》中已经介绍过）。所以 Kotlin Coroutine 的出现及时有效地解决了这些问题。

因此，可以预见，Kotlin Coroutine 将会越来越多地出现在 Java 服务器端和 Android 等领域的应用中。所以，理解 Kotlin Coroutine 实现原理很有意义。

另外，Coroutine 并不是 Kotlin 的发明，很多其它语言都有 Coroutine 这个概念，比如 LISP、Python、Javascript 等。Kotlin 的实现原理也借鉴了很多其它的语言。所以，理解 Kotlin Coroutine 的原理，也能够帮助理解其它语言的 Coroutine 技术的底层原理。

本篇介绍 Kotlin Coroutine 实现原理的文章就到这里。后续 Kotlin Coroutine 相关的文章将会介绍 Kotlin Coroutine 与 Spring Reactor 项目的整合、Kotlin Coroutine 与 Quasar、Alibaba JDK 等技术方案的对比，等等。尽请关注。
