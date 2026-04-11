---
{"dg-publish":true,"permalink":"/Program/Mixed/面试官灵魂拷问：为什么 SQL 语句不要过多的 join？/","noteIcon":"","created":"2026-03-28T20:17:34.336+08:00","dg-note-properties":{}}
---

[酱紫君](//www.zhihu.com/people/GalAster)




`JOIN`是一种虚调用，`SQL` 为什么不应该 `join` 和你 `Java` 为什么不要写一堆 `interface` 然后 `XXImpl` 是一个道理。

`virtual` 这种东西最大的危害不是慢，实际上速度并不慢，最主要的是阻止了优化机会，缓存失效外加分支预测失败那就慢出几个数量级了。

`JOIN`也是一样的，直接测没感觉多慢，实际业务中真的慢就是因为计划引擎没法自动优化。

不但不吃常规优化，甚至还反优化，提前优化可能最后效果反而不好，编译器看到直接麻了。

* * *

首先你要知道，一切业务都是 [QKV](https://zhida.zhihu.com/search?content_id=747887761&content_type=Answer&match_order=1&q=QKV&zhida_source=entity)，人肉负责 QK，机器负责 KV。

无论是数组、链表、向量、图、对象，还是数据库里的关系表，全都可以划分成直接查询和间接查询。

直接查询的特点就是常数复杂度，比如 Array，比如 Vector，一次寻址就可以获取值。

间接查询与之相反，至少是对数复杂度，执行时间是不固定的，编译器在内部的 Cost 模型没法当成一个常量折叠掉。

传播来传播去相当的麻烦，并由此衍生出种种优化策略。

* * *

SQL 中的一张表在物理上是一个堆（[Heap Table](https://zhida.zhihu.com/search?content_id=747887761&content_type=Answer&match_order=1&q=Heap+Table&zhida_source=entity)），一般来说，每一行都有一个物理地址。

类似于 `ctid=(0,1)` ，表示在第0个数据块的第1个位置，由此硬盘才知道去哪取数据。

这是第一层 KV。

然后数据库的索引是另一种 KV，是为磁盘I/O优化过的KV，它的 `Key`是索引列的值，它的`Value`就指向主表中实际数据行的物理地址`ctid`。

数据库分析里一般把索引 KV 看成常数，实际上索引由于物理介质和代码实现差别，也是有一定速度起伏的。

简单起见，暂时把所有的 KV 都看成常数查询。

* * *

### Index Scan

那么从你写的 Query 就会从最原始的 Key，经过 KV -> KV -> KV -> KV 找到最终的 Value。

SQL 引擎的目标就是让你的跳转在时间上最快，注意这里跳的少不一定时间上快，优化一定优化的是时间。

假如你有个 `customer_id`的 Unique 索引，当你写：

```sql
EXPLAIN ANALYZE SELECT * FROM customers WHERE customer_id = 114514;
```

那么 Planner 知道至多返回一行，就会直接翻译成一次 `Index Scan`。

这个东西和编程语言里的静态调用一样是非常快的。

SQL 引擎知道用 114514 这个Key，在几次磁盘 I/O内，就能精准定位到 ctid，然后去堆表中取出Value。

这个过程的成本是 O(log⁡N)O(\\log N)O(\\log N) ，对于一般的 TB级小表来说，一般就是几次常规 I/O 操作。

整体结果数量明确，路径确定，开销可预测，优化完全可以给到位。

* * *

但是你有 `JOIN` 就不一样了，比如你写：

```sql
EXPLAIN ANALYZE
SELECT o.order_id, c.customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date > '1919-08-10';
```

Planner 是不可能静态优化的，因为这里存在运行时的不确定性，不能胡乱优化。

XJB 乱优化会有一个资源放大的问题，反而会更慢。

当 Planner 看到一个`JOIN`时，必须从多种`JOIN`算法中选择一个最优的，并且这个选择过程本身也有巨大的开销，并且需要依赖于`ANALYZE`命令收集的统计信息。

* * *

### Nested Loop Join

如果外表非常小（GB级），且内表的 `JOIN`键上有索引时，就应该选择 **`Nested Loop Join`。** 

直接遍历`FROM`后的第一张表每一行，然后对于每一行，直接获取其 `JOIN KEY`，然后去内表的索引中进行一次 `Index Scan`来查找匹配项就好了。

虽然实现很傻但是这个其实是最快的，且不会有额外的开销。

但是前提是外表不能太大，否则循环次数就太高了。

* * *

### Merge Join

如果两个表都已经按照 `JOIN KEY` 排好序了，那么 Planner 就会选择 `Merge Join`，用两个指针同步地扫描两个表。

资源消耗非常低，查询速度非常快。

但是排序本身并不是一个便宜的操作，而且你也不可能建满索引。

所以一般实际业务中除非特意优化过，不然一般很少触发这个路径。

* * *

### Hash Join

如果外表很大，那么 Planner 就会选择**`Hash Join`。** 

首先选择较小的内表，直接在内存中基于`JOIN KEY` 构建一个 `Hash Table`。

接着逐行扫描较大的外表，再次获取 `JOIN KEY`，然后去内存中的 `Hash Table` 里进行一次 `Probe` 查询，这是一个`O(1)`的操作。

因此整体上能得到一个加速。

这就相当于编程语言中创建了一个 `vtable`，可以减掉一次 KV，有效加速后续查询。

但是你的内存至少得装得下这个临时的加速内表。

如果跑着跑着发现装不下，总不能直接奔溃了吧，前面都查了那么多了。

那么就会选择溢出到磁盘，此时速度基本上得下降三个数量级。

* * *

### 优化爆炸

如果只是两个表 `JOIN`，问题还算是比较简单的。

但对于`A JOIN B JOIN C JOIN D`呢，Planner 不仅要为每个`JOIN`选择算法，还必须考虑 JOIN的顺序。

到底是用 `(A JOIN B) JOIN (C JOIN D)` 还是 `((A JOIN C) JOIN B) JOIN D`？

JOIN 顺序的可能性是 (n-1)!(n-1)! ，以阶乘级增长的，而且估算错一个就优化到空气上去了。

像是超过12个表的 JOIN，PostgreSQL会有[GEQO](https://zhida.zhihu.com/search?content_id=747887761&content_type=Answer&match_order=1&q=GEQO&zhida_source=entity) 这样基于遗传算法的高级优化器去做决策。

而你用的某某数据库估计是没有这种运行时优化的。

* * *

所以说 Planner 一定要根据数据规模选择合适的算法，用专业术语叫做基数（[Cardinality](https://zhida.zhihu.com/search?content_id=747887761&content_type=Answer&match_order=1&q=Cardinality&zhida_source=entity)）。

有些国产数据库对 Cardinality 的认知和 Rust 一样天真，就知道个下限，然后不是 Loop Join 就是 Hash Join。

你说你这能快吗？

呃，至少比 Cardinality 概念都没有的数据库好，while loop 一把梭，就这还卖好几万。

一眼就知道充满了没被知识污染过的天真。

你不慢谁慢？

* * *

Planner 很完美其实也没什么卵用，因为`JOIN`的执行过程本身也是资源密集型的。

没被知识污染过的`Nested Loop`经常会导致对内表重复进行任意 I/O 查询，完全打乱 OS 调度，这被称为I/O放大，好好优化减少掉 80% 不是问题。

`Hash Join`或`Merge Join`在排序阶段会有一个内存竞争，虽然总的来说是够的，但就是莫名其妙溢出到磁盘，然后一大堆临时文件读写，性能当场爆炸。

多表 JOIN 产生的中间结果集可能比任何一个原始表都大得多，如果估算的不对，就会进一步加剧内存和 I/O 的压力。

大多数执行计划的性能都是有个最优工作区间的，超出这个区间，跑都跑了，总不能中断啊，只能想办法咬牙继续硬算。

* * *

另一方面计算哈希值、比较键、在内存中移动数据等等，这些都是 CPU 密集型操作，OS 还要时不时过来没事找事。

所以说 JOIN 发下去 CPU，内存，IO 三个都吃。

一个完整的 Cost 模型得综合考虑每种算法的 CPU 成本、I/O成本，外加表的行数、宽度、数据分布的倾斜度等统计信息，外加历史运行情况的微调，最终选择一个好的执行计划。

所以有的全推给 SQL 引擎，不如自己多做些小优化。

* * *

### Devirtualization


具体的优化手段各个领域其实都是互通的，编程语言里面昂贵的虚调用能通过去虚拟化变成廉价的静态调用，数据库也是一样的道理。

如果你每次都需要外表 id 的同时带 name， 那你直接把这个 name 冗余到内表里就行了。

原本写：

```sql
EXPLAIN ANALYZE
SELECT o.order_id, c.customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE c.customer_name = '逸一时误一世';
```

现在写：

```sql
SELECT order_id, customer_name, price FROM orders WHERE c.customer_name = '逸一时误一世';
```

这样用空间换时间就省去了一次间接查询。

DBA 写周报就会说，通过什么 [Denormalization](https://zhida.zhihu.com/search?content_id=747887761&content_type=Answer&match_order=1&q=Denormalization&zhida_source=entity) 优化了多少多少。


### Inlining

当然 Denormalization 数据更新是个问题，只适合只读数据。

编程语言里一个常见优化手法就是把小对象内联了，数据库里这个手法叫做 [Covering Index](https://zhida.zhihu.com/search?content_id=747887761&content_type=Answer&match_order=1&q=Covering+Index&zhida_source=entity)（覆盖索引）。

常规索引扫描后，引擎还需要根据索引中存储的`ctid`返回主表获取查询所需的其他列，这个过程会显示为 `Heap Fetch`。

这个 I/O 有时候挺慢的，因为这玩意儿可能不在 Cache Line 甚至可能在 Code Section 里。

那么我们其实可以在创建索引时就用`INCLUDE`子句将查询所需的所有列都包含在索引的叶子节点中。

原本写：

```sql
CREATE INDEX idx_orders_customer ON orders (customer_id);
EXPLAIN SELECT order_date, total_amount FROM orders WHERE customer_id = 114514;
-- Index Scan + Heap Fetch
```

现在写：

```sql
CREATE INDEX idx_orders_customer_covering ON orders (customer_id) INCLUDE (order_date, total_amount);
EXPLAIN SELECT order_date, total_amount FROM orders WHERE customer_id = 114514;
--- Index Only Scan
```

SQL 引擎在索引中找到满足条件的叶子节点后，立即就会发现所需的所有数据都已经在索引里了，它完全不需要再访问主表。



### Memoization

最后一种优化手法即使缓存和记忆化，用数据库的术语叫做 [Materialized View](https://zhida.zhihu.com/search?content_id=747887761&content_type=Answer&match_order=1&q=Materialized+View&zhida_source=entity)（物化视图）。

你只要定期执行`REFRESH MATERIALIZED VIEW`来更新缓存就行了。

PostgreSQL 比较好的一点就是有 `MATERIALIZED CTE` ，因为很多时候你不跑业务你是不知道谁 hot 的，乱建表建索引反而会降低性能，有这个东西会方便很多。

虽然优化很重要，但提前优化是万恶之源。


### JIT

写了这么多，其实也可以看出来，一个高效的 SQL 能静态决定的事情其实并不多，具体情况还得要跑起来才知道。

这些全都属于 JIT 优化，PostgreSQL 本身就有基于 LLVM 的 JIT 加速，可以跳过解释环节，直接加速热点数据变形。

虽然 `LLVM` 的 JIT 也是废物一个，但是有总比没有强。

其他数据库也有对应的技术，而且比 PostgreSQL 都要精细的多。

比如 SQL Server 有 Adaptive Query Processing，会根据 CE Feedback，Memory Grant Feedback 进行 Adaptive Join。

Oracle 也有类似的 Statistics Feedback，会自动进行 Automatic Reoptimization。

这些都是真正需要底蕴的优化。

国产数据库大多还处于卷基建把基本查询的常数降下来的朴素优化认知阶段，离这些真正卷生卷死的优化技术还有很长的路要走。

在这样的背景下，少用点 JOIN，多写些人肉优化总是没错的。
