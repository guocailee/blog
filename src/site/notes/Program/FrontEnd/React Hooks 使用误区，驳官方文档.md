---
{"dg-publish":true,"permalink":"/Program/FrontEnd/React Hooks 使用误区，驳官方文档/","tags":["React"],"noteIcon":"","created":"2025-03-06T21:28:25.972+08:00"}
---

作为 React Hooks 库 [ahooks](https://github.com/alibaba/hooks) 的作者，我应该算一个非常非常资深的 React Hooks 用户。在两年多的 React Hooks 使用过程中，我越来越发现大家（包括我自己）对 React Hooks 的使用姿势存在很大误区，归根到底是官方文档的教程很不严谨，存在错误的指引。

## 1. 不是所有的依赖都必须放到依赖数组中

对于所有的 React Hooks 用户，都有一个共识：“useEffect 中使用到外部变量，都应该放到第二个数组参数中”，同时我们会安装 [eslint-plugin-react-hooks](https://link.zhihu.com/?target=https%3A//www.npmjs.com/package/eslint-plugin-react-hooks%23installation) 插件，来提醒自己是不是忘了某些变量。

以上共识来自官方文档：

![](https://pic1.zhimg.com/80/v2-ebf96ba6957ed68a2aafe8680f0ae718_720w.jpg)

我愿称该条规则为万恶之源，这条规则以高亮展示，所有的新人都很重视，包括我自己。然而在实际的开发中，发现事情并不是这样的。

下面举一个比较简单的例子，要求如下：当 `props.count` 和 `count` 变化时，上报当前所有数据。

这个例子比较简单，先贴下源码：

```js
function Demo(props) {
  const [count, setCount] = useState(0);
  const [text, setText] = useState('');
  const [a, setA] = useState('');

  useEffect(() => {
    monitor(props.count, count, text, a);
  }, [props.count, count]);

  return (
    <div>
      <button
        onClick={() => setCount(c => c + 1)}
      >
        click
      </button>
      <input value={text} onChange={e => setText(e.target.value)} />
      <input value={a} onChange={e => setA(e.target.value)} />
    </div>
  )
}

```

我们能看到示例代码中，useEffect 是不符合 React 官方建议的，`text` 和 `a` 变量没有放到依赖数组中，ESLint 警告如下：

![](https://pic4.zhimg.com/80/v2-df7e7bad9268b45a406fa84cafb2692f_720w.jpg)

那如果按照规范，我们把依赖项都放到第二个数组参数中，会怎样呢？

```js
  useEffect(() => {
    monitor(props.count, count, text, a);
  }, [props.count, count, text, a]);

```

如上的代码虽然符合了 React 官方的规范，但不满足我们的业务需求了，当 `text` 或者 `a` 变化时，也触发了函数执行。

此时陷入了困境，当满足 useEffect 使用规范时，业务需求就不能满足了。当满足业务需求时，useEffect 就不规范了。

我的建议为：

1.  **不要使用** `eslint-plugin-react-hooks` **插件，或者可以选择性忽略该插件的警告。** 
2.  **只有一种情况，需要把变量放到 deps 数组中，那就是当该变量变化时，需要触发 useEffect 函数执行。而不是因为 useEffect 中用到了这个变量！**

## 2. deps 参数不能缓解闭包问题

假如完全按第二个建议来写代码，很多人又担心，会不会造成一些不必要的闭包问题？我的结论是：**闭包问题和 useEffect 的 deps 参数没有太大关系。** 

比如我有一个这样的需求：当进入页面 3s 后，输出当前最新的 count。代码如下：

```js
function Demo() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const timer = setTimeout(() => {
      console.log(count)
    }, 3000);
    return () => {
      clearTimeout(timer);
    }
  }, [])

  return (
    <button
      onClick={() => setCount(c => c + 1)}
    >
      click
    </button>
  )
}

```

以上代码，实现了初始化 3s 后，输出 count。但很遗憾，这里肯定会出闭包问题，哪怕进来之后我们多次点击了 button，输出的 count 仍然为 0。

那假如我们把 `count` 放到 deps 中，是不是就好了？

```js
useEffect(() => {
    const timer = setTimeout(() => {
      console.log(count)
    }, 3000);
    return () => {
      clearTimeout(timer);
    }
  }, [count])

```

如上代码，此时确实没有闭包问题了，但在每次 `count` 变化时，定时器卸载并重新开始计时了，不满足我们的最初需求了。

要解决的唯一办法为：

```js
const [count, setCount] = useState(0);

// 通过 ref 来记忆最新的 count
const countRef = useRef(count);
countRef.current = count;

useEffect(() => {
  const timer = setTimeout(() => {
    console.log(countRef.current)
  }, 3000);
  return () => {
    clearTimeout(timer);
  }
}, [])

```

虽然上面的代码，很绕，但确实，只有这个解决方案。请记住这段代码，功能真的很强大。

```js
const countRef = useRef(count);
countRef.current = count;

```

上面的例子，可以发现，闭包问题是不能仅仅通过遵守 React 规则来避免的。我们必须清晰的知道，在什么场景下会出现闭包问题。

### 2.1 正常情况下是不会有闭包问题的

```js
const [a, setA] = useState(0);
const [b, setB] = useState(0);

const c = a + b;

useEffect(()=>{
    console.log(a, b, c)
}, [a]);

useEffect(()=>{
    console.log(a, b, c)
}, [b]);

useEffect(()=>{
    console.log(a, b, c)
}, [c]);

```

在一般的使用过程中，是不会有闭包问题的，如上代码中，完全不会有闭包问题，和 deps 怎么写没有任何关系。

### 2.2 延迟调用会存在闭包问题

**在延迟调用的场景下，一定会存在闭包问题。**  什么是延迟调用？

1.  使用 setTimeout、setInterval、Promise.then 等
2.  useEffect 的卸载函数

```js
const getUsername = () => {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      resolve('John');
    }, 3000);
  })
}

function Demo() {
  const [count, setCount] = useState(0);

  // setTimeout 会造成闭包问题
  useEffect(() => {
    const timer = setTimeout(() => {
      console.log(count);
    }, 3000);
    return () => {
      clearTimeout(timer);
    }
  }, [])

  // setInterval 会造成闭包问题
  useEffect(() => {
    const timer = setInterval(() => {
      console.log(count);
    }, 3000);
    return () => {
      clearInterval(timer);
    }
  }, [])

  // Promise.then 会造成闭包问题
  useEffect(() => {
    getUsername().then(() => {
      console.log(count);
    });
  }, [])

  // useEffect 卸载函数会造成闭包问题
  useEffect(() => {
    return () => {
      console.log(count);
    }
  }, []);

  return (
    <button
      onClick={() => setCount(c => c + 1)}
    >
      click
    </button>
  )
}

```

在以上示例代码中，四种情况均会出现闭包问题，永远输出 `0`。这四种情况的根因都是一样的，我们看一下代码的执行顺序：

1.  组件初始化，此时 `count = 0`
2.  执行 useEffect，此时 useEffect 的函数执行，JS 引用链记录了对 `count=0` 的引用关系
3.  点击 button，count 变化，但对之前的引用已经无能为力了

可以看到，闭包问题均是出现在延迟调用的场景下。解决办法如下：

```js
const [count, setCount] = useState(0);

// 通过 ref 来记忆最新的 count
const countRef = useRef(count);
countRef.current = count;

useEffect(() => {
  const timer = setTimeout(() => {
    console.log(countRef.current)
  }, 3000);
  return () => {
    clearTimeout(timer);
  }
}, [])

......

```

通过 `useRef` 来保证任何时候访问的 `countRef.current` 都是最新的，以解决闭包问题。

到这里，我重申下我对 useEffect 的建议：

1.  **只有变化时，需要重新执行 useEffect 的变量，才要放到 deps 中。而不是 useEffect 用到的变量都放到 deps 中。** 
2.  **在有延迟调用场景时，可以通过 ref 来解决闭包问题。** 

## 3. 尽量不要用 useCallback

我建议在项目中尽量不要用 useCallback，大部分场景下，不仅没有提升性能，反而让代码可读性变的很差。

### 3.1 useCallback 大部分场景没有提升性能

useCallback 可以记住函数，避免函数重复生成，这样函数在传递给子组件时，可以避免子组件重复渲染，提高性能。

```js
const someFunc = useCallback(()=> {
   doSomething();
}, []);

return <ExpensiveComponent func={someFunc} />

```

基于以上认知，很多同学（包括我自己）在写代码时，只要是个函数，都加个 useCallback，是你么？反正我以前是。

但我们要注意，提高性能还必须有另外一个条件，子组件必须使用了 `shouldComponentUpdate` 或者 `React.memo` 来忽略同样的参数重复渲染。

假如 `ExpensiveComponent` 组件只是一个普通组件，是没有任何用的。比如下面这样：

```js
const ExpensiveComponent = ({ func }) => {
  return (
    <div onClick={func}>
        hello
    </div>
  )
}

```

必须通过 `React.memo` 包裹 `ExpensiveComponent` ，才会避免参数不变的情况下的重复渲染，提高性能。

```js
const ExpensiveComponent = React.memo(({ func }) => {
  return (
    <div onClick={func}>
        hello
    </div>
  )
})

```

**所以，useCallback 是要和 `shouldComponentUpdate/React.memo` 配套使用的，你用对了吗？当然，我建议一般项目中不用考虑性能优化的问题，也就是不要使用 useCallback 了，除非有个别非常复杂的组件，单独使用即可。** 

### 3.2 useCallback 让代码可读性变差

我看到过一些代码，使用 useCallback 后，大概长这样：

```js
const someFuncA = useCallback((d, g, x, y)=> {
   doSomething(a, b, c, d, g, x, y);
}, [a, b, c]);

const someFuncB = useCallback(()=> {
   someFuncA(d, g, x, y);
}, [someFuncA, d, g, x, y]);

useEffect(()=>{
  someFuncB();
}, [someFuncB]);

```

在上面的代码中，变量依赖一层一层传递，最终要判断具体哪些变量变化会触发 useEffect 执行，是一件很头疼的事情。

我期望不要用 useCallback，直接裸写函数就好：

```js
const someFuncA = (d, g, x, y)=> {
   doSomething(a, b, c, d, g, x, y);
};

const someFuncB = ()=> {
   someFuncA(d, g, x, y);
};

useEffect(()=>{
  someFuncB();
}, [...]);

```

在 useEffect 存在延迟调用的场景下，可能造成闭包问题，那通过咱们万能的方法就能解决：

```diff
const someFuncA = (d, g, x, y)=> {
   doSomething(a, b, c, d, g, x, y);
};

const someFuncB = ()=> {
   someFuncA(d, g, x, y);
};

+ const someFuncBRef = useRef(someFuncB);
+ someFuncBRef.current = someFuncB;

useEffect(()=>{
+  setTimeout(()=>{
+    someFuncBRef.current();
+  }, 1000)
}, [...]);

```

**对 useCallback 的建议就一句话：没事别用 useCallback。** 

## 4. useMemo 建议适当使用

相较于 useCallback 而言，useMemo 的收益是显而易见的。

```js
// 没有使用 useMemo
const memoizedValue = computeExpensiveValue(a, b);

// 使用 useMemo
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);

```

如果没有使用 useMemo，`computeExpensiveValue` 会在每一次渲染的时候执行。如果使用了 `useMemo`，只有在 `a` 和 `b` 变化时，才会执行一次 `computeExpensiveValue`。

这笔账大家应该都会算，**所以我建议 useMemo 可以适当使用。** 

当然也不是无节制的使用，在很简单的基础类型计算时，可能 useMemo 并不划算。

```js
const a = 1;
const b = 2;

const c = useMemo(()=> a + b, [a, b]);

```

比如上面的例子，请问计算 `a+b` 的消耗大？还是记录 `a/b` ，并比较`a/b` 是否变化的消耗大？

明显 `a+b` 消耗更小。

```js
const a = 1;
const b = 2;

const c = a + b;

```

这笔账大家可以自己算，我建议简单的基础类型计算，就不要用 useMemo 了~

## 5. useState 的正确使用姿势

useState 应该算最简单的一个 Hooks，但在使用中，也有很多技巧可循，如果严格按照以下几点，代码可维护性直接翻倍。

### 5.1 能用其他状态计算出来就不用单独声明状态

**一个 state 必须不能通过其它 state/props 直接计算出来，否则就不用定义 state。** 

```js
const SomeComponent = (props) => {
  
  const [source, setSource] = useState([
      {type: 'done', value: 1},
      {type: 'doing', value: 2},
  ])
  
  const [doneSource, setDoneSource] = useState([])
  const [doingSource, setDoingSource] = useState([])

  useEffect(() => {
    setDoingSource(source.filter(item => item.type === 'doing'))
    setDoneSource(source.filter(item => item.type === 'done'))
  }, [source])
  
  return (
    <div>
       ..... 
    </div>
  )
}

```

上面的示例中，变量 `doneSource` 和 `doingSource` 可以通过变量 `source` 计算出来，那就不要定义 `doneSource` 和 `doingSource` 了！

```js
const SomeComponent = (props) => {
  
  const [source, setSource] = useState([
      {type: 'done', value: 1},
      {type: 'doing', value: 2},
    ])
  
  const doneSource = useMemo(()=> source.filter(item => item.type === 'done'), [source]);
  const doingSource = useMemo(()=> source.filter(item => item.type === 'doing'), [source]);
  
  return (
    <div>
       ..... 
    </div>
  )
}

```

一般在项目中此类问题都比较隐晦，层层传递，在 Code Review 中很难一眼看出。如果能把变量定义清楚，那事情就成功了一半。

### 5.2 保证数据源唯一

**在项目中同一个数据，保证只存储在一个地方。** 

不要既存在 redux 中，又在组件中定义了一个 state 存储。

不要既存在父级组件中，又在当前组件中定义了一个 state 存储。

不要既存在 url query 中，又在组件中定义了一个 state 存储。

```js
function SearchBox({ data }) {
  const [searchKey, setSearchKey] = useState(getQuery('key'));

  const handleSearchChange = e => {
    const key = e.target.value;
    setSearchKey(key);
    history.push(`/movie-list?key=${key}`);
  }

  return (
      <input
        value={searchKey}
        placeholder="Search..."
        onChange={handleSearchChange}
      />
  );
}

```

在上面的示例中，`searchKey` 存储在两个地方，既在 url query 上，又定义了一个 state。完全可以优化成下面这样：

```js
function SearchBox({ data }) {
  const searchKey = parse(localtion.search)?.key;

  const handleSearchChange = e => {
    const key = e.target.value;
    history.push(`/movie-list?key=${key}`);
  }

  return (
      <input
        value={searchKey}
        placeholder="Search..."
        onChange={handleSearchChange}
      />
  );
}

```

在实际项目开发中，此类问题也是比较隐晦，编码时应注意。

### 5.3 useState 适当合并

项目中有木有写过这样的代码：

```js
const [firstName, setFirstName] = useState();
const [lastName, setLastName] = useState();
const [school, setSchool] = useState();
const [age, setAge] = useState();
const [address, setAddress] = useState();

const [weather, setWeather] = useState();
const [room, setRoom] = useState();

```

反正我最开始是写过，useState 拆分过细，导致代码中一大片 useState。

我建议，同样含义的变量可以合并成一个 state，代码可读性会提升很多：

```js
const [userInfo, setUserInfo] = useState({
  firstName,
  lastName,
  school,
  age,
  address
});

const [weather, setWeather] = useState();
const [room, setRoom] = useState();

```

当然这种方式我们在变更变量时，一定不要忘记带上老的字段，比如我们只想修改 `firstName`：

```js
setUserInfo(s=> ({
  ...s,
  fristName,
}))

```

其实如果是 React Class 组件，state 是会自动合并的：

```js
this.setState({
  firstName
})

```

在 Hooks 中，可以有这种用法吗？其实是可以的，我们自己封装一个 Hooks 就可以，比如 ahooks 的 [useSetState](https://ahooks.js.org/zh-CN/hooks/use-set-state/)，就封装了类似的逻辑：

```js
const [userInfo, setUserInfo] = useSetState({
  firstName,
  lastName,
  school,
  age,
  address
});

// 自动合并
setUserInfo({
  firstName
})

```

我自己在项目中大量使用了 `useSetState` 来代替 `useState`，来管理复杂类型的 state，妈妈更爱我了。

## 六、总结

作为资深的 React Hooks 用户，我很认可 React Hooks 带来的提效，这也是我这几年完全拥抱 Hooks 的原因。同时我也越来越觉得 React Hooks 难驾驭，尤其随着 React 18 的 concurrent mode 的到来，不知道会带来什么坑。

最后再给大家三个建议：

1.  可以多使用别人封装好的高级 Hooks 来提效，比如 [ahooks](https://github.com/alibaba/hooks) 库（哈哈哈
2.  可以多看看别人封装好的 Hooks 源码，加深对 React Hooks 理解，比如 [ahooks](https://link.zhihu.com/?target=https%3A//github.com/alibaba/hooks) 库（哈哈哈
3.  可以关注下我的公众号【**前端技术砖家**】，我会经常发布一些我自己写的技术文章，以及转发一些我认为比较好的文章，爱你哟（づ￣3￣）づ╭❤～

    [https://zhuanlan.zhihu.com/p/450513902](https://zhuanlan.zhihu.com/p/450513902)
