---
title: "Golang 普通指针,unsafe.Pointer,uintptr的关系"
date: 2020-12-01T11:23:51+08:00
draft: true
original: true
categories: 
  - Golang
tags: 
  - 指针
---


### Golang指针

* *类型: 普通指针类型，用于传递对象地址，不能进行指针运算。
* unsafe.Pointer: 通用指针类型，用于转换不同类型的指针，不能进行指针运算，不能读取内存存储的值（必须转换到某一类型的普通指针）。
* uintptr: 用于指针运算，GC 不把 uintptr 当指针，uintptr 无法持有对象。uintptr 类型的目标会被回收。

unsafe.Pointer 是桥梁，可以让任意类型的指针实现相互转换，也可以将任意类型的指针转换为 uintptr 进行指针运算。

unsafe.Pointer 不能参与指针运算，比如你要在某个指针地址上加上一个偏移量，Pointer是不能做这个运算的，那么谁可以呢?

就是uintptr类型了，只要将Pointer类型转换成uintptr类型，做完加减法后，转换成Pointer，通过*操作，取值，修改值，随意。

总结：unsafe.Pointer 可以让你的变量在不同的普通指针类

https://www.cnblogs.com/-wenli/p/12682477.html