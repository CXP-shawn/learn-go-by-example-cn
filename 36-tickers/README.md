# 第 36 关：Ticker（打点器）（师兄带你学 Go）

## 🎯 这一关你会学到
time.NewTicker 会按照你指定的间隔，周期性地往 .C 通道里塞一个 time.Time 值，只要你不叫停它就会永远打下去。想让它停下来，调用 .Stop() 即可——但注意 Stop 不会关闭 .C 通道，所以退出 goroutine 要靠你自己发信号。

## 🤔 先想一个问题
你有没有注意过地铁站台上那块倒计时牌？「下一班：4 分 53 秒……4 分 52 秒……」——不管你在不在，列车每隔固定时间就会进站，不多等你一秒。游戏里角色每秒自动回 5 点血、心跳监控每 200ms 采一次样、日志系统每隔 30 秒 flush 一次缓冲区，背后都是同一个思路：固定节拍，周而复始。Go 的 Ticker 就是这么一个「节拍器」——你拧好发条，它就一直嘀嗒嘀嗒地走，直到你叫它停。

## 📖 看代码
```go
// [定时器](timers) 是当你想要在未来某一刻执行一次时使用的
// - _打点器_ 则是为你想要以固定的时间间隔重复执行而准备的。
// 这里是一个打点器的例子，它将定时的执行，直到我们将它停止。

package main

import (
	"fmt"
	"time"
)

func main() {

	// 打点器和定时器的机制有点相似：使用一个通道来发送数据。
	// 这里我们使用通道内建的 `select`，等待每 500ms 到达一次的值。
	ticker := time.NewTicker(500 * time.Millisecond)
	done := make(chan bool)

	go func() {
		for {
			select {
			case <-done:
				return
			case t := <-ticker.C:
				fmt.Println("Tick at", t)
			}
		}
	}()

	// 打点器可以和定时器一样被停止。
	// 打点器一旦停止，将不能再从它的通道中接收到值。
	// 我们将在运行 1600ms 后停止这个打点器。
	time.Sleep(1600 * time.Millisecond)
	ticker.Stop()
	done <- true
	fmt.Println("Ticker stopped")
}
```

## 🔍 师兄给你逐行拆
在正式看 Ticker 的代码之前，先把这一关前半段的 Timer 和 Ticker 的根本区别说清楚，后面理解起来会顺很多。

Timer 是「一次性闹钟」：定好时间，响一声，结束。Ticker 是「循环闹钟」：定好间隔，每隔这么久响一次，永不停歇，除非你主动 Stop。这个区别决定了两者的使用场景几乎不重叠——Timer 适合「延迟执行某件事」，Ticker 适合「周期性做某件事」。

好，进入 Ticker 的代码。

**第一段：创建 Ticker**

```go
ticker := time.NewTicker(500 * time.Millisecond)
```

time.NewTicker 接收一个 time.Duration 参数，这里传入 500 毫秒。返回值是 *time.Ticker，是个指针类型。Ticker 结构体里最重要的字段就是 C，类型是 <-chan time.Time，一个只读的时间通道。每过 500ms，Go 运行时就会往 C 里发一个 time.Time 值，记录的是「这次 tick 触发的时刻」。你不需要自己写任何循环计时的逻辑，运行时帮你全包了。

这里有个细节值得留意：500 * time.Millisecond 这种写法，是 Go 时间包里非常惯用的写法。time.Millisecond 本身是一个 time.Duration 类型的常量，值是 1,000,000 纳秒，乘以 500 就是 500 毫秒。不要傻乎乎地传一个裸数字 500 进去，那会被当成 500 纳秒，几乎等于什么都没等。

**第二段：done 通道——退出信号**

```go
done := make(chan bool)
```

这一行单独看很简单，但它的存在意义非常关键。后面你会看到，我们把「监听 ticker」的逻辑放在一个 goroutine 里，那个 goroutine 需要一个方式知道「该结束了」。done 就是主协程和子协程之间的「收工信号」。

