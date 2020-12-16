---
title: "使用go构建命令行应用"
date: 2020-12-16T12:48:26+08:00
draft: false
original: true
categories: 
  - Golang
tags: 
  - cmdline
---

众所周知，Golang编写的程序都会build成一个可执行程序，很简洁，不像Java要应用一堆的jar包

大家也都看到很多大部分命令行程序都是用flag对程序进行配置，我们写程序代码的时候就需要处理命令行中的flag，如果我们自己来解析很麻烦，所以就已经有大神开源了相应的包，这里是用的是[cli](https://github.com/urfave/cli)

### 快速开始

[cli](https://github.com/urfave/cli)的设计哲学是API应该有趣且简洁的，所以[cli](https://github.com/urfave/cli)应用只需要在main方法中一行代码就能完成

<!--more-->


```go
package main

import (
	"github.com/urfave/cli/v2"
	"os"
)

func main() {
	(&cli.App{}).Run(os.Args)
}
```

运行结果

```
$ go run main.go
NAME:
   main - A new cli application

USAGE:
   main [global options] command [command options] [arguments...]

COMMANDS:
   help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --help, -h  show help (default: false)
```

可以看到运行结果中显示简单的帮助信息，但是这并没有多大的用处。接下来再添加一些action可以执行和一些帮助文档

```go
package main

import (
	"fmt"
	log "github.com/sirupsen/logrus"
	"github.com/urfave/cli/v2"
	"os"
)

func main() {
	app := cli.App{
		Name:  "boom",
		Usage: "make an explosive entrance",
		Action: func(c *cli.Context) error {
			fmt.Println("boom! I say!")
			return nil
		},
	}

	err := app.Run(os.Args)
	if err != nil {
		log.Fatal(err)
	}
}
```

运行结果

```
$ go run main.go
boom! I say!

$ go run main.go --help
NAME:
   boom - make an explosive entrance

USAGE:
   main [global options] command [command options] [arguments...]

COMMANDS:
   help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --help, -h  show help (default: false)
```

现在运行可以看到有了执行结果，而不是显示帮助信息。同时帮助信息也发生了一点点的变化

### 例子

接下来以一个greet的程序示例来看看如果使用[cli](https://github.com/urfave/cli)来完成更多的功能

```go
package main

import (
	"fmt"
	log "github.com/sirupsen/logrus"
	"github.com/urfave/cli/v2"
	"os"
)

func main() {
	app := cli.App{
		Name:  "greet",
		Usage: "fight the loneliness!",
		Action: func(c *cli.Context) error {
			fmt.Println("Hello friend!")
			return nil
		},
	}

	err := app.Run(os.Args)
	if err != nil {
		log.Fatal(err)
	}
}
```

编译成可执行程序，然后运行程序

```
$ go build -o greet

$ greet
Hello friend!

$ greet help
NAME:
   greet - fight the loneliness!

USAGE:
   greet [global options] command [command options] [arguments...]

COMMANDS:
   help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --help, -h  show help (default: false)
```

### 参数

你可以在`cli.Context`中获取到命令行参数

```go
package main

import (
	"fmt"
	log "github.com/sirupsen/logrus"
	"github.com/urfave/cli/v2"
	"os"
)

func main() {
	app := cli.App{
		Name:  "greet",
		Usage: "fight the loneliness!",
		Action: func(c *cli.Context) error {
			fmt.Printf("Hello %q", c.Args().Get(0))
			return nil
		},
	}

	err := app.Run(os.Args)
	if err != nil {
		log.Fatal(err)
	}
}
```

运行一下，并加入参数

```
$ greet golang
Hello "golang"
```

### Flags

设置flag也很简单

```go
package main

import (
	"fmt"
	log "github.com/sirupsen/logrus"
	"github.com/urfave/cli/v2"
	"os"
)

func main() {
	app := cli.App{
		Name:  "greet",
		Usage: "fight the loneliness!",
		Flags: []cli.Flag{
			&cli.StringFlag{
				Name:  "lang",
				Value: "english",
				Usage: "language for greeting",
			},
		},
		Action: func(c *cli.Context) error {
			name := "Bob"
			if c.NArg() > 0 {
				name = c.Args().Get(0)
			}
			if c.String("lang") == "spanish" {
				fmt.Println("Hola ", name)
			} else {
				fmt.Println("Hello", name)
			}
			return nil
		},
	}

	err := app.Run(os.Args)
	if err != nil {
		log.Fatal(err)
	}
}
```

看下有什么变化，发现指定的flag在帮助文档的GLOBAL OPTION部分显示，而且可以通过c.String("name")的方式获取到指定flag的参数值

```
$ greet --help
NAME:
   greet - fight the loneliness!

USAGE:
   greet [global options] command [command options] [arguments...]

COMMANDS:
   help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --lang value  language for greeting (default: "english")
   --help, -h    show help (default: false)
   
$ greet golang
Hello  golang

$ greet --lang=spanish golang
Hola  golang
```

除了c.String("name")方式外，可以指定flag值存放到变量中

```go
package main

import (
	"fmt"
	log "github.com/sirupsen/logrus"
	"github.com/urfave/cli/v2"
	"os"
)

func main() {
	var language string

	app := cli.App{
		Name:  "greet",
		Usage: "fight the loneliness!",
		Flags: []cli.Flag{
			&cli.StringFlag{
				Name:  "lang",
				Value: "english",
				Usage: "language for greeting",
				Destination: &language,
			},
		},
		Action: func(c *cli.Context) error {
			name := "Bob"
			if c.NArg() > 0 {
				name = c.Args().Get(0)
			}
			if language == "spanish" {
				fmt.Println("Hola ", name)
			} else {
				fmt.Println("Hello ", name)
			}
			return nil
		},
	}

	err := app.Run(os.Args)
	if err != nil {
		log.Fatal(err)
	}
}
```

试试刚才运行的命令会得到同样的结果

#### 占位符值

有时在帮助文案中想使用指定flag的值，可以使用反引号的占位符

```go
package main

import (
	log "github.com/sirupsen/logrus"
	"github.com/urfave/cli/v2"
	"os"
)

func main() {
	app := cli.App{
		Name:  "greet",
		Usage: "fight the loneliness!",
		Flags: []cli.Flag{
			&cli.StringFlag{
				Name:    "config",
				Aliases: []string{"c"},
				Usage:   "Load configuration from `FILE`",
			},
		},
	}

	err := app.Run(os.Args)
	if err != nil {
		log.Fatal(err)
	}
}
```

看看帮助文档内容

```
$ greet --help
NAME:
   greet - fight the loneliness!

USAGE:
   greet [global options] command [command options] [arguments...]

COMMANDS:
   help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --config FILE, -c FILE  Load configuration from FILE
   --help, -h              show help (default: false)
```

可以看到出现了

```
--config FILE, -c FILE   Load configuration from FILE
```

> 注意：只有第一个反引号占位符有效，后续的反引号会按原样显示

#### 可选名称

你可能需要设置一个可选的参数名(或短flag)，可以提供一个逗号分割的列表给`Name`当作可选名称

```go
package main

import (
	"fmt"
	log "github.com/sirupsen/logrus"
	"github.com/urfave/cli/v2"
	"os"
)

func main() {
	var language string

	app := cli.App{
		Name:  "greet",
		Usage: "fight the loneliness!",
		Flags: []cli.Flag{
			&cli.StringFlag{
				Name:  "lang",
				Aliases: []string{"l"},
				Value: "english",
				Usage: "language for greeting",
				Destination: &language,
			},
		},
		Action: func(c *cli.Context) error {
			name := "Bob"
			if c.NArg() > 0 {
				name = c.Args().Get(0)
			}
			if language == "spanish" {
				fmt.Println("Hola ", name)
			} else {
				fmt.Println("Hello ", name)
			}
			return nil
		},
	}

	err := app.Run(os.Args)
	if err != nil {
		log.Fatal(err)
	}
}
```

这样就可以使用`--lang spanish`或`-l spanish`指定flag的值

看一下帮助文档的变化并使用别名运行试试

```
$ greet --help
NAME:
   greet - fight the loneliness!

USAGE:
   greet [global options] command [command options] [arguments...]

COMMANDS:
   help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --lang value, -l value  language for greeting (default: "english")
   --help, -h              show help (default: false)
   
$ greet golang
Hello  golang   

$ greet --lang spanish golang
Hola  golang

$ greet -l spanish golang
Hola  golang
```

#### 顺序

flag和command默认是按定义的顺序显示的。然而可以使用`FlagsByName` or `CommandsByName`进行重新排序

```go
package main

import (
	log "github.com/sirupsen/logrus"
	"github.com/urfave/cli/v2"
	"os"
	"sort"
)

func main() {
	app := cli.App{
		Name:  "greet",
		Usage: "fight the loneliness!",
		Flags: []cli.Flag{
			&cli.StringFlag{
				Name:  "lang",
				Value: "english",
				Usage: "Language for the greeting",
			},
			&cli.StringFlag{
				Name:  "config, c",
				Usage: "Load configuration from `FILE`",
			},
		},
		Commands: []*cli.Command{
			{
				Name:    "complete",
				Aliases: []string{"c"},
				Usage:   "complete a task on the list",
				Action: func(c *cli.Context) error {
					return nil
				},
			},
			{
				Name:    "add",
				Aliases: []string{"a"},
				Usage:   "add a task to the list",
				Action: func(c *cli.Context) error {
					return nil
				},
			},
		},
	}

	sort.Sort(cli.FlagsByName(app.Flags))
	sort.Sort(cli.CommandsByName(app.Commands))

	err := app.Run(os.Args)
	if err != nil {
		log.Fatal(err)
	}
}
```

没有排序前

```
greet h
NAME:
   greet - fight the loneliness!

USAGE:
   greet [global options] command [command options] [arguments...]

COMMANDS:
   complete, c  complete a task on the list
   add, a       add a task to the list
   help, h      Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --lang value   Language for the greeting (default: "english")
   --config FILE  Load configuration from FILE
   --help, -h     show help (default: false)
```

排序后

```
greet h
NAME:
   greet - fight the loneliness!

USAGE:
   greet [global options] command [command options] [arguments...]

COMMANDS:
   add, a       add a task to the list
   complete, c  complete a task on the list
   help, h      Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --config FILE  Load configuration from FILE
   --lang value   Language for the greeting (default: "english")
   --help, -h     show help (default: false)
```

#### 环境变量中读取值

也可以使用`EnvVars`从环境变量中读取值作为默认值

```go
package main

import (
	log "github.com/sirupsen/logrus"
	"github.com/urfave/cli/v2"
	"os"
)

func main() {
	app := cli.App{
		Name:  "greet",
		Usage: "fight the loneliness!",
		Flags: []cli.Flag{
			&cli.StringFlag{
				Name:  "lang",
				Aliases: []string{"l"},
				Value: "english",
				Usage: "Language for the greeting",
				EnvVars: []string{"APP_LANG"},
			},
		},
	}

	err := app.Run(os.Args)
	if err != nil {
		log.Fatal(err)
	}
}
```

如果`EnvVars`包含多个string，第一个环境变量将会用来作为默认值

```go
package main

import (
  "log"
  "os"

  "github.com/urfave/cli/v2"
)

func main() {
  app := &cli.App{
    Flags: []cli.Flag{
      &cli.StringFlag{
        Name:    "lang",
        Aliases: []string{"l"},
        Value:   "english",
        Usage:   "language for the greeting",
        EnvVars: []string{"LEGACY_COMPAT_LANG", "APP_LANG", "LANG"},
      },
    },
  }

  err := app.Run(os.Args)
  if err != nil {
    log.Fatal(err)
  }
}
```

#### 从文件中读取值

也可以使用`FilePath`从文件中读取值作为默认值

```go
package main

import (
  "log"
  "os"

  "github.com/urfave/cli/v2"
)

func main() {
  app := cli.NewApp()

  app.Flags = []cli.Flag {
    &cli.StringFlag{
      Name: "password, p",
      Usage: "password for the mysql database",
      FilePath: "/etc/mysql/password",
    },
  }

  err := app.Run(os.Args)
  if err != nil {
    log.Fatal(err)
  }
}
```

> 注意：从文件中读取的默认值优先于环境变量中设置的默认值

#### 必要flag

可以设置`Required`字段为`true`是flag是必须提供的flag。如果不提供这个必须的flag将会报错

```go
package main

import (
	log "github.com/sirupsen/logrus"
	"github.com/urfave/cli/v2"
	"os"
)

func main() {
	app := cli.App{
		Name:  "greet",
		Usage: "fight the loneliness!",
		Flags: []cli.Flag{
			&cli.StringFlag{
				Name:     "lang",
				Aliases:  []string{"l"},
				Value:    "english",
				Usage:    "Language for the greeting",
				EnvVars:  []string{"APP_LANG"},
				Required: true,
			},
		},
	}

	err := app.Run(os.Args)
	if err != nil {
		log.Fatal(err)
	}
}
```

不指定`lang`运行会报错，指定了正常运行

```
greet
NAME:
   greet - fight the loneliness!

USAGE:
   greet [global options] command [command options] [arguments...]

COMMANDS:
   help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --lang value, -l value  Language for the greeting (default: "english") [$APP_LANG]
   --help, -h              show help (default: false)
FATA[0000] Required flag "lang" not set   
```

#### 帮助文档中的默认值

有时指定flag的默认值的文案是很有用的。比如这个默认值不是确定的，而是计算出来的，可以使用`DefaultText`指定默认值的文案

```go
package main

import (
  "log"
  "os"

  "github.com/urfave/cli/v2"
)

func main() {
  app := &cli.App{
    Flags: []cli.Flag{
      &cli.IntFlag{
        Name:    "port",
        Usage:   "Use a randomized port",
        Value: 0,
        DefaultText: "random",
      },
    },
  }

  err := app.Run(os.Args)
  if err != nil {
    log.Fatal(err)
  }
}
```

输出会像这样

```
--port value  Use a randomized port (default: random)
```

#### 优先级

flag的优先级如下(从高到低)：

0. 命令行指定的flag值
1. 环境变量
2. 配置文件
3. flag指定的默认值

### 子命令

子命令可以参考git命令来理解

```go
package main

import (
  "fmt"
  "log"
  "os"

  "github.com/urfave/cli/v2"
)

func main() {
  app := &cli.App{
    Commands: []*cli.Command{
      {
        Name:    "add",
        Aliases: []string{"a"},
        Usage:   "add a task to the list",
        Action:  func(c *cli.Context) error {
          fmt.Println("added task: ", c.Args().First())
          return nil
        },
      },
      {
        Name:    "complete",
        Aliases: []string{"c"},
        Usage:   "complete a task on the list",
        Action:  func(c *cli.Context) error {
          fmt.Println("completed task: ", c.Args().First())
          return nil
        },
      },
      {
        Name:        "template",
        Aliases:     []string{"t"},
        Usage:       "options for task templates",
        Subcommands: []*cli.Command{
          {
            Name:  "add",
            Usage: "add a new template",
            Action: func(c *cli.Context) error {
              fmt.Println("new task template: ", c.Args().First())
              return nil
            },
          },
          {
            Name:  "remove",
            Usage: "remove an existing template",
            Action: func(c *cli.Context) error {
              fmt.Println("removed task template: ", c.Args().First())
              return nil
            },
          },
        },
      },
    },
  }

  err := app.Run(os.Args)
  if err != nil {
    log.Fatal(err)
  }
}
```