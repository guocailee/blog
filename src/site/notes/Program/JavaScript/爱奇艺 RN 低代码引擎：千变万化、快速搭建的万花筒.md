---
{"dg-publish":true,"permalink":"/program/java-script/rn/","noteIcon":""}
---



React Native 是具有高动态化能力的跨平台开发框架，低代码是近几年非常热门的趋势，而爱奇艺早在 2018 年就尝试将二者结合，开发出了万花筒引擎和专题页低代码平台，用于快速搭建爱奇艺 App 中的各类内容展示型页面，赋能不会写代码的内容运营和编辑们，针对热门的综艺和影视剧内容，用很少的人力投入就能快速搭建出具有丰富排版和动画效果的专题页面，并直接投放上线。

在 2021 年 5 月北京 QCon 全球软件开发大会上，爱奇艺基线研发部高级经理甘泉分享了爱奇艺设计和应用万花筒引擎和低代码平台的实践经验、遇到的挑战以及解决思路。我们整理了他的演讲，以期解决您在 React Native 低代码系统架构设计时可能遭遇的某些问题。（下文以甘泉老师第一人称叙述）

在 6 月北京 ArchSummit 架构师峰会上，也策划了低代码平台设计专题，邀请了便利蜂、途虎养车、Treelab 等企业的专家来分享各自实践，感兴趣的可以点击底部阅读原文查看议题内容。

爱奇艺专题页面包含丰富排版和动画效果。这类页面均由以下低代码方案实现：**用零代码搭建页面、用低代码开发页面内常规组件实现常规动画、用专业代码开发页面内定制化组件实现个性动画。** 

业务背景

移动端常规页面开发流程包括：提出需求、需求审核、视觉设计、前后端开发、测试验证、修复 Bug、跟版上线七步，涉及到产品经理、设计师、工程师、QA 等角色。

