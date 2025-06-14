---
{"dg-publish":true,"permalink":"/Program/Mixed/Reactor线程切换解析/","noteIcon":"","created":"2025-04-01T23:45:04.433+08:00"}
---

在 Reactor（Project Reactor）中，线程切换通常通过 `publishOn` 和 `subscribeOn` 这两个操作符来完成。它们的作用如下：

- **`publishOn(Scheduler scheduler)`**：影响的是 **下游**（之后的操作）执行的线程。
- **`subscribeOn(Scheduler scheduler)`**：影响的是 **上游**（数据源及之前的操作）执行的线程。
    

## 1. `publishOn` —— 控制下游线程

`publishOn` 适用于需要在不同的线程上执行流处理的情况。例如，数据流的某部分计算量大，可以切换到 `Schedulers.parallel()` 以提高效率。

```java
Flux.range(1, 5)
    .map(i -> {
        System.out.println("First map on: " + Thread.currentThread().getName());
        return i;
    })
    .publishOn(Schedulers.parallel()) // 切换到并行线程池
    .map(i -> {
        System.out.println("Second map on: " + Thread.currentThread().getName());
        return i;
    })
    .subscribe();
```

### 运行示例输出：

```
First map on: main
First map on: main
First map on: main
First map on: main
First map on: main
Second map on: parallel-1
Second map on: parallel-2
Second map on: parallel-3
Second map on: parallel-4
Second map on: parallel-5
```

> **解释：**
> 
> - `First map` 在 **main 线程** 中执行。
>     
> - `publishOn(Schedulers.parallel())` 之后，`Second map` 被切换到 `parallel-x` 线程池执行。
>     

---

## 2. `subscribeOn` —— 控制上游线程

`subscribeOn` 影响的是 **数据的订阅及生成线程**，即上游（数据源）的执行线程。通常用于 I/O 操作，比如读取数据库或调用 HTTP API。

```java
Flux.range(1, 5)
    .subscribeOn(Schedulers.boundedElastic()) // 影响上游
    .map(i -> {
        System.out.println("Mapping on: " + Thread.currentThread().getName());
        return i;
    })
    .subscribe();
```

### 运行示例输出：

```
Mapping on: boundedElastic-1
Mapping on: boundedElastic-1
Mapping on: boundedElastic-1
Mapping on: boundedElastic-1
Mapping on: boundedElastic-1
```

> **解释：**
> 
> - `subscribeOn(Schedulers.boundedElastic())` 影响的是 **上游的数据流产生的线程**，所以 `map` 也在 `boundedElastic` 线程池中执行。
>     

---

## 3. `publishOn` + `subscribeOn` 组合使用

如果同时使用 `subscribeOn` 和 `publishOn`，**`subscribeOn` 影响的是数据的生成线程，而 `publishOn` 影响的是下游的执行线程**。

```java
Flux.range(1, 5)
    .subscribeOn(Schedulers.boundedElastic()) // 影响上游
    .map(i -> {
        System.out.println("First map on: " + Thread.currentThread().getName());
        return i;
    })
    .publishOn(Schedulers.parallel()) // 影响下游
    .map(i -> {
        System.out.println("Second map on: " + Thread.currentThread().getName());
        return i;
    })
    .subscribe();
```

### 运行示例输出：

```
First map on: boundedElastic-1
First map on: boundedElastic-1
First map on: boundedElastic-1
First map on: boundedElastic-1
First map on: boundedElastic-1
Second map on: parallel-1
Second map on: parallel-2
Second map on: parallel-3
Second map on: parallel-4
Second map on: parallel-5
```

> **解释：**
> 
> - `subscribeOn(Schedulers.boundedElastic())` 让上游的 `map` 在 `boundedElastic` 线程池执行。
>     
> - `publishOn(Schedulers.parallel())` 让下游的 `map` 在 `parallel-x` 线程池执行。
>     

---

## 4. `Schedulers` 选择：

Reactor 提供了一些默认的 `Scheduler`：

- `Schedulers.parallel()`：适用于 **CPU 密集型** 任务，线程数与 CPU 核心数相等。
    
- `Schedulers.boundedElastic()`：适用于 **I/O 密集型** 任务（文件、数据库、网络），支持动态扩展。
    
- `Schedulers.single()`：单线程调度器，适用于顺序执行的任务。
    
- `Schedulers.immediate()`：在当前线程执行（无切换）。
    
- `Schedulers.fromExecutorService()`：可以自定义线程池。
    

---

## 总结：

| 操作符                      | 作用           | 影响范围          |
| ------------------------ | ------------ | ------------- |
| `subscribeOn(Scheduler)` | 切换 **上游** 线程 | 数据源（订阅时的执行线程） |
| `publishOn(Scheduler)`   | 切换 **下游** 线程 | 影响它之后的操作      |

如果你需要：

1. **控制数据生成（I/O 操作）线程** → 用 `subscribeOn(Schedulers.boundedElastic())`。
2. **控制数据处理（计算任务）线程** → 用 `publishOn(Schedulers.parallel())`。

你在实际开发中会经常遇到 I/O 线程切换（比如 WebFlux 请求、数据库查询）和计算线程切换（比如数据处理、流式计算），掌握 `subscribeOn` 和 `publishOn` 的区别对优化性能很有帮助。