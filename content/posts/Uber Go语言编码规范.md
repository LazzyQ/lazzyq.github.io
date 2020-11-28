---
title: "Uber Go语言编码规范"
date: 2020-11-29T00:44:23+08:00
draft: false
original: false
displayCopyright: false
categories: 
  - Golang
tags: 
  - 编码规范
---


[Uber](https://www.uber.com/)是世界领先的生活出行服务提供商，也是[Go语言](https://tonybai.com/tag/go)的早期adopter，根据[Uber工程博客](https://eng.uber.com/)的内容，大致可以判断出Go语言在Uber内部扮演了十分重要的角色。Uber内部的Go语言工程实践也是硕果累累，有大量Go实现的内部工具[被Uber开源到github上](https://github.com/uber-go)，诸如被Gopher圈熟知的[zap](https://github.com/uber-go/zap)、[jaeger](https://github.com/jaegertracing/jaeger)等。2018年年末Uber将内部的[Go风格规范](https://github.com/uber-go/guide)开源到github，经过一年的积累和更新，该规范已经初具规模，并受到广大Gopher的关注。本文是该规范的中文版本，并”夹带“了部分笔者的点评，希望对国内Gopher有所帮助。

> 注：该版本基于[commit 3baa2bd](https://github.com/uber-go/guide/commit/3baa2bdd4677d7ef650be138a7c53b49d36da645)翻译，后续不会持续更新。

### 介绍

样式(style)是支配我们代码的惯例。术语“样式”有点用词不当，因为这些约定涵盖的范围不限于由gofmt替我们处理的源文件格式。

本指南的目的是通过详细描述在Uber编写Go代码的注意事项来管理这种复杂性。这些规则的存在是为了使代码库易于管理，同时仍然允许工程师更有效地使用Go语言功能。

该指南最初由[Prashant Varanasi](https://github.com/prashantv)和[Simon Newton](https://github.com/nomis52)编写，目的是使一些同事能快速使用Go。多年来，该指南已根据其他人的反馈进行了修改。

本文档记录了我们在Uber遵循的Go代码中的惯用约定。其中许多是Go的通用准则，而其他扩展准则依赖于下面外部的指南：

- [Effective Go](https://golang.org/doc/effective_go.html)
- [The Go common mistakes guide](https://github.com/golang/go/wiki/CodeReviewComments)

所有代码都应该通过[golint](https://github.com/golang/lint)和[go vet](https://tip.golang.org/cmd/vet/)的检查并无错误。我们建议您将编辑器设置为：

- 保存时运行[goimports](https://github.com/golang/tools/tree/master/cmd/goimports)
- 运行golint和go vet检查源码

您可以在以下Go编辑器工具支持页面中找到更为详细的信息：https://github.com/golang/go/wiki/IDEsAndTextEditorPlugins

### 指导原则

#### 指向interface的指针

您几乎不需要指向接口类型的指针。您应该将接口作为值进行传递，在这样的传递过程中，实质上传递的底层数据仍然可以是指针。

接口实质上在底层用两个字段表示：

- 一个指向某些特定类型信息的指针。您可以将其视为“类型”。
- 数据指针。如果存储的数据是指针，则直接存储。如果存储的数据是一个值，则存储指向该值的指针。

如果要接口方法修改底层数据，则必须用指向目标对象的指针赋值给接口类型变量(译注：感觉原指南中这里表达过于简略，不是很清晰，因此在翻译时增加了自己的一些诠释)。

#### 接收器(receiver)与接口

使用值接收器的方法既可以通过值调用，也可以通过指针调用。

例如:

```go
type S struct {
  data string
}

func (s S) Read() string {
  return s.data
}

func (s *S) Write(str string) {
  s.data = str
}

sVals := map[int]S{1: {"A"}}

// 你只能通过值调用Read
sVals[1].Read()

// 下面无法通过编译：
//  sVals[1].Write("test")

sPtrs := map[int]*S{1: {"A"}}

// 通过指针既可以调用Read，也可以调用Write方法
sPtrs[1].Read()
sPtrs[1].Write("test")
```

同样，即使该方法具有值接收器，也可以通过指针来满足接口。

```go
type F interface {
  f()
}

type S1 struct{}

func (s S1) f() {}

type S2 struct{}

func (s *S2) f() {}

s1Val := S1{}
s1Ptr := &S1{}
s2Val := S2{}
s2Ptr := &S2{}

var i F
i = s1Val
i = s1Ptr
i = s2Ptr

// 下面代码无法通过编译。因为s2Val是一个值，而S2的f方法中没有使用值接收器
//   i = s2Val
```

《Effective Go》中有一段关于[“pointers vs values”](https://golang.org/doc/effective_go.html#pointers_vs_values)的精彩讲解。

> 译注：关于Go类型的method集合的问题，在我之前的文章[《关于Go，你可能不注意的7件事》](https://tonybai.com/2015/09/17/7-things-you-may-not-pay-attation-to-in-go)中有详尽说明。

#### 零值Mutex是有效的

sync.Mutex和sync.RWMutex是有效的。因此你几乎不需要一个指向mutex的指针。

**Bad**:

```go
mu := new(sync.Mutex)
mu.Lock()
```

vs.

**Good**:

```go
var mu sync.Mutex
mu.Lock()
```

如果你使用结构体指针，mutex可以非指针形式作为结构体的组成字段，或者更好的方式是直接嵌入到结构体中。

如果是私有结构体类型或是要实现Mutex接口的类型，我们可以使用嵌入mutex的方法：

```go
type smap struct {
  sync.Mutex

  data map[string]string
}

func newSMap() *smap {
  return &smap{
    data: make(map[string]string),
  }
}

func (m *smap) Get(k string) string {
  m.Lock()
  defer m.Unlock()

  return m.data[k]
}
```

对于导出类型，请使用私有锁：

```go
type SMap struct {
  mu sync.Mutex

  data map[string]string
}

func NewSMap() *SMap {
  return &SMap{
    data: make(map[string]string),
  }
}

func (m *SMap) Get(k string) string {
  m.mu.Lock()
  defer m.mu.Unlock()

  return m.data[k]
}
```

#### 在边界处拷贝Slices和Maps

slices和maps包含了指向底层数据的指针，因此在需要复制它们时要特别注意。

**接收Slices和Maps**

请记住，当map或slice作为函数参数传入时，如果您存储了对它们的引用，则用户可以对其进行修改。

**Bad**

```go
unc (d *Driver) SetTrips(trips []Trip) {
  d.trips = trips
}

trips := ...
d1.SetTrips(trips)

// 你是要修改d1.trips吗？
trips[0] = ...
```

vs.

**Good**

```go
func (d *Driver) SetTrips(trips []Trip) {
  d.trips = make([]Trip, len(trips))
  copy(d.trips, trips)
}

trips := ...
d1.SetTrips(trips)

// 这里我们修改trips[0]，但不会影响到d1.trips
trips[0] = ...
```

**返回slices或maps**

同样，请注意用户对暴露内部状态的map或slice的修改。

**Bad**

```go
type Stats struct {
  sync.Mutex

  counters map[string]int
}

// Snapshot返回当前状态
func (s *Stats) Snapshot() map[string]int {
  s.Lock()
  defer s.Unlock()

  return s.counters
}

// snapshot不再受到锁的保护
snapshot := stats.Snapshot()
```

vs.

**Good**

```go
type Stats struct {
  sync.Mutex

  counters map[string]int
}

func (s *Stats) Snapshot() map[string]int {
  s.Lock()
  defer s.Unlock()

  result := make(map[string]int, len(s.counters))
  for k, v := range s.counters {
    result[k] = v
  }
  return result
}

// snapshot现在是一个拷贝
snapshot := stats.Snapshot()
```

#### 使用[defer](https://tonybai.com/2013/02/03/implement-go-defer-in-c/)做清理

使用defer清理资源，诸如文件和锁。

**Bad**

```go
p.Lock()
if p.count < 10 {
  p.Unlock()
  return p.count
}

p.count++
newCount := p.count
p.Unlock()

return newCount

// 当有多个return分支时，很容易遗忘unlock
```

vs.

**Good**

```go
p.Lock()
defer p.Unlock()

if p.count < 10 {
  return p.count
}

p.count++
return p.count

// 更可读
```

Defer的开销非常小，只有在您可以证明函数执行时间处于纳秒级的程度时，才应避免这样做。使用defer提升可读性是值得的，因为使用它们的成本微不足道。尤其适用于那些不仅仅是简单内存访问的较大的方法，在这些方法中其他计算的资源消耗远超过defer。

#### Channel的size要么是1，要么是无缓冲的

channel通常size应为1或是无缓冲的。默认情况下，channel是无缓冲的，其size为零。任何其他尺寸都必须经过严格的审查。考虑如何确定大小，是什么阻止了channel在负载下被填满并阻止写入，以及发生这种情况时发生了什么。

**Bad**

```go
// 应该足以满足任何人
c := make(chan int, 64)
```

vs.

**Good**

```go
// 大小：1
c := make(chan int, 1) // 或
// 无缓冲channel，大小为0
c := make(chan int)
```

#### 枚举从1开始

在Go中引入枚举的标准方法是声明一个自定义类型和一个使用了iota的const组。由于变量的默认值为0，因此通常应以非零值开头枚举。

**Bad**

```go
type Operation int

const (
  Add Operation = iota
  Subtract
  Multiply
)

// Add=0, Subtract=1, Multiply=2
```

vs.

**Good**

```go
type Operation int

const (
  Add Operation = iota + 1
  Subtract
  Multiply
)

// Add=1, Subtract=2, Multiply=3
```

在某些情况下，使用零值是有意义的(枚举从零开始)，例如，当零值是理想的默认行为时。

```go
type LogOutput int

const (
  LogToStdout LogOutput = iota
  LogToFile
  LogToRemote
)

// LogToStdout=0, LogToFile=1, LogToRemote=2
```

#### 错误类型

Go中有多种声明错误（Error)的选项：

- errors.New 对于简单静态字符串的错误
- fmt.Errorf 用于格式化的错误字符串
- 实现Error()方法的自定义类型
- 使用 “pkg/errors”.Wrap的wrapped error

返回错误时，请考虑以下因素以确定最佳选择：

- 这是一个不需要额外信息的简单错误吗？如果是这样，errors.New 就足够了。
- 客户需要检测并处理此错误吗？如果是这样，则应使用自定义类型并实现该Error()方法。
- 您是否正在传播下游函数返回的错误？如果是这样，请查看本文后面有关错误包装(Error Wrap)部分的内容
- 否则，fmt.Errorf就可以。

如果客户端需要检测错误，并且您已使用创建了一个简单的错误errors.New，请使用一个错误变量(sentinel error )。

**Bad**

```go
// package foo

func Open() error {
  return errors.New("could not open")
}

// package bar

func use() {
  if err := foo.Open(); err != nil {
    if err.Error() == "could not open" {
      // handle
    } else {
      panic("unknown error")
    }
  }
}
```

vs.

**Good**

```go
// package foo

var ErrCouldNotOpen = errors.New("could not open")

func Open() error {
  return ErrCouldNotOpen
}

// package bar

if err := foo.Open(); err != nil {
  if err == foo.ErrCouldNotOpen {
    // handle
  } else {
    panic("unknown error")
  }
}
```

如果您有可能需要客户端检测的错误，并且想向其中添加更多信息（例如，它不是静态字符串），则应使用自定义类型。

**Bad**

```go
func open(file string) error {
  return fmt.Errorf("file %q not found", file)
}

func use() {
  if err := open(); err != nil {
    if strings.Contains(err.Error(), "not found") {
      // handle
    } else {
      panic("unknown error")
    }
  }
}
```

vs.

**Good**

```go
type errNotFound struct {
  file string
}

func (e errNotFound) Error() string {
  return fmt.Sprintf("file %q not found", e.file)
}

func open(file string) error {
  return errNotFound{file: file}
}

func use() {
  if err := open(); err != nil {
    if _, ok := err.(errNotFound); ok {
      // handle
    } else {
      panic("unknown error")
    }
  }
}
```

直接导出自定义错误类型时要小心，因为它们已成为程序包公共API的一部分。最好公开匹配器功能以检查错误。

```go
// package foo

type errNotFound struct {
  file string
}

func (e errNotFound) Error() string {
  return fmt.Sprintf("file %q not found", e.file)
}

func IsNotFoundError(err error) bool {
  _, ok := err.(errNotFound)
  return ok
}

func Open(file string) error {
  return errNotFound{file: file}
}

// package bar

if err := foo.Open("foo"); err != nil {
  if foo.IsNotFoundError(err) {
    // handle
  } else {
    panic("unknown error")
  }
}
```

#### 错误包装(Error Wrapping)

一个(函数/方法)调用失败时，有三种主要的错误传播方式：

- 如果没有要添加的其他上下文，并且您想要维护原始错误类型，则返回原始错误。
- 添加上下文，使用”pkg/errors”.Wrap以便错误消息提供更多上下文，”pkg/errors”.Cause可用于提取原始错误。
- 使用fmt.Errorf，如果调用者不需要检测或处理的特定错误情况。

建议在可能的地方添加上下文，以使您获得诸如“调用服务foo：连接被拒绝”之类的更有用的错误，而不是诸如“连接被拒绝”之类的模糊错误。

在将上下文添加到返回的错误时，请避免使用“ failed to”之类的短语来保持上下文简洁，这些短语会陈述明显的内容，并随着错误在堆栈中的渗透而逐渐堆积：

**Bad**

```go
s, err := store.New()
if err != nil {
    return fmt.Errorf(
        "failed to create new store: %s", err)
}

failed to x: failed to y: failed to create new store: the error
```

vs.

**Good**

```go
s, err := store.New()
if err != nil {
    return fmt.Errorf(
        "new store: %s", err)
}

x: y: new store: the error
```

但是，一旦将错误发送到另一个系统，就应该明确消息是错误消息（例如使用err标记，或在日志中以”Failed”为前缀）。

另请参见[Don’t just check errors, handle them gracefully](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully).

#### 处理类型断言失败

类型断言的单个返回值形式针对不正确的类型将产生panic。因此，请始终使用“comma ok”的惯用法。

**Bad**

```
t := i.(string)
```

vs.

**Good**

```go
t, ok := i.(string)
if !ok {
  // 优雅地处理错误
}
```

#### 不要[panic](https://tonybai.com/2019/04/04/notes-about-fixing-a-go-panic-problem)

在生产环境中运行的代码必须避免出现panic。panic是[级联失败](https://en.wikipedia.org/wiki/Cascading_failure)的主要根源 。如果发生错误，该函数必须返回错误，并允许调用方决定如何处理它。

**Bad**

```go
func foo(bar string) {
  if len(bar) == 0 {
    panic("bar must not be empty")
  }
  // ...
}

func main() {
  if len(os.Args) != 2 {
    fmt.Println("USAGE: foo <bar>")
    os.Exit(1)
  }
  foo(os.Args[1])
}
```

vs.

**Good**

```go
func foo(bar string) error {
  if len(bar) == 0
    return errors.New("bar must not be empty")
  }
  // ...
  return nil
}

func main() {
  if len(os.Args) != 2 {
    fmt.Println("USAGE: foo <bar>")
    os.Exit(1)
  }
  if err := foo(os.Args[1]); err != nil {
    panic(err)
  }
}
```

panic/recover不是错误处理策略。仅当发生不可恢复的事情（例如:nil引用）时，程序才必须panic。程序初始化是一个例外：程序启动时应使程序中止的不良情况可能会引起panic。

```go
var _statusTemplate = template.Must(template.New("name").Parse("_statusHTML"))
```

即便是在test中，也优先使用t.Fatal或t.FailNow来标记test是失败的，而不是panic。

**Bad**

```go
// func TestFoo(t *testing.T)

f, err := ioutil.TempFile("", "test")
if err != nil {
  panic("failed to set up test")
}
```

vs.

**Good**

```go
// func TestFoo(t *testing.T)

f, err := ioutil.TempFile("", "test")
if err != nil {
  t.Fatal("failed to set up test")
}
```

#### 使用go.uber.org/atomic

使用sync/atomic包的原子操作对原始类型（int32，int64等）进行操作(译注：指atomic包的方法名中均使用原始类型名，如SwapInt32等)，因此很容易忘记使用原子操作来读取或修改变量。

go.uber.org/atomic通过隐藏基础类型为这些操作增加了类型安全性。此外，它包括一个方便的atomic.Bool类型。

**Bad**

```go
type foo struct {
  running int32  // atomic
}

func (f* foo) start() {
  if atomic.SwapInt32(&f.running, 1) == 1 {
     // already running…
     return
  }
  // start the Foo
}

func (f *foo) isRunning() bool {
  return f.running == 1  // race!
}
```

vs.

**Good**

```go
type foo struct {
  running atomic.Bool
}

func (f *foo) start() {
  if f.running.Swap(true) {
     // already running…
     return
  }
  // start the Foo
}

func (f *foo) isRunning() bool {
  return f.running.Load()
}
```

### 性能

性能方面的特定准则，适用于热路径。

#### 优先使用strconv而不是fmt

将原语转换为字符串或从字符串转换时，strconv速度比fmt快。

**Bad**

```go
for i := 0; i < b.N; i++ {
  s := fmt.Sprint(rand.Int())
}

BenchmarkFmtSprint-4    143 ns/op    2 allocs/op
```

vs.

**Good**

```go
for i := 0; i < b.N; i++ {
  s := strconv.Itoa(rand.Int())
}

BenchmarkStrconv-4    64.2 ns/op    1 allocs/op
```

#### 避免字符串到字节的转换

不要反复从固定字符串创建字节slice。相反，请执行一次转换并捕获结果。

**Bad**

```go
for i := 0; i < b.N; i++ {
  w.Write([]byte("Hello world"))
}

BenchmarkBad-4   50000000   22.2 ns/op
```

vs.

**Good**

```go
data := []byte("Hello world")
for i := 0; i < b.N; i++ {
  w.Write(data)
}

BenchmarkGood-4  500000000   3.25 ns/op
```

### 样式

#### 相似的声明放在一组

Go语言支持将相似的声明放在一个组内：

**Bad**

```
import "a"
import "b"
```

vs.

**Good**

```
import (
  "a"
  "b"
)
```

这同样适用于常量、变量和类型声明：

**Bad**

```
const a = 1
const b = 2

var a = 1
var b = 2

type Area float64
type Volume float64
```

vs.

**Good**

```
const (
  a = 1
  b = 2
)

var (
  a = 1
  b = 2
)

type (
  Area float64
  Volume float64
)
```

仅将相关的声明放在一组。不要将不相关的声明放在一组。

**Bad**

```
type Operation int

const (
  Add Operation = iota + 1
  Subtract
  Multiply
  ENV_VAR = "MY_ENV"
)
```

vs.

**Good**

```
type Operation int

const (
  Add Operation = iota + 1
  Subtract
  Multiply
)

const ENV_VAR = "MY_ENV"
```

分组使用的位置没有限制，例如：你可以在函数内部使用它们：

**Bad**

```
func f() string {
  var red = color.New(0xff0000)
  var green = color.New(0x00ff00)
  var blue = color.New(0x0000ff)

  ...
}
```

vs.

**Good**

```
func f() string {
  var (
    red   = color.New(0xff0000)
    green = color.New(0x00ff00)
    blue  = color.New(0x0000ff)
  )

  ...
}
```

#### import组内的包导入顺序

应该有两类导入组：

- 标准库
- 其他一切

默认情况下，这是goimports应用的分组。

**Bad**

```
import (
  "fmt"
  "os"
  "go.uber.org/atomic"
  "golang.org/x/sync/errgroup"
)
```

vs.

**Good**

```
import (
  "fmt"
  "os"

  "go.uber.org/atomic"
  "golang.org/x/sync/errgroup"
)
```

#### 包名

当命名包时，请按下面规则选择一个名称：

- 全部小写。没有大写或下划线。
- 大多数使用命名导入的情况下，不需要重命名。
- 简短而简洁。请记住，在每个使用的地方都完整标识了该名称。
- 不用复数。例如net/url，而不是net/urls。
- 不是“common”，“util”，“shared”或“lib”。这些是不好的，信息量不足的名称。

另请参阅[Go包名称](https://blog.golang.org/package-names)和[Go包样式指南](https://rakyll.org/style-packages/)。

#### 函数名

我们遵循Go社区关于[使用MixedCaps作为函数名的约定](https://golang.org/doc/effective_go.html#mixed-caps)。有一个例外，为了对相关的测试用例进行分组，函数名可能包含下划线，如: TestMyFunction_WhatIsBeingTested。

#### 包导入别名

如果程序包名称与导入路径的最后一个元素不匹配，则必须使用导入别名。

```
import (
  "net/http"

  client "example.com/client-go"
  trace "example.com/trace/v2"
)
```

在所有其他情况下，除非导入之间有直接冲突，否则应避免导入别名。

**Bad**

```
import (
  "fmt"
  "os"

  nettrace "golang.net/x/trace"
)
```

vs.

**Good**

```
import (
  "fmt"
  "os"
  "runtime/trace"

  nettrace "golang.net/x/trace"
)
```

#### 函数分组与顺序

- 函数应按粗略的调用顺序排序。
- 同一文件中的函数应按接收者分组。

因此，导出的函数应先出现在文件中，放在struct、const和var定义的后面。

在定义类型之后，但在接收者的其余方法之前，可能会出现一个newXYZ()/ NewXYZ()。

由于函数是按接收者分组的，因此普通工具函数应在文件末尾出现。

**Bad**

```
func (s *something) Cost() {
  return calcCost(s.weights)
}

type something struct{ ... }

func calcCost(n int[]) int {...}

func (s *something) Stop() {...}

func newSomething() *something {
    return &something{}
}
```

vs.

**Good**

```
type something struct{ ... }

func newSomething() *something {
    return &something{}
}

func (s *something) Cost() {
  return calcCost(s.weights)
}

func (s *something) Stop() {...}

func calcCost(n int[]) int {...}
```

#### 减少嵌套

代码应通过尽可能先处理错误情况/特殊情况并尽早返回或继续循环来减少嵌套。减少嵌套多个级别的代码的代码量。

**Bad**

```go
for _, v := range data {
  if v.F1 == 1 {
    v = process(v)
    if err := v.Call(); err == nil {
      v.Send()
    } else {
      return err
    }
  } else {
    log.Printf("Invalid v: %v", v)
  }
}
```

vs.

**Good**

```go
for _, v := range data {
  if v.F1 != 1 {
    log.Printf("Invalid v: %v", v)
    continue
  }

  v = process(v)
  if err := v.Call(); err != nil {
    return err
  }
  v.Send()
}
```

#### 不必要的else

如果在if的两个分支中都设置了变量，则可以将其替换为单个if。

**Bad**

```go
var a int
if b {
  a = 100
} else {
  a = 10
}
```

vs.

**Good**

```go
a := 10
if b {
  a = 100
}
```

#### 顶层变量声明

在顶层，使用标准var关键字。请勿指定类型，除非它与表达式的类型不同。

**Bad**

```
var _s string = F()

func F() string { return "A" }
```

vs.

**Good**

```
var _s = F()
// 由于F已经明确了返回一个字符串类型，因此我们没有必要显式指定_s的类型

func F() string { return "A" }
```

如果表达式的类型与所需的类型不完全匹配，请指定类型。

```
type myError struct{}

func (myError) Error() string { return "error" }

func F() myError { return myError{} }

var _e error = F()
// F返回一个myError类型的实例，但是我们要error类型
```

#### 对于未导出的顶层常量和变量，使用_作为前缀

> 译注：这个是Uber内部的惯用法，目前看并不普适。

在未导出的顶级vars和consts， 前面加上前缀_，以使它们在使用时明确表示它们是全局符号。

例外：未导出的错误值，应以err开头。

基本依据：顶级变量和常量具有包范围作用域。使用通用名称可能很容易在其他文件中意外使用错误的值。

**Bad**

```
// foo.go

const (
  defaultPort = 8080
  defaultUser = "user"
)

// bar.go

func Bar() {
  defaultPort := 9090
  ...
  fmt.Println("Default port", defaultPort)

  // We will not see a compile error if the first line of
  // Bar() is deleted.
}
```

vs.

**Good**

```
// foo.go

const (
  _defaultPort = 8080
  _defaultUser = "user"
)
```

#### 结构体中的嵌入

嵌入式类型（例如mutex）应位于结构体内的字段列表的顶部，并且必须有一个空行将嵌入式字段与常规字段分隔开。

**Bad**

```
type Client struct {
  version int
  http.Client
}
```

vs.

**Good**

```
type Client struct {
  http.Client

  version int
}
```

#### 使用字段名初始化结构体

初始化结构体时，几乎始终应该指定字段名称。现在由go vet强制执行。

**Bad**

```
k := User{"John", "Doe", true}
```

vs.

**Good**

```
k := User{
    FirstName: "John",
    LastName: "Doe",
    Admin: true,
}
```

例外：如果有3个或更少的字段，则可以在测试表中省略字段名称。

```
tests := []struct{
}{
  op Operation
  want string
}{
  {Add, "add"},
  {Subtract, "subtract"},
}
```

#### 本地变量声明

如果将变量明确设置为某个值，则应使用短变量声明形式（:=）。

**Bad**

```
var s = "foo"
```

vs.

**Good**

```
s := "foo"
```

但是，在某些情况下，var 使用关键字时默认值会更清晰。例如，声明空切片。

**Bad**

```
func f(list []int) {
  filtered := []int{}
  for _, v := range list {
    if v > 10 {
      filtered = append(filtered, v)
    }
  }
}
```

vs.

**Good**

```
func f(list []int) {
  var filtered []int
  for _, v := range list {
    if v > 10 {
      filtered = append(filtered, v)
    }
  }
}
```

#### nil是一个有效的slice

nil是一个有效的长度为0的slice，这意味着：

- 您不应明确返回长度为零的切片。返回nil 来代替。

**Bad**

```
if x == "" {
  return []int{}
}
```

vs.

**Good**

```
if x == "" {
  return nil
}
```

- 要检查切片是否为空，请始终使用len(s) == 0。不要检查 nil。

**Bad**

```
func isEmpty(s []string) bool {
  return s == nil
}
```

vs.

**Good**

```
func isEmpty(s []string) bool {
  return len(s) == 0
}
```

- 零值切片可立即使用，无需调用make创建。

**Bad**

```
nums := []int{}
// or, nums := make([]int)

if add1 {
  nums = append(nums, 1)
}

if add2 {
  nums = append(nums, 2)
}
```

vs.

**Good**

```
var nums []int

if add1 {
  nums = append(nums, 1)
}

if add2 {
  nums = append(nums, 2)
}
```

#### 缩小变量作用域

如果有可能，尽量缩小变量作用范围。除非它与减少嵌套的规则冲突。

**Bad**

```
err := ioutil.WriteFile(name, data, 0644)
if err != nil {
    return err
}
```

vs.

**Good**

```
if err := ioutil.WriteFile(name, data, 0644); err != nil {
    return err
}
```

如果需要在if之外使用函数调用的结果，则不应尝试缩小范围。

**Bad**

```
if data, err := ioutil.ReadFile(name); err == nil {
  err = cfg.Decode(data)
  if err != nil {
    return err
  }

  fmt.Println(cfg)
  return nil
} else {
  return err
}
```

vs.

**Good**

```
data, err := ioutil.ReadFile(name)
if err != nil {
   return err
}

if err := cfg.Decode(data); err != nil {
  return err
}

fmt.Println(cfg)
return nil
```

#### 避免裸参数

函数调用中的裸参数可能会损害可读性。当参数名称的含义不明显时，请为参数添加C样式注释（/* … */）。

**Bad**

```
// func printInfo(name string, isLocal, done bool)

printInfo("foo", true, true)
```

vs.

**Good**

```
// func printInfo(name string, isLocal, done bool)

printInfo("foo", true /* isLocal */, true /* done */)
```

更好的作法是，将裸bool类型替换为自定义类型，以获得更易读和类型安全的代码。将来，该参数不仅允许两个状态（true/false）。

```
type Region int

const (
  UnknownRegion Region = iota
  Local
)

type Status int

const (
  StatusReady = iota + 1
  StatusDone
  // Maybe we will have a StatusInProgress in the future.
)

func printInfo(name string, region Region, status Status)
```

#### 使用原始字符串字面值，避免转义

Go支持[原始字符串字面值](https://golang.org/ref/spec#raw_string_lit)，可以跨越多行并包含引号。使用这些字符串可以避免更难阅读的手工转义的字符串。

**Bad**

```
wantError := "unknown name:\"test\""
```

vs.

**Good**

```
wantError := `unknown error:"test"`
```

#### 初始化结构体引用

在初始化结构引用时，请使用&T{}代替new(T)，以使其与结构体初始化一致。

**Bad**

```
sval := T{Name: "foo"}

// 不一致
sptr := new(T)
sptr.Name = "bar"
```

vs.

**Good**

```
sval := T{Name: "foo"}

sptr := &T{Name: "bar"}
```

#### 格式化字符串放在Printf外部

如果你为Printf-style函数声明格式字符串，请将格式化字符串放在外面，并将其设置为const常量。

这有助于go vet对格式字符串执行静态分析。

**Bad**

```
msg := "unexpected values %v, %v\n"
fmt.Printf(msg, 1, 2)
```

vs.

**Good**

```
const msg = "unexpected values %v, %v\n"
fmt.Printf(msg, 1, 2)
```

#### 命名Printf样式的函数

声明Printf-style函数时，请确保go vet可以检测到它并检查格式字符串。

这意味着您应尽可能使用预定义的Printf-style函数名称。go vet将默认检查这些。有关更多信息，请参见[Printf系列](https://golang.org/cmd/vet/#hdr-Printf_family)。

如果不能使用预定义的名称，请以f结束选择的名称：Wrapf，而不是Wrap。go vet可以要求检查特定的Printf样式名称，但名称必须以f结尾。

$ go vet -printfuncs = wrapf,statusf

另请参阅”[go vet：Printf家族检查](https://kuzminva.wordpress.com/2017/11/07/go-vet-printf-family-check/)“。

### 模式

#### 测试表

在核心测试逻辑重复时，将表驱动测试与子测试一起使用，以避免重复代码。

**Bad**

```
// func TestSplitHostPort(t *testing.T)

host, port, err := net.SplitHostPort("192.0.2.0:8000")
require.NoError(t, err)
assert.Equal(t, "192.0.2.0", host)
assert.Equal(t, "8000", port)

host, port, err = net.SplitHostPort("192.0.2.0:http")
require.NoError(t, err)
assert.Equal(t, "192.0.2.0", host)
assert.Equal(t, "http", port)

host, port, err = net.SplitHostPort(":8000")
require.NoError(t, err)
assert.Equal(t, "", host)
assert.Equal(t, "8000", port)

host, port, err = net.SplitHostPort("1:8")
require.NoError(t, err)
assert.Equal(t, "1", host)
assert.Equal(t, "8", port)
```

vs.

**Good**

```
// func TestSplitHostPort(t *testing.T)

tests := []struct{
  give     string
  wantHost string
  wantPort string
}{
  {
    give:     "192.0.2.0:8000",
    wantHost: "192.0.2.0",
    wantPort: "8000",
  },
  {
    give:     "192.0.2.0:http",
    wantHost: "192.0.2.0",
    wantPort: "http",
  },
  {
    give:     ":8000",
    wantHost: "",
    wantPort: "8000",
  },
  {
    give:     "1:8",
    wantHost: "1",
    wantPort: "8",
  },
}

for _, tt := range tests {
  t.Run(tt.give, func(t *testing.T) {
    host, port, err := net.SplitHostPort(tt.give)
    require.NoError(t, err)
    assert.Equal(t, tt.wantHost, host)
    assert.Equal(t, tt.wantPort, port)
  })
}
```

测试表使向错误消息添加上下文，减少重复的逻辑以及添加新的测试用例变得更加容易。

我们遵循这样的约定：将结构体切片称为tests。 每个测试用例称为tt。此外，我们鼓励使用give和want前缀说明每个测试用例的输入和输出值。

```
tests := []struct{
  give     string
  wantHost string
  wantPort string
}{
  // ...
}

for _, tt := range tests {
  // ...
}
```

#### 功能选项

功能选项是一种模式，您可以在其中声明一个不透明Option类型，该类型在某些内部结构中记录信息。您接受这些选项的可变编号，并根据内部结构上的选项记录的全部信息采取行动。

将此模式用于您需要扩展的构造函数和其他公共API中的可选参数，尤其是在这些功能上已经具有三个或更多参数的情况下。

**Bad**

```
// package db

func Connect(
  addr string,
  timeout time.Duration,
  caching bool,
) (*Connection, error) {
  // ...
}

// Timeout and caching must always be provided,
// even if the user wants to use the default.

db.Connect(addr, db.DefaultTimeout, db.DefaultCaching)
db.Connect(addr, newTimeout, db.DefaultCaching)
db.Connect(addr, db.DefaultTimeout, false /* caching */)
db.Connect(addr, newTimeout, false /* caching */)
```

vs.

**Good**

```
type options struct {
  timeout time.Duration
  caching bool
}

// Option overrides behavior of Connect.
type Option interface {
  apply(*options)
}

type optionFunc func(*options)

func (f optionFunc) apply(o *options) {
  f(o)
}

func WithTimeout(t time.Duration) Option {
  return optionFunc(func(o *options) {
    o.timeout = t
  })
}

func WithCaching(cache bool) Option {
  return optionFunc(func(o *options) {
    o.caching = cache
  })
}

// Connect creates a connection.
func Connect(
  addr string,
  opts ...Option,
) (*Connection, error) {
  options := options{
    timeout: defaultTimeout,
    caching: defaultCaching,
  }

  for _, o := range opts {
    o.apply(&options)
  }

  // ...
}

// Options must be provided only if needed.

db.Connect(addr)
db.Connect(addr, db.WithTimeout(newTimeout))
db.Connect(addr, db.WithCaching(false))
db.Connect(
  addr,
  db.WithCaching(false),
  db.WithTimeout(newTimeout),
)
```

还可以参考下面资料：

- [Self-referential functions and the design of options](https://commandcenter.blogspot.com/2014/01/self-referential-functions-and-design.html)
- [Functional options for friendly APIs](https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis)