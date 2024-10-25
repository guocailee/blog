---
{"dg-publish":true,"permalink":"/Program/FrontEnd/使用 React Hooks 的心智负担/","noteIcon":""}
---


> 原文： [Mental Overhead of React Hooks​liyuanqiu.com/docs/mental-overhead-of-react-hooks](https://link.zhihu.com/?target=https%3A//liyuanqiu.com/docs/mental-overhead-of-react-hooks)

* * *

当我们初次接触 `React Hooks` 时，我相信很多人都会很兴奋，迫不及待地想尝鲜。

但很快，我们尝到了苦头，然后明白了一个真相：

```text
React Hooks 不是那么容易掌握的
```

我们来看下面一个代码段：

```ts
// 这是一个典型的自定义 Hook
// 它从服务端获取一个 foo 对象
function useFoo(): undefined | Foo {
  const [foo, setFoo] = useState<undefined | Foo>();
  useEffect(() => {
    fetch(...)
      .then(res => res.json())
      .then(json => setFoo(json));
  }, []);
  return foo;
}

```

自定义 Hook `useFoo` 非常容易理解。它的返回值将会是 undefined，直到服务器给出了正确的 JSON 数据。

我们再看另一个自定义 Hook：

```ts
function useBar(foo: Foo): Bar {
  return useMemo(() => {
    // 根据 foo 计算出一个 Bar 的实例，并返回
  }, [foo]);
}

```

好了，好戏开始了：

```ts
import useFoo from './useFoo';
import useBar from './useBar';

function MyReactComponent() {
  const foo = useFoo();
  // 哦！一些果酱溅到了我的脸上！（译者注：？？？）
  const bar = useBar(foo);
  return ...;
}

```

此处有 bug，`useBar` 只接受一个类型为 `Foo` 的 `foo`，但实际传入的 `foo` 可能是 `undefined`，因此，我们需要在某处判断一下 `foo` 是否为 `undefined`：

```ts
function MyReactComponent() {
  const foo = useFoo();
  if (foo === undefined) {
    return null;
  }
  // 举起手帕。。。
  const bar = useBar(foo);
  return ...;
}

```

显然，犯了个大错误，React Hooks 是不允许条件性地调用的。（译者注：[Rules of Hooks – React](https://link.zhihu.com/?target=https%3A//reactjs.org/docs/hooks-rules.html%23only-call-hooks-at-the-top-level)）

所以我们只能将判断塞进 `useBar` 内部。。。

```ts
function useBar(foo: undefined | Foo): Bar {
  return useMemo(() => {
    if (foo === undefined) {
      return <此处返回个啥？又一个 undefined？>;
    }
    // 根据 foo 计算出一个 Bar 的实例，并返回
  }, [foo]);
}

```

这样做是能够解决问题的，但实在很丑陋，且根本不优雅：

-   对于 `useFoo`，它返回 `undefined | Foo` 是合理的，这个行为符合其业务逻辑。
-   对于 `useBar`，它完全没有必要关心传进来的 `foo` 是否是 `undefined`，请记住，我们的名字是 `useBar`，不是 `useBarByUseFoo`。

而且，如果 `useBar` 能够接受 `undefined`，那它应该返回什么？一个空的 `Bar` 对象？还是又一个 `undefined`？无论哪一种，都不合理。

（译者注：而且 `useFoo` 和 `useBar` 不一定是你写的，你就更没可能修改它们了。）

所以最终，我选择包装 `MyReactComponent`：

```ts
import useFoo from './useFoo';
import useBar from './useBar';

function MyReactComponent({ foo }: { foo: Foo }) {
  const bar = useBar(foo);
  return ...;
}

function FooWrapper() {
  const foo = useFoo();
  if (foo === undefined) {
    return null;
  }
  return <MyReactComponent foo={foo} />;
}

export default FooWrapper;

```

最终，问题得到了解决，但我的脑子累了个半死。

`React Hooks` 很酷，但真的需要很强的编程思维去追随它。我们写 Hooks，然后重新思考它，然后重构代码，一种循环，直到退休。（译者注：退休？？？什么鬼？？？）

 [https://www.zhihu.com/question/350523308](https://www.zhihu.com/question/350523308)
