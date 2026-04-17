# 第 64 关：行过滤器（师兄带你学 Go）

## 🎯 这一关你会学到

掌握用 bufio.Scanner 包裹 os.Stdin 做逐行流式处理的经典模式——这是 Go 里写命令行行过滤器工具的标准姿势，读一行、处理一行、输出一行，内存占用极低，天生适合 Unix 管道组合。

## 🤔 先想一个问题

Linux 老鸟都爱 grep/sed/awk，一条管道串起来，数据哗哗流过去。Go 完全可以写自己的小工具插进管道链：cat access.log | go run myfilter.go | sort | uniq -c。本关就是打通这条路的钥匙——行过滤器，stdin 进、处理、stdout 出，Unix 哲学 do one thing well 的 Go 实现。

## 📖 看代码

```go
// _行过滤器（line filter）_ 是一种常见的程序类型，
// 它读取 stdin 上的输入，对其进行处理，然后将处理结果打印到 stdout。
// `grep` 和 `sed` 就是常见的行过滤器。

// 这里是一个使用 Go 编写的行过滤器示例，它将所有的输入文字转化为大写的版本。
// 你可以使用这个模式来写一个你自己的 Go 行过滤器。

package main

import (
	"bufio"
	"fmt"
	"os"
	"strings"
)

func main() {

	// 用带缓冲的 scanner 包装无缓冲的 `os.Stdin`，
	// 这为我们提供了一种方便的 `Scan` 方法，
	// 将 scanner 前进到下一个 `令牌`（默认为：下一行）。
	scanner := bufio.NewScanner(os.Stdin)

	for scanner.Scan() {
		// `Text` 返回当前的 token，这里指的是输入的下一行。
		ucl := strings.ToUpper(scanner.Text())

		// 输出转换为大写后的行。
		fmt.Println(ucl)
	}

	// 检查 `Scan` 的错误。
	// 文件结束符（EOF）是可以接受的，它不会被 `Scan` 当作一个错误。
	if err := scanner.Err(); err != nil {
		fmt.Fprintln(os.Stderr, "error:", err)
		os.Exit(1)
	}
}
```

## 🔍 师兄给你逐行拆

好，第 64 关，行过滤器（Line Filter）。这一关非常有 Unix 味道，强烈建议你泡一杯咖啡，慢慢品。

**行过滤器是个啥？**

行过滤器就是一类程序：从标准输入（stdin）一行一行地读数据，对每行做点什么处理，然后把结果写到标准输出（stdout）。你用过的 grep、sed、awk，全都是这个套路。grep 过滤匹配的行，sed 做文本替换，awk 做列操作——本质上都是

好，段 B 来了，直接冲进循环核心。

**for scanner.Scan() 这一行，是整个程序的灵魂。**

Scan() 每次调用，就往前读一行，把内容存到内部缓冲区，同时返回一个 bool。返回 true 代表

段 C：深入 Scanner 与实战场景

Scanner 默认的内部缓冲区大小是 64KB，对大多数日志行已经够用。但如果你处理的是超长行——比如某些数据库慢查询日志或 minified JSON——Scanner 会返回 bufio.ErrTooLong 错误，扫描直接中止。这时候调用 scanner.Buffer(buf, maxSize) 就能解决问题，第一个参数传一个自定义字节切片，第二个参数指定最大允许长度，比如 1024*1024 就是 1MB 上限，非常简单。

如果行长度完全不可预测，或者你不想预先分配大块内存，也可以换用 bufio.Reader。reader.ReadString('\n') 会一直读到换行符为止，底层动态拼接，不怕超长。另一个选择是 reader.ReadLine()，它返回的是字节切片，遇到超长行时 isPrefix 字段会置为 true，提示你还没读完，需要循环拼接剩余部分，稍微麻烦一点，但内存控制更精细。

Scanner 真正强大的地方在于自定义 SplitFunc。默认是 bufio.ScanLines 按行切割，换成 bufio.ScanWords 就变成按空白符切词，适合统计词频或解析以空格分隔的配置文件。换成 bufio.ScanRunes 则逐个 Unicode 字符扫描，处理多语言文本时特别方便。你还可以自己实现 SplitFunc，签名是 func(data []byte, atEOF bool) (advance int, token []byte, err error)，按任意分隔符切割都不在话下，比如以竖线或制表符分隔的数据格式。

