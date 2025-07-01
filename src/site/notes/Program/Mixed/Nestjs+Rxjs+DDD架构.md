---
{"dg-publish":true,"permalink":"/Program/Mixed/Nestjs+Rxjs+DDD架构/","noteIcon":"","created":"2025-06-03T15:38:30.563+08:00"}
---

在 NestJS 中结合 RxJS 实现领域驱动设计（DDD）架构，需要将响应式编程与分层架构思想结合。以下是一个详细的设计方案和代码示例：

### 分层架构设计
```
src/
├── domain/                # 领域层（纯业务逻辑）
│   ├── models/           # 聚合根/实体/值对象
│   ├── repositories/     # 仓储接口定义
│   ├── services/         # 领域服务
│   └── events/           # 领域事件定义
├── application/          # 应用层（用例协调）
│   ├── services/         # 应用服务
│   └── dtos/            # 数据传输对象
├── infrastructure/       # 基础设施层
│   ├── db/              # 数据库相关（TypeORM）
│   ├── events/          # 事件实现（RxJS）
│   └── external/        # 外部服务集成
└── interfaces/           # 接口层
    ├── rest/            # REST API 控制器
    └── graphql/         # GraphQL 解析器
```

### 核心实现示例

#### 1. 领域层 - 值对象
```typescript
// domain/models/email.ts
export class Email {
  private readonly value: string;

  constructor(email: string) {
    if (!this.validate(email)) {
      throw new Error('Invalid email format');
    }
    this.value = email;
  }

  private validate(email: string): boolean {
    const re = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return re.test(email);
  }

  toString(): string {
    return this.value;
  }
}
```

#### 2. 领域层 - 聚合根
```typescript
// domain/models/user.aggregate.ts
import { Email } from './email';

export class User {
  private readonly id: string;
  private email: Email;
  private isActive = false;

  constructor(id: string, email: Email) {
    this.id = id;
    this.email = email;
  }

  activate() {
    this.isActive = true;
    // 触发领域事件
    DomainEvents.publish(new UserActivatedEvent(this.id));
  }

  // 工厂方法
  static create(userData: { email: string }) {
    return new User(
      uuidv4(),
      new Email(userData.email)
    );
  }
}
```

#### 3. 应用服务（使用 RxJS 管道）
```typescript
// application/services/user.service.ts
@Injectable()
export class UserService {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly eventBus: EventBus // RxJS Subject
  ) {}

  registerUser(command: RegisterUserCommand): Observable<User> {
    return from(this.userRepository.exists(command.email)).pipe(
      switchMap(exists => {
        if (exists) throw new ConflictException('Email already exists');
        
        const user = User.create(command);
        return from(this.userRepository.save(user)).pipe(
          tap(savedUser => {
            // 发布领域事件
            this.eventBus.publish(
              new UserRegisteredEvent(savedUser.id, savedUser.email)
            );
          }),
          map(savedUser => UserDto.fromDomain(savedUser))
        );
      }),
      catchError(error => throwError(() => new Error(error)))
    );
  }
}
```

#### 4. 基础设施层 - 事件总线（RxJS 实现）
```typescript
// infrastructure/events/event-bus.ts
@Injectable()
export class EventBus {
  private subject$ = new Subject<DomainEvent>();

  publish<T extends DomainEvent>(event: T): void {
    this.subject$.next(event);
  }

  ofType<T extends DomainEvent>(eventType: Type<T>): Observable<T> {
    return this.subject$.pipe(
      filter(event => event instanceof eventType),
      map(event => event as T)
    );
  }
}
```

#### 5. 领域事件处理
```typescript
// infrastructure/events/handlers/user-events.handler.ts
@Injectable()
export class UserEventsHandler {

  constructor(
    private readonly mailService: MailService,
    private readonly analytics: AnalyticsService
  ) {
    this.eventBus.ofType(UserRegisteredEvent).subscribe(event => {
      this.handleUserRegistered(event);
    });
  }

  private handleUserRegistered(event: UserRegisteredEvent) {
    // 使用 RxJS 合并多个异步操作
    merge(
      from(this.mailService.sendWelcomeEmail(event.email)),
      from(this.analytics.trackUserRegistration(event.userId))
    ).pipe(
      catchError(err => {
        console.error('Event handling failed:', err);
        return of();
      })
    ).subscribe();
  }
}
```

#### 6. 响应式仓储实现（TypeORM + RxJS）
```typescript
// infrastructure/db/repositories/user.repository.ts
@Injectable()
export class TypeOrmUserRepository implements UserRepository {
  constructor(
    @InjectRepository(UserEntity)
    private readonly ormRepo: Repository<UserEntity>
  ) {}

  findById(id: string): Observable<User> {
    return from(this.ormRepo.findOneBy({ id })).pipe(
      map(entity => UserMapper.toDomain(entity))
    );
  }

  save(user: User): Observable<void> {
    const entity = UserMapper.toPersistence(user);
    return from(this.ormRepo.save(entity)).pipe(
      mapTo(undefined)
    );
  }
}
```

### 关键设计点

1. **响应式数据流**：
   - 使用 RxJS 操作符处理异步流程（switchMap、catchError 等）
   - 领域事件处理采用 Observable 流式处理
   - 数据库操作返回 Observable

2. **领域隔离**：
   - 领域模型不依赖任何框架
   - 业务规则封装在值对象和聚合根中
   - 领域服务只处理核心业务逻辑

3. **事件驱动**：
   - 使用 RxJS Subject 实现事件总线
   - 领域事件触发跨聚合操作
   - 支持事件溯源（Event Sourcing）扩展

4. **依赖方向**：
   ``` 
   接口层 → 应用层 → 领域层 ← 基础设施层
   ```

5. **复杂事务处理**：
```typescript
// 使用 RxJS 处理分布式事务
handleOrderCreation() {
  return this.paymentService.processPayment().pipe(
    mergeMap(() => this.inventoryService.reserveStock()),
    mergeMap(() => this.orderRepository.save()),
    catchError(error => this.compensateActions())
  );
}
```

### 优势分析

1. **响应式优势**：
   - 背压处理
   - 非阻塞 I/O
   - 复杂异步流程组合
   - 细粒度的取消控制

2. **DDD 优势**：
   - 清晰的领域边界
   - 可扩展的架构
   - 更好的业务逻辑封装
   - 方便进行领域演化

3. **NestJS 整合**：
   - 依赖注入系统管理复杂依赖
   - 模块化组织代码
   - 拦截器、管道等与 RxJS 天然集成

这种架构特别适合需要处理复杂业务规则、高并发场景和分布式系统的应用，能够有效分离关注点并保持代码的可维护性。