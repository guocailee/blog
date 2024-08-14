---
{"dg-publish":true,"permalink":"/Program/Kotlin/Kotlin 内联函数 inline/","noteIcon":""}
---

#Kotlin

`Kotlin` 中新增了「内联函数」，内联函数起初是在 `C++` 里面的。

那在 `Kotlin` 中加入内联函数，是有什么作用呢？

以下内容分为以下几部分：

1.  什么是 `inline` 内联函数
2.  `inline` 内联函数的作用和使用
    -   2.1 不应该使用 `inline` 的情况
    -   2.2 应该使用 `inline` 的情况
    -   2.3 `inline` 提高效率的原因
3.  内联函数的一些其他用处；
    -   3.1 支持 `return` 退出函数
    -   3.2 禁止内联：`noinline`
4.  小结
5.  参考链接

## 1. 什么是 `inline` 内联函数呢

简单来说：  
当一个函数被内联 `inline` 标注后，在调用它的地方，会把这个函数方法体中的所以代码移动到调用的地方，而不是通过方法间压栈进栈的方式。

代码示例：  
### 1.1 使用 `inline` 的代码

```kotlin
// 在 main() 中调用 makeTest()
fun main() {
    Log.i("zc_test", "main() start")
    makeTest()
    Log.i("zc_test", "main() end")
}
// 内联函数 makeTest()
private inline fun makeTest() {
    Log.i("zc_test", "makeTest")
} 
```

### 1.2 使用 `inline` 编译成 `java` 的代码

```kotlin
public final void main() {
    Log.i("zc_test", "main() start");
    int $i$f$makeTest = false;
    Log.i("zc_test", "makeTest");
    Log.i("zc_test", "main() end");
} 
```

### 1.3 当 `makeTest()` 不在被 `inline` 修饰时, 被编辑成 `java` 的代码

```kotlin
public final void main() {
    Log.i("zc_test", "main() start");
    this.makeTest();
    Log.i("zc_test", "main() end");
} 
```

可以看到，当 `makeTest()` 被`inline` 修饰时， 在 `main()` 中原来调用 `makeTest()` 的地方被替换成了 `makeTest()` 里面的代码。

**换句话说：在编译时期，把调用这个函数的地方用这个函数的方法体进行替换。** 

这就是 `inline` 的本质。

至于 `Kotlin` 内联函数有什么作用呢？

## 2. `Kotlin` 内联函数的作用和使用
由上面可以知道， `inline` 的本质：在编译时期，把调用这个函数的地方用这个函数的方法体进行替换。

那么我们什么时候应该使用 `inline` 什么时候不应该使用呢？

### 2.1 不应该使用 `inline` 的情况
当使用 `inline` 标注时，如果是下面这样，无参数的函数时：

```kotlin
//makeTest() 没有任何的参数
private inline fun makeTest() {
     Log.i("zc_test", "makeTest")
}
//或者带有基本变量参数的函数，编译器也会报错。
private inline fun makeTest2(test: String) {
     Log.i("zc_test", "makeTest")
} 
```

这个时候 `AndroidStudio` 编译器会在 `inline` 位置有黄色警告，  
`Expected performance impact of inlining '...' can be insignificant. Inlining works best for functions with lambda parameters`,  
翻译过来就是，在这个位置使用 `inline` 并不会有很大的提高，`inline` 适合在包含 `lambda` 参数的函数上。

也就是说 `inline` 在一般的方法是标注，是不会起到很大作用的，`inline` 能带来的性能提升，往往是在参数是 `lambda` 的函数上。

* * *

