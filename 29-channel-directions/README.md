# 第 29 关：通道方向（师兄带你学 Go）

## 🎯 这一关你会学到

本关学习如何限定 channel 的读写方向。在函数参数里写 `chan<- T` 表示只能向该 channel 发送数据，写 `<-chan T` 表示只能从该 channel 接收数据。这样的方向声明让编译器在编译期就能帮你检查误用，避免一个本该只写数据的函数却意外读取了数据，让代码意图更明确、更安全。

## 🤔 先想一个问题

想象公司门禁卡的分级权限管理：财务部的员工卡只能刷开财务室的门，进不了研发区；访客卡只能刷开访客接待区，碰不了任何内部门禁。这就是**最小权限原则**——每个角色只拥有完成自己职责所必需的权限，多余的权限一律封死，既减少误操作，也降低安全风险。"只写 channel"，一个只负责从 channel 里取数据的函数，参数就声明成"只读 channel"——编译器帮你把关，同事看代码也一目了然。

**Go 允许在函数签名里把 channel 限定为单向：`chan<- T` 表示只能发送（箭头指向 channel），`<-chan T` 表示只能接收（箭头指向变量）。编译器在编译期就拦下反方向的误用，代码更安全、意图更清晰。**

## 📖 看代码

```go
// 当使用通道作为函数的参数时，你可以指定这个通道是否为只读或只写。
// 该特性可以提升程序的类型安全。

package main

import "fmt"

// `ping` 函数定义了一个只能发送数据的（只写）通道。
// 尝试从这个通道接收数据会是一个编译时错误。
func ping(pings chan<- string, msg string) {
	pings <- msg
}

// `pong` 函数接收两个通道，`pings` 仅用于接收数据（只读），`pongs` 仅用于发送数据（只写）。
func pong(pings <-chan string, pongs chan<- string) {
	msg := <-pings
	pongs <- msg
}

func main() {
	pings := make(chan string, 1)
	pongs := make(chan string, 1)
	ping(pings, "passed message")
	pong(pings, pongs)
	fmt.Println(<-pongs)
}
```

## 🔍 师兄给你逐行拆

### chan<- T：只写 channel

```go
func ping(pings chan<- string, msg string) {
    pings <- msg
}
```

`pings chan<- string` 的意思是：这个 channel **只能发送，不能接收**。函数内部只允许写 `pings <- msg`，如果误写了 `<-pings`，编译器会直接报错，将错误消灭在编译阶段。

**记忆技巧：箭头靠近 chan。**

```
chan<- string   // 箭头指向 chan，数据从外部流入 chan → 只写
<-chan string   // 箭头背对 chan，数据从 chan 流出   → 只读
```

把普通双向 channel 传给只写参数是合法的，Go 会自动收窄权限：

```go
ch := make(chan string, 1)
ping(ch, "hello") // ch 是双向的，传入后变只写，完全 OK
```

这种约束让函数签名即文档——调用者一眼就知道 `ping` 只会往 channel 里放数据，不会偷偷消费它，代码意图更清晰，协作更安全。

### `<-chan T`：只读 channel

```go
func pong(pings <-chan string, pongs chan<- string) {
    msg := <-pings
    pongs <- msg
}
```

`pings <-chan string` 表示。这个 channel **只能接收，不能发送**。函数体里只允许 `msg := <-pings`，如果写了 `pings <- "x"` 编译器直接拒绝。

**记忆方法**：箭头远离 chan——数据从 chan 里流出来，所以是读。

本例 `pong` 同时接收两个方向受限的 channel：从只读 pings 里拿值、往只写 pongs 里送值。函数签名白纸黑字写死了谁只读、谁只写，内部根本不可能搞反方向，编译器会帮你挡住所有误用。

### 双向 channel 传给单向参数：自动窄化

`main` 里创建的 `pings := make(chan string, 1)` 是**双向** channel，既可发送也可接收。当我们调用 `ping(pings, "hello")` 时，函数签名要求的是 `chan<- string`（只写），Go 编译器会自动将双向 channel **窄化**为只写，无需任何显式转换。同理，调用 `pong(pings, pongs)` 时，`pings` 自动窄化为只读 `<-chan string`，`pongs` 自动窄化为只写 `chan<- string`。

这种**单向窄化**是合法且自动的。但反过来——试图将单向 channel 扩展为双向——编译器会直接报错。

这与 Go 接口/子类型的设计思想一脉相承：**能力减少的方向合法，能力增加的方向不合法**。单向 channel 作为函数参数，本质上是一种编译期约束，让每个函数只拥有它真正需要的权限，从而让数据流向更清晰、更安全。

### 什么时候用方向限定？

方向限定的核心价值是**让意图可见、让错误提前暴露**。

**函数接口语义化**：生产者函数接收 `chan<- T`（只写），消费者函数接收 `<-chan T`（只读）。"谁产、谁消"，不用一行行读函数体。

**防止函数内写错方向**：左手持有一个只读 channel 却想往里写？编译失败，BUG 消灭在敲键盘的那一刻。

**API 返回值限制外部行为**：返回 `<-chan Event` 的函数让调用方只能读，不能偷偷往里塞东西。标准库的 `time.After` 就是典范——返回 `<-chan time.Time`，你只能读超时信号，不能搞破坏。

**经验法则**：能单向就用单向；只有局部内部用的临时 channel 才用双向。

## 🏃 跑一下试试

```bash
$ go run channel-directions.go
passed message
```
`ping` 把 `"passed message"` 写进 `pings` channel，`pong` 从 `pings` 读出来再写到 `pongs`，`main` 最后从 `pongs` 读出来打印。

## 💡 师兄的碎碎念

- 单向 channel 只在函数参数和接口签名里出现，让调用方拿到受限视图，防止误用
- `time.After` 返回 `<-chan time.Time`，是标准库中只读 channel 的经典示范
- 双向 → 单向的转换由编译器自动完成，反方向（单向 → 双向）则编译报错
- 方向限定是纯类型层面的约束，不影响 channel 的缓冲大小和阻塞/非阻塞行为

## 🎓 这一关的知识点清单

- 只写 channel `chan<- T`：函数只能向其发送数据，编译器阻止读取操作
- 只读 channel `<-chan T`：函数只能从其接收数据，编译器阻止发送操作
- 方向窄化规则：双向 `chan T` 可隐式转换为单向类型，反向转换不合法
- 函数签名中的方向限定：在参数类型处声明方向，调用方传入普通双向 channel 即可
- 标准库典型用法：`time.After`、`time.Tick` 等返回 `<-chan T`，是只读 channel 的范本

## ➡️ 下一关

本区间（第 17–29 关）到此结束。下一批（第 30 关 Select 开始）会由其他 agent 处理，敬请期待。
