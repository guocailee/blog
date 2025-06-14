---
{"dg-publish":true,"permalink":"/Program/Reactive/在Reactor中subscribeOn，publishOn有什么区别/","noteIcon":"","created":"2025-03-06T21:28:25.983+08:00"}
---

在 Reactor 中，`subscribeOn` 和 `publishOn` 是用于控制流的调度和执行的两个重要操作符。

1. `subscribeOn`:
   - `subscribeOn` 用于指定在哪个线程执行订阅操作（即 `subscribe()` 方法）。
   - 它的影响范围是在整个流的生命周期内，包括数据源的生成和操作符的执行。
   - 使用 `subscribeOn` 可以决定整个流的调度策略，它会影响整个流的执行线程。

2. `publishOn`:
   - `publishOn` 用于指定在哪个线程执行操作符的后续操作，即从该操作符开始的后续操作。
   - 它只影响其后的操作符，不会影响之前的部分。
   - 使用 `publishOn` 可以在流中的特定位置更改执行线程，从而实现部分流的调度策略变更。

总结：
- `subscribeOn` 影响整个流的调度策略，可以在流的起始位置指定执行线程。
- `publishOn` 用于更改从该操作符开始的后续操作的执行线程，可以在流的中间位置进行调度策略变更。

通常情况下，建议在流的起始位置使用 `subscribeOn`，以确定数据源的生成和整个流的调度策略，而在流的中间位置使用 `publishOn` 进行局部的调度策略变更。


以下是使用 Kotlin 和 Reactor 库编写的示例，演示了 `subscribeOn` 和 `publishOn` 的用法：

```kotlin
import reactor.core.publisher.Flux
import reactor.core.scheduler.Schedulers

fun main() {
    // 使用 subscribeOn 来指定数据源的生成线程
    val flux = Flux.range(1, 5)
        .subscribeOn(Schedulers.boundedElastic()) // 使用 elastic 调度器生成数据
        .map { it * 2 }
        .doOnNext { println("Processing on thread: ${Thread.currentThread().name}") }

    // 使用 publishOn 来指定后续操作的执行线程
    flux
        .publishOn(Schedulers.parallel()) // 使用 parallel 调度器执行后续操作
        .subscribe { println("Received: $it on thread: ${Thread.currentThread().name}") }

    Thread.sleep(1000) // 等待异步操作完成
}
```
输出如下：
```bash
Processing on thread: boundedElastic-1
Processing on thread: boundedElastic-1
Processing on thread: boundedElastic-1
Processing on thread: boundedElastic-1
Processing on thread: boundedElastic-1
Received: 2 on thread: parallel-1
Received: 4 on thread: parallel-1
Received: 6 on thread: parallel-1
Received: 8 on thread: parallel-1
Received: 10 on thread: parallel-1

Process finished with exit code 0
```

在这个示例中：

- `Flux.range(1, 5)` 生成一个包含 1 到 5 的整数序列的 Flux。
- `subscribeOn(Schedulers.boundedElastic())` 指定生成数据的线程为`boundedElastic` 线程池，即在后台异步执行。
- `map { it * 2 }` 对每个元素进行乘以 2 的操作。
- `publishOn(Schedulers.parallel())` 指定后续操作的执行线程为` Parallel` 线程池，即在并行执行。
- `doOnNext` 操作输出处理数据的线程信息。
- `subscribe` 订阅并输出每个元素接收到的值及其执行线程信息。

通过运行此示例，您可以看到不同操作所在的线程以及使用 `subscribeOn` 和 `publishOn` 控制的执行线程切换。