---
title: "Golang String源码探险"
date: 2020-11-29T13:52:24+08:00
draft: false
original: true
categories: 
  - Golang
tags: 
  - Golang
---


### 源起

最近在看Golang的一个开源项目，里面发现了如下代码片段

```
// TODO 为什么要这么转和 string([]byte) 有什么区别
func SliceByteToString(b []byte) string {
	return *(*string)(unsafe.Pointer(&b))
}
```

看函数名称就知道这个函数的功能是想把`[]byte`转成`string`。正常情况下我们可以直接使用`string([]byte)`就能转化，为什么单独写个函数，而且函数里面的实现也是奇奇怪怪的

<!--more-->

### 探险

存在即合理，那么我们来看看这2者在性能上有什么差别吧

```go
package str

import (
	"testing"
	"unsafe"
)

var (
	byteData = []byte("一去二三里，烟村四五家")
)

func BenchmarkSlice2String(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_ = string(byteData)
	}
}

func BenchmarkSlice2String2(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_ = SliceByteToString(byteData)
	}
}

func SliceByteToString(b []byte) string {
	return *(*string)(unsafe.Pointer(&b))
}
```

```
$ go test -benchmem -bench .
goos: darwin
goarch: amd64

BenchmarkSlice2String-8    	42547107	        27.6 ns/op	      48 B/op	       1 allocs/op
BenchmarkSlice2String2-8   	1000000000	         0.300 ns/op	       0 B/op	       0 allocs/op
PASS
```

可以看到使用了`SliceByteToString`函数明显性能提升，而且内存分配也更少，那这里面到底是什么原因呢？

先看看`src/builtin/builtin.go`中对`string`的解释

```
// string is the set of all strings of 8-bit bytes, conventionally but not
// necessarily representing UTF-8-encoded text. A string may be empty, but
// not nil. Values of string type are immutable.
type string string
```

所以string是8比特字节的集合，通常但并不一定是UTF-8编码的文本。

另外，还提到了两点，非常重要：

- string可以为空（长度为0），但不会是nil；
- string对象不可以修改。

接下来看一下`src/runtime/string.go`中`string`对应的结构`go:stringStruct`

```go
type stringStruct struct {
	str unsafe.Pointer  // 某个数组的首地址
	len int             // 字符串长度
}
```

可以看到里面有一个数组的指针，那这个数组是什么呢？

#### 字符串声明

使用下面的代码可以声明一个string类型的变量并且赋值

```go
var str string
str = "Hello World"
```

在这个过程中会字符串构建`stringStruct`然后在转换成`string`

```go
//go:nosplit
func gostringnocopy(str *byte) string {
	ss := stringStruct{str: unsafe.Pointer(str), len: findnull(str)}
	s := *(*string)(unsafe.Pointer(&ss))
	return s
}
```

string在runtime包中就是stringStruct，对外呈现叫做string

由此我们可以联想到slice，string和slice其实很像，看一下slice的定义`src/runtime/slice.go`

```go
type slice struct {
	array unsafe.Pointer  //数组的指针
	len   int // 长度
	cap   int // 容量
}
```

除了多个cap外，看起来和string结构很像，但是它们又有很大的区别

```go
s := "A1" // 分配存储"A1"的内存空间，s结构体里的str指针指向这快内存
s[0] = 1 // 编译报错，不允许修改string中的内容
s = "A2"  // 重新给"A2"的分配内存空间，s结构体里的str指针指向这快内存
```

```go
s := []byte{0, 1} // 分配存储1数组的内存空间，s结构体的array指针指向这个数组。
s[0] = 1
```

string的指针指向的内容是不可以更改的，所以每更改一次字符串，就得重新分配一次内存，之前分配空间的还得由gc回收，这是导致string操作低效的根本原因

#### string转为[]byte

语法`[]byte(string)`源码如下：

```go
// The constant is known to the compiler.
// There is no fundamental theory behind this number.
const tmpStringBufSize = 32

type tmpBuf [tmpStringBufSize]byte

func stringtoslicebyte(buf *tmpBuf, s string) []byte {
	var b []byte
	if buf != nil && len(s) <= len(buf) {
		*buf = tmpBuf{}
		b = buf[:len(s)]
	} else {
		b = rawbyteslice(len(s))
	}
	copy(b, s)
	return b
}

// rawbyteslice allocates a new byte slice. The byte slice is not zeroed.
func rawbyteslice(size int) (b []byte) {
	cap := roundupsize(uintptr(size))
	p := mallocgc(cap, nil, false)
	if cap != uintptr(size) {
		memclrNoHeapPointers(add(p, uintptr(size)), cap-uintptr(size))
	}

	*(*slice)(unsafe.Pointer(&b)) = slice{p, size, int(cap)}
	return
}
```

