# 第 47 关：Recover（师兄带你学 Go）

🎯 这一关你会学到
• 什么是 recover——Go 的内置函数，用来从 panic 中"抢救"回来
• recover 必须写在 defer 里才有效，裸调用一点用没有
• recover 的返回值就是 panic 传入的那个值
• panic + defer + recover 三位一体的兜底模式

🤔 先想一个问题

你开一家外卖小店，高峰期突然有一单的顾客地址填错了，快递员没送到，按以前的写法整个店就歇业了——这不合理。更合理的处理是：**这一单标记异常、退钱、道歉，其他订单继续发**。Go 的 recover 就是干这事儿的——当某个请求/任务触发了 panic，咱们不希望整个服务进程跟着崩，只想把这一条流程兜住、记下日志、让别的请求继续跑。这一关咱们就把"崩了还能活过来"这套机制搞清楚。

## 📖 看代码

```go
// Go 通过使用 `recover` 内置函数，可以从 panic 中 _恢复recover_ 。
// `recover` 可以阻止 `panic` 中止程序，并让它继续执行。

// 在这样的例子中很有用：当其中一个客户端连接出现严重错误，服务器不希望崩溃。
// 相反，服务器希望关闭该连接并继续为其他的客户端提供服务。
// 实际上，这就是Go的 `net/http` 包默认对于 HTTP 服务器的处理。

package main

import "fmt"

// 这是一个 panic 函数
func mayPanic() {
	panic("a problem")
}

func main() {
	// 必须在 defer 函数中调用 `recover`。
	// 当跳出引发 panic 的函数时，defer 会被激活，
	// 其中的 `recover` 会捕获 `panic`。
	defer func() {
		if r := recover(); r != nil {
			// `recover` 的返回值是在调用 `panic` 时抛出的错误。
			fmt.Println("Recovered. Error:\n", r)
		}
	}()

	mayPanic()

	// 这行代码不会执行，因为 mayPanic 函数会调用 `panic`。
	// `main` 程序的执行在 `panic` 点停止，并在继续处理完 `defer` 后结束。
	fmt.Println("After mayPanic()")
}
```

🔍 师兄给你逐行拆

**recover 是个啥？又一个内置函数**

recover 和 panic 是一对亲兄弟，都是 Go 的**内置函数（built-in function）**——编译器内置、不用 import 就能用。它的作用非常专一：**在 defer 里调用时，能拿到当前 goroutine 正在发生的 panic value，并把 panic 流程停下来、让外层函数正常返回**。

师兄特别强调：recover **必须写在 defer 注册的函数里面**才有效。你直接在正常流程里调 recover()，它返回 nil，啥也抓不到。这是 Go 设计上的硬性规则——只有 defer 里才是 runtime 认可的"可以接住 panic"的位置。为啥是这样？因为前面讲过，panic 会触发栈展开（unwinding），沿途只执行 defer 注册的函数，所以 recover 只能蹲在 defer 里才能跟 runtime 的异常处理流程碰面。

**mayPanic 和 main 的调用关系**

先看上面这个函数：

```go
func mayPanic() {
    panic("a problem")
}
```

mayPanic 很简单，一进来就 panic 抛字符串 "a problem"。注意 panic 在哪里发生——它发生在 **mayPanic 函数内部**，不是 main。这个细节很关键，等下看 defer 跑在谁身上。

**main 里的魔法：defer 闭包 + recover**

```go
defer func() {
    if r := recover(); r != nil {
        fmt.Println("Recovered. Error:\n", r)
    }
}()

mayPanic()

fmt.Println("After mayPanic()")
```

main 函数的第一件事是 `defer func(){...}()` —— 注意这是一个 **匿名函数**（anonymous function，没名字、就地定义、立刻得到一个函数值）后面跟着一对小括号 `()` 表示"立刻调用这个函数"。但因为前面有 defer，实际的"立刻调用"被推迟到 main 要返回的那一刻。

这个 defer 函数内部写了 `if r := recover(); r != nil` ——这是 Go 常用的**if 初始化语句**（if 后面能带一个分号表达式，先执行赋值再判断），先调 recover() 拿返回值 r，再判断 r 是不是 nil。如果 r 不是 nil 说明确实有 panic 被接住了，打印出来；是 nil 则啥都不做。

