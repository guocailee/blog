---
{"dg-publish":true,"permalink":"/Program/FrontEnd/preventDefault的一点妙用/","noteIcon":"","created":"2024-05-22T16:17:54.148+08:00"}
---


众所周知，event.preventDefault() 用于阻止浏览器的默认行为。如 a 标签的点击事件，默认行为是跳转，但可以在 onClick 的处理函数中使用 e.preventDefault() 阻止跳转，如下：

```html
  <a href="http://www.baidu.com" id="baiduLink">去百度</a>
  <script>
    document.querySelector('#baiduLink').addEventListener('click', e => {
      e.preventDefault() // 这里会阻止跳往百度
    })
  </script>
```

**我们在组件设计上能否借用 preventDefault() 来干一些事情呢？**

假设我们有一个 Task 组件，该组件用于展示任务名称，并提供一个删除按钮。点击删除将会删除该任务。

```js
function Task(props: {
  name: string,
  id: string,
  onDelete?: (id:string, e: React.MouseEvent<HTMLButtonElement, MouseEvent>) => void
}) {
​
  function internalDelete (id:string, e: React.MouseEvent<HTMLButtonElement, MouseEvent>) {
    props.onDelete?.(id, e);
    console.log('开始执行内部删除逻辑')
  }
​
  return <p> {props.name} <button onClick={e => internalDelete(props.id, e)}>Del</button> </p>
}

```

**如果我们想让用户控制是否允许删除任务可以怎么办呢？**

方法 1：提供一个 showDeleteButton 的 prop, 如果返回 false 不显示删除按钮

方法 2: 给用户用户传入的 onDelete 方法的返回值赋予一些特殊含义，如果返回 false 则不允许删除

方法 3： 用户在 onDelete 方法里面使用 event.preventDefault() 来阻止组件的默认删除行为。

方法 3 的关键点在于**event 有一个属性`defaultPrevented`** 来判断事件在前面是否已经调用 preventDefault(); 在 react 的合成事件中有一个`isDefaultPrevented()`方法跟原生的这个属性用处相同。

```js
function Task(props: {
  name: string,
  id: string,
  onDelete?: (id:string, e: React.MouseEvent<HTMLButtonElement, MouseEvent>) => void
}) {
​
  function internalDelete (id:string, e: React.MouseEvent<HTMLButtonElement, MouseEvent>) {
    props.onDelete?.(id, e);
​
    // ⚠️⚠️ 这里可以判断是否用户想阻止删除
    if (e.isDefaultPrevented()) {
      console.log('用户想阻止删除')
      return
    }
​
    console.log('开始执行内部删除逻辑')
  }
​
  return <p> {props.name} <button onClick={e => internalDelete(props.id, e)}>Del</button> </p>
}
​
export default () => {
  function onDelete (id: string, event: React.MouseEvent<HTMLButtonElement, MouseEvent>) {
    if (id === '1') {
      console.log(`id为1的任务不允许删除！`)
      event.preventDefault();
    } else {
      console.log('其他的随便你删除吧')
    }
  }
​
  return <Task id={'1'} name={'task name'} onDelete={onDelete}></Task>
}

```


event.preventDefault() 与 e.defaultPrevented 是否给你的组件设计提供了一种新思路呢？ 
