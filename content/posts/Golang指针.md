---
title: "Golang指针"
date: 2020-12-23T11:23:51+08:00
draft: true
original: true
categories: 
  - Golang
tags: 
  - Golang基础
---

### 源起

最近看到一段关于对象重用的代码，场景就是我们需要频繁的创建`[]byte`对象，为了减轻GC压力，所以考虑重用这些`[]byte`对象，代码大概是这样的

```go
func (p *LimitedPool) Get(size int) *[]byte {
	sp := p.findPool(size) // 根据需要[]byte的size查找对应的sync.Pool池对象
	if sp == nil {
		data := make([]byte, size)
		return &data
	}
	buf := sp.pool.Get().(*[]byte)  // 从sync.Pool池中获取
	*buf = (*buf)[:size]  // 将池中获取到的[]byte进行切片，将len设置成size大小
	return buf
}

func (p *LimitedPool) Put(b *[]byte) {
	sp := p.findPutPool(cap(*b)) // b 就是Get()方法中返回的*[]byte，这里根据[]byte的cap大小查找对应的sync.Pool池对象
	if sp == nil {
		return
	}
	*b = (*b)[:cap(*b)] // 将len和cap置为相同，Get()方法中的反向操作
	sp.pool.Put(b) // 归还到sync.Pool池中
}
```

不知道大家有什么疑问没有，至少我看完这段代码后产生了2个疑问：

1. 为什么`sync.Pool`中要存储`*[]byte`，直接用`[]byte`不行吗？
2. 获取的`[]byte`经过了切片操作后，还是原来的`[]byte`吗？

其实这2个问题多多少少和指针有关系，所以我们先补充一下指针的基础知识

### 指针的使用

```go
func TestBasePointer(t *testing.T) {
	num := 1
	numPointer := &num
	t.Logf("num的值: %v, num的指针: %v, num的指针: %p, 通过指针获取值: %v", num, numPointer, numPointer, *numPointer)
}
```

输出结果是

```
num的值: 1, num的指针: 0xc000016290, num的指针: 0xc000016290, 通过指针获取值: 1
```



### Golang指针

* *类型: 普通指针类型，用于传递对象地址，不能进行指针运算。
* unsafe.Pointer: 通用指针类型，用于转换不同类型的指针，不能进行指针运算，不能读取内存存储的值（必须转换到某一类型的普通指针）。
* uintptr: 用于指针运算，GC 不把 uintptr 当指针，uintptr 无法持有对象。uintptr 类型的目标会被回收。

unsafe.Pointer 是桥梁，可以让任意类型的指针实现相互转换，也可以将任意类型的指针转换为 uintptr 进行指针运算。

unsafe.Pointer 不能参与指针运算，比如你要在某个指针地址上加上一个偏移量，Pointer是不能做这个运算的，那么谁可以呢?

就是uintptr类型了，只要将Pointer类型转换成uintptr类型，做完加减法后，转换成Pointer，通过*操作，取值，修改值，随意。

总结：unsafe.Pointer 可以让你的变量在不同的普通指针类

### 参考资料

[Go 普通指针类型、unsafe.Pointer、uintptr之间的关系](https://www.cnblogs.com/-wenli/p/12682477.html)