---
{"dg-publish":true,"permalink":"/Program/Framework/Java DDD目录结构/","noteIcon":""}
---

在设计 Java DDD（领域驱动设计）项目的目录结构时，遵循清晰的分层原则、模块化的设计、以及业务逻辑和技术细节分离的思想是非常重要的。下面是一个典型的 Java DDD 项目的目录设计示例：

### 1. 按设计模块的目录分层

```bash
/src
 ├── main
 │    ├── java
 │    │    └── com.example.myapp
 │    │         ├── application
 │    │         ├── domain
 │    │         ├── infrastructure
 │    │         └── interfaces
 │    └── resources
 └── test
```

### 2. 详细的层级说明

#### 1. `application`（应用层）
应用层负责处理业务用例（Use Case），它是应用的入口，协调领域模型和基础设施来完成业务需求。这里的代码主要是服务层代码，包括服务接口和实现等。
   
   - **示例目录结构：**
     ```bash
     application
     ├── service
     ├── command
     ├── dto
     └── handler
     ```
   - **内容：**
     - `service`：应用服务，负责编排和处理业务流程，不包含具体业务逻辑。
     - `command`：与业务流程相关的命令对象。
     - `dto`：数据传输对象（Data Transfer Object），用于传递数据。
     - `handler`：命令处理器，处理具体的应用逻辑。

#### 2. `domain`（领域层）
领域层是 DDD 的核心部分，封装了所有的业务逻辑和规则。它应该独立于应用层和基础设施层。所有的业务实体、聚合根、值对象、领域事件和领域服务都放在这里。

   - **示例目录结构：**
     ```bash
     domain
     ├── model
     │   ├── entity
     │   ├── aggregate
     │   ├── valueobject
     │   └── event
     ├── repository
     ├── service
     └── exception
     ```
   - **内容：**
     - `entity`：领域实体，包含具有唯一标识的业务对象。
     - `aggregate`：聚合根，管理业务逻辑的聚合实体。
     - `valueobject`：值对象，没有唯一标识但有业务意义的数据类型。
     - `event`：领域事件，代表业务行为的发生。
     - `repository`：仓储接口，定义对领域实体的持久化操作接口。
     - `service`：领域服务，处理跨实体的业务逻辑。
     - `exception`：领域异常，定义与业务相关的异常。

#### 3. `infrastructure`（基础设施层）
基础设施层负责与外部技术服务的交互，包括数据库、消息队列、外部 API 等。它实现了领域层中定义的仓储接口，或者提供其他技术服务。

   - **示例目录结构：**
     ```bash
     infrastructure
     ├── repository
     ├── messaging
     ├── configuration
     └── externalservice
     ```
   - **内容：**
     - `repository`：仓储实现类，负责具体的数据库交互。
     - `messaging`：消息队列或事件总线的实现。
     - `configuration`：与外部系统集成的配置类，如数据库、缓存等配置。
     - `externalservice`：外部服务接口的实现。

#### 4. `interfaces`（接口层/适配器层）
接口层负责处理用户交互、API 接口、Web 请求等。这里主要是控制器、REST API 接口或者用户界面的适配器。

   - **示例目录结构：**
     ```bash
     interfaces
     ├── web
     ├── rest
     └── dto
     ```
   - **内容：**
     - `web`：MVC 控制器，用于处理 Web 端的用户请求。
     - `rest`：REST API 控制器，用于处理外部系统的 API 请求。
     - `dto`：用于传输的输入和输出 DTO。

#### 5. `test`（测试）
所有的单元测试、集成测试都应放在 `test` 目录中。每一层都应有相应的测试，以确保功能的完整性。

   - **内容：**
     - 单元测试：针对 `domain` 层的业务逻辑。
     - 集成测试：针对 `infrastructure` 层的数据库操作、消息队列等技术细节。
     - 接口测试：针对 `interfaces` 层的 API 接口。


如果要按照业务进行分层设计 Java DDD 项目的目录结构，可以采用**按模块划分业务领域**的方式，将每个业务领域模块化，每个模块内部再遵循 DDD 分层的原则（如领域层、应用层、基础设施层、接口层等）。这种设计适合复杂系统，因为每个业务模块可以独立开发、测试和维护。

### 3. 按业务模块划分的顶层目录

