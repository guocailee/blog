---
{"dg-publish":true,"permalink":"/Program/Mixed/V8 TurboFan的IR设计/","noteIcon":"","created":"2025-03-06T21:28:25.978+08:00"}
---

[如何理解v8的sea of nodes IR设计，能否推荐一些paper阅读？](//www.zhihu.com/question/290982869/answer/474629999)




> 作者: [RednaxelaFX](//www.zhihu.com/people/rednaxelafx)
> 软件工程师、主攻高级编程语言虚拟机的设计与实现


谢邀。啊我之前居然没在知乎写过这方面的内容，找了圈传送门没找到只好新码个回答了。

V8 TurboFan的IR设计我只是略知一二，是从别的编译器的有色眼镜下粗略观察得到的结论，所以本回答里提到的内容都请自行验证喔。

要说TurboFan就不得不提HotSpot VM的C2编译器。V8 JavaScript引擎跟HotSpot JVM有深深的共同血缘，runtime部分同样源自Self->Strongtalk这一系，而JIT编译器之间也是颇有关联的。V8 Crankshaft基本上就是HotSpot C1编译器（Client Compiler）移植到V8上的，而TurboFan则可以看作是HotSpot C2编译器（Server Compiler）在V8上的现代化再实现。其Sea-of-Nodes IR自然也是继承自HotSpot C2的创始人——Cliff Click大神的PhD作品。

八卦：TurboFan是Ben L. Titzer带队设计和实现的。Ben大大当初做TurboFan的动机（之一？）是看另外一个JS引擎的某个新编译器及其作者不爽，于是想自己也做个新编译器干翻对方 >_<|||

结果双方都用了比较不传统的[基于图](https://www.zhihu.com/search?q=%E5%9F%BA%E4%BA%8E%E5%9B%BE&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A474629999%7D)的IR。对方是谁我就不（用）说了。

  

要理解Sea-of-Nodes IR的基础设计和特性，Cliff Click大神在90年代初的论文是必读的。其中我自己最喜欢的一篇入门论文是 From Quads to Graphs，是Cliff大神1993年发的文，通过例子很生动地阐述了一个[线性IR](https://www.zhihu.com/search?q=%E7%BA%BF%E6%80%A7IR&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A474629999%7D)通过怎样的演进就变成了Sea-of-Nodes。

  

[Sea-of-Nodes IR](https://www.zhihu.com/search?q=Sea-of-Nodes%20IR&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A474629999%7D)可以说是一种Program Dependence Graph（PDG），但是具体在IR里嵌入的依赖信息不一样，而且SoN强调基础IR要是SSA形式的而泛指的PDG并没有这样的要求（有许多PDG变种的基础IR也是SSA形式的就是）。

粗略来说，SoN IR的精髓就是“在统一的表达形式下，把分析进行彻底”。用这样的IR所表达的程序里完全不需要“变量”的概念了，一切都是经过透彻分析后的“值”。各种操作并不操作变量，而是从依赖获取输入值，运算后产生新的值。每个“值”自身会携带足够依赖信息来判明它在怎样的路径上有效，依赖的数据输入是哪些；通过一个值的局部信息就足以对它做很多分析和优化（这叫做[稀疏分析](https://www.zhihu.com/search?q=%E7%A8%80%E7%96%8F%E5%88%86%E6%9E%90&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A474629999%7D)，sparse analysis），而不需要[顺序遍历](https://www.zhihu.com/search?q=%E9%A1%BA%E5%BA%8F%E9%81%8D%E5%8E%86&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A474629999%7D)整个IR才能获取足够信息来做这样的分析/优化（这叫做[密集分析](https://www.zhihu.com/search?q=%E5%AF%86%E9%9B%86%E5%88%86%E6%9E%90&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A474629999%7D)，dense analysis）。

对典型的、受限的局部变量而言这是比较好理解的：编译器可以彻底分析出某个[局部变量](https://www.zhihu.com/search?q=%E5%B1%80%E9%83%A8%E5%8F%98%E9%87%8F&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A474629999%7D)到底每一次在哪里被赋予了怎样的值，从而建立每次赋值（或者叫做定值，definition）所依赖的输入的所有信息。

而对结构体的字段（或者说对象的字段）、全局变量（或者说[静态变量](https://www.zhihu.com/search?q=%E9%9D%99%E6%80%81%E5%8F%98%E9%87%8F&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A474629999%7D)）来说，这就比较有趣了：这里肯定得有变量啊，SoN IR要怎么表达呢？

  

当我们把几种完成度比较高的SoN IR的实现放在一起看的话，它们所记录的依赖信息分别是：

*   Cliff PhD thesis / HotSpot C2：Control, I/O, Memory, Frame Pointer, Return Address, Data, (Other Scheduling Constraint)

*   但其实在C2里只用了Control, Memory, Data, (Other Scheduling Constraint)

*   libFirm：基本上跟Cliff的PhD论文一样，在其基础上有所扩展
*   Graal：Control, Data
*   TurboFan：Control, Data, Effect

嗯？C2用了Control, Memory, Data，为啥到Graal和TurboFan的不一样了呢？最重要的是，“Memory”去哪里了？

答案是：后两者是对前者的简化。

  

C2里，Control Dependence（控制依赖）可以看作是常规[控制流图](https://www.zhihu.com/search?q=%E6%8E%A7%E5%88%B6%E6%B5%81%E5%9B%BE&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A474629999%7D)反过来画。不是基本块持有一个[线性列表](https://www.zhihu.com/search?q=%E7%BA%BF%E6%80%A7%E5%88%97%E8%A1%A8&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A474629999%7D)里面装着节点，而是每个节点都记录着自己的控制依赖是谁——要哪个[前驱控制节点](https://www.zhihu.com/search?q=%E5%89%8D%E9%A9%B1%E6%8E%A7%E5%88%B6%E8%8A%82%E7%82%B9&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A474629999%7D)执行到的时候我才可以执行。

Data Dependence很简单，就是[use-def链](https://www.zhihu.com/search?q=use-def%E9%93%BE&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A474629999%7D)，换句话说一个节点的数据输入都是谁。例如说 a + b，这个“+”的数据依赖就是a和b。C2里为了操作方便实现的是双向的use-def + def-use链，但就实现IR语义而言这里最重要的还是use-def链。

Memory Dependence，这就有趣了。C2的Sea-of-Nodes IR把“内存状态”（Memory）看作一个大的黑盒子的值，而Store/Load这样的对内存的操作都是对这个大值的其中一个给定的切片做了运算，产生一个新的[切片值](https://www.zhihu.com/search?q=%E5%88%87%E7%89%87%E5%80%BC&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A474629999%7D)。把内存（的切片）看作值之后，它就可以跟普通值一样放进SSA形式来描述了，于是各种本来存在内存里的变量就也可以通过“值”的方式来描述。所以C2 IR的内存相关子图也叫做Memory Graph或者Memory SSA。当某个内存操作需要观察到某个之前的内存操作所产生的副作用，在C2 IR里就表达为这个节点的Memory Dependence是上一个节点所生成的Memory切片值。

例如说，下面这段[伪C代码](https://www.zhihu.com/search?q=%E4%BC%AAC%E4%BB%A3%E7%A0%81&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A474629999%7D)：

```c
// global variable x, local variable y
x = 42
y = x
x = 255
```

这段代码对应的（简化的）IR就是：

```text
%store1 = Store %global_memory, %offset_x, 42  // store1: a new memory slice value
%load1  = Load  %store1, %offset_x             // load1: the loaded value
%store2 = Store %store1, %offset_x, 255
```

这个例子里，简化的[Store节点](https://www.zhihu.com/search?q=Store%E8%8A%82%E7%82%B9&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A474629999%7D)接收三个参数：memory state, offset, value。

第一个Store节点接收的memory state是global memory，就是个大黑盒子。这个节点的语义是：从全局内存状态这个大黑盒子里，对offset这个位置赋值为value，并产生一个新的内存切片值，这个切片值只针对offset有效，里面含有的值就是刚才赋值进来的value。

然后[Load节点](https://www.zhihu.com/search?q=Load%E8%8A%82%E7%82%B9&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A474629999%7D)以刚才的Store节点所生成的内存切片值为memory输入，并读取位于offset_x的值。Load节点并没有对内存做修改，所以没有产生新的内存切片值

然后第二个Store节点接收的memory输入是前一个Store所产生的内存切片，以及offset和新的值，并产生新的针对offset_x的内存切片值。

这样，通过在内存操作节点上记录内存依赖信息，我们就可以知道这些操作必须遵守的顺序：一个Load肯定会在它前面的Store产生了副作用之后才执行，因为前面的Store对这个Load来说是一个输入的依赖。两个Store之前的顺序也是很直观就固化了下来。两个Load如果依赖同样的内存输入则顺序随意，如果依赖不一样的内存输入则是毫不相关的两个运算，更加是顺序随意。

但C2 IR里没有显式记录anti-dependence（Write-After-Read）——后面的一个Store必须要调度在前面的一个Load之后，这个信息要在IR上额外运算得到。

前面简化的例子忽略了内存操作节点的控制依赖，加上来的话在控制流的汇合点上就会看到内存切片值也通过Phi节点来交汇在一起，就跟普通的局部变量当作值看的处理一样。

  

这块上，libFirm的设计跟C2的很相似，最大的不同是libFirm的Load也产生内存切片值，于是Load-Store（anti-dependence）顺序就在IR层面上显式固定下来了。

  

说了半天，Graal和TurboFan呢？

  

这Memory SSA并不是所有优化都要用到，但所有对IR做匹配（pattern matching）和对IR做变换（transformation）的逻辑都必须要识别和维护Memory SSA的属性，在现实中这是件很麻烦的事，一个不小心破坏了Memory SSA的属性整个IR就坏掉了。

  

Graal的解决办法是：根本不在IR上记录Memory SSA，而是把C2的Memory Dependence给融合到Control Dependence里，于是不但一般基本块的头会作为Control Dependence，连Store/Load这样的内存操作也会作为Control Dependence来处理。乍一看这不是很无厘头么，一个Store怎么能跟一个If相提并论。但再想想也有点道理，反正都是描述调度顺序上的限制，放一起又何妨？

这样，当Graal在不做内存相关优化的时候，内存操作就是被Control Dependence钉死在IR图里的；而当它进入内存操作优化的阶段时，会在IR之外用额外的数据结构记录类似Memory SSA的信息，此时某些内存操作节点的Control Dependence就会放宽松，让这些内存操作节点可以更自由的浮动（改变调度顺序）。现实中这是非常方便的，也是我喜欢Graal IR的一点。

  

TurboFan的处理方式也是类似，只不过算是在C2跟Graal之间：Control跟Effect还是分开的，Control还是对应传统带CFG的IR的基本块，Effect则记录“副作用的顺序”——主要就是内存操作的顺序。它只记录顺序而不必维护SSA形式的其它特性，所以并不强制要求要通过[Phi节点](https://www.zhihu.com/search?q=Phi%E8%8A%82%E7%82%B9&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A474629999%7D)来处理交汇啥的。

于是在不对内存操作做优化的时候，IR里就没有暴露细粒度的[内存切片](https://www.zhihu.com/search?q=%E5%86%85%E5%AD%98%E5%88%87%E7%89%87&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A474629999%7D)信息，也就不必维护这个信息；等到需要对内存操作做优化时，在外部创建内存切片的信息并相应地做优化，优化完的结果就是Effect所记录的顺序会比优化前更精简更准确。

  

大致就是这么个故事…写得很乱所以请多多包涵 >_<|||

  

传送门：

[From Quads to Graphs: An Intermediate Representation's Journey​citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.51.4637![](https://pic3.zhimg.com/v2-c7e9be6099085b327d40b721e0ec5b46_ipico.jpg)
](https://link.zhihu.com/?target=http%3A//citeseerx.ist.psu.edu/viewdoc/summary%3Fdoi%3D10.1.1.51.4637)

[http://hllvm.group.iteye.com/group/topic/39493#post-258162​hllvm.group.iteye.com/group/topic/39493#post-258162](https://link.zhihu.com/?target=http%3A//hllvm.group.iteye.com/group/topic/39493%23post-258162)

[如何设计三地址中间代码的数据结构，以便于基本块分析和代码优化？324 赞同 · 39 评论回答](https://www.zhihu.com/question/33518780/answer/56731699)

[如何评价只有 LLVM 10% 代码的 QBE？232 赞同 · 5 评论回答![](https://pic2.zhimg.com/e01ece15ee5b46d1368287ca4835de23_ipico.jpg)
](https://www.zhihu.com/question/43956056/answer/97177805)

[编译器具体实现中比较巧妙的思想有哪些？272 赞同 · 19 评论回答](https://www.zhihu.com/question/27943883/answer/39035471)

（一大堆未完待续，俺那个汗呐）

[编辑于 2018-08-21 20:28](//www.zhihu.com/question/290982869/answer/474629999)
