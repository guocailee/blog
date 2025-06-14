---
{"dg-publish":true,"permalink":"/Program/FrontEnd/深入解析Flutter下一代渲染引擎Impeller - 知乎/","noteIcon":"","created":"2025-03-06T21:28:25.975+08:00"}
---


> 魏国梁：字节 _Flutter_ _Infra_ 工程师， _Flutter Member_，长期专注 _Flutter_ 引擎技术  
> 袁 欣：字节 _Flutter_ _Infra_ 工程师， 长期关注渲染技术发展  
> 谢昊辰：字节 _Flutter_ _Infra_ 工程师，_Impeller Contributor_

**Impeller项目启动背景**
------------------

_2022_ 年 _6_ 月在 _Flutter_ _3.0_ 版本中 _Google_ 官方正式将渲染器 _Impeller_ 从独立仓库中合入 _Flutter Engine_ 主干进行迭代，这是 _2021_ 年 _Flutter_ 团队推动重新实现 _Flutter_ 渲染后端以来，首次正式明确了 _Impeller_ 未来代替 _Skia_ 作为 _Flutter_ 主渲染方案的定位。 _Impeller_ 的出现是 _Flutter_ 团队用以彻底解决 _SkSL_（_Skia Shading Language_） 引入的 _Jank_ 问题所做的重要尝试。官方首次注意到 _Flutter_ 的 _Jank_ 问题是在 _2015_ 年，当时推出的最重要的优化是对 _Dart_ 代码使用 _AOT_ 编译优化执行效率。 在 _Impeller_出现之前，_Flutter_ 对渲染性能的优化大多停留在 _Skia_ 上层，如渲染线程优先级的提升，在着色器编译过久的情况下切换 _CPU_ 绘制等策略性优化。

_Jank_ 类型分为两种：首次运行卡顿(_Early-onset Jank)_和非首次运行卡顿，_Early-onset_ _Jank_ 的本质是运行时着色器的编译行为阻塞了 _Flutter_ _Raster_ 线程对渲染指令的提交。在 _Native_ 应用中，开发者通常会基于 _UIkit_ 等系统级别的 _UI_ 框架开发应用，极少需要自定义着色器，_Core Animation_ 等 _framework_ 使用的着色器在 _OS_ 启动阶段就可以完成编译，着色器编译产物对所有的 _app_ 而言全局共享，所以 _Native_ 应用极少出现着色器编译引起的性能问题_，_更常见的是用户逻辑对 _UI_ 线程过度占用_。_官方为了优化 _Early-onset Jank_ ，推出了_SkSL_ 的 _Warmup_ 方案，_Warmup_ 本质是将部分性能敏感的 _SkSL_ 生成时间前置到编译期，仍然需要在运行时将 _SkSL_ 转换为 _MSL_ 才能在 _GPU_ 上执行。_Warmup_ 方案需要在开发期间在真实设备上捕获 _SkSL_ 导出配置文件_，_在应用打包时通过编译参数可以将部分 _SkSL_ 预置在应用中。此外由于 _SkSL_ 创建过程中捕获了用户设备特定的参数，不同设备 _Warmup_ 配置文件不能相互通用，这种方案带来的性能提升非常有限。

在 _2019_ 年 _Apple_ 宣布在其生态中废弃 _OpenGL_ 后， _Flutter_ 迅速完成了渲染层对 _Metal_ 的适配。 与预期不符的是， _Metal_ 的切换使得 _Early-onset_ _Jank_ 的情况更加恶化，_Warmup_ 方案的实现需要依赖 _Skia_ 团队对 _Metal_ 的预编译做支持，由于 S_kia_ 团队的排期问题，一度导致 _Warmup_ 方案在 _Metal_ 后端上不可用。与此同时社区中对 _iOS_ 平台 _Jank_ 问题的反馈更加强烈，社区中一度出现屏蔽 _Metal_ 的 _Flutter Engine Build_，回退到 _GL_ 后端虽然能一定程度改善首帧性能但是在 _iOS_ 平台上会出现视觉效果的退化，与之相对的是，由于 _Android_ 平台上拥有 _iOS_ 缺失的着色器机器码的缓存能力， _Android_ 平台出现 _Jank_ 的概率比 _iOS_ 低很多。

除了社区中出现的通用问题外，_Flutter_ _infra_ 团队也经常收到字节内部业务方遇到的 _Jank_ 问题的反馈，反馈较集中的有转场动画首次卡顿、列表滚动过程中随机卡顿等场景：

