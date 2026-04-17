# 第 71 关：命令行子命令（师兄带你学 Go）

## 🎯 这一关你会学到

- 掌握**子命令模式**的设计思路：用 `os.Args[1]` 识别子命令名称，再把剩余参数交给对应逻辑处理
- 学会用 `flag.NewFlagSet(name, errorHandling)` 为每个子命令创建**独立的 flag 集合**，避免全局污染
- 熟悉在 FlagSet 上调用 `.Bool()` / `.String()` / `.Int()` 注册子命令专属 flag
- 理解为什么解析时要写 `xxxCmd.Parse(os.Args[2:])` 而非 `os.Args[1:]`——第 1 个元素已被消耗掉
- 掌握用 `switch` + `default` 分支处理未知子命令，并以 `os.Exit(1)` 给调用方返回非零退出码

## 🤔 先想一个问题

你每天都在用 `git`——`git commit`、`git push`、`git log`，每个子命令还能跟自己专属的 flag，感觉天经地义。但如果让你自己用 Go 写一个类似风格的命令行工具，你会怎么做？直接一个 `flag.Parse()` 塞到 `main` 里？那很快就会乱成一锅粥。Go 标准库其实早就给你备好了 `flag.NewFlagSet`，让每个子命令拥有独立的 flag 世界，互不干扰。这一关我们就来亲手搭一个 git 风格的命令行骨架。

## 📖 看代码

```go
// `go` 和 `git` 这种命令行工具，都有很多的 *子命令* 。
// 并且每个工具都有一套自己的 flag，比如：
// `go build` 和 `go get` 是 `go` 里面的两个不同的子命令。
// `flag` 包让我们可以轻松的为工具定义简单的子命令。

package main

import (
	"flag"
	"fmt"
	"os"
)

func main() {

	// 我们使用 `NewFlagSet` 函数声明一个子命令，
	// 然后为这个子命令定义一个专用的 flag。
	fooCmd := flag.NewFlagSet("foo", flag.ExitOnError)
	fooEnable := fooCmd.Bool("enable", false, "enable")
	fooName := fooCmd.String("name", "", "name")

	// 对于不同的子命令，我们可以定义不同的 flag。
	barCmd := flag.NewFlagSet("bar", flag.ExitOnError)
	barLevel := barCmd.Int("level", 0, "level")

	// 子命令应作为程序的第一个参数传入。
	if len(os.Args) < 2 {
		fmt.Println("expected 'foo' or 'bar' subcommands")
		os.Exit(1)
	}

	// 检查哪一个子命令被调用了。
	switch os.Args[1] {

	// 每个子命令，都会解析自己的 flag 并允许它访问后续的位置参数。
	case "foo":
		fooCmd.Parse(os.Args[2:])
		fmt.Println("subcommand 'foo'")
		fmt.Println("  enable:", *fooEnable)
		fmt.Println("  name:", *fooName)
		fmt.Println("  tail:", fooCmd.Args())
	case "bar":
		barCmd.Parse(os.Args[2:])
		fmt.Println("subcommand 'bar'")
		fmt.Println("  level:", *barLevel)
		fmt.Println("  tail:", barCmd.Args())
	default:
		fmt.Println("expected 'foo' or 'bar' subcommands")
		os.Exit(1)
	}
}
```

## 🔍 师兄给你逐行拆

### 啥叫子命令？

师弟，你平时用 `git` 的时候，有没有注意到一件事：`git commit`、`git push`、`git log` 这几个命令，虽然都顶着 `git` 这个名字，但它们其实是完全不同的小工具。`git commit` 有 `-m`、`--amend` 这些专属 flag；`git log` 有 `--oneline`、`--graph`、`--author` 这些只有它才认识的参数；`git push` 则有 `--force`、`--tags` 等选项。你要是把 `--oneline` 塞给 `git commit`，它根本不知道你在说啥。

`go` 工具链也是一样的道理。`go build`、`go run`、`go test` 三兄弟，各自都有一套独立的 flag 体系。`go test` 认识 `-v`、`-run`、`-cover`，但 `go build` 压根不搭理这些。每个子命令就像是一个相对独立的小程序，只是借着同一个父命令的壳子对外亮相而已。

