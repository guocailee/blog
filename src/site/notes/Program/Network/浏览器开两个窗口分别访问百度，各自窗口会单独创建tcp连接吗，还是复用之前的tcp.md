---
{"dg-publish":true,"permalink":"/Program/Network/浏览器开两个窗口分别访问百度，各自窗口会单独创建tcp连接吗，还是复用之前的tcp/","noteIcon":"","created":"2024-05-22T16:17:54.158+08:00"}
---





以 Chrome 来说，你可以在 DevTools 的[网络面板](https://www.zhihu.com/search?q=%E7%BD%91%E7%BB%9C%E9%9D%A2%E6%9D%BF&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A2688086521%7D)调出连接 ID 一栏，验证我说的这些知识点。

开两个窗口和开两个标签其实是一样的，Chrome 负责发送网络请求的进程是单独的，叫 Network Service，所有[网络请求](https://www.zhihu.com/search?q=%E7%BD%91%E7%BB%9C%E8%AF%B7%E6%B1%82&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A2688086521%7D)都是它发送的，而不是负责渲染页面的 renderer 进程：

![](https://picx.zhimg.com/80/v2-d910c3634c0109fbed35869263bf8ffc_720w.webp?source=2c26e567)

你问的这个问题妙就妙在是[问百度](https://www.zhihu.com/search?q=%E9%97%AE%E7%99%BE%E5%BA%A6&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A2688086521%7D)，百度的 [www.baidu.com](https://link.zhihu.com/?target=http%3A//www.baidu.com/) 恰好是为数不多还在用 HTTP/1.1 的域名。说开两个百度页面，那还得看是怎么开？是同时开？还是一个加载完了再开另外一个，这在 HTTP/1.1 下是有区别的，如果有先后顺序，先开的这个页面里最多会出现 6 个连接，但往往这些请求不是并发的，某个请求开始的时候，另外一个已经结束了，所以就能复用它的连接，我这个截图里演示的是 4 个连接：

![](https://pica.zhimg.com/80/v2-95c692005794c048c3f6541e09128467_720w.webp?source=2c26e567)

如果 [www.baidu.com](https://link.zhihu.com/?target=http%3A//www.baidu.com/) 的请求很多，总会把 6 个连接用完的。这个时候在开另外[一个窗口](https://www.zhihu.com/search?q=%E4%B8%80%E4%B8%AA%E7%AA%97%E5%8F%A3&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A2688086521%7D)打开百度，就会复用这 6 个连接，因为 keep-alive 嘛。

但如果是同时用两个窗口打开百度，那就纠缠混用起来了，可能最多能创建 12 个连接。

**编辑：有评论指出我上面这句结论是错的，的确，是我脑补出了一个更美好的 HTTP 1.1 的世界，实际上 network service 根本不管有几个页面正在打开中，它针对某个 HTTP/1.1 域名的连接上限一直是 6 个，**[所以只要你一个页面里有 6 个连接卡主了，你再打开多少新标签尝试也没用](https://link.zhihu.com/?target=https%3A//source.chromium.org/chromium/chromium/src/%2B/main%3Aservices/network/resource_scheduler/resource_scheduler.cc%3Bl%3D766)**，会一直白屏，除非把之前那个页面关掉，无痕窗口除外。下图就是我开两个页面后，超过 6 个的请求在 pending：** 

![](https://picx.zhimg.com/80/v2-eceec31f081830042c8baf89aeebcbfb_720w.webp?source=2c26e567)

百度就说完了，另外就是现在主流的支持 HTTP/2 的域名了，比如 [http://www.taobao.com](https://link.zhihu.com/?target=http%3A//www.taobao.com)，多个窗口的多个请求，不管同时不同时发起，都是只有 1 个连接 ID，[多路复用](https://www.zhihu.com/search?q=%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A2688086521%7D)就这个意思。

![](https://picx.zhimg.com/80/v2-f2945e08a1086b5b057d795c0e20eb2f_720w.webp?source=2c26e567)

该问题已加入各大厂和培训班面试题库。