![](https://pic3.zhimg.com/80/v2-4d39b8f94ab0bcedbd997ad5ad890546_720w.jpg)

转场动画触发的着色器编译，耗时～100ms

![](https://pic4.zhimg.com/80/v2-ebcf25da860d685804bb712e87936903_720w.jpg)

转场动画触发的着色器编译，耗时～100ms

在这篇文章中，我们尝试从 _Metal_ 着色器编译方案，矢量渲染器原理和 _Flutter_ _Engine_ 渲染层的接口设计三个维度去探究 _Impeller_ 想要解决的问题和渲染器背后的相关技术。

**Metal Shader Compilation演进**
------------------------------

一般而言，不同的渲染后端会使用独立的着色器语言，与 _JavaScript_ 等常见脚本语言的执行过程类似，不同语言编写的着色器程序为了能在 _GPU_ 硬件上执行，需要经历完整的 _lexical analysis / syntax analysis_ / _Abstrat_ _Syntax Tree_ （抽象语法树，下文简称 _AST_）构建，_IR_ 优化，_binary generation_ 的过程。着色器的编译处理是在厂商提供的驱动中实现，其中具体的实现对上层开发者并不可见。 _Mesa_ 是一个在 _MIT_ 许可证下开源的三维计算机图形库，以开源形式实现了 _OpenGL_ 的 _api_ 接口。通过 _Mesa_ 中对 _GLSL_ 的处理可以观察到完整的着色器处理流水线。 如下图所示，上层提供的 _GLSL_ 源文件被 _Mesa_ 处理为 _AST_ 后首先会被编译为 _GLSL IR_， 这是一种 _High-Level IR_，经过优化后会生成另一种 _Low-Level IR_ ：_NIR_，_NIR_ 结合当前 _GPU_ 的硬件信息被处理为真正的可执行文件。不同的 _IR_ 用来执行不同粒度的优化操作，通常底层 _IR_ 更面向可执行文件的生成，而上层 _IR_ 可以进行诸如 _dead code elimination_ 等粗粒度优化。常见的高级语言（如 _Swift_ ）的编译过程也存在 _High-Level IR_ (_Swift_ _IL_) 到 _Low-Level IR_ (_LLVM_ _IR_)的转换。

![](https://pic4.zhimg.com/80/v2-e82109c11583ee62f0bc2440f31dfc6f_720w.jpg)

随着 _Vulkan_ 的发展， _OpenGL_ _4.6_ 标准中引入了对 _SPIR-V_ 格式的支持。_SPIR-V_（_Standard Portable Intermediate Representation_）是一种标准化的 _IR_，统一了图形着色器语言与并行计算（_GPGPU_ 应用）领域。它允许不同的着色器语言转化为标准化的中间表示，以便优化或转化为其他高级语言，或直接传给_Vulkan_、_OpenGL_ 或 _OpenCL_ 驱动执行。_SPIR-V_ 消除了设备驱动程序中对高级语言前端编译器的需求，大大降低了驱动程序的复杂性，使广泛的语言和框架前端能够在不同的硬件架构上运行。_Mesa_ 中使用 _SPIR-V_ 格式的着色器程序可以在编译时直接对接到 _NIR_ 层，缩短着色器机器码编译的开销， 有助于系统渲染性能的提升。

![](https://pic3.zhimg.com/80/v2-d212c1b8277a64cb2f9f2b649aacf2e2_720w.jpg)

在 _Metal_ 应用中， 使用 _Metal Shading Language_（以下简称 _MSL_ ）编写的着色器源码首先被处理为 _AIR (Apple_ _IR\*\*)_ 格式的中间表示。如果着色器源码是以字符形式在工程中引用，这一步会在运行时在用户设备上进行，如果着色器被添加为工程的Target，着色器源码会在编译期在 _Xcode_ 中跟随项目构建生成 _MetalLib_: 一种设计用来存放 _AIR_ 的容器格式。随后 _AIR_ 会在运行时，根据当前设备 _GPU_ 的硬件信息，被 _Metal Compiler Service_ 用 _JIT_ 编译为可供执行的机器码。相比源码形式，将着色器源码打包为 _MetalLib_ 有助于降低运行时生着色器机器码的开销。 着色器机器码的编译会在每一次渲染管线状态对象（_**P**ipeline_ _**S**tate_ _**O**bject_，下文简称 _PSO_）创建时发生，一个 _PSO_ 持有当前渲染管线关联的所有状态，包含光栅化各阶段的着色器机器码，颜色混合状态，深度信息，模版掩码状态，多重采样信息等等。_PSO_ 通常被设计为一个 _imutable object_（不可变对象），如果需要更改 _PSO_ 中的状态需要创建一个新的 _PSO_ 拷贝。

![](https://pic4.zhimg.com/80/v2-262932747adfa68fc39f83fda06a1ccb_720w.jpg)

由于 _PSO_ 可能在应用生命周期中多次创建， 为了防止着色器的重复编译开销，所有编译过的着色器机器码会被 _Metal_ 缓存用来加速后续 _PSO_ 的创建过程，这个缓存称为 _Metal Shader Cache_ ，完全由 _Metal_ 内部管理，不受开发者控制。应用通常会在启动阶段一次性创建大量 _PSO_ 对象，由于此时 _Metal_ 中没有任何着色器的编译缓存，_PSO_ 的创建会触发所有的着色器完整执行从 _AIR_ 到机器码的编译过程，整个集中编译阶段是一个 _CPU_ 密集型操作。在游戏中通常在玩家进入新关卡前利用 _Loading Screen_ 准备好下一场景所需的 _PSO_，然而常规 _app_ 中用户的预期是能够即点即用，一旦着色器编译时间超过 _16_ _ms_，用户就会感受到明显的卡顿和掉帧。

![](https://pic4.zhimg.com/80/v2-4635e9926a31e8d867f0e065dc496def_720w.jpg)

在 _Metal 2_ 中， _Apple_ 首次为开发者引入了手动控制着色器缓存的能力：_Metal Binary Archive_。_Metal Binary Archive_ 的缓存层次位于 _Metal Shader Cache 之上_， 这意味着 _Metal Binary Archive_ 中的缓存在 _PSO_ 创建时会被优先使用_。_在**运行时**，开发者可以通过 _Metal_ _Pipeline_ _Manager_ 手动将性能敏感的着色器函数添加至 _Metal Binary Archive_ 对象中并序列化至磁盘中。应用再次冷启后，此时创建相同的 _PSO_ 即是一个轻量化操作，没有任何着色器编译开销。缓存的 _Binary Archive_ 甚至可以二次分发给相同设备的用户，如果本地 _Binary Archive_ 中缓存的机器码与当前设备的硬件信息不匹配，_Metal_ 会回落至完整的编译流水线，确保应用的正常执行。游戏堡垒之夜「_Fortnite_」 在启动阶段需要创建多达 1700 个 _PSO_ 对象，通过使用 _Metal Binary Archive_ 来加速 _PSO_ 创建，启动耗时从 _**1m26s**_ 优化为 _**3s**_，速度提升_**28倍**_。

_Metal Binary Archive_ 通过内存映射的方式供 _GPU_ 直接访问文件系统中的着色器缓存，因此打开 _Metal Binary Archive_ 时会占用设备宝贵的虚拟内存地址空间。与缓存所有的着色器函数相比，更明智的做法是根据具体的业务场景将缓存分层，在页面退出后及时关闭对应的缓存_，_释放不必要的虚拟内存空间。_Metal Shader Cache_ 的黑盒管理机制无法保证着色器在使用时不会出现二次编译_，_而 _Metal Binary Archive_ 可以确保其中的缓存的着色器函数在应用生命周期内始终可用。_Metal Binary Archive_ 虽然允许开发者手动管理着色器缓存，却依然需要通过在运行时搜集机器码来构建，**无法保证应用初次安装时的使用体验**。在 _2022_ 年 _WWDC_ 中，_Metal 3_ 终于弥补了这个遗留的缺陷，为开发者带来了在离线构建 _Metal Binary Archive_ 的能力：

![](https://pic2.zhimg.com/80/v2-abc9ac86bca6cf277ac9b277d9c8ac0d_720w.jpg)

构建离线 _Metal Binary Archive_ 需要使用一种全新的配置文件 _Pipeline_ _Scrip_t，_Pipeline Script_ 其实是 _Pipeline State Descriptor_ 的一种 _JSON_ 表示，其中配置了 _PSO_ 创建所需的各种状态信息，开发者可以直接编辑生成，也可以在运行时捕获 _PSO_ 获得。给定 _Pipeline Script_ 和 _MetalLib_，通过 _Metal_ 工具链提供的 _metal_ 命令即可离线构建出包含着色器机器码的 _Metal Binary Archive_。_Metal Binary Archive_ 中的机器码可能会包含多种 _GPU_ 架构_，_由于 _Metal Binary Archive_ 需要内置在应用中提交市场_，_开发者可以综合考虑包体积的因素剔除不必要的架构支持。

![](https://pic4.zhimg.com/80/v2-d567537c25ed676b08a13ebc614e6a93_720w.jpg)

通过离线构建 _Metal Binary Archive_，着色器编译的开销只存在于编译阶段，应用启动阶段 _PSO_ 的创建开销大大降低。_Metal Binary Archive_ 不止可以优化应用的首屏性能， 真实的业务场景下，一些 _PSO_ 对象会迟滞到具体页面才会被创建，触发新的着色器编译流程。一旦编译耗时过长，就会影响当前 _RunLoop_ 下 _Metal_ 绘制指令的提交， _Metal Binary Archive_ 可以确保在应用的生命周期内， 核心交互路径下的着色器缓存始终为可用状态，将节省的 _CPU_ 时间片用来处理与用户交互强相关的逻辑， 大大提升应用的响应性和使用体验。

**矢量渲染基础概念**
------------

矢量渲染泛指在平面坐标系内通过组装几何图元来生成图像信息的手段，通过定义一套完整的绘制指令，可以在不同的终端上还原出不失真的图形， 任何前端的视窗都可以被看作一个 _2D_ 平面的矢量渲染画布，_Chrome_ 与 _Android_ 渲染系统就是基于 _Google_ 的 _2D_ 图形库 _Skia_ 构建。对应用开发而言，矢量渲染技术也扮演重要角色，如文本 / 图表 / 地图 / _SVG_ / _Lottie_ 等都依赖矢量渲染能力来提供高品质的视觉效果。

![](https://pic1.zhimg.com/80/v2-9769147c0dc8ad3d470e938333b45950_720w.jpg)

矢量渲染的基础单元是 _Path_（路径），_Path_ 可以包含单个或多个 _Contour_（轮廓），_Contour在_一些渲染器中也称为 _SubPath_，_Contour_ 由连续的 _Segment_（直线/高阶贝塞尔曲线）组成，标准的几何构型(圆形/矩形)均可被视为一种特殊的 _Path_，一些特殊的 _Path_ 可以包含坑洞或者自交叉（如五角星⭐️），这类 _Path_ 的处理需要一些特殊的方案。围绕 _Path_ 可以构造出各种复杂的图形，著名的老虎 _SVG_一共包含480条 _Path_ ，通过对其中不同 _Path_ 的描边和填充，可以呈现出极富表现力的视觉效果：

![](https://pic4.zhimg.com/80/v2-14f7ec8e6fc9cc9a4e1e059af72ca3ef_720w.jpg)

高阶贝塞尔通过起始点和额外的控制点来定义一条曲线， 在将这样的抽象曲线交付给后端进行渲染前，我们需要首先要对贝塞尔曲线做插值来近似模拟这条曲线，这个操作通常称为 _Flatten_ ，_GPU_真实渲染的是一由组离散的点来近似模拟的曲线。 根据 _Path_ 定义的差异， 这一组离散的点会构成不同种类的多边形，对 _Path_ 的处理简化为了对多边形的处理，我们以一个简单的凹多边形为例来了解 _Path_ 的描边和填充操作是如何实现的：

![](https://pic1.zhimg.com/80/v2-632fd888bf34cef26abd326fca120a84_720w.jpg)
![](https://pic3.zhimg.com/80/v2-c1e259f3d4a0b74dcb867ee909f33bee_720w.jpg)

多边形的描边操作，由于描边宽度的存在，描边的真实着色区域会有一半落在 _Path_ 定义的区域之外。遍历多边形的外边缘的每条边，根据每条边两侧的顶点，描边宽度以及边缘的斜率可以组装出一组模拟描边行为的三角形图元，如上图所示：一个方向上的描边是由两个相结合的三角形构成。 针对不同的 _Line Join_ 风格，结合处有可能需要做不同的处理， 但是原理类似。将描边的三角形提交 _GPU_ 可以渲染得到正确的描边效果，除了纯色的描边，结合不同的着色器可以实现渐变和纹理的填充效果。

多边形的填充方法相比描边更加复杂，目前主流的矢量渲染器有两种不同的实现思路：

### **基于模版掩码的填充（NanoVG）**

基于模版掩码的填充是在 _OpenGL_ 红宝书中所描述的一种填充多边形的经典方法。_Skia_ 在简单的场景下也会使用这种方法做多边形的填充。 这种绘制方法分为两步：首先利用 _StencilBuffer_ 来记录实际绘制区域，这一步只写入 _StencilBuffer_，不操作 _Color Attachment_，然后再进行一次绘制，通过_StencilBuffer_ 记录的模版掩码，只向特定的像素位置写入颜色信息。通过图例可以更直观的了解这个过程：第一步，打开 _StencilBuffer_ 的写入开关，使用 _GL\_TRIANGLE\_FAN_ 形式绘制所有的顶点， _GL_会自动根据顶点索引组装两组三角形基元 _0 -> 1 -> 2_ 和 _0 -> 2 -> 3_，_GL_ 中通常指定逆时针方向为三角形片元的正面， 0 -> 1 -> 2 三角形所包围的区域在 _StencilBuffer_ 中做 +1 操作， 由于顶点3是多边形的凹点， _0 -> 2 -> 3_ 三角形的环绕数被翻转为了顺时针，我们可以在 _StencilBuffer_ 中对顺时针包裹的区域做 -1 操作， 此时 _StencilBuffer_ 中所有标记为 1 的像素就是我们所需要的绘制区域，再次提交相同的顶点进行绘制，打开颜色写入，就可以得到正确的绘制结果。这种方法巧妙的利用了凹多边形会改变局部三角形环绕方向的特性。

![](https://pic1.zhimg.com/80/v2-fd18364c6cffeab88b91db7e95855850_720w.jpg)

模版掩码可以正确处理复杂的多边形， 但是由于需要进行两段式的绘制， 对于复杂的多边形性能绘制性能瓶颈较明显， 此外 _StencilBuffer_ 等操作都是由 _GL_ 驱动层所实现，几乎不可能进行任何的性能优化， 这种绘制方法常在一些追求小尺寸的矢量渲染器中使用（_NanoVG_）， 在一些文章中通常也被称为 _Stencil & Cover_ 。

### **基于三角剖分的填充（Skia）**

_Skia_ 中对多边形的渲染是由 _Tesselation_ 和 _Triangulation_ 两步构成，_Tesselation_ 原意指在多边形中**新增顶点**来构造更加细分的几何图元，_Triangulation_ 是指连接多边形自身的顶点构造可以填充满自身的若干三角图元（**不增加顶点的情况下）**， _Triangulation_ 可以认为是 _Tessellation_ 的一种特例，在 _Skia 中描_述的 _Tessellation_ 其实是指一种对复杂多边形的拆分操作，了解多边形的 _Triangulation_ 首先我们需要引入单调多边形的概念：

![](https://pic2.zhimg.com/80/v2-a56a177a742720f689c834edf267f505_720w.jpg)

对于任意一个多边形 _p_ 而言， 如果存在一条直线 _l_， _l_ 的垂线与 _p_ 相交的部分都在 _p_ 的内部， 那么称多边形 _p_ 是相对于 _l_ 的单调多边形。 单调多边形的**单调性是相对于某一特定方向**而言，针对上图的示例我们可以很容易找到一个方向的直线作为反例。 利用单调多边形在 _l_ 方向上的左右两个极点可以把多边形进一步分拆为上下两条边，每条边上的顶点在 _l_ 方向上会确保是有序的，这个特性可以用来实现剖分算法。

以下图中的凹多边形为例子，复杂多边形的完整处理思路是：**首先使用** **_Tesselation_** **算法将其拆分为若干个单调多边形**（下图中两个蓝色区域），通常会在多边形的凹点进行拆分，**得到一组单调多边形的集合后，** **再分别对每一个单调多边形进行三角化**，单调多边形的 _Triangulation_ 算法比较著名的有 _[EarCut](https://link.zhihu.com/?target=https%3A//github.com/mapbox/earcut)_， 也有一些实现如 _libtess2_ 可以同时对复杂多边形进行 _Tesselation_ / _Triangulation_ 两步操作， _libtess2_ 使用 _Delaunay_ 算法来对单调多边形实现剖分， _Delaunay_ 算法可以避免剖分出现过于狭长的三角形。 无论使用何种方案，最终的产物都是能够直接交付给 _GPU_ 进行渲染的三角形 _Mesh_ 集合。

![](https://pic1.zhimg.com/80/v2-8092e97c1dca3c94234c71d4c51e43f4_720w.jpg)
![](https://pic1.zhimg.com/80/v2-d4aa40b76b924fbed8cf196d404cd5dc_720w.jpg)

针对上文中的凹多边形， 剖分后的产物会是如上图所示的两个三角形， 三角形可以被认为是一种最简单的单调多边形， 提交这两个三角形即可实现此凹多边形的正确填充。基于三角剖分的填充方案， 最大的瓶颈是**拆分单调多边形**和**单调多边形三角化**两个步骤的的算法选择， 由于这两步完全由上层实现， 因此对后期优化更加友好， 目前业界最新的方案已经可以实现利用 _GPU_ 或者深度学习的方法实现剖分的加速。

**Flutter DisplayList**
-----------------------

_DisplayList_ 出现之前，_Skia_ 使用 _SkPicture_ 来搜集每一帧的绘制指令，随后在 _Raster_ 线程回放完成当前帧的绘制。_gl_ 函数在进入 _GPU_ 执行前，仍然会有一部分逻辑如 _PSO_ 状态检测 / 指令封装等操作在 _CPU_ 上执行，录制回放能力可以避免绘制操作占用宝贵的主线程时间片。 _DisplayList_ 和 _SkPicture_ 的作用类似，那么为什么还需要将 _SkPicture_ 向 _DisplayList_ 做迁移 ？_Skia_ 对 _Flutter_ 来说属于第三方依赖，涉及到 _SkPicture_ 的优化一般需要由 _Skia_ 团队支持，对 _Skia_ 团队而言 S_kPicture_ 的能力不只服务于 _Flutter_ 业务，_Flutter_ 团队如果修改 _SkPicture_ 的源码会对 _Skia_ 的代码有比较大的入侵， 而为了解决长期遗留的 _Jank_ 问题， _Flutter_ 团队又不得不考虑在 _SkPicture_ 这一层进行优化_。2020_ 年 _3_ 月，_[liyuqian](https://link.zhihu.com/?target=https%3A//github.com/liyuqian)_ _在_创建一个 _[flutter issue](https://link.zhihu.com/?target=https%3A//github.com/flutter/flutter/issues/53501)_ 中首次提出了 _DisplayList_ 的设想，预期相较于 _SkPicture_ 会有如下三个方面的优势：

*   _DisplayList_ 相比 _SkPicture_ 有更高的可操作性去优化光栅化时期产生的缓存；
*   _DisplayList_ 有助于实现更好的着色器预热方案；
*   _DisplayList_ 相比 _SkPicture_可以更好的对每一帧进行性能分析；

在 _Flutter_ _RoadMap_ 明确了 _Impeller_ 的替换目标后，_DisplayList_ 能更好的实现 _Flutter Engine_ 层对渲染器的解耦，从而保障后续渲染层能无缝的从 _Skia_ 迁移到 _Impeller_ 中。在最新的 _Flutter 3.0_ 代码， _DisplayList_ 相关的代码位于 _[https://github.com/flutter/engine/tree/main/display\_list](https://link.zhihu.com/?target=https%3A//github.com/flutter/engine/tree/main/display_list)_ 中，

_DisplayList_ 作为 _Recoder_ 的过程和使用 _SkPicture_ 差别不大，核心是在 _[http://canvas.cc](https://link.zhihu.com/?target=http%3A//canvas.cc)_ 中进行了切换：

```cpp
// https://github.com/flutter/engine/blob/main/lib/ui/painting/canvas.cc#L260
// lib/ui/painting/canvas.cc
void Canvas::drawRect(double left,
                      double top,
                      double right,
                      double bottom,
                      const Paint& paint,
                      const PaintData& paint_data) {
  if (display_list_recorder_) {
    paint.sync_to(builder(), kDrawRectFlags);
    builder()->drawRect(SkRect::MakeLTRB(left, top, right, bottom));
  }
  // 3.0 因为默认开启了 DisplayList 作为 Recorder 所以下面的已经删除 
  // else if (canvas_) {
  //  SkPaint sk_paint;
  //  canvas_->drawRect(SkRect::MakeLTRB(left, top, right, bottom),
  //                  *paint.paint(sk_paint));
  // }
}
​
// lib/ui/painting/canvas.h
DisplayListBuilder* builder() {
  return display_list_recorder_->builder().get();
}

```

从上面的代码可以看出，是在 _Canvas_ 的 _DrawOp_ 中进行了 _DisplayList_ 还是 _SkPicture_ 的选择，一次_DrawOp_ 的录制过程如下图所示：

![](https://pic1.zhimg.com/80/v2-2d8edbb385e9d49ecc026847e2c1bb5c_720w.jpg)

DisplayList Record DrawOp 过程

图中 _Push<DrawRectOp>_ 的操作，_DrawRectOp_ 定义在 _display\_list\_ops.h_ 中：

```cpp
// https://github.com/flutter/engine/blob/main/display_list/display_list_ops.h#L554
// display_list/display_list_ops.h
[[define]] DEFINE_DRAW_1ARG_OP(op_name, arg_type, arg_name)                  \
  struct Draw##op_name##Op final : DLOp {                                 \
    static const auto kType = DisplayListOpType::kDraw##op_name;          \
                                                                          \
    explicit Draw##op_name##Op(arg_type arg_name) : arg_name(arg_name) {} \
                                                                          \
    const arg_type arg_name;                                              \
                                                                          \
    void dispatch(Dispatcher& dispatcher) const {                         \
      dispatcher.draw##op_name(arg_name);                                 \
    }                                                                     \
  };
DEFINE_DRAW_1ARG_OP(Rect, SkRect, rect)
DEFINE_DRAW_1ARG_OP(Oval, SkRect, oval)
DEFINE_DRAW_1ARG_OP(RRect, SkRRect, rrect)
[[undef]] DEFINE_DRAW_1ARG_OP

```

将宏定义展开可以看到如下定义， 这里 _DrawRectOp_ 是一种单参数 _DLOp_， _DrawRectOp_ 中的 _dispatch_ 方法会将 _drawRect_ 操作派发给 _dispatcher_ 来实际执行_。_

```cpp
struct DrawRectOp final ：DLOp {
    static const auto kType = DisplayListOpType::kDrawRect;
    explicit DrawRectOp(arg_type arg_name) : rect(rect) {}
    const SkRect rect;  
    void dispatch(Dispatcher& dispatcher) const {                         
      dispatcher.drawRect(arg_name);                                 
    } 
}

```

在 _LLDB_ 中可以打印出 _DrawRectOp_ 的相关信息:

![](https://pic4.zhimg.com/80/v2-eca42cda8cb8ccb261b2f3c57a546e37_720w.jpg)

_Push<DrawRectOp>_ 中的_Push_ 函数的实现如下，_storage\__ 是一个一维数组，同来存储 _DrawOp_，在添加元素前会先进行容量的判断，是否需要扩容，随后创建 _DrawRectOp_ 并对 _Type_ 和 参数 _rect_ 进行赋值，并累加 _op\_count\__，完成 _DrawOp_ 的添加。

```cpp
// https://github.com/flutter/engine/blob/main/display_list/display_list_builder.cc#L27
// display_list/display_list_builder.cc
void* DisplayListBuilder::Push(size_t pod, int op_inc, Args&&... args) {
  size_t size = SkAlignPtr(sizeof(T) + pod);
  // 扩容
  if (used_ + size > allocated_) {
    // Next greater multiple of DL_BUILDER_PAGE.
    allocated_ = (used_ + size + DL_BUILDER_PAGE) & ~(DL_BUILDER_PAGE - 1);
    storage_.realloc(allocated_);
    FML_DCHECK(storage_.get());
    memset(storage_.get() + used_, 0, allocated_ - used_);
  }
  FML_DCHECK(used_ + size <= allocated_);
  // 如 new DrawRectOp
  auto op = reinterpret_cast<T*>(storage_.get() + used_);
  used_ += size;
  new (op) T{std::forward<Args>(args)...};
  op->type = T::kType;
  op->size = size;
  op_count_ += op_inc;
  return op + 1;
}

```

_DisplayList_ 记录 _DrawOp_ 的流程如下：

*   首先通过调用 _BeginRecording_ 创建 _DisplayListCanvasRecoder_ (继承自 _SkCanvasNoDraw_) 之后创建核心类 _DisplayListBuilder_ 并返回 _Canvas_ 给应用层；
*   应用层通过 _Canvas_ 调用如 _drawRect_ 方法，将会被以 _DrawRectOp_ 记录在 _DisplayListBuilder_ 的 _storage\__ 中；
*   最后调用 _endRecording_ 将 _DisplayListBuilder_ 的 _storage\__ 转移到 _DisplayList_ 中，后面在 _SceneBuilder_ 阶段，_DisplayList_ 会被封装到 _DisplayListLayer_ 中；

_DisplayList_ 中的几个核心概念：_DisplayListCanvasRecorder_ 作为命令记录的载体，其中包含了 _DisplayListBuilder_。_DisplayListBuilder_ 的 _storage_ 是真实记录 _DLOp_ 的载体，_DisplayList_ 将会记录 _DisplayListBuilder_ 的 _storage_，并最终被包裹在 _DisplayListLayer_ 中，作为记录 _DLOp_ 的载体。_DisplayListCanvasDispatcher_ 作为最后派发至 _SkCanvas_ 或者 _Impeller_ 的 _Wrapper_ 层。

**Impeller 渲染流程和架构设计**
----------------------

### **Impeller 概览**

![](https://pic3.zhimg.com/80/v2-013a38fba688fd07ae86517e7e6890b2_720w.jpg)

_Impeller_ 的目标是为 _Flutter_ 提供具备 _predictable performance_ 的渲染支持，_Skia_ 的渲染机制需要应用在启动过程中动态生成 _SkSL，_这一部分着色器需要在运行时转换为 _MSL_，才能进一步被编译为可执行的机器码，整个编译过程会对 _Raster_ 线程形成阻塞。_Impeller_ 放弃了使用 _SkSL_ 转而使用 _GLSL_ _4.6_ 作为上层的着色器语言，通过 _Impeller_ 内置的 _ImpellerC_ 编译器，在编译期即可将所有的着色器转换为 _Metal Shading language_， 并使用 _MetalLib_ 格式打包为 _AIR_ 字节码内置在应用中。_Impeller_ 的另一个优势是大量使用 _Modern Graphics APIs_ ，_Metal_ 的设计可以充分利用 _CPU_ 多核优势并行提交渲染指令， 大幅减少了驱动层对 _PSO_ 的状态校验， 相对于 _GL_ 后端仅仅将上层渲染接口的调用切换为 _Metal_ 就可以为应用带来约 _～10%_ 的渲染性能提升。

在一个 _Flutter_ 应用中，_RenderObject_ 的 _Paint_ 操作最终会转换为 _Canvas_ 的 _draw options_，绘制操作在 _Engine_ 层组装成 _DisplayList_ 之后通过 _DisplayListDispatcher_ 分发到不同的渲染器来执行具体的渲染操作。_Impeller_ 中实现了_DisplayListDispatcher_ 接口，这意味着 _Impeller_ 可以消费上层传递的 _DisplayList_ 数据。_Aiks_ 层维护了 _Canvas，Paint_ 等绘制对象的句柄。_Entity_ 可以理解为 _Impeller_ 中的一个原子绘制行为，如 _drawRect_ 操作，其中保存了执行一次绘制所有的状态信息，_Canvas_ 会通过 _Entity_ 中保存的状态设置画布的 _Transform，BlendMode_ 等属性。 _Entity_ 中最关键的组成部分是 _Contents_。_Contents_ 中持有了着色器的编译产物， 被用来实际控制当前 _Entity_ 的绘制效果， _Contents_ 有多种子类，来承接填充/纹理着色等不同的绘制任务。_Renderer_ 层可以理解为与具体渲染 _api_ 沟通的桥梁，_Renderer_ 会将 _Entity_ 中的信息（包含_Contents_ 中保存的着色器句柄）转换为 _Metal / OpenGL_ 等渲染后端的具体 _api_ 调用。

### **_Impeller_ 绘制流程**

![](https://pic2.zhimg.com/80/v2-085d6c614e5d7cd5d03f4487a9b579c9_720w.jpg)

_Flutter_ _Engine_ 层的 _LayerTree_ 在被 _Impeller_ 绘制前需要首先被转换为 _EntityPassTree_ ，_UI_ 线程在接收到 _v-sync_ 信号后会将 _LayerTree_ 从_UI_ 线程提交到 _Raster_ 线程，在 _Raster_ 线程中会遍历 _LayerTree_ 的每个节点并通过 _DisplayListRecorder_ 记录各个节点的绘制信息以及 _saveLayer_ 操作， _LayerTree_ 中可以做可以 _Raster Cache_ 的子树其绘制结果会被缓存为位图， _DisplayListRecorder_ 会将对应子树的绘制操作转换为 _drawImage_ 操作，加速后续渲染速度。 _DisplayListRecorder_ 完成指令录制后，就可以提交当前帧。 _DisplayListRecorder 中_的指令缓存会被用来创建 _DisplayList 对象_，_DisplayList_ 被_DisplayListDispatcher_ 的实现者（_Skia / Impeller_）消费，回放 _DisplayList_ 其中所有的 _DisplayListOptions_ 可以将绘制操作转换为 _EntityPassTree。_

![](https://pic3.zhimg.com/80/v2-3719f8cf770d77ad3c652b6fb57355aa_720w.jpg)

完成 _EntityPassTree_ 的构建之后，需要把 _EntityPassTree_ 中的指令解析出来执行。_EntityPassTree_ 绘制操作以 _Entity_ 对象为单位，_Impeller_ 中使用 _Vector_ 来管理一个绘制上下文中多个不同的 _Entity_ 对象。 通常 _Canvas_ 在执行复杂绘制操作时会使用 _SaveLayer_ 开辟一个新的绘制上下文，在 _iOS_ 上习惯称为离屏渲染， _SaveLayer_ 操作在 _Impeller_ 中会被标记为创建一个新的 _EntityPass_，用于记录独立上下文中的 _Entity_，新的 _EntityPass_ 会被记录到父节点的 _EntityPass_ 列表中， _EntityPass_ 的创建流程如上图所示。

![](https://pic1.zhimg.com/80/v2-c21114308fdd424e056e32e087f3c44c_720w.jpg)

_Metal_ 在上层为设备的 _GPU_ 硬件抽象了 _CommandQueue_ 的概念，_CommandQueue_ 与 _GPU 数量_一一对应，_CommandQueue_ 中可包含一个或者多个 _CommandBuffer_。_CommandBuffer_ 是实际绘制指令 _RenderCommand_ 存放的队列，简单的应用可以只包含一个 _CommandBuffer，_不同的线程可以通过持有不同_CommandBuffer_ 来加速 _RenderCommand_ 的提交。_RenderCommand_ 由 _RenderCommandEncoder 的 Encode_ 操作产生，_RenderCommandEncoder_ 定义了此次绘制结果的保存方式_，_绘制结果的像素格式以及绘制开始或结束时 _Framebuffer attachmement_ 所需要做的操作(_clear / store)，RenderCommand_ 包含了最终交付给 _Metal_ 的真实 _drawcall_ 操作。

_Entity_ 中的 _Command_ 转化为真正的 _MTLRenderCommand 时，_还携带了一个重要的信息：_PSO 。Entity_ 从 _DisplayList_ 中继承的绘制状态最终会变为 _MTLRenderCommand_ 关联的 _PSO ，MTLRenderCommand_ 被消费时 _Metal_ 驱动层会首先读取 _PSO_ 调整渲染管线状态，再执行着色器进行绘制，完成当前的绘制操作_。_

### **ImpellerC 编译器设计**

![](https://pic4.zhimg.com/80/v2-4907034503cc325cb4f99a5a1be77a8f_720w.jpg)

_ImpellerC_ 是 _Impeller_ 内置的着色器编译解决方案，源码位于 _Impeller_ 的 _compiler_ 目录下 ，它能够在编译期将 _Impeller_ 上层编写的 _glsl_ 源文件转化为两个产物：_1._ 目标平台对应的着色器文件；_2._ 根据着色器 _uniform_ 信息生成的反射文件，其中包含了着色器 _uniform_ 的 _struct_ 布局等信息。反射文件中的 _struct_ 类型作为 _model_ 层，使得上层使用无需关心具体后端的 _uniform_ 赋值方式，极大地增强了 _Impeller_ 的跨平台属性，为编写不同平台的着色器代码提供了便利。

在编译 _Flutter_ _Engine_ 工程中 _Impeller_ 部分时，_gn_ 会首先将 _compile_r 目录下的文件编译出为 _ImpellerC_ 可执行文件，再使用 _ImpellerC_ 对 _entity_/_content_/_shaders_ 目录下的所有着色器进行预处理。_GL_ 后端会将着色器源码处理为 _hex_ 格式并整合到一个头文件中， 而 _Metal_ 后端会在 _GLSL_ 完成 _MSL_ 的转译后进一步处理为 _MetalLib。_

_ImpellerC_ 在处理 _glsl_ 源文件时，会调用 _shaderc_ 对 _glsl_ 文件进行编译。s_haderc_是 _Google_ 维护的着色器编译器，可以 _glsl_ 源码编译为 _SPIR-V_。_shaderc_ 的编译过程使用了 _glslang 和_ _[SPIRV-Tools](https://link.zhihu.com/?target=https%3A//github.com/KhronosGroup/SPIRV-Tools)_ 两个开源工具： _glslang_ 是 _glsl_ 的编译前端_，_负责将 _glsl_ 处理为 _AST_ ， _[SPIRV-Tools](https://link.zhihu.com/?target=https%3A//github.com/KhronosGroup/SPIRV-Tools)_ 可以接管剩下的工作将 _AST_ 进一步编译为 _SPIR-V，_在这一步的编译过程中，为了能得到正确的反射信息，_ImpellerC_ 会对 _shaderc_ 限制优化等级。

随后 _ImpellerC_ 会调用 _SPIR-V Cross_ 对上一步骤得到的 _SPIR-V_ 进行反汇编，得到 _SPIR-V_ _IR ，_这是一种 _SPIR-V Cross_ 内部使用的数据结构，_SPIR-V Cross_ 会在其之上进行进一步优化。_ImpellerC_ 随后会调用 _SPIR-V Cross_ 创建目标平台的 _Compiler_ _Backend_（_MSLCompiler_ / _GLSLCompiler_ / _SKSLCompiler_）， _Compiler Backend_ 中封装了目标平台着色器语言的具体转译逻辑 。同时 _SPIR-V Cross_ 会从 _SPIR-V IR_ 中提取 _Uniform_ 数量，变量类型和偏移值等反射信息，

```cpp
struct ShaderStructMemberMetadata {
  ShaderType type; // the data type (bool, int, float, etc.)
  std::string name; // the uniform member name "frame_info.mvp"
  size_t offset;
  size_t size;
};

```

_Reflector_ 在得到这些信息后，会对内置的 _.h_ 与 _.cc_ 模版进行填充，得到可供 _Impeller_ 引用的 _.h_ 与_.cc_ 文件，上层可以反射文件的类型方便的生成数据 _memcpy_ 到对应的 _buffer_ 中实现与着色器的通讯。对于_Metal_ 和 _GLES3_ 来说，由于原生支持 _UBO_，最终会通过对应后端提供的 _UBO_ 接口来实现 传值，对于不支持 _UBO_ 的 _GLES2_ 来说，对 _UBO_ 的赋值需要转换为 _glUniform\*_ 系列 _api_ 对 _Uniform_ 中每个字段的单独赋值，在 _shader program link_ 后，_Impeller_ 在运行时通过 _glGetUniformLocation_ 得到所有字段在 _buffer_ 中的位置，与反射文件中提取出的偏移值结合，_Impeller_ 就可以得到每个 _Uniform_ 字段的位置信息，这个过程会在 _Imepller Context_ 创建时生成一次，随后 _Impeller_ 会维护 _Uniform_ 字段的信息。对于上层来说，不管是 _GLES2_ 还是其他后端， 通过 _Reflector_ 与着色器的通讯过程都是一样的。

完成着色器转译和反射文件提取后，就可以实际执行 _uniform_ 数据的绑定，_Entity_ 在触发绘制操作时会首先调用 _Content_ 的 _Render_ 函数， 其中会创建一个供 _Metal_ 消费的 _Command_ 对象，_Command_ 会提交到 _RenderPass_ 中等待调度， _uniform_ 数据的绑定发生在 _Command 创建这一步_。 如下图所示： _VS::FrameInfo_ 和 _FS::GradientInfo_ 是反射生成的两个 _Struct_ 类型， 初始化 _VS::FrameInfo_ 和 _FS::GradientInfo_ 的实例并赋值后，通过 _VS::BindFrameInfo_ 和 _FS::BindGradientInfo_ 函数即可实现数据和 _uniform_ 的绑定。

```cpp
VS::FrameInfo frame_info;
frame_info.mvp = Matrix::MakeOrthographic(pass.GetRenderTargetSize()) * entity.GetTransformation();
​
FS::GradientInfo gradient_info;
gradient_info.start_point = start_point_;
gradient_info.end_point = end_point_;
gradient_info.start_color = colors_[0].Premultiply();
gradient_info.end_color = colors_[1].Premultiply();
​
Command cmd;
cmd.label = "LinearGradientFill";
cmd.pipeline = renderer.GetGradientFillPipeline(OptionsFromPassAndEntity(pass, entity));
cmd.stencil_reference = entity.GetStencilDepth();
cmd.BindVertices(vertices_builder.CreateVertexBuffer(pass.GetTransientsBuffer()));
cmd.primitive_type = PrimitiveType::kTriangle;
FS::BindGradientInfo(cmd, pass.GetTransientsBuffer().EmplaceUniform(gradient_info));
VS::BindFrameInfo(cmd, pass.GetTransientsBuffer().EmplaceUniform(frame_info));
return pass.AddCommand(std::move(cmd));

```

LinearGradientContents Render函数实现

_Impeller_ 完整的着色器处理流水线如下图所示：

![](https://pic3.zhimg.com/80/v2-8bf821f595dc4e3ec19d216d96e72f16_720w.jpg)

**总结**
------

_Impeller_ 是 _Flutter_ 为了治理 _SkSL_ 编译耗时引入的的性能问题所做的重要尝试，_Skia_ 的渲染机制需要在运行时动态创建 _SkSL，_导致着色器编译的时间后移， _Impeller_ 通过在编译期完成 _GLSL_ 至 _MSL_ 的转换，在 _iOS_ 平台上可以直接使用 _MetalLib_ 构建着色器机器码，并且引入确定性的缓存策略来提升渲染性能表现。随着今年 _WWDC_ 中 _Apple_ 补齐了离线构建 _Metal Binary Archive_ 的能力， _Metal 3_ 已经具备了全场景下高性能渲染的能力。 _Impeller_ 作为 _Flutter_ 独占的渲染方案_，_ 没有 _Skia_ 的历史负担_，_ 更容易充分利用 _Apple_ 的技术优化，这意味着 _Impeller_ 的性能表现还有进一步提升的可能。

_Impeller_ 目前使用了基于 _libtess2_ 的三角剖分方案， 根据社区的 _RoadMap_，_Impeller_ 还会继续探索 _GPU_ 剖分等高阶的三角化方案用来替换陈旧的 _libtess2_ 实现。_Impeller_ 总体是一个移动优先的渲染解决方案，目前已经具备 _GL_ 和 _Metal_ 两个完整的渲染后端实现_，_ _Vulkan_ 的支持目前正在进行中，官方目前没有支持 _CPU_ 软绘的计划。_Impeller_ 短期内不会也没有可能作为 _Skia_ 的替代品， 不过其优秀的架构设计使其依然有潜力剥离出 _Flutter_ 成为一个独立的渲染解决方案， 未来可能会对基于 _Skia_ 的自绘方案形成挑战， 我们对 _Impeller_ 后续的发展也会持续保持关注。