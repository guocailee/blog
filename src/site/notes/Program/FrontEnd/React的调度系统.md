---
{"dg-publish":true,"permalink":"/Program/FrontEnd/React的调度系统/","noteIcon":"","created":"2024-05-22T16:17:54.148+08:00"}
---

## 1. 引言

这次介绍的文章是 [scheduling-in-react](https://philippspiess.com/scheduling-in-react/)，简单来说就是 React 的调度系统，为了得到更顺滑的用户体验。

毕竟前端做到最后，都是体验优化，前端带给用户的价值核心就在于此。

## 2. 概述

文章从 Dan 在 [JSConf](https://reactjs.org/blog/2018/03/01/sneak-peek-beyond-react-16.html) 提到的 Demo 说起：

![](/img/user/z-attchements/media/TB1q4DhQIfpK1RjSZFOXXa6nFXa-663-405.gif)

这是一个测试性能的 Demo，随着输入框字符的增加，下方图表展示的数据量会急速提升。在 Synchronous 与 Debounced 模式下的效果都不尽如人意，只有 Concurrent 模式下看起来是顺畅的。

那么为什么普通的 Demo 会很卡呢？

这就涉及到浏览器 Event Loop 规则了。

JS 是单线程的，浏览器同一时间只能做一件事情，而肉眼能识别的刷新频率在 60FPS 左右，这意味着我们需要在 16ms 之内完成 Demo 中的三件事：响应用户输入，做动画，Dom 渲染。

然而目前几乎所有框架都使用同步渲染模式，这意味着如果一个渲染函数执行时间超过了 16ms，则不可避免的发生卡顿。

总结一下有两个主要问题：

1. 长时间运行的任务造成页面卡顿，我们需要保证所有任务能在几毫秒内完成，这样才能保证页面的流畅。
2. 不同任务优先级不同，比如响应用户输入的任务优先级就高于动画。这个很好理解。

### React 调度机制

为了解决这个问题，React16 通过 Concurrent（并行渲染） 与 Scheduler（调度）两个角度解决问题：

- **Concurrent：** 将同步的渲染变成可拆解为多步的异步渲染，这样可以将超过 16ms 的渲染代码分几次执行。
- **Scheduler：** 调度系统，支持不同渲染优先级，对 Concurrent 进行调度。当然，调度系统对低优先级任务会不断提高优先级，所以不会出现低优先级任务总得不到执行的情况。

为了保证不产生阻塞的感觉，调度系统会将所有待执行的回调函数存在一份清单中，在每次浏览器渲染时间分片间尽可能的执行，并将没有执行完的内容 Hold 住留到下个分片处理。

Concurrent 的正式 API 会在 2019 Q2 发布，现在可以通过 `<React.unstable_ConcurrentMode>` API 方式调用：

```jsx
ReactDOM.render(
  <React.unstable_ConcurrentMode>
    <App />
  </React.unstable_ConcurrentMode>,
  rootElement
);
```

只申明这个是不够的，因为我们还没有申明各函数执行的优先级。我们可以通过 `npm i scheduler` 包来申明函数的优先级：

```jsx
import { unstable_next } from "scheduler";

function SearchBox(props) {
  const [inputValue, setInputValue] = React.useState();

  function handleChange(event) {
    const value = event.target.value;

    setInputValue(value);
    unstable_next(function() {
      props.onChange(value);
      sendAnalyticsNotification(value);
    });
  }

  return <input type="text" value={inputValue} onChange={handleChange} />;
}
```

在 `unstable_next()` 作用域下的代码优先级是 `Normal`，那么产生的效果是：

1. 如果 `props.onChange(value)` 可以在 16ms 内执行完，则与不使用 `unstable_next` 没有区别。
2. 如果 `props.onChange(value)` 的执行时间过长，可能这个函数会在下次几次的 Render 中陆续执行，不会阻塞后续的高优先级任务。

### 调度带来的限制

调度系统也存在两个问题。

1. 调度系统只能有一个，如果同时存在两个调度系统，就无法保证调度正确性。
2. 调度系统能力有限，只能在浏览器提供的能力范围内进行调度，而无法影响比如 Html 的渲染、回收周期。

为了解决这个问题，Chrome 正在与 React、Polymer、Ember、Google Maps、Web Standars Community 共同创建一个 [浏览器调度规范](https://github.com/WICG/main-thread-scheduling)，提供浏览器级别 API，可以让调度控制更底层的渲染时机，也保证调度器的唯一性。

## 3. 精读

关于 React 调度系统的剖析，可以读 [[Program/FrontEnd/深入剖析 React Concurrent\|深入剖析 React Concurrent]] 这篇文章，感谢我们团队的 淡苍 提供。

简单来说，一次 Render 一般涉及到许多子节点，而 Fiber 架构在 Render 阶段可以暂停，一个一个节点的执行，从而实现了调度的能力。

### React 调度能力的限制

> 这意味着，如果你的 React 应用目前是流畅的，开启 Concurrent 并不会对你的应用带来性能体验上的提升，如果你的 React 应用目前是卡顿的，或者在某些场景下是卡顿的，那么 Concurrent 或许可以挽救你一下，带来一些改变。

正如《深入剖析 React Concurrent》一文提到的，如果你的应用没有性能问题，就不要指望 React 调度能力有所帮助了。

这也是在说，如果一段代码逻辑不存在性能问题，就不需要使用 Concurrent 优化，因为这种优化是无效的。我们需要能分辨哪些逻辑需要优化，哪些逻辑不要。

### 从现在开始尝试 Function Component

为了配合 React Schedule 的实现，学会使用 Function Component 模式编写组件是很重要的，因为：

1. Class Component 的生命周期概念阻碍了 React 调度系统对任务的拆分。
2. 调度系统可能对 `componentWillMount` 重复调用，使得 Class Component 模式下很容易写出错误的代码。
3. Function Component 遵循了更严格的副作用分离，这使得 Concurrent 执行过程不会引发意外效果。

### React.lazy

与 Concurrent 一起发布的，还有 React 组件动态 import 与载入方案。正常的组件载入是这样的：

```jsx
import OtherComponent from "./OtherComponent";

function MyComponent() {
  return (
    <div>
      <OtherComponent />
    </div>
  );
}
```

但如果使用了 `import()` 动态载入，可以使用 `React.lazy` 让动态引入的组件像普通组件一样被使用：

```jsx
const OtherComponent = React.lazy(() => import("./OtherComponent"));

function MyComponent() {
  return (
    <div>
      <OtherComponent />
    </div>
  );
}
```

如果要加入 Loading，就可以配合 `Suspense` 一起使用：

```jsx
import React, { lazy, Suspense } from "react";
const OtherComponent = lazy(() => import("./OtherComponent"));

function MyComponent() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <OtherComponent />
    </Suspense>
  );
}
```

和 Concurrent 类似，React.lazy 方案也是一种对性能有益的组件加载方案。

### 调度分类

调度分 4 个等级：

- **Immediate**：立即执行，最高优先级。
- **render-blocking**：会阻塞渲染的优先级，优先级类似 `requestAnimationFrame`。如果这种优先级任务不能被执行，就可能导致 UI 渲染被 block。
- **default**：默认优先级，普通的优先级。优先级可以理解为 `setTimeout(0)` 的优先级。
- **idle**：比如通知等任务，用户看不到或者不在意的。

目前建议的 API 类似如下：

```js
function mytask() {
  ...
}

myQueue = TaskQueue.default("render-blocking")
```

先创建一个执行队列，并设置队列的优先级。

```js
taskId = myQueue.postTask(myTask, <list of args>);
```

再提交队列，拿到当前队列的执行 id，通过这个 id 可以判断队列何时执行完毕。

```js
myQueue.cancelTask(taskId);
```

必要的时候可以取消某个函数的执行。

## 4. 总结

随着 Hooks 的发布，即将到来的 Concurrent 与 Suspense 你是否准备好了呢？

笔者希望大家一起思考，这三种 API 会给前端开发带来什么样的改变？欢迎留言！

> 讨论地址是：[精读《Scheduling in React》 · Issue #146 · dt-fe/weekly](https://github.com/dt-fe/weekly/issues/146)

**如果你想参与讨论，请 [点击这里](https://github.com/dt-fe/weekly)，每周都有新的主题，周末或周一发布。前端精读 - 帮你筛选靠谱的内容。**

> 关注 **前端精读微信公众号**

<img width=200 src="https://img.alicdn.com/tfs/TB165W0MCzqK1RjSZFLXXcn2XXa-258-258.jpg">


> 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享 3.0 许可证](https://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）