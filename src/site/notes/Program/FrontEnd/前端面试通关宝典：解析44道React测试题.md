---
{"dg-publish":true,"permalink":"/Program/FrontEnd/前端面试通关宝典：解析44道React测试题/","noteIcon":"","created":"2025-03-06T21:28:25.975+08:00"}
---

#React 

## 1\. 你了解哪些 React hooks？

*   useState: 用于管理函数组件中的状态。
    
*   useEffect: 用于在函数组件中执行 side effects，例如获取数据或订阅事件。
    
*   useContext: 用于访问函数组件当中 React 上下文的值。
    
*   useRef: 用于为跨渲染持续存在的元素或值创建可变引用。
    
*   useCallback: 用于记忆函数，以防止不必要的重新渲染。
    
*   useMemo: 用于记忆值，即将成本高昂的计算结果缓存起来以提高性能。
    
*   useReducer: 负责使用 reducer 函数管理状态，原理类似于 Redux。
    
*   useLayoutEffect: 与 useEffect 类似，但效果会在所有 DOM 更改之后再同步运行。
    

这些 hooks 提供强大的工具，可用于管理状态、处理 side effects 和重用 React 函数组件当中的逻辑。

了解更多：https://react.dev/reference/react

## 2\. 虚拟 DOM 是什么?

虚拟 DOM 是 React 中提出的概念，用于为实际 DOM（文档对象模型）创建一个轻量化虚拟表示，并将其存储在内存当中。这是一种用于优化 Web 应用程序性能的编程技术。

当 React 组件的数据或状态发生变更时，虚拟 DOM 也会随之更新，而非直接操作实际 DOM。此后，虚拟 DOM 计算组件先前状态与更新状态之间差异的过程，被称为“diffing”。

一旦发现存在差异，React 将仅更新实际 DOM 当中的必要部分，借此高效反映变更内容。这种方式最大限度减少了实际 DOM 上的操作数量，进而提高了应用程序的整体性能。

通过使用虚拟 DOM，React 在提供动态及交互式用户界面创建方法的同时，也保证应用程序始终拥有最佳效率和渲染速度。

## 3\. 如何渲染一个元素数组？

要渲染一个元素数组，可以使用 map() 方法迭代该数组，并返回一个新的 React 元素数组。

```javascript
const languages = [
  "JavaScript",
  "TypeScript",
  "Python",
];

function App() {
  return (
    <div>
      <ul>{languages.map((language) => <li>{language}</li>)}</ul>
    </div>
  );
}
```

了解更多：https://react.dev/learn/rendering-lists

## 4\. 受控组件与非受控组件之间有何区别？

受控组件与非受控组件之间的最大区别，在于如何管理和更新自身状态。

受控组件的状态由 React 负责控制，该组件接受其当前值并通过 props 进行更新。当值发生改变时，它会触发回调函数，也就是说该组件不会存储自己的内部状态。相反，由父组件管理该值并将其传递给受控组件。

```javascript
import { useState } from 'react'; 

function App() { 
  const [value, setValue] = useState(''); 

  return ( 
    <div> 
      <h3>Controlled Component</h3> 
      <input name="name" value={name} onChange={(e) => setValue(e.target.value)} />
      <button onClick={() => console.log(value)}>Get Value</button> 
    </div> 
  ); 
} 
```

另一方面，非受控组件则使用 refs 或其他方法在内部管理自身状态。这类组件独立存储并更新其状态，不依赖于 props 或回调。父组件对非受控组件的状态控制能力较弱。

```javascript
import { useRef } from 'react'; 

function App() { 
  const inputRef = useRef(null); 

  return ( 
    <div className="App"> 
      <h3>Uncontrolled Component</h3> 
      <input type="text" name="name" ref={inputRef} /> 
      <button onClick={() => console.log(inputRef.current.value)}>Get Value</button> 
    </div> 
  ); 
} 
```

了解更多：https://react.dev/learn/sharing-state-between-components#controlled-and-uncontrolled-components

## 5\. 基于类的 React 组件， 与函数式 React 组件之间有何区别？

基于类的组件和函数组件之间的主要区别，在于二者的定义方式和所用语法不同。

