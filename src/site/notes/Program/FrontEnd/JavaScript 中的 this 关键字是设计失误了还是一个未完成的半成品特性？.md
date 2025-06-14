---
{"dg-publish":true,"permalink":"/Program/FrontEnd/JavaScript 中的 this 关键字是设计失误了还是一个未完成的半成品特性？/","noteIcon":"","created":"2025-03-06T21:28:25.972+08:00"}
---

作者：谷雨同学  
链接：https://www.zhihu.com/question/654178975/answer/3480674806  
来源：知乎  
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。  
  

算不上设计失误，只是早期 JS 设计得过于灵活。

当我们说编程语言中的 `this` 或者 `self`，我们在说什么？其实我们在指定一种特殊的[函数调用形式](https://zhida.zhihu.com/search?content_id=662923015&content_type=Answer&match_order=1&q=%E5%87%BD%E6%95%B0%E8%B0%83%E7%94%A8%E5%BD%A2%E5%BC%8F&zhida_source=entity)——**中缀调用形式**。出于直观，通常的函数调用形式是前缀的，即函数名出现在所有实参之前：

```javascript
f(a, b, c)
```

几乎所有的编程语言都这样设计。但是[面向对象编程范式](https://zhida.zhihu.com/search?content_id=662923015&content_type=Answer&match_order=1&q=%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%BC%96%E7%A8%8B%E8%8C%83%E5%BC%8F&zhida_source=entity)的发明使人们发现，中缀调用形式具有更好的可读性：

```cpp
a.f(b, c)
```

即把函数名放在第一个参数和剩余参数中间。尤其是 `f` 是某种和 `a` 相关的操作的时候，这种写法非常好看好懂（否则可以参考 C 中的面向对象的那些库的设计，各种 `xxx_foo(&xxx, ...)`，看着很费劲）。

那接下来的问题就是，当使用中缀调用形式调用一个函数时，是否需要**将这个“被提前到函数名之前的实参”用一个特殊的形参名来指示**？C++ 和 Java 选择回答“是”。于是，所有可以被中缀调用的函数（C++ 中的非静态成员函数、Java 中的[非静态方法](https://zhida.zhihu.com/search?content_id=662923015&content_type=Answer&match_order=1&q=%E9%9D%9E%E9%9D%99%E6%80%81%E6%96%B9%E6%B3%95&zhida_source=entity)），都通过 `this` 关键字来访问这个特殊形参。

```cpp
void A::foo(B b, C c) {
    // 收到的三个实参分别是 *this, b, c
    std::println("{}, {}, {}", *this, b, c);
}
int main() {
    A a; B b; C c;
    a.A::foo(b, c); // 中缀调用 A::foo。
}
```

Python 对这个问题回答“否”。所有可以被中缀调用的函数，其首个形参就是被提前在函数名之前的那个实参。

```python
class A:
    def foo(a, b, c):
        print(a, b, c)
        
if __name__ == "__main__":
    a = A(); b = B(); c = C()
    a.foo(b, c)
```

JavaScript 是跟风 Java 设计的，所以肯定也引入了 `this`。在大部分的使用场景下，这是一点问题没有的：

```js
class A {
    foo(b, c) {
        console.log(this, b, c);
    }
}
const a = new A(), b = new B(), c = new C();
a.foo(b, c); // very nice.
```

但是问题出在，JS 是动态类型的；如果不去用中缀调用形式调用 `a.foo`，那就有些混乱了。

```js
const bar = A.prototype.foo;
bar(b, c); // So what about `this`?
```

这个时候 `bar` ，也就是原先 `A` 类里的 `foo` 函数，缺少了本应出现在 `foo` 左侧的首个实参 `a`，而它是需要传给形参 `this` 的。失去中缀调用形式后，`foo` 的 `this` 就只能指向一些其他的东西了，比如 `globalThis`，或者严格模式下的 _undefined_。

顺便一提，你只要给这个 `bar` 再以中缀调用的形式调用一下，它就能得到一个 `this` 值了。

```js
var bar = A.prototype.foo; // var 声明为 globalThis 增加一个属性
globalThis.bar(b, c);      // OK, this = globalThis
```

一言以蔽之，JS 的 `this` 的混乱单纯是因为，我有一个期望中缀调用的函数，但你偏偏不去中缀调用导致的。

其他编程语言有这个问题吗？首先那些在之前问题中回答“否”的编程语言不会有；因为在他们眼中，中缀调用不过是普通的前缀调用的“[语法糖](https://zhida.zhihu.com/search?content_id=662923015&content_type=Answer&match_order=1&q=%E8%AF%AD%E6%B3%95%E7%B3%96&zhida_source=entity)”罢了，没有什么特殊的[形参语法](https://zhida.zhihu.com/search?content_id=662923015&content_type=Answer&match_order=1&q=%E5%BD%A2%E5%8F%82%E8%AF%AD%E6%B3%95&zhida_source=entity)。

```python
class A:
    def foo(a, b, c):
        print(a, b, c)
        
if __name__ == "__main__":
    a = A(); b = B(); c = C()
    bar = A.foo
#   bar(b, c)    # Error: 参数数量不对
    bar(a, b, c) # OK
```

其次，静态类型语言不会有这个问题，这些期望中缀调用的函数有独特的类型，不能使用非中缀的语法使用。

```cpp
void A::foo(B b, C c) {
    std::println("{}, {}, {}", *this, b, c);
}
int main() {
    A a; B b; C c;
    auto bar = &A::foo; // OK, 成员函数指针
//  bar(b, c);          // Error, 不可以非中缀调用成员函数
    a.*bar(b, c);       // OK, 必须有一个前置实参
}
```

最后，其它那些用隐式 `this` 的动态类型语言，大多禁止了“非中缀调用”成员函数的语法（比如 `a.foo` 或者 `A.foo` 后面必须跟着括号立即调用，不能作为其他表达式的操作数）。所以这个问题就似乎只在 JS 上存在，并且被那些培训机构和语言小鬼津津乐道。

---

最后扯个题外话，把中缀调用形式做到极致的还得是 [Smalltalk 系语言](https://zhida.zhihu.com/search?content_id=662923015&content_type=Answer&match_order=1&q=Smalltalk+%E7%B3%BB%E8%AF%AD%E8%A8%80&zhida_source=entity)，比如我们看一下 Objective-C 选手的做法：

```objective-c
@interface A {}
- (void)fooWithB: (B)b andC: (C)c;
@end

int main() {
    A *a = [A new]; B b; C c;
    [a fooWithB: b andC: c];   // WTF?
}
```

这里被调用的函数名是 `fooWithB:andC`， 直接拆到所有参数的中间去……`a` 和 `b` 中间夹着 `fooWithB`，然后 `b` 和 `c` 中间夹着 `andC`，将《中缀》表现得淋漓尽致，嗯。