![](https://mmbiz.qpic.cn/mmbiz_png/XIibZ0YbvibkVdGldcbdwiaRl0lII0TCmqPibicw9oWIrwFLXjDCU1PUuuaFU4SaNeiaCfP5Oqr2D6lIM9b9DKtUKCfQ/640?wx_fmt=png)

专题页开发流程与一般页面在流程、人员、成本上、发布周期、向前兼容性上均存在差异。专题页开发只有提出需求、视觉设计、搭建页面、自测验证、投放上线五步。其中，需求是由编辑提出的。

![](https://mmbiz.qpic.cn/mmbiz_png/XIibZ0YbvibkVdGldcbdwiaRl0lII0TCmqPWAoicrMkpGxXUVibOy3vuTd12k17FRlo0wjxhWiaOgnXKOIHrxhSoTL8w/640?wx_fmt=png)

在专题页开发中，**开发流程需极度简化才能方便编辑操作，页面需花样丰富才能吸引用户，搭建要足够快捷才可保证内容时效性，增加新布局样式和特性需不依赖 App 发版才可便于更新。** 

为实现以上需求，我们对比了 H5、RN 和当时爱奇艺已有的自研的原生 DSL 方案的优缺点，综合成本、性能、动态性等多方面的考虑，决定自研一个结合 RN 和 DSL 的新方案：Kaleidoscope 引擎（万花筒引擎）。

![](https://mmbiz.qpic.cn/mmbiz_png/XIibZ0YbvibkVdGldcbdwiaRl0lII0TCmqPYsY5T99rlSuRicvlUu818OIkXkEF75nJPIfyXjTVdgl4ZlXyUOh1N6w/640?wx_fmt=png)

Kaleidoscope DSL 设计思想

这个引擎之所以被命名为 Kaleidoscope（万花筒），是因为其灵感来自真正的万花筒。万花筒是在圆筒一端放入彩色碎片，圆筒中间放置三棱镜，从另一端的孔中即可观测到对称的美丽图像，结构简单、体积小巧，却千变万化。Kaleidoscope 采取了相通的设计思路。

![](https://mmbiz.qpic.cn/mmbiz_png/XIibZ0YbvibkVdGldcbdwiaRl0lII0TCmqP7vaf2CBibVtIMmHiaSeFSZ0ACJgap9uerlk8qiaYRibPr7jKJficzXfN20w/640?wx_fmt=png)

**Kaleidoscope 层次结构分为四层，App、Page、Item、Element。** 最外层为 App 层，一个 App 中包含一个 Page，Page 中包含一组 Item 组件（Item 可理解为一个长列表中的 Item），每个 Item 由多个 Element 构成。而 Element 就相当于万花筒中的彩色碎片。

Element 分为多种类型，**容器型允许互相嵌套，可包含子元素，可实现用简单布局（如横纵布局）组合出复杂的布局；元件型为最小元素，无子元素，不可再分；复合型可实现特殊布局和交互。** 

Kaleidoscope 数据格式采用了 JSON 格式，体积小，易生产，很通用；样式布局描述系统采用了 Flexbox，RN 支持良好，布局能力强，学习成本低，其数据结构如下图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/XIibZ0YbvibkVdGldcbdwiaRl0lII0TCmqPlXA02bS25PreCrblxHtzkaZYe2lAbianQ6SqGsYpYEyiaXmddg22DO6Q/640?wx_fmt=png)

图中左侧为一个 JSON 结构，其中 Page 对象最重要的属性为 Items 数组，**单个 Item 由多个 contents 嵌套组成（图中右侧），再加上容器型 Element 中的 content，即可实现千变万化的布局。** 

DSL 贯穿了 Kaleidoscope 方案，和 RN 堪称天生一对：

-   **JSON 结构和 RN 组件一一对应；**
-   **JSON Property 和 RN Prop 也一一对应；**
-   **RN 提倡组合大于继承，适用 DSL 场景。** 

通过 RN 组件的设计和封装，即可用一个简洁的架构实现分而治之。

整体方案的架构设计

Kaleidoscope 整体架构图分为前端、后端和低代码平台三部分。前端面向内容消费者，后端和低代码平台面向内容生产者以及部分开发人员。

![](https://mmbiz.qpic.cn/mmbiz_png/XIibZ0YbvibkVdGldcbdwiaRl0lII0TCmqP5bShPib1wY28aVcR4UOQLAbsVBsoXvnicN3HADr3DqJtH0EEZEuPbibpQ/640?wx_fmt=png)

前端架构中 Kaleidoscope 引擎处于下面架构图的第二层，主要负责数据的下载、缓存和解析，依赖于 QYRN 框架。QYRN 框架是爱奇艺以 RN 官方框架为基础封装的框架，它丰富了官方框架的组件库，扩展了 UI 组件和原生模块，实现了热更新机制、数据投递和 Bridge 实例复用。

![](https://mmbiz.qpic.cn/mmbiz_png/XIibZ0YbvibkVdGldcbdwiaRl0lII0TCmqPO4iaad0vFKXUic23LKmLD4jSfl2dxyyKcWloSzRtF3DRbnJnNkVAyciaQ/640?wx_fmt=png)

后端架构如下所示。页面数据来自于编辑们搭建页面的配置信息和内容原数据的数据流（虚线下的流程）。两类数据一静一动，会在不同端的 Worker Service 上融合，返回给不同用户端。

![](https://mmbiz.qpic.cn/mmbiz_png/XIibZ0YbvibkVdGldcbdwiaRl0lII0TCmqPjJNbtdjLIbUq1dQjx72jjqOl3icly6cLWReWBUJuVly6mkiaw6Tt2GTg/640?wx_fmt=png)

**低代码平台架构的核心为第三层 Portal，由两块组成：面向组件开发同学的组件开发平台和编辑使用的页面可视化搭建平台。** 页面可视化搭建平台实现完全零代码，组件开发平台只需要用少量代码就可以开发组件，通常给外包人员使用。

![](https://mmbiz.qpic.cn/mmbiz_png/XIibZ0YbvibkVdGldcbdwiaRl0lII0TCmqPp8S61BSVDFZrumOEeHZXzicz0FcK9FTrquic8zHItr3pKgUdjzDnlhng/640?wx_fmt=png)

万花筒引擎的实现思路

RN 的动态性一直是其不容小觑的优势。爱奇艺原生的 DSL 引擎也具有一定的动态性，可迅速实现界面更新、UI 改版等。**基于 RN 和 DSL 强强结合，万花筒引擎实现了引擎的高动态性。** 

同时，**爱奇艺基于可嵌套容器实现了高布局灵活性。** 可嵌套容器为前述容器型 Element，此处不赘述了。我会重点介绍爱奇艺是如何**基于 JS Card 实现高业务适应性、基于层叠实现低数据冗余、基于依赖注入实现高扩展性的。** 

基于 JS Card 实现了高业务适应性

参考 HTML 中通过夹带啦啦啦啦标签编写 JS 实现动态效果，**为提高业务适应性，我们在 JSON 中也添加了 JS 脚本**。针对无法用 JSON 描述的非通用需求场景，我们允许以 Card（列表项）为粒度直接用 RN 开发，打包成 JS 字符串之后和其他 JSON 数据一起下发，动态加载运行。Card 为前文中的 Item 被渲染后的带界面的占有一定高度的实体。

JS Card 工作分为六步：

![](https://mmbiz.qpic.cn/mmbiz_png/XIibZ0YbvibkVdGldcbdwiaRl0lII0TCmqPVOx0ic9oVhibdecibCDibCQ2XSXZnOnIo17bMZZ3eVQfrCzWJ3sfSupmicQ/640?wx_fmt=png)

1.  开发 JS Card。开发 JS Card 和开发 RN 组件唯一差别是需要将组件类存入 global 对象。
2.  打包 JS 字符串。打包时需要注意双引号转义和包体积大小；
3.  发布 JS Card。在低代码平台的组件开发平台创建一个 JS Card 组件，进行发布；
4.  接口返回页面数据。发布后，端上请求接口即可获取 JS 字符串，放入 DSL 配置对象的 base.script 属性中，允许一个页面中添加多个 JS Card；
5.  解析执行 JS 注册。引擎解析数据后，使用 eval 函数执行 JS 字符串，将字符串中定义的组件的 class 加到上下文中，从 global 中获取对象并添加到 JS Card 注册表中进行管理；
6.  渲染。如 Item 有 ScriptID 属性，则从 JS Card 注册表中找到对应组件进行渲染。

基于层叠实现低数据冗余

为了减少页面中的冗余信息，爱奇艺基于 CSS 的层叠样式表提出了层叠的概念，**支持了信息复用和组合**，针对样式数据提出了 Style 和 Theme，针对 pingback 数据提出了 Statistics 和 CommonStats。可在 Style 中引用 Theme，在 Statistics 引用 CommonStats，且支持按先后顺序覆盖合成。

举例来说，Style 可通过 ref 属性引用 Theme 中定义好的类（如下图左），引用了多个 Theme 可通过样式继承和覆盖机制合成具体的 Style（如下图右）。

![](https://mmbiz.qpic.cn/mmbiz_png/XIibZ0YbvibkVdGldcbdwiaRl0lII0TCmqPwZUYHRS1GG7qx4yVxlA8wPtFoLWxCAGiaxeSSU6sX8VvCjWw2JUdVxQ/640?wx_fmt=png)

基于依赖注入实现高扩展性

不同业务方前来对接，但是需求不同时，爱奇艺采用了**剥离核心功能，支持业务方按需裁剪、扩展和定制的思路**。我们采用依赖注入的方式实现和使用所有的 Page、Item 和 Element 组件，打破了组件之间的强依赖。业务方可自由组合复用已有的组件，自定义组件，实现过程如下图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/XIibZ0YbvibkVdGldcbdwiaRl0lII0TCmqPWsiat0MzSyrjyRjAJuibKtutrVj6IYCBtBl49WsgeBnsaCZ2vN25icF0g/640?wx_fmt=png)

1.  配置核心抽象类的具体依赖：这里核心抽象类可分为 5 种：Pages、Services、Items、Elements、Components；
2.  创建业务配置类：向框架注入 5 种核心抽象类实例配置；
3.  引入业务配置类：在代码入口注册 App 组件；
4.  引用举例：在 App 组件中解耦引用注入的 listPage 组件。

低代码平台设计与实现

下图为爱奇艺编辑日常使用的页面搭建系统：

![](https://mmbiz.qpic.cn/mmbiz_png/XIibZ0YbvibkVdGldcbdwiaRl0lII0TCmqP8YHhEVgsIqAG0OcPuYmQj3ofBKgRbggGOhcYzq1KuNNm1Rk2Wx0udw/640?wx_fmt=png)

左侧为组件库，中间为预览区；右侧为配置区。**组件均为 Card 粒度**，鼠标拖动即可选中，在配置区即可配置 Card 的数据和样式。

该平台扩展需要使用下图中的组件开发平台添加组件、排版和样式。

![](https://mmbiz.qpic.cn/mmbiz_png/XIibZ0YbvibkVdGldcbdwiaRl0lII0TCmqPHKgCuVz2XBSBNslr6RkByAFQI8mmpVOawc7eojOoLwrC8rCe078iaQw/640?wx_fmt=png)

制作组件需写一个描述该组件 UI 形式的 JSON 模板。该 JSON 只有内容占位符，具体内容需**使用 Web IDE** 针对组件编写一个 JS 函数来填充。**该函数有四个输入：组件 JSON 模板内容、编辑搭建页面时绑定的数据源对应的内容数据和填写的标题等配置信息、手机屏幕分辨率等需要参考的扩展信息，返回一个输出：描述一个 Item 的 JSON 对象**。这个 JS 函数跑在基于 Nashorn 的 JS 引擎上，外包稍加培训便可编写。

![](https://mmbiz.qpic.cn/mmbiz_png/XIibZ0YbvibkVdGldcbdwiaRl0lII0TCmqPL1lvHe0yDVpjns92nkRsB8Hn0cujn5kVP7l8vtRTRuicStZNQuFQ7Ug/640?wx_fmt=png)

该平台面向多业务开放。业务申请开通后，开发后端的数据源即可利用爱奇艺现有组件搭建页面，也可用低代码平台开发组件。

性能优化

万花筒引擎的性能优化实践

我们在引擎上实现了懒加载（Inline Requires），当需使用模块时再加载其信息，提升了首屏加载性能。在 Android 端使用了 Hermes 引擎，性能优于 JSC。

针对引擎瘦身，我们实现了**分包和精简依赖**，减少了 Bundle 体积和首屏加载耗时。爱奇艺众多业务共享一个引擎，使用自研打包工具可将包含 RN 基础功能的代码剥离出去（大约 684KB），打出只有业务代码的 Bundle。精简 NPM 依赖库，可拆出不常用的组件，解除不必要的依赖。

![](https://mmbiz.qpic.cn/mmbiz_png/XIibZ0YbvibkVdGldcbdwiaRl0lII0TCmqP0kOsFA6AibcT6Hctom6fDhbtwk2xMmkXaagII1F9bM2qdFicvMQZPqjA/640?wx_fmt=png)

JS Card 方面，我们在业务分包后再次分包，用自研的 JS Card 专用打包工具进行打包，将 JS Card 中万花筒引擎自身已经包含的依赖库在打包的时候剔除。经过层层分包爱奇艺已上线的 JS Card 如下图所示，体积明显下降。

![](https://mmbiz.qpic.cn/mmbiz_png/XIibZ0YbvibkVdGldcbdwiaRl0lII0TCmqPw9k3psbkpDzCPFiay8eiab0YUdibmicKficGU0ibu4Bs9CUw98Uja9Sqgefg/640?wx_fmt=png)

开发、测试、调试、上线相关工具生态建设

在日常开发中，我们开发了一些工具，解决 DevOps 中的各种问题。

前端开发测试工具

**我们开发了注册制二维码生成器，生成 RN 专题页跳转二维码。扫码后即可预览，启动 RN 的 JS 远程调试工具后，即可调试引擎或 JS Card**。注册制是爱奇艺统一的参数化路由机制，如下图所示，在页面中设置专题页参数，用爱奇艺 App 扫描生成的二维码，即可打开对应的专题页。

![](https://mmbiz.qpic.cn/mmbiz_png/XIibZ0YbvibkVdGldcbdwiaRl0lII0TCmqPmQHmavaT3GnZooW8VsmbiacdnibUUJMiaXJAmGN4DEl1YgWIb57hZfydg/640?wx_fmt=png)

JS Card 质量监控

业务方开发的 JS Card 代码问题不好定位。我们实现了 JS Card 运行时异常的捕获和投递，建设了统一的 JS Card 打包服务。利用该打包服务，可获取所有 source map 文件，可在爱奇艺的 APM 系统中还原崩溃栈中被混淆的 JS 代码符号，效果如下所示：

![](https://mmbiz.qpic.cn/mmbiz_png/XIibZ0YbvibkVdGldcbdwiaRl0lII0TCmqPEIGyeGetmfn1tHZYiaxMHm6SpSOdeYJ7VNQJWAG5x9LaKkPJZmlas9A/640?wx_fmt=png)

端上捕获投递的原始 JS 崩溃栈（图中左侧），没有详细信息。还原后（图中右侧）路径文件名详细，便于问题诊断。

组件生态

内容上我们建设了模板中心和组件市场。编辑搭建页面时，挑选模板中心中较合适的模板稍加修改即可。

![](https://mmbiz.qpic.cn/mmbiz_png/XIibZ0YbvibkVdGldcbdwiaRl0lII0TCmqPWXBnDjyk99JuzSciaFTIauqDAEdLkQnNTFLQZPdVNKbicx0gUOaibNfsQ/640?wx_fmt=png)

组件市场上业务方可贡献自己的组件，复用已有的组件，共同建设组件生态。

![](https://mmbiz.qpic.cn/mmbiz_png/XIibZ0YbvibkVdGldcbdwiaRl0lII0TCmqPjGicsFgMCnPCefZCQ8icRicAd38KKcevoNicLamuKTsrVU380jOkhCRbDg/640?wx_fmt=png)

上线效果和经验总结

Kaleidoscope 方案先后在爱奇艺、随刻、爱奇艺票务、爱奇艺小说共 4 款 App 上落地，在 GPhone、iPhone、iPad 三端总 UV 峰值过亿，主要应用于首页顶导航特色专题、一般专题、播单、会员权益等 4 个场景，这些场景各有特点：

-   在首页顶导航特色专题页上，实现了两个 RN 专题页之间切换效果；导航栏背景色可随着切换页面而变化，为用户带来沉浸式体验；
-   在会员顶导航专题页上，大量定制化 JS Card 组件实现了众多个性化需求，例如：吸顶导航组件，可以滚动页面到指定的 Item；
-   在播单上，实现了播放器根据页面滚动位置自动开播，一个视频播放完毕后自动滚动到下一个视频自动开播等高级交互特性；
-   在一般专题上，无需引擎迭代，即可快速搭建、上线页面。

从 2018 年 12 月正式全量上线，Kaleidoscope 稳定运行至今 2 年半左右，在 GPhone、iPhone、iPad 三端总共上线过 1297 个专题页，平均 1 天 1.4 个，最近 30 天爱奇艺 Phone 端上线了 63 个页面，平均搭建时长 19.25 分钟。

在这个过程中 **RN+DSL+JS Card 方案满足了各类需求，且几乎不需要跟版；得益于 RN 跨平台特点和简洁灵活的架构设计，万花筒引擎开发和维护成本极低；引擎每次变更都经过代码评审，保证了线上持久稳定运行**。

同时，我们发现 **JS Card 是一把双刃剑，由于开发权外放，要注意性能，并进行治理；DSL 和引擎不断迭代会带来一些前后端兼容性问题，需要引入较为复杂的版本管理机制；另外，新老框架过渡替换也会带来的一系列问题，我们通过多轮灰度进行规避。** 

嘉宾介绍

甘泉，2016 年底加入爱奇艺，先后从事过 RN 框架开发，RN 框架、Flutter 框架、小程序框架、WebView 容器优化和开发团队管理，目前担任 App 中台团队负责人。在跨平台开发框架优化和应用方面具有丰富经验，在 App 中台建设方面也有一些心得。