基于类的组件被定义为 ES6 类，属于 React.Component 类的扩展。它们使用 render 方法返回定义组件输出的 JSX（JavaScript XML）。类组件可以通过 this.state 和 this.setState() 访问其生命周期方法和状态管理。

```javascript
class App extends React.Component {
  state = {
    value: 0,
  };

  handleAgeChange = () => {
    this.setState({
      value: this.state.value + 1 
    });
  };

  render() {
    return (
      <>
        <p>Value is {this.state.value}</p>
        <button onClick={this.handleAgeChange}>
        Increment value
        </button>
      </>
    );
  }
}
```

另一方面，函数组件被定义为简单的 JavaScript 函数。它们接受 props 作为参数并直接返回 JSX。函数组件无权访问生命周期方法或者状态。但随着 React 16.8 中 React hooks 机制的出现，函数组件现在也可以管理状态并使用其他功能，例如上下文和效果。

```javascript
import { useState } from 'react';

const App = () => {
  const [value, setValue] = useState(0);

  const handleAgeChange = () => {
    setValue(value + 1);
  };

  return (
      <>
        <p>Value is {value}</p>
        <button onClick={handleAgeChange}>
        Increment value
        </button>
      </>
  );
}
```

一般来讲，函数组件往往更简单、易于阅读和测试。所以除非确实需要类组件，否则建议大家尽量使用函数组件。

## 6\. 组件的生命周期方法有哪些？

生命周期方法，属于一种钩入组件生命周期各个阶段的方法，允许开发者在特定时间执行特定的代码。

以下是几种主要生命周期方法：

1.  constructor: 这也是创建组件时调用的第一个方法，用于初始化状态并绑定事件处理程序。在函数组件中，我们可以使用 useState hook 来实现类似的效果。
    
2.  render: 此方法负责渲染 JSX 标记，并返回要在屏幕上显示的内容。
    
3.  componentDidMount: 此方法将在组件于 DOM 中渲染后被立即调用，通常用于初始化任务，例如 API 调用或设置事件侦听器。
    
4.  componentDidUpdate: 此方法会在组件的 props 或 state 发生变更时被调用，允许开发者执行 side effects、根据变更更新组件或者触发其他 API 调用。
    
5.  componentWillUnmount: 此方法会在组件从 DOM 中删除之前被调用，用于清理conponentDidMount 中设置的一切资源，例如删除事件侦听器或取消计时器。
    

某些生命周期方法（例如 componentWillMount、componentWillReceiveProps 和 componentWillUpdate）现已被弃用，或者被其他方法或 hooks 所替代。

至于“this”方法，是指类组件的当前实例。我们可以用它访问组件内的属性和方法。在函数组件中不需要使用“this”，因为函数不会绑定至特定实例。

## 7\. 使用 useState 时应该注意什么？

useState 会返回一个状态值和一条更新该值的函数。

```cs
const [value, setValue] = useState('Some state');
```

在初始渲染期间，返回的状态与传递来的首个参数值相匹配。setState 函数用于更新该状态，它采用新的状态值作为参数并对组件的重新渲染操作进行排队。setState 函数还可以接受回调函数作为参数，该函数会将之前的状态值作为参数。  

了解更多：https://react.dev/reference/react/useState

## 8\. 使用 useEffect 时应该注意什么?

useEffect hook 允许我们在函数组件中执行 side effects。

在 React 的渲染阶段，函数组件的主体之内不得出现突变、订阅、计时器、日志记录及其他 side effects，这些可能导致用户界面中出现难以理解的错误和一致性冲突。

相反，这里建议使用 useEffect。传递给 useEffect 的函数将在渲染被提交至屏幕后才开始执行；而如果您传递一组依赖项作为第二参数，则每当有依赖项发生变更时，都会调用该函数。

```javascript
useEffect(() => {
  console.log('Logging something');
}, [])
```

了解更多：https://react.dev/reference/react/useEffect

## 9\. 如何跟踪函数组件是否被卸载?

一般来说，useEffect 所创建的资源需要在组件离开屏幕前进行清理或重置，例如订阅或计时器标记。

为此，传递给 useEffect 的函数可以返回一个清理函数。该清理函数将在组件被从用户界面中删除之前运行，防止发生内存泄漏。此外，如果组件经过多次渲染（属于常见情况），则在执行下一效果之前会先清除上一效果。

