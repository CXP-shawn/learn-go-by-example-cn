# 第 76 关：生成进程（师兄带你学 Go）

## 🎯 这一关你会学到

- 学会用 `os/exec` 包在 Go 程序里启动并控制外部子进程
- 掌握 `exec.Command` 构造命令、`Output()` 一次性获取输出的基本用法
- 理解参数必须分开传入切片、不能拼成一整段字符串的原因
- 学会用 `StdinPipe` / `StdoutPipe` 手动操控子进程的标准输入输出
- 掌握 `Start` + `Wait` 组合实现异步启动子进程的模式
- 了解如何借助 `bash -c` 执行带 shell 特性（管道、展开）的复杂命令

## 🤔 先想一个问题

师弟，你有没有在写脚本的时候想着：要是能在 Go 里直接调一句 `grep` 或者 `ffmpeg` 就好了，不用自己重新实现那一堆逻辑。或者你用 Python 的时候用过 `subprocess`，觉得这玩意儿真香，想知道 Go 里有没有同款。答案是有的，而且比你想象的要清爽得多——这一关我们就来聊聊怎么从 Go 程序里把外部命令给「召唤」出来跑一圈。

## 📖 看代码

```go
// 有时，我们的 Go 程序需要生成其他的、非 Go 的进程。

package main

import (
	"fmt"
	"io"
	"os/exec"
)

func main() {

	// 我们将从一个简单的命令开始，没有参数或者输入，仅打印一些信息到标准输出流。
	// `exec.Command` 可以帮助我们创建一个对象，来表示这个外部进程。
	dateCmd := exec.Command("date")

	// `.Output` 是另一个帮助函数，常用于处理运行命令、等待命令完成并收集其输出。
	// 如果没有错误，`dateOut` 将保存带有日期信息的字节。
	dateOut, err := dateCmd.Output()
	if err != nil {
		panic(err)
	}
	fmt.Println("> date")
	fmt.Println(string(dateOut))

	// 下面我们将看看一个稍复杂的例子，
	// 我们将从外部进程的 `stdin` 输入数据并从 `stdout` 收集结果。
	grepCmd := exec.Command("grep", "hello")

	// 这里我们明确的获取输入/输出管道，运行这个进程，
	// 写入一些输入数据、读取输出结果，最后等待程序运行结束。
	grepIn, _ := grepCmd.StdinPipe()
	grepOut, _ := grepCmd.StdoutPipe()
	grepCmd.Start()
	grepIn.Write([]byte("hello grep\ngoodbye grep"))
	grepIn.Close()
	grepBytes, _ := io.ReadAll(grepOut)
	grepCmd.Wait()

	// 上面的例子中，我们忽略了错误检测，
	// 当然，你也可以使用常见的 `if err != nil` 方式来进行错误检查。
	// 我们只收集了 `StdoutPipe` 的结果，
	// 但是你可以使用相同的方法收集 `StderrPipe` 的结果。
	fmt.Println("> grep hello")
	fmt.Println(string(grepBytes))

	// 注意，在生成命令时，我们需要提供一个明确描述命令和参数的数组，而不能只传递一个命令行字符串。
	// 如果你想使用一个字符串生成一个完整的命令，那么你可以使用 `bash` 命令的 `-c` 选项：
	lsCmd := exec.Command("bash", "-c", "ls -a -l -h")
	lsOut, err := lsCmd.Output()
	if err != nil {
		panic(err)
	}
	fmt.Println("> ls -a -l -h")
	fmt.Println(string(lsOut))
}
```

## 🔍 师兄给你逐行拆

### os/exec 包是什么，为什么要用它

师弟，在写实际项目的时候，你会发现有一类需求非常常见：明明系统里已经有现成的工具（比如 `ffmpeg`、`grep`、`git`、`convert`），没必要在 Go 里重新造一遍，直接调用就好。这时候 `os/exec` 包就是你的好朋友。它是 Go 标准库提供的「外部进程启动器」，让你可以在 Go 程序里像在终端里一样去运行任意命令，并且拿到它的输出、控制它的输入、等待它结束。你可以把它理解成 Go 版本的 Python `subprocess` 或者 Node 里的 `child_process`，思路是完全一样的，只是语法换了一套。

### exec.Command：先构造再执行

启动外部命令的第一步是构造一个「命令对象」，用的是 `exec.Command` 函数。它的签名大概长这样：第一个参数是可执行文件名（或完整路径），后面的参数是命令行参数，每个参数单独一个字符串。

