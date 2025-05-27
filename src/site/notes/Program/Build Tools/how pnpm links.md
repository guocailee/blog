---
{"dg-publish":true,"permalink":"/Program/Build Tools/how pnpm links/","noteIcon":"","created":"2024-05-22T16:17:54.135+08:00"}
---

### 背景

日常我们会经常碰到关于 pnpm 以及幻影依赖的问题，有的问题比较复杂，涉及到了 pnpm 背后的实现原理，因此本文展开讲一讲 pnpm 的 link 机制。

我们通常说 pnpm 的一大优点就是避免了幻影依赖，默认禁止了 hoist，但是当我们说起 hoist 的时候，说的可能不是一回事，因为 pnpm 的 hoist 可能分为很多种情况。而且 pnpm 禁止不同 hoist 采取的策略也有所不同。

我们就结合 pnpm 的 link 策略来看看不同 hoist 的表现行为。

在讨论具体的 hoist 行为前，我们需要先区分两种代码，一种为 application code 即我们日常开发的业务代码，另一种为 vendor code 即三方库的代码，也包括三方库自身依赖的三方库。

这时候 hoist 的不同表现就体现在 vendor 和 application 的各种交互上。

> 因为 pnpm@6 和 pnpm@7 存在一些差异，本章讨论都建立在 pnpm@7 基础上

### application 与 vendor 之间的 hoist 行为 (public-hoist)

public-hoist 最常见也是我们日常所说的 hoist 行为。即我们的 application code 能够访问未声明在 application 的 dependency 的里直接依赖的 vendor code。

当我们配置 pnpm 的 node-linker 为 hoisted 的情况下，即默认所有的三方库都被 hoist。