```javascript
useEffect(() => {
  function handleChange(value) {
    setValue(value);
  }
  SomeAPI.doFunction(id, handleChange);

  return function cleanup() {
    SomeAPI.undoFunction(id, handleChange);
  };
})
```

## 10\. React 中的 props 是什么？

Props 是指从父组件传递给当前组件的数据。Props 有只读限制，无法更改。

```kotlin
// 父组件
const Parent = () => {
  const data = "Hello, World!";

  return (
    <div>
      <Child data={data} />
    </div>
  );
};

// 子组件
const Child = ({ data }) => {
  return <div>{data}</div>;
};
```

了解更多：https://react.dev/learn/passing-props-to-a-component

## 11\. 什么是状态管理器？你曾经使用过

哪些状态管理器，或者了解过哪些状态管理器？

状态管理器是帮助管理应用程序状态的工具或库，负责提供一个集中的存储或容器，用以容纳并管理可由应用程序中各个组件访问并更新的数据。

状态管理器可以解决以下几个问题。首先，最好将数据同与之相关的逻辑 / 组件彼此分离。第二，在使用本地状态并在组件之间进行传递时，由于组件中可能存在深层嵌套，因此代码往往会比较复杂。通过建立全局存储，我们可以访问并修改来自任意组件的数据。

除了 React Context 以外，常见的状态管理库还有 Redux 和 MobX。

了解更多：https://mobx.js.org/README.html

了解更多：https://redux-toolkit.js.org/

## 12\. 在哪些情况下可以使用本地状态，

什么时候应该使用全局状态？

如果仅需要在单一组件内使用，而且无需传递给其他组件，则建议使用本地状态。本地状态还适用于组件只需在列表中表示单一项目的情况。但如果组件拆分涉及到嵌套组件，而且数据需要沿层次结构进行传递，则最好使用全局状态。

## 13\. Redux 中的 reducer 是什么，

它会用到哪些参数？

Reducer 属于纯函数，并将状态和操作作为参数。在 reducer 内部，我们会跟踪接收到的 action 类型，再根据它修改状态并返回一个新的状态对象。

```javascript
export default function appReducer(state = initialState, action) {
  // Reducer通常会查看action类型字段来决定如何执行
  switch (action.type) {
    // 根据action的具体类型选择执行方式
    default:
      // 如果此reducer无法识别action类型
      // 或者此action不重要，则直接返回现有状态
  }
}
```

了解更多：https://redux.js.org/tutorials/fundamentals/part-3-state-actions-reducers

## 14\. Action 是什么，

我们如何改变 Redux 中的状态?

Action 属于简单的 JavaScript 对象，其组成为字段加类型。

```css
{
  type: "SOME_TYPE"
}
```

我们也可以为其添加数据作为负载payload。要改变状态，则须调用我们传递给action的dispatch调度函数。  

```css
{
  type: "SOME_TYPE",
  payload: "Any payload",
}
```

了解更多：https://redux.js.org/tutorials/fundamentals/part-3-state-actions-reducers

## 15\. Redux 实现的是哪种模式?

Redux 实现的是 Flux 模式，即应用程序的可预测状态管理模式。它通过引入单向数据流与应用程序状态的集中存储机制，帮助管理应用程序状态。

了解更多：https://www.newline.co/fullstack-react/30-days-of-react/day-18/#:~:text=Flux%20is%20a%20pattern%20for,default%20method%20for%20handling%20data.

## 16\. Mobx 实现的是哪种模式?

Mobx 实现的是 Observer 模式，也被称为`发布-订阅`模式。

了解更多：https://www.patterns.dev/posts/observer-pattern

## 17\. 在使用 Mobx 时应该注意什么?

Mobx 提供 observable 和 computed 等修饰器来定义可观察状态与反应函数。用 action 修饰的操作用于修改状态，确保跟踪所有变更。Mobx 还提供自动依赖项跟踪、不同反应类型、对反应性的细粒度控制，以及通过 mobx-react 包与 React 的无缝集成。总的来说，Mobx 能够以可观察状态的变化为基础，自动执行更新过程以简化状态管理。

