---
{"dg-publish":true,"permalink":"/Program/Reactive/Project Reactor 之 publishOn 与 subscribeOn/","noteIcon":"","created":"2024-05-22T16:17:54.160+08:00"}
---

# 一、概述

在 Spring Reactor 项目中，有两个出镜较少的方法：`publishOn` 和 `subscribeOn`。这两个方法的作用是指定执行 Reactive Streaming 的 Scheduler（可理解为线程池）。

为何需要指定执行 Scheduler 呢？一个显而易见的原因是：组成一个反应式流的代码有快有慢，例如 NIO、BIO。如果将这些功能都放在一个线程里执行，快的就会被慢的影响，所以需要相互隔离。这是这两个方法应用的最典型的场景。

# 二、Scheduler

在介绍 `publishOn` 和 `subscribeOn` 方法之前，需要先介绍 [[Schedule (zh)\|Schedule (zh)]]r 这个概念。在 Reactor 中，`Scheduler` 用来定义执行调度任务的抽象。可以简单理解为线程池，但其实际作用要更多。先简单介绍 `Scheduler` 的实现：

-   `Schedulers.elastic()`: 调度器会动态创建工作线程，线程数无上界，类似于 `Execturos.newCachedThreadPool()`
-   `Schedulers.parallel()`: 创建固定线程数的调度器，默认线程数等于 CPU 核心数。

关于 `Scheduler` 的更多作用留在以后介绍。

# 三、publishOn 与 subscribeOn

接下来进入正题。先看两个例子（来自 [https://github.com/reactor/lite-rx-api-hands-on](https://github.com/reactor/lite-rx-api-hands-on)）

**`publishOn` 的例子**

```kotlin
Mono<Void> fluxToBlockingRepository(Flux<User> flux, 
                                    BlockingRepository<User> repository) {
    return flux
            .publishOn(Schedulers.elastic())
            .doOnNext(repository::save)
            .then();
} 
```

**`subscribeOn` 的例子**

```kotlin
Flux<User> blockingRepositoryToFlux(BlockingRepository<User> repository)  {
    return Flux.defer(() -> Flux.fromIterable(repository.findAll()))
               .subscribeOn(Schedulers.elastic());
} 
```

这里的 `repository` 的类型是 `BlockingRepository`，指的是会导致线程阻塞的数据库操作的集合，例如 JPA、MyBatis 等基于 JDBC 技术实现的 DAO。

在第一个例子中，在执行了 `publishOn(Schedulers.elastic())` 之后，`repository::save` 就会被 `Schedulers.elastic()` 定义的线程池所执行。

而在第二个例子中，`subscribeOn(Schedulers.elastic())` 的作用类似。它使得 `repository.findAll()`（也包括 `Flux.fromIterable`）的执行发生在 `Schedulers.elastic()` 所定义的线程池中。

从上面的描述看，`publishOn` 和 `subscribeOn` 的作用类似，那两者的区别又是什么？

## 两者的区别

简单说，两者的区别在于影响范围。`publishOn` 影响在其之后的 operator 执行的线程池，而 `subscribeOn` 则会从源头影响整个执行过程。所以，`publishOn` 的影响范围和它的位置有关，而 `subscribeOn` 的影响范围则和位置无关。

看个 `publishOn` 和 `subscribeOn` 同时使用的例子

```kotlin
Flux.just("tom"）
        .map(s -> {
            System.out.println("[map] Thread name: " + Thread.currentThread().getName());
            return s.concat("@mail.com");
        })
        .publishOn(Schedulers.newElastic("thread-publishOn"))
        .filter(s -> {
            System.out.println("[filter] Thread name: " + Thread.currentThread().getName());
            return s.startsWith("t");
        })
        .subscribeOn(Schedulers.newElastic("thread-subscribeOn"))
        .subscribe(s -> {
            System.out.println("[subscribe] Thread name: " + Thread.currentThread().getName());
            System.out.println(s);
        }); 
```

输出结果如下：

    [map] Thread name: thread-subscribeOn-3
    [filter] Thread name: thread-publishOn-4
    [subscribe] Thread name: thread-publishOn-4
    tom@mail.com 

从上面的例子可以看出，`subscribeOn` 定义在 `publishOn` 之后，但是却从源头开始生效。而在 `publishOn` 执行之后，线程池变更为 `publishOn` 所定义的。

## 实际用途

这里介绍 `publishOn` 和 `subscribeOn` 的一种实际用途，那就是反应式编程和传统的，会导致线程阻塞的编程技术混用的场景。其实开头两个例子已经解释了这个场景。

在第一个 `publishOn` 的例子中，`repository::save` 会导致线程阻塞，为了避免造成对其它反应式操作的影响，便使用 `publishOn` 改变其执行线程。

在第二个 `subscribeOn` 的例子中，`repository.findAll()` 会导致线程阻塞。但是其是源头的 publisher，因此不能使用 `publishOn` 改变其 执行线程。这时就需要使用 `subscribeOn`，在源头上修改其执行线程。

这样，通过 `publishOn` 和 `subscribeOn` 就在反应式编程中实现了线程池隔离的目的，一定程度上避免了会导致线程阻塞的程序执行影响到反应式编程的程序执行效率。

## 局限性

使用 `publishOn` 和 `subscribeOn` 只能在一定程度上避免反应式编程代码执行的效率被影响。因为用来隔离的线程池资源终归是有限的，比如当出现数据库资源不足、慢查询等问题时，对应的线程池资源如果被耗尽，还是会使整个反应式编程的执行效率受到影响。

目前，Redis、Mongo、Couchbase 等非关系型数据库均有相应的反应式编程的解决方案，但是关系型数据库却没有理想的方案。一个重要原因是 JDBC 本身就是一个阻塞式的 API，根本不可能让其适应反应式编程。因此需要一个新的方案。目前 Oracle 正在推动 ADBA (Asynchronous Database Access API)，使得关系型数据库可以满足异步编程的需要。但是，因为是 Oracle 主导，大家都懂的，所以目前前景还不是很明朗。另外一个技术方案是 Spring 推动的 R2DBC，从名字上来看就很像是 JDBC 在反应式编程领域的对应的解决方案。目前可以支持 PostgreSQL，支持 MySQL 目前还尚需时日。

# 后续

接下来关于 Project Reactor 的文章我打算向大家介绍一下 Hot 和 Cold Publisher 的概念以及 Project Reactor 的源码实现。

