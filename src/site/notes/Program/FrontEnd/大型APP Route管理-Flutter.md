---
{"dg-publish":true,"permalink":"/Program/FrontEnd/大型APP Route管理-Flutter/","noteIcon":"","created":"2024-05-22T16:17:54.143+08:00"}
---


## 1、认识 Flutter 路由导航

前面大家了解了 Web 和 Native 端的路由管理，这篇文章出场的是大前端领域备受开发者喜欢的新秀 Flutter。

Flutter 作为一款跨平台 UI 框架，借鉴了 React（Web 开发框架）的响应式的 UI 框架设计思想等。在 Flutter 中，一切皆是 Widget（组件），其中 StatefulWidget（有状态的组件）和 React 中的组件类似，可以通过数据变化去手动更新视图，Flutter 路由管理实现的核心组件 Navigator 就是这样的一个 StatefulWidget。

### 1.1 Route（路由页面）

页面的包装类，一个页面想要被路由统一管理，必须包装为一个 Route，Route 并不是一个 widget，但是在页面栈实现中起到至关重要的作用，它是一个抽象类，其继承关系如下，其中每一层都处理了与其相关的一些逻辑。

![](/img/user/z-attchements/media/640-14.png)

- **OverlayRoute**：在导航器的 Overlay 中显示控件的路由。主要将路由转换为 Widget 插入控件树。
- **TransitionRoute**：具有进入和退出过渡动画的路由。主要处理路由过渡动效。
- **ModalRoute**：阻止与下层路由交互的路由。它覆盖整个导航器。但它们不一定是不透明的。例如一个对话框。主要处理事件的拦截。
- **PageRoute**：替换整个屏幕的模态路由。由它派生出了我们熟悉的**MaterialPageRoute**，主要用于 Flutter 的页面切换。
- **PopupRoute**：在当前路由上覆盖 Widget 的模态路由。主要用于弹出框，对话框之类。

### 1.2 Navigator（导航器）

管理所有的 Route 的 Widget，实现路由导航的核心 widget。它维护了一个路由栈集合（`List<RouteEntry>`），当你调用 push，pop 方法时，Navigator 都会以栈的方式对这个集合进行添加或删除，并通过路由栈状态变化实现对页面栈的更新。

- 我们不需要手动创建 Navigator，开发中使用的 MaterialApp、CupertinoApp、WidgetsApp 它们默认是有插入 Navigator 的，我们在需要的时候可以直接使用 Navigator.of(context)。Navigator 的 widget 构建流程如下：

![](/img/user/z-attchements/media/640-14.png)

- 当我们想使用导航操作时，Navigator 提供了如下几个常用的方法：

```typescript
// 路由跳转：传入一个路由对象
Future<T> push<T extends Object>(Route<T> route)

// 路由跳转：传入一个名称（命名路由）
Future<T> pushNamed<T extends Object>(
  String routeName, {
    Object arguments,
  })

// 路由返回：可以传入一个参数
bool pop<T extends Object>([ T result ])
```

### 1.3. Overlay（页面栈）

真正的页面栈，Overlay 可以翻译为叠加层或覆盖层。按照官方的解释，它是一个可以独立管理的覆盖层堆栈。它维护一个页面栈集合（`List<OverlayEntry>`），实现页面栈到 widget 的转换过程，它同时也拥有一个私有类 Theatre 来进行页面 widget 的绘制。

- Theatre：它的名字非常形象的表达了它的功能：剧院。你有很多组件以一层层覆盖的模式绘制在界面上时，如果其中某一层的组件以全屏不透明的模式绘制在界面上，那它下层的组件就不需要再进行绘制了。

## 2、Flutter 路由管理实现

### 2.1 导航器初始化

Navigator 是一个有状态的 widget，NavigatorState 在初始化时主要做了两件事：

1. 根据配置参数创建初始化路由，初始化路由会放入 history（路由栈）里面去
2. 调用 flushHistoryUpdates 方法刷新路由栈

这里最为复杂也是最核心的就是这个刷新路由栈的方法，该方法会根据不同的路由生命周期（RouteLifecycle）做相应处理。初始化创建的路由会设置其路由状态为 RouteLifecycle.add，在 flushHistoryUpdates 中会调用 route 的插入方法将根路由转换为 OverlayEntry 对象，插入到 Overlay 管理的页面栈中并通知 Overlay 更新视图。下面是伪代码，可以简单了解一下：