为什么不直接用 ticker.Stop() 来通知 goroutine 退出？因为 Stop 不会关闭 C 通道。这是 Go 标准库的设计决定——如果 Stop 关闭了 C，那么正在 <-ticker.C 上等待的代码就会立刻收到零值并继续执行，可能引发逻辑错误。所以标准库选择「Stop 只是停止往 C 里发值，但 C 本身还开着」。这就意味着，Stop 之后如果你的 goroutine 还在 select 里等 ticker.C，它会永远卡在那里——goroutine 泄漏。done 通道就是用来解决这个问题的。

**第三段：goroutine 里的 for + select**

```go
go func() {
    for {
        select {
        case <-done:
            return
        case t := <-ticker.C:
            fmt.Println("Tick at", t)
        }
    }
}()
```

这是整段代码的核心，拆开来慢慢看。

首先是 go func(){}()，匿名函数立即启动为一个新的 goroutine，和主协程并发跑。

然后是 for {} 死循环。注意这里没有任何循环条件，就是无限转。这是 Go 里处理「持续监听通道」的标准写法，不要觉得奇怪。

循环体里是 select，同时监听两个 case：

第一个 case 是 <-done。如果 done 通道里有值进来，说明主协程发来了「收工」信号，直接 return，goroutine 干净退出。

第二个 case 是 t := <-ticker.C。每当 Ticker 触发一次，C 里就会有一个 time.Time 值，select 捡起来，赋给变量 t，然后打印「Tick at」加上那个时间戳。t 是真实的触发时刻，精确到纳秒级，打印出来你会看到类似 2009-11-10 23:00:00.5 +0000 UTC m=+0.500... 这样的内容。

select 的语义是：哪个 case 准备好了就走哪个；如果两个同时准备好，随机选一个。这里两个 case 同时触发的概率极低，实际上 done 信号来的时候 ticker.C 通常是空的，所以行为上很确定。

**第四段：主协程的等待与收尾**

```go
time.Sleep(1600 * time.Millisecond)
ticker.Stop()
done <- true
fmt.Println("Ticker stopped")
```

主协程 sleep 1600 毫秒。在这 1600ms 里，Ticker 每 500ms 触发一次：

- 第 500ms：第 1 次 tick，goroutine 打印「Tick at ...」
- 第 1000ms：第 2 次 tick，goroutine 打印「Tick at ...」
- 第 1500ms：第 3 次 tick，goroutine 打印「Tick at ...」
- 第 1600ms：主协程醒来

所以你大概会看到 3 行 Tick 输出。为什么说「大概」？因为 goroutine 调度和系统定时器有一定误差，但在这个量级上，3 次是稳定可预期的结果。

主协程醒来之后，先调用 ticker.Stop()。这一步让运行时停止往 C 里投递新值——已经在 C 里排队的值不受影响，但不会再有新的了。

然后 done <- true，往 done 通道里发一个值。这是个无缓冲通道，所以这里会短暂阻塞，直到 goroutine 里的 select 接收了这个值。goroutine 收到之后执行 return，干净退出，没有任何泄漏。

最后打印「Ticker stopped」，程序结束。

**Stop 不关闭 C，这件事要烙在脑子里**

再强调一遍这个设计，因为很多初学者会在这里踩坑。time.Timer 的 Stop 也是同样的行为——Stop 之后 C 不会被 close。如果你的代码里只有 <-ticker.C 而没有退出机制，调用 Stop 之后那个 goroutine 就永远卡死了。正确的姿势就是像示例这样：单独准备一个 done 通道，Stop 和发 done 信号配合使用，两件事缺一不可。

**和 Timer 对比一下，加深印象**

