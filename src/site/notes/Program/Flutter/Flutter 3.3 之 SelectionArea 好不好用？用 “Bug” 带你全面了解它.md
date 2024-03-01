---
{"dg-publish":true,"permalink":"/program/flutter/flutter-3-3-selection-area-bug/","noteIcon":""}
---


随着 Flutter 3.3 正式版发布，Global Selection 终于有了官方的正式支持 . 该功能补全了 Flutter 长时间存在 Selection 异常等问题，特别是在 Flutter Web 下经常会有选择文本时与预期的行为不匹配的情况。

![](https://pic4.zhimg.com/v2-07df55ab358ee185cc186cb58874e8b7_b.jpg)

## 使用
------

**使用 `SelectionArea` 也十分简单，如下代码所示，只需要在你想要支持的地方添加 `SelectionArea` 即可**，甚至可以在每个路由下的 `Scaffold` 添加 `SelectionArea` 来全面启用支持。

![](https://pic3.zhimg.com/v2-502d5dd5bd47707e37a554ea65a0e876_b.jpg)

默认情况下 `SelectionArea` 已经实现了所有常见的功能，并且 Flutter 针对不同平台进行了差异化实现，如下图所示 Android 和 iOS 会有不同的样式效果。

![](https://pic2.zhimg.com/v2-b4f1a5a016c27a6c83374afc126090c1_b.jpg)
![](https://pic1.zhimg.com/v2-84bf5979cdc93bdc70605f0de09f8780_b.jpg)
![](https://pic2.zhimg.com/v2-865728cc58995bec0cf394f33a5c67cd_b.jpg)

**当然，也许这时候你会发现在 iOS 上的 Toolbar 居然没有全选**，其实这是因为 iOS 使用了 `TextSelectionControls` 默认的 `canSelectAll` 判断，这个判断里有一个条件就是需要 selection 的 `start == end` 才符合条件。

![](https://pic2.zhimg.com/v2-889a46ef60a95ee0ec293249a7b15e5d_b.png)

所以如果你觉得这个判断有问题，完全可以自己 `override` 一个自定义的 `TextSelectionControls` ，比如在 `canSelectAll` 直接 `return true` 。

![](https://pic4.zhimg.com/v2-429985acd96d2b81bcec9fd80a5abf57_b.jpg)
![](https://pic4.zhimg.com/v2-fd34916ba84c757719e551cf1835832f_b.jpg)

是的，**对于 `SelectionArea` 我们可以通过继承 `TextSelectionControls` 来自定义**：

*   通过 `buildToolbar` 自定义弹出的 Toolbar 样式和逻辑，甚至你可以添加一些额外的标签能力，比如 “插入图片”
*   通过 `buildHandle` 自定义 Selection Handle 可拖动部分的样式

而在 `SelectionArea` 里，不管是 Handle 还是 Toolbar ，都是通过新增 `Overlay` 来实现样式，这部分的逻辑主要在 `SelectionOverlay` 对象：

![](https://pic1.zhimg.com/v2-61206bcc1aed5198018dfa4dde7b2298_b.jpg)

> 如果你还不了解 `Overlay` ，可以简单理解为：**默认情况下所有的路由页面都在一个 `Overlay` 下，打开一个 Route 就是添加一个 `OverlayEntry` 到 `Overlay` 里**。

所以 Handle 和 Toolbar 都是通过 `OverlayEntry` 打开的特殊“路由”控件，拥有新的层级，例如下方右图就是 Toolbar 所在的 `OverlayEntry` 。

![](https://pic3.zhimg.com/v2-df95b9b7b7bdf494192e675c3899fc16_b.jpg)
![](https://pic2.zhimg.com/v2-01856321ed5496b3757c46f9fd1ac0e5_b.jpg)

**另外，对于 Handle 的颜色定义，默认情况下主要来自 `TextSelectionTheme` 和 `Theme`** 。

例如 `MaterialTextSelectionControls` 里，start 和 end 两个 Handle 的颜色，默认是通过 `TextSelectionTheme` 的 `selectionHandleColor` 或者 `Theme` 的 `primary` 来设置。

![](https://pic3.zhimg.com/v2-ae3747c39261a5f41bae51ccf1747c26_b.jpg)

那文字的选中区域的颜色是怎么来的？难道也是 `OverlayEntry` 吗？

答案是否定的，这部分颜色主要是来自于文本绘制时 Canvas 的渲染。

如下代码所示，**当文本被绘制时，会判断当前是否有被选中的片段，如果存在选中的片段，会调用绘制对应的选中图层**。

![](https://pic4.zhimg.com/v2-b81883a0d91651f32e6e65a8ebc8f96f_b.jpg)

而对于文字的选中区块的颜色，默认是通过 `DefaultSelectionStyle` 的 `selectionColor` 来显示，当然，如下右图所示，在 `MaterialApp` 里它依然和 `TextSelectionTheme` 的 `selectionColor` 或者 `Theme` 的 `primary` 有关系。

![](https://pic1.zhimg.com/v2-717021f0ec8a651b4a6a1c040113f4f8_b.jpg)
![](https://pic2.zhimg.com/v2-d3d911895bf1502341efceb6b3a6913d_b.jpg)

**那如果你还想要在 `SelectionArea` 下的某些内容不允许被选中呢**？

这里 Flutter 提供了 `SelectionContainer.disabled` 实现，只要在对应内容嵌套 `SelectionContainer.disabled` ，那么这部分内容下的文本就无法被选中。

![](https://pic3.zhimg.com/v2-a0104601b116bb9f0bb09bc9d3b63bda_b.jpg)
![](https://pic1.zhimg.com/v2-bd73931e72d4e51826122261d779dfb4_b.jpg)

为什么嵌套 `SelectionContainer.disabled` 就可以禁用文本选中的能力？这其实和 `SelectionArea` 的实现有关系：

> `SelectionContainer` 内部实现了一个 `InheritedWidget` ，它会往下共享一个 `SelectionRegistrar` ，而默认情况下 `SelectionArea` 内部使用了 `SelectionContainer` 并且往下共享了对应的 Registrar 实现。

*   `SelectionArea` 内部的 `SelectionContainer` 是有对应的 `registrar` 实现往下共享
*   `SelectionContainer.disabled` 内部的 `registrar` 是 `null`

**所以根本区别就在于 `SelectionContainer.disabled` 里没有 `registrar`** ，如下左图所示，加了 disabled 后获取到的 `registrar` 是 null ，那么如下右侧代码所示，在后续可选中区域的更新逻辑中就会直接 return 。

![](https://pic3.zhimg.com/v2-131dd8bea7fb1f8e716143d3cbca2d66_b.jpg)
![](https://pic3.zhimg.com/v2-f21dc3d2ccd228062109cbc8639aef06_b.jpg)

到这里你应该大致理解了如何使用和自定义一些 `SelectionArea` 的能力，那么接下来介绍两个 “Bug” ，通过这两个 “Bug” 我们深入理解 `SelectionArea` 内部的实现情况 。

## 问题1
-------

如下代码所示，**当使用了 `WidgetSpan` 之后，默认情况下，用户在开始位置拖拽 Handle 进行选择时会无法选中 `WidgetSpan` 里的文本**。

![](https://pic4.zhimg.com/v2-4c2cd4bfc397923b0d827319dd25ed77_b.jpg)
![](https://pic2.zhimg.com/v2-3e6b7505df3f355904f3a1fe694f5185_b.jpg)

> PS：其实拖动可以选中，只是这里暂时以不能选中的情况下作为切入点。

为什么会这样？首先要知道，上面代码在使用了 `WidgetSpan` 包裹 `Hello World` 之后，其实是存在两个`Text` ，也就是上述的 UI 是由两个 `RenderParagraph` 绘制完成。

那么对于最外层的 `Text` ，其实它的文本内容是 `“Flutter is the best!”`，注意这段文本，其实文本里此时是多了两个空格。

![](https://pic2.zhimg.com/v2-6cd6fe0697e75f665f5052f1089c7f89_b.jpg)

之所以会有这两个空格，其实是因为 `WidgetSpan` 使用了 `0xFFFC` 的占位符，这段占位符在渲染时，就会被替换为 `WidgetSpan` 对应的 `Hello World` 和猫头图片。

![](https://pic4.zhimg.com/v2-033f0cd5a342f08653dab06a2e0ecb87_b.jpg)

**那么这时候如果我们选择复制，复制出来的内容会是 `Flutter isthe best!`** ，中间的两个占位符是不会复制出来，因为在获取可选择片段时，会把对应的 `placeholderCodeUnit` 剔除。

![](https://pic2.zhimg.com/v2-b86a2c69c7879c08e22dc36e353c93d5_b.jpg)

另外，当我们点击复制的时候， `WidgetSpan` 所在的 `Hello World` 并没有被选中，所以此时调用 `getSelectedContent` 就会得到 null ，也就是没有内容。

![](https://pic1.zhimg.com/v2-e734bfe4f638cf276bf78993b92e5378_b.jpg)

所以可以看到：此时在手动拖拽选择时，`WidgetSpan` 里的文本是不会被选中，因为它处于不同的 `Text` ，对于外层 `Text` 而言它只是个占位符。

> 当然，**其实在拖动 Handle 还是可以选中 `WidgetSpan` 里的文本，比如你从 `Hello World` 开始拖动，这里拖动选中不了的原因后面会解释**。

## 问题 2
--------

如果当我们点击了全选会怎么样？如下图所示，在我们点击全选之后，可以看到两个“奇怪”的问题：

*   `WidgetSpan` 里的 `Hello World` 可以被选中了
*   左侧的 Start Handle 位置不是在文本开头，而是在 `WidgetSpan` 开始

![](https://pic3.zhimg.com/v2-50358b015819648f43a14c447218a4aa_b.jpg)

我们首先看第一点，**为什么点击全选时，`WidgetSpan` 里的 `Hello World` 可以被选中**？

其实全选操作和拖拽 Handle 最大的不同就是：它是往下直接发出全选事件 `SelectAllSelectionEvent` ，而该事件会触发所有 child 响应事件，自然也就包括了 `WidgetSpan` 里的 `Hello World` 。

![](https://pic2.zhimg.com/v2-79c5bbb3ed96502c251368119a6384d1_b.jpg)

最后负责响应 SelectAll 事件的对象是 `_SelectableFragment` ，这里主要有两个关键逻辑：

*   `_handleSelectAll` 获取得到 `_textSelectionStart` 和 `_textSelectionEnd` ，表明此时控件已经被选中
*   `didChangeSelection` 里通过 `paragraph.markNeedsPaint()` 触发重绘，然后增加选中时的覆盖颜色

![](https://pic4.zhimg.com/v2-e5ade23a7eeb1fa0c014eae059e1358f_b.jpg)

可以看到，由于此时 `WidgetSpan` 里的 `Hello World` 也直接响应了全选事件，所以它会处于选中状态，这样之后在 `getSelectedContent` 调用里也可以获取到内容，也就是能够 `Hello World` 能被复制出来。

![](https://pic3.zhimg.com/v2-f27a3cc463a14bc9a71cc3d73c5c4d0e_b.jpg)

**但是此时复制出来的内容会是 `Hello World!Flutter isthe best!`** ，是不是感觉还不对？这就是我们要说的第二个问题，左侧的 Start Handle 位置不是在文本开头。

![](https://pic3.zhimg.com/v2-50358b015819648f43a14c447218a4aa_b.jpg)

首先我们看，为什么复制出来之后的内容会是 `Hello World!Flutter isthe best!` ？

正如前面说到的，复制调用的是 `getSelectedContent` 方法，如下代码所示，**可以看到在 `selectables` 这个 `List` 的第一位就是 `Hello World` ，所以最终拼接出来的文本会是 `Hello World!Flutter isthe best!`** 。

![](https://pic2.zhimg.com/v2-7ecfa4ea250e6346dca4a668ffc577c5_b.jpg)

那为什么 `Hello World` 会排在 `selectables` 的第一位？ 这就需要讲到 Flutter 里对 Selectable 的一个排序逻辑。

我们知道 `Text` 内部是通过 `RenderParagraph` 实现文本绘制，而 `RenderParagraph` 在初始化的时候，**如果存在 `_registrar` ，也就是存在 `SelectionArea` 的时候，就会通过 `add` 把支持选中的片段添加 `SelectionArea` 内部的 `_additions`** 里。

![](https://pic3.zhimg.com/v2-d2b0240f4bf4d242b9f776f23b356aca_b.jpg)

之后 `SelectionArea` 内部会对可选中的内容进行排序，如下代码所示，在`sort` 之前，此时的 `Hello World` 在 `_additions` 列表的最末端，因为它处于 `WidgetSpan` 的 child 里，所以是最晚被加入到 `_additions` 的。

![](https://pic2.zhimg.com/v2-e9a37b87ce96fe73d3a44defeb456bc1_b.jpg)

而在执行完 `sort`之后 ，可以看到此时 `Hello World` 跑到了列表的最前面，**这也是为什么复制出来的内容顺序是 `Hello World` 开头，然后 Start Handle 会显示在 `Hello World` 的原因**。

![](https://pic3.zhimg.com/v2-c9b82d37a6c65c3e93008719fa0afdc6_b.jpg)

`sort` 的逻辑主要是通过 `compareOrder` 实现，简单分析 `compareOrder` 的排序实现，可以看到其中有一个 `_compareVertically` 的逻辑，通过调试对比，**可以看到此时因为 `Hello World` 所处的 `Rect`（top）比其他文本高，所以它被认为是更高优先级的位置，类似于被误认为是上一行的情况**。

![](https://pic3.zhimg.com/v2-baab0512c55c2e51d84a4437336fa56a_b.jpg)

知道了问题那就很好处理了，**如下代码所示，如果此时调整一下 `WidgetSpan` 的高度，可以看到全选逻辑下 Start Handle 正常了，但是.... End Handle 位置又不对了**。

![](https://pic4.zhimg.com/v2-3ae6bc6c705e478a0869378815bd9c5b_b.jpg)
![](https://pic2.zhimg.com/v2-fc939d6e1e83ea86358fc8a7a93e15b5_b.jpg)

此时复制出来的内容会是 `Flutter isthe best!Hello World!` ，**因为这个时候会有一个很“微妙”的偏差值，导致 `Hello World` 排序时被排列到最后面**，从而导致 End Handle 不是预期的位置。

![](https://pic3.zhimg.com/v2-ffdf5afa4f85c7a6e498d1e2eb096e6e_b.png)

另外，这时候你会发现，如下左侧动图所示，**此时拖动 Handle 是可以选中 `WidgetSpan` 里的 `Hello World`** ，其实之前的情况下也可以，不过需要如右侧动图所示，需要从 `Hello World` 开始拖动，**因为最开始的情况下 `selectables` 里 `Hello World` 的排序层级更高，所以如果想要拖动选中，也需要从它开始**。

![](https://pic2.zhimg.com/v2-6b67f3dd7ac4a55f7e330c0eb441592d_b.gif)

![](https://pic1.zhimg.com/v2-361c5e7b05779ed7a96e8157c3f4667c_b.jpg)

> 目前这个问题在 master 和 stable 分支均可以复现，对应 issue 我也提交在 [#111021](https://link.zhihu.com/?target=https%3A//github.com/flutter/flutter/issues/111021) 。

## 最后
------

虽然 `SelectionArea` 的出现补全了 Flutter 的长久以来的短板之一，不过基于 `SelectionArea` 实现的复杂程度，目前 `SelectionArea` 还有不少的细节需要优化，但是万事开头难，本次 3.3 `SelectionArea` 的落地也算是一个不错的开始。

最后，相信通过本文大家应该对 `SelectionArea` 的使用和实现都有了一定的了解，如果你还有什么问题，欢迎留言评论交流～