然后 main 执行 `mayPanic()` —— 这一调，mayPanic 里 panic 爆炸，当前 goroutine 开始栈展开：先退出 mayPanic 函数、回到 main、准备退出 main。退 main 的过程中 runtime 发现 main 登记了一个 defer——那就跑它。defer 里的 recover() 被调用了，这时候 runtime 正好有 panic value "a problem"，recover 就把它交出来并**停止 panic 流程**。

recover 拿到的值 r 就是 panic 当初传进去的东西——这里是字符串 "a problem"。打印完"Recovered. Error: a problem"之后，defer 结束，main 也跟着正常结束（退出码 0），不会被 panic 拖死。

**被跳过的那一行**

注意 main 最后有一句 `fmt.Println("After mayPanic()")` —— 这行**永远不会执行**。recover 只能让外层函数"正常结束"，但触发 panic 那一刻之后原本要跑的代码已经被跳过了。recover 不是"回到 panic 点继续"，而是"在出事函数之外接住，让父函数善终"。师兄我形容这感觉像是：你骑电动车摔了，路人把你送回家；但你原本要去取快递的行程已经废了，路人不会替你把快递取回来。

**recover 的典型应用：HTTP 服务器**

源码开头注释提了一嘴——**net/http 包就是这么干的**。HTTP 服务器一般每来一个请求就开一个 goroutine 处理，如果这个 goroutine 里代码 panic 了，整个服务进程就得崩——这不行。所以 net/http 在每个请求的 goroutine 里 defer 了 recover，保证一个请求炸了，服务器还能接着服务别的请求。这是 recover 最经典的 "把爆炸范围缩小到一条连接/一次请求"的用法。

**goroutine 隔离的坑**

师兄最后补一个大坑：recover 只能接住**当前 goroutine** 的 panic。如果你在 main 里 defer recover，但用 go 关键字起了一个新的 goroutine，那个新 goroutine 里 panic 了，main 的 recover 根本抓不住——新 goroutine 会把整个进程搞崩。所以每个可能 panic 的 goroutine 都得自己 defer+recover，别指望"中心化兜底"。

🏃 跑一下试试

保存为 recover.go，然后：

```bash
go run recover.go
```

预期输出：

```text
Recovered. Error:
 a problem
```

逐段解释：
- 第一行 "Recovered. Error:" 是代码里 fmt.Println 打印的前缀，\n 让 "a problem" 换行显示
- 第二行 " a problem" 前面有个空格是 fmt.Println 的多参数自动加空格的特性，内容就是 panic 传进去的字符串
- 注意**没有** "After mayPanic()"，也没有 "exit status 2"——说明 panic 被 recover 成功接住，main 正常退出（退出码 0）

💡 师兄的碎碎念

recover 是 Go 兜底的最后一道防线，但它不是救世主，用得不对反而挖坑。师兄给几条肌肉记忆：

1. **recover 只能在 defer 函数里直接调用**。你在 defer 里又调了一个封装函数、在封装函数里 recover ——没用。必须是 defer 的那个函数体里第一层调用 recover。

2. **不要滥用 recover 抹平所有错误**。recover 吞掉 panic 会让真正的 bug 被藏起来。正确姿势是：recover 后记详细日志+栈（用 debug.Stack() 或 runtime/debug 拿调用栈），让 ops 知道出事了。

3. **recover 只接当前 goroutine**。每个 `go func(){}()` 里都得自己 defer+recover，别幻想 main 能兜住子 goroutine。

4. **能用 error 就别用 panic/recover 架构**。这俩是异常控制流，成本比 error 分支高，维护性也差。把它当安全网，不是日常。

吐槽一句：有些 Java 背景的同学喜欢把 panic/recover 玩成 try/catch，满屏 recover。师兄劝你放下 Java 的包袱，Go 的礼貌写法是返回 error。

🎓 知识点清单

- recover 是内置函数，用来从 panic 中恢复
- recover 必须写在 defer 注册的函数里才有效
- recover 的返回值就是 panic 当初传入的那个值
- 成功 recover 后，外层函数正常返回（退出码 0），panic 点之后的代码不会执行
- 每个 goroutine 都得自己 defer+recover，不能跨 goroutine 兜底
- net/http 默认帮你在请求处理器外层套了 recover，所以 handler panic 不会崩服务
- recover 成功后记得打日志+栈，别默默吞掉 bug
- 日常错误处理用 error，panic/recover 只在真正不可恢复或"隔离爆炸半径"时用

➡️ 下一关

panic / defer / recover 三连终于讲完。从下一关开始，咱们换个话题——看看 Go 标准库里那堆好用的**字符串处理函数**。第 48 关 "字符串函数 string-functions" 等你。
