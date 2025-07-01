---
{"dg-publish":true,"permalink":"/Program/Mixed/复杂推理模型从服务器移植到Web浏览器的理论和实战/","noteIcon":"","created":"2024-05-22T16:17:54.155+08:00"}
---

## 一  背景

随着机器学习的应用面越来越广，能在浏览器中跑模型推理的 Javascript 框架引擎也越来越多了。在项目中，前端同学可能会找到一些跑在服务端的 python 算法模型，很想将其直接集成到自己的代码中，以 Javascript 语言在浏览器中运行。

对于一部分简单的模型，推理的前处理、后处理比较容易，不涉及复杂的科学计算，碰到这种模型，最多做个模型格式转化，然后用推理框架直接跑就可以了，这种移植成本很低。

而很大一部分模型会涉及复杂的前处理、后处理，包括大量的矩阵运算、图像处理等 Python 代码。这种情况一般的思路就是用 Javascript 语言将 Python 代码手工翻译一遍，这么做的问题是费时费力还容易出错。

Pyodide 作为浏览器中的科学计算框架，很好的解决了这个问题：浏览器中运行原生的 Python 代码进行前、后处理，大量 numpy、scipy 的矩阵、张量等计算无需翻译为 Javascript，为移植节省了很多工作。本文就基于 pyodide 框架，从理论和实战两个角度，帮助前端同学解决复杂模型的移植这一棘手问题。

## 二  原理篇

Pyodide 是个可以在浏览器中跑的 WebAssembly（wasm）应用。它基于 CPython 的源代码进行了扩展，使用 emscripten 编译成为 wasm，同时也把一大堆科学计算相关的 pypi 包也编译成了 wasm，这样就能在浏览器中解释执行 python 语句进行科学计算了。所以 pyodide 也必然遵循 wasm 的各种约束。Pyodide 在浏览器中的位置如下图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naKll50Ks5VicMcZIliaM0ZIwB9tRdjCzKmBCP1gXwyyMD01KicI4LEbcMp4n85nE0LsyLOgSLmU7CMUg/640?wx_fmt=png)

### 1  wasm 内存布局

这是 wasm 线性内存的布局：

![](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naKll50Ks5VicMcZIliaM0ZIwBEUBcwFlrkTeI2UNvyoibK4ialbOheNzw7deBsuab1PeFqh3Rv0F1Jhicw/640?wx_fmt=png)

Data 数据段是从 0x400 开始的， Function Table 表也在其中，起始地址为 memoryBase（Emscripten 中默认为 1024，即 0x400），STACKTOP 为栈地址起始，堆地址起始为 STACK_MAX。而我们实际更关心的是 Javascript 内存与 wasm 内存的互相访问。

### 2  Javascript 与 Python 的互访

浏览器基于安全方面的考虑，防止 wasm 程序把浏览器搞崩溃，通过把 wasm 运行在一个沙箱化的执行环境中，禁止了 wasm 程序访问 Javascript 内存，而 Javascript 代码却可以访问 wasm 内存。因为 wasm 内存本质上是一个巨大的 ArrayBuffer，接受 Javascript 的管理。我们称之为 “单向内存访问”。

作为一个 wasm 格式的普通程序，pyodide 被调用起来后，当然只能直接访问 wasm 内存。

![](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naKll50Ks5VicMcZIliaM0ZIwBeVCYlHDHlekicX4TOLVkKrEzibQuSeqowySxWeWSzMDTXhxcnby4B6FA/640?wx_fmt=png)

为了实现互访，pyodide 引入了 proxy，类似于指针：在 Javascript 侧，通过一个 PyProxy 对象来引用 python 内存里的对象；在 Python 侧，通过一个 JsProxy 对象来引用 Javascript 内存里的对象。

在 Javascript 侧生成一个 PyProxy 对象：

```cs
const arr_pyproxy = pyodide.globals.get('arr')  // arr是python里的一个全局对象
```

在 Python 侧生成一个 JsProxy 对象：

```python
import js
from js import foo   # foo是Javascript里的一个全局对象
```

互访时的类型转换分为如下三个等级：

-   【自动转换】对于简单类型，如数字、字符串、布尔等，会被自动拷贝内存值，此时产生的就不是 Proxy、而是最终的值了。


-   【半自动转换】非简单的内置类型，都需要通过 to_js()、to_py() 方式来显式转换：


-   对于 Python 内置的 list、dict、numpy.ndarray 等对象，不属于简单类型，不会自动转换类型，必须通过 pyodide.to_js() 来转，相应的会被转成 JS 的 list、map、TypedArray 类型


