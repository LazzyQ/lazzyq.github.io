---
title: "Golang内存分配"
date: 2021-01-23T11:19:32+08:00
draft: true
original: true
categories: 
  - Golang
tags: 
  - Golang基础
---

### 缘起

这篇文章是之前在技术群看到关于`strings.Builder`问题引发的一些列知识点的学习，这篇是其中一点，后续所有的知识点学习完成后，最后再来思考这个`strings.Builder`的问题

程序中的数据和变量都会被分配到程序所在的`虚拟内存`中，内存空间包含两个重要区域—-栈区（Stack）和堆区（Heap）。函数调用的参数、返回值以及局部变量大都会被分配到栈上，这部分内存会由编译器进行管理；不同编程语言使用不同的方法管理堆区的内存，C++ 等编程语言会由工程师主动申请和释放内存，Go 以及 Java 等编程语言会由工程师和编译器共同管理，堆中的对象由内存分配器分配并由垃圾收集器回收。不同的编程语言会选择不同的方式管理内存。

### 物理内存

首先先来看看分配的对象`内存`在物理上是如何实现的。

我们都知道`内存`中存储的数据是掉电就没有了，我们现在的的计算机的内存大小都是`GB`为单位的，那么在物理上如何存储这些`GB`的数据呢？要解决这个问题，先来看看，如何存储`1bit`的数据吧





### 参考资料

* [内存分配器](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-memory-allocator/)
* [栈内存管理](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-stack-management/)
* [图解Go语言内存分配](https://juejin.cn/post/6844903795739082760)
* [寄存器&内存](https://www.bilibili.com/video/BV1EW411u7th?p=6)