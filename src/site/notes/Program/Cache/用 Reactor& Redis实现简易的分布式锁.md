---
{"dg-publish":true,"permalink":"/program/cache/reactor-and-redis/","noteIcon":""}
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