来看几个实战场景：

日志脱敏——逐行读取，用 strings.ReplaceAll 或正则把手机号、身份证号替换成星号，再写到新文件，整个流程内存占用极低。

JSON Line 解析——每行一个独立 JSON 对象是日志系统的常见格式，Scanner 逐行读入后交给 json.Unmarshal，干净利落。

CSV 处理——简单场景可以 SplitFunc 切逗号，复杂场景直接用标准库 encoding/csv 的 Reader，底层同样基于 bufio。

grep 简化版——逐行读取标准输入，判断 strings.Contains(line, keyword)，匹配则打印，几十行代码就能复现核心功能，面试手写题的好素材。

掌握这些，行过滤器就从玩具级别升级成了生产可用的工具。

好，第 64 关「行过滤器」深度篇，咱们从「怎么真正接进管道链」一路讲到「并发 worker pool」，系好安全带。

---

**一、接入 Unix 管道链**

行过滤器的灵魂是：它既是某人的下游，又是某人的上游。测试阶段你会写：

```
echo -e "hello\nworld\nfoo" | go run filter.go
```

`go run` 会编译后立刻执行，stdin 就是 echo 的输出，stdout 就是终端。没问题就继续往右接：

```
echo -e "hello\nworld\nfoo" | go run filter.go | grep o
```

部署时千万别留着 `go run`，它每次都重新编译，浪费几百毫秒，在高频管道里是灾难。老老实实 `go build -o myfilter .`，然后：

```
cat bigfile.txt | ./myfilter | sort | uniq -c
```

二进制冷启动几毫秒，才是管道该有的速度。

---

**二、stdout 与 stderr 的分家——这个太重要了**

新手最常犯的错误：把调试信息、错误提示一股脑全用 `fmt.Println` 输出，结果下游 `awk` 或 `jq` 读到奇怪的一行直接崩掉，然后对着屏幕一脸懵。

规矩只有一条：**数据走 stdout，消息走 stderr**。

```go
fmt.Println(processedLine)          // 给下游管道
fmt.Fprintln(os.Stderr, "警告：...") // 给人看，不污染数据流
```

stderr 默认直接打到终端，不会被管道拦截，用户能看到错误，下游程序却对此毫不知情，皆大欢喜。如果你真的想把 stderr 也重定向，shell 里加 `2>/dev/null` 或 `2>&1` 是用户自己的事，跟你的程序无关。

---

**三、返回码语义——shell 的握手信号**

shell 脚本里有 `&&` 和 `||`，它们靠的就是上一条命令的退出码。

```bash
./myfilter < input.txt && echo "成功" || echo "出错了"
```

Go 程序正常走完 main 函数，退出码自动是 0，表示成功。遇到无法处理的错误：

```go
if err := scanner.Err(); err != nil {
    fmt.Fprintln(os.Stderr, "读取失败:", err)
    os.Exit(1)
}
```

`os.Exit(1)` 告诉 shell「我挂了」，`||` 后面的补救逻辑就会触发。别用 `panic`，那会把 goroutine 栈全喷到 stderr，吓到运维同事。

---

**四、信号处理——优雅地死**

用户按 Ctrl+C 发的是 SIGINT，下游管道提前关闭发的是 SIGPIPE，这两个都会让你的程序猝死。`bufio.Writer` 如果有缓冲没 flush，数据就丢了。

```go
sigCh := make(chan os.Signal, 1)
signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
go func() {
    <-sigCh
    writer.Flush() // 先 flush
    os.Exit(0)
}()
```

SIGPIPE 稍微特殊，Go 运行时在 Linux 上默认会把写管道失败转成 `EPIPE` 错误返回给你，不会直接 kill 进程，所以你只需检查 `fmt.Fprintln` 的返回错误，发现 EPIPE 就干净退出，不要慌。

---

**五、跨平台换行——那个藏在行尾的幽灵**

