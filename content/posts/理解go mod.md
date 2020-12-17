---
title: "go mod实践"
date: 2020-12-16T18:04:54+08:00
draft: false
original: true
categories: 
  - Golang
tags: 
  - gomod
---

### 基础知识

#### 版本号规范

go mod 对版本号的定义是有一定要求的，它要求的格式为 v\<major\>.\<minor\>.\<patch\>，如果 major 版本号大于 1 时，其版本号还需要体现在 Module 名字中。比如 我的项目 github.com/pibigstar/go-demo，如果我的版本号增长到 v2.x.x 时，我的 Module 名字也需要相应的改变为： github.com/pibigstar/go-demo/v2， 有人可能就要问了，我不改可以吗？ 可以的！但是 go mod 会在你依赖的后面打一个 +incompatible 标志

#### 伪版本

我们将项目上传到 github 后，如果不打 tag，或 tag 不符合 v\<major\>.\<minor\>.\<patch\> 这个格式，那么当我们用 go mod 去拉这个项目的时候，就会将 commitId 作为版本号，它的格式大概是 vx.y.z-yyyymmddhhmmss-abcdef格式

<!--more-->
#### indirect 标志

> 我们用 go mod 的时候应该经常会看到 有的依赖后面会打了一个 // indirect 的标识位，这个标识位是表示 间接的依赖。

什么叫间接依赖呢？打个比方，项目 A 依赖了项目 B，项目 B 又依赖了项目 C，那么对项目 A 而言，项目 C 就是间接依赖，这里要注意，并不是所有的间接依赖都会出现在 go.mod 文件中。间接依赖出现在 go.mod 文件的情况，可能符合下面的场景的一种或多种：

* 直接依赖未启用 Go module
* 直接依赖 go.mod 文件中缺失部分依赖

##### 直接依赖未启用 Go module

如下图所示，Module A 依赖 B，但是 B 还未切换成 Module，即没有 go.mod 文件，此时，当使用 go mod tidy 命令更新 A 的 go.mod 文件时，B 的两个依赖 B1 和 B2 将会被添加到 A 的 go.mod 文件中（前提是 A 之前没有依赖 B1 和 B2），并且 B1 和 B2 还会被添加 // indirect 的注释。

![](/理解gomod/f9WLznfssH.png)

那么项目 A 的依赖关系就会变成下面这个样子

```go
require (
    B vx.x.x
    B1 vx.x.x // indirect
    B2 vx.x.x // indirect
)
```

##### go.mod 文件中缺失部分依赖

> 如下图所示，Module B 虽然提供了 go.mod 文件中，但 go.mod 文件中只添加了依赖 B1，那么此时 A 在引用 B 时，则会在 A 的 go.mod 文件中添加 B2 作为间接依赖，B1 则不会出现在 A 的 go.mod 文件中。

![](/理解gomod/1w2MVOkx0Z.png)

此时项目 A 的依赖关系就会变成下面这个样子

```go
require (
    B vx.x.x
    B2 vx.x.x // indirect
)
```

间接依赖出现在 go.mod 中，可以一定程度上说明依赖有瑕疵，要么是其不支持 Go module，要么是其 go.mod 文件不完整。
由于 Go 语言从 v1.11 版本才推出 module 的特性，众多开源软件迁移到 go module 还需要一段时间，在过渡期必然会出现间接依赖，但随着时间的推进，在 go.mod 中出现 //indirect 的机率会越来越低。
出现间接依赖可能意味着你在使用过时的软件，如果有精力的话还是推荐尽快消除间接依赖。可以通过使用依赖的新版本或者替换依赖的方式消除间接依赖。

#### replace 使用

> replace 指替换，它指示编译工具替换 require 指定中出现的包

这里要注意两点：

* replace 只在 main module 里面有效

> 什么叫 main module? 打个比方，项目 A 的 module 用 replace 替换了本地文件，那么当项目 B 引用项目 A 后，项目 A 的 replace 会失效，此时对 replace 而言，项目 A 就是 main module

* replace 指定中需要替换的包及其版本号必须出现在 require 列表中才有效

#### exclude 使用

go.mod 文件中的 exclude 指令用于排除某个包的特定版本，其与 replace 类似，也仅在当前 module 为 main module 时有效，其他项目引用当前项目时，exclude 指令会被忽略。

