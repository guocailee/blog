---
{"dg-publish":true,"permalink":"/Program/Flutter/大型APP Route管理-Flutter/","noteIcon":""}
---


收录于: [[Flutter\|Flutter]]   [[Flutter\|Flutter]]   [[FE\|FE]]

前面大家了解了Web和Native端的路由管理，这篇文章出场的是大前端领域备受开发者喜欢的新秀Flutter。  

Flutter作为一款跨平台UI框架，借鉴了React（Web开发框架）的响应式的UI框架设计思想等。在Flutter中，一切皆是Widget（组件），其中StatefulWidget（有状态的组件）和React中的组件类似，可以通过数据变化去手动更新视图，Flutter路由管理实现的核心组件Navigator就是这样的一个StatefulWidget。

## 1、认识Flutter路由导航

### 1.1 Route（路由页面）
---------------

页面的包装类，一个页面想要被路由统一管理，必须包装为一个Route，Route并不是一个widget，但是在页面栈实现中起到至关重要的作用，它是一个抽象类，其继承关系如下，其中每一层都处理了与其相关的一些逻辑。

![](/img/user/Dictionary/attchements/media/640-14.png)

*   **OverlayRoute**：在导航器的Overlay中显示控件的路由。主要将路由转换为Widget插入控件树。
    
*   **TransitionRoute**：具有进入和退出过渡动画的路由。主要处理路由过渡动效。
    
*   **ModalRoute**：阻止与下层路由交互的路由。它覆盖整个导航器。但它们不一定是不透明的。例如一个对话框。主要处理事件的拦截。
    
*   **PageRoute**：替换整个屏幕的模态路由。由它派生出了我们熟悉的**MaterialPageRoute**，主要用于Flutter的页面切换。
    
*   **PopupRoute**：在当前路由上覆盖Widget的模态路由。主要用于弹出框，对话框之类。
    

### 1.2 Navigator（导航器）

管理所有的Route的Widget，实现路由导航的核心widget。它维护了一个路由栈集合（List<\_RouteEntry>），当你调用push，pop方法时，Navigator都会以栈的方式对这个集合进行添加或删除，并通过路由栈状态变化实现对页面栈的更新。

*   我们不需要手动创建Navigator，开发中使用的MaterialApp、CupertinoApp、WidgetsApp它们默认是有插入Navigator的，我们在需要的时候可以直接使用Navigator.of(context)。Navigator的widget构建流程如下： 
    

![](/img/user/Dictionary/attchements/media/640-14.png)

*   当我们想使用导航操作时，Navigator提供了如下几个常用的方法：
    

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

真正的页面栈，Overlay可以翻译为叠加层或覆盖层。按照官方的解释，它是一个可以独立管理的覆盖层堆栈。它维护一个页面栈集合（List<OverlayEntry>），实现页面栈到widget的转换过程，它同时也拥有一个私有类\_Theatre来进行页面widget的绘制。

*   \_Theatre：它的名字非常形象的表达了它的功能：剧院。你有很多组件以一层层覆盖的模式绘制在界面上时，如果其中某一层的组件以全屏不透明的模式绘制在界面上，那它下层的组件就不需要再进行绘制了。
    

## 2、Flutter路由管理实现

### 2.1 导航器初始化

        Navigator是一个有状态的widget，NavigatorState在初始化时主要做了两件事：

1.  根据配置参数创建初始化路由，初始化路由会放入\_history（路由栈）里面去
    
2.  调用\_flushHistoryUpdates方法刷新路由栈
    

        这里最为复杂也是最核心的就是这个刷新路由栈的方法，该方法会根据不同的路由生命周期（\_RouteLifecycle）做相应处理。初始化创建的路由会设置其路由状态为\_RouteLifecycle.add，在\_flushHistoryUpdates中会调用route的插入方法将根路由转换为OverlayEntry对象，插入到Overlay管理的页面栈中并通知Overlay更新视图。下面是伪代码，可以简单了解一下：

```cs
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

### 2.2 页面绘制

        Overlay在Navigator的build方法中创建，Overlay会遍历OverlayEntry列表，将保存的实体信息对象封装为\_OverlayEntryWidget控件，最终将包含\_OverlayEntryWidget的列表交给\_Theatre控件插入控件树中用于渲染。

        \_Theatre控件将页面分为了两种，一种是舞台上的（onstage）演员，另一种则是舞台下的（offstage）观众。当某个包装页面的OverlayEntry的opaque属性为true时，表示占满全屏且不透明，那么以它为分界线，它之下的所有页面都不需要绘制了（因为被挡住了看不见）。如果OverlayEntry的maintainState属性也为true，则被分到舞台下的观众那一组，否则，没有进入剧院的资格。

![](/img/user/Dictionary/attchements/media/640-1.jpg)

### 2.3 导航操作

        前面我们了解了Navigator的初始化构建流程，对于页面栈刷新已经有了一些认识，接下来我们来看看push方法都做了些什么。

```typescript
Future<T> push<T extends Object>(Route<T> route) {
  // 使用_RouteEntry包装传进来的路由然后入栈，设置路由状态为push
  _history.add([_RouteEntry(route, _RouteLifecycle.push)]);
  _flushHistoryUpdates();// 刷新路由栈
  ...
}
```

        在刷新路由栈的时候push状态的路由也会插入两个新的OverlayEntry，并在所有操作完成后触发Overlay更新。下图是push前后各widget中的栈的变化。

![](/img/user/Dictionary/attchements/media/640-14.png)

**3、**Flutter路由管理实现总结

        从以上流程实现可以看出，Flutter页面栈的实现依赖于：

*   Navigator持有路由栈，提供导航方法，当路由栈发生变化时同步更新页面栈。
    
*   Overlay持有页面栈，它实现页面栈到渲染用widget集合的转换过程，并能够接受到路由栈更新的通知去同步更新视图。
    
*   \_Theatre负责对页面widget进行动态构建和绘制，保证渲染性能。
    

  

        上面讲到是纯Flutter中路由管理的实现，但是在我们开发中可能还会遇到Flutter-Native混编的模式，对这块感兴趣的同学们可以在我们团队这篇文章 [Flutter 核心原理与混合开发模式](http://mp.weixin.qq.com/s?__biz=MzI1NjEwMTM4OA==&mid=2651233359&idx=1&sn=801697fcda8ba5ac17510ff86b034f65&chksm=f1d9e12cc6ae683a1245aa2d2762493774ff9a76c92392e1064c9372f1ff16396ef8b4d2d6ed&scene=21#wechat_redirect) 中解锁更多知识。

  


  

        至此，我们了解到了Flutter端是如何去实现路由管理的，那么，就请期待我们最后一篇文章**《大前端开发中的路由管理之六：总结篇》**。

## 参考资料

1.  [Flutter路由源码剖析]( https://zhuanlan.zhihu.com/p/208788731)
2. [ Flutter路由导航](https://zhuanlan.zhihu.com/p/144820879)



  