```dart
// [NavigatorState] 伪代码
List<_RouteEntry> _history = <_RouteEntry>[]; // 路由栈集合
void initState() {  _history.addAll([_RouteEntry(route, _RouteLifecycle.add)]);// 1.依据初始化构建参数创建初始化路由_RouteEntry,放入_history集合
  _flushHistoryUpdates(); // 2.更新页面栈
}
void _flushHistoryUpdates() {
  // 1.遍历路由栈集合
  // 2.状态为_RouteLifecycle.add的路由调用
  while(遍历条件) {
    switch (路由状态) {
      case _RouteLifecycle.add: // 执行add对应方法 初始化跟路由会执行
      case _RouteLifecycle.push: // 执行push动作对应方法
      ...
    }
  }
  // 3.对应路由监听器触发，无用路由和页面销毁，overlay组件更新
}
Widget build() {
  // 构建Overlay 并通过一个_overlayKey持有OverlayState
}
```

### 2.2  页面绘制

Overlay 在 Navigator 的 build 方法中创建，Overlay 会遍历 OverlayEntry 列表，将保存的实体信息对象封装为 OverlayEntryWidget 控件，最终将包含 OverlayEntryWidget 的列表交给 Theatre 控件插入控件树中用于渲染。

Theatre 控件将页面分为了两种，一种是舞台上的（onstage）演员，另一种则是舞台下的（offstage）观众。当某个包装页面的 OverlayEntry 的 opaque 属性为 true 时，表示占满全屏且不透明，那么以它为分界线，它之下的所有页面都不需要绘制了（因为被挡住了看不见）。如果 OverlayEntry 的 maintainState 属性也为 true，则被分到舞台下的观众那一组，否则，没有进入剧院的资格。

![](/img/user/z-attchements/media/640-1.jpg)

### 2.3  导航操作

前面我们了解了 Navigator 的初始化构建流程，对于页面栈刷新已经有了一些认识，接下来我们来看看 push 方法都做了些什么。

```typescript
Future<T> push<T extends Object>(Route<T> route) {
  // 使用_RouteEntry包装传进来的路由然后入栈，设置路由状态为push
  _history.add([_RouteEntry(route, _RouteLifecycle.push)]);
  _flushHistoryUpdates();// 刷新路由栈
  ...
}
```

在刷新路由栈的时候 push 状态的路由也会插入两个新的 OverlayEntry，并在所有操作完成后触发 Overlay 更新。下图是 push 前后各 widget 中的栈的变化。

![](/img/user/z-attchements/media/640-14.png)

**3、**Flutter 路由管理实现总结

从以上流程实现可以看出，Flutter 页面栈的实现依赖于：

- Navigator 持有路由栈，提供导航方法，当路由栈发生变化时同步更新页面栈。
- Overlay 持有页面栈，它实现页面栈到渲染用 widget 集合的转换过程，并能够接受到路由栈更新的通知去同步更新视图。
- Theatre 负责对页面 widget 进行动态构建和绘制，保证渲染性能。

上面讲到是纯 Flutter 中路由管理的实现，但是在我们开发中可能还会遇到 Flutter-Native 混编的模式，对这块感兴趣的同学们可以在我们团队这篇文章  [Flutter 核心原理与混合开发模式](http://mp.weixin.qq.com/s?__biz=MzI1NjEwMTM4OA==&mid=2651233359&idx=1&sn=801697fcda8ba5ac17510ff86b034f65&chksm=f1d9e12cc6ae683a1245aa2d2762493774ff9a76c92392e1064c9372f1ff16396ef8b4d2d6ed&scene=21#wechat_redirect)  中解锁更多知识。

至此，我们了解到了 Flutter 端是如何去实现路由管理的，那么，就请期待我们最后一篇文章**《大前端开发中的路由管理之六：总结篇》**。

## 参考资料

1.  [Flutter 路由源码剖析](https://zhuanlan.zhihu.com/p/208788731) 
2. [ Flutter 路由导航](https://zhuanlan.zhihu.com/p/144820879)
