# 第 35 关：Timer（定时器）（师兄带你学 Go）

## 🎯 这一关你会学到
time.NewTimer 让你创建一个「只触发一次」的定时器：到点后它会往自己的 .C 通道发一个值，你用 <-timer.C 阻塞等待即可。和 time.Sleep 最本质的区别在于，Timer 可以在到点之前调用 .Stop() 把它掐掉，Sleep 一旦睡下去就没办法反悔。

## 🤔 先想一个问题
你应该见过这种场景：外卖 App 下单，商家说「30 分钟后送达」，但你等到一半突然不想要了，直接取消订单——闹钟没响，外卖没来，皆大欢喜。time.NewTimer 就是这种「可以取消的预约闹钟」：你设好倒计时，到点它会敲你一下；但如果中途反悔，调一句 Stop()，闹钟就哑了，不会再打扰你。

## 📖 看代码
```go
// 我们经常需要在未来的某个时间点运行 Go 代码，或者每隔一定时间重复运行代码。
// Go 内置的 _定时器_ 和 _打点器_ 特性让这些变得很简单。
// 我们会先学习定时器，然后再学习[打点器](tickers)。

package main

import (
	"fmt"
	"time"
)

func main() {

	// 定时器表示在未来某一时刻的独立事件。
	// 你告诉定时器需要等待的时间，然后它将提供一个用于通知的通道。
	// 这里的定时器将等待 2 秒。
	timer1 := time.NewTimer(2 * time.Second)

	// `<-timer1.C` 会一直阻塞，
	// 直到定时器的通道 `C` 明确的发送了定时器失效的值。
	<-timer1.C
	fmt.Println("Timer 1 fired")

	// 如果你需要的仅仅是单纯的等待，使用 `time.Sleep` 就够了。
	// 使用定时器的原因之一就是，你可以在定时器触发之前将其取消。
	// 例如这样。
	timer2 := time.NewTimer(time.Second)
	go func() {
		<-timer2.C
		fmt.Println("Timer 2 fired")
	}()
	stop2 := timer2.Stop()
	if stop2 {
		fmt.Println("Timer 2 stopped")
	}

	// 给 `timer2` 足够的时间来触发它，以证明它实际上已经停止了。
	time.Sleep(2 * time.Second)
}
```

## 🔍 师兄给你逐行拆
我们先把完整代码在脑子里过一遍，再逐段细讲。

**第一段：创建 timer1 并阻塞等待**

```go
timer1 := time.NewTimer(2 * time.Second)
<-timer1.C
fmt.Println("Timer 1 fired")
```

time.NewTimer(2 * time.Second) 做了什么？它返回一个 *time.Timer 指针，内部帮你起了一个倒计时。这个结构体只有一个导出字段：C，类型是 <-chan time.Time，也就是一个只读通道。Go 运行时会在你指定的时间到了之后，往这个 C 通道里塞一个 time.Time 值（就是「触发时刻」的时间戳）。

<-timer1.C 这一行是关键。这是一个普通的通道接收操作，当 C 里还没有值的时候，它会老老实实地阻塞在这里，当前 goroutine 挂起，不占 CPU。2 秒到了，运行时往 C 里写值，这一行解除阻塞，程序继续往下走，打印「Timer 1 fired」。

这和 time.Sleep(2 * time.Second) 的效果看起来一样：都是等 2 秒。但本质不同——Sleep 是「我蒙头睡觉，你叫不醒我」；Timer 是「我守着一个通道，你随时可以不往里放值，我就永远等不到触发」。这个差异在第二段会体现出来。

顺带一提：如果你不在乎触发时的具体时间戳，<-timer1.C 收到值直接丢掉也没关系，就像上面代码写的那样，只是用它来「卡一个时间点」。

**第二段：创建 timer2，在 goroutine 里等，主协程抢先 Stop**

```go
timer2 := time.NewTimer(time.Second)
go func() {
    <-timer2.C
    fmt.Println("Timer 2 fired")
}()

stopped2 := timer2.Stop()
if stopped2 {
    fmt.Println("Timer 2 stopped")
}

time.Sleep(2 * time.Second)
```

这一段是整个例子的重点，拆开来一句一句看。

time.NewTimer(time.Second) 创建了一个 1 秒后触发的 timer2。

紧接着，我们开了一个匿名 goroutine，里面做的事情很简单：阻塞在 <-timer2.C，等到触发了就打印「Timer 2 fired」。注意，这个 goroutine 是并发的，开完之后主 goroutine 继续往下跑，不等它。

然后主 goroutine 立刻调用 timer2.Stop()。Stop() 的语义是：「如果这个 timer 还没触发，就把它取消，让它永远不往 C 里写值。」返回值是布尔型：返回 true 表示「我成功阻止了它触发」；返回 false 表示「来不及了，它已经触发了，或者已经被 Stop 过了」。