师兄给你打个比方：你想象一下外卖 App 里的不同餐厅。虽然都在同一个平台上，但川菜馆有川菜馆的菜单，日料店有日料店的套餐，你不能拿着川菜馆的菜单去日料店点餐，反过来也一样。子命令模式的核心就是：一个主命令下挂着一组各自独立的小工具，共用一个入口，但内部完全解耦。Go 标准库的 `flag` 包里的 `NewFlagSet`，就是专门给这种设计方式准备的好工具。

### NewFlagSet：给每个子命令开一个工作台

师弟，你有没有遇到过这种情况：程序里有 `add`、`delete` 两个子命令，各自需要不同的参数，但一不小心 flag 就打架了？这正是 `flag.NewFlagSet` 要解决的问题。

你可以把 `flag.NewFlagSet` 理解成**给每个子命令单独开一张工作台**。每张工作台上只摆自己的工具，互相不干扰。它的函数签名长这样：

```go
func NewFlagSet(name string, errorHandling ErrorHandling) *FlagSet
```

两个参数各有分工。**第一个参数 `name`** 是这个 FlagSet 的名字，通常就传子命令的名称，比如 `"add"` 或 `"delete"`。它的作用是在解析出错时，帮你在错误信息里标明是哪个子命令出了问题，方便定位。**第二个参数 `errorHandling`** 控制出错后的行为，日常开发最常用 `flag.ExitOnError`，意思是一旦遇到不认识的 flag，就直接退出程序并打印提示；如果你想自己处理错误，也可以选 `flag.ContinueOnError`，让它把错误返回给你。

那么，它和直接调用 `flag.String`、`flag.Int` 这些全局函数有什么区别呢？秘密在于：**全局函数背后共用的是一个默认 FlagSet**，也就是 `flag.CommandLine`。所有用全局函数注册的 flag 都挤在同一张工作台上，子命令一多，名字冲突和逻辑混乱几乎是必然的。而 `flag.NewFlagSet` 让你为每个子命令创建独立实例，用 `fs.String`、`fs.Int` 把 flag 注册到各自的 FlagSet 上，解析时调用 `fs.Parse(args)` 只处理属于自己的参数，完全隔离。

一句话总结：全局 flag 适合小脚本图省事，只要你的程序有多个子命令，就应该用 `flag.NewFlagSet` 给每个子命令开一张专属工作台，这才是规范且可维护的做法。

### 子命令声明 flag 的方法

拿到 `fooCmd` 这个 `FlagSet` 之后，下一步就是往上面挂具体的 flag。方法名你看着会觉得眼熟——`fooCmd.Bool`、`fooCmd.String`、`fooCmd.Int`，和 `flag` 包顶层的 `flag.Bool`、`flag.String`、`flag.Int` 长得一模一样，连参数签名都没变：第一个参数是 flag 的名字，第二个是默认值，第三个是帮助文本，返回值同样是对应类型的指针。

```go
verbose := fooCmd.Bool("verbose", false, "输出详细日志")
output  := fooCmd.String("output", "result.txt", "输出文件路径")
retry   := fooCmd.Int("retry", 3, "最大重试次数")
```

用法和顶层函数几乎一样，解析完之后通过解引用 `*verbose`、`*output`、`*retry` 就能拿到用户传入的值。

师弟你可以这样理解：`flag` 包的顶层函数操作的是一个全局默认的 `FlagSet`，就像一家连锁餐厅统一用总部的点餐小程序；而 `fooCmd` 和 `barCmd` 各自是独立的 `FlagSet`，就好比两家分开经营的店，各有自己的点餐小程序，你在甲店下单的选项，乙店的系统根本看不到。声明在 `fooCmd` 上的 `-verbose` 只属于 `foo` 子命令，`bar` 子命令的 `FlagSet` 完全不知道这个 flag 的存在，两边互不干扰，这正是我们引入 `FlagSet` 的核心目的。

### 派发：靠 os.Args 的第 1 位决定走哪条分支

师弟，上一节我们说完了 `os.Args` 是什么，这一节来聊聊程序怎么根据它做「派发」。

你可以把整个过程想象成去吃饭：你先得选定一家餐厅，才能坐进去看那家餐厅的菜单。子命令也是一样的道理——程序必须先读 `os.Args[1]`，认出用户想进哪个「餐厅」，然后才把剩余参数交给那个子命令去处理。