-   反过来也类似，通过 to_py() 方法，JS 的 TypedArray 转为 memoryview，list、map 转为 list、dict


-   【手动转换】各种 class、function 和用户自定义类型，因为对方的语言没有对应的现成类型，所以只能以 proxy 的形式存在，需要通过运算符来间接操纵，就像操纵提线木偶一样。为了达到方便操纵的目的，pyodide 对两种语言进行了语法模拟，用一种语言里的操作符模拟另一种语言的类似行为。例如：JS 中的 let a=new XXX()，在 Python 中就变为 a=XXX.new()。

![](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naKll50Ks5VicMcZIliaM0ZIwBrfuQZzgibvzeoicAbhKPee1WS1ETW5dJoVVbIib0TobW1IqceUb4X0HxQ/640?wx_fmt=png)

这里列举了一部分，详情可以查文档（见文章底部）。

![](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naKll50Ks5VicMcZIliaM0ZIwBGcsibZTc6El1TOsbSJxZ91Vqo3D7NVUHDIB8s0gN7tQlC0OdekGqJeQ/640?wx_fmt=png)

Javascript 的模块也可以引入到 Python 中，这样 Python 就能直接调用该模块的接口和方法了。例如，pyodide 没有编译 opencv 包，可以使用 opencv.js：

```swift
import pyodide
import js.cv as cv2
print(dir(cv2))
```

这对于 pyodide 缺失的 pypi 包是个很好的补充。

## 三  实践篇

我们从一个空白页面开始。使用浏览器打开测试页面（测试页面见文章底部）。

### 1  初始化 python

为了方便观察运行过程，使用动态的方式加载所需 js 和执行 python 代码。打开浏览器控制台，依次运行以下语句：

```javascript
function loadJS( url, callback ){
  var script = document.createElement('script'),
  fn = callback || function(){};
  script.type = 'text/javascript';
  script.onload = function(){
      fn();
  };
  script.src = url;
  document.getElementsByTagName('head')[0].appendChild(script);
}
// 加载opencv
loadJS('https://test-bucket-duplicate.oss-cn-hangzhou.aliyuncs.com/public/opencv/opencv.js',  function(){
    console.log('js load ok');
});

// 加载推理引擎onnxruntime.js。当然也可以使用其他推理引擎
loadJS('https://test-bucket-duplicate.oss-cn-hangzhou.aliyuncs.com/public/onnxruntime/onnx.min.js',  function(){
    console.log('js load ok');
});

// 初始化python运行环境
loadJS('https://test-bucket-duplicate.oss-cn-hangzhou.aliyuncs.com/public/pyodide/0.18.0/pyodide.js',  function(){
    console.log('js load ok');
});
pyodide = await loadPyodide({ indexURL : "https://test-bucket-duplicate.oss-cn-hangzhou.aliyuncs.com/public/pyodide/0.18.0/"});
await pyodide.loadPackage(['micropip']);
```

![](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naKll50Ks5VicMcZIliaM0ZIwBuNqQraYRaFExHtHJNPH3bLjhEn15z7cIoEyZU4jHugG7glWdBazhMw/640?wx_fmt=png)

至此，python 和 pip 就安装完毕了，都位于内存文件系统中。我们可以查看一下 python 被安装到了哪里：

![](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naKll50Ks5VicMcZIliaM0ZIwBX2YqNKDULfVf71yAWE03yr9kp0jYVQSsaUtavScLibfffBDibfGqFg6w/640?wx_fmt=png)

注意，这个文件系统是内存里虚拟出来的，刷新页面就丢失了。不过由于浏览器本身有缓存，所以刷新页面后从服务端再次加载 pyodide 的引导 js 和主体 wasm 还是比较快的，只要不清理浏览器缓存。 

### 2  加载 pypi 包

在 pyodide 初始化完成后，python 系统自带的标准模块可以直接 import。第三方模块需要用 micropip.install() 安装：

-   pypi.org 上的纯 python 包可以用 micropip.install() 直接安装
-   含有 C 语言扩展（编译为动态链接库）的 wheel 包，需要对照官方已编译包的列表
-   在列表中的直接用 micropip.install() 安装
-   不在这个列表里的，就需要自己手动编译后发布到服务器后再用 micropip.install() 安装。
下图展示了业内常用的两种编译为 wasm 的方式。

![](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naKll50Ks5VicMcZIliaM0ZIwBcaVxjOn3UKK76x1BWSOcklgJuYs5vVY5q7xA5aia3PfgSjbnkvUhNJQ/640?wx_fmt=png)