比如想运行 `echo hello world`，你应该写：

```go
cmd := exec.Command("echo", "hello", "world")
```

注意，这里调用完 `exec.Command` 之后并不会立刻执行任何东西，它只是帮你建了一个 `*exec.Cmd` 对象，记录了你想运行什么、带什么参数。真正的执行要等你调用后续的方法。

### Output：最简单的「一把梭」用法

如果你只是想跑一个命令、拿到它的标准输出、不需要跟它交互，那 `Output()` 方法就够用了。它会在内部帮你 `Start()` + `Wait()`，等命令跑完之后把所有 stdout 内容作为字节切片返回。

```go
cmd := exec.Command("date")
out, err := cmd.Output()
if err != nil {
    log.Fatal(err)
}
fmt.Println(string(out))
```

这是最干净的写法，适合「我只要结果，不管过程」的场景，比如读取系统时间、查询某个命令的版本号、跑个小脚本拿返回值之类的。

### 参数为什么必须分开成切片，不能传整段字符串

这是新手最常犯的错误，师兄专门拎出来说。有同学会这样写：

```go
// 错误示范！
cmd := exec.Command("echo hello world")
```

把整条命令作为一个字符串传进去，结果 Go 会尝试去找一个文件名叫做 `"echo hello world"` 的可执行文件，当然找不到，直接报错。

正确的姿势是把命令名和每个参数分别作为独立字符串，就像这样：

```go
cmd := exec.Command("echo", "hello", "world")
```

这是因为 `os/exec` 不经过 shell 解析，它直接把你的参数列表原样传给操作系统的 `exec` 系统调用。好处是安全——没有 shell 注入的风险；坏处是你没法直接写带管道符、通配符的命令。这个问题我们后面会解决。

### StdinPipe 和 StdoutPipe：手动接管管道

有时候你需要跟子进程「双向通信」——比如你要往它的 stdin 喂数据，同时读它的 stdout。这时候就需要用 `StdinPipe()` 和 `StdoutPipe()`，手动拿到对应的管道。

以 `grep` 为例，我们想从 Go 代码里把几行字塞给 grep，然后读它的过滤结果：

```go
cmd := exec.Command("grep", "foo")

stdinPipe, _ := cmd.StdinPipe()

cmd.Start()

fmt.Fprintln(stdinPipe, "foo")
fmt.Fprintln(stdinPipe, "bar")
fmt.Fprintln(stdinPipe, "baz foo")
stdinPipe.Close() // 关闭 stdin，告诉 grep 没有更多输入了

cmd.Wait()
```

这里有个细节很重要：写完数据之后必须手动 `Close()` stdin 管道，否则 `grep` 会一直等待下一行输入，永远不退出，你的程序就卡死了。这跟你和朋友打电话，说完了要主动挂断是一个道理，不挂断对方不知道你讲完了。

### Start + Wait：异步启动，手动等待

`Output()` 是同步的，整个过程必须等命令跑完才能继续。但很多时候你想启动一个进程，自己先去做别的事，最后再来「收尸」，这时候就用 `Start()` + `Wait()` 的组合。

`Start()` 会立刻返回，子进程在后台跑。之后你可以做任何事。等你想等它结束了，调 `Wait()`，它会阻塞直到子进程退出，并返回错误信息（如果有的话）。

```go
cmd := exec.Command("sleep", "2")
cmd.Start()
fmt.Println("子进程已经启动，我先去泡个茶")
cmd.Wait()
fmt.Println("子进程跑完了")
```

这个模式在你需要同时启动多个子进程、或者需要给子进程塞数据再等它处理完的场景里非常常用。

### bash -c 技巧：让 shell 帮你处理复杂命令

前面说了，`os/exec` 不经过 shell 解析，所以管道符 `|`、通配符 `*`、变量展开这些 shell 特性都没法直接用。但实际工作里经常需要跑类似 `cat access.log | grep ERROR | wc -l` 这样的命令，怎么办？

最简单的办法：把整条命令交给 bash 去解析，让 bash 处理那些 shell 特性：

```go
cmd := exec.Command("bash", "-c", "ls | grep go")
out, _ := cmd.Output()
fmt.Println(string(out))
```

这里 `-c` 是让 bash 接受后面的字符串作为命令来执行，相当于你在终端里敲 `bash -c "ls | grep go"`。这样管道符就交给 bash 去处理，Go 只负责启动 bash 就好了。

