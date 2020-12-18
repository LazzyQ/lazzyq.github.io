---
title: "Golang Plan9汇编"
date: 2020-12-14T21:45:17+08:00
draft: false
original: true
categories: 
  - Golang
tags: 
  - plan9
  - 汇编
---

### 源起

最近在看draveness大佬的博客[函数调用](https://draveness.me/golang/docs/part2-foundation/ch04-basic/golang-function-call/)，发现里面出现了一些汇编语句，所以打算来入门一下。

我一直怀着一个问题：函数调用的时候参数是值存储形式的，返回值是不是也是值传递的？底层是怎么实现的？带着这个问题我们开始吧

### 帮助看懂汇编指令

* 汇编语言中mov和lea的区别有哪些？

lea是“load effective address”的缩写，简单的说，lea指令可以用来将一个内存地址直接赋给目的操作数，例如：lea eax,[ebx+8]就是将ebx+8这个值直接赋给eax，而不是把ebx+8处的内存地址里的数据赋给eax。而mov指令则恰恰相反，例如：mov eax,[ebx+8]则是把内存地址为ebx+8处的数据赋给eax。

一句话就是lea不会解引用

<!--more-->

* MOVB，MOVQ等这些指令后缀代表什么意思？

表示的是指令操作数的宽度，B表示8位，S表示16位，L表示32位，Q表示64位

* (8)SP这类表示什么意思？

表示的是地址SP+8，以此类推

* 函数开头的`TEXT	"".add(SB), NOSPLIT|ABIInternal, $0-16` 中`$0-16`是什么意思？

$0 表示栈帧(局部变量+可能需要的额外调用函数的参数空间的总大小，但不包括调用其它函数时的 ret address 的大小)大小为0, $16 表示参数及返回值大小为16个字节

* `FUNCDATA` 和 `PCDATA`是什么？
  
`FUNCDATA` 和 `PCDATA` 是编译器产生的，用于保存一些和垃圾收集相关的信息

### 练手

### 参数为数值类型的函数调用

首先来个简单的程序

```go
package main

//go:noinline
func add(a, b int) int {
	return a + b
}

func main() {
	add(1, 2)
}
```

不清楚`//go:noinline`是什么的同学可以看看[我的这篇博客](../golang中的pragmas)

然后编译一下

```go
go tool compile -S -N main.go
```

先看main函数

```go
"".main STEXT size=71 args=0x0 locals=0x20
	0x0000 00000 (main.go:8)	TEXT	"".main(SB), ABIInternal, $32-0 // 栈大小32，参数大小0
	0x0000 00000 (main.go:8)	MOVQ	(TLS), CX
	0x0009 00009 (main.go:8)	CMPQ	SP, 16(CX)
	0x000d 00013 (main.go:8)	PCDATA	$0, $-2
	0x000d 00013 (main.go:8)	JLS	64
	0x000f 00015 (main.go:8)	PCDATA	$0, $-1
	0x000f 00015 (main.go:8)	SUBQ	$32, SP // 分配 32 字节栈空间
	0x0013 00019 (main.go:8)	MOVQ	BP, 24(SP) // 将旧的BP存储到SP+24
	0x0018 00024 (main.go:8)	LEAQ	24(SP), BP // BP指向SP+24
	0x001d 00029 (main.go:8)	FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB) // FUNCDATA 跟垃圾回收有关
	0x001d 00029 (main.go:8)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB) // FUNCDATA 跟垃圾回收有关
	0x001d 00029 (main.go:9)	MOVQ	$1, (SP)   // 设置参数值1到SP
	0x0025 00037 (main.go:9)	MOVQ	$2, 8(SP)  // 设置参数值2到SP+8
	0x002e 00046 (main.go:9)	PCDATA	$1, $0
	0x002e 00046 (main.go:9)	CALL	"".add(SB) // 调用add方法，将call指令的下一条指令压栈
	0x0033 00051 (main.go:10)	MOVQ	24(SP), BP // 恢复BP为旧的BP值
	0x0038 00056 (main.go:10)	ADDQ	$32, SP    // 栈缩容，回收32字节的栈空间
	0x003c 00060 (main.go:10)	RET
	0x003d 00061 (main.go:10)	NOP
	0x003d 00061 (main.go:8)	PCDATA	$1, $-1
	0x003d 00061 (main.go:8)	PCDATA	$0, $-2
	0x003d 00061 (main.go:8)	NOP
	0x0040 00064 (main.go:8)	CALL	runtime.morestack_noctxt(SB)
	0x0045 00069 (main.go:8)	PCDATA	$0, $-1
	0x0045 00069 (main.go:8)	JMP	0
```

再来看看add方法

```go
"".add STEXT nosplit size=25 args=0x18 locals=0x0
	0x0000 00000 (main.go:4)	TEXT	"".add(SB), NOSPLIT|ABIInternal, $0-24 // 栈大小为0，参数大小24字节
	0x0000 00000 (main.go:4)	FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB) // FUNCDATA 跟垃圾回收有关
	0x0000 00000 (main.go:4)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB) // FUNCDATA 跟垃圾回收有关
	0x0000 00000 (main.go:4)	MOVQ	$0, "".~r2+24(SP) // 将返回值SP+24，也就是返回值置为0
	0x0009 00009 (main.go:5)	MOVQ	"".a+8(SP), AX // .a表示助记符，没有特别含义，AX = SP+8中的值(a)，也就是AX = 1
	0x000e 00014 (main.go:5)	ADDQ	"".b+16(SP), AX //.b表示助记符，没有特别含义，AX = AX + SP+16的值(b)，也就是AX = 1 + 2
	0x0013 00019 (main.go:5)	MOVQ	AX, "".~r2+24(SP) // AX的值赋值给SP+24
	0x0018 00024 (main.go:5)	RET // 栈顶的返回地址弹出
```

从这段代码中可以看出Go使用栈进行参数和返回值的传递

这里我的疑惑是在main方法中，参数是设置在`(SP)`和`8(SP)`位置的，也就是下面这2行

```go
0x001d 00029 (main.go:9)	MOVQ	$1, (SP)   // 设置参数值1到SP
0x0025 00037 (main.go:9)	MOVQ	$2, 8(SP)  // 设置参数值2到SP+8
```

但是在add方法中，获取参数是从`8(SP)`和`16(SP)`获取的，也就是这2行

```go
0x0009 00009 (main.go:5)	MOVQ	"".a+8(SP), AX // .a表示助记符，没有特别含义，AX = SP+8中的值(a)，也就是AX = 1
0x000e 00014 (main.go:5)	ADDQ	"".b+16(SP), AX //.b表示助记符，没有特别含义，AX = AX + SP+16的值(b)，也就是AX = 1 + 2
```

为什么偏移位置变了，猜测是SP向下偏移了8字节，那么SP的修改是在哪改动的呢？如果SP向下偏移了8个字节，但是在main函数中销毁栈的时候却只销毁了24个字节的空间，那么这8个字节是怎么增加和销毁的呢？

这个就和call指令有关系了，call指令执行过程中，call指令会把当前rip的值入栈（入栈的位置就是返回地址），栈顶SP也会下移到返回地址位置，然后把rip的值修改为call指令后面的操作数，也就是add函数第一条指令的地址，这样cpu就会跳转到add函数去执行。与call指令相反的操作就是ret指令了。

> 与call有个相似的命令：JMP。但是JMP指令不会将rip当前值压栈，栈顶SP调整，它是直接跳转到另一个地方运行，不会返回，一般是强制跳转。而call通过和ret指令返回配对使用。

通过这个例子知道了大概的调用流程，这里在补一个栈的示意图

### 参数类型为string的函数调用
 
一切从简

```go
package main

//go:noinline
func stringParam(s string) {}

func main() {
	var x = "abcc"
	stringParam(x)
}
```

汇编后main函数得到的结果

```
"".main STEXT size=72 args=0x0 locals=0x18
	0x0000 00000 (main.go:6)	TEXT	"".main(SB), ABIInternal, $24-0 // 栈帧大小24字节
	0x0000 00000 (main.go:6)	MOVQ	(TLS), CX
	0x0009 00009 (main.go:6)	CMPQ	SP, 16(CX)
	0x000d 00013 (main.go:6)	PCDATA	$0, $-2
	0x000d 00013 (main.go:6)	JLS	65
	0x000f 00015 (main.go:6)	PCDATA	$0, $-1
	0x000f 00015 (main.go:6)	SUBQ	$24, SP // 申请24字节大小的栈帧
	0x0013 00019 (main.go:6)	MOVQ	BP, 16(SP) // 保存旧的BP值
	0x0018 00024 (main.go:6)	LEAQ	16(SP), BP // BP指向新的地址
	0x001d 00029 (main.go:6)	FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x001d 00029 (main.go:6)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x001d 00029 (main.go:8)	LEAQ	go.string."abcc"(SB), AX // 获取 RODATA 段中的字符串地址
	0x0024 00036 (main.go:8)	MOVQ	AX, (SP) // 将获取到的地址放在栈顶，作为第一个参数
	0x0028 00040 (main.go:8)	MOVQ	$4, 8(SP) // 字符串长度作为第二个参数
	0x0031 00049 (main.go:8)	PCDATA	$1, $0
	0x0031 00049 (main.go:8)	CALL	"".stringParam(SB) // 调用 stringParam 函数
	0x0036 00054 (main.go:9)	MOVQ	16(SP), BP
	0x003b 00059 (main.go:9)	ADDQ	$24, SP
	0x003f 00063 (main.go:9)	NOP
	0x0040 00064 (main.go:9)	RET
	0x0041 00065 (main.go:9)	NOP
	0x0041 00065 (main.go:6)	PCDATA	$1, $-1
	0x0041 00065 (main.go:6)	PCDATA	$0, $-2
	0x0041 00065 (main.go:6)	CALL	runtime.morestack_noctxt(SB)
	0x0046 00070 (main.go:6)	PCDATA	$0, $-1
	0x0046 00070 (main.go:6)	JMP	0
```

在汇编层面 string 就是地址 + 字符串长度。

> TODO 打算写slice。struce的，太复杂了，等研究清楚了再补充

### 参考资料

* [golang 汇编](https://lrita.github.io/2017/12/12/golang-asm/)
* [golang汇编基础知识](https://guidao.github.io/asm.html)
* [Go语言汇编入门](https://blog.csdn.net/qq_31930499/article/details/100881461)
* [曹大 plan9 assembly 完全解析](https://github.com/cch123/golang-notes/blob/master/assembly.md)
* [Go 函数调用 ━ 栈和寄存器视角](https://segmentfault.com/a/1190000019753885)