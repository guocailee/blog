---
{"dg-publish":true,"permalink":"/Program/Java/Java 异步编程：从 Future 到 Loom/","noteIcon":"","created":"2025-03-06T21:28:25.976+08:00"}
---

众所周知，Java 开始方法执行到结束，都是由同一个线程完成的。这种方式虽易于开发调试，但容易因为锁、IO 等原因导致线程挂起，产生线程上下文切换。随着对应用并发能力要求越来越高，频繁的线程上下文切换所带来的成本变得难以忽视。同时，线程也是相对宝贵的资源，无限制的增加线程是不可能的。优秀的技术人员应该能让应用使用更少的线程资源实现更高的并发能力。这便是我们今天要讨论的话题 —— Java 异步编程技术。

异步编程其实并没有清晰定义。通常我们认为，从方法开始到结束都必须在同一个线程内调度执行的编程方式可被认为是同步编程方式。但因为这样的方式是我们习以为常的，所以也就没有专门名字去称呼它。与这种同步方式相对的，便是异步。即方法的开始到结束可以由不同的线程调度执行的编程方式，被成为异步编程。

**异步编程技术目的，重点并非提高并发能力，而是提高伸缩性 (Scalability)**。现在的 Web 服务，应付 QPS 几百上千，甚至上万的场景并没有太大问题，但问题是如何在并发请求量突增的场景中提供稳定服务呢？如果一个应用能稳定提供 QPS 1000 的服务。假如在某一个大促活动中，这个应用的 QPS 突然增加到 10000 怎么办？或者 QPS 没变，但这个应用所依赖的服务发生故障，或网络超时。当这些情况发生时，服务还能稳定提供吗？虽然熔断、限流等技术能够解决这种场景下服务的可用性问题，但这毕竟是一种舍车保帅的做法。**是否能在流量突增时仍保证服务质量呢？答案是肯定的，那就是异步编程 + NIO。NIO 技术本身现在已经很成熟了，关键是用一种什么样的异步编程技术将 NIO 落地到系统，尤其是业务快速迭代的前台、中台系统中。** 

**这就是本文讨论 Java 异步编程的原因。Java 应用开发领域究竟有哪些技术可以用来提升系统的伸缩性？本文将按照这些技术的演化历程，介绍一下这些技术的意义和演化过程：** 

-   Future
-   Callback
-   Servlet 3.0
-   反应式编程
-   Kotlin 协程
-   Project Loom

# 一、Future

J.U.C 中的 Future 算是 Java 对异步编程的第一个解决方案。当向线程池 submit 一个任务后，这个任务便被另一个线程执行了：

    Future future = threadPool.submit(() -> {
      foobar();
      return result;
    });

    Object result = future.get(); 

但这个解决方案有很多缺陷：

1.  无法方便得知任务何时完成
2.  无法方便获得任务结果
3.  在主线程获得任务结果会导致主线程阻塞

# 二、Callback

为了解决使用 Future 所存在的问题，人们提出了一个叫 Callback 的解决方案。比如 Google Guava 包中的 ListenableFuture 就是基于此实现的：
```java
ListeningExecutorService service = MoreExecutors.listeningDecorator(Executors.newFixedThreadPool(10));
    ListenableFuture<Explosion> explosion = service.submit(new Callable<Explosion>() {
      public Explosion call() {
        return pushBigRedButton();
      }
    });
    Futures.addCallback(explosion, new FutureCallback<Explosion>() {
      // we want this handler to run immediately after we push the big red button!
      public void onSuccess(Explosion explosion) {
        walkAwayFrom(explosion);
      }
      public void onFailure(Throwable thrown) {
        battleArchNemesis(); // escaped the explosion!
      }
    }); 
```

通过执行 `ListenableFuture<Explosion> explosion = service.submit(new Callable<Explosion>() {})` 创建异步任务。通过 `Futures.addCallback(explosion, new FutureCallback<Explosion>() {}` 添加处理结果的回调函数。这样避免获取并处理异步任务执行结果阻塞调起线程的问题。Callback 是将任务执行结果作为接口的入参，在任务完成时回调 Callback 接口，执行后续任务，从而解决纯 Future 方案无法方便获得任务执行结果的问题。