## 18\. 如何访问 Mobx 状态下的变量?

我们可以使用 observalbe 装饰器将该变量定义为 observable，借此实现对状态下变量的访问。例如：

```typescript
import { observable, computed } from 'mobx';

class MyStore {
  @observable myVariable = 'Hello Mobx';

  @computed get capitalizedVariable() {
    return this.myVariable.toUpperCase();
  }
}

const store = new MyStore();
console.log(store.capitalizedVariable); // Output: HELLO MOBX

store.myVariable = 'Hi Mobx';
console.log(store.capitalizedVariable); // Output: HI MOBX
```

在本示例中，使用 observable 装饰器将 myVariable 定义为 observable。之后，我们可以使用 store.myVariable 访问该变量。如此一来，对 myVariable 所做的任何更改都会自动触发相关组件或反应的更新。

了解更多：https://mobx.js.org/actions.html

## 19\. Redux 与 Mobx 之间有什么区别？

Redux 是一种更简单也更严格的状态管理库，要求遵循单向数据流和不变性原则。它需要用到更多的样板代码和显式更新，但与 React 的集成效果非常出色。

另一方面，Mobx 则提供更加灵活、直观的 API，而且样板代码也更少。它允许开发者直接修改状态并自动跟踪变更，借此获得更好的应用程序性能。具体选择 Redux 还是 Mobx，要视实际需求和开发偏好而定。

## 20\. 什么是 JSX?

默认情况下，可以使用以下语法在 React 中创建元素：

```javascript
const someElement = React.createElement(
  'h3',
  {className: 'title__value'},
  'Some Title Value'
);
```

但我们在实际开发中，往往更习惯用以下形式：

```javascript
const someElement = (
  <h3 className='title__value'>Some Title Value</h3>
);
```

这就是所谓 JSX，是一种用于简化代码理解和开发表达的语言扩展。

了解更多：https://react.dev/learn/writing-markup-with-jsx#jsx-putting-markup-into-javascript

## 21\. React 中的 props 钻取是什么?

Props 钻取，是指在某些中间组件并不直接使用 props，仍通过多层嵌套组件传递 props 的过程。这可能导致代码结构繁琐且复杂。

```kotlin
// 父组件
const Parent = () => {
  const data = "Hello, World!";

  return (
    <div>
      <ChildA data={data} />
    </div>
  );
};

// 中间ChildA组件
const ChildA = ({ data }) => {
  return (
    <div>
      <ChildB data={data} />
    </div>
  );
};

// 叶ChildB组件
const ChildB = ({ data }) => {
  return <div>{data}</div>;
};
```

在此示例中，data prop 从父组件传递到了 ChildA，之后又从 ChildA 传递至 ChildB，其中 ChildA 并不直接使用该 prop。当存在多个层次的嵌套，或者当组件树中较靠下的组件需要访问数据时，可能会使用这种方式，但这会导致代码更难以维护和理解。

我们可以通过其他模式（例如 Redux 或 MobX 等上下文或状态管理库）来缓解 props 钻取。这些方法既允许组件访问数据，又不需要通过每个中间组件传递 props。

## 22\. 如何根据条件渲染元素?

我们可以使用各种条件运算符，包括三元运算符。

```javascript
return (
  <div>
    {isVisible && <span>I'm visible!</span>}
  </div>
);
return (
  <div>
    {isOnline ? <span>I'm online!</span> : <span>I'm offline</span>}
  </div>
);
if (isOnline) {
  element = <span>I'm online!</span>;
} else {
  element = <span>I'm offline</span>;
}

return (
  <div>
    {element}
  </div>
);
```



## 23\. userMemo 的用途是什么？它是如何起效的?

useMemo 用于缓存和记忆计算结果。

传递创建函数和依赖项数组。只有当任意依赖项的值发生变更时，useMemo 才会重新计算记忆值。这种优化方式能避免每次渲染都引发昂贵的计算过程。

使用首个参数，该函数将接受执行计算的回调；使用第二个依赖项数组，则仅当至少一个依赖项发生更改时，该函数才会重新执行计算。

```javascript
const memoValue = useMemo(() => computeFunc(paramA, paramB), [paramA, paramB]);
```