exclude 指令在实际的项目中很少被使用，因为很少会显式地排除某个包的某个版本，除非我们知道某个版本有严重 bug。 比如指令 exclude github.com/pibigstar/go-demo v1.1.0，表示不使用 v1.1.0 版本。


### 遇到的问题

#### replace只在main module里面有效

在我的项目中，直接引用只有beats这一个项目

```go
import (
	"os"

	"github.com/elastic/beats/v7/x-pack/filebeat/cmd"
)

func main() {
	if err := cmd.Filebeat().Execute(); err != nil {
		os.Exit(1)
	}
}
```

直接运行`go run main.go`，然后就报错了

```
go: github.com/elastic/beats/v7@v7.10.1 requires
	github.com/Shopify/sarama@v0.0.0-00010101000000-000000000000: invalid version: unknown revision 000000000000
```

报错 很明显，就是`github.com/elastic/beats/v7`依赖的`github.com/Shopify/sarama`找不到相应的版本，我打开了`github.com/elastic/beats/v7`的go.mod文件，确认了一下

![](/理解gomod/beatsmod.png)

找不到版本怎么办，出现这种情况有很多种可能，比如`github.com/elastic/beats/v7`在编写的时候，依赖的`github.com/Shopify/sarama`版本还存在，之后`github.com/Shopify/sarama`修改，版本不存在了等等。这些问题都是我们不能控制的，那怎么解决呢？

在`github.com/elastic/beats/v7`项目的go.mod文件中还有replace

![](/理解gomod/beatsreplace.png)

在前面的基础只是一节中，说到过replace至于在main module才能生效，所以我们需要将相应的replace也添加到我们的项目中

> 如果引用的第三方包很多，如果大部分都有这种问题的话，我们自己项目里面的就要写很多的这种replace，真的是蛋疼

#### 出现checksum mismatch

删除go.sum，清空mod缓存，重新下载包

```
$rm go.sum
$go clean -modcache
```

如果还不行，那就把checksum的校验关了吧 

```
GOSUMDB='off'
```

#### case-insensitive import collision

这个问题报错是这样子的

```
go/pkg/mod/github.com/apache/pulsar-client-go@v0.3.0/pulsar/internal/compression/zstd_cgo.go:27:2: case-insensitive import collision: "github.com/datadog/zstd" and "github.com/DataDog/zstd"
```

根据报错信息我们知道因为引入包存在大小写不同的2种包导致

最初我以为是因为`github.com/apache/pulsar-client-go@v0.3.0`这个包导致的，而且也有人提了[issue](https://github.com/apache/pulsar-client-go/issues/379)，但是这个issue中使用的是v0.2.0，v0.3.0已经修复这个问题了，将依赖都改成了`github.com/datadog/zstd`

说明我们项目中还有依赖`github.com/DataDog/zstd`，可以使用`go mod why github.com/DataDog/zstd`来看看`github.com/DataDog/zstd`是怎么引用进来的

```
$ go mod why github.com/DataDog/zstd
# github.com/DataDog/zstd
gitlab.xiaoduoai.com/xxx/xxxxx
github.com/elastic/beats/v7/x-pack/filebeat/cmd
github.com/elastic/beats/v7/x-pack/filebeat/input/default-inputs
github.com/elastic/beats/v7/x-pack/filebeat/input/cloudfoundry
github.com/elastic/beats/v7/x-pack/libbeat/common/cloudfoundry
github.com/elastic/beats/v7/x-pack/libbeat/persistentcache
github.com/dgraph-io/badger/v2
github.com/dgraph-io/badger/v2/y
github.com/DataDog/zstd
```

可以看到`github.com/DataDog/zstd`是由`github.com/dgraph-io/badger/v2`引入的，我也去badger项目确认了一下，它们确实使用的是`github.com/DataDog/zstd`

原以为这个问题使用replace也能解决，类似这样

```
replace (
...
github.com/DataDog/zstd => github.com/DataDog/zstd va.b.c
...
)
```

但是我大意了，如果项目中同时存在多种大小写不同的引用，要么统一，要么就不使用，要不然这种是无解的

由于`github.com/DataDog/zstd`是`github.com/dgraph-io/badger/v2`引入的，要么给`github.com/dgraph-io/badger/v2`提PR修改，要么clone这个仓库，我们自己改了，然后replace成我们自己的仓库

### 参考资料

* [这一次，彻底掌握go mod](https://learnku.com/articles/47737)
* [Go Module 实践中的问题](https://romatic.net/post/gomod/)