在这个例子里，因为 timer2 设的是 1 秒，而主 goroutine 在创建完 goroutine 之后几乎立刻就调用了 Stop()，远在 1 秒之前，所以 Stop() 几乎必然返回 true，打印「Timer 2 stopped」。

那个等在 <-timer2.C 里的 goroutine 会怎样？C 通道里永远不会有值进来了（Stop 保证了这一点），所以它会永远阻塞下去——或者说，等主程序退出，它就一起消失了。这就是为什么我们最后要 time.Sleep(2 * time.Second)：等足够长的时间，看看「Timer 2 fired」到底有没有打印出来。如果 Stop 成功了，这 2 秒里你看不到这行字；如果 Stop 没拦住（比如 Stop 调晚了），你就会看到它。

**Stop 返回 false 的几种情况**

这里有个容易踩坑的细节，师兄专门说一下。Stop 返回 false 有两种可能：

第一种：timer 已经触发了。也就是说，运行时已经往 C 里写了值，Stop 晚了一步。这时候 C 里有一个未读的值，如果你不去读它，它就一直堵在那里。如果你之后又想复用这个 timer（用 Reset），官方文档明确提示要先把 C 里的值排干：

```go
if !timer.Stop() {
    <-timer.C // 排掉已触发的值
}
```

第二种：timer 已经被 Stop 过一次了，再调 Stop 也是返回 false。

这个「Stop 之后 C 里可能有残留值」的问题在并发场景下尤其容易出 bug，记住就好。

**和 time.Sleep 的对比总结**

一句话：time.Sleep 是不可中断的，time.NewTimer 是可以取消的。如果你只是想「等一段时间然后继续」，两者等价，用 Sleep 更简洁。如果你需要「等一段时间，但保留中途取消的权利」，就用 Timer。实际项目里，Timer 还经常配合 select 使用，比如「等某个通道有消息，但最多只等 3 秒，超时就走另一条路」，这是 Sleep 完全做不到的事情，后面的章节会讲到。

## 🏃 跑一下试试
```bash
# 文件：timers.go
cat << 'EOF' > timers.go
package main

import (
    "fmt"
    "time"
)

func main() {
    // timer1：2 秒后触发，主线程阻塞等
    timer1 := time.NewTimer(2 * time.Second)
    <-timer1.C
    fmt.Println("Timer 1 fired")

    // timer2：本来也等 1 秒，但我们提前 Stop 掉
    timer2 := time.NewTimer(1 * time.Second)
    go func() {
        <-timer2.C
        fmt.Println("Timer 2 fired") // 永远不会打印
    }()

    stop2 := timer2.Stop()
    if stop2 {
        fmt.Println("Timer 2 stopped")
    }

    // 再等 2 秒，证明 timer2 的 goroutine 真的凉了
    time.Sleep(2 * time.Second)
}
EOF
go run timers.go
```

预期输出：
```
Timer 1 fired
Timer 2 stopped
```

解释：timer1 阻塞 2 秒后 C 有值，主线程继续；timer2 在 goroutine 还没来得及读 C 之前就被 Stop() 掐掉了，C 永远不会有值，goroutine 挂死（实际随进程退出）；最后那个 Sleep 只是给你眼见为实的机会——等了 2 秒，"Timer 2 fired" 就是不出现。

## 💡 师兄的碎碎念
① Stop() 返回 false 意味着 timer 已经触发或已经被 Stop 过了——这时候 C 里可能还躺着一个未读的值，务必用模板清干净：if !t.Stop() { select { case <-t.C: default: } }，否则下次复用会出幺蛾子。
② 一次性延迟任务用 Timer，周期性反复触发用 Ticker——两者别搞混，下一关就是 Ticker。
③ time.AfterFunc(d, f) 是语法糖，省去手写 goroutine + <-C，直接把回调 f 丢进新 goroutine 跑，简单场景很好用，但同样要注意 Stop() 的竞态。
④ 别在一个服务里堆几百个长时 Timer 不管——每个 Timer 底层都有运行时堆结构占内存，用完记得 Stop()，能复用就复用，别让它悄悄泄漏。

## 🎓 这一关的知识点清单
✅ timer1 := time.NewTimer(2 * time.Second) 创建成功
✅ <-timer1.C 会阻塞直到 2 秒后收到信号
✅ timer2.Stop() 在 goroutine 等待之前成功调用，返回 true
✅ 最后 time.Sleep(2 * time.Second) 留足时间证明 timer2 的 goroutine 确实没触发
✅ 输出只有两行：Timer 1 fired → Timer 2 stopped，顺序不乱

## ➡️ 下一关
第 36 关来了——「Ticker 打点器」。Timer 是一次性的闹钟，Ticker 是每隔固定时间就响一次的节拍器。场景换成周期心跳、定时轮询、限流打点，Ticker 全包了。下一关咱们看看 Ticker 怎么用 Stop() 优雅停掉，以及为啥不能让 Ticker 泄漏。

👉 [第 36 关：Ticker](../36-tickers/)
