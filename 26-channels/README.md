# 第 26 关：通道（师兄带你学 Go）

## 🎯 这一关你会学到

本关学习 Go 的 channel（通道）。channel 是协程之间传递数据的管道，使用 `<-` 操作符进行发送和接收。默认情况下，channel 是无缓冲的，发送方和接收方必须同时就绪，否则会阻塞等待——这让协程间的通信天然具备同步语义，是 Go 并发模型的核心机制之一。

## 🤔 先想一个问题

想象一家餐厅的。"传菜口"——后厨做完一道菜放上去，服务员过来取走。厨师不需要敲服务员的门，服务员也不用闯进后厨翻锅台。传菜口既帮两端解耦，又自带"同步"：菜没做好服务员空等、菜做好了没人端走厨师手里端着不能放——Go 的 channel 完全是一回事。

**Channel 是 Go 协程间通信和同步的原语。用 `make(chan T)` 创建，`ch <- v` 发送，`v := <-ch` 接收；无缓冲 channel 的收发会阻塞直到另一方就绪，天然提供同步语义。**

## 📖 看代码

```go
// _通道(channels)_ 是连接多个协程的管道。
// 你可以从一个协程将值发送到通道，然后在另一个协程中接收。

package main

import "fmt"

func main() {

	// 使用 `make(chan val-type)` 创建一个新的通道。
	// 通道类型就是他们需要传递值的类型。
	messages := make(chan string)

	// 使用 `channel <-` 语法 _发送_ 一个新的值到通道中。
	// 这里我们在一个新的协程中发送 `"ping"` 到上面创建的 `messages` 通道中。
	go func() { messages <- "ping" }()

	// 使用 `<-channel` 语法从通道中 _接收_ 一个值。
	// 这里我们会收到在上面发送的 `"ping"` 消息并将其打印出来。
	msg := <-messages
	fmt.Println(msg)
}
```

## 🔍 师兄给你逐行拆

### make(chan T)：创建管道

`messages := make(chan string)` 创建一个元素类型为 `string` 的 channel。"一个只放 string 的通道"。

**零值是 `nil`**。对一个 nil channel 做读或写都会**永远阻塞**——在 `select` 中这被用作"禁用某个分支"的技巧。

channel 是**引用类型**，传参时传的是底层结构的引用，多个 goroutine 共享同一个 channel 不需要加锁，channel 内部已经做好了并发安全的排队。

### 发送：ch <- v

```go
messages <- "ping"
```

这行代码的意思是：**把字符串 `"ping"` 送进 `messages` channel**。箭头 `<-` 的方向就是数据流动的方向——从右边的值，流向左边的 channel，直观易懂。

**阻塞语义**是理解 channel 的关键。对于无缓冲 channel，发送方执行 `ch <- v` 后会**原地阻塞**，哪儿也去不了，直到另一端有 goroutine 执行 `<-ch` 来接收为止。这是 Go 的设计哲学：用阻塞来天然同步，不需要额外的锁。

本例中写法如下：

```go
go func() { messages <- "ping" }()
```

把发送操作放进一个新协程——原因正是阻塞语义：如果在主 goroutine 里直接发送，主协程会阻塞在那一行，永远到不了下面的接收语句，造成**死锁**。开一个新协程，让它在后台等待，主协程继续往下走到 `<-messages` 完成配对，双方才各自解除阻塞，程序顺利运行。

### 接收：msg := <-ch

```go
msg := <-messages
```

箭头方向反过来，意思也反过来——**从 channel 里取出一个值**，赋给 `msg`。

和发送一样，无缓冲 channel 的接收**同样会阻塞**：如果此刻没有发送方，接收方就原地等待，直到有值送来为止。

这正是无缓冲 channel 的核心语义，常称为 **rendezvous（会合）**——发送方和接收方必须**同时就绪**，两端对上了才能交换数据，任何一方先到都得等另一方。

回到本例的执行流程：

1. 主协程启动 goroutine 后，立刻走到 `msg := <-messages`，**阻塞**；
2. 新 goroutine 执行 `messages <- "ping"`，发送成功；
3. 主协程被唤醒，`msg` 拿到 `"ping"`；
4. `fmt.Println(msg)` 打印结果。

正是这种会合机制，让 channel 天然充当了协程间的**同步点**，无需额外的锁或条件变量。

### 同步是免费赠品：阻塞就是信号

Channel 的发送与接收天生带有**阻塞语义**，这为你免费提供了协程间的同步能力——无需显式调用 `sync.WaitGroup`，也不需要用 `time.Sleep` 傻等，光靠收发的顺序就能保证执行顺序。

```go
messages := make(chan string)
go func() { messages <- "ping" }()
fmt.Println(<-messages)
```

`main` 协程走到 `<-messages` 这一行时会**原地阻塞**，直到子协程把 `"ping"` 送进来才继续执行。"等子协程做完"。

Rob Pike 的 Go 并发哲学名言：

> **Don't communicate by sharing memory; share memory by communicating.**
>
> 不要通过共享内存来通信，要通过通信（channel）来共享内存。

channel + goroutine 的配合远比锁+共享变量更容易写对、更容易读懂。

## 🏃 跑一下试试

```bash
$ go run channels.go
ping
```

`ping` 从协程发出、在 main 接收并打印。由于 channel 收发都是阻塞的，我们**不需要** `time.Sleep` 就能确定等到消息。

## 💡 师兄的碎碎念

- 无缓冲 channel 是同步通信原语：发送方在接收方就绪前阻塞，接收方在发送方就绪前阻塞，两端在「握手」瞬间完成数据交换
- 对 `nil` channel 读写永远阻塞，常用于 `select` 中动态禁用某个分支，是一种惯用技巧
- 向已关闭的 channel 发送会引发 panic；接收已关闭 channel 会立即返回零值，并可通过 `v, ok := <-ch` 中的 `ok == false` 检测关闭状态
- channel 方向可在类型签名中约束：`chan<- T` 表示只写，`<-chan T` 表示只读，有助于编译期静态验证并清晰表达函数意图

## 🎓 这一关的知识点清单

- **channel 类型**：用 `chan T` 声明，元素类型 T 紧跟 `chan` 关键字，零值是 `nil`。
- **make 创建**：`make(chan T)` 无缓冲、`make(chan T, n)` 带容量为 n 的缓冲。
- **发送与接收箭头方向**：`ch <- v` 箭头向左发送；`v := <-ch` 箭头向左接收（看箭头指向谁谁就是"接受方"）。
- **阻塞语义**：无缓冲 channel 的收发是 rendezvous——发送方与接收方必须同时就绪才配对放行，任一方先到就等另一方。
- **同步功能**：利用阻塞语义做协程同步，免去 `sync.WaitGroup` / `time.Sleep` 的手动管理。

## ➡️ 下一关

继续学习带缓冲的 channel，了解如何解耦发送方与接收方的执行节奏。

[下一关：Channel Buffering →](../27-channel-buffering/)
