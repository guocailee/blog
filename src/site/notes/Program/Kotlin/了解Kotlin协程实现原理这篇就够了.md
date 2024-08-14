---
{"dg-publish":true,"permalink":"/Program/Kotlin/了解Kotlin协程实现原理这篇就够了/","noteIcon":""}
---

> 简单来说 **Kotlin协程** 一种异步编程的同步顺序写法，它跟线程是两个不同的概念，所以不要被 **协程** 两字弄混淆

来看一下一个具体的场景：我们进行网络请求得到数据后处理数据，代码如下

```kotlin
 fun setUpUI(){
        val data = requestData()
        doSomethingElse()
        processData(data)
    }

    fun requestData(): String {
        Thread.sleep(2000)
        return "Ui Data"
    }

    fun doSomethingElse(){
        println("doSomethingElse")
    }

    fun processData(data: String) {
        println("updateUI$data")
    } 
```

这是一段同步式代码，通常情况 `requestData()` 是一段耗时操作并不能在主线程执行，不然就会造成主线程阻塞，所以我们把它改写成异步的方式。

### Callback回调

第一种就是我们普遍使用的方式： **异步回调**，把耗时操作之后的逻辑封装成一个回调，耗时操作放在另外一个线程执行

```kotlin
 fun setUpUI(){
        requestDataAsync{it -> processData(it)}
        doSomethingElse()
    }
    
    fun requestDataAsync(callback: (resultData: String) -> Unit) {
        Thread { callback(requestData()) }.start()
    } 
```

函数代码块执行图如下：

