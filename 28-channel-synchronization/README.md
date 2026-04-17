# 第 28 关：通道同步（师兄带你学 Go）

## 🎯 这一关你会学到

本关演示如何用 channel 实现协程同步。主协程调用 `go worker(done)` 启动子协程后，执行 `<-done` 进入阻塞状态，耐心等待子协程往 channel 里写入值。子协程完成任务后执行 `done <- true`，主协程随即收到信号、解除阻塞、继续运行。这种模式让两个协程的执行顺序得到精确控制，是 Go 并发编程中最基础也最常用的同步手段。

## 🤔 先想一个问题

想象你去楼下洗衣房丢衣服。你不可能搬把椅子坐在机器旁边干等三十分钟——你按下启动按钮，就回去该干嘛干嘛，刷视频、写代码、泡杯茶都行。蜂鸣器"叮"一声响，你回来收衣服。`done` channel 就是那声"叮"——子协程做完往 done 里丢个值，主协程收到信号就知道：可以继续了。

**channel 天然是协程间的"通知开关"：主协程 `<-done` 阻塞等信号，子协程 `done <- true` 发信号。只要有人在一端等、另一端送，就能实现精确的"等做完再继续"。**

## 📖 看代码

```go
// 我们可以使用通道来同步协程之间的执行状态。
// 这儿有一个例子，使用阻塞接收的方式，实现了等待另一个协程完成。
// 如果需要等待多个协程，[WaitGroup](waitgroups) 是一个更好的选择。

package main

import (
	"fmt"
	"time"
)

// 我们将要在协程中运行这个函数。
// `done` 通道将被用于通知其他协程这个函数已经完成工作。
func worker(done chan bool) {
	fmt.Print("working...")
	time.Sleep(time.Second)
	fmt.Println("done")

	// 发送一个值来通知我们已经完工啦。
	done <- true
}

func main() {

	// 运行一个 worker 协程，并给予用于通知的通道。
	done := make(chan bool, 1)
	go worker(done)

	// 程序将一直阻塞，直至收到 worker 使用通道发送的通知。
	<-done
}
```

## 🔍 师兄给你逐行拆

### worker 里发信号：done <- true

```go
func worker(done chan bool) {
    fmt.Print("working...")
    time.Sleep(time.Second)
    fmt.Println("done")
    done <- true  // 告诉外界：我干完了
}
```

worker 把活干完之后，最后一行往 `done` 这个 channel 发一个值。发的是 `true`，但内容无所谓——重要的是**这次通信本身**。发送方阻塞在这里，直到有人来收；接收方阻塞在那里，直到有人来发。两端一碰头，握手完成，接收方就知道：worker 做完了。"用全局布尔变量 + 轮询检查"相比，channel 方案没有自旋、没有竞态条件、也不会错过信号——这就是 Go 推崇 channel 做同步的原因。

### main 里等信号：`<-done`

```go
done := make(chan bool, 1)
go worker(done)
<-done   // 等 worker 往 done 里丢个值
```

`<-done` 这一行会**阻塞**，main 在此原地等待，直到 worker 向 `done` 发送一个值才继续执行，然后程序退出。这样可以保证 worker 里的 `"working..."` 和 `"done"` 都打印完毕，不会被 main 提前退出截断。

这里用的是 `make(chan bool, 1)`——带缓冲 1。好处是：worker 执行完毕调用 `done <- true` 时，即使 main 还没走到 `<-done`，这次发送也**不会阻塞**，值先存进缓冲区，worker 可以立刻返回。

如果换成无缓冲 `make(chan bool)`，也能正常工作：因为 main 在启动 goroutine 后紧接着就到达 `<-done` 等待，worker 发送时 main 必然已经就位，双方握手成功。两种写法都对，带缓冲版本对时序的依赖更宽松一些。

### 信号型 channel：chan struct{} 比 chan bool 更地道

既然我们不关心 channel 里传什么，只关心。"有没有信号传过来"，那其实用 `chan struct{}` 比 `chan bool` 更地道。`struct{}` 是**空结构体**，占 0 字节内存，存粹是一个"占位符"。

```go
done := make(chan struct{})
go func() {
    // ... 干活 ...
    done <- struct{}{}   // 或者 close(done)，见下一节
}()
<-done
```

Go 社区惯用：**信号型 channel 一律用 `chan struct{}`**——语义更清晰（这是信号不是数据），内存更省。

### 等多个协程：close(done) 或 WaitGroup

如果要等**多个**协程全部完成，有两种常见写法。

**`sync.WaitGroup`**：启动协程前调用 `wg.Add(1)`，每个 worker 函数内用 `defer wg.Done()` 标记完成，主协程调用 `wg.Wait()` 阻塞直到计数归零。"等这批协程全都完成"的经典场景。

**close channel 广播**：当某个事件发生时 `close(done)`，所有在 `<-done` 上阻塞的协程都会**同时被唤醒**（接收立即返回零值）。这是"广播取消信号"的惯用法，常用于 `ctx.Done()`。

**经验法则**：
- 单协程同步用一个 `done` channel；
- 多个相同类型的协程等全完用 `WaitGroup`；
- "广播取消/关停"信号用 `close(done)`。

## 🏃 跑一下试试

```bash
$ go run channel-synchronization.go
working...done
```

`working...` 和 `done` 在同一行是因为 `worker` 里先用 `fmt.Print`（不换行）打印 `working...`，再用 `fmt.Println` 打印 `done`。主协程执行 `<-done` 阻塞等待 worker 通过 channel 发送信号，保证 worker 完整执行后主程序才退出，所以不会错过任何输出。

## 💡 师兄的碎碎念

- **信号型 channel 推荐 `chan struct{}`**：`struct{}` 不占内存、语义明确就是"信号"，比 `chan bool` 更地道。
- **只发一次可以用 `close(done)`**：关闭后所有接收方立即解除阻塞拿到零值，特别适合"取消/结束"广播。
- **单 vs 多协程**：单协程同步用 done channel；多个协程完成用 `sync.WaitGroup`；别用 N 个 done channel 复杂化问题。
- **永远别用 `time.Sleep` 等协程**：靠猜时间是定时炸弹，channel/WaitGroup 才是唯一正确姿势。

## 🎓 这一关的知识点清单

- **channel 做同步原语**：channel 的收发阻塞语义天然替代锁和条件变量，是 Go 最推崇的协程同步方式。
- **`<-ch` 阻塞等值**：主协程 `<-done` 会一直等，直到子协程向 `done` 发送；避免用 `time.Sleep` 估等时间。
- **`chan struct{}` 信号通道**：只传"有无"信息时用空结构体，0 字节、语义明确、最地道。
- **close vs send**：`done <- x` 发一次信号；`close(done)` 广播所有等待方一次性解除阻塞（适合"一对多"通知）。
- **多协程同步方案**：单个用 channel，多个用 `sync.WaitGroup`（计数模式），广播取消用 `close` 或 `context.Context`。

## ➡️ 下一关

继续学习如何限制 channel 的读写方向，让函数签名更安全清晰。

[下一关：Channel Directions →](../29-channel-directions/)
