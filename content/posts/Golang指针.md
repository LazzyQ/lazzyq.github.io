---
title: "Golang指针"
date: 2020-12-25T11:23:51+08:00
draft: false
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
2. 获取的`[]byte`经过了切片操作后，返还会`sync.Pool`中会有问题吗？

其实这2个问题多多少少和指针有关系，所以我们先补充一下指针的基础知识

<!--more-->

### 指针的使用

**简单使用**

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

指针不仅可能指向`int`,`string`这些基本类型，还可以是`struct`，`slice`等复杂的结构，只要在内存中分配，我们都可以通过指针指过去

![指针](/Golang指针/指针.png)

**进一步使用指针**

除了使用`&`取地址，`*`解引用之外，Golang中还有`unsafe.Pointer`和`uintptr`的指针操作

* *类型: 普通指针类型，用于传递对象地址，不能进行指针运算。
* unsafe.Pointer: 通用指针类型，用于转换不同类型的指针，不能进行指针运算，不能读取内存存储的值（必须转换到某一类型的普通指针）。
* uintptr: 用于指针运算，GC 不把 uintptr 当指针，uintptr 无法持有对象。uintptr 类型的目标会被回收。

`unsafe.Pointer`称为通用指针，官方文档对该类型有四个重要描述：

1. 任何类型的指针都可以被转化为Pointer
2. Pointer可以被转化为任何类型的指针
3. uintptr可以被转化为Pointer
4. Pointer可以被转化为uintptr

我们不可以直接通过`*`操作来获取`unsafe.Pointer`指针指向的真实变量的值，因为我们并不知道变量的具体类型。

`unsafe.Pointer`是桥梁，可以让任意类型的指针实现相互转换，比如

```go
func TestUnsafePointerBridge(t *testing.T) {
	var vInt64 int64 = 1
	p := unsafe.Pointer(&vInt64)
	var i int = *(*int)(p)
	t.Logf("%v", i)
}
```

正常情况下`var i int = vInt64`是编译不过的，但是通过`unsafe.Pointer`这个桥梁就实现了`int64`到`int`的转换

也可以将任意类型的指针转换为`uintptr`进行指针运算，比如

```go
func TestUintptr(t *testing.T) {
	s := make([]int, 0, 10)
	s = append(s, 0, 1, 2)
	t.Logf("len: %d, cap: %d", len(s), cap(s))

	sp := unsafe.Pointer(&s)
	sa := uintptr(sp)
	sa = sa + unsafe.Sizeof(sp)
	t.Logf("len: %d", *(*int)(unsafe.Pointer(sa)))
	sa = sa + unsafe.Sizeof(int(0))
	t.Logf("cap: %d", *(*int)(unsafe.Pointer(sa)))
}
```

输出结果

```
len: 3, cap: 10
len: 3
cap: 10
```

这段代码中我们初始化了一个`slice`，然后将这个`slice`的指针转换成`unsafe.Pointer`，再将`unsafe.Pointer`转换成`uintptr`，通过`uintptr`进行指针运算，读取值就是反向进行转换，`uintptr`转换成`unsafe.Pointer`，然后转换成具体类型的指针，然后通过`*`解引用读取值。

![指针](/Golang指针/转换关系.png)

基础知识就到这，接下来就是解答之前的2个问题了

### 解惑

首先第1个问题：为什么`sync.Pool`中要存储`*[]byte`，直接用`[]byte`不行吗？

答案是，换成`[]byte`也行，但是会有一定的性能损失。

可能就有人会问为什么了？损失在什么地方？我们一步一步来

首先性能损失，我们来个benchmark测试一下

```go
func BenchmarkSyncPoolSlice(b *testing.B) {
	p := sync.Pool{
		New: func() interface{} {
			return make([]byte, 128)
		},
	}

	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			bytes, _ := p.Get().([]byte)
			bytes[0] = 1
			p.Put(bytes)
		}
	})

}

func BenchmarkSyncPoolSlicePointer(b *testing.B) {
	p := sync.Pool{
		New: func() interface{} {
			bytes := make([]byte, 128)
			return &bytes
		},
	}
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			bytes, _ := p.Get().(*[]byte)
			(*bytes)[0] = 1
			p.Put(bytes)
		}
	})
}
```

得到的结果如下

