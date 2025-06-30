---
{"dg-publish":true,"permalink":"/Program/Reactive/Reactive Streams 与 Java 8 Stream 有什么区别/","noteIcon":"","created":"2024-05-22T16:17:54.160+08:00"}
---

之前曾写过一篇介绍 Spring 提出的 Reactive Streams 实现方案 —— Project Reactor 的文章：《Reactor 入门与实践》，其中介绍了 Spring Reactor API 的用法。相信很多人都能发现，Spring Reactor 的 API 看上去和 Java 8 Stream 很像：

**Project Reactor:**

```kotlin
Flux.fromIterable(Character.captainAmerica3())
        .filter(Character::isPlay)
        .map(Character::getName)
        .subscribe(System.out::println); 
```

**Java 8 Stream:**

```kotlin
Character.captainAmerica3().stream()
        .filter(Character::isPlay)
        .map(Character::getName)
        .forEach(System.out::println); 
```

上面两段代码分别由 Spring Reactor 和 Java 8 Stream 实现，功能是打印《美队 3》参演的角色名称。代码的形式和大部分方法名称都是相同的。

那 Spring Reactor 或者说 Reactive Streams 和 Java 8 Stream 的差别是什么呢？

简单来说，这两者最大的差别是前者是 Push-based，后者是 Pull-based。

# Push 与 Pull

Reactive Streams 与 Java 8 Stream 最大的不同在于，前者是 Push 模式，后者是 Pull 模式。可能大部分人对推与拉的概念不是很了解。先做一个形象的介绍，常见的操作集合类的方式就是 Pull 模式：

```kotlin
List<User> users = userRpcClient.findAllUsers();
for (String user : users) {
    // do something
} 
```

为什么说这样的代码是 Pull 模式呢？因为这样的代码，客户端主动向服务端请求数据，然后在操作数据。好比客户端主动从服务端拉取数据，故称为拉模式。

那 Push 模式是什么样呢？

```kotlin
userRpcPublisher.subscribe(new UserSubscriber() {
    public void onNext(User user) {
        // do something
    }
}); 
```

简单来说就是 Callback 风格的代码通常都是 Push 模式的。

> Pull 模式的代码的问题在于，如果 `userRpcClient.findAllUsers()` 表示的操作是一个很耗时的操作，那 Pull-based 的代码的并发能力将很成问题。这就是 Push 模式出现的原因。

当然，上面两个例子只是为了给大家一个直观的介绍。并不是说传统形式的代码就一定是 Pull 模式，Callback 风格的代码就一定是 Pull 模式。

换言之，代码形式并不是 Pull 与 Push 的本质。从更深的层面说，Pull 模式对应的是同步的、命令式的程序，Push 模式对应的是异步的、非阻塞的、反应式的程序。

因此，虽然在代码形式上说 Java 8 Stream 和 Reactive Streams 的代码有些像，但从本质上来说，同步、阻塞的 Java 8 Stream 与异步、非阻塞的 Reactive Streams 有着很大的差别。

因此 Reactive Streams 不仅在形的层面，以接口定义的形式对反应式编程做出了规范，更在实的层面定义了 TCK，用来保证相关实现确实满足了异步、非阻塞等等的要求。 
 [https://www.jianshu.com/p/a6a8de73f4bf](https://www.jianshu.com/p/a6a8de73f4bf)
