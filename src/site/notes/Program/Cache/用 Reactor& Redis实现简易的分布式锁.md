---
{"dg-publish":true,"permalink":"/Program/Cache/用 Reactor& Redis实现简易的分布式锁/","noteIcon":"","created":"2025-03-06T21:28:25.968+08:00"}
---

```kotlin
inline fun <reified R : Publisher<*>> ReactiveRedisTemplate<String, String?>.lock(  
    lockKey: String,  
    lockTime: Duration = Duration.ofMinutes(5),  
    crossinline fetch: () -> R  
): R {  
    val isFlux = R::class.jvmName == Flux::class.jvmName  
    val key = "${lockKey}:locker"  
    val filter = this.opsForValue().setIfAbsent(key, "locked", lockTime)  
        .filter {  
            it        }    return if (isFlux) {  
        filter.flatMapMany { fetch() }  
    } else {  
        filter.flatMap { fetch() as Mono<*> }  
    }.trigger {  
        this.opsForValue().delete(key)  
    } as R  
  
}
```