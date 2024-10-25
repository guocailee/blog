---
{"dg-publish":true,"permalink":"/Program/FrontEnd/React Hooks 是否可以改为用类似Vue3 Composition API 的方式实现/","noteIcon":""}
---

**不能，因为是很不一样的[心智模型](https://www.zhihu.com/search?q=%E5%BF%83%E6%99%BA%E6%A8%A1%E5%9E%8B&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A1125724740%7D)（Mental Model）。**我觉得很多同学只关注到了这两套 API 在功能上都能复用逻辑的相似点，而低估了两个框架体系「大背景」上的差异。
1.  文观点不代表公司。我是觉得圈子里不认同 Hooks 的声音太多了（比如  [@徐飞](//www.zhihu.com/people/c5198d4e9c0145aee04dd53cc6590edd) 叔叔、  [@贺师俊](//www.zhihu.com/people/3ec3b166992a5a90a1083945d2490d38) 贺老、  [@题叶](//www.zhihu.com/people/790dccce26904cdcd11b0fad3bac37b7)  同学等老朋友 no offensive），所以自愿出来平衡一下。
2.  是我确实好久没有实际写前端了 ，React Hooks 实战还不多，Vue 3 只草草略读了 [Composition API RFC](https://link.zhihu.com/?target=https%3A//vue-composition-api-rfc.netlify.com/) 与之前中文的 [Vue Function-based API RFC](https://zhuanlan.zhihu.com/p/68477600)（所以对细节并不太熟悉。）欢迎大家务必指正、补充（与要求补充）。

## 引言

「框架 / 库是编程语言上的抽象」，并不意味着框架的设计就无法跳脱出其实现语言的习语（idioms）与编程范式（paradiams）。

得益于 JavaScript 几个非常 Lisp 的特点：一等公民函数、动态类型、一定的宏支持（比如 Babel），这几年前端框架的发展可以看到很多编程语言设计的思路：框架成为了由 DSL 与 API 构成的特定语法（syntax）、从 JavaScript 中扬弃以及由 API 附加的语义（semantics）、支撑这套体系运作的运行时（runtime）、以及所表达的心智模型（mental model）的结合。

## Vue 3, "Reactive (Closure-based) OOP"

先来看 Vue（本文中的 Vue 主要指使用 Composition API 下的 Vue）

```js
const Counter = {
  setup(initialProps) {
    const count = reactive({count: 0}) // or `ref(0)`
    const inc = () => { count.value++ }
    return {count, inc}
  }
  template: "..."
} 

```

Vue 为组件挑选的语义是「对象」：Composition API 的`setup` 只会调用一次并返回一个对象合并到 `Counter` 组件上，这个对象与其成员全都是持久的引用，包括保存在 `inc` 闭包中的状态 `count` 也是持久的。渲染则是对组件上的`template` 域的具象化。

Vue 附加的核心语义是（基于可变数据的）「响应式（reactive）」：状态 `count` 是一个响应式对象，`inc` 进行状态更改的方式是对 `count` 直接修改，状态更改的结果是执行所有观察者（watcher）的逻辑，包括重渲染和执行副作用（`watchEffect`) ，都是基于这个语义纳入[数据流](https://www.zhihu.com/search?q=%E6%95%B0%E6%8D%AE%E6%B5%81&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A1125724740%7D)。

有的同学（比如题主）说，如果改成返回一个 `render` 函数，**直接利用闭包来保存组件变量**，你还说这是对象的语义吗？

```js
return (props) => <a onClick={inc}>{count.value}</a>

```

_是的_。Vue 的实现需要保持这个函数与其闭包的引用不变（referential identity）来满足状态被持久化的语义，是 JavaScript 用闭包模拟对象私有属性的经典模式[\[1\]](#ref_1)。（「闭包是穷人的对象，对象是穷人的闭包」）

为了帮助你理解，如果我们有一门假想的 Vue 语言的话……

```js
// hypothetical Vue lang
component Counter (props) {    // constructor
  @reactive count = 0          // field
  inc() { count.value ++ }     // method
  render() { return <a onClick={inc}>{count.value}</a> }
}

```

是不是有基于类的 OOP 内味了？只不过 Vue 的对象是基于单例（或者闭包）而非类（或原型）实现，以及成员是被施过 reactive 魔法哒！这里我们展示了如何从概念上将 Vue 归约到 OOP 的心智模型，不过需要注意得是，Vue 整个体系（考虑状态管理、数据流控制）的心智模型有很多 FP 的东西在里面，仍然和传统观念（比如 Java）的 OOP 有很大差别，[\[2\]](#ref_2)[\[3\]](#ref_3)。

Vue 运行时的核心是 dependency tracking（依赖追踪），首先它使得 reactive 语义对用户相对 implicit（隐式），依赖都是自动收集的，大大降低了用户的心智负担。其次呢它非常细的跟踪粒度配合 Vue 使用静态化程度比较高模板使得重渲染自动就可以做到非常精准。

总结起来，Vue 在组件方面的心智模型仍然是「拥有数据与行为且自响应式的对象」，只要照着这个思路去想，就比较好理解「为什么 Vue 的状态可以使用可变数据结构」、「为什么 Vue 需要`ref` 包装值类型」，以及 RFC 在对比 React Hooks 时提到的「为什么 Vue 更接近大家习惯的 JS 」（这点比较主观就是了）、「为什么 Vue 的 GC 压力会更小」、「为什么 Vue 不需要手动声明依赖」等优势的由来了。

## React, "Purely (Semi-Monadic/Algebraic) FP"

再来看 React（本文中的 React 主要指 Hooks API 下的 React）

```js
function Counter(props) {
  const [count, setCount] = React.useState(0);
  const inc = () => setCount(count + 1)
  return <a onClick={inc}>{count}</a>
}

```

React 为组件挑选的语义是「函数」，每次渲染都是对 `Counter` 这个函数的一次真实调用。每次 `useState` 都会执行并从 React 那取出当前的状态给 `count`，每次也都会创建一个新的 `inc` 函数（故其闭包中捕获的也是新的 `count` 值）。

React 附加的核心语义是一个副作用受控的「执行上下文（evaluation context）」，通俗得说就是 React 这个运行环境：状态 `count` 每次都要从 React 上下文中取出，`inc` 对状态更改的方式是用 `setCount` 更新上下文里的内容，状态更改的结果是这个函数会被重新调用，调用时函数就会从新的上下文中获得新的状态、进行重渲染和安排上（schedule）受上下文控制的副作用（`useEffect`) 。

为了帮助你理解，如果我们有一门假想的 React 语言的话……

```js
// hypothetical React lang Ⅰ
component Counter = (props) =>      // function
  @context.state {                  // context provides `get_or` and `put`
    count <- get_or(0)              // get from context (or use default value)
    let inc = () => put(count + 1)  // callback can update the context
    return <a onClick={inc}>{count}</a>
  }

```

是不是有基于 Monad 的 Haskell 内味了？只不过 React 把 API 做得完全不需要你弄懂这些复杂的东西[\[4\]](#ref_4)。如果你不熟悉 Monad 这个纯 FP 的概念，我们可以先不严谨[\[5\]](#ref_5)得把它当做文中的「上下文」。扔出 M-bomb 的原因是大家通常把它作为在纯 FP 中处理副作用的标杆，帮助我们展示如何把 React 归约到纯 FP。

有的同学（比如  [@题叶](//www.zhihu.com/people/790dccce26904cdcd11b0fad3bac37b7) ）会疑惑，这怎么跟我认得的「纯函数」不一样呢，这也是「纯函数式编程」吗？其实如果我们把语法糖展开就会变成：

```js
component Counter = (props) =>
  context.state("count").get_or(0).then([count, put] => {  // `then` is monadic bind.
    let inc = () => put(count + 1)         
    return <a onClock={inc}>{count}</a>
  }).unwrap() // assuming it's safe.

```

有没有想到同样被称作 Monad，具备异步上下文的 Promise？

再（过度）简化一些，你可以想象成最直白的 state-passing 风格（实际上 2018 年时 React 团队就这么考虑过类似的 API [\[6\]](#ref_6)，也是 Seb 的理论基础之一[\[7\]](#ref_7)）:

```js
component Counter = (props, context: {stateMap) => {
  let count = context.stateMap.get("count", or=0);
  let inc = () => context.stateMap.set("count", count + 1); // functional update
  return <a onClick={inc}>{count}</a>
}

```

不过，React 从实现到 API 设计都更靠近与追求[\[8\]](#ref_8)的心智模型是一个相对较新的纯 FP 概念 —— Algebraic Effect（代数作用），虽然名字听起来相当迷惑，但其实它在描述副作用上比 Monad 反而更不花哨（少一些 ceremony），理解起来也更加容易，Dan 有一篇给 JSer 看的很易懂的博文[\[9\]](#ref_9)并且有中文翻译[\[10\]](#ref_10)。我们可以先把它当作「可以重新恢复的 `try-catch` 」

为了帮助你理解，如果我们又又又又有一门假想的 React 语言的话……

```js
// hypothetical React lang Ⅱ 
component Counter = (props) => {
  let count = perform getState("count"),or=0); // 1. `perform` "throw" effects to the context
                                               // 4. resume with the continuation to here
  let inc = () => perform putState("count", s=count + 1);
  return <a onClick={inc}>{count}</a>
}

// call site
try <Counter /> handle  // 2.try-handle pattern match effects 
                        // 3. get state from the context and then resume
| getState(key, or) => resume with context.state(key) ?? or 
| putState(key, s) => context.state(key)=s; resume with void

```

是不是有感觉一些了？我们从组件里「扔出去」去更改 「执行上下文」里的状态，然后再「恢复」回来……

即便 React 已经很努力的降低了 API 的门槛，但其思维的愈加纯函数式确实会在更多程序员眼里非常「离经叛道」。

所以为什么我们需要「纯函数式编程」？抛开大家可能已经熟悉的声明式、数据流清晰、局部推理、易于组合外，其背后的学术理论支撑使得其在编译期静态分析与优化、并发调度异步场合都有很高的上升空间（高理论上限）。

React 现在运行时侧重的核心是 cooperative multitasking（协作式多任务），以实现 concurrency（并发）、schedule（调度）等底层能力。很多同学只听说过后端的高并发，其实像多任务操作系统这样的「终极 UI」就是一个高并发且依赖诸如分时（time-slicing）、优先处理（re-priortizing）等进程调度的场景。React 希望把这些技术带给 UI 开发者（比如 Suspense，比如 Selective Hydration，比如 RN 新架构中的 Fabrics），第一步是运行时用 Fiber 架构重写不依赖原生调用栈，第二步就是用 Hooks 解决 Class API 在纯度上约束力不够的问题。不纯的组件在 React 并发模式下很容易出现数据竞态（data race）的问题。

总结起来，React 在组件方面的心智模型是「副作用受上下文托管的纯函数」，只要照着这个思路去想，就比较好理解「为什么 React 中倾向于使用不可变数据结构」、「为什么 useEffect 默认会执行 cleanup 来保持幂等性」、「为什么 React 需要 `useRef` 这样的跨渲染 ref cell 机制来做 mutable ref 」、「为什么 React 的性能优化是 `useMemo`, `Memo` 这样 FP 风格的 memoization 」、「为什么 React 需要 `useMemo` `useCallback` 来保持 referential identity 」、「为什么 React 需要用依赖列表来进行 cache invalidation」等问题了。

## 补充

2016 年底时我就觉得 React 和 Vue 的（一个）终极区别在于「可变」还是「不可变」。

![](https://pic3.zhimg.com/80/v2-22e50932c60262bf381456de3017b217_720w.jpg?source=1940ef5c)

Seb 在 Hooks 发布后收到一些质疑的 brain dump[\[11\]](#ref_11) 里写到：

> It's interesting because there are really two approaches evolving. There's a **mutable + change tracking** approach and there's an **immutability + referential equality testing** approach. It's difficult to mix and match them when you build new features on top. So that's why React has been pushing a bit harder on immutability lately to be able to build on top of it. Both have various tradeoffs but others are doing good research in other areas, so we've decided to focus on this direction and see where it leads us.

不全部翻译了，说得是整个「大前端」社区里最主要的两条道路分歧：

-   可变 + 变更追踪。包括 Vue，Angular，
-   不可变 + 引用相等性。包括 React，Elm，(Flutter?)

这个分歧其实与我之前行文的侧重点「为组件挑选的语义」其实是对偶的：

-   前者是对传统 Imperative Programming（包括 OOP）思路的一种增强，加入了 Reactivity。
-   后者则是传统 Functional Programming 在 UI 开发领域的发扬光大（Functional Reactive Programming?)，只不过 React 是用一种比较「超越 JS 语言」的方式去实现得。

这两条道路从底子就很不同，所以才造成了 React 和 Vue 在大家眼里的渐行渐远吧。

不过最近多看了一些 Svelte、 SwiftUI[\[12\]](#ref_12)和 Jetpack Compose[\[13\]](#ref_13)也开始有了一些[殊途同归](https://www.zhihu.com/search?q=%E6%AE%8A%E9%80%94%E5%90%8C%E5%BD%92&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A1125724740%7D)的感觉，Props 无论是跟着 View 销毁还是函数参数总是暂时性的输入，States 无论跟着组件实例还是置外总必须是持久化的，至于怎么判断更新，像 `array.push` 这种 mutable state 的场景总是不好 track 得，于是就只能各显神通了：React 想通过 reference equality 自动、Vue 3 想通过 Proxy 自动，但其实只要能把 change set 搞出来就行，Vue2/Svelte/SwiftUI/Compose 这些让用户手动给提示得不是也工作得好好得吗？只要能把变更集算出来传递给视图层，那视图层就只管更新（rerender/rebuild/recomposite）就是了。

## 补充 2

如果是在重度依赖 Flux (Vuex/Redux, whatever) 的场景，可能 Vue/React 会更像是一个只负责渲染的 dumb/passive layer，这种时候上文说的 Vue/React 的差异会显得不明显，因为大部分的状态管理（state management）都已经扔到更外层去做了。

不过，考虑需要组件内聚的场景（即组件自己有私有状态，需要 self-conatined）以及 React Hooks / Vue Composition APIs 开始接管更多（除状态之外的，比如 IO）副作用，这种差异只会变得越来越明显得。

以上。

## 参考

1.  [^](#ref_1_0)JavaScript 模块化七日谈 - 黄玄的博客 | Hux Blog [https://huangxuan.me/2015/07/09/js-module-7day/](https://huangxuan.me/2015/07/09/js-module-7day/)
2.  [^](#ref_2_0)如何理解尤雨溪在 2019 VueConf 上所讲的 UI 类框架很少使用面向对象的特性这件事？- 黄玄的回答 [https://www.zhihu.com/question/328958700/answer/714287394](https://www.zhihu.com/question/328958700/answer/714287394)
3.  [^](#ref_3_0)前端是否适合使用面向对象的方式编程？- 黄玄的回答 [https://www.zhihu.com/question/329005869/answer/739525268](https://www.zhihu.com/question/329005869/answer/739525268)
4.  [^](#ref_4_0)React Hooks 的引入会对之后的 React 项目开发产生什么影响？- 黄玄的回答 [https://www.zhihu.com/question/302916879/answer/536846510](https://www.zhihu.com/question/302916879/answer/536846510)
5.  [^](#ref_5_0)React 上下文的组合是通过调用顺序在运行时里维护一个链表而非基于参数化多态的层叠（比如 Monad Transformer）来表达，可以看到都是线性的。
6.  [^](#ref_6_0)State-passing Style Hooks [https://mobile.twitter.com/acdlite/status/971598256454098944](https://mobile.twitter.com/acdlite/status/971598256454098944)
7.  [^](#ref_7_0)[https://github.com/reactjs/react-basic](https://github.com/reactjs/react-basic)
8.  [^](#ref_8_0)可以把上下文或者 Hooks 的调用视为一次 stack unwinding + resume continuation。同样，考虑 row polymorphism 也是线性的。
9.  [^](#ref_9_0)Algebraic Effects for the Rest of Us [https://overreacted.io/algebraic-effects-for-the-rest-of-us/](https://overreacted.io/algebraic-effects-for-the-rest-of-us/)
10. [^](#ref_10_0)通俗易懂的代数效应 [https://overreacted.io/zh-hans/algebraic-effects-for-the-rest-of-us/](https://overreacted.io/zh-hans/algebraic-effects-for-the-rest-of-us/)
11. [^](#ref_11_0)Why React [https://gist.github.com/sebmarkbage/a5ef436427437a98408672108df01919](https://gist.github.com/sebmarkbage/a5ef436427437a98408672108df01919)
12. [^](#ref_12_0)[https://swiftwithmajid.com/2019/06/12/understanding-property-wrappers-in-swiftui/](https://swiftwithmajid.com/2019/06/12/understanding-property-wrappers-in-swiftui/)
13. [^](#ref_13_0)[https://developer.android.com/jetpack/compose/state#remember](https://developer.android.com/jetpack/compose/state#remember)

>[Evanyou](https://www.zhihu.com/people/evanyou) : 其实所谓的 “不认同” 并不是说这么设计是错误的，而是一个在基于面对的业务需求前提下，concurrency 带来的收益能否平衡掉使用时的心智负担的问题。这跟有人觉得 angular 好有人觉得不好一个道理。对于 FB 来说 concurrency 带来的一点点 metrics 提升都是很大 scale 的收益，那点心智负担对于 FB 水准的工程师来说也完全可以承受，但是（在我看来）对于占绝大多数的普通应用和前端来说，这都不是一个好的 trade-off。所以这里又涉及到另一个问题：那就是 React 毕竟是为 FB 服务的，在一个 trade-off 更有利 FB 而不是广大普通前端的情况下，React 一定会选择对 FB 有利的那一边。

> [贺师俊](https://www.zhihu.com/people/he-shi-jun): 嗯。其实小右、叔叔、题叶和我虽然都吐槽hooks，但每个人的出发点和侧重也是有区别的。比方说我的重点其实并不是吐槽hooks，而是吐槽react选的fp路线和js是难以适配的，说严重一点是框架和语言处于互相挖坑的情况。vue和js的配合虽然也不是说完全完美（比如Proxy本身的设计并不是为了reactive，因此存在潜在的小问题——倒霉的private fields放大了这个问题），但是总体上比react要好很多。也就是说按照我的看法，在互相挖坑这件事情上，react和js互相的责任是8:2，而vue和js互相的责任是2:8。

>  我记得贺老和   [@工业聚](https://www.zhihu.com/people/6751e943236c0381facaf51cf6fa1f43) 
 讨论过类似问题，工大是觉得自造语言是 hate js 路线， 上 JS 是 love js 路线。贺老是觉得 JS 不够 FP 所以 React 应该选自造语言这条路线，选 JS 是为了拉拢开发者。其实再往下看的话，这里涉及到一个「不同人对 JavaScript 语言认可的子集以及发展方向期待不同的问题」。其实 JS 由于出身 Lisp 其 core 比所有 C 风格的主流语言都对 FP 友好。从表达力来看，JS 其实不需要额外的 FP 支持，只是语法偏 C（现在补上来一些了），反而其 OO 部分从出身开始表达力就是瘸腿需要从其 FP 部分借（现在语义也补上来了）。所以 JS 是真的挺「两条腿走路」得，开发者的多样性也拉得非常开。像 Jordan 对 JS 的期待就是「发展成一个 untyped OCaml」，我觉得 Flow 比 TS 好的部分也是其「FP taste」，应该是代表了一部分人的声音得。
 
> 大家觉得在 js 里搞 fp 是叛逆，其实搞 OO 更叛逆。  
对 js 来说，失去 OO，失去的只是枷锁；失去 fp，失去一切。closure, callback, higher order function，function as data 等 first class 的 function 带来的特性，是支撑前端开发日常工作的真正核心要素。prototype, class 等，是在这个基础上，复用某个对象里的函数，以及提供模板化的语法，属于锦上添花。  
the good parts of js 我觉得是 js function 的两重特性：  
1. 当不改变闭包里的变量的值，保持 immutable 时，构成 pure function，作为 constructor 去通过组合构造出复杂的数据结构；  
2. 当允许改变闭包里的值，支持 mutable 和 impure 时，构成对 IO 的封装，作为 destructor 抽象了从 IO World 里获取一个值的操作，通过 codata 形式的函数组合，构建出复杂的操作结构。 react 是利用第一种函数性质，component 是 view constructor。rxjs 是利用第二种函数性质，observable 是对外部世界的观察和求值，operators 用以组合后续行为。vue 也是第二种函数性质，setup 方法返回的 reactive state 的本质是 state 背后的 watcher 函数对闭包里的变量的保留和处理。  
vue 和 rxjs 都是第二重属性，它们的差别在于元编程的含量。rxjs 是 raw 的函数组合，包含显性层次。vue 通过 proxy，将 watcher 在参数传递中隐藏了起来。  
当然，其实所有框架都或多或少要同时使用 js 函数的二重特性。如 react hooks 是运用函数的第二重特性为第一重特性服务，useState 依赖的闭包变量 currentDispatcher 是 mutable的。但它被框架协调为不影响 function component 的 input -> ouput，满足了外延意义上的 pure。（划重点，@题叶 对此表示不认可）  
vue 内部的 functional ui 也有 pure constrcutor 的成分。  
其他 js 语言特性承担的表达能力倒没那么重要了。  
