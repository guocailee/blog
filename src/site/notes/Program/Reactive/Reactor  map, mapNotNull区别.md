---
{"dg-publish":true,"permalink":"/Program/Reactive/Reactor  map, mapNotNull区别/","noteIcon":"","created":"2025-03-06T21:28:25.982+08:00"}
---



```kotlin
Mono.just("").mapNotNull {  
        if (it == "") {  
            null  
        } else ""  
    }.switchIfEmpty {  
        // 触发  
        println("null")  
        Mono.just("")  
    }.subscribe { println(it) }  
// 抛出NPE异常
Mono.just("").map {  
        if (it == "") {  
            null  
        } else ""  
    }.switchIfEmpty {  
        // 不触发  
        println("null")  
        Mono.just("")  
    }.subscribe {  
        println(it)  
    }

```

1.  mapNotNull 可以返回null， 并转成Mono.empty()
2.  map 不允许返回null，会引发NPE异常
> 无论何时，应该优先使用`mapNotNull`