了解更多：

https://react.dev/reference/react/useMemo

## 24\. useCallback 的用途是什么，它是如何起效的?

useCallback hook 将返回回调的一个记忆版本，且仅当依赖项之一的值发生更改时该版本才会随之更改。其主要作用，就是在将回调传递给依赖于链接等效性的子组件时，避免触发不必要的渲染。

```javascript
const callbackValue = useCallback(() => computeFunc(paramA, paramB), [paramA, paramB]);
```

了解更多：

https://react.dev/reference/react/useCallback

## 25\. useMemo 和 useCallback 之间有什么区别?

1.  useMemo 用于记忆计算结果，而 useCallback 用于记忆函数本体。
    
2.  useMemo 缓存的是计算值，如果依赖项未更改，则在后续渲染时直接返回该值。
    
3.  useCallback 缓存的是函数本体，只要依赖项未发生更改，则直接返回同一实例。
    

## 26\. React Context 是什么?

React Context 是一项功能，提供一种通过组件树传递数据的方法，避免在每个层次上手动传递 props。它允许我们创建一个全局状态，树中的任何组件无论位置如何、均可以访问该状态。当我们需要在未通过 props 直接连接的多个组件之间共享数据时，就可以使用 Context。

React Context API 包含以下三个主要部分：

1.  createContext: 此函数用于创建一个新的 context 上下文对象。
    
2.  Context.Provider: 此组件用于向 context 提供值，其中打包有需要访问该值的组件。
    
3.  Context.Consumer 或 useContext hook: 此组件或 hook 负责使用 context 中的值。它可以在上下文提供方内的任意组件中使用。
    

通过使用 React Context，我们可以避免 prop 钻取（即在多个层次的组件间传递项目）并轻松管理更高级别的状态，保证代码更具组织性的执行效率。

了解更多：

https://react.dev/learn/passing-data-deeply-with-context

## 27\. useContext 的用途是什么，它是如何起效的?

在典型的 React 应用程序当中，数据使用 props 以自上而下（从父组件到子组件）的方式传递。但是，这样的方式对于某些特定类型的 props（例如选定的语言、UI 主题）来说可能过于繁琐，因为需要将其传递给应用程序中的多个组件。Context 上下文提供一种在组件之间共享此类数据的方法，而无需通过树结构中的各个层次显式传递 props。

当 context 的值改变时，调用 useContext 的组件也将随之进行重新渲染。如果重新渲染组件的成本很高，这里可以使用记忆机制进行优化。

```javascript
const App = () => {
  const theme = useContext(ThemeContext);
 
  return (
    <div style={{ color: theme.palette.primary.main }}>
      Some div
    </div>
  );
}
```

了解更多：

https://react.dev/reference/react/useContext

## 28\. useRef 的用途是什么，它是如何起效的?

useRef 返回一个可修改的 ref 对象，即一个属性。其中的当前值由传递的参数进行初始化。返回的对象将在组件的整个生命周期之内持续存在，且不会因渲染而发生改变。

其常见用法是以命令的形式访问后代，例如使用 ref，我们可以显式引用 DOM 元素。

```javascript
const App = () => {
  const inputRef = useRef(null);
 
  const buttonClick = () => {
    inputRef.current.focus();
  }
 
  return (
    <>
      <input ref={inputRef} type="text" />
      <button onClick={buttonClick}>Focus on input tag</button>
    </>
  )
}
```

了解更多：

https://react.dev/reference/react/useRef

## 29\. React.memo() 是什么?

React.memo() 是一种高阶组件。如果您的组件始终使用不变的 props 渲染相同的内容，则可以将其打包在 React.memo() 调用当中，通过记忆固定结果的方式提高性能。也就是说，React 将直接使用上次渲染的结果，避免重新执行渲染。React.memo() 只影响 props 的更改。如果一个函数组件被打包进 React.memo 并使用 useState、useReducer 或 useContext，那么当状态或上下文发生变化时，它将被重新渲染。

```javascript
import { memo } from 'react';
 
const MemoComponent = memo(MemoComponent = (props) => {
  
});
```

了解更多：

https://react.dev/reference/react/memo

## 30\. React Fragment 是什么?

