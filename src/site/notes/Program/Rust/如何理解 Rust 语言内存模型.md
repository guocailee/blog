---
{"dg-publish":true,"permalink":"/Program/Rust/如何理解 Rust 语言内存模型/","noteIcon":"","created":"2025-03-06T21:28:25.983+08:00"}
---

#Rust 
## 背景

随着Rust越来越多的应用良好表现，吸引了越来越多的开发者关注和各领域对应的Rust解决方案出现。对于从C/GO/Java/Python这些语言开发者来说，学习Rust语言最大的挑战就是需要理解Rust语言的内存管理模型。而Rust开创性的所有权管理机制，是我们理解和精通该门语言必须要首先弄清楚的要点。而这也是为什么大家一致认为Rust的学习曲线陡峭的最核心原因。

> 本系列文章主要是阅读图灵系列丛书《Rust程序设计》读书笔记，再以有经验程序员新学Rust语言的路线来编写。

## 示例

我们首先来看一段代码：  

```rust
let v = vec![4, 8, 19, 27, 34, 10];
let r = &v;
let aside = v;
println!("{}", r[0]) 
```


这段代码逻辑很简单，仅仅是定义了三个变量，并做一些赋值和打印操作。三个变量定义如下：  
1. 变量v，定义v为一个向量，其内包含了6个Int类型数字。  
2. 变量r，再将v的地址赋值给变量r，相当于r是v向量变量的引用。  
3. 变量aside，最后将变量v赋值给变量aside。  

最后打印r变量的第0个元素。  
如果根据我们以往的编程经验，该段代码找不出来任何逻辑和写法问题，但是在Rust中，编译运行该段代码，其会报错：  

```rust
warning: unused variable: `aside`
 --> src/main.rs:4:9
  |
4 |     let aside = v;
  |         ^^^^^ help: if this is intentional, prefix it with an underscore: `_aside`
  |
  = note: `#[warn(unused_variables)]` on by default

error[E0505]: cannot move out of `v` because it is borrowed
 --> src/main.rs:4:17
  |
3 |     let r = &v;
  |             -- borrow of `v` occurs here
4 |     let aside = v;
  |                 ^ move out of `v` occurs here
5 |     println!("{}", r[0]);
  |                    - borrow later used here