Windows 的文件换行是 `\r\n`，Scanner 默认按 `\n` 切，结果每行末尾偷偷藏了一个 `\r`。你的字符串比较会悄悄失败，日志里打出来还看不见。

```go
line = strings.TrimRight(line, "\r")
```

一行代码，放在处理逻辑最前面，永绝后患。有人用 `strings.TrimSpace`，但那会把行首行尾的空格也吃掉，语义不同，按需选择。

---

**六、性能：Scanner 省内存，worker pool 提吞吐**

`bufio.Scanner` 默认每次只把一行加载进内存，大文件友好，比 `ioutil.ReadAll` 强多了。默认缓冲是 64KB 每行，超长行要自己扩：

```go
scanner.Buffer(make([]byte, 1024*1024), 1024*1024)
```

如果每行处理很重（比如要正则匹配、调外部服务），可以上 worker pool：

```go
lines := make(chan string, 100)
results := make(chan string, 100)

// 读主 goroutine
go func() {
    for scanner.Scan() {
        lines <- scanner.Text()
    }
    close(lines)
}()

// worker pool
var wg sync.WaitGroup
for i := 0; i < 8; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        for line := range lines {
            results <- process(line)
        }
    }()
}

// 关闭 results
go func() { wg.Wait(); close(results) }()

// 汇总写 stdout
for r := range results {
    fmt.Println(r)
}
```

注意：worker pool 输出顺序不保证和输入一致，如果你的场景要求保序，就给每行加序号，汇总时排序后再输出，复杂度稍微上去一点，但绝对值得。

---

**总结一句话**：行过滤器看着简单，真正做到「数据不丢、错误不污染、信号不猝死、跨平台不踩坑、大文件不爆内存」，才算合格的 Unix 公民。加油！

## 🏃 跑一下试试

测试方法有两种。第一种管道方式：在终端执行 echo 'hello go' | go run line-filters.go，程序会输出 HELLO GO；连续多行可用 printf 'hello go\nfmt\n' | go run line-filters.go，依次输出 HELLO GO 和 FMT。第二种文件重定向：先准备一个 some.txt，内容若干行，再执行 go run line-filters.go < some.txt，程序会把每一行转成大写后打印。若想部署为独立命令，执行 go build -o line-filters 生成二进制，之后 cat some.txt | ./line-filters 即可，省去每次编译的开销。交互模式下直接运行 go run line-filters.go，手动逐行输入，按 Ctrl+D 发送 EOF 结束。

## 💡 师兄的碎碎念

💡 关键提示：① 默认 buffer 大小坑：bufio.Scanner 默认单行上限 64KB，若处理超长行需调用 scanner.Buffer() 手动扩容，否则 scanner.Scan() 会静默截断并报错，线上环境容易踩坑。② stdout 与 stderr 分家：业务输出写 os.Stdout，错误信息写 os.Stderr，这样下游管道只收到干净数据，不会把错误日志混进处理结果。③ 退出返回码：若中途出现错误，用 os.Exit(1) 显式退出，调用方脚本才能通过 $? 感知失败，不要只打印错误后继续执行。④ go build 部署：生产环境提前编译成二进制，避免每次 go run 触发编译，启动更快，适合高频管道场景。⑤ SIGPIPE 处理：下游提前关闭管道时（如 head -n 5 截断），程序会收到 SIGPIPE，Go 运行时默认会让写操作返回错误，需检查 fmt.Println 等的错误返回或用 signal 包捕获，防止程序异常崩溃。

## 🎓 知识点清单

- bufio.NewScanner(os.Stdin) 已正确初始化
- for scanner.Scan() 逐行读取循环无误
- strings.ToUpper() 转换逻辑正确
- fmt.Println() 输出每行结果
- 程序能正常处理空行与 EOF
- 编译无警告，go vet 通过
- 管道测试与文件重定向测试均通过

## ➡️ 下一关

行过滤器掌握之后，我们进入第 65 关「文件路径 (file-paths)」——学习 filepath.Join、filepath.Dir、filepath.Ext 等路径操作，写出跨平台兼容的文件处理程序，是后续 I/O 实战的重要基础。