```
$ go test -bench=. -benchmem
goos: darwin
goarch: amd64
pkg: github.com/LazzyQ/daily-go/basic/pointer
BenchmarkSyncPoolSlice-8          	62884874	        16.7 ns/op	      32 B/op	       1 allocs/op
BenchmarkSyncPoolSlicePointer-8   	246374400	         4.49 ns/op	       0 B/op	       0 allocs/op
PASS
ok  	github.com/zengqiang96/daily-go/basic/pointer	2.769s

$ go test -bench=. -benchmem
goos: darwin
goarch: amd64
pkg: github.com/LazzyQ/daily-go/basic/pointer
BenchmarkSyncPoolSlice-8          	62338276	        16.8 ns/op	      32 B/op	       1 allocs/op
BenchmarkSyncPoolSlicePointer-8   	237126912	         4.84 ns/op	       0 B/op	       0 allocs/op
PASS
ok  	github.com/zengqiang96/daily-go/basic/pointer	2.828s

$ go test -bench=. -benchmem
goos: darwin
goarch: amd64
pkg: github.com/LazzyQ/daily-go/basic/pointer
BenchmarkSyncPoolSlice-8          	64518142	        17.6 ns/op	      32 B/op	       1 allocs/op
BenchmarkSyncPoolSlicePointer-8   	248263164	         5.20 ns/op	       0 B/op	       0 allocs/op
PASS
ok  	github.com/zengqiang96/daily-go/basic/pointer	4.131s
```

可以看到，使用`*[]byte`性能明显更好

接下来就来分析一下为什么

在前面指针的基础知识中，`*`，`unsafe.Pointer`和`uintptr`转换的例子中

```go
func TestUintptr(t *testing.T) {
	s := make([]int, 0, 10)
	s = append(s, 0, 1, 2)
	t.Logf("len: %d, cap: %d", len(s), cap(s))

	sp := unsafe.Pointer(&s)
	sa := uintptr(sp)
	sa = sa + unsafe.Sizeof(sp)
	t.Logf("len: %d", *(*int)(unsafe.Pointer(sa)))
	sa = sa + unsafe.Sizeof(int(0))
	t.Logf("cap: %d", *(*int)(unsafe.Pointer(sa)))
}
```

在使用`uintptr`进行指针计算的时候我们首先偏移了`unsafe.Sizeof(sp)`，也就是1个`unsafe.Pointer`占用的长度，然后偏移了`unsafe.Sizeof(int(0))`，也就是1个`int`占用的长度，我为啥会这么写，不是偏移1个`byte`，不是偏移其他什么类型的长度呢？因为我知道`slice`也是结构体类型，`runtime`中使用这样的结构来表示

```go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

所以第1次偏移`unsafe.Pointer`就能获取到`len`，再偏移`int`就能获取到`cap`

知道这一点，大概就明白为啥要使用`*[]byte`了

在go中都是值传递的，所以在调用`sync.Pool`的`Get`方法返回的如果是`*[]byte`，那么值传递拷贝的就是一个地址值，而如果是`[]byte`，那么值传递拷贝的就是一个`slice`结构体，明显拷贝的东西变多了。

所以这2者虽然对于`[]byte`的占用都内存不会进行拷贝，但是在方法调用返回结果拷贝的数据量不同造成了性能的不同

然后是第2个问题：获取的`[]byte`经过了切片操作后，返还会`sync.Pool`中会有问题吗？

有了第1个问题的认识后，这个问题其实也不难，我们需要明白对`[]byte`做了切片操作后得到的新的`[]byte`是怎样的

在`slice`的结构中有一个`array`字段的指针，它指向的就是存储数据的内存空间，我们先来验证一下对`slice`做了切片操作得到的`slice`的`array`指针发生了变化没有

```go
func TestSliceSlice(t *testing.T) {
	s := make([]int, 0, 10)
	s = append(s, 0, 1, 2, 3, 4)

	s1 := s[:3]

	t.Logf("s array: %v, s1 array: %p",
		*(*unsafe.Pointer)(unsafe.Pointer(uintptr(unsafe.Pointer(&s)))),
		*(*unsafe.Pointer)(unsafe.Pointer(uintptr(unsafe.Pointer(&s1)))))
}
```

输出结果

```
s array: 0xc0000e0140, s1 array: 0xc0000e0140
```

嗯。。。这个转换有点无话可说，我也是想了半天才写出来，反正最后可以看到结果`s`和`s1`的`array`指针都指向同一个内存地址，说明底层数组只有1份，只是`slice`结构体有多份，所以在放回`sync.Pool`的时候，指向`slice`的地址已经变了，画个图再解释一下

![slice](/Golang指针/slice.png)

所以从`sync.Pool`的`Get`方法获取到的`*[]byte`是`0x3E4A6F5A`，经过切片操作然后返还回`sync.Pool`的`*[]byte`其实变成`0x3E4A6D7B`了，但是底层的数组只有1份

### 参考资料

[Go 普通指针类型、unsafe.Pointer、uintptr之间的关系](https://www.cnblogs.com/-wenli/p/12682477.html)