但 Callback 产生了新的问题，那就是代码可读性的问题。因为使用 Callback 之后，代码的字面形式和其所表达的业务含义不匹配，即业务的先后关系到了代码层面变成了包含和被包含的关系。

**因此，如果大量使用 Callback 机制，将使大量的应该是先后的业务逻辑在代码形式上表现为层层嵌套。这会导致代码难以理解和维护。这便是所谓的 Callback Hell（回调地狱）问题。** 

Callback Hell 问题可以从两个方向进行一定的解决：一是事件驱动机制、二是链式调用。前者被如 Vert.x 所使用，后者被 CompletableFuture、反应式编程等技术采用。但这些优化的效果有限，不能根本上解决 Callback 机制所带来的代码可维护性的下降。

## Callback 与 NIO

**Callback 真正体现价值，是它与 NIO 技术结合之后。原因也很简单：对于 CPU 密集型应用，采用 Callback 风格没有意义；对于 IO 密集型应用，如果是使用 BIO，Callback 同样没有意义，因为最终会有一个线程是因为 IO 而阻塞。而只有使用 NIO 才能避免线程阻塞，也必须使用 Callback 风格，才能使应用得以被开发出来。NIO 的广泛应用是在 Apache Mina、JBoss Netty 等技术出现之后。这些技术很大程度地简化了 NIO 技术的使用，但直接使用它们开发业务系统还是很繁琐。** 

下面看一个真实的例子。这个例子背后的完整应用的功能是将微软 Exchange 服务接口（Exchange Web Service）转换为 Rest 风格的接口，下面这段代码是这个应用的一部分。

```java
    public class EwsCalendarHandler extends ChannelInboundHandlerAdapter {
        @Override
        public void channelRead(final ChannelHandlerContext ctx, Object msg) {
            if (msg instanceof HttpRequest) {
                final HttpRequest origReq = (HttpRequest) msg;

                HttpRequest request = translateRequest(origReq);

                if (backendChannel == null) {
                    connectBackendFuture = connectBackend(ctx, StaticConfiguration.EXCHANGE_PORT);
                    sendMessageAfterConnected(ctx, request);
                } else if (backendChannel.isActive()) {
                    setHttpRequestToBackendHandler(request);
                    sendObjectAndFlush(ctx, request);
                } else {
                    sendMessageAfterConnected(ctx, request);
                }
            } else if (msg instanceof HttpContent) {
                HttpContent content = (HttpContent) msg;
                if (backendChannel == null || !backendChannel.isActive()) {
                    sendMessageAfterConnected(ctx, content);
                } else {
                    sendObjectAndFlush(ctx, content);
                }
            }
        }
        
        private void sendMessageAfterConnected(final ChannelHandlerContext ctx, final HttpObject message) {
            if (connectBackendFuture == null) {
                LOGGER.warn("next hop connect future is null, drop the message and return: {}", message);
                return;
            }

            connectBackendFuture.addListener((ChannelFutureListener) future -> {
                if (future.isSuccess()) {
                    ChannelFuture f = sendObjectAndFlush(ctx, message);
                    if (f != null) {
                        f.addListener((future1) ->
                                           backendChannel.attr(FIND_ITEM_START_ATTR_KEY).set(System.currentTimeMillis())
                        );
                    }

                }
            });
        }
    } 
```

在方法 `sendMessageAfterConnected` 中，我们已经能看到嵌套两层的 Callback。而上面实例中的 `EwsCalendarHandler` 所实现的 `ChannelInboundHandler` 接口，本质上也是一个回调接口。

其实上面的例子只有一级服务调用。在微服务流行的今天，多级服务调用很常见，一个服务先调 A，再用结果 A 调 B，然后用结果 B 调用 C，等等。这样的场景，如果直接用 Netty 开发，技术难度会比传统方式增加很多。这其中的难度来自两方面，一是 NIO 和 Netty 本身的技术难度，二是 Callback 风格所导致的代码理解和维护的困难。

因此，直接使用 Netty，通常局限在基础架构层面，在前台和中台业务系统中，应用较少。

# 三、Servlet 3.0