自己编译 wasm package 的方法可参考官方手册，大致步骤就是 pull 官方的编译基础镜像，把待编译包的 setup.cfg 文件放到模块目录里，再加上些 hack 的语句和配置（如果有的话），然后指定目标进行编译。编译成功后部署时，需要注意 2 点：

-   设置允许跨域


-   对于 wasm 格式的文件请求，响应 Header 里应当带上："Content-type": "application/wasm"

下面是一个自建 wasm 服务器的 nginx/openresty 示例配置：

```nginx
        location ~ ^/wasm/ {
          add_header 'Access-Control-Allow-Origin' "*";
          add_header 'Access-Control-Allow-Credentials' "true";
          root /path/to/wasm_dir;
          header_filter_by_lua '
            uri = ngx.var.uri
            if string.match(uri, ".js$") == nil then
                ngx.header["Content-type"] = "application/wasm"
            end
          ';
        }
```

回到我们的推理实例， 现在用 pip 安装模型推理所需的 numpy 和 Pillow 包并将其 import：

```python
await pyodide.runPythonAsync(`
import micropip
micropip.install(["numpy", "Pillow"])
`);

await pyodide.runPythonAsync(`
import pyodide
import js.cv as cv2
import js.onnx as onnxruntime
import numpy as np
`);
```

这样 python 所需的 opencv、onnxruntime 包就已全部导入了。

### 3  opencv 的使用

一般 python 里的图片数组都是从 JS 里传过来的，这里我们模拟构造一张图片，然后用 opencv 对其 resize。上面提到过，pyodide 官方的 opencv 还没编译出来。如果涉及到的 opencv 方法调用有其他 pypi 包的替代品，那是最好的：比如，cv.resize 可以用 Pillow 库的 PIL.resize 代替（注意 Pillow 的 resize 速度比 opencv 的 resize 要慢）；cv.threshold 可以用 numpy.where 代替。 否则只能调用 opencv.js 的能力了。为了演示 pyodide 语法，这里都从 opencv.js 库里调用。

```makefile
await pyodide.runPythonAsync(`
# 构造一个1080p图片
h,w = 1080,1920
img = np.arange(h * w * 3, dtype=np.uint8).reshape(h, w, 3)

# 使用cv2.resize将其缩小为1/10
# 原python代码：small_img = cv2.resize(img, (h_small, w_small))
# 改成调用opencv.js：
h_small,w_small = 108, 192
mat = cv2.matFromArray(h, w, cv2.CV_8UC3, pyodide.to_js(img.reshape(h * w * 3)))
dst = cv2.Mat.new(h_small, w_small, cv2.CV_8UC3)  
cv2.resize(mat, dst, cv2.Size.new(w_small, h_small), 0, 0, cv2.INTER_NEAREST)
small_img = np.asarray(dst.data.to_py()).reshape(h_small, w_small, 3)
`);
```

传参原则：除了简单的数字、字符串类型可以直接传，其他类型都需要通过 pyodide.to_js() 转换后再传入。 返回值的获取也类似，除了简单的数字、字符串类型可以直接获取，其他类型都需要通过 xx.to_py() 转换后获取结果。

接着对一个 mask 检测其轮廓：

```makefile
await pyodide.runPythonAsync(`
# 使用cv2.findContours来检测轮廓。假设mask为二维numpy数组，只有0、1两个值
# 原python代码：contours  = cv2.findContours(mask, cv2.RETR_CCOMP,cv2.CHAIN_APPROX_NONE)
# 改成调用opencv.js：
contours_jsproxy = cv2.MatVector.new()   # cv2.Mat数组，对应opencv.js中的 contours = new cv.MatVector()语句
hierarchy_jsproxy = cv2.Mat.new()
mat = cv2.matFromArray(mask.shape[0], mask.shape[1],  cv2.CV_8UC1, pyodide.to_js(mask.reshape(mask.size)))
cv2.findContours(mat, pyodide.to_js(contours_jsproxy), pyodide.to_js(hierarchy_jsproxy), cv2.RETR_LIST, cv2.CHAIN_APPROX_SIMPLE)
# contours js格式转python格式
contours = []
for i in range(contours_jsproxy.size()):
  c_jsproxy = contours_jsproxy.get(i)
  c = np.asarray(c_jsproxy.data32S.to_py()).reshape(c_jsproxy.rows, c_jsproxy.cols, 2)
  contours.append(c)
`);
```

### 4  推理引擎的使用

最后，用 onnx.js 加载模型并进行推理，详细语法可参考 onnx.js 官方文档。其他 js 版的推理引擎也都可以参考各自的文档。