先来看一段典型的派发代码：

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    if len(os.Args) < 2 {
        fmt.Println("用法：mytool <子命令> [参数...]")
        fmt.Println("可用子命令：add、del、list")
        os.Exit(1)
    }

    switch os.Args[1] {
    case "add":
        runAdd(os.Args[2:])
    case "del":
        runDel(os.Args[2:])
    case "list":
        runList(os.Args[2:])
    default:
        fmt.Fprintf(os.Stderr, "未知子命令：%s\n", os.Args[1])
        os.Exit(1)
    }
}
```

代码逻辑分三步走。第一步：先判断 `len(os.Args) < 2`，如果用户什么参数都没给，就打印一段使用提示并调用 `os.Exit(1)` 干净退出，绝不让程序带着空切片继续跑下去。第二步：用 `switch` 对 `os.Args[1]` 做精准匹配，每个 `case` 对应一个子命令名，匹配到哪个就调用哪个处理函数。第三步：把 `os.Args[2:]` 切片传进去，子命令函数拿到的就是属于自己的那份参数，包括它自己的 flag 和位置参数。

这里有三个下标要牢记。`os.Args[0]` 永远是程序本身的路径，比如 `./mytool`，你在派发逻辑里根本用不到它。`os.Args[1]` 才是用户敲的第一个词，也就是子命令名，这是派发的核心依据。`os.Args[2]` 及之后的部分，才是该子命令自己需要解析的 flag 和位置参数，传给子命令处理函数就好。

记住「先选餐厅再看菜单」这个比喻，派发层只负责认门牌，具体怎么点菜是各个子命令自己的事，层次清晰，代码也好维护。

### 各自 Parse：只喂自己的那份参数

师弟，上一步我们已经用 `os.Args[1]` 判断出要走哪条子命令分支，接下来每个子命令的 `FlagSet` 都得自己动手调用 `Parse`，而且只能把属于自己的那段参数喂进去。

你可以这样理解：你今天同时在两家店下了外卖，A 店的订单号绝对不能递给 B 店的收银台，否则人家根本找不到单子，只会一脸懵地报错。`os.Args` 就是你今天所有的订单信息，`os.Args[0]` 是你的手机号（程序名），`os.Args[1]` 是你选的那家店（子命令名），从 `os.Args[2:]` 往后才是这家店真正需要处理的订单明细。所以正确写法是：

```go
fooCmd.Parse(os.Args[2:])
```

千万不要把整个 `os.Args` 塞进去。一旦你这么做，`FlagSet` 会试图把 `os.Args[0]`（程序名）和 `os.Args[1]`（子命令名）当成 flag 去解析，立刻就会疯狂报错，日志刷屏，让你怀疑人生。

`Parse` 跑完之后，你之前用 `fooCmd.String`、`fooCmd.Int` 等方法拿到的指针就可以安全解引用了，直接 `*fooName` 就能读到用户传进来的值。除了具名 flag，如果用户还传了一些没有 `--key` 前缀的裸参数，调用 `fooCmd.Args()` 就能把它们作为字符串切片全部拿回来，非常方便。

记住这条铁律：去哪家店，就把那家店的订单号交给那家店，多一张少一张都会出乱子。每个 `FlagSet` 只负责解析自己那段，职责清晰，程序才能稳稳当当地跑起来。

### 兜底：default 分支和非零退出码

小师弟，咱们写命令行工具的时候，用户输入的子命令五花八门，总有你没料到的情况。这时候 `switch` 的 `default` 分支就是那道最后的防线，专门接住所有未匹配到的子命令。

通常做法是这样的：在 `default` 里先用 `fmt.Fprintf(os.Stderr, ...)` 把帮助信息打印到标准错误流，告诉用户支持哪些子命令、怎么正确使用，然后紧跟一句 `os.Exit(1)`，带着非零退出码退出进程。为什么一定要非零退出码？因为 CI 流水线、shell 脚本、Makefile 都依赖进程退出码来判断任务是否成功。退出码为 `0` 意味着一切正常，非零则意味着出了问题。如果你默默地用 `os.Exit(0)` 收场，上游调用方根本感知不到失败，排查问题的时候就会抓瞎。

这里有个坑要提醒你：`os.Exit` 是直接终止进程的，**不会触发任何 `defer`**。所以如果你在函数里注册了 `defer` 做资源清理，千万不要指望 `os.Exit` 之后它还能跑到。如果清理工作很重要，就在调用 `os.Exit` 之前手动完成，或者用 `return` 加错误码的方式把退出逻辑推到 `main` 函数里统一处理。

师兄最想给你传递的一个习惯是：**失败要大声报警，不要默默吞错**。程序遇到非预期输入，就应该清晰地告诉用户哪里出了问题，然后以明确的失败状态退出，这才是对调用方负责任的做法。

## 🏃 跑一下试试

先编译出可执行文件，然后依次跑以下四个场景。

**场景 1：使用 `foo` 子命令**

```shell
./command-line-subcommands foo -enable -name=joe a1 a2
```

```text
subcommand 'foo'
  enable: true
  name: joe
  tail: [a1 a2]
