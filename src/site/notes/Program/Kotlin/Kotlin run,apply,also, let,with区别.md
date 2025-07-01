---
{"dg-publish":true,"permalink":"/Program/Kotlin/Kotlin run,apply,also, let,with区别/","noteIcon":"","created":"2024-07-18T14:45:45.906+08:00"}
---

#Kotlin 

在 Kotlin 中，`run`、`with`、`apply`、`also` 和 `let` 是标准库中的几个扩展函数，它们都用于对象的上下文中执行代码块，并具有不同的用途和返回值。下面是它们之间的主要区别：

###  **`run`**：
    - 作用：在对象上下文中执行一个代码块，并返回代码块的结果。
    - 语法：`object.run { }`
    - 返回值：Lambda 表达式的结果。
    - 示例：
    
```kotlin
val result = "Kotlin".run {
  this.length
}
println(result) // 输出 6
```

###  **`with`**：
    - 作用：在不需要返回值的情况下使用对象的上下文执行代码块，返回代码块的结果。
    - 语法：`with(object) { }`
    - 返回值：Lambda 表达式的结果。
    - 示例：
```kotlin
  val result = with("Kotlin") {
      length
  }
  println(result) // 输出 6
```

### **`apply`**：
    - 作用：在对象上下文中执行一个代码块，并返回该对象本身。
    - 语法：`object.apply { }`
    - 返回值：对象本身。
    - 示例：
```kotlin
  val result = StringBuilder().apply {
      append("Kotlin")
      append(" is awesome")
  }
  println(result.toString()) // 输出 "Kotlin is awesome"
```

### **`also`**：
    - 作用：在对象上下文中执行一个代码块，并返回该对象本身。
    - 语法：`object.also { }`
    - 返回值：对象本身。
    - 示例：
```kotlin
  val result = "Kotlin".also {
      println(it) // 输出 "Kotlin"
  }
  println(result) // 输出 "Kotlin"
```

###  **`let`**：
    - 作用：在对象上下文中执行一个代码块，并返回代码块的结果。通常用于处理可空对象。
    - 语法：`object?.let { }`
    - 返回值：Lambda 表达式的结果。
    - 示例：
```kotlin
  val name: String? = "Kotlin"
  val result = name?.let {
      it.length
  }
  println(result) // 输出 6
```

### 总结

- **`run`** 和 **`let`** 都返回代码块的结果，但 **`run`** 是直接在对象上调用，而 **`let`** 适用于可空对象。
- **`with`** 不是扩展函数，作用类似于 **`run`**，但需要传入对象作为参数。
- **`apply`** 和 **`also`** 都返回对象本身，但 **`apply`** 提供了 `this` 作为上下文对象，**`also`** 提供了 `it` 作为上下文对象。
![Pasted image 20241014143155.png](/img/user/z-attchements/Pasted%20image%2020241014143155.png)
![Pasted image 20241014143218.png](/img/user/z-attchements/Pasted%20image%2020241014143218.png)
根据需求选择合适的函数可以使代码更简洁、更易读。