---
{"dg-publish":true,"permalink":"/Program/Framework/DDD落地 - 京东/","noteIcon":""}
---

## 前言

最近有空会跟同事讨论DDD架构的实践落地的情况，但真实情况是，实际中对于[领域驱动设计](https://www.zhihu.com/search?q=%E9%A2%86%E5%9F%9F%E9%A9%B1%E5%8A%A8%E8%AE%BE%E8%AE%A1&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3003007311%7D)中的实体，值对象，聚合根，领域事件这些战术类的实践落地，每个人理解依然因人而异，大概率是因为这些概念还是有一些抽象，同时有有别于传统的MVC架构开发。

在此，通过小demo的方式跟大家分享一下我对DDD中战术层级的理解，算是抛砖引玉，该理解仅代表我个人在现阶段的一个理解，也可能未来随着业务经验深入，还会有不同的理解。

既然说是小demo，还是要从业务场景出发，也就是我最熟知的电商业务场景说起。但是该篇文章里， 我会简化一些实际业务场景中的复杂度，通过最小颗粒度的demo，来反映实践过程中的基本问题。

## 一个简单的demo业务场景

话不多说，我先抛出我自己假设的一个业务场景，就是我们熟知的电商网站下单购物的场景。具体细节如下：

### 1. 实体

• 商品：拥有唯一标识、名称、价格、库存等属性。

• 订单：拥有唯一标识、下单时间、状态等属性。订单包含多个订单项。

### 2. 值对象

• 地址：拥有省、市、区、详细地址等属性。

### 3. 域事件

• 订单创建事件：当用户下单时触发该事件，包含订单信息、[商品信息](https://www.zhihu.com/search?q=%E5%95%86%E5%93%81%E4%BF%A1%E6%81%AF&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3003007311%7D)等数据。

• 订单支付事件：当用户完成支付时触发该事件，包含订单信息、支付金额等数据。

• 订单发货事件：当商家发货时触发该事件，包含订单信息、[快递公司](https://www.zhihu.com/search?q=%E5%BF%AB%E9%80%92%E5%85%AC%E5%8F%B8&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3003007311%7D)、[快递单号](https://www.zhihu.com/search?q=%E5%BF%AB%E9%80%92%E5%8D%95%E5%8F%B7&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3003007311%7D)等数据。

### 4. 聚合根

• 商品聚合根：包含商品实体和相关的值对象，负责商品的创建、修改、查询等操作。

• 订单聚合根：包含订单实体和相关的值对象，负责订单的创建、修改、查询等操作。

### 5. 对外接口服务

• 创建订单接口：用户提交购买请求后，系统创建相应的订单，并触发订单创建事件。

• 支付订单接口：用户完成支付后，系统更新订单状态，并触发订单支付事件。

• 发货接口：商家发货后，系统更新订单状态，并触发订单发货事件。

• 查询订单接口：用户可以根据[订单号](https://www.zhihu.com/search?q=%E8%AE%A2%E5%8D%95%E5%8F%B7&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3003007311%7D)等条件查询自己的订单信息。

该demo中，商品和订单是两个核心领域概念，分别由对应的聚合根负责管理。同时，通过定义领域事件，实现了不同业务场景下的数据更新和通知。最后，对外提供了一组简单的接口服务，方便系统的使用和扩展。

## demo的java代码实现

好了，有了以上我们对业务场景的充分剖析，确定了子域，接下来我们该写我们的代码。

1.商品[实体类](https://www.zhihu.com/search?q=%E5%AE%9E%E4%BD%93%E7%B1%BB&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3003007311%7D)：

```java
// 省略getter/setter方法
public class Product {
    private Long id;
    private String name;
    private BigDecimal price;
    private Integer stock;
}
```

2\. 订单实体类

```java
// 省略getter/setter方法
public class Order {
    private Long id;
    private LocalDateTime createTime;
    private Integer status;
    private List orderItems;
}
```

3\. 订单项实体类

```java
// 省略getter/setter方法
public class OrderItem {
    private Long id;
    private Product product;
    private Integer quantity;
    private BigDecimal price;
}
```

4\. 地址值对象

```java
// 省略getter/setter方法 
public class Address {
    private String province;
    private String city;
    private String district;
    private String detail;
}
```

5\. 领域事件类

```java
//订单创建领域事件
public class OrderCreatedEvent {
    private Order order;
    private List orderItems;

    public OrderCreatedEvent(Order order, List orderItems) {
        this.order = order;
        this.orderItems = orderItems;
    }
}


//订单支付领域事件
public class OrderPaidEvent {
    private Order order;
    private BigDecimal amount;

    public OrderPaidEvent(Order order, BigDecimal amount) {
        this.order = order;
        this.amount = amount;
    }
}

//订单
public class OrderShippedEvent {
    private Order order;
    private String expressCompany;
    private String expressNo;

    public OrderShippedEvent(Order order, String expressCompany, String expressNo) {
        this.order = order;
        this.expressCompany = expressCompany;
        this.expressNo = expressNo;
    }
}
```

6\. 商品聚合根

```java
public class ProductAggregate {
    private ProductService productService;

    public void createProduct(Product product) {
        productService.create(product);
    }

    public void updateProduct(Product product) {
        productService.update(product);
    }

    public void deleteProduct(Long productId) {
        productService.delete(productId);
    }

    public Product getProductById(Long productId) {
        return productService.getById(productId);
    }
}
```

7\. 订单聚合根

```java
public class OrderAggregate {
    private OrderService orderService;

    public void createOrder(Order order, List orderItems) {
        orderService.create(order);
        // 触发订单创建事件 
        DomainEventPublisher.publish(new OrderCreatedEvent(order, orderItems));
    }

    public void payOrder(Long orderId, BigDecimal amount) {
        orderService.pay(orderId, amount);
        // 触发订单支付事件
        DomainEventPublisher.publish(new OrderPaidEvent(orderService.getById(orderId), amount));
    }

    public void shipOrder(Long orderId, String expressCompany, String expressNo) {
        orderService.ship(orderId, expressCompany, expressNo);
        // 触发订单发货事件 
        DomainEventPublisher.publish(new OrderShippedEvent(orderService.getById(orderId), expressCompany, expressNo));
    }

    public Order getOrderById(Long orderId) {
        return orderService.getById(orderId);
    }
}
```

## 总结

通过以上demo，对于实体和值对象，大家会很好理解，并且很直观。但是， 我额外想重点解释一下聚合根和领域事件的概念

1\. 聚合根

从上面的demo可以看出，在合根类中，我们定义了商品和订单的增、删、查等操作，并且为订单定义了创建订单、支付订单、发货等[业务逻辑](https://www.zhihu.com/search?q=%E4%B8%9A%E5%8A%A1%E9%80%BB%E8%BE%91&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3003007311%7D)代码。

聚合根是一个对象，它代表一组相关联的对象的整体。在聚合根内部，可以包含多个实体对象和值对象。聚合根通常可以通过唯一[标识符](https://www.zhihu.com/search?q=%E6%A0%87%E8%AF%86%E7%AC%A6&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3003007311%7D)来进行识别和访问。它是整个聚合的管理者，负责维护聚合之内的一致性，并协调各个实体对象之间的关系。聚合根通常具有丰富的行为和操作，可以对聚合内部的对象进行复杂的操作。

所以说，真正的聚合根内的方法是基于充血模型封装的，而不是仅仅是对对象的数据封装。在聚合根中，对象不仅封装了数据，还包含了相应的行为和业务逻辑。这意味着在一个聚合根中，对象可以自己处理自己的业务逻辑，而不需要外部的控制。就如同demo中所写的那样，订单对象可能包含一些关于订单处理和交付的方法，如确认订单、取消订单、发货等。

2\. 领域事件

领域事件是DDD中最重要的概念之一，他是解决子域之间耦合的重要手段，因为它们提供了一种将领域概念和业务语言转化为代码的方法。当一个领域事件发生时，它会触发一些操作，这些操作可能会更改系统的状态，也可能会导致其他领域事件的发生。通过对领域事件进行建模，我们可以更好地了解业务过程并设计出更加符合实际需求的系统。

在DDD中，领域事件通常由三个部分组成：

1.事件名称：这个名称应该能够简洁明了地描述事件所代表的业务意义。

2.相关数据：这些数据包含了事件发生时与事件相关的所有信息。例如，在一个[电子商务系统](https://www.zhihu.com/search?q=%E7%94%B5%E5%AD%90%E5%95%86%E5%8A%A1%E7%B3%BB%E7%BB%9F&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3003007311%7D)中，如果订单被提交，则订单信息以及买家和卖家的信息都应该包括在该事件中。

3.发送者和接收者：发送者通常是触发事件的对象，接收者则是事件处理的对象。

领域事件在DDD中有很多用途。例如，它们可以用来触发其他[业务流程](https://www.zhihu.com/search?q=%E4%B8%9A%E5%8A%A1%E6%B5%81%E7%A8%8B&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3003007311%7D)、更新数据库或通知其他子系统。它们还可以用于解决一些复杂的业务逻辑问题，例如并发、数据同步和错误处理等等。

总之，领域事件是DDD架构中非常重要的概念，它可以帮助我们更好地理解业务过程，设计出更加符合实际需求的系统，并提高系统的可维护性和可扩展性。