```

`-enable` 是布尔 flag，单独出现即为 `true`；`-name=joe` 赋了字符串值；`a1 a2` 是解析完 flag 之后剩下的位置参数，通过 `fooCmd.Args()` 拿到。

**场景 2：使用 `bar` 子命令**

```shell
./command-line-subcommands bar -level=8 a1
```

```text
subcommand 'bar'
  level: 8
  tail: [a1]
```

`bar` 只注册了 `-level` 整数 flag，跟 `foo` 的 flag 完全隔离——你在这里写 `-enable` 会直接报错退出，因为 `barCmd` 根本不认识这个 flag。

**场景 3：不传任何子命令**

```shell
./command-line-subcommands
```

```text
expected 'foo' or 'bar' subcommands
```

`main` 里先检查 `len(os.Args) < 2`，条件成立就打印提示并 `os.Exit(1)`，不会走到 `switch`。

**场景 4：传入未知子命令**

```shell
./command-line-subcommands blah
```

```text
expected 'foo' or 'bar' subcommands
```

`switch` 的 `default` 分支兜底，同样打印提示并退出。两种

## 💡 师兄的碎碎念

**师兄碎碎念**

**`ExitOnError` vs `ContinueOnError`**：`NewFlagSet` 第二个参数控制解析出错时的行为。`ExitOnError` 是最常用的选择——遇到非法 flag 直接打印用法并 `os.Exit(2)`，用户体验干净利落。`ContinueOnError` 则会把错误以 `error` 返回，让你自己决定怎么处理，适合需要在测试里捕获错误信息的场景。日常写工具脚本用 `ExitOnError` 就够了，别过度设计。

**为什么是 `os.Args[2:]` 而不是 `os.Args[1:]`**：`os.Args[0]` 是程序名，`os.Args[1]` 是子命令名（比如 `foo`），这两个信息在 `switch` 里已经被消耗掉了。如果把 `os.Args[1:]` 交给 `fooCmd.Parse`，它会看到第一个元素是 `foo` 这个字符串，解析时会把它当成一个未知 flag 或位置参数，结果不符合预期。记住：**谁消耗了哪一段，后面就从下一段开始**。

**多子命令时怎么组织更好**：本关的 `switch` 写法在两三个子命令时没问题，但子命令一多，`main` 函数就会膨胀得很难看。进阶做法是把每个子命令封装成独立的结构体或包，`main` 只做注册和派发。如果你的工具已经有五个以上子命令，可以考虑引入 **cobra** 库，它帮你把这套模式做到了极致，但那是后话，先把标准库玩熟。

**debug 第一步：先打印 `os.Args`**：遇到 flag 解析行为诡异时，第一件事是在 `main` 开头加一行 `fmt.Println(os.Args)`，把实际接收到的参数序列打出来，很多时候你会发现 shell 的引号或转义已经替你

## 🎓 知识点清单

- [ ] 已理解 `flag.NewFlagSet` 与全局 `flag` 包的区别，明白为什么每个子命令需要独立的 FlagSet
- [ ] 能说出 `os.Args[1]` 用来派发子命令、`os.Args[2:]` 用来传给子命令解析的原因
- [ ] 亲手跑通了 `foo` 和 `bar` 两个子命令，观察到各自的 flag 和剩余参数输出
- [ ] 故意不传子命令或传未知子命令，确认触发了帮助提示和 `default` 分支的 `os.Exit(1)`
- [ ] 理解 `ExitOnError` 模式在 flag 解析出错时会直接退出程序
- [ ] 能描述在项目变大后如何把子命令封装成独立结构体或引入 cobra 库来管理

## ➡️ 下一关

下一关我们把目光从命令行参数转向程序运行环境本身——学习如何读取和使用操作系统的环境变量，快来看 [第 72 关：环境变量](../72-environment-variables/README.md)，掌握十二因子应用配置的第一步。