可以看到b是新分配的，然后再将s复制给b，至于为啥copy函数可以直接把string复制给[]byte，那是因为go源码单独实现了一个`slicestringcopy`函数来实现，具体可以看`src/runtime/slice.go`

#### []byte转string

语法`string([]byte)`源码如下：

```go
// slicebytetostring converts a byte slice to a string.
// It is inserted by the compiler into generated code.
// ptr is a pointer to the first element of the slice;
// n is the length of the slice.
// Buf is a fixed-size buffer for the result,
// it is not nil if the result does not escape.
func slicebytetostring(buf *tmpBuf, ptr *byte, n int) (str string) {
	if n == 0 {
		// Turns out to be a relatively common case.
		// Consider that you want to parse out data between parens in "foo()bar",
		// you find the indices and convert the subslice to string.
		return ""
	}
	if raceenabled {
		racereadrangepc(unsafe.Pointer(ptr),
			uintptr(n),
			getcallerpc(),
			funcPC(slicebytetostring))
	}
	if msanenabled {
		msanread(unsafe.Pointer(ptr), uintptr(n))
	}
	if n == 1 {
		p := unsafe.Pointer(&staticuint64s[*ptr])
		if sys.BigEndian {
			p = add(p, 7)
		}
		stringStructOf(&str).str = p
		stringStructOf(&str).len = 1
		return
	}

	var p unsafe.Pointer
	if buf != nil && n <= len(buf) {
		p = unsafe.Pointer(buf)
	} else {
		p = mallocgc(uintptr(n), nil, false)
	}
	stringStructOf(&str).str = p
	stringStructOf(&str).len = n
	memmove(p, unsafe.Pointer(ptr), uintptr(n))
	return
}
```

可以看到也是发生了内存拷贝的，正因为`string`和`[]byte`相互转换都会有新的内存分配，才导致其代价不小。不过对于现在的计算机来说已经不算什么了。但是如果你的系统中存在大量的转换，同时你也清楚转换后对`[]byte`做出的改变会带来什么影响，你可以尝试使用文章开头给出的转换方式。

```go
func main() {
	b := []byte{72, 73}
	fmt.Println(string(b))
	str := SliceByteToString(b)
	fmt.Println(str)
	b[0] = 74
	fmt.Println(str)
}
```

输出结果

```
HI
HI
JI
```

可以看到，`str`随着`b`的修改而发生了变化，这其实违背了`string`是不可变的，所以需要谨慎使用

同样的逻辑我们也可以写出`StringToSliceByte`的方法

```go
func StringToSliceByte(s string) []byte {
	x := (*[2]uintptr)(unsafe.Pointer(&s))
	h := [3]uintptr{x[0], x[1], x[1]}
	return *(*[]byte)(unsafe.Pointer(&h))
}
```

然后我们在来试一试

```go
func main() {
	str := "HI"
	b := StringToSliceByte(str)
	fmt.Println(b)

	b[0] = 71
	fmt.Println(b)
}
```

输出结果

```
[72 73]
unexpected fault address 0x10cb3b1
fatal error: fault
[signal SIGBUS: bus error code=0x2 addr=0x10cb3b1 pc=0x10a6b58]

goroutine 1 [running]:
runtime.throw(0x10cb684, 0x5)
	/usr/local/Cellar/go/1.15.2/libexec/src/runtime/panic.go:1116 +0x72 fp=0xc000098ea8 sp=0xc000098e78 pc=0x1031832
runtime.sigpanic()
	/usr/local/Cellar/go/1.15.2/libexec/src/runtime/signal_unix.go:717 +0x465 fp=0xc000098ed8 sp=0xc000098ea8 pc=0x1045a65
main.main()
	/Users/zengqiang96/codespace/daily-go/basic/logger/main.go:13 +0x118 fp=0xc000098f88 sp=0xc000098ed8 pc=0x10a6b58
runtime.main()
	/usr/local/Cellar/go/1.15.2/libexec/src/runtime/proc.go:204 +0x209 fp=0xc000098fe0 sp=0xc000098f88 pc=0x1034009
runtime.goexit()
	/usr/local/Cellar/go/1.15.2/libexec/src/runtime/asm_amd64.s:1374 +0x1 fp=0xc000098fe8 sp=0xc000098fe0 pc=0x1062661
exit status 2
```

可以看到转换是正常的，但是在对转换后的`slice`做出修改后，出现了异常。由于`string`底层对应的数组是只读的，通过`StringToSliceByte`方法我们重用了这部分数据，所以就不能对转换后的slice进行修改

最后使用这种奇巧淫技的时候三思而后行啊