Timer 用完就没了，想再用得重新 NewTimer 或者调用 Reset。Ticker 一旦创建就持续跑，Stop 之后也不能重启，想再用得重新 NewTicker。从这个角度看，Ticker 更像是一个「持续运转的水泵」，你要么让它一直转，要么彻底关掉，没有「暂停再继续」这回事。

另外，Timer.C 只会发一个值，发完就空了；Ticker.C 会持续有值进来，不主动消费的话会堆积（虽然标准库内部对 C 的缓冲区做了限制，容量是 1，超出的值会被丢弃，不会无限堆积）。所以如果你的消费速度赶不上 Ticker 的触发频率，tick 会被静默丢弃，这个也需要注意。

整体来看，这段代码的结构非常值得当作模板记住：NewTicker → goroutine 里 for+select 监听 C 和 done → 主协程 sleep → Stop + 发 done 信号。这套组合拳在实际项目里会反复用到，从健康检查到定时上报，到处都是它的影子。

## 🏃 跑一下试试
```bash
# 文件：ticker.go
cat << 'EOF' > ticker.go
package main

import (
    "fmt"
    "time"
)

func main() {
    ticker := time.NewTicker(500 * time.Millisecond)
    done := make(chan bool)

    go func() {
        for {
            select {
            case <-done:
                return
            case t := <-ticker.C:
                fmt.Println("Tick at", t)
            }
        }
    }()

    time.Sleep(1600 * time.Millisecond)
    ticker.Stop()
    done <- true
    fmt.Println("Ticker stopped")
}
EOF

go run ticker.go
```

预期输出（时间戳因机器而异，重点是恰好 3 行 Tick）：
```
Tick at 2026-04-17 06:59:02.5 +0000 UTC
Tick at 2026-04-17 06:59:03.0 +0000 UTC
Tick at 2026-04-17 06:59:03.5 +0000 UTC
Ticker stopped
```

## 💡 师兄的碎碎念
- **Stop 不会 close `ticker.C`**：调完 `ticker.Stop()` 之后通道还开着，只是不再往里送值。所以后续 select 必须靠 `done` 之类的退出信号来结束 goroutine，别指望通道关闭来触发退出。
- **长跑服务务必 `defer ticker.Stop()`**：函数一旦提前 return 而没有 Stop，ticker 的内部计时器会一直跑，goroutine 也跟着泄漏，生产里这是常见的隐形内存/goroutine 泄漏根源。
- **Ticker 内部 buffer 只有 1，消费跟不上会丢 tick**：它不会堆积，多出来的 tick 直接扔掉。如果你的消费逻辑比 500ms 慢，就会静默丢点，设计时要把消费耗时控制在间隔以内，或者把重活扔给另一个 goroutine 异步处理。
- **不要 `for range ticker.C`**：这写法根本没有退出口，goroutine 会永远跑下去。标准姿势是 `for { select { case <-ticker.C: ... case <-done: return } }`，用 select 同时守住退出信号。

## 🎓 这一关的知识点清单
- [ ] 用 `time.NewTicker(500 * time.Millisecond)` 创建 ticker，拿到 `*time.Ticker`
- [ ] goroutine 里用 `for + select`，同时监听 `ticker.C` 和退出信号 `done`
- [ ] 主 goroutine `time.Sleep(1600 * time.Millisecond)` 后先调 `ticker.Stop()`，再往 `done` 发信号
- [ ] 确认输出恰好 3 行 `Tick at ...` + 1 行 `Ticker stopped`
- [ ] 理解 1600ms ÷ 500ms = 3.2，所以只来得及 tick 3 次

## ➡️ 下一关
第 37 关「Worker Pools（工作池）」——这一关你会看到 ticker 和 channel 协作的真实战场：多个 worker goroutine 同时从任务 channel 拉活，主协程派发任务、收集结果。把这关的 select + done 模式带进去，你就能写出可优雅退出的长跑工作池，值得好好啃。

👉 [第 37 关：工作池](../37-worker-pools/)
