---
{"dg-publish":true,"permalink":"/Program/Reactive/Rxjs 操作符/","noteIcon":"","created":"2025-03-06T21:28:25.982+08:00"}
---

RxJS 的操作符种类繁多，分类清晰，学习起来可以从 **功能类别** 入手。以下是对所有操作符的详细分类和简单说明：

---

## **1. 创建类操作符**
用于创建 Observable。

| 操作符           | 描述                                                                 |
|------------------|----------------------------------------------------------------------|
| **`of`**         | 从一组值创建一个同步发射的 Observable。                              |
| **`from`**       | 将数组、Promise、迭代器或 Observable-like 对象转换为 Observable。     |
| **`fromEvent`**  | 从事件创建 Observable，比如 DOM 事件。                               |
| **`interval`**   | 按固定时间间隔发出数字序列（无限 Observable）。                      |
| **`timer`**      | 在一段时间后触发（一次或间隔性）发出值。                             |
| **`range`**      | 发出一段连续的数字序列（如从 1 到 5）。                              |
| **`empty`**      | 创建一个立即完成的 Observable。                                       |
| **`never`**      | 创建永不发出任何值的 Observable。                                     |
| **`throwError`** | 创建立即抛出错误的 Observable。                                       |
| **`defer`**      | 延迟创建 Observable，直到有订阅时才实际执行。                         |
| **`generate`**   | 使用初始值、条件和更新逻辑生成 Observable。                           |

---

## **2. 转换类操作符**
用于转换数据。

| 操作符            | 描述                                                                 |
|-------------------|----------------------------------------------------------------------|
| **`map`**         | 对发出的每个值应用函数并返回新值。                                    |
| **`mapTo`**       | 将所有值映射为指定的常量值。                                          |
| **`switchMap`**   | 映射为新的 Observable，取消之前的订阅，仅订阅最新的 Observable。       |
| **`mergeMap`**    | 映射为新的 Observable，并发处理所有内部 Observable。                  |
| **`concatMap`**   | 映射为新的 Observable，顺序订阅内部 Observable。                      |
| **`exhaustMap`**  | 映射为新的 Observable，忽略新值直到当前内部 Observable 完成。          |
| **`scan`**        | 类似于 `reduce`，但会持续发出中间累积值。                              |
| **`buffer`**      | 收集发出的值，直到另一个 Observable 发出值时，将其作为数组发出。        |
| **`bufferCount`** | 每当收集到指定数量的值时，将其作为数组发出。                           |
| **`bufferTime`**  | 收集值并在固定时间间隔后发出数组。                                    |
| **`bufferToggle`**| 开关式收集缓冲区，使用一个开/关控制 Observable。                       |
| **`toArray`**     | 将所有值收集到数组后发出。                                            |

---

## **3. 过滤类操作符**
用于筛选和限制数据流。

| 操作符           | 描述                                                                 |
|------------------|----------------------------------------------------------------------|
| **`filter`**     | 根据条件过滤值，只发出满足条件的值。                                  |
| **`take`**       | 仅发出前 N 个值。                                                    |
| **`takeLast`**   | 仅发出 Observable 完成前的最后 N 个值。                               |
| **`takeUntil`**  | 发出值直到另一个 Observable 发出值。                                   |
| **`skip`**       | 跳过前 N 个值。                                                      |
| **`skipLast`**   | 跳过 Observable 完成前的最后 N 个值。                                 |
| **`distinct`**   | 过滤掉重复值（基于默认或指定的比较器）。                              |
| **`distinctUntilChanged`** | 仅发出相邻的不同值。                                         |
| **`debounceTime`**| 忽略快速发出的值，仅在指定时间内未发出新值时才发出值。                 |
| **`throttleTime`**| 限制发出值的频率，每隔指定时间发出一次。                              |
| **`auditTime`**  | 忽略一段时间内的值，仅发出最后一个值。                                 |

---

## **4. 合并类操作符**
用于组合多个 Observable。

| 操作符           | 描述                                                                 |
|------------------|----------------------------------------------------------------------|
| **`merge`**      | 并行合并多个 Observable，按发出值的时间顺序合并发出值。               |
| **`concat`**     | 串行连接多个 Observable，等待前一个完成再订阅下一个。                 |
| **`combineLatest`**| 当任何一个 Observable 发出新值时，将所有 Observable 的最新值作为数组发出。 |
| **`zip`**        | 将多个 Observable 的值按索引一一配对并发出（数组形式）。               |
| **`forkJoin`**   | 等待所有 Observable 完成并发出每个 Observable 的最终值（数组形式）。   |
| **`startWith`**  | 在源 Observable 之前插入一个值。                                      |
| **`withLatestFrom`** | 在源 Observable 发出值时，结合另一个 Observable 的最新值。          |

---

## **5. 错误处理类操作符**
用于处理错误。

| 操作符           | 描述                                                                 |
|------------------|----------------------------------------------------------------------|
| **`catchError`** | 捕获错误并返回新的 Observable。                                       |
| **`retry`**      | 在错误时重新订阅源 Observable（指定次数）。                          |
| **`retryWhen`**  | 在错误时重新订阅，但可以通过自定义逻辑控制何时重试。                   |
| **`onErrorResumeNext`** | 在错误时继续下一个 Observable。                                  |

---

## **6. 多播类操作符**
用于共享和复用 Observable。

| 操作符           | 描述                                                                 |
|------------------|----------------------------------------------------------------------|
| **`share`**      | 使 Observable 变为共享（热 Observable），多个订阅共享同一数据流。      |
| **`publish`**    | 将 Observable 转为 ConnectableObservable，需手动调用 `connect()`。     |
| **`shareReplay`**| 类似于 `share`，但会缓存最近发出的值以供新订阅者接收。                 |
| **`refCount`**   | 自动管理 ConnectableObservable 的连接和断开。                         |

---

## **7. 工具类操作符**
辅助调试和控制流。

| 操作符           | 描述                                                                 |
|------------------|----------------------------------------------------------------------|
| **`tap`**        | 允许在流中插入副作用操作（如日志、调试）。                             |
| **`delay`**      | 延迟发出值。                                                         |
| **`timeout`**    | 如果在指定时间内未发出值，则抛出错误。                                |
| **`finalize`**   | 在 Observable 完成或出错时执行清理操作。                               |
| **`repeat`**     | 重复订阅 Observable（指定次数或无限次）。                             |

---

这些操作符是 RxJS 的基础，但并不需要一次性掌握所有，实际开发中可根据需求逐步学习和应用。