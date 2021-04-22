---
title: "Golang 内存对齐"
date: 2021-04-22T11:32:28+08:00
draft: true
original: true
categories: 
  - Golang
tags: 
  - Golang基础
---

# struct对象占用多大的内存？

基本类型比如int64，int32等我们清楚的知道会占用多大的内存，但是对于struct这种复合类型，如何知道占用多大的内存呢？struct中每个字段类型占用的内存总和吗？今天我们就来讨论一下这个问题。

首先先测试一下，下面的2个结构体占用多大的空间

```go
type T1 struct {
	A uint8  //1
	B uint16 //2
	E uint8  //1
	C uint32 //4
	D uint64 //8
}

type T2 struct {
	A uint8  //1
	E uint8  //1
	B uint16 //2
	C uint32 //4
	D uint64 //8
}

unsafe.Sizeof(t1) // 24
unsafe.Sizeof(t2) // 16
```

不知道有没有回答正确，反正我第一次是没有回答正确的，接下来让我来说一下我的理解过程。

# struct的内存布局

复合类型就是内部包含多种数据类型的数据类型，它们都是存储在一块连续的内存空间中，struct在golang中就是其内部字段在内存中逐个排列的。这个和Java对象不同，Java对象还包含对象头信息，而golang是没有对象头的。

先来看看T1和T2在内存中的布局情况：


# 参考资料

* https://www.cnblogs.com/-wenli/p/12681044.html
