---
{"dg-publish":true,"permalink":"/Program/Framework/国内酒店交易DDD应用与实践——代码篇/","noteIcon":"","created":"2025-03-06T21:28:25.970+08:00"}
---


作者：Qunar技术沙龙

    
    


**本期讲解 DDD 代码落地实践内容**

这一篇、我们将结合 DDD 概念[[Program/Framework/国内酒店交易DDD应用与实践——理论篇\|国内酒店交易DDD应用与实践——理论篇]]和酒店交易应用场景，重点讲述用户接口层、应用层、领域层、资源层 DDD 四层架构中，酒店交易是如何落地 DDD 实施，包括实施过程中、代码层面需要注意的标准和原则；

# 一、酒店交易采用的四层 DDD 松散架构 

![](https://static001.geekbang.org/infoq/3b/3b78dbbfd26895d6a807cf9d332f8adf.png)

DDD 四层架构和每一层作用职责

1. 用户接口层:用户接口层只用于处理用户显示和用户请求和数据的渲染展现，不应该包含领域或业务逻辑(业务逻辑在领域设计中有明确的定义、业务指流程或行为或一个动作发生一定要产生推动下一个行为或动作的过程是业务,页面渲染、检索类并不属于业务逻辑)；
    
2. 应用层:应用服务位于应用层中。应用服务用于控制持久化事务和安全认证。应用服务本身并不处理业务逻辑，应用服务是轻量级，主要用于协调对领域对象的操作， 应用服务的通常用途是：接收来自用户界面的输入参数，再通过资源库获取到聚合实例，然后执行相应的命令操作；
    
3. 领域层:或称为模型层，负责表达业务概念和业务状态信息以及业务规则。该领域（问题域）完成所有复杂的业务知识抽象和规则定义；
    
4. 基础设施层:一是为领域模型提供持久化机制；
    

分层架构一般可以分为严格分层架构和松散分层架构两种

严格分层架构，某层只能与直接位于其下方的层发生耦合；

松散分层架构，则允许任意上方层与任意下方层发生耦合；

由于用户界面层和应用服务通常需要与基础设施打交道， 许多系统都是基于松散分层架构的

# 二、DDD 实施与代码落地

**简要版订单生单流程、支付流程、完结流程、逆向流程**

![](https://static001.geekbang.org/infoq/94/946521448a83941be75f98de5a83b2bf.png)

**我们结合订单正向、逆向业务流程阐述 DDD 实施过程**

## 1、酒店交易通用语言构建

![](https://static001.geekbang.org/infoq/84/8432b690f9091ef58b657fa940a4b3c8.png)

## 2、四层架构代码实现

### 2.1 用户接口层代码实现示例(订单)

代码示例为、订单协议层(用户接口层)实现，核心逻辑处理，用于接受命令或处理用户显示和数据的渲染展现逻辑；

因为是后端服务、我们把对外提供协议放置用户接口层实现；

注入 订单应用层 对象、提供接受 生单命令执行、提交订单命令执行....

![](https://static001.geekbang.org/infoq/e4/e4d45fa722b07df8e5578c70e7432324.png)

### 2.2 应用层代码实现示例(订单)

**代码示例为、订单应用层实现，核心逻辑，接收来自用户界面的输入参数，再通过资源库获取到聚合实例，然后执行相应的命令操作。**

**原则** 应用服务位于应用层中，用于控制持久化事务和安全认证。本身并不处理业务逻辑，应用服务是轻量级，主要用于协调对领域对象的操作。

![](https://static001.geekbang.org/infoq/3c/3caa67d08c0df3e65b68d8384049b5f9.png)

### 2.3 领域层代码实现示例(订单)

**订单聚合根代码实现示例**

聚合根原则与标准

1. 聚合根 是实体，具备唯一标识，有独立的生命周期，一个聚合只有一个聚合根，聚合根负责聚合之内实体和值对象组织和协调变更，不允许绕过聚合根直接访问聚合内实体进行交互，这样会导致聚合处于不一致状态，违背聚合不变条件的原则,在设计聚合根时要遵守一定原则；
    
2. 聚合根到聚合根：通过 ID 关联；
    
3. 聚合根到其内部的实体，直接对象引用；
    
4. 聚合根到值对象，直接对象引用；
    
5. 公开行为接口 显示的表达领域概念，作为外部通信的入口，只有聚合根才能被外部访问到，聚合根维护聚合的内部一致性；
    
6. 领域服务表示一个无状态的操作，它用于实现特定于某个领域的任务。当某个动作不适合放在聚合对象上时，可以使用领域服务。
    

图示：多个实体、资金实体、主单实体、产品值对象构建订单聚合根、结合聚合规则和标准、业务流程、抽象出 提交订单行为、确认订单行为、取消订单行为等

仅包括 @Getter 我们需要通过一些简单的措施来包装标识不被修改，我们可以将 setter 向用户隐藏，确保标识存在的情况下不会被更新等。聚合根创建可以采用工厂模式处理。

![](https://static001.geekbang.org/infoq/0f/0f3680772fd1efe5011175a61bbde356.png)

**工厂代码实现示例**

要点与职责:

保护内部状态对于领域结构很重要，使用 getter&setter 方会公开聚合的内部会导致聚合处于不一致状态。

大型复杂业务系统，实体和聚合创建过程很复杂，很难去通过简单构造器方式来创建对象；

工厂模式决解了这个问题，图示，订单聚合根采用工厂模式处理。当建立聚合根时，其他对象(实体、值对象)可以自动创建，保证聚合不变条件、同时对外封装复杂性

![](https://static001.geekbang.org/infoq/ca/caed178d581f9a10386461c7b06e8d34.png)

**资金实体代码实现示例**

实体原则与标准

1. 唯一标识
    
2. 生命周期有状态
    
3. 体现相关业务行为
    

图示：结合酒店交易资金业务、抽象资金实体行为(房费行为、支付行为、佣金行为等)、满足业务流程；

实体仅包括 @Getter 我们需要通过一些简单的措施来包装标识不被修改，我们必须将 setter 向用户隐藏，确保标识存在的情况下不会被更新等。

![](https://static001.geekbang.org/infoq/23/230aa2e645fbf4602cb53ff124724c36.png)

### 2.4 资源层代码实现示例

**订单仓储代码实现示例**

图示：订单仓储层实施过程、仅需要提供 2 个方法

方法 1：提供 ID 获取聚合根方法

方法 2：聚合根统一存储、保障聚合根内部一致性;

**资源库**

我们的实践是资源库作为业务与数据的隔离层，屏蔽底层数据表细节， 同时完成 PO 与 DO 的转化。

DO 与 PO 的转化带来的好处是领域层不会直接依赖底层实现，便于后续更换底层实现或功能迁移。

资源库接口定义在领域层，接口实现在基础设施层。

仓储用于保存和获取聚合对象，应该将仓储看作一个对象的集合，保障聚合根内部一致性;而不是数据库的 CRUD；

我们基于领域不变条件来设计聚合，每次请求应该在一个聚合实例上执行一个命令方法 聚合设计都有 4 个基本原则：

1. 【一致性原则】
    
2. 【保持不变条件】
    
3. 【一个事务只修改一个聚合】
    
4. 【小聚合】 
    

**聚合内一致性原则与代码实现**

以聚合根作为对外的接口人，聚合内部保证数据强一致性，代码中通过版本控制以及同时更新的方式保证

![](https://static001.geekbang.org/infoq/23/230aa2e645fbf4602cb53ff124724c36.png)

  

![](https://static001.geekbang.org/infoq/99/9979c6ceea7392c7e81a11738a0b67c7.png)

**聚合间一致性原则与代码实现**

以酒店交易逆向流程为例、用户发起申请取消、订单审核、执行退款、过程中涉及履约聚合、订单聚合、支付聚合多个聚合根处理；

![](https://static001.geekbang.org/infoq/9f/9f0c68f616d14d4a4199433f4cf94947.png)

多个聚合根如何协调保障一致性?先了解标准和原则

**基准原则**

一个事务只能更新一个聚合，如业务流程需要系统支持多个聚合共同完成一个业务，那么聚合间如何处理?

实现最终一致性的技术手段有很多，可以采用 MQ 机制，在事件消费的流程中处理对其他聚合的变更。

既可以满足一个事务只修改一个聚合这一基准原则，又可以借助消息队列中间件提供的失败重试机制，完成最终一致性、保证业务准确性。

**参考：订单取消审核成功后，审核聚合完成审核业务后会发送审核成功事件，订单聚合监听审核成功时间取消订单，支付聚合监听到取消事件后进行退款流程；**

**审核领域代码实现**

第一步执行审核聚合根审核取消行为

第二步请求审核资源库完成聚合根存储

第三步发送**审核结果事件**

![](https://static001.geekbang.org/infoq/8a/8a493e76fb33e0273040705cb6f73b5d.png)

第四步 订单领域监听该**审核结果事件**

第五步 通过唯一标识、获取订单聚合根

第六步 执行订单取消行为方法

第七步 请求订单资源库完成订单聚合更新

第八步 发送**订单取消事件**

![](https://static001.geekbang.org/infoq/a0/a0bb5e83d42b2e58014fc281c3ad2ab2.png)

第九步 支付领域监听**订单取消事件**

第十步 执行支付领域退款领域服务、完成逆向流程

![](https://static001.geekbang.org/infoq/e2/e2bf8fe3c048dfdce2892a8159415e83.png)

# 小结

本篇文章内容、结合 DDD 领域概念、交易业务流程、讲解了酒店交易采用四层 DDD 架构代码实施落地、每一层架构应用原则和标准，包括：

1. 酒店交易通用语言构建展示
    
2. 用户接口层代码实现、
    
3. 应用层代码实现和原则;
    
4. 订单聚合根代码实现、如何保障聚合内一致性、工厂代码实现示例
    
5. 资金实体代码实现
    
6. 订单仓储代码实现、仓储层方法定义标准
    
7. 聚合间一致性原则与代码实现、如何保障多个聚合一致性