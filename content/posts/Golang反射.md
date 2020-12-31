---
title: "Golang反射"
date: 2020-12-28T22:42:14+08:00
draft: false
original: true
categories: 
  - Golang
tags: 
  - Golang基础
---

### 源起

最近在看开源项目，里面用到了反射，由于平时日常开发中几乎没有用到反射，所以对反射的知识点比较陌生，所以来学习记录一下。

### Golang接口

使用反射时，最常见的2个方法就是`reflect.TypeOf`和`reflect.ValueOf`

```go
// TypeOf returns the reflection Type that represents the dynamic type of i.
// If i is a nil interface value, TypeOf returns nil.
func TypeOf(i interface{}) Type {
	eface := *(*emptyInterface)(unsafe.Pointer(&i))
	return toType(eface.typ)
}
```

```go
// ValueOf returns a new Value initialized to the concrete value
// stored in the interface i. ValueOf(nil) returns the zero Value.
func ValueOf(i interface{}) Value {
	if i == nil {
		return Value{}
	}

	// TODO: Maybe allow contents of a Value to live on the stack.
	// For now we make the contents always escape to the heap. It
	// makes life easier in a few places (see chanrecv/mapassign
	// comment below).
	escapes(i)

	return unpackEface(i)
}
```

通过源码中的注释知道`Type`表示`i`的动态类型，`Value`表示`i`中存储的具体值。想要了解这是什么意思就需要了解go中的`interface`是什么。

<!--more-->

go的`interface`分别由2种结构体来表示: `iface`和`eface`

`iface`表示包含方法的接口，`eface`则表示的是没有任何方法的空接口

```go
type iface struct {
	tab  *itab
	data unsafe.Pointer
}

type itab struct {
	inter *interfacetype
	_type *_type
	hash  uint32 // copy of _type.hash. Used for type switches.
	_     [4]byte
	fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}
```

```go
type eface struct {
	_type *_type
	data  unsafe.Pointer
}
```

![golang接口结构](/golang反射/golang接口结构.png)

其中`interfacetype`表示的就是静态类型，`_type`就是动态类型

用例子来说明一下

```go
var r io.Reader
tty, err := os.OpenFile("/dev/tty", os.O_RDWR, 0)
if err != nil {
    return nil, err
}
r = tty
```

首先是`var r io.Reader`声明了变量`r`，此时结构如下

![r声明](/golang反射/r声明.png)

然后是`r = tty`，将`*os.File`的对象赋值给变量`r`，此时结构如下

![r赋值](/golang反射/r赋值.png)

除了带有方法的接口外，go中还有一种`interface{}`的空接口类型，任何类型的数据都能转换成`interface{}`

```go
var empty interface{}
empty = r
```

再将前面的`r`是赋值给`interface{}`的`empty`变量，此时结构如下

![empty结构](/golang反射/empty结构.png)

对于任何一个对象都分为类型和数据2个部分，在接口中(类型，数据)是绑定在一起的，而反射将类型和数据分别表示，也就是`reflect.TypeOf`和`reflect.ValueOf`的作用

### 类型转换