在 React 当中，经常需要从组件返回多个元素。Fragments 允许我们建立子元素列表，而无需在 DOM 当中创建不必要的节点。

```xml
<>
  <OneChild />
  <AnotherChild />
</>
// or
<React.Fragment>
  <OneChild />
  <AnotherChild />
</React.Fragment>
```

了解更多：

https://react.dev/reference/react/Fragment

## 31\. React Reconciliation 是什么?

Reconciliation 协调是一种 React 算法，用于区分一个元素树与另一个元素树，借此确定其中需要替换的部分。

Reconciliation 也就是我们之前提到的虚拟 DOM 背后的算法。其基本定义是：在渲染一个 React 应用程序时，描述该应用程序的元素树是在保留的内存当中生成。之后，该树将被包含在渲染环境当中——例如浏览器应用程序，并被翻译成一组 DOM 操作。当应用程序状态更新时，则会生成一个新树。将新树与之前的树进行比较，即可准确计算并得出对更新后的应用程序进行重绘所需要的操作。

了解更多：

https://react.dev/learn/preserving-and-resetting-state

## 32\. 在使用 map() 时，为什么需要列表中的键?

这些键将帮助 React 确定哪些元素已经被更改、添加或者移除。只有指定了这些键，React 才能随时间推移匹配数组元素。在选择键时，最好使用那些能够明确区分列表项与其相邻的字符串。大多数情况下，我们可以使用数据中的 ID 作为键。

```javascript
const languages = [
  {
    id: 1,
    lang: "JavaScript",
  },
  {
    id: 2,
    lang: "TypeScript",
  },
  {
    id: 3,
    lang: "Python",
  },
];
 
const App = () => {
  return (
    <div>
      <ul>{languages.map((language) => (
        <li key={`${language.id}_${language.lang}`}>{language.lang}</li>
      ))}
      </ul>
    </div>
  );
}
```

了解更多：

https://react.dev/learn/rendering-lists#keeping-list-items-in-order-with-key

## 33\. 如何在 Redux Thunk 中处理异步操作?

要使用 Redux Thunk，我们需要将其作为中间件导入。操作创建者不仅需要返回一个对象，还应返回一个以 dispatch 调度为参数的函数。

```javascript
export const addUser = ({ firstName, lastName }) => {
  return dispatch => {
    dispatch(addUserStart());
  }
 
  axios.post('https://jsonplaceholder.typicode.com/users', {
    firstName,
    lastName,
    completed: false
  })
  .then(res => {
    dispatch(addUserSuccess(res.data));
  })
  .catch(error => {
    dispatch(addUserError(error.message));
  })
}
```

了解更多：

https://redux.js.org/usage/writing-logic-thunks

## 34\. 如何跟踪函数组件中对象字段的变化?

要实现变化跟踪，我们需要使用 useEffect hook，并将对象的字段作为依赖项数组进行传递。

```javascript
useEffect(() => {
  console.log('Changed!')
}, [obj.someField])
```

## 35\. 如何访问 DOM 元素?

这里可以使用由 React.createRef() 或 useRef() hook 创建的 refs，并将该 ref 属性附加至 React 元素。通过访问所创建的 ref 引用，即可通过 ref.current 来访问 DOM 元素。

```javascript
const App = () => {
  const myRef = useRef(null);
 
  const handleClick = () => {
    console.log(myRef.current); 
  };
 
  return (
    <div>
      <input type="text" ref={myRef} />
      <button onClick={handleClick}>Click Me</button>
    </div>
  );
}
 
export default App;
```

36\. 什么是自定义 hook?

自定义 hook，是用于在不同组件之间重用逻辑的函数。这是一种打包可重用逻辑的方法，以便在多个组件之间轻松实现共享和重用。自定义 hook 通常是以单词“use”开头的函数，并可根据需求调用其他 hooks。

了解更多：

https://react.dev/learn/reusing-logic-with-custom-hooks

## 37\. 什么是公共 API?

以索引文件为例，公共 API 通常是指向外部模块或组件公开、且可供访问的接口或函数。

以下是表示公共 API 的索引文件代码示例：

```javascript

 
export function greet(name) {
  return `Hello, ${name}!`;
}
 
export function calculateSum(a, b) {
  return a + b;
}
```

