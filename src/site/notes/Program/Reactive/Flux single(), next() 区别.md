---
{"dg-publish":true,"permalink":"/Program/Reactive/Flux single(), next() 区别/","noteIcon":"","created":"2025-03-06T21:28:25.982+08:00"}
---


`next()` 获取第一个元素并取消订阅

`single()` 获取一个确定的元素，如果为空则抛出异常 

根据实际情况


Which one to choose depends on your use case. If the source is guaranteed to emit exactly one element, then you can use `single()`. Otherwise use `next()`.