---
{"dg-publish":true,"permalink":"/Program/Java/SpringBoot Redis 分布式锁的正确实现方式/","noteIcon":"","created":"2024-05-22T16:17:54.146+08:00"}
---

#分布式

>作者:**码哥字节** ,   9年Java后端开发和架构经验，Infoq签约作者，51CTO Top红人，阿里云开发者社区专家博主，擅长 Redis，Spring，Kafka，MySQL，职场管理。愿大家拥抱硬核技术和对象，面向人民币编程。](#)

在说分布式锁之前，我们先说下为什么需要分布式锁。

在单机部署的时候，我们可以使用 Java 中提供的 JUC 锁机制避免多线程同时操作一个共享变量产生的安全问题。JUC 锁机制只能保证同一个 JVM 进程中的同一时刻只有一个线程操作共享资源。

一个应用部署多个节点，多个进程如果要修改同一个共享资源，为了避免操作乱序导致的并发安全问题，这个时候就需要引入分布式锁，**分布式锁就是用来控制同一时刻，只有一个 JVM 进程中的一个线程可以访问被保护的资源**。

[[Program/Framework/分布式#一、分布式锁\|分布式#一、分布式锁]]重要，然而很多公司的系统可能还在跑着有缺陷的分布式锁方案，其中不乏一些大型公司。

所以，码哥**今天分享一个正确 Redis 分布式锁代码实战**，让你一飞冲天，该代码可直接用于生产，不是简单的 demo。

温馨提示：如果你只想看代码实战部分，可直接翻到 Spring Boot 实战章节。

## **一、错误的分布式锁**
  
说正确方案之前，先来一个错误的，知道错在哪，才能意识到如何写正确。

在银行工作的小白老师，使用 Redis SET 指令实现加锁， 指令满足了当 key 不存在则设置 value，同时设置超时时间，并且满足原子语意。

  
```nginx
SET lockKey 1 NX PX expireTime
```

  

*   lockKey 表示锁的资源，value 设置成 1。
*   NX：表示只有 lockKey 不存在的时候才能 SET 成功，从而保证只有一个客户端可以获得锁。
*   PX expireTime 设置锁的超时时间，单位是毫秒；也可以使用 EX seconds 以秒为单位设置超时时间。
    


至于解锁操作，小白老师果决的使用 DEL 指令删除。一个分布式锁方案出来了，一气呵成，组员不明觉厉，纷纷竖起大拇指，伪代码如下。
  

```cs
//加锁成功
if（jedis.set(lockKey, 1, "NX", "EX", 10) == 1）{
  try {
      do work //执行业务

  } finally {
    //释放锁
     jedis.del(key);
  }
}
```

  

然而，这是一个错误的分布式锁。问题在于解锁的操作有可能出现释放别人的锁的情况。

  

有可能出现释放别人的锁的情况。

  

1.  客户端 A 获取锁成功，设置超时时间 10 秒。
2.  客户端 A 执行业务逻辑，但是因为某些原因（网络问题、FullGC、代码垃圾性能差）执行很慢，时间超过 10 秒，锁因为超时自动释放了。
3.  客户端 B 加锁成功。
4.  客户端 A 执行 DEL 释放锁，相当于把客户端 B 的锁释放了。
    

原因很简单：**客户端加锁时，没有设置一个唯一标识**。释放锁的逻辑并不会检查这把锁的归属，直接删除。

  

## **二、残血版分布式锁**

  

小白老师：“码哥，怎么解决释放别人的锁的情况呢？”

  

解决方法：客户端加锁时设置一个“唯一标识”，可以让 value 存储客户端的唯一标识，比如随机数、 UUID 等；释放锁时判断锁的唯一标识与客户端的标识是否匹配，匹配才能删除。

  

**加锁**

  

```nginx
SET lockKey randomValue NX PX 3000
```

  

**解锁**

  

删除锁的时候判断唯一标识是否匹配伪代码如下。

  

```cs
if (jedis.get(lockKey).equals(randomValue)) {
    jedis.del(lockKey);
}
```

  

加锁、解锁的伪代码如下所示。

  

```cs
try (Jedis jedis = pool.getResource()) {
    //加锁成功
    if（ jedis.set(lockKey, randomValue, "NX", "PX", 3000) == 1） {
        do work //执行业务
    }
} finally {
    //判断是不是当前线程加的锁,是才释放
    if (randomValue.equals(jedis.get(keylockKey)) {
            jedis.del(lockKey); //释放锁
        }
    }
}
```

  

到这里，很多公司可能都是使用这个方式来实现分布式锁。

  

小白：“码哥，还有问题。判断锁的唯一标识是否与当前客户端匹配和删除操作不是原子操作。”

  

聪明。这个方案还存在原子性问题，存在其他客户端把锁给释放的问题。

  

1.  客户端 A 执行唯一标识匹配成功，还来不及执行DEL释放锁操作，锁过期被释放。
2.  客户端 B 获取锁成功，value 设置了自己的客户端唯一标识。
3.  客户端 A 继续执行 DEL删除锁操作，相当于把客户端 B 的锁给删了。
    


## **三、青铜版分布式锁**

  

虽然叫青铜版，这也是我们最常用的分布式锁方案之一了，这个版本没有太大的硬伤，并且比较简单。

小白老师：“码哥，这如何是好，如何解决解锁不是原子操作的问题？分布式锁这么多门道，是我肤浅了。”

解决方案很简单，解锁的逻辑我们可以通过 Lua 脚本来实现判断和删除的过程。

*   KEYS\[1\] 是 lockKey。
*   ARGV\[1\] 表示客户端的唯一标识 requestId。
    
返回 nil 表示锁不存在，已经被删除了。只有返回值是 1 才表示加锁成功。

```kotlin
// key 不存在，返回 null
if (redis.call('exists', KEYS[1]) == 0) then
   return nil;
end;
// 获取 KEY[1] 中的 value 与 ARGV[1] 匹配，匹配则 del，返回 1。不匹配 return 0 解锁失败
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1]);
else
    return 0;
end;
```

  

使用上面的脚本，每个锁都用一个随机值作为唯一标识，当删除锁的客户端的“唯一标识”与锁的 value 匹配的时候，才能执行删除操作。这个方案已经相对完美，我们用的最多的可能就是这个方案了。

  

理论知识学完了，上实战。

  

## **四、Spring Boot 环境准备**

  

接下来码哥，给你一个基于 Spring Boot 并且能用于生产实战的代码。在上实战代码之前，先把 Spring Boot 集成 Redis 的环境搞定。

  

### **添加依赖**

  

代码基于 Spring Boot 2.7.18 ，使用 lettuce 客户端来操作 Redis。添加 spring-boot-starter-data-redis依赖。

  

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <!-- Import dependency management from Spring Boot -->
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>2.7.18</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.28</version>
            <optional>true</optional>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!--redis依赖-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>

    <!--Jackson依赖-->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

  

---

### **SpringBoot 配置**

  

先配置 yaml。

  

```properties
server:
  servlet:
    context-path: /redis
  port: 9011
spring:
  application:
    name: redis
  redis:
    host: 127.0.0.1
    port: 6379
    password: magebyte
    timeout: 6000
    client-type: lettuce
    lettuce:
      pool:
        max-active: 300
        max-idle: 100
        max-wait: 1000ms
        min-idle: 5
```

  

RedisTemplate 默认序列化方式不具备可读性，我们改下配置，使用 JSON 序列化。注意了，这一步是附加操作，与分布式锁没有关系，是码哥顺带给你的彩蛋。

  

```typescript
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate < String, Object > redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate < String, Object > redisTemplate = new RedisTemplate < String, Object > ();
        redisTemplate.setConnectionFactory(connectionFactory);

        StringRedisSerializer stringRedisSerializer = new StringRedisSerializer();

        redisTemplate.setKeySerializer(stringRedisSerializer); // key的序列化类型

        Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        // 方法过期，改为下面代码
        //        objectMapper.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        objectMapper.activateDefaultTyping(LaissezFaireSubTypeValidator.instance,
            ObjectMapper.DefaultTyping.NON_FINAL, JsonTypeInfo.As.PROPERTY);
        jackson2JsonRedisSerializer.setObjectMapper(objectMapper);
        jackson2JsonRedisSerializer.setObjectMapper(objectMapper);

        redisTemplate.setValueSerializer(jackson2JsonRedisSerializer); // value的序列化类型
        redisTemplate.setHashKeySerializer(stringRedisSerializer);
        redisTemplate.setHashValueSerializer(jackson2JsonRedisSerializer);
        redisTemplate.afterPropertiesSet();
        return redisTemplate;
    }
}
```

  

把分布式锁接口定义出来，所谓面向接口和对象编程，代码如有神。顺带用英文显摆下什么叫做专业。

  

```java
/**
 * 分布式锁
 */
public interface Lock {

    /**
     * Tries to acquire the lock with defined <code>leaseTime</code>.
     * Waits up to defined <code>waitTime</code> if necessary until the lock became available.
     * <p>
     * Lock will be released automatically after defined <code>leaseTime</code> interval.
     *
     * @param waitTime  the maximum time to acquire the lock
     * @param leaseTime lease time
     * @param unit      time unit
     * @return <code>true</code> if lock is successfully acquired,
     * otherwise <code>false</code> if lock is already set.
     * @throws InterruptedException - if the thread is interrupted
     */
    boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException;

    /**
     * Acquires the lock with defined <code>leaseTime</code>.
     * Waits if necessary until lock became available.
     * <p>
     * Lock will be released automatically after defined <code>leaseTime</code> interval.
     *
     * @param leaseTime the maximum time to hold the lock after it's acquisition,
     *                  if it hasn't already been released by invoking <code>unlock</code>.
     *                  If leaseTime is -1, hold the lock until explicitly unlocked.
     * @param unit      the time unit
     */
    void lock(long leaseTime, TimeUnit unit);

    /**
     * Releases the lock.
     *
     * <p><b>Implementation Considerations</b>
     *
     * <p>A {@code Lock} implementation will usually impose
     * restrictions on which thread can release a lock (typically only the
     * holder of the lock can release it) and may throw
     * an (unchecked) exception if the restriction is violated.
     * Any restrictions and the exception
     * type must be documented by that {@code Lock} implementation.
     */
    void unlock();
}
```


## **五、青铜分布式锁实战**

DistributedLock 实现 Lock 接口，构造方法实现 resourceName 和 StringRedisTemplate 的属性设置。客户端唯一标识使用uuid:threadId 组成。

### **DistributedLock**

```java
public class DistributedLock implements Lock {
    /**
     * 标识 id
     */
    private final String id = UUID.randomUUID().toString();

    /**
     * 资源名称
     */
    private final String resourceName;

    private final List < String > keys = new ArrayList < > (1);


    /**
     * redis 客户端
     */
    private final StringRedisTemplate redisTemplate;

    public DistributedLock(String resourceName, StringRedisTemplate redisTemplate) {
        this.resourceName = resourceName;
        this.redisTemplate = redisTemplate;
        keys.add(resourceName);
    }

    private String getRequestId(long threadId) {
        return id + ":" + threadId;
    }
}
```

  

---

### **加锁 tryLock、lock**

tryLock 以阻塞等待 waitTime 时间的方式来尝试获取锁。获取成功则返回 true，反之 false。tryAcquire 方法相当于执行了 Redis 的SET resourceName uuid:threadID NX PX {leaseTime} 指令。

  

与 tryLock不同的是， lock 一直尝试自旋阻塞等待获取分布式锁，直到获取成功为止。而 tryLock 只会阻塞等待 waitTime 时间。

  

此外，为了让程序更加健壮，码哥实现了阻塞等待获取分布式锁，让你用的更加开心，面试不慌加薪不难。如果你不需要自旋阻塞等待获取锁，那把 while 代码块删除即可。

  

```java
@Override
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
    long time = unit.toMillis(waitTime);
    long current = System.currentTimeMillis();
    long threadId = Thread.currentThread().getId();
    // 获取锁
    Boolean isAcquire = tryAcquire(leaseTime, unit, threadId);
    // lock acquired
    if (Boolean.TRUE.equals(isAcquire)) {
        return true;
    }

    time -= System.currentTimeMillis() - current;
    // 等待时间用完，获取锁失败
    if (time <= 0) {
        return false;
    }
    // 自旋获取锁
    while (true) {
        long currentTime = System.currentTimeMillis();
        isAcquire = tryAcquire(leaseTime, unit, threadId);
        // lock acquired
        if (Boolean.TRUE.equals(isAcquire)) {
            return true;
        }

        time -= System.currentTimeMillis() - currentTime;
        if (time <= 0) {
            return false;
        }
    }
}

@Override
public void lock(long leaseTime, TimeUnit unit) {
    long threadId = Thread.currentThread().getId();
    Boolean acquired;
    do {
        acquired = tryAcquire(leaseTime, unit, threadId);
    } while (Boolean.FALSE.equals(acquired));
}

private Boolean tryAcquire(long leaseTime, TimeUnit unit, long threadId) {
    return redisTemplate.opsForValue().setIfAbsent(resourceName, getRequestId(threadId), leaseTime, unit);
}
```

  

---

### **解锁 unlock**


解锁的逻辑是通过执行 lua 脚本实现。
  

```java
@Override public void unlock() {
    
    long threadId = Thread.currentThread().getId();

    // 执行 lua 脚本
    DefaultRedisScript <Long> redisScript = new DefaultRedisScript < > (LuaScript.unlockScript(), Long.class);
    Long opStatus = redisTemplate.execute(redisScript, keys, getRequestId(threadId));
    if (opStatus == null) {
        throw new IllegalMonitorStateException("attempt to unlock lock, not locked by current thread by node id: " +
            id + " thread-id: " + threadId);
    }

}
```

###   **LuaScript**

其实这个脚本就是在讲解青铜板分布式锁原理的那段代码，具体逻辑已经解释过，这里就不再重复分析。

  
```cs
public class LuaScript {

    private LuaScript() {

    }

    /**
     * 分布式锁解锁脚本
     *
     * @return 当且仅当返回 `1`才表示加锁成功.
     */
    public static String unlockScript() {
        return "if (redis.call('exists', KEYS[1]) == 0) then " +
            "return nil;" +
            "end; " +
            "if redis.call('get',KEYS[1]) == ARGV[1] then" +
            "   return redis.call('del',KEYS[1]);" +
            "else" +
            "   return 0;" +
            "end;";
    }
}
```

  

---

### **RedisLockClient**

  

最后，还需要提供一个客户端给方便使用。

  

```java
@Component
public class RedisLockClient {

    @Autowired
    private StringRedisTemplate redisTemplate;


    public Lock getLock(String name) {
        return new DistributedLock(name, redisTemplate);
    }

}
```


单元测试来一个。

  
```java
@Slf4j
@SpringBootTest(classes = RedisApplication.class)
public class RedisLockTest {

    @Autowired
    private RedisLockClient redisLockClient;


    @Test
    public void testLockSuccess() throws InterruptedException {
        Lock lock = redisLockClient.getLock("order:pay");
        try {
            boolean isLock = lock.tryLock(10, 30, TimeUnit.SECONDS);
            if (!isLock) {
                log.warn("加锁失败");
                return;
            }
            TimeUnit.SECONDS.sleep(3);
            log.info("业务逻辑执行完成");
        } finally {
            lock.unlock();
        }

    }

}
```

  

有两个点需要注意。

  

*   释放锁的代码一定要放在 finally{} 块中。否则一旦执行业务逻辑过程中抛出异常，程序就无法执行释放锁的流程。只能干等着锁超时释放。
*   加锁的代码应该写在 try {} 代码中，放在 try 外面的话，如果执行加锁异常（客户端网络连接超时），但是实际指令已经发送到服务端并执行，就会导致没有机会执行解锁的代码。
    

小白：“码哥，这个方案你管它叫青铜级别而已，这么说还有王者、超神版？我们公司还用错误版分布式锁，难怪有时候出现重复订单，是我肤浅了。”

赶紧将这个方案替换原来的错误或者残血版的 Redis 分布式锁吧。


  

