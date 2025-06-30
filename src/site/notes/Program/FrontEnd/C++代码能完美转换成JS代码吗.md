---
{"dg-publish":true,"permalink":"/Program/FrontEnd/C++代码能完美转换成JS代码吗/","noteIcon":"","created":"2025-02-08T11:47:28.865+08:00"}
---

作者：doodlewind  
链接：https://www.zhihu.com/question/33686710/answer/1614152353  
来源：知乎  
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。  
  

不考虑「人肉重写一切皆可转换」这种正确而无用的废话，对于「具体有哪些 C++ 特性无法被完美地自动编译成 JS」这个问题，这在 Emscripten 编译器作者 Alon Zakai 的论文 [Emscripten: An LLVM-to-JavaScript Compiler](https://link.zhihu.com/?target=https%3A//github.com/emscripten-core/emscripten/blob/master/docs/paper.pdf) 中已经论述得很清楚了，值得科普。

首先需要澄清一种误区，那就是把 Emscripten 当作 Babel 和最早的 C++ 编译器那种源到源的「转译器」。这种方式基本只能抽取出原始语言中写起来与 JS 兼容的一个子集，并且可能由于原语言和 JS 的语义差别，导致计算结果上的差异。比如 Python 到 JS 的转译器 Pyjamas 就会把 Python 的除法表达式转译成 JS 的除法表达式，这样 `1 / 2` 在 Python 里是 0（Python 整数除法返回整数），而在转译出的 JS 里就是 0.5（JS 里一切皆 number）。这种差异是可以致命的。实际上哪怕是 Dart 这种看起来和 JS 相当接近的语言，[都没有选择转译这条路](https://link.zhihu.com/?target=https%3A//github.com/dart-lang/sdk/issues/32405)。所以转译器模式虽然可能符合很多前端同学发明 [AST 转换](https://zhida.zhihu.com/search?content_id=323557635&content_type=Answer&match_order=1&q=AST+%E8%BD%AC%E6%8D%A2&zhida_source=entity)轮子的梦想，但它在工程上并不是种足够通用的手段。

那么，什么才是「真正通用的手段」呢？Emscripten 选择的是从低层面的 [LLVM IR](https://zhida.zhihu.com/search?content_id=323557635&content_type=Answer&match_order=1&q=LLVM+IR&zhida_source=entity) 汇编出发，生成出与其兼容的 JavaScript。这意味着比如下面这段高斯同学小学二年级就会优化的 C 语言代码：

```c
// demo.c
#include <stdio.h>
int main() {
  int sum = 0;
  for (int i = 1; i < 100; i++)
    sum += i;
  printf("1+...+100=%d\n", sum);
  return 0;
}
```

就可以用 `clang demo.c -emit-llvm -S -o demo.ll` 这一行命令，先编译出下面这样的 LLVM IR 代码（这里我加了点方便理解的注释）：

```nasm
; demo.ll
@.str = private constant [14 x i8] c"1+...+100=%d\0A\00"

define i32 @main() {
  %1 = alloca i32, align 4
  %sum = alloca i32, align 4
  %i = alloca i32, align 4
  store i32 0, i32* %1 ; 准备循环初始值
  store i32 0, i32* %sum, align 4
  store i32 1, i32* %i, align 4
  br label %2 ; 跳进 for 循环体
; <label>:2
  %3 = load i32* %i, align 4
  %4 = icmp slt i32 %3, 100 ; 判断是否 i < 100 继续循环
  br i1 %4, label %5, label %12 ;
; <label>:5
  %6 = load i32* %i, align 4
  %7 = load i32* %sum, align 4
  %8 = add nsw i32 %7, %6 ; sum += i
  store i32 %8, i32* %sum, align 4
  br label %9
; <label>:9
  %10 = load i32* %i, align 4
  %11 = add nsw i32 %10, 1 ; 循环未结束，i++ 后再来一轮
  store i32 %11, i32* %i, align 4
  br label %2

; <label>:12
  %13 = load i32* %sum, align 4
  %14 = call i32 (i8*, ...)* @printf(i8* getelementptr inbounds ([14 x i8]* @.str, i32 0, i32 0), i32 %13)
  ret i32 0
}
```

Emscripten 处理的就是这种形式的 IR 输入。初看起来，这种 IR 格式比原本的 C++ 更难翻译成 JS。但别忘了如果要处理普通的 C++ 代码，还要支持预处理器、类、模板等复杂特性。而 LLVM 汇编则层次较低，工作起来也比较简单。并且编译它还有一个好处，那就是只要搞定一次这个层面的代码生成，Emscripten 就能支持所有可以编译到 LLVM IR 的语言了。

要想知道这个过程有什么限制，我们就必须知道 Emscripten 到底会把 LLVM IR 编译成什么样的 JS。例如上面这个例子，对应的 JS 输出就是这样的：

```js
function _main() {
  var __stackBase__ = STACKTOP;
  STACKTOP += 12; // 在栈上分配 3 个变量，每个 4 字节
  var __label__ = -1;
  while(1) switch(__label__) {
    case -1:
      var $1 = __stackBase__;
      var $sum = __stackBase__+4;
      var $i = __stackBase__+8;
      HEAP[$1] = 0;
      HEAP[$sum] = 0;
      HEAP[$i] = 0;
      __label__ = 0; break;
    case 0:
      var $3 = HEAP[$i];
      var $4 = $3 < 100;
      if ($4) { __label__ = 1; break; }
      else    { __label__ = 2; break; }
    case 1:
      var $6 = HEAP[$i];
      var $7 = HEAP[$sum];
      var $8 = $7 + $6;
      HEAP[$sum] = $8;
      __label__ = 3; break;
    case 3:
      var $10 = HEAP[$i];
      var $11 = $10 + 1;
      HEAP[$i] = $11;
      __label__ = 0; break;
    case 2:
      var $13 = HEAP[$sum];
      var $14 = _printf(__str, $13);
      STACKTOP = __stackBase__;
      return 0;
  }
}
```

这段 JS 代码很冗长，但不难与 LLVM IR 一一对应起来。划几个重点：

- 为了让控制流在 LLVM 的基本块（Basic Block）之间移动，生成的代码结构是个 while 循环中的 switch 语句。这里 switch 的 label 对应各个目标基本块的标签名，然后以 break 的形式跳转到相应的基本块。在每一个基本块里，每行代码都对应一行 LLVM IR，这种转换是很简单直观的。
- 程序的内存由 `HEAP` 这个 JS 数组来模拟实现。对内存的读写直接对应于对 JS 数组的读写。另外 `STACKTOP` 变量对应栈顶。特别注意，这里在栈上为 32 位整数分配了 4 个内存位置，但只对其中的第一个位置进行写入。
- LLVM 中的函数被编译成了 JS 的函数，LLVM 的函数调用对应于普通的 JS 函数调用。可见 Emscripten 生成的是尽可能「正常」的 JavaScript。
- LLVM 中的加法运算也由 JS 模拟实现。同样地，这种加法运算的语义与原始 LLVM IR 的语义并不完全相同，比如溢出截断之类。

关键的地方来了，这种机制存在哪些「不能完美转换」的代码呢？这里可能导致兼容问题的地方，主要有这么两点：

- 读写内存时的 LSC（[Load-Store Consistency](https://link.zhihu.com/?target=https%3A//emscripten.org/docs/site/glossary.html)）规则。
- 运算语义的差异。

所谓 LSC，指的是一旦往内存中某个位置写入了一个具备某种类型的值，那么后续对其的读取都应使用同样的类型。比如某个变量如果被赋值为了 32 位浮点数，那么读取这个值的时候就应该始终读取 32 位浮点数，而不是 16 位无符号整数之类。像下面这两行代码，就不符合 [LSC 规则](https://zhida.zhihu.com/search?content_id=323557635&content_type=Answer&match_order=1&q=LSC+%E8%A7%84%E5%88%99&zhida_source=entity)：

```c
int x = 12345;
// 把 32 位整数的第一个字节读出来，这可以用来判断大小端
printf("first byte: %d\n", *((char*)&x));
```

为什么 LSC 会影响 C++ 到 JS 的兼容性呢？因为 JS 中并没有真正的 32 位整数。如果要完美保证上面这两行（违反 LSC 规则的）代码的兼容性，就必须生成这样的 JS：

```js
var x_value = 12345;
var x_addr = stackAlloc(4);
HEAP[x_addr] = (x_value >> 0) & 255; // 第 1 个字节
HEAP[x_addr+1] = (x_value >> 8) & 255; // 第 2 个字节
HEAP[x_addr+2] = (x_value >> 16) & 255; // 第 3 个字节
HEAP[x_addr+3] = (x_value >> 24) & 255; // 第 4 个字节
[...]
printf("first byte: %d\n", HEAP[x_addr]);
```

如果每个整数变量都要编译成这么冗长的 JS，显然是非常低效的。因此更好的方式是简单处理成这样：

```js
var x_value = 12345;
var x_addr = stackAlloc(4);
HEAP[x_addr] = x_value; // 直接存进整个 int
[...]
printf("first byte: %d\n", HEAP[x_addr]);
```

这时栈上的第一个变量位置是 `HEAP[__stackBase__]`，第二个变量的位置是 `HEAP[__stackBase__+4]`，而中间的数组下标就不再逐个字节存储值了。Emscripten 默认选择的就是这种方式，这显然不兼容上面那两行「直接把 int 的第一个字节取出来」的代码。实际工程中的 C++ 代码一般都符合 LSC 规则，Emscripten 也为特殊情况提供了 [SAFE_HEAP](https://zhida.zhihu.com/search?content_id=323557635&content_type=Answer&match_order=1&q=SAFE_HEAP&zhida_source=entity) 和 [QUANTUM_SIZE](https://zhida.zhihu.com/search?content_id=323557635&content_type=Answer&match_order=1&q=QUANTUM_SIZE&zhida_source=entity) 等配置，不过这并不能保证完美地自动解决 LSC 的潜在问题。违反 LSC 的情况很少见，主要是这么几种：

- 检查大小端等 CPU 特性的代码，这其实很容易人工兼容掉。
- `memset` 及其相关的代码。比如某段存着各种类型的内存区域，可能在 64 位设备上就会被 `memset` 批量刷入 64 位的 0。Emscripten 通过在运行时标准库的 `memcpy` 上做兼容，规避了这时的问题。
- 某些被优化器「过度优化」的 C++ 代码。比如 2 个 32 位整数可能被 LLVM IR 的优化器合并成单个 64 位的值，从而违反 LSC。但这也没关系，只要你编译时不加入这种优化器就没事。

所以简单总结一下第一点：**某些违反 LSC 规则的 C++，不能被 Emscripten 完美编译成 JS**。

然后还有第二点，那就是运算语义上的差异。比如对于下面这行执行 8 位整数加法的IR：

```nasm
add i8 %1, %2
```

它如果编译成 `$1 + $2` 这样的 JS，是有较大可能存在溢出问题的。为此 Emscripten 既支持生成带运行时检查的精确代码，也支持生成直接转译形式的高效代码（毕竟平时的大多数计算并不会溢出）。那么如何寻找哪些位置可能存在溢出呢？它设计的辅助工具是这样的：

1. 先编译出带有 `CHECK_OVERFLOWS`、`CHECK_SIGNS` 和 `CHECK_ROUNDINGS` 等额外运行时检查的 JS 代码。
2. 将一份测试数据输入生成的 JS，在发现溢出、取整等问题的位置给出警告。
3. 重新编译 C++ 为 JS，仅在发现警告的相应位置添加运行时检查。

虽然这种方式依赖输入的测试数据，但实践中这能显著地提高性能。如果要保证 100% 的正确性，也可以默认全部加上运行时检查。

最后，对于所谓的「完美转换」，还有一个很有意思的地方，那就是相信大家都会觉得，这种转换方式默认生成的代码实在非常丑吧……丑本身也是一种「不完美」的问题！这还真不是审美层面上的品味问题，由于现代 JS 引擎一般会往死里优化那些「朴素的、普通的、正常的」逻辑，**因此生成的代码越接近普通的 JS 写法，理论上越有更好的性能**（有趣的是，这句话是对机器来说才成立的。对人类来说，[反而是手写晦涩 JS 更可能发挥出极致性能](https://www.zhihu.com/question/402807137/answer/1322391162)，这还真是个围城）。具体到 Emscripten 为了追求性能的「美化」而作出的努力，则包括这么两点：

1. 变量的原生化（Variable Nativization），也就是把上面那些存在 `HEAD[__stackTop+8]` 里的变量尽量优化成 `var x` 的形式，从而更利于 JS 引擎的优化。
2. 循环重建（Relooping），也就是把那个到处乱跳的 switch 语句尽量优化成一个更语义化的普通循环。这个东西属于 Emscripten 论文里真正的干货部分，不过和这个回答关系不大，有兴趣的同学可以自己去论文里查阅实现细节。

总之在经过 Emscripten 的优化之后，上面的 JS 会变成这样：

```js
function _main() {
  var __label__;
  var $1;
  var $sum;
  var $i;
  $1 = 0;
  $sum = 0;
  $i = 0;
  $2$2: while(1) {
    var $3 = $i;
    var $4 = $3 < 100;
    if (!($4)) { __label__ = 2; break $2$2; }
    var $6 = $i;
    var $7 = $sum;
    var $8 = $7 + $6;
    $sum = $8;
    var $10 = $i;
    var $11 = $10 + 1;
    $i = $11;
    __label__ = 0; continue $2$2;
  }
  var $13 = $sum;
  var $14 = _printf(__str, $13);
  return 0;
}
```

你可能会奇怪为什么多出来了这么多冗余的变量，个人理解中这和 LLVM 的 [SSA](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/Static_single_assignment_form) 机制有关。不过这还不是最终形态，通过 Google Closure Compiler，上面这段代码还能继续被「极致精简」成这样：

```js
function K() {
  var a, b;
  b = a = 0;
  a:for(;;) {
    if(!(b < 100)) {
      break a
    }
    a += b;
    b += 1;
  }
  _printf(J, a);
  return 0;
}
```

看起来是不是已经和最初的 C 源码相差不大了？可见 Emscripten 这一套组合拳下来，不仅最大限度地支持了 C++ 的各类语义，理想情况下生成的 JS 还能接近「转译器」直接转换 AST 生成的代码。

当然，在涉及到「美感」的语境下，对所谓「完美转换」的定义，显然就是主观而开放的了。虽然 Emscripten 将「生成接近人类编写的朴素 JS」作为优化性能的手段，但「有多接近」就无法量化衡量了。说到底，所谓「完美」也不过是真空中的球形鸡。哪怕 clang 本身也不能直接编译那些依赖了其他编译器特殊特性的项目，需要上层适配（[Clang Language Compatibility](https://link.zhihu.com/?target=https%3A//clang.llvm.org/compatibility.html)）。但即便有这些问题，clang 仍然公认很香，不是吗？

最后，要注意这是一篇 2011 年的论文。Emscripten 这个编译器在工程上早已高度实用化了。有了后来 [asm.js](https://zhida.zhihu.com/search?content_id=323557635&content_type=Answer&match_order=1&q=asm.js&zhida_source=entity) 与 WASM 的加持，当时文中许多兼容问题与性能问题应该也已经不再是问题了。我最近尝试了用它把 1995 年（C99 规范出现之前的）最早的 JS 引擎 Mocha [编译到 JS 和 WASM](https://zhuanlan.zhihu.com/p/330586852)。在获得 clang 编译出的原生版本后，这完全就是把 clang 换成 emcc 再加个把编译参数的事情，一次成功，非常轻松。

所以，C++ 到 JS 的转换虽然存在特定的能力限制，但这项技术早已经（在传统前端圈外）获得广泛使用了。而对新技术的「祛魅」，更需要我们把理论与实践相结合。如果你还有更多的困惑，不妨这就去读读论文，再试试写几段 C++ 代码编译成 LLVM IR 和 JS 如何？坐而论道，不如起而行之。