然后再来看看go里面的类型转换，这部分可以参考go官方的blog[类型转换](https://golang.org/ref/spec#Conversions)。类型转换有很多，这里我们只讨论和接口相关的类型转换，加深理解一下`iface`与`eface`，相当于实战吧

**函数调用时的隐式转换**

我们经常使用`fmt.Println`等方法来打印信息，先看看`fmt.Println`的定义

```go
// Println formats using the default formats for its operands and writes to standard output.
// Spaces are always added between operands and a newline is appended.
// It returns the number of bytes written and any write error encountered.
func Println(a ...interface{}) (n int, err error) {
	return Fprintln(os.Stdout, a...)
}
```

`fmt.Println`的参数是`interface{}`类型的，所以当我们传给`fmt.Println`的参数是`string`，`struct`等类型的时候，其实都发生了一次转换为`interface`的隐式转换，类似于前面的`empty = r`

除了转化成`interface{}`，还可以是之前`r = tty`这样的转换成`io.Reader`等接口类型

**显示转换**

除了隐式转换，还可以进行显示转换

```go
var a int = 25
b := interface{}(a)
```

**类型断言**

这部分可以参考go官方的blog[类型断言](https://golang.org/ref/spec#Type_assertions)

前面的类型转换都是转为`interface{}`，而有时我们需要将`interface{}`转换为其他类型，这时候就需要类型断言，比如

```go
var i interface{} = 1
b := a.(int)
```

### 使用反射

```
//接口数据  =====》 反射对象
1. Reflection goes from interface value to reflection object.

//反射对象 ===> 接口数据
2. Reflection goes from reflection object to interface value.

// 倘若数据可更改，可通过反射对象来修改它
3. To modify a reflection object, the value must be settable.
```

第1、2点比较好理解，直接看代码

```go
type User struct {
	Age  int
	Name string
}

func TestReflect(t *testing.T) {
	user := User{1, "小强"}

	ut := reflect.TypeOf(user)
	uv := reflect.ValueOf(user)
  t.Log(ut, uv)
  t.Log(uv.Type())
  
	userBack := uv.Interface().(User)
	t.Log(userBack)
}
```

从这段代码可以看出，通过反射可以进行如下的转换

![转换](/golang反射/转换.png)

第3点可以使用反射修改修改对象

```go
var x float64 = 3.4
v := reflect.ValueOf(x)
fmt.Println(v.CanSet()) // false
```

通过`CanSet`方法的返回值为`false`可以知道，不能对值进行修改。因为go都是值传递的，所以`reflect.ValueOf(x)`返回的其实是`x`的复制`v`，所以通过`v`是不能对`x`进行修改的


```go
var x float64 = 3.4
v := reflect.ValueOf(&x)
fmt.Println(v.CanSet()) // false
```

这次传递的是`x`的指针，但是`CanSet`方法仍然返回`false`，这是因为`reflect.ValueOf(&x)`是指针的`v`，如果要修改值，因为要把指针指向的值取出来，这是就需要用到`Elem`方法

```go
var x float64 = 3.4
v := reflect.ValueOf(&x)
fmt.Println(v.Elem().CanSet()) // false
```

### 其他方法

**Kind**

在go源码中定义了`Kind`是一些的常量，所以通过`Kind`方法返回的只能是下面这些值

```go
const (
	Invalid Kind = iota
	Bool
	Int
	Int8
	Int16
	Int32
	Int64
	Uint
	Uint8
	Uint16
	Uint32
	Uint64
	Uintptr
	Float32
	Float64
	Complex64
	Complex128
	Array
	Chan
	Func
	Interface
	Map
	Ptr
	Slice
	String
	Struct
	UnsafePointer
)
```

要注意一点的是，反射对象的类型`Kind`描述了基础类型，而不是静态类型

```go
type MyInt int

var x MyInt = 7
v := reflect.ValueOf(x)
fmt.Println(v.Kind())  // int
```

**Indirect**

```go
// Indirect returns the value that v points to.
// If v is a nil pointer, Indirect returns a zero Value.
// If v is not a pointer, Indirect returns v.
func Indirect(v Value) Value {
	if v.Kind() != Ptr {
		return v
	}
	return v.Elem()
}
```

这个方法和`Elem`差不多，只不过当调用`Elem`的对象不是`interface`或指针的时候会直接panic，而Indirect会温柔一些，如果不是指针会直接返回，是指针的时候就会返回指针指向的值

### 参考资料

* [图解go反射实现原理](https://i6448038.github.io/2020/02/15/golang-reflection/)
* [一篇理解什么是CanSet, CanAddr？](https://mp.weixin.qq.com/s/0CUuoO5mv0mJ3Zb_p1wCvA)
* [反射](https://draveness.me/golang/docs/part2-foundation/ch04-basic/golang-reflect/)
* [Go语言interface底层实现](https://i6448038.github.io/2018/10/01/Golang-interface/)
* [反射的法则](https://blog.golang.org/laws-of-reflection)
* [类型转换](https://golang.org/ref/spec#Conversions)
* [类型断言](https://golang.org/ref/spec#Type_assertions)