在此示例中，index.js 文件充当公共 API，其中会导出函数 greet() 和 calculateSum()。通过导入，即可从其他模块处访问这些函数。其他模块可以导入，并使用这些函数作为自身实现的组成部分：

```javascript

 
import { greet, calculateSum } from './index.js';
 
console.log(greet('John')); 
console.log(calculateSum(5, 3)); 
```

通过从索引文件导出特定函数，我们即可定义模块的公共 API，并允许其他模块使用这些函数。

## 38\. 创建自定义 hook 需要遵循哪些规则?

1.  Hook 的名称以“use”开头。
    
2.  如果必要，可使用现成的 hook。
    
3.  不要为 hook 调用设置条件。
    
4.  将可重用逻辑提取至自定义 hook 当中。
    
5.  自定义 hook 必须为纯函数。
    
6.  自定义 hook 可以返回值，也可以返回其他 hook。
    
7.  尽可能在命名时清晰描述自定义 hook 的作用。
    

了解更多：

https://react.dev/learn/reusing-logic-with-custom-hooks

## 39\. 什么是 SSR（服务器端渲染）?

服务器端渲染（SSR）是一种用于在服务器上渲染页面，再将完整渲染的页面发送至客户端进行呈现的技术。它允许服务器为网页生成完整的 HTML 标记（包括其中的动态内容），并将其作为请求的响应发送至客户端。

在传统的客户端渲染方法当中，客户端只接收最小的 HTML 页面，之后向服务器发出额外的数据和资源请求，再利用这些数据和资源在客户端渲染页面。这不仅会导致初始页面加载速度变慢，对于搜索引擎优化（SEO）也有负面影响，因为搜索引擎爬虫很难对 JavaScript 驱动的内容建立索引。

使用 SSR，服务器通过执行必要的 JavaScript 代码来生成用于最终页面呈现的 HTML。也就是说，客户端从服务器处接收到的完全渲染后的页面，从而减少了额外的资源需求。SSR 缩短了初始页面加载时间，并允许搜索引擎轻松索引内容，从而实现更好的 SEO 效果。

SSR 在各种框架和库中都很常见，例如用于 React 的 Next.js 和用于 Vue.js 的 Nuxt.js。这些框架能够帮助我们处理服务器端渲染逻辑，大大降低 SSR 的实现门槛。

## 40\. 使用 SSR 有哪些好处?

1.  缩短初始加载时间：SSR 允许服务器将完全渲染的 HTML 页面发送至客户端，从而减少客户端所需的处理量。这样可以缩短初始加载时间，让用户更快看到完整页面。
2.  SEO 友好：搜索引擎可以有效抓取并索引 SSR 页面的内容，因为完全渲染的 HTML 在初始响应中已经可用。这就提高了内容在搜索引擎中的可见性，有助于带来更好的搜索排名。
3.  可及性：SSR 使得禁用 JavaScript 或者使用辅助选项的用户也能正常访问内容。通过在服务器上生成 HTML，SSR 能够为所有用户提供可靠且易于访问的浏览体验。
4.  低带宽环境下性能更好：SSR 减少了客户端需要下载的数据量，有利于低带宽或高延迟环境下的用户体验。这一点对于使用移动网络、或者固网速度较慢的用户尤其重要。
    

尽管 SSR 具备诸多优势，但需要注意的是，与客户端渲染方法相比，这也会给服务器负载和维护带来更多复杂性。因此在实施之前，应认真考虑缓存、可扩展性和服务器端渲染性能优化等因素。

## 41\. 你了解 Next.js 中的哪些主要函数?

1\. getStaticProps: 此方法用于在构建过程中获取数据，并将页面预渲染为静态 HTML。它能确保构建时的数据可用性，且不会因后续请求而发生更改。

```javascript
export async function getStaticProps() {
  const res = await fetch('https://api.example.com/data');
  const data = await res.json();
 
  return {
    props: {
      data
    }
  };
}
```

2\. getServerSideProps: 此方法用于根据每个请求获取数据，并在服务器端预渲染页面。如果需要获取经常更改、或者仅供特定用户使用的数据，则可以使用此方法。