For more information about this error, try `rustc --explain E0505`. 
```


从这段错误，我们可以得出以下结论：  
1.Rust能够在编译阶段发现代码runtime阶段内存错误问题。  
2.Rust语言编译时能够非常详尽的解释编译错误。  
3.编译器提示在将v的引用赋值给r时，相当于将v借用给了r，借用完成后，再次将v赋值给aside操作时，所有权出现了Move操作，v的所有权到了变量aside之上，这个时候，再去访问r变量，就会出现报错，相当于访问了一个悬空指针。  
而这个报错，正是由于Rust的所有权机制导致的。那么，我们这篇文章主要是为了解释下面两个问题:  
1.Rust所有权制度到底是为了解决什么问题出现的？  
2.Rust所有权制度如何解决该问题的？

## Rust内存模型

在我们过往的编程语言使用经验中，我们都了解类似C/C++语言，内存管理都是靠程序员手动来维护的，new/free的操作都是程序员自己去控制，其性能非常高，但是由此引入了非常多的内存问题，比如访问了已经释放了的内存地址，或者内存没有释放导致内存泄漏从而系统不稳定等等问题。为了最大限度避免这些问题，在使用C/C++语言时，大多数程序员都要手写一个内存池来进行内存管理，而由此带来的内存碎片等问题都不容易处理。而Java类的，内存管理都由Jvm虚拟机来完成，减少了程序员的出错，但是Jvm虚拟机进行自动内存整理(GC)时，又带来了很大的CPU波动。影响了业务系统，导致业务系统在高性能计算时由于GC出现而出现大幅度时延。  
而Rust即想得到C/C++手动管理内存的性能，又想自动化去管理内存，还要避免类似Java统一GC的性能损失。不得不说，这种成年人的选择总会让人为之精神一振。在此背景下，Rust所有权系统应运而生，做到了熊掌与鱼兼得，后续，我们主要分析下Rust所有权系统是如何工作的。

## Rust内存结构示例

Rust所有权系统主要是用来做自动化内存管理，那么，我们首先分析下Rust代码内存结构。其示意图如下所示：

![](https://res.cloudinary.com/practicaldev/image/fetch/s--77aGZC1F--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/nx7zpi93u1y03ijhrw1g.png)

和C语言类似，Rust程序内存布局包括了堆、栈、静态数据区、只读数据区和只读代码区。  
其中，对于每个区存放的内容，大抵可以如下分类：  
1.栈：在编译阶段就可以确定哪些数据可以存放到栈上，由编译器管理，函数局部变量等，存放到栈中。  
2.堆：由程序员编写的代码来申请使用，一般做大量数据读写时使用，运行时申请。  
3.静态数据区：一般的静态函数、静态局部变量和静态全局变量存放区域，在程序启动时初始化。  
4.Literals（只读数据区）：存放代码的文字常量区域。  
5.Instructions（只读代码区）：存放可执行代码区域。

我们以一段Rust代码来举例，了解下Rust中各变量是如何存储的，代码如下：  

```rust
let noodles = "noodles".to_string();
let oodles = &noodles[1..];
let poodles = "👁_👁"; 
```


该段代码内存布局如下图所示：

![](https://res.cloudinary.com/practicaldev/image/fetch/s--AfOEaRTP--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/6fxzquzvfwcheb598c49.png)  
分析内存结构图，我们可以了解到下面结论：  
1.noodles,oodles,poodles三个变量都存储在栈上，并且都是三个胖指针。  
2.noodles和oodles是指向同一块内存，只不过指针首地址不一样。  
3.poodles变量的数据内容是存储在预分配的只读内存区。  
4.noodles的变量存储格式包括三个部分，第一个部分是指向数据存储堆内存首地址，第二个部分是该变量的容量，第三个部分是该变量的长度。(实际上Rust里String是用Vec来实现的，所以这里的容量是Vec管理策略来决定，Rust里分配原则是2->4->8，如果容量不够，下次申请的为前一次的2倍。)  
5.字符串常量poodles的内存是提前分配好的只读内存区。  
6.引用并不做深度拷贝操作，仅仅是指针指向数据堆内存地址。

## 多语言内存赋值解析

内存管理体现在每个语言对赋值操作的实现中，我们可以对比下Python、C++和Rust这几种比较有代表性的语言，了解下他们各自对赋值操作内存是如何管理的。

### [](#python)Python

我们以下面代码为例：  

```python
s = ['udon', 'ramen', 'soba']
t = s
u = s 
```


整个操作的内存变化如下图所示：  
![](https://res.cloudinary.com/practicaldev/image/fetch/s--Nuo4kLXM--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/dkg8dr2tv0mybopgq4ri.png)

我们分析可以得出结论：  
1.Python的字符串和列表底层都是胖指针的形式存储，列表指针的存储内容为：引用计数，列表长度，列表数据指针，列表容量。字符串指针的存储内容为：引用计数，字符串长度，文本数据内容。  
2.局部变量存储在栈中。  
3.赋值操作过程为，t = s，新建一个对象t，指向s的内存地址，并将s对象的引用计数+1。u = s，再新建一个对象u，指向s的内存地址，并将s对象的引用计数+1，s对象的引用计数为3，表示被3个对象使用。  
4.释放s内存数据得维护s的引用计数，引用计数为0时可以清理该内存数据。

### [](#c)C++

对应的，C++赋值示例代码如下：  

```c++
using namespace std;
vector<string> s = {"udon", "ramen", "doba"};
vector<string> t = s;
vector<string> u = s; 
```


该段代码内存结构变化如下图所示：  
[![](https://res.cloudinary.com/practicaldev/image/fetch/s--4RzwrwFG--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/aj66zqx9zmiy4tj83kav.png)
](https://res.cloudinary.com/practicaldev/image/fetch/s--4RzwrwFG--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/aj66zqx9zmiy4tj83kav.png)  
分析后，我们可以得出结论：  
1.向量s局部变量在内存中存储在栈中。其也是一个胖指针，三个字段内容为向量数据堆内存地址，向量占用空间大小，向量长度。堆内存地址数据存储也是三个胖指针，指针地址字段指向的分别是三个字符串的内存地址。  
2.t = s操作过程实际上是复制了一份s对象的数据，包括堆内存数据，并将新的堆内存数据指向t胖指针的堆内存数据地址。  
3.u = s操作过程和t = s操作过程一致。  
4.完成赋值操作后，内存中有三份s对象一样的数据，存储在不同的堆中。  
5.释放s，t，u三个对象内存很简单，各自维护自己生命周期即可。

### [](#rust)Rust

最后，我们再来看Rust赋值代码：  

```rust
let s = vec!["udon".to_string(), "ramen".to_string(), "soba".to_string()];
let t = s;
let u = s; 
```

Enter fullscreen mode Exit fullscreen mode

Rust内存结构变化如下图所示：  
![](https://res.cloudinary.com/practicaldev/image/fetch/s--yqU1H9OP--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ga4x0js1mzmd5berk9ky.png)

我们可以得出结论：  
1.Rust中向量存储和字符串存储方式和C++一样，都是胖指针，指针内容格式一致。  
2.t = s操作，将t胖指针的堆内存数据地址指向s的堆内存数据地址，s对象变成悬空指针，无法访问。  
3.u = s操作，会报错，此时s为悬空指针，不能访问。  
4.向量s在堆内存中数据只有一份。  
5.释放数据的操作也简单，因为堆内存中只有一份数据，在脱离了作用域后会自动释放内存数据。

### [](#%E4%B8%89%E7%B1%BB%E8%AF%AD%E8%A8%80%E5%AF%B9%E6%AF%94)三类语言对比

对比C++，Python和Rust语言在相同赋值语句下，其内存布局变化，我们可以很直观的得出下面结论：  
1.整个赋值过程占用内存最小的是Rust语言和Python语言。  
2.C++语言赋值操作最为笨重，需要做数据深度拷贝。  
3.在释放内存操作时，最高效简单的是Rust语言和C++语言。  
4.Python语言在释放内存时需要维护引用计数，较为复杂。

> 注意，这里只是对比最常用情况，实际上Rust也支持类似Python的引用计数内存管理方法和C++的深度拷贝操作。有兴趣可以去了解相关文档。

## 引用介绍

在上述变量赋值操作过程中，实际上是一个变量所有权转移的过程。那么，是否可以直接使用类似C语言指针的方式去操作一个变量呢？答案是肯定的，Rust提供一种引用的数据类型来完成此目的。

> Rust中对变量的引用，称之为借用（Borrowing），使用完毕后，需要归还。

我们来看一段示例代码：  

```rust
use std::rc::Rc
let s:Rc<String> = Rc::new("shirataki".to_string());
let t:Rc<String> = s.clone();
let u:Rc<String> = s.clone(); 
```


这段代码中，是用了Rust的Rc包来创建一个可以被多个变量同时借用的引用变量，Rust中还提供一个Arc包来实现相同功能，区别是Rc非线程安全型，Arc是线程安全型的。Rc使用引用计数方式来实现，Arc是Atomic Rc。Arc相比Rc会额外带来性能损耗，需要用户根据场景选用。

引用适用领域：  
1.图操作中一个点被多个边包含。  
2.一个变量被多个线程同时操作。

上段代码在内存中的存储格式为：  
![](https://res.cloudinary.com/practicaldev/image/fetch/s--0gSGyICo--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0qjfex64ad5zejp6izk6.png)


为了保证整个机制在各个场景下的可靠，不出现数据竞争情况，Rust引入了所有权树。

## 所有权树

Rust中，所有权规则总结如下：  
1.Rust中的每个值都有一个被称为其所有者的变量(即：值的所有者是某个变量)。  
2.值在任一时刻有且只有一个所有者。  
3.当所有者(变量)离开作用域，这个值将被销毁。

如下图所示，变量的所有权可以被借用，主要包括了可变引用和非可变引用。这里的可变引用和不可变引用可以使用读写锁来理解。通常来说，只读的不可变引用即是只读引用，变量可以被多个只读引用来同时借用，由于是只读的，所以不存在变量共享问题。而可修改引用，则要求一个变量只能被一个可修改引用借用，而且，对该变量的访问，只能通过该可变引用来访问。规则如下图所示：

![](https://res.cloudinary.com/practicaldev/image/fetch/s--QxcJV2JA--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ni4iifs2cdqct9k3jg7b.png)


所有权树表示了所有权行为都是可以推导的，也就是说在编译阶段编译器即可发现各类的内存管理问题，所以这也是Rust内存安全性的保证。

## 所有权示例代码分析

> 大家可以通过下面的代码片段及后面注释中的解释来理解Rust的所有权树。

1.代码片段1：  

```rust
let mut x = 10;
let r1 = &x;
let r2 = &x;       //可以，允许多次共享借用
x += 10;           //错误，不能给x赋值，因为他已经被借用了
let m = &mut x;    //错误，不能借用x的可修改引用，因为他已经借出了不可 
                          //修改引用