> 在一篇文章上看到这样一段话：  
> 众所周知，`JVM` 内部已经实现了内联优化，它会在任何可以通过内联来提升性能的地方将函数调用内联化，并且相对于手动将普通函数定义为内联，通过 `JVM` 内联优化所生成的字节码，每个函数的实现只会出现一次，这样在保证减少运行时开销的同时，也没有增加字节码的尺寸；  
> 链接：[https://www.jianshu.com/p/678a49054238](https://www.jianshu.com/p/678a49054238)  
> 来源：简书
>
> 这段话也在间接证明了编译器给的警告，`inline` 不适合在无参数的函数中， 适合在包含 `lambda` 参数的函数上。

* * *

### 2.2 应该使用 `inline` 的地方： 带有 `lambda` 参数的函数

当我们写一个会被经常调用的带 `lambda`参数的函数时， 可使用该方式。

例如代码：

```kotlin
// body 是本身一个函数
fun foo(body:() -> Unit) {
    println("foo() hahaha")
    ordinaryFunction(body)
}

inline fun ordinaryFunction(block: () -> Unit) {
    println("hahha")
    block.invoke()
    println("hahha233333")
} 
```

在上述代码中，我们把 `foo()` 的函数参数 `body` 作为一个参数传递给 `ordinaryFunction()` ,

这是我们可以通过在 `ordinaryFunction()` 上面标注 `inline` 从而使得方法的调用栈少一层，使得代码变为：

```kotlin
fun foo(body:() -> Unit) {
    println("hahha")
    block.invoke()
    println("hahha233333")
} 
```

### 2.3 `inline` 的使用规则 
那么什么时候使用，什么时候不使用 `inline` 呢？  
根据上面，我们大致可分为两种：

1.  不带参数，或是带有普通参数的函数，不建议使用 `inline`
2.  带有 `lambda` 函数参数的函数，建议使用 `inline`

### 2.4 `inline` 提高效率的原因

为什么要使用 `inline` 呢？必然是因为使用 `inline` 会带来效率的提升。  
**我们比较一下使用了 `inline` 和不使用 `inline` 编译成 `java` 代码的差异**

当然上述 `ordinaryFunction()` 也可以不使用 `inline` 标注，我们看一下编译成 `java` 的代码样式, 「对比」添加了 `inline` 的标注的 `java` 代码，我们发现，当不添加 `inline` 时，代码中，多出了一个类：

```kotlin
final class TestInline$main$1$1 extends Lambda implements Function0 {
   public static final TestInline$main$1$1 INSTANCE = new TestInline$main$1$1();
   // $FF: synthetic method
   // $FF: bridge method
   public Object invoke() {
      this.invoke();
      return Unit.INSTANCE;
   }

   public final void invoke() {
   }

   TestInline$main$1$1() {
      super(0);
   }
} 
```

它便是在编译过程中，因为 `lambda` 参数 多出来的类，无疑中会增加内存的分配。

所以我们就知道了，在 `kotlin` 中，因为出现了大量的 `高阶函数` -- 「高阶函数是将函数用作参数或返回值的函数」，使得越来越多的地方出现 `函数参数` 不断传递的现象，每一个函数参数都会被编译成一个对象， 使得内存分配（对于函数对象和类）和虚拟调用会增加运行时间开销。所以才会出现 `inline` 内联函数。可以通过 `inline` 的标注，把原本需要生成一个类的开销节省了， 同时也少了一层方法栈的调用。

## 3. `inline` 的其他作用

除了上述的功能点外，还有一些值得注意的小地方。

### 3.1 支持 `return` 退出函数
在编码中，我们通常习惯使用 `return` 返回退出这个函数，但是 `lambda` 表达式不能使包含它的函数返回。

例如代码：

```kotlin
fun foo(body:()->Unit) {
    ordinaryFunction {
        println("zc_testlabama 表达式退出")
        return
    }
    println("zc_test --->foo() end")
} 
fun ordinaryFunction(block: () -> Unit) {
    println("hahha")
    block.invoke()
    println("hahha233333")
} 
```

如果在 `ordinaryFunction` 这个方法没有 `inline` 的标注，编译器会在 `return` 的位置出错，`return is not allowed here`.

> 解决上述错误的方式，可以为 `return` 添加标签，例如 `return@ordinaryFunction`, 但是这样的话，方法执行只会退出 `lambda` 表达式，后面的代码 `println("zc_test --->foo() end")` 还是会走到的。

当我们添加上 `inline` 时，正确的代码如下：

```kotlin
fun foo(body:()->Unit) {
    ordinaryFunction {
        // 因为标识为 inline 的函数会被插入到调用出，此时 return 肯定是 return 到该整个方法
        println("zc_testlabama 表达式退出")
        return
    }
    println("zc_test --->foo() end")
}
// 如果不使用 inline， 上面代码会被报错。因为「不允许这么做」
inline fun ordinaryFunction(block: () -> Unit) {
    println("hahha")
    block.invoke()
    println("hahha233333")
} 
```

当我们添加了 `inline` 标志后，在 `ordinaryFunction{}` 的 `return` 时就会退出整个 `foo()` 函数，因此结尾的 `println("zc_test --->foo() end")` 是不会被调用的。

**`inline` 可以让函数参数里面的 `return` 生效**

> `kotlin` 官方注释：`break` 和 `continue` 在内联的 `lambda` 表达式中还不可用，但我们也计划支持它们。

### 3.2 禁止内联：`noinline`

为什么会有 `noinline`呢？为什么需要这种方式呢？

官网中这么写着：如果希望只内联一部分传给内联函数的 `lambda` 表达式参数，那么可以用 `noinline` 修饰符标记不希望内联的函数参数, 代码如：

```kotlin
inline fun foo(inlined: () -> Unit, noinline notInlined: () -> Unit) { …… } 
```

什么时候我们会需要 `noinline` 呢？  
例如代码：

```kotlin
inline fun foo(testName:String, body:()->Unit) {
    // 这里会报错。。。
    ordinaryFunction(body)
    println("zc_test --->foo() end")
} 
fun ordinaryFunction(block: () -> Unit) {
    println("hahha")
    block.invoke()
    println("hahha233333")
} 
```

如果 `ordinaryFunction()` 不使用 `inline` 标注，是一般的函数，这里是不允许把内联函数 `foo()` 的函数参数 `body` 传递给 `ordinaryFunction()`。

即：**内联函数的「函数参数」 不允许作为参数传递给非内联的函数**，  
如果我们想要实现上述的调用，便可以使用 `noinline` 标注内联函数 `foo()` 的 `body` 参数

```kotlin
inline fun foo(testName:String, noinline body:()->Unit) {
    ...
} 
```

上述代码便可以正常运行了。

## 4. 小结

上面对 `inline` 做了一些简单的介绍。大部分都是实践中产生的结论。  
使用了很多代码，这是不可避免的，只有使用多了，才会比较熟悉这些 `kotlin` 中的属性。

当然也很局限，个人水平有限，有如错误，还请指出。

## 5. 参考链接

`简书：https://www.jianshu.com/p/678a49054238`

`官网：https://www.kotlincn.net/docs/reference/inline-functions.html`

`文章来自：` [kotlin inline 初步解析](https://links.jianshu.com/go?to=%25E6%2596%2587%25E7%25AB%25A0%25E6%259D%25A5%25E8%2587%25AA%25EF%25BC%259Ahttps%3A%2F%2Fmp.weixin.qq.com%2Fs%2FwYxQZT3TpGSU-AYeLhwJXA)

 [https://www.jianshu.com/p/8a0d5bae9cdf](https://www.jianshu.com/p/8a0d5bae9cdf)
