---
{"dg-publish":true,"permalink":"/Program/MessageQueue/一文带你搞定RabbitMQ延迟队列/","noteIcon":"","created":"2025-03-06T21:28:25.977+08:00"}
---

#MQ #RabbitMQ 
## 一、说明 

在上一篇中，介绍了[[Program/MessageQueue/一文带你搞定RabbitMQ死信队列\|一文带你搞定RabbitMQ死信队列]]，何时使用以及如何使用 RabbitMQ 的死信队列。相信通过上一篇的学习，对于死信队列已经有了更多的了解，这一篇的内容也跟死信队列息息相关，如果你还不了解死信队列，那么建议你先进行上一篇文章的阅读。

这一篇里，我们将继续介绍 RabbitMQ 的高级特性，通过本篇的学习，你将收获：

1.  什么是延时队列
2.  延时队列使用场景
3.  RabbitMQ 中的 TTL
4.  如何利用 RabbitMQ 来实现延时队列

## 二、本文大纲

以下是本文大纲：

[![](https://i.loli.net/2019/07/28/5d3d74d99699d43032.png)
](https://i.loli.net/2019/07/28/5d3d74d99699d43032.png)

本文阅读前，需要对 RabbitMQ 以及死信队列有一个简单的了解。

## 三、什么是延时队列

`延时队列`，首先，它是一种队列，队列意味着内部的元素是`有序`的，元素出队和入队是有方向性的，元素从一端进入，从另一端取出。

其次，`延时队列`，最重要的特性就体现在它的`延时`属性上，跟普通的队列不一样的是，`普通队列中的元素总是等着希望被早点取出处理，而延时队列中的元素则是希望被在指定时间得到取出和处理`，所以延时队列中的元素是都是带时间属性的，通常来说是需要被处理的消息或者任务。

简单来说，延时队列就是用来存放需要在指定时间被处理的元素的队列。

## 四、延时队列使用场景

那么什么时候需要用延时队列呢？考虑一下以下场景：

1.  订单在十分钟之内未支付则自动取消。
2.  新创建的店铺，如果在十天内都没有上传过商品，则自动发送消息提醒。
3.  账单在一周内未支付，则自动结算。
4.  用户注册成功后，如果三天内没有登陆则进行短信提醒。
5.  用户发起退款，如果三天内没有得到处理则通知相关运营人员。
6.  预定会议后，需要在预定的时间点前十分钟通知各个与会人员参加会议。

这些场景都有一个特点，需要在某个事件发生之后或者之前的指定时间点完成某一项任务，如：发生订单生成事件，在十分钟之后检查该订单支付状态，然后将未支付的订单进行关闭；发生店铺创建事件，十天后检查该店铺上新商品数，然后通知上新数为 0 的商户；发生账单生成事件，检查账单支付状态，然后自动结算未支付的账单；发生新用户注册事件，三天后检查新注册用户的活动数据，然后通知没有任何活动记录的用户；发生退款事件，在三天之后检查该订单是否已被处理，如仍未被处理，则发送消息给相关运营人员；发生预定会议事件，判断离会议开始是否只有十分钟了，如果是，则通知各个与会人员。

看起来似乎使用定时任务，一直轮询数据，每秒查一次，取出需要被处理的数据，然后处理不就完事了吗？如果数据量比较少，确实可以这样做，比如：对于 “如果账单一周内未支付则进行自动结算” 这样的需求，如果对于时间不是严格限制，而是宽松意义上的一周，那么每天晚上跑个定时任务检查一下所有未支付的账单，确实也是一个可行的方案。但对于数据量比较大，并且时效性较强的场景，如：“订单十分钟内未支付则关闭“，短期内未支付的订单数据可能会有很多，活动期间甚至会达到百万甚至千万级别，对这么庞大的数据量仍旧使用轮询的方式显然是不可取的，很可能在一秒内无法完成所有订单的检查，同时会给数据库带来很大压力，无法满足业务要求而且性能低下。

更重要的一点是，不！优！雅！

没错，作为一名有追求的程序员，始终应该追求更优雅的架构和更优雅的代码风格，写代码要像写诗一样优美。【滑稽】

这时候，延时队列就可以闪亮登场了，以上场景，正是延时队列的用武之地。

既然`延时队列`可以解决很多特定场景下，带时间属性的任务需求，那么如何构造一个延时队列呢？接下来，本文将介绍如何用 RabbitMQ 来实现延时队列。

## 五、RabbitMQ 中的 TTL

在介绍延时队列之前，还需要先介绍一下 RabbitMQ 中的一个高级特性——`TTL（Time To Live）`。

`TTL`是什么呢？`TTL`是 RabbitMQ 中一个消息或者队列的属性，表明`一条消息或者该队列中的所有消息的最大存活时间`，单位是毫秒。换句话说，如果一条消息设置了 TTL 属性或者进入了设置 TTL 属性的队列，那么这条消息如果在 TTL 设置的时间内没有被消费，则会成为 “死信”（至于什么是死信，请翻看上一篇）。如果同时配置了队列的 TTL 和消息的 TTL，那么较小的那个值将会被使用。

那么，如何设置这个 TTL 值呢？有两种方式，第一种是在创建队列的时候设置队列的 “x-message-ttl” 属性，如下：

```java

Map<String, Object> args = new HashMap<String, Object>();
args.put("x-message-ttl", 6000);
channel.queueDeclare(queueName, durable, exclusive, autoDelete, args);

```

这样所有被投递到该队列的消息都最多不会存活超过 6s。

另一种方式便是针对每条消息设置 TTL，代码如下：

```java


AMQP.BasicProperties.Builder builder = new AMQP.BasicProperties.Builder();
builder.expiration("6000");
AMQP.BasicProperties properties = builder.build();
channel.basicPublish(exchangeName, routingKey, mandatory, properties, "msg body".getBytes());


```

这样这条消息的过期时间也被设置成了 6s。

但这两种方式是有区别的，**如果设置了队列的 TTL 属性，那么一旦消息过期，就会被队列丢弃，而第二种方式，消息即使过期，也不一定会被马上丢弃，因为消息是否过期是在即将投递到消费者之前判定的，如果当前队列有严重的消息积压情况，则已过期的消息也许还能存活较长时间。** 

另外，还需要注意的一点是，如果不设置 TTL，表示消息永远不会过期，如果将 TTL 设置为 0，则表示除非此时可以直接投递该消息到消费者，否则该消息将会被丢弃。

## 六、如何利用 RabbitMQ 实现延时队列

前一篇里介绍了如果设置死信队列，前文中又介绍了 TTL，至此，利用 RabbitMQ 实现延时队列的两大要素已经集齐，接下来只需要将它们进行调和，再加入一点点调味料，延时队列就可以新鲜出炉了。

想想看，`延时队列`，不就是想要消息延迟多久被处理吗，TTL 则刚好能让消息在延迟多久之后成为死信，另一方面，成为死信的消息都会被投递到死信队列里，这样只需要消费者一直消费死信队列里的消息就万事大吉了，因为里面的消息都是希望被立即处理的消息。

从下图可以大致看出消息的流向：

[![](https://i.loli.net/2019/07/28/5d3d743143ecc85643.png)
](https://i.loli.net/2019/07/28/5d3d743143ecc85643.png)

生产者生产一条延时消息，根据需要延时时间的不同，利用不同的 routingkey 将消息路由到不同的延时队列，每个队列都设置了不同的 TTL 属性，并绑定在同一个死信交换机中，消息过期后，根据 routingkey 的不同，又会被路由到不同的死信队列中，消费者只需要监听对应的死信队列进行处理即可。

下面来看代码：

先声明交换机、队列以及他们的绑定关系：

```java


@Configuration
public class RabbitMQConfig {

    public static final String DELAY_EXCHANGE_NAME = "delay.queue.demo.business.exchange";
    public static final String DELAY_QUEUEA_NAME = "delay.queue.demo.business.queuea";
    public static final String DELAY_QUEUEB_NAME = "delay.queue.demo.business.queueb";
    public static final String DELAY_QUEUEA_ROUTING_KEY = "delay.queue.demo.business.queuea.routingkey";
    public static final String DELAY_QUEUEB_ROUTING_KEY = "delay.queue.demo.business.queueb.routingkey";
    public static final String DEAD_LETTER_EXCHANGE = "delay.queue.demo.deadletter.exchange";
    public static final String DEAD_LETTER_QUEUEA_ROUTING_KEY = "delay.queue.demo.deadletter.delay_10s.routingkey";
    public static final String DEAD_LETTER_QUEUEB_ROUTING_KEY = "delay.queue.demo.deadletter.delay_60s.routingkey";
    public static final String DEAD_LETTER_QUEUEA_NAME = "delay.queue.demo.deadletter.queuea";
    public static final String DEAD_LETTER_QUEUEB_NAME = "delay.queue.demo.deadletter.queueb";

    // 声明延时Exchange
    @Bean("delayExchange")
    public DirectExchange delayExchange(){
        return new DirectExchange(DELAY_EXCHANGE_NAME);
    }

    // 声明死信Exchange
    @Bean("deadLetterExchange")
    public DirectExchange deadLetterExchange(){
        return new DirectExchange(DEAD_LETTER_EXCHANGE);
    }

    // 声明延时队列A 延时10s
    // 并绑定到对应的死信交换机
    @Bean("delayQueueA")
    public Queue delayQueueA(){
        Map<String, Object> args = new HashMap<>(2);
        // x-dead-letter-exchange    这里声明当前队列绑定的死信交换机
        args.put("x-dead-letter-exchange", DEAD_LETTER_EXCHANGE);
        // x-dead-letter-routing-key  这里声明当前队列的死信路由key
        args.put("x-dead-letter-routing-key", DEAD_LETTER_QUEUEA_ROUTING_KEY);
        // x-message-ttl  声明队列的TTL
        args.put("x-message-ttl", 6000);
        return QueueBuilder.durable(DELAY_QUEUEA_NAME).withArguments(args).build();
    }

    // 声明延时队列B 延时 60s
    // 并绑定到对应的死信交换机
    @Bean("delayQueueB")
    public Queue delayQueueB(){
        Map<String, Object> args = new HashMap<>(2);
        // x-dead-letter-exchange    这里声明当前队列绑定的死信交换机
        args.put("x-dead-letter-exchange", DEAD_LETTER_EXCHANGE);
        // x-dead-letter-routing-key  这里声明当前队列的死信路由key
        args.put("x-dead-letter-routing-key", DEAD_LETTER_QUEUEB_ROUTING_KEY);
        // x-message-ttl  声明队列的TTL
        args.put("x-message-ttl", 60000);
        return QueueBuilder.durable(DELAY_QUEUEB_NAME).withArguments(args).build();
    }

    // 声明死信队列A 用于接收延时10s处理的消息
    @Bean("deadLetterQueueA")
    public Queue deadLetterQueueA(){
        return new Queue(DEAD_LETTER_QUEUEA_NAME);
    }

    // 声明死信队列B 用于接收延时60s处理的消息
    @Bean("deadLetterQueueB")
    public Queue deadLetterQueueB(){
        return new Queue(DEAD_LETTER_QUEUEB_NAME);
    }

    // 声明延时队列A绑定关系
    @Bean
    public Binding delayBindingA(@Qualifier("delayQueueA") Queue queue,
                                    @Qualifier("delayExchange") DirectExchange exchange){
        return BindingBuilder.bind(queue).to(exchange).with(DELAY_QUEUEA_ROUTING_KEY);
    }

    // 声明业务队列B绑定关系
    @Bean
    public Binding delayBindingB(@Qualifier("delayQueueB") Queue queue,
                                    @Qualifier("delayExchange") DirectExchange exchange){
        return BindingBuilder.bind(queue).to(exchange).with(DELAY_QUEUEB_ROUTING_KEY);
    }

    // 声明死信队列A绑定关系
    @Bean
    public Binding deadLetterBindingA(@Qualifier("deadLetterQueueA") Queue queue,
                                    @Qualifier("deadLetterExchange") DirectExchange exchange){
        return BindingBuilder.bind(queue).to(exchange).with(DEAD_LETTER_QUEUEA_ROUTING_KEY);
    }

    // 声明死信队列B绑定关系
    @Bean
    public Binding deadLetterBindingB(@Qualifier("deadLetterQueueB") Queue queue,
                                      @Qualifier("deadLetterExchange") DirectExchange exchange){
        return BindingBuilder.bind(queue).to(exchange).with(DEAD_LETTER_QUEUEB_ROUTING_KEY);
    }
} 


```

接下来，创建两个消费者，分别对两个死信队列的消息进行消费：

```java


@Slf4j
@Component
public class DeadLetterQueueConsumer {

    @RabbitListener(queues = DEAD_LETTER_QUEUEA_NAME)
    public void receiveA(Message message, Channel channel) throws IOException {
        String msg = new String(message.getBody());
        log.info("当前时间：{},死信队列A收到消息：{}", new Date().toString(), msg);
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
    }

    @RabbitListener(queues = DEAD_LETTER_QUEUEB_NAME)
    public void receiveB(Message message, Channel channel) throws IOException {
        String msg = new String(message.getBody());
        log.info("当前时间：{},死信队列B收到消息：{}", new Date().toString(), msg);
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
    }
} 


```

然后是消息的生产者：

```java


@Component
public class DelayMessageSender {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void sendMsg(String msg, DelayTypeEnum type){
        switch (type){
            case DELAY_10s:
                rabbitTemplate.convertAndSend(DELAY_EXCHANGE_NAME, DELAY_QUEUEA_ROUTING_KEY, msg);
                break;
            case DELAY_60s:
                rabbitTemplate.convertAndSend(DELAY_EXCHANGE_NAME, DELAY_QUEUEB_ROUTING_KEY, msg);
                break;
        }
    }
}


```

接下来，我们暴露一个 web 接口来生产消息：

```java


@Slf4j
@RequestMapping("rabbitmq")
@RestController
public class RabbitMQMsgController {

    @Autowired
    private DelayMessageSender sender;

    @RequestMapping("sendmsg")
    public void sendMsg(String msg, Integer delayType){
        log.info("当前时间：{},收到请求，msg:{},delayType:{}", new Date(), msg, delayType);
        sender.sendMsg(msg, Objects.requireNonNull(DelayTypeEnum.getDelayTypeEnumByValue(delayType)));
    }
} 


```

准备就绪，启动！

打开 rabbitMQ 的[管理后台](http://localhost:15672)，可以看到我们刚才创建的交换机和队列信息：

[![](https://i.loli.net/2019/07/28/5d3d54e15534398514.png)
](https://i.loli.net/2019/07/28/5d3d54e15534398514.png)

[![](https://i.loli.net/2019/07/28/5d3d54e17df8183993.png)
](https://i.loli.net/2019/07/28/5d3d54e17df8183993.png)

[![](https://i.loli.net/2019/07/28/5d3d54e16952546955.png)
](https://i.loli.net/2019/07/28/5d3d54e16952546955.png)

接下来，我们来发送几条消息，[http://localhost:8080/rabbitmq/sendmsg?msg=testMsg1&delayType=1](http://localhost:8080/rabbitmq/sendmsg?msg=testMsg1&delayType=1) [http://localhost:8080/rabbitmq/sendmsg?msg=testMsg2&delayType=2](http://localhost:8080/rabbitmq/sendmsg?msg=testMsg2&delayType=2)

日志如下：

```


2019-07-28 16:02:19.813  INFO 3860 --- [nio-8080-exec-9] c.m.d.controller.RabbitMQMsgController   : 当前时间：Sun Jul 28 16:02:19 CST 2019,收到请求，msg:testMsg1,delayType:1
2019-07-28 16:02:19.815  INFO 3860 --- [nio-8080-exec-9] .l.DirectReplyToMessageListenerContainer : SimpleConsumer [queue=amq.rabbitmq.reply-to, consumerTag=amq.ctag-o-qPpkWIkRm73DIrOIVhig identity=766339] started
2019-07-28 16:02:25.829  INFO 3860 --- [ntContainer#1-1] c.m.d.mq.DeadLetterQueueConsumer         : 当前时间：Sun Jul 28 16:02:25 CST 2019,死信队列A收到消息：testMsg1
2019-07-28 16:02:41.326  INFO 3860 --- [nio-8080-exec-1] c.m.d.controller.RabbitMQMsgController   : 当前时间：Sun Jul 28 16:02:41 CST 2019,收到请求，msg:testMsg2,delayType:2
2019-07-28 16:03:41.329  INFO 3860 --- [ntContainer#0-1] c.m.d.mq.DeadLetterQueueConsumer         : 当前时间：Sun Jul 28 16:03:41 CST 2019,死信队列B收到消息：testMsg2


```

第一条消息在 6s 后变成了死信消息，然后被消费者消费掉，第二条消息在 60s 之后变成了死信消息，然后被消费掉，这样，一个还算 ok 的延时队列就打造完成了。

不过，等等，如果这样使用的话，岂不是每增加一个新的时间需求，就要新增一个队列，这里只有 6s 和 60s 两个时间选项，如果需要一个小时后处理，那么就需要增加 TTL 为一个小时的队列，如果是预定会议室然后提前通知这样的场景，岂不是要增加无数个队列才能满足需求？？

嗯，仔细想想，事情并不简单。

## 七、RabbitMQ 延时队列优化 

显然，需要一种更通用的方案才能满足需求，那么就只能将 TTL 设置在消息属性里了。我们来试一试。

增加一个延时队列，用于接收设置为任意延时时长的消息，增加一个相应的死信队列和 routingkey：

```java


@Configuration
public class RabbitMQConfig {

    public static final String DELAY_EXCHANGE_NAME = "delay.queue.demo.business.exchange";
    public static final String DELAY_QUEUEC_NAME = "delay.queue.demo.business.queuec";
    public static final String DELAY_QUEUEC_ROUTING_KEY = "delay.queue.demo.business.queuec.routingkey";
    public static final String DEAD_LETTER_EXCHANGE = "delay.queue.demo.deadletter.exchange";
    public static final String DEAD_LETTER_QUEUEC_ROUTING_KEY = "delay.queue.demo.deadletter.delay_anytime.routingkey";
    public static final String DEAD_LETTER_QUEUEC_NAME = "delay.queue.demo.deadletter.queuec";

    // 声明延时Exchange
    @Bean("delayExchange")
    public DirectExchange delayExchange(){
        return new DirectExchange(DELAY_EXCHANGE_NAME);
    }

    // 声明死信Exchange
    @Bean("deadLetterExchange")
    public DirectExchange deadLetterExchange(){
        return new DirectExchange(DEAD_LETTER_EXCHANGE);
    }

    // 声明延时队列C 不设置TTL
    // 并绑定到对应的死信交换机
    @Bean("delayQueueC")
    public Queue delayQueueC(){
        Map<String, Object> args = new HashMap<>(3);
        // x-dead-letter-exchange    这里声明当前队列绑定的死信交换机
        args.put("x-dead-letter-exchange", DEAD_LETTER_EXCHANGE);
        // x-dead-letter-routing-key  这里声明当前队列的死信路由key
        args.put("x-dead-letter-routing-key", DEAD_LETTER_QUEUEC_ROUTING_KEY);
        return QueueBuilder.durable(DELAY_QUEUEC_NAME).withArguments(args).build();
    }

    // 声明死信队列C 用于接收延时任意时长处理的消息
    @Bean("deadLetterQueueC")
    public Queue deadLetterQueueC(){
        return new Queue(DEAD_LETTER_QUEUEC_NAME);
    }

    // 声明延时列C绑定关系
    @Bean
    public Binding delayBindingC(@Qualifier("delayQueueC") Queue queue,
                                 @Qualifier("delayExchange") DirectExchange exchange){
        return BindingBuilder.bind(queue).to(exchange).with(DELAY_QUEUEC_ROUTING_KEY);
    }

    // 声明死信队列C绑定关系
    @Bean
    public Binding deadLetterBindingC(@Qualifier("deadLetterQueueC") Queue queue,
                                      @Qualifier("deadLetterExchange") DirectExchange exchange){
        return BindingBuilder.bind(queue).to(exchange).with(DEAD_LETTER_QUEUEC_ROUTING_KEY);
    }
} 


```

增加一个死信队列 C 的消费者：

```java


@RabbitListener(queues = DEAD_LETTER_QUEUEC_NAME)
public void receiveC(Message message, Channel channel) throws IOException {
    String msg = new String(message.getBody());
    log.info("当前时间：{},死信队列C收到消息：{}", new Date().toString(), msg);
    channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
}` 


```

再次启动！然后访问：[http://localhost:8080/rabbitmq/delayMsg?msg=testMsg1delayTime=5000](http://localhost:8080/rabbitmq/delayMsg?msg=testMsg1delayTime=5000) 来生产消息，注意这里的单位是毫秒。

```


2019-07-28 16:45:07.033 INFO 31468 --- [nio-8080-exec-4] c.m.d.controller.RabbitMQMsgController   : 当前时间：Sun Jul 28 16:45:07 CST 2019,收到请求，msg:testMsg1,delayTime:5000
2019-07-28 16:45:11.694 INFO 31468 --- [nio-8080-exec-5] c.m.d.controller.RabbitMQMsgController   : 当前时间：Sun Jul 28 16:45:11 CST 2019,收到请求，msg:testMsg2,delayTime:5000
2019-07-28 16:45:12.048 INFO 31468 --- [ntContainer#1-1] c.m.d.mq.DeadLetterQueueConsumer         : 当前时间：Sun Jul 28 16:45:12 CST 2019,死信队列C收到消息：testMsg1
2019-07-28 16:45:16.709 INFO 31468 --- [ntContainer#1-1] c.m.d.mq.DeadLetterQueueConsumer         : 当前时间：Sun Jul 28 16:45:16 CST 2019,死信队列C收到消息：testMsg2 


```

看起来似乎没什么问题，但不要高兴的太早，在最开始的时候，就介绍过，如果使用在消息属性上设置 TTL 的方式，消息可能并不会按时 “死亡 “，因为 RabbitMQ 只会检查第一个消息是否过期，如果过期则丢到死信队列，索引如果第一个消息的延时时长很长，而第二个消息的延时时长很短，则第二个消息并不会优先得到执行。

实验一下：

```


`2019-07-28 16:49:02.957 INFO 31468 --- [nio-8080-exec-8] c.m.d.controller.RabbitMQMsgController   : 当前时间：Sun Jul 28 16:49:02 CST 2019,收到请求，msg:longDelayedMsg,delayTime:20000
2019-07-28 16:49:10.671 INFO 31468 --- [nio-8080-exec-9] c.m.d.controller.RabbitMQMsgController   : 当前时间：Sun Jul 28 16:49:10 CST 2019,收到请求，msg:shortDelayedMsg,delayTime:2000
2019-07-28 16:49:22.969 INFO 31468 --- [ntContainer#1-1] c.m.d.mq.DeadLetterQueueConsumer         : 当前时间：Sun Jul 28 16:49:22 CST 2019,死信队列C收到消息：longDelayedMsg
2019-07-28 16:49:22.970 INFO 31468 --- [ntContainer#1-1] c.m.d.mq.DeadLetterQueueConsumer         : 当前时间：Sun Jul 28 16:49:22 CST 2019,死信队列C收到消息：shortDelayedMsg` 


```

我们先发了一个延时时长为 20s 的消息，然后发了一个延时时长为 2s 的消息，结果显示，第二个消息会在等第一个消息成为死信后才会 “死亡 “。

## 八、利用 RabbitMQ 插件实现延迟队列 

上文中提到的问题，确实是一个硬伤，如果不能实现在消息粒度上添加 TTL，并使其在设置的 TTL 时间及时死亡，就无法设计成一个通用的延时队列。

那如何解决这个问题呢？不要慌，安装一个插件即可：[https://www.rabbitmq.com/community-plugins.html](https://www.rabbitmq.com/community-plugins.html) ，下载 rabbitmq_delayed_message_exchange 插件，然后解压放置到 RabbitMQ 的插件目录。

接下来，进入 RabbitMQ 的安装目录下的 sbin 目录，执行下面命令让该插件生效，然后重启 RabbitMQ。

```


rabbitmq-plugins enable rabbitmq_delayed_message_exchange 


```

然后，我们再声明几个 Bean：

```java


@Configuration
public class DelayedRabbitMQConfig {
    public static final String DELAYED_QUEUE_NAME = "delay.queue.demo.delay.queue";
    public static final String DELAYED_EXCHANGE_NAME = "delay.queue.demo.delay.exchange";
    public static final String DELAYED_ROUTING_KEY = "delay.queue.demo.delay.routingkey";

    @Bean
    public Queue immediateQueue() {
        return new Queue(DELAYED_QUEUE_NAME);
    }

    @Bean
    public CustomExchange customExchange() {
        Map<String, Object> args = new HashMap<>();
        args.put("x-delayed-type", "direct");
        return new CustomExchange(DELAYED_EXCHANGE_NAME, "x-delayed-message", true, false, args);
    }

    @Bean
    public Binding bindingNotify(@Qualifier("immediateQueue") Queue queue,
                                 @Qualifier("customExchange") CustomExchange customExchange) {
        return BindingBuilder.bind(queue).to(customExchange).with(DELAYED_ROUTING_KEY).noargs();
    }
} 


```

controller 层再添加一个入口：

```java


@RequestMapping("delayMsg2")
public void delayMsg2(String msg, Integer delayTime) {
    log.info("当前时间：{},收到请求，msg:{},delayTime:{}", new Date(), msg, delayTime);
    sender.sendDelayMsg(msg, delayTime);
}

```

消息生产者的代码也需要修改：

```java

public void sendDelayMsg(String msg, Integer delayTime) {
    rabbitTemplate.convertAndSend(DELAYED_EXCHANGE_NAME, DELAYED_ROUTING_KEY, msg, a ->{
        a.getMessageProperties().setDelay(delayTime);
        return a;
    });
}


```

最后，再创建一个消费者：

```java

@RabbitListener(queues = DELAYED_QUEUE_NAME)
public void receiveD(Message message, Channel channel) throws IOException {
    String msg = new String(message.getBody());
    log.info("当前时间：{},延时队列收到消息：{}", new Date().toString(), msg);
    channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
}


```

一切准备就绪，启动！然后分别访问以下链接：

```


`http://localhost:8080/rabbitmq/delayMsg2?msg=msg1&delayTime=20000
http://localhost:8080/rabbitmq/delayMsg2?msg=msg2&delayTime=2000` 


```

日志如下：

```


2019-07-28 17:28:13.729  INFO 25804 --- [nio-8080-exec-2] c.m.d.controller.RabbitMQMsgController   : 当前时间：Sun Jul 28 17:28:13 CST 2019,收到请求，msg:msg1,delayTime:20000
2019-07-28 17:28:20.607  INFO 25804 --- [nio-8080-exec-1] c.m.d.controller.RabbitMQMsgController   : 当前时间：Sun Jul 28 17:28:20 CST 2019,收到请求，msg:msg2,delayTime:2000
2019-07-28 17:28:22.624  INFO 25804 --- [ntContainer#1-1] c.m.d.mq.DeadLetterQueueConsumer         : 当前时间：Sun Jul 28 17:28:22 CST 2019,延时队列收到消息：msg2
2019-07-28 17:28:33.751  INFO 25804 --- [ntContainer#1-1] c.m.d.mq.DeadLetterQueueConsumer         : 当前时间：Sun Jul 28 17:28:33 CST 2019,延时队列收到消息：msg1


```

第二个消息被先消费掉了，符合预期。至此，RabbitMQ 实现延时队列的部分就完结了。

## 九、总结 

延时队列在需要延时处理的场景下非常有用，使用 RabbitMQ 来实现延时队列可以很好的利用 RabbitMQ 的特性，如：消息可靠发送、消息可靠投递、死信队列来保障消息至少被消费一次以及未被正确处理的消息不会被丢弃。另外，通过 RabbitMQ 集群的特性，可以很好的解决单点故障问题，不会因为单个节点挂掉导致延时队列不可用或者消息丢失。

当然，延时队列还有很多其它选择，比如利用 Java 的 DelayQueu，利用 Redis 的 zset，利用 Quartz 或者利用 kafka 的时间轮，这些方式各有特点，但就像炉石传说一般，这些知识就好比手里的卡牌，知道的越多，可以用的卡牌也就越多，遇到问题便能游刃有余，所以需要大量的知识储备和经验积累才能打造出更出色的卡牌组合，让自己解决问题的能力得到更好的提升。

但另一方面，随着时间的流逝和阅历的增长，越来越感觉到自己的能力有限，无法独自面对纷繁复杂且多变的业务需求，在很多方面需要其他人的协助才能很好的完成任务。也知道闻道有先后，术业有专攻，不会再狂妄自大，觉得自己能把所有事情都搞定，也将重心慢慢转移到研究如何有效的进行团队合作上来，我相信一个高度协调的团队永远比一个人战斗要更有价值。

花了一个周末的时间完成了这篇文章，文中所有的代码都上传到了 github，[https://github.com/MFrank2016/delayed-queue-demo 如有需要可以自行查阅，希望能对你有帮助，如果有错误的地方，欢迎指正，也欢迎关注我的公众号进行留言交流。](https://github.com/MFrank2016/delayed-queue-demo)

[![](https://i.loli.net/2019/07/14/5d2af6692a8f432182.png)

[«](https://www.cnblogs.com/mfrank/p/11184929.html) 上一篇： [【RabbitMQ】一文带你搞定 RabbitMQ 死信队列](https://www.cnblogs.com/mfrank/p/11184929.html "发布于 2019-07-14 17:32")  
[»](https://www.cnblogs.com/mfrank/p/11336770.html) 下一篇： [【最佳实践】如何优雅的进行重试](https://www.cnblogs.com/mfrank/p/11336770.html "发布于 2019-08-11 21:22")