let mut y = 20;
let m1 = &mut y;
let m2 = &mut y;   //错误，不能借两次给可修改引用
let z = y;         //错误，不能直接访问y，因为他已经借出了可修改引用 
```


1.  代码片段2：

```rust
let mut w = (107, 109); 
let r = &w;
let r0 = &r.0;         //可以，共享引用可以再借用为共享引用
let m1 = &mut r.1;     //错误，共享引用不能再借用为可修改引用 
```


1.  代码片段3：

```
let mut v = (136, 139);
let m = &mut v;
let m0 = &mut m.0;    //可以，可修改引用可以再借用为可修改引用
*m0 = 137;
let r1 = &m.1;        //可以，可修改应用可以再借为共享引用，且不与可 
                             //修改引用m0重叠

v.1;                  //错误，禁止通过其他路径访问可修改引用 
```


## 思考

1.Rust中是否不存在内存泄漏？

如下图所示，在Rust中可以创建引用循环，在此情况下，引用计数永远不可能为0，就会发生内存泄漏。  
![](https://res.cloudinary.com/practicaldev/image/fetch/s--3P8rtLIo--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qm8hruu89ac2mm2jwbsd.png)
为了避免该问题，Rust中引入了RefCell机制，该机制不在本文详细描述，大家可以搜索下相关文章，后续其他文章也会专门讲解。