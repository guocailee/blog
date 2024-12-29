---
{"dg-publish":true,"permalink":"/Program/Kotlin/Kotlin多字段排序/","noteIcon":""}
---

```kotlin
data class Person(  
    val name: String,  
    val score: Int  
)  
val max = listOf(  
    Person("bob", 100),  
    Person("alias", 100),  
    Person("Jack", 99)  
).maxWithOrNull(compareBy({  
    it.score  
}, {  
    it.name  
}))  
println(max)

```