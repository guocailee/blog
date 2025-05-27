---
{"dg-publish":true,"permalink":"/Program/FrontEnd/为什么前端很多工具都不使用js构建/","noteIcon":"","created":"2024-05-22T16:17:54.151+08:00"}
---

翻译自《Rust Is The Future of JavaScript Infrastructure》，网址：[Rust Is The Future of JavaScript Infrastructure – Lee Robinson](https://link.zhihu.com/?target=https%3A//leerob.io/blog/rust)


Rust 是一个快速、可靠、节约内存的编程语言。在过去六年的 stackoverflow 的最受喜爱的编程语言（[2016](https://link.zhihu.com/?target=https%3A//insights.stackoverflow.com/survey/2016%23technology-most-loved-dreaded-and-wanted)，[2017](https://link.zhihu.com/?target=https%3A//insights.stackoverflow.com/survey/2017%23most-loved-dreaded-and-wanted)，[2018](https://link.zhihu.com/?target=https%3A//insights.stackoverflow.com/survey/2018%23technology-_-most-loved-dreaded-and-wanted-languages)，[2019](https://link.zhihu.com/?target=https%3A//insights.stackoverflow.com/survey/2019%23technology-_-most-loved-dreaded-and-wanted-languages)，[2020](https://link.zhihu.com/?target=https%3A//insights.stackoverflow.com/survey/2020%23most-loved-dreaded-and-wanted)，[2021](https://link.zhihu.com/?target=https%3A//insights.stackoverflow.com/survey/2021/%23technology-most-loved-dreaded-and-wanted)）中，连续得到榜首的位置。Mozilla 创造了 Rust，Facebook、Apple、Amazon、Microsoft 和 Google 都使用 Rust 去开发系统基础设施、加密、虚拟化以及其他的层级较低的软件。

为什么 Rust 开始替换 Javascript 的 Web 生态系统的重要组成部分包括压缩（Terser）、编译（Babel）、格式化（Prettier）、打包（webpack）、代码检查（ESLint）、以及更多其他的库？

Rust 是什么？
---------

Rust 帮助开发者开发节约内存的快速软件。它是同样关注代码安全、简洁语法的 C++ 或者 C 的一种现代的替换选择。

Rust 和 JavaScript 非常不一样。JavaScript 会去找到不用的变量和对象，然后自动清除它们。这个机制叫做垃圾回收（[Garbage Collection](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/Garbage_collection_%28computer_science%29)）。Rust 则希望开发者自己去规划手动的内存管理。

通过使用 Rust，开发者对于内存可以有更大的控制权，使用上则比 C++ 或者 Go 要少一些痛苦。

> Rust 使用了一种非常独特的内存管理手段，其核心理念基于 “所有权”（owernership）。基本上，Rust 会追踪“谁”在读和写具体的一块内存。Rust 知道一块内存是谁在使用，也知道这块内存是否已经没人使用，这时可以立即释放这块内存。Rust 在编译时强制内存规则，这个机制让内存 bug 几乎不会存在于运行时（runtime）。你不需要手动追踪内存，编译器会帮你处理好。  
> \--[Discord](https://link.zhihu.com/?target=https%3A//blog.discord.com/why-discord-is-switching-from-go-to-rust-a190bbca2b1f)

Adoption
--------

除了在上面提到过的使用 Rust 的公司，Rust 也被很多流行的开源库使用：

*   [Firecracker](https://link.zhihu.com/?target=https%3A//github.com/firecracker-microvm/firecracker) (AWS)
*   [Bottlerocket](https://link.zhihu.com/?target=https%3A//github.com/bottlerocket-os/bottlerocket) (AWS)
*   [Quiche](https://link.zhihu.com/?target=https%3A//github.com/cloudflare/quiche) (Cloudflare)
*   [Neqo](https://link.zhihu.com/?target=https%3A//github.com/mozilla/neqo) (Mozilla)

> Rust 让我们力量倍增，使用 Rust 是我们做的最好的决策之一。除了性能，Rust 关注正确性的工程哲学帮助我们更好的掌控复杂度。我们可以使用 Rust 的类型系统编写复杂的变量，然后让 Rust 的编译器帮我们做检查。  
> \--[Dropbox](https://link.zhihu.com/?target=https%3A//dropbox.tech/infrastructure/rewriting-the-heart-of-our-sync-engine)

从 JavaScript 到 Rust
-------------------

JavaScript 是最广泛应用的编程语言，在每一台有浏览器的设备上运行着。在过去十年，围绕着 JavaScript 建立了一个非常巨大的生态系统。

*   **Webpack:** 帮助开发者把多个 JavaScript 文件打包成一个。
*   **Babel:** 帮助开发者使用现代 JavaScript 语法编写支持老式浏览器的程序。
*   **Terser:** 帮助开发者压缩生成的代码（用以节省流量和混淆）。
*   **Prettier:** 帮助开发者统一格式化所有的代码。
*   **ESLint:** 帮助开发者在编写代码时自动找出潜在的问题。

像这样的库还有很多，成千上万的代码以及更多的被修复的 Bug，都成为了当今 Web 应用程序得以顺利运行的基石。这个工具都使用 JavaScript 或者 TypeScript 编写。这个机制运行的是 OK 的，但是我们遇到了 JS 的优化峰值。新一代工具使用了更高效的设计，影响着以前的工具。

SWC
---

[SWC](https://link.zhihu.com/?target=http%3A//swc.rs/)，创建于 2017 年，目标是提供一个可扩展的基于 Rust 的致力于制造下一代快速开发工具的平台。这个工具被一些知名项目使用，比如 Next.js、Parcel 和 Deno，还有一些公司也在使用他，包括 Vercel, ByteDance, Tencent, Shopify。

SWC 可以用来编译、压缩、打包，还可以使用插件提供的功能。你可以用这些功能来做代码变换。上面提到的高级工具，比如 Next.js 也在跑着这些变换。

Deno
----

[Deno](https://link.zhihu.com/?target=https%3A//deno.land/)，创建于 2018 年，是一个简单、现代、安全的 JavaScript 和 TypeScript 的运行时。Deno 使用了 [V8](https://link.zhihu.com/?target=https%3A//v8.dev/) 以及使用 Rust 开发。Deno 的创始人是 Node.js 的创始人，它企图替换 Node.js。Deno 在 2020 的五月发布了 [1.0](https://link.zhihu.com/?target=https%3A//deno.com/blog/v1) 版本。

Deno 的代码审查器、代码格式器、文档生成器都使用 [SWC](https://link.zhihu.com/?target=https%3A//twitter.com/devongovett/status/1369033422002389000) 建造。

esbuild
-------

[esbuild](https://link.zhihu.com/?target=https%3A//esbuild.github.io/),创建于 2020 年 1 月，它是一个用 go 编写的 JavaScript 打包器，比现存的其他工具快 10-100 倍。

> 我在尝试建造一个建造工具：A）在特定的使用场景（打包 JavaScript、TypeScript、也许还有 CSS）使用良好 B）重置开发者社区对于 JavaScript 建造工具速度的期望值。我们现在的工具实在太慢了。  
> \--Evan， _esbuild 作者 ([Source](https://link.zhihu.com/?target=https%3A//news.ycombinator.com/item%3Fid%3D22336334))_

使用系统编程语言、例如 Go 或者 Rust，来建造 JavaScript 的工具，在 esbuild 发布时，还是比较稀少的。个人观点，esbuild 启发了非常一大批工具开发者使用系统编程语言，让这些工具变得更快。Evan 选择使用 go：

> _用 Rust 编写的话，如果给予一定的努力，应该可以运行的也挺好。但是在一个更高的层级，Go 使用起来更有舒服。这个项目对我来说只是一个副业项目，所以它必须有趣。_  
> _– Evan, esbuild 作者([Source](https://link.zhihu.com/?target=https%3A//news.ycombinator.com/item%3Fid%3D22336284))_

有一些人认为，Rust 可以表现的更好，但是两者都可以实现 Evan 的目标，以及去影响社区：

> _即便使用非常基本的优化，Rust 也能比 Go 优化最好的表现要好。用 Rust 写高性能程序比研究非常高深 Go 要简单的多。_  
> _–[Discord](https://link.zhihu.com/?target=https%3A//blog.discord.com/why-discord-is-switching-from-go-to-rust-a190bbca2b1f)_

Rome
----
[Rome](https://link.zhihu.com/?target=https%3A//rome.tools/blog/2020/08/08/introducing-rome), 创建于 2020 年 8 月，是一个可以用来针对 JavaScript、TypeScript、HTML、JSON、Markdown 和 CSS 的代码检查器、编译器、打包器、测试器、以及更多的功能。Rome 的愿景是替换和统一前端开发的工具链。它的作者叫[Sebastian](https://link.zhihu.com/?target=https%3A//twitter.com/sebmck)，他也开发了 Babel。

为什么要开发所有东西？

> 为了让 Babel 可以作为其他工具的底层基石，要做一些必要的改变，这些改变基本上要完全调整 Babel 的架构。Babel 的架构是我在 2014 年正在学习解析器、语法树、编译器的时候做的。  
> _\- Sebastian ([Source](https://link.zhihu.com/?target=https%3A//rome.tools/blog/2020/08/08/introducing-rome))_

Rome 使用 TypeScript 编写，可以跑在 Node.js 上，但是现在正在用 [Rust](https://link.zhihu.com/?target=https%3A//twitter.com/rometools/status/1422616144763097091) 重写。

Rust + WebAssembly
------------------

[WebAssembly](https://link.zhihu.com/?target=https%3A//webassembly.org/docs/use-cases/)_(WASM)_是一种 Rust 可以编译到的便携低级语言。WASM 跑在浏览器里，可以和 JavaScript 互相调用，几乎所有现代浏览器都支持。

> WASM 比 JS 快的多，但是没有原生开发速度快。我们测试来看，Parcel 使用 WASM 比原生开发慢 10-20 倍。  
> _–[Devon Govett](https://link.zhihu.com/?target=https%3A//twitter.com/devongovett)_

WASM 还不是目前最完美的方案，但是依然可以帮助开发者提供更快的体验。Rust 官方已经承诺更高质量的 [WASM](https://link.zhihu.com/?target=https%3A//www.rust-lang.org/what/wasm) 实现。对于开发者来说，意味着你可以享受到 Rust 的性能（相比于 Go），即便你仍然编译到 Web 上（使用 WASM）。

以下是目前比较早期的库和框架：

*   [Yew](https://link.zhihu.com/?target=https%3A//yew.rs/)
*   [Percy](https://link.zhihu.com/?target=https%3A//github.com/chinedufn/percy)
*   [Seed](https://link.zhihu.com/?target=https%3A//github.com/seed-rs/seed)
*   [Sycamore](https://link.zhihu.com/?target=https%3A//github.com/sycamore-rs/sycamore)
*   [Stork](https://link.zhihu.com/?target=https%3A//stork-search.net/)

这些基于 Rust 的 web 框架，虽然编译到 WASM 但是并不计划去替换 JavaScript，而是选择与 JavaScript 共生。这是我们乐意见到的：Rust 即让 JavaScript 的功能更快，也能实现面向未来的编译到 [WASM](https://link.zhihu.com/?target=https%3A//rustwasm.github.io/docs/book/introduction.html)。

一路都是 Rust。

为什么不用 Rust？
-----------

Rust 有非常陡峭的学习曲线。Rust 的抽象层级要比绝大部分 Web 语言低。

> Rust 让你思考你的代码的方方面面，这对于系统开发是极为重要的。Rust 强制你思考内存是如何共享和拷贝的。Rust 强制你思考少见但是真实的边缘场景，以确保这些边缘场景也被控制住了。在任何能提升你效率的地方，Rust 都会帮你。  
> _– Tom MacWright ([Source](https://link.zhihu.com/?target=https%3A//macwright.com/2021/01/15/rust.html))_

Rust 在 Web 社区的使用还是很稀少的。还没被大规模应用。尽管对于 JavaScript 的工具制造者们，学习 Rust 是一个障碍。但是有趣的是，开发者们宁愿使用更快的工具，哪怕他们更难去[贡献代码](https://link.zhihu.com/?target=https%3A//twitter.com/devongovett/status/1261379312898306048)（因为 rust 更难）。[更快的软件最终获得胜利。](https://link.zhihu.com/?target=https%3A//craigmod.com/essays/fast_software/)（天下武功唯快不破）

目前，还是很难找到 Rust 的库去做大家喜欢的服务（比如鉴权、数据库、交易等等）。但是我认为，当 Rust 和 WASM 大规模应用以后，这些问题会自己解决的。但是不是现在。我们需要 JavaScript 工具帮助我们搭建一个桥梁，然后慢慢改进性能问题。

JavaScript 工具的未来
----------------

我相信，Rust 是 JavaScript 工具的未来。[Next.js 12](https://link.zhihu.com/?target=http%3A//nextjs.org/12) 开始全面使用 SWC 和 Rust 替换 Babel（编译器）和 Terser（代码压缩器）。为什么呢？

*   **扩展性:** SWC can be used as a Crate inside Next.js, without having to fork the library or workaround design constraints.
*   **性能:** 通过使用 SWC，我们获得了 3 倍的刷新速度和 5 倍的打包速度，而且性能提升仍然有很大的空间。
*   **WebAssembly:** Rust 支持 WASM，对于所有平台都可以支持，也可以带 Next.js 到任何地方。
*   **Community:** Rust 社区和生态系统非常惊人，还在持续增长。

不仅是 Next.js 使用 SWC，还有：

*   [Deno’s](https://link.zhihu.com/?target=https%3A//deno.land/) 的代码审查器、格式器、文档生成器都使用 [SWC](https://link.zhihu.com/?target=https%3A//twitter.com/devongovett/status/1369033422002389000)。
*   [Rome](https://link.zhihu.com/?target=https%3A//rome.tools/) 正在使用 [Rust](https://link.zhihu.com/?target=https%3A//twitter.com/rometools/status/1422616144763097091) 重写，计划使用SWC。（经提醒，这里有误）
*   [dprint](https://link.zhihu.com/?target=https%3A//github.com/devongovett/dprint-node), 使用 SWC创建,比 Prettier 快 [30x](https://link.zhihu.com/?target=https%3A//twitter.com/devongovett/status/1400138335721455617) 倍。
*   [Parcel](https://link.zhihu.com/?target=https%3A//parceljs.org/) 使用 SWC，改善了 [10x](https://link.zhihu.com/?target=https%3A//v2.parceljs.org/blog/beta3/) 的性能。

> Parcel 使用 SWC 就像用一个库。之前，我们使用 Babel 的解析器，用 JS 编写。现在我们使用 SWC 的解析器，用 [Rust](https://link.zhihu.com/?target=https%3A//github.com/parcel-bundler/parcel/tree/v2/packages/transformers/js/core/src) 编写。包括作用域提升的实现，依赖收集，还有其他。这个和 Deno 制造在 SWC 上面的原理一样。  
> _–[Devon Govett](https://link.zhihu.com/?target=https%3A//twitter.com/devongovett)_

虽然这是 Rust 的早期使用，有一些重要的点还要在这里指出来：

*   **插件:** 用 Rust 写插件对于大部分 JS 开发者还是比较难的事儿。用 JS 来制造插件体系，会有性能问题。这个问题目前还没有太好的解决方案。
*   **打包:** 目前我们感兴趣的是 swcpack，这个是 webpack 的替代工具。虽然现在还在开发阶段，但是未来可期。
*   **WebAssembly:** 虽然使用 Rust 和编译到 WASM 看起来很诱人，但是目前还有大量工作需要做。

不管怎样，我对 Rust 会持续加大影响 JavaScript 生态圈这件事是无比自信的。想象一下全世界使用 Next.js 的项目在打包时已经得到 Rust 带来的性能好处。然后，你可以像其他 NPM 包一样，使用 Next.js。

这是我想生活（和开发）的世界。