```javascript
export async function getServerSideProps() {
  const res = await fetch('https://api.example.com/data');
  const data = await res.json();
 
  return {
    props: {
      data
    }
  };
}
```

3\. getStaticPaths: 此方法用于在动态路由当中，指定需要在构建时预渲染的路径列表，常用于获取带有参数的动态路由数据。

```javascript
export async function getStaticPaths() {
  const res = await fetch('https://api.example.com/posts');
  const posts = await res.json();
 
  const paths = posts.map((post) => ({
    params: { id: post.id }
  }));
 
  return {
    paths,
    fallback: false
  };
}
```

了解更多:

https://nextjs.org/docs/app/building-your-application/data-fetching/fetching-caching-and-revalidating

## 42\. 什么是 linters?

Linters 是用于检查源代码中是否存在潜在错误、bug 风格不一致及可维护性问题的工具，可帮助执行编码标准并保障整个代码库的代码质量与一致性。

Linters 的基本原理，是扫描源代码并将其与一组预先定义的规则或指南进行比较。具体规则可以包含语法与格式约定、最佳实践、潜在 bug 及代码异味。当 linter 发现违规情况时，将会生成警告或错误，并高亮显示需要注意的一行或多行代码。

使用 linter 有以下几大好处：

1.  代码质量：Linters 有助于发现并防范潜在 bug、代码异味和反模式，从而提高代码质量。
    
2.  一致性：Linters 会强制执行编码约定和风格指南，确保整个代码库的格式和代码结构保持一致，特别适合多位开发者处理同一项目的情况。
    
3.  可维护性：通过尽早发现问题并推行良好的编码实践，linters 有助于提高代码的可维护性，让代码库更易于理解、修改和扩展。
    
4.  效率：Linters 能够实现代码审查流程自动化，抢在常见错误引发开发 / 生产流程实际问题前将其发现，帮助开发人员节约时间。
    

常见的 linters 工具包括面向 JavaScript 的 ESLint，用于 CSS 的 Stylelint 以及 Sass。

了解更多：

https://eslint.org/docs/latest/use/getting-started

43\. 你了解哪些 React 架构解决方案?

市面上有多种用于构建 React 项目的架构解决方案和模式，比较主流的包括：

1.  MVC（模型 - 视图 - 控制器）：MVC 是一种传统架构模式，主要将应用程序分为三大主要组成部分：模型、视图和控制器。React 可以在 View 视图层中渲染 UI，而其他库或框架则可用于 Model 模型层和 Controller 控制器层。
    
2.  Flux：Flux 是 Facebook 专门针对 React 应用程序推出的应用架构。它遵循单向数据流，即数据仅沿单一方向流动，因此应用程序的状态更改更易于理解和调试。
    
3.  原子设计：原子设计并非 React 所独有，而是一种将 UI 拆分成更小且可重用组件的设计方法。它鼓励构建小型、独立且能通过组合建立更复杂 UI 的组件。
    
4.  容器与组件模式：此模式将表示（组件）与逻辑和状态管理（容器）区分开来。其中组件专门负责渲染 UI，而容器则处理业务逻辑和状态管理。
    
5.  功能切片设计：这是一种用于组织和构建 React 应用程序的现代架构方法，强调根据功能或模块对应用程序代码库进行划分，从而解决可扩展性、可维护性和可重用性等难题。
    

## 44\. 什么是功能切片设计?

这是一种用于组织和构建 React 应用程序的现代架构方法，强调根据功能或模块对应用程序代码库进行划分，从而解决可扩展性、可维护性和可重用性等难题。

在功能切片设计当中，应用程序的每个功能或模块都被组织为一个单独的目录，其中包含全部必要组件、actions、reducers 及其他相关文件。这有助于保持代码库的模块化和隔离性，降低开发、测试和维护的难度。

功能切片设计有助于对关注点做清晰划分，并为功能划定边界。这允许不同团队或开发人员独立处理不同功能，而不必分神于冲突或依赖项管理。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/XIibZ0YbvibkXrIAkibiaHqIEYd03mTCUgQtIYk4ZYtXMZFR7XlZPjFzPic9CvxByaNxXSlKUz71rxC0ibsq5zeALEbg/640?wx_fmt=png&from=appmsg)