上面讲到，如果直接使用 Netty 开发应用，将不可避免地遇到 Netty 和 NIO 本身的技术挑战，以及 Callback Hell 问题。对于前者，Servlet 3.0 提供了一个解决方案。

**▼ 示例：Servlet 3.0 ▼**

    @WebServlet(urlPatterns = "/demo", asyncSupported = true)
    public class AsyncDemoServlet extends HttpServlet {
        @Override
        public void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException, ServletException {
            // Do Something
     
            AsyncContext ctx = req.startAsync();
            startAsyncTask(ctx);
        }
    }
     
    private void startAsyncTask(AsyncContext ctx) {
        requestRpcService(result -> {
            try {
                PrintWriter out = ctx.getResponse().getWriter();
                out.println(result);
                out.flush();
                ctx.complete();
            } catch (Exception e) {
                e.printStackTrace();
            }
        });
    } 

Servlet 3.0 的出现，解决了在过去基于 Servlet 的 Web 应用中，接受请求和返回响应必须在同一个线程的问题，实现了如下目标：

1.  可以避免了 Web 容器的线程被阻塞挂起
2.  使请求接收之后的任务处理可由专门线程完成
3.  不同任务可以实现线程池隔离
4.  结合 NIO 技术实现更高效的 Web 服务

除了直接使用 Servlet 3.0，也可以选择 Spring MVC 的 Deferred Result。

**▼ 示例：Spring MVC DeferredResult ▼**

```java
    @GetMapping("/async-deferredresult")
    public DeferredResult<ResponseEntity<?>> handleReqDefResult(Model model) {
        LOG.info("Received async-deferredresult request");
        DeferredResult<ResponseEntity<?>> output = new DeferredResult<>();
         
        ForkJoinPool.commonPool().submit(() -> {
            LOG.info("Processing in separate thread");
            try {
                Thread.sleep(6000);
            } catch (InterruptedException e) {
            }
            output.setResult(ResponseEntity.ok("ok"));
        });
         
        LOG.info("servlet thread freed");
        return output;
    } 
```

## Servlet 3.0 的技术局限

Servlet 3.0 并不是用来解决前面提到的 Callback Hell 问题的，它只是降低了异步 Web 编程的技术门槛。对于 Callback Hell 问题，使用 Servlet 3.0 或类似技术时同样会遇到。解决 Callback Hell 还需另寻他法。

# 四、反应式编程

现在挡在异步编程最大的障碍就是 Callback Hell，因为 Callback Hell 对代码可读性有很大杀伤力。而本节介绍的反应式编程技术，除了响应性、伸缩性、容错性以外，从开发人员的角度来讲，就是代码可读性要比 Callback 提升了许多。

**▼ 图：反应式编程的特性 ▼**