![](https://ethanhua.github.io/img/callback_flow.jpg)

虽然异步回调能很好的处理异步问题，但是有两点坏处：

*   将一条控制流分割成两条控制流，**增加代码复杂性**
*   如果嵌套层级过多，会造成**回调地狱**，同时控制流数量也可能呈指数上升

### Future

`Java 5`后并发包里面引入了`Future`，一个`Future`代表一个异步运算未来的值，于是上面的逻辑可以这样写：

```kotlin
 fun setUpUI(){
        val future = requestDataAsync()
        doSomethingElse()
        processData(future.get())
    }

    fun requestDataAsync():Future<String>{
        val futureTask = FutureTask(Callable<String> { requestData() })
        Thread(futureTask).start()
        return futureTask
    } 
```

函数代码块执行逻辑如下：

![](https://ethanhua.github.io/img/future_flow.jpg)

虽然这样可以解决控制流分叉（把 `requestDataAsync` 看做一个整体，内部隐藏了异步的细节，从外部来看，执行流就只有主线程的执行流了）和嵌套地狱的问题，但是主线程还是会阻塞。

> 在`future.get()`这一步如果 **线程A** 的任务 `requestData` 还没有执行完返回结果，则主线程就会调用`LockSupport.park()`一直阻塞在此处，直到 `requestData()` 返回结果调用监听此结果的所有线程`unpark()`恢复执行。

由此可见 `Future` 虽然解决了`callback`的问题，但是引入了新的问题，主线程在调用`future.get()`的时候会阻塞，显然这样我们是不能接受的。

### [](#Reactive "Reactive")Reactive

我们来看另外一种异步编程方式：Java 8 中的 `CompleteableFuture` 和 `RxKotlin`，这两者都是同一种风格，我们用`RxKotlin`举例说明：

```
fun setUpUI(){
        Single.fromCallable<String> { requestData() }
              .subscribe { t -> processData(t!!) }
        doSomethingElse()
} 
```

这种方式的函数代码块执行逻辑流同第一种`callback`方式。

Rx风格可以有效解决嵌套回调的问题，但是没有解决控制流分叉的问题，`RxJava`是一个优秀的库，除了解决嵌套回调外，另外拥有丰富的操作符，在处理复杂的事件流方面有着强大的能力，但是有没有更简洁更符合人类思维的写法呢？就如同写同步逻辑样，**koltin协程**为我们提供了可能。

### kotlin协程

**kotlin协程** 是一种异步编程的顺序写法，将上诉例子中的代码可以改写成下面这样（为了表达出主线程不会阻塞，加了两行代码）：

```kotlin
 fun setUpUI(){
        GlobalScope.launch(Main) { 
            val dataDeferred  = requestDataAsync()
            doSomethingElse()
            val data = dataDeferred.await()
            processData(data)
        }
        Thread.sleep(1000)
        doSomethingElse2()
    }

    fun requestDataAsync():Deferred<String>{
        
        return GlobalScope.async { 
            requestData()
        }
    }  

    fun doSomethingElse2(){
        println("doSomethingElse2")
    } 
```

这种写法有点类似于 **Future**，`Deferred.await()`方法如同`Future.get()`方法，但不同的是 `await()` 不会阻塞主线程，只是会挂起协程，当`requestData()`执行完返回结果后，`processData()`的逻辑会恢复执行，在此时间段，主线程可以去执行其他的逻辑流，并不会阻塞在此处，比如此处会去执行函数 `doSomethingElse2()`，逻辑执行流如下：

![](https://ethanhua.github.io/img/coroutine_flow.jpg)

由于协程不会阻塞主线程，所以主线程执行顺序有两种可能，如果在调用`dataDeferred.await()`之后 1s (此处使用`Thread.sleep(1000)`)后还没有返回结果就执行`doSomethingElse2()`, 反之先执行 `processData()`。

[](#Kotlin协程代码实现细节 "Kotlin协程代码实现细节")Kotlin协程代码实现细节
--------------------------------------------------

由上述的例子我们知道了协程的逻辑执行流，我们来想一想这中间大概是怎么实现的，先思考这段话：

> **协程不会阻塞当前线程，执行异步逻辑时将该点之后的代码块的执行中断，在异步回调回来后恢复执行**

这里有两个关键点： **1 不阻塞** 、**2 中断、恢复执行**

*   **中断、恢复执行**  
    是不是有点类似有线程的运行机制，线程执行代码块时不一定一次就能执行完，cpu 分配的时间片有限，也会经历**暂停-恢复**的过程，终止的时候中间的状态使用线程栈帧来保存，从而使得恢复的时候知道终止时执行到的位置和其他所依赖的中间状态数据。同样，协程在挂起（不同于暂停，挂起不会阻塞线程运行）的时候也需要记录执行位置和一些中间变量。
    
*   **不阻塞**  
    我们要保证不阻塞,同时又要是同步式写法，该怎样做呢？异步回调`callback`是一种不阻塞的方式，或许协程底层也是这种实现方式（除此之外我不知道有什么不阻塞的底层实现方式），只不过外层帮我们封装成现在的同步式写法了。
    

带着这两个问题来看协程的源码，看看`kotlin`协程是怎么解决的，下面进入**高能源码**追踪部分，先来张图提提神

![](https://ethanhua.github.io/img/IMG_1205.JPG)

首先还是从上述例子中的代码为入口分析整个执行流程：

### 协程的启动

**GlobalScope.launch()**

```
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
} 
```

第一个参数 **context** 是协程上下文，此处我们传入了 **Main** 用于主线程调度，先忽略这部分。  
第二个参数 **start** 此处我们没有传值则使用默认值，代表启动方式默认值为立即执行。  
第三个参数 **block** 是协程真正执行的代码块，即我们例子中的`launch{}`花括号中的代码块，此处为一个`suspend CoroutineScope.() -> Unit`函数，这是怎么来的呢？我们编写代码的时候并没有这个东西，其实它由编译器生成的，我们的`block`代码块经过编译器编译后会生成一个继承`Continuation`的类，为了关注主要逻辑方便理解，去掉了一些无关代码大概代码如下:

```
final void setUpUI() {
        BuildersKt__Builders_commonKt.launch$default(GlobalScope.INSTANCE, 
        Dispatchers.getMain(), 
        null,
        
        (Function2)new KotlinTest$setUpUI.KotlinTest$setUpUI$1(this, (Continuation)null), 2, null);
        this.doSomethingElse2();
}

final class setUpUI$1 extends SuspendLambda implements Function2{
    public final Object invokeSuspend(Object result) {
        switch (this.label) {
            case 0:
                doSomethingElse()
                
                Deferred async$default = BuildersKt.async$default(coroutineScope, (CoroutineContext) Dispatchers.getDefault(), null, (Function2) new 1(null), 2, null);
                this.label = 1;
                
                if (async$default.await(this) == coroutine_suspended) {
                    return coroutine_suspended;
                }
                break;
            case 1:
                val data = result;
                processData(data)
                break;
        }
    }
} 
```

可以看到传入到 `launch` 函数第三个参数位置的是一个编译后生成的 `SuspendLambda` 类实例  
`SuspendLambda` 本质上是一个 `Continuation`，而 Continuation 是一个有着恢复操作的接口，刚好对应了我们上面例子中提到的**终止-恢复**的**恢复执行逻辑**

```
 @SinceKotlin("1.3")
public interface Continuation<in T> {
    
    
    public val context: CoroutineContext

    
    public fun resumeWith(result: Result<T>)
} 
```

`SuspendLambda` 继承结构如下

> `SuspendLambda` \> `ContinuationImpl` \> `BaseContinuationImpl` \> `Continuation`

每一层封装对应添加了不同的功能，我们先忽略掉这些功能细节，着眼于我们的主线，继续跟进`launch` 函数执行过程，由于第二个参数是默认值，所以创建的是 `StandaloneCoroutine`， 调用链如下：

`coroutine.start(start, coroutine, block)`  
-\> `CoroutineStart.start(block, receiver, this)`  
-\> `CoroutineStart.invoke(block: suspend () -> T, completion: Continuation<T>)`  
-\> `block.startCoroutineCancellable(completion)`  
-\> `createCoroutineUnintercepted(completion).intercepted().resumeCancellable(Unit)`

最后走到  
`createCoroutineUnintercepted(completion).intercepted().resumeCancellable(Unit)`

这里创建了一个协程，并链式调用 `intercepted`、`resumeCancellable` 方法，利用协程上下文中的 `ContinuationInterceptor` 对协程的执行进行拦截，`intercepted` 实际上调用的是 `ContinuationImpl` 的 `intercepted` 方法

```kotlin
internal abstract class ContinuationImpl(
    completion: Continuation<Any?>?,
    private val _context: CoroutineContext?
) : BaseContinuationImpl(completion) {
    ...
    public fun intercepted(): Continuation<Any?> =
        intercepted
            ?: (context[ContinuationInterceptor]?.interceptContinuation(this) ?: this)
                .also { intercepted = it }
    ...
} 
```

`context[ContinuationInterceptor]?.interceptContinuation`调用的是 `CoroutineDispatcher` 的 `interceptContinuation` 方法

```kotlin
public final <T> Continuation<T> interceptContinuation(@NotNull final Continuation<? super T> continuation) {
        Intrinsics.checkParameterIsNotNull(continuation, "continuation");
        return new DispatchedContinuation<T>(this, continuation);
    } 
```

最终创建了一个 `DispatchedContinuation` 可分发的协程实例，我们继续看`resumeCancellable` 方法

```kotlin
internal fun <T> Continuation<T>.resumeCancellable(value: T) = when (this) {
    
    is DispatchedContinuation -> resumeCancellable(value)
    else -> resume(value)
}

inline fun resumeCancellable(value: T) {
        
        
        if (dispatcher.isDispatchNeeded(context)) {
            _state = value
            resumeMode = MODE_CANCELLABLE
            dispatcher.dispatch(context, this)
        } else {
            UndispatchedEventLoop.execute(this, value, MODE_CANCELLABLE) {
                if (!resumeCancelled()) {
                    resumeUndispatched(value)
                }
            }
        }
    } 
```

最终走到 `dispatcher.dispatch(context, this)` 而这里的 `dispatcher` 就是通过工厂方法创建的 `HandlerDispatcher` ，`dispatch()` 函数第二个参数`this`是一个`runnable`这里为 `DispatchedTask`

**HandlerDispatcher**

```kotlin
 internal class HandlerContext private constructor(
    private val handler: Handler,
    private val name: String?,
    private val invokeImmediately: Boolean
) : HandlerDispatcher(), Delay {
    ...
    
    override fun dispatch(context: CoroutineContext, block: Runnable) {
        handler.post(block)
    }
    ...
} 
```

这里借用 Android 的主线程消息队列来在主线程中执行 `block Runnable` 而这个 `Runnable` 即为 `DispatchedTask`

```kotlin
internal abstract class DispatchedTask<in T>(
    @JvmField var resumeMode: Int
) : SchedulerTask() {
    ...
    public final override fun run() {
          ...
            withCoroutineContext(context, delegate.countOrElement) {
                if (job != null && !job.isActive)
                    
                    continuation.resumeWithException(job.getCancellationException())
                else {
                    val exception = getExceptionalResult(state)
                    if (exception != null)
                        
                        continuation.resumeWithStackTrace(exception)
                    else
                        
                        continuation.resume(getSuccessfulResult(state))
                }
            }
             ...
    }
}

@InlineOnly public inline fun <T> Continuation<T>.resume(value: T): Unit =
    resumeWith(Result.success(value))

internal abstract class BaseContinuationImpl(...) {
    
    public final override fun resumeWith(result: Result<Any?>) {
        ...
        val outcome = invokeSuspend(param)
        ...
    }
    
    protected abstract fun invokeSuspend(result: Result<Any?>): Any?
} 
```

最终调用到 `continuation.resumeWith()` 而 `resumeWith()` 中会调用 `invokeSuspend`，即之前编译器生成的 `SuspendLambda` 中的 `invokeSuspend` 方法

```kotlin
final class setUpUI$1 extends SuspendLambda implements Function2{
    public final Object invokeSuspend(Object result) {
        switch (this.label) {
            case 0:
                doSomethingElse()
                
                Deferred async$default = BuildersKt.async$default(coroutineScope, (CoroutineContext) Dispatchers.getDefault(), null, (Function2) new 1(null), 2, null);
                this.label = 1;
                
                if (async$default.await(this) == coroutine_suspended) {
                    return coroutine_suspended;
                }
                break;
            case 1:
                val data = result;
                processData(data)
                break;
        }
    }
} 
```

> **这段代码是一个状态机机制，每一个挂起点都是一种状态，协程恢复只是跳转到下一个状态，挂起点将执行过程分割成多个片段，利用状态机的机制保证各个片段按顺序执行。** 

如果没有挂起点就只有一个初始状态，类似于`callback`回调，所以对应了之前我们分析的非阻塞的异步底层实现其实也是一种`callback`回调，只不过有多个挂起点时就会有多个`callback`回调，我们把多个`callback`回调封装成了一个**状态机**。

到这里就完成了协程的启动过程，下面我们来分析协程的终止过程 （挂起过程）

### 协程的挂起

从上文可知协程的启动经历 **构建协程** -\> **线程调度** -\> **协程 resume** 三个过程进入到编译器生成的 `SuspendLambda` 的 `invokeSuspend` 方法中的

```kotlin
if (async$default.await(this) == coroutine_suspended) {
        return coroutine_suspended;
} 
```

`async` 也是一个协程，如果状态为挂起`coroutine_suspended`，则执行流直接 return 返回，如果已达到完成状态直接跳转下一个状态 case 1 最终走完整个协程代码块。

这里需要注意的是：

*   启动一个新的协程并不会挂起当前协程，只有当使用库函数 `await`、`yield`方法时才会将当前的协程挂起。
*   协程挂起并不会阻塞线程，线程在挂起点 return 后可以去执行其他的代码块。

协程的挂起过程很简单，代码块直接返回，当前状态保存在状态机 `SuspendLambda` 中，可以想象到协程恢复的时候也是调用 `SuspendLambda` 的 `invokeSuspend` 从而进入下一个状态继续执行的。

### 协程的恢复

通过前面的代码，我们大概可以想到，协程A 的恢复是在 新协程B 执行完毕后调用 协程A 的 `resume` 函数从而进入状态机恢复执行的, 这样 新协程B 就得持有 协程A 的引用，然后在执行完成后通知 协程A 恢复执行  
事实也是如此，我们从`await`方法入手

**DeferredCoroutine**

```kotlin
private open class DeferredCoroutine<T>(
    parentContext: CoroutineContext,
    active: Boolean
) : AbstractCoroutine<T>(parentContext, active), Deferred<T>, SelectClause1<T> {
    ...
    override suspend fun await(): T = awaitInternal() as T
    ...
}

internal suspend fun awaitInternal(): Any? {
        
        while (true) { 
            val state = this.state
            if (state !is Incomplete) {
                
                if (state is CompletedExceptionally) { 
                    recoverAndThrow(state.cause)
                }
                
                return state.unboxState()

            }
            if (startInternal(state) >= 0) break 
        }
        
        return awaitSuspend() 
    }

private suspend fun awaitSuspend(): Any? = suspendCoroutineUninterceptedOrReturn { uCont ->
        
        val cont = AwaitContinuation(uCont.intercepted(), this)
        cont.disposeOnCancellation(invokeOnCompletion(ResumeAwaitOnCompletion(this, cont).asHandler))
        cont.getResult()
} 
```

`awaitSuspend()` 函数中有三个过程我们需要关注

```kotlin
 suspendCoroutineUninterceptedOrReturn{ uCont ->
       
}

ResumeAwaitOnCompletion(this, cont).asHandler

invokeOnCompletion(handler: CompletionHandler) 
```

`suspendCoroutineUninterceptedOrReturn() lambda`直接返回了一个协程实例 `uCont`，我们大概可以确定这就是调用者协程的引用，源码里并没有看到实现，但是通过查看`JobSupport`的 class 源码发现调用`await`的时候就已经将当前协程作为参数传入了进去:

```
 if (async$default.await(this) == coroutine_suspended) {
                    return coroutine_suspended;
} 
```

再来看 `ResumeAwaitOnCompletion(this, cont).asHandler`

```
private class ResumeAwaitOnCompletion<T>(
    job: JobSupport,
    private val continuation: CancellableContinuationImpl<T>
) : JobNode<JobSupport>(job) {
    override fun invoke(cause: Throwable?) {
        ...
        
        continuation.resume(state.unboxState() as T)
        ...
    }
    ...
} 
```

构建了一个 `ResumeAwaitOnCompletion` 对象作为 `JobNode`任务节点，同时作为一个 `Handler` 回调对象。  
**invokeOnCompletion**

```
 public final override fun invokeOnCompletion(
        onCancelling: Boolean,
        invokeImmediately: Boolean,
        handler: CompletionHandler
    ): DisposableHandle {
        loopOnState { state ->
            when (state) {
                ... 
                is Incomplete -> {
                    val node = nodeCache ?: makeNode(handler, onCancelling).also { nodeCache = it }
                                    if (!addLastAtomic(state, list, node)) return@loopOnState
                    return node
                }
                else -> { 
                    if (invokeImmediately) handler.invokeIt((state as? CompletedExceptionally)?.cause)
                    return NonDisposableHandle
                }
                ...
            }
        }
    } 
```

如果新协程还未执行完成则将`handler`构造成一个`Node`并添加节点列表中去，函数返回 node，如果已经完成了则直接调用 `handler.invokeIt` 从而调用到 `Continuation.resume` 恢复执行。

到这里已经很明显了，将调用`await`的协程构造成一个 `handler node` 节点添加到新协程的 node 列表中，待新协程执行完毕后，循环列表通知每个`handler`恢复其协程的执行，这里用列表来保存`handler` 是因为可能有许多个协程调用`await`等待这个指定协程的完成，后面就不继续贴代码了，恢复流程如下：

从 `async` 协程的 `SuspendLambda` 的子类的  
`completion.resumeWith(outcome)`  
-\> `AbstractCoroutine.resumeWith(result)`  
-\> `JobSupport.tryFinalizeSimpleState()`  
-\> `JobSupport.completeStateFinalization()`  
-\> `state.list?.notifyCompletion(cause)`  
-\> `node.invoke`

到这里协程的整个调用执行流过程就分析完了，我们跟踪源码前的两个问题也得到了答案。

## 最后

协程只是编程历史发展过程中产生的对开发者友好的一种异步编程方式，相对于传统的多线程并发编程并不能提高额外的性能，协程并发编程的底层仍是用线程池实现，因为耗时的操作总要线程去执行，协程并不能减少额外的执行逻辑，另外协程因为创建了中间的一系列封装对象，比传统的多线程编程增加了额外的内存消耗，官方文档对此特定说明了编译器会进行一些优化，从而尽可能减少内存消耗，开发者无需担心，但协程所带来的开发维护成本大大减低、内存上的妥协带来的是对开发者的友好还是值得的。

**总结一下协程的优势**

*   **简洁** 更高抽象隐藏异步实现细节，实现写异步如同同步的顺序式写法
*   **比闭包更好用的上下文参数** 传统回调方式严重依赖闭包特性，而要在一个连续逻辑中写出多个回调函数且共享相同的上下文，这样的代码难读懂、难写对、难维护，协程就简单多了，一个上下文抽象搞定
*   **更方便处理并发逻辑** 父子协程概念 可以更容易的管理协程的执行

> 这边文章只分析的协程的主干实现逻辑，另外的功能还没有分析，比如 上下文的抽象、错误处理、通道模型等等，另外思考下这几个问题：**Java** 中为什么没有协程? 其他语言协程的实现方式同 **Kotlin** 一样吗？比如 **C#/Python**

下篇文章来揭晓，谢谢阅读！祝大家圣诞节快乐！  
**Thanks** [Kotlin协程官方设计文档](https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md)

![](https://ethanhua.github.io/img/IMG_1206.JPG)