我们简单看个例子 [demo](https://link.zhihu.com/?target=https%3A//github.com/hardfist/explain-pnpm/tree/node-linker)， 这里我们虽然只依赖了 express 这个库，但是仍然可以在 src/index.js 自由的访问 debug 这个库，这个正是因为 hoist 所致。

*   目录结构

```java
node_modules/
  debug
  express
  ...
 src/index.js
 package.json
```

*   package.json

```java
{
  "dependencies": {
    "express": "4.18.1"
  }
}
```

虽然这带来了一定的便捷性，但是同样带来了很大的危害，具体的危害见 [phantom-deps](https://link.zhihu.com/?target=https%3A//rushjs.io/zh-cn/pages/advanced/phantom_deps/)，此处不再赘述。

对于这类幻影依赖，pnpm 默认是严格禁止的，那么是如何做到禁止的呢。方法很简单，只要不将 express 之外的库直接放置到项目根目录 (root) 下的 node_modules 里即可。[example](https://link.zhihu.com/?target=https%3A//github.com/hardfist/explain-pnpm/tree/node-linker)

此时的项目结构如下，我们看到 node_modules 里已经没有了 express 之外的库了，这样自然无法在 src/index.js 里进行访问了。

```java
node_modules/
   express
   .pnpm
src/index.js
```

但是因为 prettier 和 eslint 的相关设计缺陷，导致其经常强依赖其相关的 plugin 存放在项目根目录的 node_modules 里，因此 pnpm 默认并没有禁止所有的库的 hoist 行为，而是给 eslint 和 prettier 开了后门。

默认值见 [public-hoist-pattern](https://link.zhihu.com/?target=https%3A//pnpm.io/npmrc%23public-hoist-pattern)。

如我们引入了 create-react-app 这个直接依赖进行安装后，惊讶的发现我们根目录的 node_modules 多了很多的其他的依赖。我们发现其中都是 eslint 和 types 的相关依赖被 hoist 上来。

![](https://pic2.zhimg.com/v2-d1db25e0f4391ea67e3815b9e406fde9_b.jpg)

hoist 问题似乎就这样迎刃而解了。但是这里碰到的一个问题是，如果我们直接依赖了 A 和 B 两个库，但是 A 和 B 又同时依赖了同一个版本的 C，那么我们的 C 该怎么处理。

最简单粗暴的处理方式即将 C 放在 A 和 B 各自的 node_modules 里。

```java
node_modules/
  A/
    node_modules/
    C
  B/
    node_modules/
    C
```

此时面临的问题就是 C 的内容是重复的，占据了我们磁盘空间，如果我们的项目非常大，那么这将充斥着我们的磁盘。（这也是 npm@{1,2}的默认行为）

既然害怕 C 冲突，那么很简单，我们将 C 链接到一个地方不就行了吗。

### symlink 与 hardlink

一般操作系统都支持两种链接方式，软链接 (symlink) 和硬链接 (hardlink), 这两种链接方式在 pnpm 都有使用，我们以一个例子为例 (example from hardlink vs symlink) 简单介绍下异同。

我们先创建一个文件，以及其 hardlink 和 symlink

```java
echo "111" > a
ln a b
ln -s a c
```

此时 a、b、c 的结果为

```java
cat a --> 111
cat b --> 111
cat c --> 111
```

我们看到 a、b、c 的结果保持同步，如果我们尝试下删除 a 文件，此时我们可以看到

```java
rm a
cat a --> No such file or directory
cat b --> 111
cat c --> No such file or directory
```

此时可以看到，c 的内容一并被删除，但是 b 的内容不受到影响，我们再尝试将 a 的内容复原

```java
echo "222" > a
cat a --> 222
cat b --> 111
cat c --> 222
```

此时我们发现 a 和 b 的内容不一致，但是 a 和 c 的内容一致，这反映了 hardlink 和 symlink 的一个重要区别

*   删除文件并不会影响 hardlink 的内容，但是会影响 symlink 的内容  
    
*   文件删除后再恢复内容，那么 hardlink 的 link 关系将不再维持，后续所有变更不会同步到 hardlink 里，但是仍然会和 symlink 保持同步  
    

因为 hardlink 难以保证和原文件的一致性，因此难以保证 hmr 的正常。

hardlink 相比 symlink 还有一个限制就是其无法支持 hard link 到一个目录，而 symlink 可以。

### node resolve

另一个区别就是两者在 node resolve 情况下行为的差异

我们创建三个 package

```java
echo "console.log('resolve:', module.paths[0]);" >> a/index.js
ln a/index.js b/index.js
ln -s a/index.js c/index.js
```

我们看下 三个目录的寻路算法

```java
node a/index.js --> a/node_modules
node b/index.js --> b/node_modules
node c/index.js --> a/node_modules
```

我们发现对于 hardlink 其 resolve 算法和被 link 的原文件无关，而对于 symlink 其 resolve 算法是从被 linked 的源文件算起，这对于运行时行为是比较大的差异，会影响到最终寻路的结果。

其实 symlink 不一定是基于被 linked 路径算起，大部分的工具和 node 都提供了一个 preserveSymlink 参数 (typescript symlink, webpack symlink, node symlink）

当我们使用 preserveLink 的时候，symlink 的计算路径就是基于该 symlink 的路径而非被 linked 文件路径进行计算。

```java
node --preserve-symlinks-main --preserve-symlinks c/index.js --> c/node_modules
```

使用 preserveLink 和 hard link 最大的风险在于，可能导致该查找到的库查找不到，或者同一个库 resolve 到了不同的结果，从而破坏了单例模式和导致 bundle 了多份产物，导致包大小问题。

### vendor 与 vendor 的 hoist 行为 (hoist)

vendor 和 vendor 的 hoist 是指，一个三方库可以访问不在其依赖里的其他三方库代码，这听起来有点不可思议，既然一个库用到了某个依赖，那理所当然应当将其列入其依赖，否则这个库肯定跑不起来啊，然而不幸的是，仍然有大量的三方库，没遵守这个约定。

以 webpack-cli 为例，其虽然依赖了 ts-node 等来将 ts 配置文件翻译为 js，但是其并没有将 ts-node 列入到其 dependency 和 peer-dependency 里，这里存在的一个风险就是，如果你之前恰好安装了 webpack-cli 和另一个库 A 且另一个库 A 正好又依赖了 ts-node, 并且你的 webpack 配置文件使用了 ts 文件，那么你很幸运的能够将 webpack 跑起来，突然有一天库 A 决定不使用 ts-node 作为依赖，那么不幸的是你的 webpack 将无法正常编译。

虽然有这种潜在的风险，奈何整个 js 的生态库都良莠不齐，导致 pnpm 也只能默认开启 hoist 模式，默认所有的 vendor 都是可以互相蹭的。你如果比较有追求，可以通过设置 [hoist](https://link.zhihu.com/?target=https%3A//pnpm.io/npmrc%23hoist) 为 false, 关闭三方库的 vendor 的 hoist 行为。

### 不同级别的拓扑结构

事实上 pnpm 支持四种级别的 node_modules 结构，从松到严依次为

### hoisted 模式

所有的三方库都平铺在根目录的 node_modules，这意味着 application code 能访问所有的依赖代码（无论是否在 dependency 里），所有的依赖也能互相访问其他依赖的代码（无论是否在 dependency)，这也是 npm 的默认模式。

### semi strict 模式

这也是 pnpm 的默认模式，这意味着 application code 仅能够访问其依赖里的库（types 和 eslint 相关库除外）, 但是所有的依赖仍然能够互相访问其他依赖的代码。

```json
; All packages are hoisted to node_modules/.pnpm/node_modules
hoist-pattern[]=*

; All types are hoisted to the root in order to make TypeScript happy
public-hoist-pattern[]=*types*

; All ESLint-related packages are hoisted to the root as well
public-hoist-pattern[]=*eslint*
```

### strict 模式

这种情况下，我们既禁止 application code 访问依赖外的代码，也禁止三方依赖访问其他非依赖里的三方依赖代码。这个模式也是最推荐业务使用的模式，但是不幸的是，pnpm 出于对生态的兼容性，做了妥协，默认并没有设置为该模式，但是作为有追求的业务方的你，应该使用这个模式。这可以保证你的业务不会突然有一天因为依赖问题突然挂掉。

### pnp 模式

即使 pnpm 开了最严格的 strict 模式，但是其只能控制本项目内的 node\_modules 的拓扑结构，项目父目录的 node\_modules 并不受到影响，所以仍然存在幻影依赖的风险，这个根因在于 node 的 resolve 算法是递归向上查找的，因此在不 hack node resolve 算法的基础上，是无法根除幻影依赖的，所以更激进的方式，就是修改 node 的 resolve 的默认行为，这样就保证了其不会递归向上查找，pnp 即采取了此种方式来根除幻影依赖问题，但是其也会带来新的问题，此处就不再多赘述。

那么问题来了，如果我设置了最严格的方式，但是三方库的依赖有 bug 咋整呢？

### 依赖修复方案

如果你的三方库的依赖存在 bug，pnpm 提供了多种方式来对依赖进行修复，你可以根据自己的需求选择合适的依赖修复方案。

### overrides | resolutions

如果你的某个依赖 A 的二级依赖 B 存在 bug，但是你又不想升级依赖 A，那么通过 overrides 可以强行指定 B 的版本。

```java
{
  "pnpm": {
    "overrides": {
      "B": "15.0.0",
    }
  }
}
```

但是其带来了一个问题就是，你将所以的 B 依赖版本都统一成了 15.0.0, 这可能不符合你的预期。虽然可以通过一些高级的语法来进行更精细的控制如

```java
{
  "pnpm": {
    "overrides": {
      "A@1>B": "15.0.0",
    }
  }
}
```

但是我们有更精细的方式进行控制

### packageExtensions

另一个常见的问题就是缺依赖，如 webpack-cli 依赖了 ts-node 但没将其列入依赖，我们就可以通过 packageExtensions 帮助其追加依赖，当然也可以通过这个方式来修改依赖。

```java
{
  "pnpm": {
    "packageExtensions": {
      "webpack-cli": {
        "peerDependencies": {
          "ts-node": "*"
        }
      },
      "express@1": {
        "optionalDependencies": {
          "typescript": "2"
        }
      },
      "fork-ts-checker-webpack-plugin": {
        "dependencies": {
          "@babel/core": "1"
        },
        "peerDependencies": {
          "eslint": ">= 6"
        },
        "peerDependenciesMeta": {
          "eslint": {
            "optional": true
          }
        }
      }
    }
  }
}
```

### .pnpmfile.cjs

上面两个方案都是针对的比较简单的修复场景，如果碰到比较复杂的依赖修复，如依赖了很多的判断条件，那么通过 hook 来进行控制将更为灵活。

如上面两种修复都可以基于 hook 进行实现。

```java
function readPackage(pkg, context) {
  if (pkg.name === 'A' && pkg.version.startsWith('1.')) {
    pkg.dependencies = {
      ...pkg.dependencies,
      B: '15.0.0'
    }
  }
  if (pkg.name === 'webpack-cli') {
    pkg.peerDependencies = {
      ...pkg.peerDependencies,
      "ts-node": "*"
    }
  }

  return pkg
}

module.exports = {
  hooks: {
    readPackage
  }
}
```

> 如果遇到 `readPackage` 钩子没有全量执行的问题，请尝试运行 `emo i --fix-lockfile`.

### npm alias

以上的修复都是针对，某个依赖的问题在其他版本上已经被修复，我们只需要重定向到其他版本即可，但是也可能存在这个 bug 在所有版本都有问题，那么这时候我们通常需要自行 fork 对应库的版本进行修复，因为我们没有原来库的发版权限，因此通常需要换个库名，此时可能需要修改所有引用库的地方的库名，但是通过 npm alias 我们只需要进行下 alias 即可解决这个问题。

如下，我们使用了一个修复了 bug 版本的 react-virtualized-fixed-import 来替代原本有 bug 的 react-virtualized.

```java
{
  "dependencies": {
     "react-virtualized": "npm:@byted-cg/react-virtualized-fixed-import@9.22.3"
  }
}
```

### How Pnpm Links

传统的 npm 的 node_modules 的拓扑结构，是难以精确控制 hoist 和 public-hoist 的，我们接下来看看 pnpm 是如何实现精确的控制 public-hoist 和 hoist 的行为的。

*   package.json

```java
{
  "dependencies": {
    "express": "4.18.1",
    "koa": "2.13.4"
  }
}
```

### 隐藏根目录的非直接依赖 vendor

首先为了解决 public-hoist，我们需要将非依赖的库不暴露到根目录的 node_modules 里避免被直接访问，即

```java
├── node_modules
│   ├── .modules.yaml
│   ├── .pnpm
│   │   ├── accepts@1.3.8
│   │   ├── array-flatten@1.1.1
│   │   ├── etc...
│   ├── express
│   └── koa ->
```

### 对依赖进行 link，解决同一版本重复问题

为了避免两个 vendor 里的同一版本依赖出现多次，我们需要将其进行链接到同一地方，如这里的 koa 和 express 使用了同一个版本的 accepts，我们可以看到首先我们会将 node\_modules/koa 和 node\_modules/express 链接到 node\_modules/.pnpm/express@4.18.1/node\_modules/express 和 .pnpm/koa@2.13.4/node\_modules/koa 里，然后会在 express 和 koa 里分别将 accepts 链接到同一个地方即 node\_modules/.pnpm/accepts@1.3.8/node_modules/accepts 里，这样保证了同一个版本只安装一次。

```java
├── node_modules
│   ├── .modules.yaml
│   ├── .pnpm
│   │   ├── accepts@1.3.8
│   │   ├── array-flatten@1.1.1
│   │   ├── express@4.18.1
│   │   │   └── node_modules
│   │   │       ├── accepts -> ../../accepts@1.3.8/node_modules/accepts
│   │   ├── koa@2.13.4
│   │   │   └── node_modules
│   │   │       ├── accepts -> ../../accepts@1.3.8/node_modules/accepts
│   │   │       ├── koa -> {store}/koa
│   ├── express -> .pnpm/express@4.18.1/node_modules/express
│   └── koa -> .pnpm/koa@2.13.4/node_modules/koa
```

这里有个很特别的设计，我们并不是将 koa 直接链接到 .pnpm/koa@2.13.4 里而是将其链接到 .pnpm/koa@2.13.4/node_modules/koa 里，为什么要这么设计呢？

出于如下几个目的

*   保证 koa 可以自己引用自己，这虽然算不上是一个 feature，但是不幸的是 npm 的 hoist 支持了该存在，因此 pnpm 需要进行兼容

```java
// koa/xxx.js
const koa = require('koa')
```

*   避免了循环 symlink: 如 a 依赖了 b，b 依赖了 a, 那么如果都使用 symlink，那么将很容易出现循环 symlink  
    
*   处理多 peerDep 问题：当存在多种版本的 peerDep 的时候，我们必须保证我们能同时 resolve 到不同的 peerDep 的版本。  
    

这里有个需要注意的地方就是 koa@2.13.4/node\_modules/koa 是到 store 的 koa 的 hardlink, 而 koa 的依赖则是指向 .pnpm/xxx/node\_modules/xxx 的 symlink，正是由于 hardlink 的存在，避免了出现 circular symlink 的情况。

### 通过 hardlink 实现跨项目的资源共享

pnpm 的另一大优势在于可以实现跨项目的内容共享，如上述的 .pnpm/koa@2.13.4/node\_modules/koa 其 hardlink 到全局 store 里的内容。hardlink 虽然可以实现内容共享，但是存在一个很大的风险就是 a 项目里如果修改了 node\_modules 里的的内容，该修改可能会影响 b 项目的 node_modules，这导致不同的项目是彼此不隔离的，这会导致非常大的风险问题。因此 rush 等方案为了屏蔽不同项目之间的 store 共享，所以默认并不会进行 store 共享，而是将 store 存储在项目中。因此 pnpm 支持了通过配置 package-import-method 支持不同的共享模式。

*   hardlink: 性能好，存在被误修改的风险  
    
*   copy: 性能差，但是安全性好  
    
*   clone: 性能好，安全性好，但是兼容性差（mac 不支持）  
    
*   clone-or-copy：先尝试 clone，如果不支持那么进行 copy, 性能好，安全性好，兼容性好  
    
*   auto: 先尝试 clone 再尝试 hardlink 再尝试 copy, 仍然存在一定的安全风险  
    

### 控制 vendor 之间的 hoist 行为

前面我们讲了如何通过控制根目录的 node\_modules 控制了 public-hoist 行为，那么如何控制 vendor 之间的 hoist 行为，答案很简单，通过 .pnpm/node\_modules 即可，因为 .pnpm/node\_modules/ 是所有 .pnpm/xxx/node\_modules/xxx 的父级目录，所以所有的三方库都能访问 .pnpm/node\_modules 的依赖，但是 application code 无法访问 .pnpm/node\_modules, 因此我们如果想让某个库被所有的其他库访问，只需要将其链接到到 .pnpm/node_modules 里即可。

如下面的 node\_modules/.pnpm/node\_modules/accepts 就可以被所有的 node\_modules/.pnpm/xxx/node\_modules/xxx 访问，如果不想其被访问，那么通过 hoist-pattern 将其从 node\_modules/.pnpm/node\_modules 移除即可。

```java
├── node_modules
│   ├── .modules.yaml
│   ├── .pnpm
│   │   ├── accepts@1.3.8
│   │   │   └── node_modules
│   │   ├── array-flatten@1.1.1
│   │   │   └── node_modules
│   │   ├── node_modules
│   │   │   ├── .bin
│   │   │   ├── accepts -> ../accepts@1.3.8/node_modules/accepts
│   │   │   ├── koa-compose -> ../koa-compose@4.1.0/node_modules/koa-compose
│   │   │   └── ylru -> ../ylru@1.3.2/node_modules/ylru
│   ├── express -> .pnpm/express@4.18.1/node_modules/express
│   └── koa -> .pnpm/koa@2.13.4/node_modules/koa
```

### PeerDependency 处理

前面我们已经很好的解决了 hoist 和 public-hoist 问题，但是还有一类更复杂的问题，就是 peerDependency 的处理，PeerDependency 把本就很复杂的 resolve 逻辑无疑又提高到了一个新的高度。

peerDep 有两个鲜明的特征，严重影响了 resolve 的流程

*   如果 package pnpm-foo peerDep 了依赖 foo-peer，那么不要求 foo 去安装 foo-peer，如果 foo 安装了 foo-peer，那么就 resolve 到 foo 的 foo-peer，否则将 resolve 到使用 foo 的地方安装的 foo-peer。  
    
*   如果 app1 依赖了 pnpm-foo@1.0.0 且 app2 也依赖了 pnpm-foo@1.0.0，那么即使他俩都依赖了同一个 pnpm-foo 的版本，但是 pnpm-foo@1.0.0 resolve 到 foo-peer 的结果可能是不同的，即 app1 依赖的 pnpm-foo@1.0.0 其 resolve 到 foo-peer 的版本可能是 1.0.0, 而 app2 依赖的 pnpm-foo@1.0.0 其 resolve 到 foo-peer 的版本可能是 2.0.0  
    

为了保证同一个 foo 能 resolve 的不同版本的 foo-peer 版本，pnpm 对 foo 进行了多次 hardlink

如下图，我们看到即使 pnpm-foo 的版本只有一份 (1.0.0)，但是为了保证其能 resolve 到不同的 foo-peer 版本，pnpm 仍然给 pnpm-foo 生成了两个 hardlink。

![](https://pic4.zhimg.com/v2-4b52afb7c0cea6719b239d4c935d5543_b.jpg)

我们看看此时 app1 和 app2 是如何加载对应的 pnpm-foo 和 foo-peer 的版本的，

*   app1/package.json

```java
"dependencies": {
    "pnpm-foo": "1.0.0",
    "foo-peer": "1.0.0"
  }
```

*   app2/package.json

```java
"dependencies": {
    "pnpm-foo": "1.0.0",
    "foo-peer": "2.0.0"
  }
```

我们看到，app1 和 app2 都加载了同一 pnpm-foo 版本，但是其 foo-peer 版本不同，此时他们的 node\_modules 结构如下，app1 链接到了 pnpm-foo@1.0.0-peer@1.0.0/node\_modules/pnpm-foo 而 app2 链接到了 pnpm-foo@1.0.0-peer@2.0.0/node_modules/pnpm-foo。

![](https://pic4.zhimg.com/v2-1eabbfac234d742fd19701ad5fdbb92b_b.jpg)

如下所示

![](https://pic1.zhimg.com/v2-51b808e5921d68e9b31dcad8a1792150_b.jpg)

我们进一步查看两者的内容是否一致，我们先看下两者的 inode 是否一致。

![](https://pic1.zhimg.com/v2-0b8925a0738c507cbee400c487cf76f4_b.jpg)

我们可以看到两者的 hardlink 完全一致，即两者虽然指向的是同一文件内容，但是他们创建了不同的 hardlink 链接。pnpm 巧妙的通过 hardlink 解决了 peerDep 的多版本问题，但是带来了另一个问题，即 peerDep 碎片化问题。

### peerDep 碎片化

我们看到因为 peerDep 的存在，导致了即使我们在项目中使用了 pnpm-foo 的同一个版本，但是为了保证 pnpm-foo 能 resolve 到不同的 peerDep 版本，从而导致存在了多个 pnpm-foo 的分身，这是典型的 npm 分身问题 (npm 分身），分身问题有哪些危害就不再赘述，最常见的就是会导致重复打包和单例模式破坏。

如我们的 app 依赖了 app1 和 app2, 我们对 app 进行打包，最终发现同一版本的 foo 被打包多次

![](https://pic2.zhimg.com/v2-4cfe8340728b3a7566cf5d3b9314ac31_b.jpg)

情况更糟糕的是，peerDep 导致的分身问题具有传染性，不仅仅是 pnpm-foo 会导致多重分身，pnpm-foo 的所有父依赖都需要进行分身来进行兼容。

即使 pnpm 的处理策略，满足 peerDep 的语义，但是可能并不符合用户的实际语义，用户大部分情况下不想 pnpm-foo 存在多份，另外也能接受 peerDep 使用同一版本，此时我们需要保证其引用的 foo-peer 是同一版本，这样才能保证 pnpm-foo 的版本只存在一份，通过 pnpm hook 我们可以比较容易的实现该需求，当然更好的办法是自己手动的修改 package.json 统一所有的 peerDep 的版本。

```java
function readPackage(pkg, context) {
  // 保证使用 pnpm-foo 的地方的 foo-peer 版本统一
  if (pkg.dependencies && pkg.peerDependencies) {
    if(pkg.dependencies["pnpm-foo"] && pkg.dependencies["foo-peer"]){
      pkg.dependencies["foo-peer"] = "1.0.0"
    }
  }
  console.log('pkg:',pkg);
  return pkg;
}

module.exports = {
  hooks: {
    readPackage
  }
}
```

### inject workspace

peerDep 的问题还不仅如此（在 pnpm 中）, pnpm 有个独特的性质，即 workspace 和三方库的 link 处理方式不同，对于三方库每个三方库都有一个指向全局 store 的 hardlink, 且当存在多重 peerDep 的版本的情况下，会存在多个 hardlink 分身，但是对于 workspace，如果 app1 依赖了某个 workspace sdk，那么这个 sdk 并不会创建 hardlink，而是直接将 app1 的 node_modules 的 sdk 软链接到 sdk。区别如下图所示

![](https://pic4.zhimg.com/v2-df28114ae435537c431a39a44e080187_b.jpg)

因为 workspace 没有使用 hardlink，这进一步导致难以为 worksapce 创建多重 hardlink 分身，因此 workspace 和三方库处理 peerDep 的方式也略有不同。

使用软链给 workspace 带来的一个问题就是 peerDep 的查找问题，我们以一个常见的 react 组件库为例。

假设我们有三个 workspace package 即 form、card 和 button，其中 form 和 card 依赖 button，并且 form 运行在 react17 下，card 运行在 react16 下，button 同时支持 react16 和 react17 两种环境，切其 peerDep 了 react。三者的 package.json 以及 index.jsx 如下如下

*   button/package.json

```json
"peerDependencies": {
    "react": "*"
  }
```

*   button/index.jsx

```js
import React from 'react';

export const Button = () => <div>button</div>

```

*   card/package.json

```json
"dependencies": {
    "react": "^16.0.0",
    "button": "workspace:*"
  }
```

*   card/index.jsx

```js
import { Button } from 'button';
console.log({Button});

```

*   form/package.json

```json
"dependencies": {
    "react": "17.0.0",
    "button": "workspace:*"
  }
```

*   form/index.jsx

```js
import { Button } from 'button';
console.log({Button});
```

此时如果我们在 form 中，使用 button 并进行打包，就惊讶的发现了一个错误

![](https://pic1.zhimg.com/v2-593ff4b071a6efe0b4f9b73f5555fd0c_b.jpg)

我们明明就在 form 里声明了 react 的版本，为啥还报依赖找不到呢，

很不幸如果我们的 form 和 button 是发布出去的三方库，那么这么写完全没问题，但是因为他们现在是 workspace，所以 pnpm 并没对其 peerDep 进行软链，这导致无法在 button 里查找到 form 中的 react 版本。

存在的解法方式有两种

*   使用 preserve-link 来保证 resolve 是相对于 form 而非 button，保证引入 form 里的 react，但这个会导致其他依赖引入可能出问题

另一种方式是基于 inject 实现，将{card,form}链接 button 的方式调整为 hardlink，这样 button resolve react 的路径就是相对于{card,form}而非是 root node_modules 了，pnpm 支持通过 dependencies injected 来调整其 symlink 为 hardlink。

*   form/package.json

```json
"dependenciesMeta": {
    "button": {
      "injected": true
    }
  }
```

这样 react 就能正常打包。但是正如前面提到过，一旦对 hardlink 做了删除、移动等操作，就会导致后续的 watch 失效，因此如果你在重新编译 button 的时候，删除了原来的产物，这将会导致 form 和 button 的 hardlink 断掉，因此导致 HMR 失效，因此这要求在使用 hardlink 的时候，不要对被 link 的文件内容进行删除操作。另一方面 watcher（如 chokidar）可能对 hardlink 支持不太友好，也可能导致即使没删除文件的情况下，一些 watch events 信息丢失 见 [watch-event-missing](https://link.zhihu.com/?target=https%3A//github.com/paulmillr/chokidar/issues/1071%23issuecomment-769749912)。