![](https://upload-images.jianshu.io/upload_images/3108769-d85dc416633c347f.jpg)


**▼ 反应式编程简单示例 ▼**

```java
    userService.getFavorites(userId)
               .flatMap(favoriteService::getDetails)
               .switchIfEmpty(suggestionService.getSuggestions())
               .take(5)
               .publishOn(UiUtils.uiThreadScheduler())
               .subscribe(uiList::show, UiUtils::errorPopup); 
```
可读性的提高原因在于反应式编程可让开发人员将实现业务的各种方法使用链式算子串联起来，而串联起来的各种方法的先后关系与执行顺序大体一致。

这其实是采用了函数式编程的设计，通过函数式编程解决了之前 Callback 设计存在的代码可读性问题。

虽然相对于 Callback，代码可读性是反应式编程的优点，但这种优点是相对的，相对于传统代码，可读性就成了反应式编程的缺点。上面的例子代码看上去还容易理解，但换成下面的例子，大家就又能重新看到 Callback Hell 的影子了：

**▼ 示例：查询最近邮件数（反应式编程版） ▼**

```java
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
因此，反应式编程只看代码形式，可以被视为 Callback 2.0。解决了之前的一些问题，但并不彻底。

目前，在 Java 领域实现了反应式编程的技术有 Spring 的 Project Reactor、Netflix RxJava 1/2 等。前者的 3.0 版本作为 Spring 5 的基础，在 17 年底发布，推动了后端领域反应式编程的发展。后者出现时间更早，在前端开发领域应用的比后端更要广泛一些。

除了开源框架，JDK 也提供了对反应式编程解决方案：JDK 8 的 CompletableFuture 不算是反应式编程，但是它在形式上带有一些反应式编程的函数式代码风格。JDK 9 Flow 实现了 Reactive Streams 规范，但是实施反应式编程需要完整的解决方案，单靠 Flow 是不够的，还是需要 Project Reactor 这样的完整解决方案。但 JDK 层面的技术能提供统一的技术抽象和实现，在统一技术方面还是有积极意义的。

## 反应式编程的应用范围

正如前面所说，反应式编程仍然存在代码可读性的问题，这个问题在加上反应式编程本身的技术门槛，使得用反应式编程技术在业务系统开发领域一直没有流行普及。但是对于核心系统、底层系统，反应式编程技术所带来的伸缩性、容错性的提升同其增加的开发成本相比通常是可以接受。因此核心系统、底层系统是适合采用反应式编程技术的。

# 五、Kotlin 协程

前面介绍的各种技术，都有明显的缺陷：Future 不是真异步；Callback 可读性差；Servlet 3.0 等技术没能解决 Callback 的缺陷；反应式编程还是难以编写复杂业务。到了 18 年，一种新的 JVM 编程语言开始流行：Kotlin。Kotlin 首先流行在 Android 开发领域，因为它得到了 Google 的首肯和支持。但对于后端开发领域，因为一项特性，使得 Kotlin 也非常值得注意。那就是 Kotlin Coroutine（后文称 Kotlin 协程）。对于这项技术，我已经写过三篇文章，分别介绍入门、原理和与 Spring Project Reactor 的整合方式。感兴趣的同学可以去我的简书和微信公众号上去看这些文章（搜索 “编走编想”）。

协程技术不是什么新技术，它在很多语言中都有实现，比如大家所熟悉的 Python、Lua、Go 都是支持协程的。在不同语言中，协程的实现方法各有不同。因为 Kotlin 的运行依赖于 JVM，不能对 JVM 进行修改，因此，Kotlin 不能在底层支持协程。同时，Kotlin 是一门编程语言，需要在语言层面支持协程，而不是像框架那样在语言层面之上支持。因此，Kotlin 对协程支持最核心的部分是在编译器中。因为对这部分原理的解释在之前文章中都有涉及，因此不在这里重复。

使用 Kotlin 协程之后最大的好处是异步代码的可读性大大提高。如果上一个示例用 Kotlin 协程实现，那就是下面的样子：

**▼ 示例：查询最近邮件数（Kotlin 协程版） ▼**

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

_目前在 Spring 应用中使用 Kotlin 协程还有些小繁琐，但在 Spring Boot 2.2 中，可以直接在 Spring WebFlux 方法上使用 suspend 关键字。_

Kotlin 协程最大的意义就是可以用看似指令式编程方式（Imperative Programming  
，即传统编程方式）去写异步编程代码。并发和代码可读性似乎两全其美了。

## Kotlin 协程的局限性

但事情不是那么完美。Kotlin 协程依赖于各种基于 Callback 的技术。像上面的例子，之所以可以用 Kotlin 协程，是因为上一个版本使用了反应式编程技术。所以，只有当一段代码使用了 ListenableFuture、CompletableFuture、Project Reactor、RxJava 等技术时，才能用 Kotlin 协程进行改造优化。那对于其它的会阻塞线程的技术，如 Object.wait、Thread.sleep、Lock、BIO 等，Kotlin 协程就无能为力了。

另外一个局限性源于 Kotlin 本身。虽然 Kotlin 兼容 Java，但这种兼容并非完美。因此，对于组件，尤其是基础组件的开发，并不推荐使用 Kotlin，而是更推荐使用 Java。这也导致 Kotlin 协程的使用范围被进一步地限制。

# 六、Project Loom

前面讲到，虽然 Kotlin 协程看上去很好，但在使用上还是有着种种限制。那有没有更好的选择呢？答案是 Project Loom ([https://openjdk.java.net/projects/loom/](https://links.jianshu.com/go?to=https%3A%2F%2Fopenjdk.java.net%2Fprojects%2Floom%2F))。这个项目在 18 年底的时候已经达到可初步演示的原型阶段。**不同于之前的方案，Project Loom 是从 JVM 层面对多线程技术进行彻底的改变。** 

Project Loom 设计思想与之前的一个开源 Java 协程技术非常相似。这个技术就是 Quasar Fiber [https://docs.paralleluniverse.co/quasar/](https://links.jianshu.com/go?to=https%3A%2F%2Fdocs.paralleluniverse.co%2Fquasar%2F) 。而现在 Project Loom 的主要设计开发人员 Ron Pressler 就是来自 Quasar Fiber。

这里建议大家读一下 Project Loom 的这篇文档：[http://cr.openjdk.java.net/~rpressler/loom/Loom-Proposal.html](https://links.jianshu.com/go?to=http%3A%2F%2Fcr.openjdk.java.net%2F%7Erpressler%2Floom%2FLoom-Proposal.html)。这篇文档介绍了发起 Project Loom 的原因，以及 Java 线程基础的很多底层设计。

其实发起 Project Loom 的原因也很简单：长期以来，Java 的线程是与操作系统的线程一一对应的，这限制了 Java 平台并发能力的提升。各种框架或其它 JVM 编程语言的解决方案，都在使用场景上有限制。例如 Kotlin 协程必须基于各种 Callback 技术，而 Callback 技术有存在编写、调试困难的问题。为了使 Java 并发能力在更大范围上得到提升，从底层进行改进便是必然。

下面这幅图很好地展示了目前 Java 并发编程方面的困境，简单的代码并发、伸缩能力差；并发、伸缩能力强的代码复杂，难以与现有代码整合。

![](https://upload-images.jianshu.io/upload_images/3108769-00cf8e476d0d9e26.jpg)

image

为了让简单和高并发这两个目标兼得，我们需要 Project Loom 这个项目。

## 使用方法

在引入 Project Loom 之后，JDK 将引入一个新类：`java.lang.Fiber`。此类与 `java.lang.Thread` 一起，都成为了 `java.lang.Strand` 的子类。即线程变成了一个虚拟的概念，有两种实现方法：Fiber 所表示的轻量线程和 Thread 所表示的传统的重量级线程。

对于应用开发人员，使用 Project Loom 很简单：

```java
    Fiber f = Fiber.schedule(() -> {
      println("Hello 1");
      lock.lock(); // 等待锁不会挂起线程
      try {
          println("Hello 2");
      } finally {
          lock.unlock();
      }
      println("Hello 3");
    }) 
```

只需执行 ```Fiber.schedule(Runnable task)``` 就能在 Fiber 中执行任务。**最重要的是，上面例子中的 `lock.lock()` 操作将不再挂起底层线程。除了 Lock 不再挂起线程以外，像 Socket BIO 操作也不再挂起线程。**  但 `synchronized`，以及 Native 方法中线程挂起操作无法避免。

```java
    synchronized (monitor) {
      // 在 Fiber 中调用这条语句还是会挂起线程。
      socket.getInputStream().read();
    } 
```

如上所示，Fiber 的使用非常简单。因此，让现有系统使用 Project Loom 很容易。像 Tomcat、Jetty 这样的 Web 容器，只需将处理请求操作从使用 ThreadPoolExecutor execute 或 submit 改为使用 Fiber schedule 即可。这个视频 [https://www.youtube.com/watch?v=vbGbXUjlRyQ&t=1240s](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.youtube.com%2Fwatch%3Fv%3DvbGbXUjlRyQ%26t%3D1240s) 中的 Demo 展示了 Jetty 使用 Project Loom 改造之后并发吞吐能力的大幅提升。

## 实现原理

接下来简单介绍一下 Project Loom 的实现原理。Project Loom 的使用主要基于 Fiber，而实现则主要基于 Continuation。Contiuation 表示一个可暂停和恢复的计算单元。在 Project Loom 中，Continuationn 使用 `java.lang.Continuation` 类实现。这个类主要供类库实现使用，而不是直接被应用开发人员使用。Continuation 主要内容如下所示：

```java
    package java.lang;

    public class Continuation implements Runnable {
      public Continuation(ContinuationScope scope, Runnable target)
      
      public final void run()
      
      public static void yield(ContinuationScope scope)
      
      public boolean isDone()
    } 
```

Continuation 实现了 Runnable 接口，构造时除了需要提供一个 Runnable 类型的参数以外，还需要提供一个 `java.lang.ContinuationScope` 的参数。ContinuationScope 顾名思义表示 Continuation 的范围。Continuation 可以被想象成是一个方法执行过程，方法可以调用其它方法。同时，方法执行也有一定的影响范围，如 try...catch 就规定了相应的范围。ContinuationScope 就起到了起到了相应的作用。

Continuation 有两个最重要的方法：run 和 yield。run 方法首次被调用时，就会执行 `Runnable target` 的 run 方法。但是，在调用了 yield 方法后，再次调用 run 方法，Continuation 就不会从头执行，而是从 yield 的位置开始执行。

为了更形象的理解，下面看一个例子：

```java
    Continuation con = new Continuation(SCOPE, () -> {
      println("A");
      Continuation.yield(SCOPE);
      println("B");
      Continuation.yield(SCOPE);
      println("C");
    });

    con.run();
    con.run();
    con.run(); 
```

输出结果：
```bash
    A
    B
    C 
```
上面的例子非常简单：创建一个 Continuation，其 Runnable target 打印 A、B、C，并在其中 yield 两次。创建之后调用三次 run() 方法。如果这样执行一个普通的 Runnable，那应该打印三次 A、B、C，一共打印九次。而 Continuation 在 yield 之后执行 run，会从 yield 的位置往后执行，而不是从头开始。

Continuation yield 类似 Thread 的 yield，但前者需要显式调用 run 方法恢复执行。

在 Project Loom 之后，`LockSupport` 的 `park` 操作将变为：

```java
    public class LockSupport {
      var strand = Strands.currentStrand();
      if (strand instanceof Fiber) {
        Continuation.yield(FIBER_SCOPE);
      } else {
        Unsafe.park(false, 0L);
      }
    } 
```

# 七、展望

Java 作为使用率最高的编程软件，在包括后端开发、手机应用开发、大数据等众多领域均有广泛应用。但毕竟是一门诞生 20 多年的编程语言，存在一些现在看来设计上的不足和受到后来者的挑战都是正常。但必须说明，我们口中的 Java 并非一门单纯的编程语言。而应该被视为 Java 语言 + JVM + Java 类库三部分组成。这三部分中，毫无疑问，JVM 是基础。但 JVM 设计之初就并非和 Java 语言紧密绑定，紧密绑定的只是字节码。由任何编程语言编译得到的合法字节码都能运行在 JVM 之上。这使得 Java 语言层面设计的不足可有其它编程语言解决，于是出现了 Groovy、Scala、Kotlin、Clojure 等众多 JVM 语言。这些语言很大程度上弥补了 Java 的不足。

但像多线程这样的技术，由于和底层虚机和操作系统有千丝万缕的联系，想要彻底改进，绕不开底层优化。这就是 Project Loom 出现的原因。相信 Project Loom 技术会将 Java 的并发能力提升至和 Golang 一样的水平，而付出的成本只是对现有项目的少量改动。

Azul 的 Deputy CTO Simon Ritter 曾透露 Project Loom 很可能在 Java 13 时发布。究竟能不能赶上 Java 13，这个不可知，好在 Java 13 的特性还未完全确定，说不定可以 Project Loom 可以赶上末班车。

就算 Project Loom 没能和 Java 13 一起发布。但目前反应式编程的趋势也非常明显。随着新版本的 Spring 和 Kotlin 的发布，反应式编程的使用、调试变得越来越简单。Dubbo 也明确表示在 3.0 中将会支持 Project Reactor。R2DBC 在不久的未来也会支持 MySQL。因此，Java 异步编程将快速发展，在易用性方面迅速赶上甚至超过 Go。

另一方面，开发人员也不要将自己局限在某种特定技术上，对各种技术都保持开放的态度是开发人员技能不断提高的前提。只会简单说某某语言、某某技术比其它技术更好的技术人员永远不会成为出色的技术人员。 
 [https://www.jianshu.com/p/5db701a764cb](https://www.jianshu.com/p/5db701a764cb)