```makefile
await pyodide.runPythonAsync(`
model_url="onnx模型的地址"
session = onnxruntime.InferenceSession.new()
session.loadModel(model_url)
session.run(......)
`);
```

通过以上的操作，我们确保了一切都在 python 语法范围内进行，这样修改原始的 Python 文件就比较容易了：把不支持的函数替换成我们自定义的调用 js 的方法；原 Python 里的推理替换成调用 js 版的推理引擎；最后在 Javascript 主程序框架里加少许调用 Python 的胶水代码就完成了。

### 5  挂载持久存储文件系统

有时我们需要对一些数据持久保存，可以利用 pyodide 提供的持久化文件系统（其实是 emscripten 提供的），见手册（文章底部）。

```javascript
// 创建挂载点
pyodide.FS.mkdir('/mnt');
// 挂载文件系统
pyodide.FS.mount(pyodide.FS.filesystems.IDBFS, {}, '/mnt');
// 写入一个文件
pyodide.FS.writeFile('/mnt/test.txt', 'hello world');
// 真正的保存文件到持久文件系统
pyodide.FS.syncfs(function (err) {
    console.log(err);
});
```

这样文件就持久保存了。即使当我们刷新页面后，仍可以通过挂载该文件系统来读出里面的内容：

```javascript
// 创建挂载点
pyodide.FS.mkdir('/mnt');
// 挂载文件系统
pyodide.FS.mount(pyodide.FS.filesystems.IDBFS, {}, '/mnt');
// 写入一个文件
pyodide.FS.writeFile('/mnt/test.txt', 'hello world');
// 真正的保存文件到持久文件系统
pyodide.FS.syncfs(function (err) {
    console.log(err);
});
```

运行结果如下：

![](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naKll50Ks5VicMcZIliaM0ZIwBeiblibCbhZGvL8SPcc9CePiaR7T8nGwWOoKEShKic2pk1DD0DyUWMuNGIg/640?wx_fmt=png)

当然，以上语句可以在 python 中以 Proxy 的语法方式运行。

持久文件系统有很多用处。例如，可以帮我们在多线程（webworker）之间共享大数据；可以把模型文件持久存储到文件系统里，无需每次都通过网络加载。

### 6  打 wheel 包

单 Python 文件无需打包，直接当成一个巨大的字符串，交给 pyodide.runPythonAsync() 运行就好了。当有多个 Python 文件时，我们可以把这些 python 文件打成普通 wheel 包，部署到 webserver，然后可以用 micropip 直接安装该 wheel 包：

```javascript
micropip.install("https://foo.com/bar-1.2.3-xxx.whl")
from bar import ...
```

注意，打 wheel 包需要有\_\_init\_\_.py 文件，哪怕是个空文件。

# 四  存在的缺陷

目前 pyodide 有如下几个缺陷：

-   Python 运行环境加载和初始化时间有点儿长，视网络情况，几秒到几十秒都有可能。


-   pypi 包支持的不完整。虽然 pypi.org 上的纯 python 包都可以直接使用，但涉及到 C 扩展写的包，如果官方还没编译出来，那就需要自己动手编译了。


-   个别很常用的包，例如 opencv，还没成功编译出来；模型推理框架一个都没有。不过还好可以通过相应的 JS 库来弥补。


-   如果 python 中调用了 js 库的话：


-   可能会产生一定的内存拷贝开销（从 wasm 内存到 JS 内存的来回拷贝）。尤其是大数组作为参数或返回值，在速度要求高的场合下，额外的内存拷贝开销就不能忽视了。


-   python 库的方法接口可能跟其对应的 js 库的接口参数、返回值格式不一致，有一定的适配工作量。

## 五  总结
尽管有上述种种缺陷，得益于代码移植的高效率和逻辑上 1:1 复刻的高可靠性保障，我们还是可以把这种方法运用到多种业务场景里，为推动机器学习技术的应用添砖加瓦。

链接：  

1、测试页面：  

[https://test-bucket-duplicate.oss-cn-hangzhou.aliyuncs.com/public/pyodide/test.html](https://test-bucket-duplicate.oss-cn-hangzhou.aliyuncs.com/public/pyodide/test.html)

2、文档：

[https://pyodide.org/en/stable/usage/type-conversions.html](https://pyodide.org/en/stable/usage/type-conversions.html)

3、官方已编译包的列表：

[https://github.com/pyodide/pyodide/tree/main/packages](https://github.com/pyodide/pyodide/tree/main/packages)

4、手册：

[https://emscripten.org/docs/api\\\_reference/Filesystem-API.html](https://emscripten.org/docs/api\_reference/Filesystem-API.html)
