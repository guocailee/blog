---
{"dg-publish":true,"permalink":"/Program/Java/Spring Cloud Stream 体系及原理介绍 Format's Notes/","noteIcon":"","created":"2024-05-22T16:17:54.145+08:00"}
---

[Spring Cloud Stream](https://spring.io/projects/spring-cloud-stream) 在 Spring Cloud 体系内用于构建高度可扩展的基于事件驱动的微服务。

Spring Cloud Stream 本身内容很多，而且它还有很多外部的依赖，想要熟悉 Spring Cloud Stream，需要了解以下知识：

-   Spring Framework(Spring Messaging, Spring Environment)
-   Spring Boot Actuator
-   Spring Boot Externalized Configuration
-   Spring Retry
-   Spring Integration
-   Spring Cloud Stream

这篇文章的目的主要是介绍 Spring Cloud Stream。面对这么多知识点，会尽量以最简单的方式带大家了解 Spring Cloud Steam(后面会以 SCS 代替 Spring Cloud Stream)。

再了解 SCS 之前，必须要了解 Spring Messaging 和 Spring Integration 这两个项目，首先我们看下这两个项目。

### Spring Messaging

Spring Messaging 模块是 Spring Framework 中的一个模块，其作用就是统一消息的编程模型。

比如消息 `Messaging` 对应的模型就包括一个消息体 Payload 和消息头 Header:

[![](https://docs.spring.io/spring-integration/reference/html/images/message.jpg)
](https://docs.spring.io/spring-integration/reference/html/images/message.jpg)

```java
package org.springframework.messaging;
public interface Message<T> {
	T getPayload();
	MessageHeaders getHeaders();
}
```

消息通道 `MessageChannel` 用于接收消息，调用 `send` 方法可以将消息发送至该消息通道中 :

[![](https://docs.spring.io/spring-integration/reference/html/images/channel.jpg)
](https://docs.spring.io/spring-integration/reference/html/images/channel.jpg)

```java
@FunctionalInterface
public interface MessageChannel {
	long INDEFINITE_TIMEOUT = -1;
	default boolean send(Message<?> message) {
		return send(message, INDEFINITE_TIMEOUT);
	}
	boolean send(Message<?> message, long timeout);
}
```

消息通道里的消息如何被消费呢？ 由消息通道的子接口可订阅的消息通道 `SubscribableChannel` 实现，被 `MessageHandler` 消息处理器所订阅:

```java
public interface SubscribableChannel extends MessageChannel {
	boolean subscribe(MessageHandler handler);
	boolean unsubscribe(MessageHandler handler);
}
```

`MessageHandler` 真正地消费 / 处理消息:

```java
@FunctionalInterface
public interface MessageHandler {
	void handleMessage(Message<?> message) throws MessagingException;
}
```

Spring Messaging 内部在消息模型的基础上衍生出了其它的一些功能，如

1.  消息接收参数及返回值处理：消息接收参数处理器 `HandlerMethodArgumentResolver` 配合 `@Header`, `@Payload` 等注解使用；消息接收后的返回值处理器 `HandlerMethodReturnValueHandler` 配合 `@SendTo` 注解使用
2.  消息体内容转换器 `MessageConverter`
3.  统一抽象的消息发送模板 `AbstractMessageSendingTemplate`
4.  消息通道拦截器 `ChannelInterceptor`
5.  …

### [](#Spring-Integration "Spring Integration")Spring Integration

[Spring Integration](https://docs.spring.io/spring-integration/reference/html/index.html) 提供了 Spring 编程模型的扩展用来支持企业集成模式 ([Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/))。

Spring Integration 是对 Spring Messaging 的扩展。它提出了不少新的概念，包括消息的路由 `MessageRoute`、消息的分发 `MessageDispatcher`、消息的过滤 `Filter`、消息的转换 `Transformer`、消息的聚合 `Aggregator`、消息的分割 `Splitter` 等等。同时还提供了包括 `MessageChannel` 的实现 `DirectChannel`、`ExecutorChannel`、`PublishSubscribeChannel` 等; `MessageHandler` 的实现 `MessageFilter`、`ServiceActivatingHandler`、`MethodInvokingSplitter` 等内容。

几种消息的处理方式：

消息的分割：

[![](https://raw.githubusercontent.com/fangjian0423/blogimages/master/images/msg-splitter.png)
](https://raw.githubusercontent.com/fangjian0423/blogimages/master/images/msg-splitter.png)

消息的聚合：

[![](https://raw.githubusercontent.com/fangjian0423/blogimages/master/images/msg-aggr.png)
](https://raw.githubusercontent.com/fangjian0423/blogimages/master/images/msg-aggr.png)

消息的过滤：

[![](https://raw.githubusercontent.com/fangjian0423/blogimages/master/images/msg-filter.png)
](https://raw.githubusercontent.com/fangjian0423/blogimages/master/images/msg-filter.png)

消息的分发：

[![](https://raw.githubusercontent.com/fangjian0423/blogimages/master/images/msg-dispatcher.png)
](https://raw.githubusercontent.com/fangjian0423/blogimages/master/images/msg-dispatcher.png)

我们以一个最简单的例子来尝试一下 Spring Integration:

```java
SubscribableChannel messageChannel = new DirectChannel(); // 1

messageChannel.subscribe(msg -> { // 2
  System.out.println("receive: " + msg.getPayload());
});

messageChannel.send(MessageBuilder.withPayload("msg from alibaba").build()); // 3
```

这段代码解释：

1.  构造一个可订阅的消息通道 `messageChannel`
2.  使用 `MessageHandler` 去消费这个消息通道里的消息
3.  发送一条消息到这个消息通道，消息最终被消息通道里的 `MessageHandler` 所消费

最后控制台打印出: `receive: msg from alibaba`

`DirectChannel` 内部有个 `UnicastingDispatcher` 类型的消息分发器，会分发到对应的消息通道 `MessageChannel` 中，从名字也可以看出来，`UnicastingDispatcher` 是个单播的分发器，只能选择一个消息通道。那么如何选择呢? 内部提供了 `LoadBalancingStrategy` 负载均衡策略，默认只有轮询的实现，可以进行扩展。

我们对上段代码做一点修改，使用多个 `MessageHandler` 去处理消息：

```java
SubscribableChannel messageChannel = new DirectChannel();

messageChannel.subscribe(msg -> {
  System.out.println("receive1: " + msg.getPayload());
});

messageChannel.subscribe(msg -> {
  System.out.println("receive2: " + msg.getPayload());
});

messageChannel.send(MessageBuilder.withPayload("msg from alibaba").build());
messageChannel.send(MessageBuilder.withPayload("msg from alibaba").build());
```

由于 `DirectChannel` 内部的消息分发器是 `UnicastingDispatcher` 单播的方式，并且采用轮询的负载均衡策略，所以这里两次的消费分别对应这两个 `MessageHandler`。控制台打印出：

```yaml
receive1: msg from alibaba
receive2: msg from alibaba
```

既然存在单播的消息分发器 `UnicastingDispatcher`，必然也会存在广播的消息分发器，那就是 `BroadcastingDispatcher`，它被 `PublishSubscribeChannel` 这个消息通道所使用。广播消息分发器会把消息分发给所有的 `MessageHandler`：

```java
SubscribableChannel messageChannel = new PublishSubscribeChannel();

messageChannel.subscribe(msg -> {
  System.out.println("receive1: " + msg.getPayload());
});

messageChannel.subscribe(msg -> {
  System.out.println("receive2: " + msg.getPayload());
});

messageChannel.send(MessageBuilder.withPayload("msg from alibaba").build());
messageChannel.send(MessageBuilder.withPayload("msg from alibaba").build());
```

发送两个消息，都被所有的 `MessageHandler` 所消费。控制台打印：

```yaml
receive1: msg from alibaba
receive2: msg from alibaba
receive1: msg from alibaba
receive2: msg from alibaba
```

### [](#Spring-Cloud-Stream "Spring Cloud Stream")Spring Cloud Stream

SCS 在 Spring Integration 的基础上进行了封装，提出了 `Binder`, `Binding`, `@EnableBinding`, `@StreamListener` 等概念; 与 Spring Boot Actuator 整合，提供了 `/bindings`, `/channels` endpoint; 与 Spring Boot Externalized Configuration 整合，提供了 `BindingProperties`, `BinderProperties` 等外部化配置类; 增强了消息发送失败的和消费失败情况下的处理逻辑等功能。

SCS 是 Spring Integration 的加强，同时与 Spring Boot 体系进行了融合，也是 Spring Cloud Bus 的基础。它屏蔽了底层消息中间件的实现细节，希望以统一的一套 API 来进行消息的发送 / 消费，底层消息中间件的实现细节由各消息中间件的 Binder 完成。

[![](https://raw.githubusercontent.com/spring-cloud/spring-cloud-stream/master/docs/src/main/asciidoc/images/SCSt-overview.png)
](https://raw.githubusercontent.com/spring-cloud/spring-cloud-stream/master/docs/src/main/asciidoc/images/SCSt-overview.png)

`Binder` 是提供与外部消息中间件集成的组件，会构造 `Binding`，提供了 2 个方法分别是 `bindConsumer` 和 `bindProducer` 分别用于构造生产者和消费者。目前官方的实现有 [Rabbit Binder](https://github.com/spring-cloud/spring-cloud-stream-binder-rabbit) 和 [Kafka Binder](https://github.com/spring-cloud/spring-cloud-stream-binder-kafka)， [Spring Cloud Alibaba](https://github.com/spring-cloud-incubator/spring-cloud-alibaba) 内部实现了 [RocketMQ Binder](https://github.com/spring-cloud-incubator/spring-cloud-alibaba/tree/master/spring-cloud-stream-binder-rocketmq)。

`Binding` 从图中可以看出，它是连接应用程序跟消息中间件的桥梁，用于消息的消费和生产。

我们来看一个最简单的使用 RocketMQ Binder 的例子，然后分析一下它的底层处理原理。

启动类及消息的发送：

```java
@SpringBootApplication
@EnableBinding({ Source.class, Sink.class }) // 1
public class SendAndReceiveApplication {
  
	public static void main(String[] args) {
		SpringApplication.run(SendAndReceiveApplication.class, args);
	}
  
    @Bean // 2
	public CustomRunner customRunner() {
		return new CustomRunner();
	}

	public static class CustomRunner implements CommandLineRunner {

		@Autowired
		private Source source;

		@Override
		public void run(String... args) throws Exception {
			int count = 5;
			for (int index = 1; index <= count; index++) {
				source.output().send(MessageBuilder.withPayload("msg-" + index).build()); // 3
			}
		}
	}
}
```

消息的接收：

```java
@Service
public class StreamListenerReceiveService {

	@StreamListener(Sink.INPUT) // 4
	public void receiveByStreamListener1(String receiveMsg) {
		System.out.println("receiveByStreamListener: " + receiveMsg);
	}

}
```

这段代码很简单，没有涉及到 RocketMQ 相关的代码，消息的发送和接收都是基于 SCS 体系完成的。如果想切换成 rabbitmq 或 kafka，只需修改配置文件即可，代码无需修改。

我们分析这段代码的原理:

1.  `@EnableBinding` 对应的两个接口属性 `Source` 和 `Sink` 是 SCS 内部提供的。SCS 内部会基于 `Source` 和 `Sink` 构造 `BindableProxyFactory`，且对应的 output 和 input 方法返回的 MessageChannel 是 `DirectChannel`。output 和 input 方法修饰的注解对应的 value 是配置文件中 binding 的 name。

```java
public interface Source {
	String OUTPUT = "output";
	@Output(Source.OUTPUT)
	MessageChannel output();
}
public interface Sink {
	String INPUT = "input";
	@Input(Sink.INPUT)
	SubscribableChannel input();
}
```

配置文件里 bindings 的 name 为 output 和 input，对应 `Source` 和 `Sink` 接口的方法上的注解里的 value：  

```plain
spring.cloud.stream.bindings.output.destination=test-topic
spring.cloud.stream.bindings.output.content-type=text/plain
spring.cloud.stream.rocketmq.bindings.output.producer.group=demo-group

spring.cloud.stream.bindings.input.destination=test-topic
spring.cloud.stream.bindings.input.content-type=text/plain
spring.cloud.stream.bindings.input.group=test-group1
```

2.  构造 `CommandLineRunner`，程序启动的时候会执行 `CustomRunner` 的 `run` 方法
3.  调用 `Source` 接口里的 output 方法获取 `DirectChannel`，并发送消息到这个消息通道中。这里跟之前 Spring Integration 章节里的代码一致
    -   Source 里的 output 发送消息到 `DirectChannel` 消息通道之后会被 `AbstractMessageChannelBinder#SendingHandler` 这个 `MessageHandler` 处理，然后它会委托给 `AbstractMessageChannelBinder#createProducerMessageHandler` 创建的 MessageHandler 处理 (该方法由不同的消息中间件实现)
    -   不同的消息中间件对应的 `AbstractMessageChannelBinder#createProducerMessageHandler` 方法返回的 MessageHandler 内部会把 Spring Message 转换成对应中间件的 Message 模型并发送到对应中间件的 broker
4.  使用 `@StreamListener` 进行消息的订阅。请注意，注解里的 `Sink.input` 对应的值是 “input”，会根据配置文件里 binding 对应的 name 为 input 的值进行配置
    -   不同的消息中间件对应的 `AbstractMessageChannelBinder#createConsumerEndpoint` 方法会使用 Consumer 订阅消息，订阅到消息后内部会把中间件对应的 Message 模型转换成 Spring Message
    -   消息转换之后会把 Spring Message 发送至 name 为 input 的消息通道中
    -   `@StreamListener` 对应的 `StreamListenerMessageHandler` 订阅了 name 为 input 的消息通道，进行了消息的消费

这个过程文字描述有点啰嗦，用一张图总结一下 (黄色部分涉及到各消息中间件的 Binder 实现以及 MQ 基本的订阅发布功能)：

[![](https://raw.githubusercontent.com/fangjian0423/blogimages/master/images/scs-progress.png)
](https://raw.githubusercontent.com/fangjian0423/blogimages/master/images/scs-progress.png)

SCS 章节的最后，我们来看一段 SCS 关于消息的处理方式的一段代码：

```java
@StreamListener(value = Sink.INPUT, condition = "headers['index']=='1'")
public void receiveByHeader(Message msg) {
  System.out.println("receive by headers['index']=='1': " + msg);
}

@StreamListener(value = Sink.INPUT, condition = "headers['index']=='9999'")
public void receivePerson(@Payload Person person) {
  System.out.println("receive Person: " + person);
}

@StreamListener(value = Sink.INPUT)
public void receiveAllMsg(String msg) {
  System.out.println("receive allMsg by StreamListener. content: " + msg);
}

@StreamListener(value = Sink.INPUT)
public void receiveHeaderAndMsg(@Header("index") String index, Message msg) {
  System.out.println("receive by HeaderAndMsg by StreamListener. content: " + msg);
}
```

有没有发现这段代码跟 Spring MVC Controller 中接收请求的代码很像? 实际上他们的架构都是类似的，Spring MVC 对于 Controller 中参数和返回值的处理类分别是 `org.springframework.web.method.support.HandlerMethodArgumentResolver`、 `org.springframework.web.method.support.HandlerMethodReturnValueHandler`。

Spring Messaging 中对于参数和返回值的处理类之前也提到过，分别是 `org.springframework.messaging.handler.invocation.HandlerMethodArgumentResolver`、`org.springframework.messaging.handler.invocation.HandlerMethodReturnValueHandler`。

它们的类名一模一样，甚至内部的方法名也一样。

### [](#总结 "总结")总结

这个是 SCS 体系相关类说明的总结：

[![](https://raw.githubusercontent.com/fangjian0423/blogimages/master/images/scs-architecture.png)
](https://raw.githubusercontent.com/fangjian0423/blogimages/master/images/scs-architecture.png)

关于 SCS 以及 RocketMQ Binder 更多相关的示例，可以参考 [RocketMQ Binder Demos](https://github.com/fangjian0423/rocketmq-binder-demo)，包含了消息的聚合、分割、过滤；消息异常处理；消息标签、sql 过滤；同步、异步消费等等。