```bash
/src
 ├── main
 │    ├── java
 │    │    └── com.example.myapp
 │    │         ├── user
 │    │         │    ├── application
 │    │         │    ├── domain
 │    │         │    ├── infrastructure
 │    │         │    └── interfaces
 │    │         ├── order
 │    │         │    ├── application
 │    │         │    ├── domain
 │    │         │    ├── infrastructure
 │    │         │    └── interfaces
 │    │         ├── product
 │    │         │    ├── application
 │    │         │    ├── domain
 │    │         │    ├── infrastructure
 │    │         │    └── interfaces
 └── test
```

### 4. 详细的分层设计（以 `user` 业务模块为例）

#### 1. `user/application`（用户模块的应用层）
应用层负责用户模块的业务逻辑调度和用例执行。通常包含应用服务、命令、DTO等。

```bash
user/application
├── service
├── command
├── dto
└── handler
```
- `service`：用户相关的应用服务，如创建用户、修改用户信息。
- `command`：与用户相关的业务操作命令，如 `CreateUserCommand`。
- `dto`：数据传输对象（如 `UserDTO`），用于与接口层通信。
- `handler`：处理应用逻辑的处理器。

#### 2. `user/domain`（用户模块的领域层）
领域层封装用户模块的核心业务逻辑和规则，领域模型、实体、值对象、聚合根和仓储接口等都在此定义。

```bash
user/domain
├── model
│   ├── entity
│   ├── aggregate
│   ├── valueobject
│   └── event
├── repository
├── service
└── exception
```
- `entity`：用户实体（如 `User`），具有唯一标识。
- `aggregate`：聚合根（如 `UserAggregate`），管理相关的领域对象。
- `valueobject`：值对象，如 `Address`（地址）等没有唯一标识的对象。
- `event`：领域事件，用于传播用户模块的业务行为（如用户注册事件）。
- `repository`：用户的仓储接口，用于定义数据持久化的行为。
- `service`：领域服务，跨多个实体处理业务逻辑。
- `exception`：与用户模块相关的领域异常。

#### 3. `user/infrastructure`（用户模块的基础设施层）
基础设施层实现与技术细节相关的工作，如数据库交互、外部服务调用等。实现了 `repository` 接口，也可以包含消息队列、缓存等基础设施相关的逻辑。

```bash
user/infrastructure
├── repository
├── messaging
├── configuration
└── externalservice
```
- `repository`：用户的仓储实现类，可能使用 JPA 或其他持久化框架。
- `messaging`：消息处理机制，如事件驱动架构中的消息发布订阅。
- `configuration`：基础设施层相关的配置，如数据库、外部API配置等。
- `externalservice`：外部系统服务接口的实现。

#### 4. `user/interfaces`（用户模块的接口层/适配器层）
接口层是与外部系统交互的地方，负责处理 REST API、用户界面或其他形式的输入/输出。控制器、DTO 和转换器通常位于此处。

```bash
user/interfaces
├── web
├── rest
└── dto
```
- `web`：用于处理 Web 应用程序中的用户交互请求（如 MVC 控制器）。
- `rest`：REST API 控制器，处理来自外部系统的请求，如 `UserController`。
- `dto`：与前端或外部系统交互的数据传输对象。

### 5. 其他业务模块
其他业务模块，如 `order`（订单模块）和 `product`（产品模块），与 `user` 模块类似，每个模块都遵循相同的结构和 DDD 分层设计原则。

### 6. 通用模块
在这种设计中，如果有一些模块是跨业务领域的，比如通用的配置、工具类、共享的领域事件或外部服务，可以在单独的模块中定义。示例：

```bash
common
├── configuration
├── messaging
├── util
└── exception
```

- `configuration`：全局配置项，如数据库配置、缓存配置等。
- `messaging`：事件总线或消息系统的配置与实现。
- `util`：工具类和通用方法。
- `exception`：全局异常处理。

### 7. 总结
按业务划分目录结构的好处是**高度模块化**和**业务解耦**，每个业务模块都是独立的单元，便于团队协作开发和维护。同时，每个模块内部依然遵循 DDD 的分层设计，确保项目结构清晰可扩展。

这种目录结构能够有效地管理复杂的业务逻辑，支持微服务架构下的演进，也有助于应对复杂系统的业务变化和技术扩展。