需要注意的是，这种方式会引入 shell 注入的潜在风险。如果 `bash -c` 后面的字符串拼接了用户输入，一定要做好转义或者校验，否则后果自负。在可信环境（比如内部脚本、固定命令）里用 `bash -c` 是完全没问题的，只是别把用户输入不加处理地塞进去。

### 错误处理：不要忽略，要读懂

最后说一句：所有的 `err` 都要检查，不要偷懒用 `_` 丢掉。`exec.Command` 系列的错误信息通常很直接，比如 `exec: "xxx": executable file not found in $PATH` 就是找不到命令，`exit status 1` 就是命令跑完但返回了非零退出码。如果你想拿到命令的 stderr 输出，可以用 `cmd.CombinedOutput()` 代替 `Output()`，它会把 stdout 和 stderr 合并返回，排查问题的时候非常好用。

## 🏃 跑一下试试

把示例代码保存为 `spawning-processes.go`，然后在终端执行：

```shell
go run spawning-processes.go
```

你会看到类似下面的输出：

```text
--> Running grep
foo
baz
--> Running echo
   echo   with   spaces
--> Running ls -a -l
total 8
drwxr-xr-x  2 user user   60 Jun  1 10:00 .
drwxr-xr-x 10 user user 4096 Jun  1 09:55 ..
-rw-r--r--  1 user user 1234 Jun  1 10:00 spawning-processes.go
```

首先是 `grep` 的部分：我们往它的 stdin 塞了三行字符串，它按照匹配规则只吐出符合条件的两行，说明管道已经正确接上了。接着是 `echo`，注意输出里那串空格是被原样保留的，这说明参数确实是按字符串切片传进去的，没有经过 shell 的空格折叠处理。最后是 `ls -a -l`，直接拿到了当前目录的文件列表，`Output()` 把所有 stdout 打包成字节切片交回给我们，转成字符串打印出来就是这个效果。

如果你的机器上没有 `grep` 或者 `ls`（比如在纯 Windows 环境下跑），部分命令会报错，建议在 Linux / macOS 或者 WSL 里测试，体验更顺滑。

## 💡 师兄的碎碎念

**师兄碎碎念——几个容易踩的坑**

**坑一：命令找不到，记得检查 PATH。**

有同学跑 `exec.Command("python3", "--version")` 报错说找不到命令，结果折腾半天发现是自己的 `python3` 装在某个虚拟环境里，系统 PATH 里根本没有。`exec.Command` 在内部调用的是 `exec.LookPath`，它只按当前进程的 PATH 环境变量去找可执行文件。解决办法要么传绝对路径，要么在启动 Go 程序之前确认 PATH 配置正确，别想当然觉得系统一定能找到。

**坑二：Output 和 Start+Wait 不能混用。**

`Output()` 内部其实就是帮你调了 `Start()` 再调 `Wait()`，所以如果你先 `StdoutPipe()`，再去调 `Output()`，会直接报错，因为 `Output()` 会尝试自己接管 stdout，但管道已经被你提前拿走了，两边打架。记住：要么全交给 `Output()` 一把梭，要么自己用管道 + `Start()` + `Wait()` 手动控制，二者只能选一套。

**坑三：Wait 一定要调，否则资源泄漏。**

用 `Start()` 启动了子进程之后，就算你不关心它的返回值，也一定要在某个时机调 `Wait()`，否则子进程变成僵尸进程，文件描述符也不会释放。在写长期运行的服务时这个问题会慢慢把你的 fd 耗光，线上排查起来非常头疼，所以养成习惯，`Start` 之后必有 `Wait`，就像开门之后一定要关门一样。

## 🎓 知识点清单

- [ ] 能用 `exec.Command` 构造一个外部命令对象并调用 `Output()` 拿到 stdout
- [ ] 理解为什么参数要拆成多个字符串，不能把整条命令塞成一个字符串
- [ ] 会用 `StdinPipe` / `StdoutPipe` 手动连接管道做流式处理
- [ ] 会用 `Start` + `Wait` 实现异步启动并等待进程结束
- [ ] 掌握 `bash -c "..."` 技巧来运行带管道符或通配符的 shell 命令
- [ ] 知道如何检查命令执行出错时的错误信息并做合理处理

## ➡️ 下一关

下一关我们要学的是更激进的玩法——直接用 [第 77 关：执行进程](../77-execing-processes/README.md) 里的 `exec.Exec`，让当前 Go 进程直接「变身」成另一个进程，彻底替换自身，感兴趣的话记得跟上。
