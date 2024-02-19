---
{"dg-publish":true,"permalink":"/Program/Reactive/Reactor  map, mapNotNull区别/","dgPassFrontmatter":true}
---



```kotlin
Mono.just("")  
    .mapNotNull {  
        if (it == "") {  
            null  
        } else ""  
    }.switchIfEmpty {  
       // 触发
        println("null")  
        Mono.just("")  
    }.subscribe { println(it) }

Mono.just("")  
    .map {  
        if (it == "") {  
            null  
        } else ""  
    }.switchIfEmpty { 
    // 不触发
        println("null")  
        Mono.just("")  
    }.subscribe { println(it) }

```