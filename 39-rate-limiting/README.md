# 第 39 关：速率限制（rate-limiting）（师兄带你学 Go）

## 🎯 这一关你会学到
本关你会学会用 time.Tick 配合 channel 实现匀速限流，让请求像流水线一样每隔固定时间才能通过一个。你还会学会用带缓冲的 channel 实现允许突发的限流，系统空闲时先把「名额」攒起来，流量突然来了可以连续放行几个。两种方式组合起来，既能保护系统不被压垮，又能在低峰期积攒弹性、高峰期从容应对。

## 🤔 先想一个问题
你去医院挂号，普通窗口每隔五分钟叫一个号，不管你来多少人，速度就是这么稳，绝不多叫也绝不少叫，这就是匀速限流，系统用同样的节奏一个一个地处理请求，谁也别想插队超速。但医院还有另一种安排：早上开诊前，挂号系统提前攒好了五个「预约名额」放在那里，诊所一开门，头五个人可以噌噌噌连续拿号不用等，等这批名额用完了，后面的人才回到每五分钟一个的正常节奏。这就是允许突发的限流，系统在空闲时把通行资格提前攒好，面对瞬间涌来的小波流量可以直接放行，而不是让人傻等，既保住了整体节奏，又给了真实业务一点喘息的弹性空间。

## 📖 看代码
```go
// [速率限制](http://en.wikipedia.org/wiki/Rate_limiting)
// 是控制服务资源利用和质量的重要机制。
// 基于协程、通道和[打点器](tickers)，Go 优雅的支持速率限制。

package main

import (
	"fmt"
	"time"
)

func main() {

	// 首先，我们将看一个基本的速率限制。
	// 假设我们想限制对收到请求的处理，我们可以通过一个渠道处理这些请求。
	requests := make(chan int, 5)
	for i := 1; i <= 5; i++ {
		requests <- i
	}
	close(requests)

	// `limiter` 通道每 200ms 接收一个值。
	// 这是我们任务速率限制的调度器。
	limiter := time.Tick(200 * time.Millisecond)

	// 通过在每次请求前阻塞 `limiter` 通道的一个接收，
	// 可以将频率限制为，每 200ms 执行一次请求。
	for req := range requests {
		<-limiter
		fmt.Println("request", req, time.Now())
	}

	// 有时候我们可能希望在速率限制方案中允许短暂的并发请求，并同时保留总体速率限制。
	// 我们可以通过缓冲通道来完成此任务。
	// `burstyLimiter` 通道允许最多 3 个爆发（bursts）事件。
	burstyLimiter := make(chan time.Time, 3)

	// 填充通道，表示允许的爆发（bursts）。
	for i := 0; i < 3; i++ {
		burstyLimiter <- time.Now()
	}

	// 每 200ms 我们将尝试添加一个新的值到 `burstyLimiter`中，
	// 直到达到 3 个的限制。
	go func() {
		for t := range time.Tick(200 * time.Millisecond) {
			burstyLimiter <- t
		}
	}()

	// 现在，模拟另外 5 个传入请求。
	// 受益于 `burstyLimiter` 的爆发（bursts）能力，前 3 个请求可以快速完成。
	burstyRequests := make(chan int, 5)
	for i := 1; i <= 5; i++ {
		burstyRequests <- i
	}
	close(burstyRequests)
	for req := range burstyRequests {
		<-burstyLimiter
		fmt.Println("request", req, time.Now())
	}
}
```

## 🔍 师兄给你逐行拆
### time.Tick 和 time.NewTicker 的区别

你第一次看到这道题，应该会被 `time.Tick` 和 `time.NewTicker` 搞混。师兄在这里帮你把这两个东西说清楚，因为这个区别在面试里也很常见，别到时候答不上来。

`time.Tick` 是一个便捷函数，它的本质是在内部创建了一个 `time.Ticker`，然后只把 `Ticker.C` 这个 channel 返回给你，底层的 `*Ticker` 结构体本身被直接丢掉了，你手里拿到的只是一根管道。问题来了：既然你没有持有那个 `*Ticker`，你就永远没有办法调用 `Stop()` 来停止它。Go 的运行时会一直维护着那个内部的 goroutine，每隔固定时间往 channel 里塞一个时间值，哪怕你早就不需要它了，它依然在那儿跑着不停，这就是经典的 goroutine 泄漏。官方文档里也明确说了，除非你确定这个 ticker 要跑到整个进程结束，否则不要用 `time.Tick`。

`time.NewTicker` 则正经多了。它返回一个 `*time.Ticker` 结构体，里面既有 `C` channel 供你读取，也有 `Stop()` 方法让你在用完之后优雅地停下来释放资源。正确的写法是拿到 ticker 之后立刻写一行 `defer ticker.Stop()`，这样不管函数是正常返回还是中途 panic 被 recover，ticker 都会被清理掉，不会留下烂摊子。

那什么时候才能用 `time.Tick` 呢？答案是：只有当这个限流器的生命周期和整个进程一样长，你才可以用它。比如你在 `main` 函数最外层设置一个全局的速率限制，程序退出就是限流器退出，这种情况用 `time.Tick` 是可以的，代码也更简洁。但如果你是在某个函数里、某个请求处理里临时创建一个限流器，用完就不要了，那就老老实实用 `time.NewTicker` 加 `defer Stop`，别图省事给自己埋坑。这道 Go by Example 的题目为了示例简洁用的是 `time.Tick`，你学完原理之后在生产代码里要知道该用哪个。

---

### 第一段：基础匀速限流（代码分析）

搞清楚了 ticker 的区别之后，我们来把这段基础限流的代码逐行拆开看。

首先是 `requests := make(chan int, 5)`，这是一个容量为 5 的带缓冲 channel。带缓冲的意思是，往里面塞最多 5 个值之前都不会阻塞，不需要有人同时在另一端读。代码紧接着用一个循环把 1 到 5 依次发进去，然后立刻 `close(requests)`。`close` 之后你还能从里面把已有的值读完，读完之后再读就会拿到零值且第二个返回值是 `false`，配合 `for req := range requests` 使用的话，channel 里的值读光了循环就自动结束，非常干净。

然后是 `limiter := time.Tick(200 * time.Millisecond)`，这就是我们上面说的那个便捷函数，每 200 毫秒往 `limiter` 这个 channel 里发一个当前时间。这里作为示例用 `Tick` 无妨，因为整个 main 函数跑完程序就结束了。

核心逻辑在这里：

```go
for req := range requests {
    <-limiter
    fmt.Println("request", req, time.Now())
}
```

每一轮循环，先从 `requests` 里拿出一个请求编号，然后执行 `<-limiter`。这一步是关键：它会阻塞在这里，直到 limiter 的下一次 tick 到来，也就是至少等 200 毫秒，才会继续往下走打印那行日志。换句话说，不管你的 requests channel 里已经堆了多少请求等着处理，每次都要等 limiter 放行才能动，处理的节奏完全由 limiter 的频率决定。

最终效果就是，5 个请求会以约 200ms、400ms、600ms、800ms、1000ms 的间隔依次打印出来，时间戳整整齐齐，非常均匀。这是最朴素的匀速限流模型，在流量控制领域有个对应的名字叫漏桶算法（Leaky Bucket）：不管上游流量多猛，到了桶这里一律按固定速率往下漏，超出的部分就等着排队，绝对不会涌出去。这种方式的好处是简单、可预期，输出速率永远稳定；缺点是没有弹性，即便系统当前完全空闲，突然来了一批请求也只能老老实实一个个等，不能利用空闲时段提前透支一点配额，这就引出了后面突发限流要解决的问题，我们下一段再聊。

### 第二段：允许突发的 burstyLimiter

代码里接着出场的这位就不走纯匀速路线了，它叫 `burstyLimiter`，是个容量为 3 的带缓冲 channel：`make(chan time.Time, 3)`。师兄带你一行一行捋：

先是一个 for 循环跑 3 次，把 `time.Now()` 往 `burstyLimiter` 里塞，把容量为 3 的缓冲区**预先塞满**。这 3 个值就是 3 个「预先攒好的令牌」。

然后启动一个 goroutine：`for t := range time.Tick(200 * time.Millisecond) { burstyLimiter <- t }`。这个 goroutine 每 200ms 醒一次，尝试往 `burstyLimiter` 里丢一个新令牌。如果桶满了（3 个都在），`burstyLimiter <- t` 这行就会阻塞，等着有人把令牌取走腾出位置才能写进去。这个阻塞机制是整个令牌桶的精髓——桶满了自然不会超发，桶空了只要 goroutine 每 200ms 能补充上，长期平均速率就是 5 次/秒。

最后循环处理 5 个 `burstyRequests`，每个请求前都要 `<-burstyLimiter` 取一个令牌。前 3 个请求来的时候，桶里已经预填了 3 个令牌，所以**瞬间都能拿到**，几乎同时打印出来，时间戳肉眼看上去几乎一致——这就是「允许突发」的效果。第 4 个请求来时桶空了，得等 goroutine 下一次补充，大约 200ms 后才拿到令牌；第 5 个又得再等 200ms。

这个模式在流量控制领域叫**令牌桶算法（Token Bucket）**，是生产环境里用得最多的限流方式之一。

### 两种限流的本质区别

师兄给你对比总结一下，心里要有数：

- **漏桶（匀速限流）**：严格按 200ms 一个的节奏放行，上游再忙再闲都一样死板。优点是输出速率绝对稳定，缺点是没有任何弹性。
- **令牌桶（突发限流）**：平均速率一样是 5 次/秒，但空闲期积攒的令牌允许瞬间连续消费最多 3 个。这更贴合真实业务——流量天然有波峰波谷，用户短时间内连点几下也能顺畅通过，不会被硬卡。

真正在生产里写限流，一般不会自己手撸这套 channel + ticker，Go 官方提供了 `golang.org/x/time/rate` 包，里面的 `rate.Limiter` 就是一个完整的令牌桶实现，支持 `Allow`、`Wait`、`Reserve` 等多种使用姿势，并发安全，考虑了各种边界情况，直接用它就好。但这一关的原理你得清楚——看到 `rate.Limiter` 不能只知道调 API，得知道底下是一个 ticker 在给一个 buffered channel 补令牌，这样遇到问题才能往下排查。

## 🏃 跑一下试试
```bash
go run rate-limiting.go
```

预期输出（时间戳相对启动时刻，已简化）：

request 1 ... ~200ms
request 2 ... ~400ms
request 3 ... ~600ms
request 4 ... ~800ms
request 5 ... ~1000ms
request 1 ... 秒过（直接消耗缓冲令牌）
request 2 ... 秒过（直接消耗缓冲令牌）
request 3 ... 秒过（直接消耗缓冲令牌）
request 4 ... ~200ms 后（等补充令牌）
request 5 ... ~400ms 后（等补充令牌）

## 💡 师兄的碎碎念
① time.Tick 会在后台启动一个永不停止的 goroutine，函数返回后 channel 也不会被 GC，长期运行的服务里慎用，换成 time.NewTicker 并在用完后 ticker.Stop() 释放资源
② 生产环境别自己用 channel 手撸限流，直接上官方令牌桶 golang.org/x/time/rate，支持 Allow / Wait / Reserve 三种姿势，还能动态调整速率
③ 分布式场景单机限流根本不够用，多实例部署要把令牌桶状态存到 Redis，用 INCR + EXPIRE 或者 Lua 脚本保证原子性，才能全局限速
④ 别在每次请求处理函数里调 time.Tick，每调一次就泄漏一个 goroutine，limiter 应该在服务启动时初始化好，作为全局或依赖注入的单例传进来

## 🎓 这一关的知识点清单
☑ 第一段：创建 200ms 的 time.Tick channel，每次 <-limiter 阻塞直到 tick 到来，5 个请求匀速放行
☑ burstyLimiter 用带缓冲 channel（容量 3）+ 预填 3 个令牌，模拟突发窗口
☑ 启动 goroutine 每 200ms 往 burstyLimiter 补一个令牌，持续补充速率
☑ 前 3 个请求直接消耗缓冲令牌，几乎无等待；后 2 个各等 200ms
☑ 两段代码都是 <-limiter 语义一致，channel 满则阻塞，天然背压

## ➡️ 下一关
好，39 关速率限制咱就收尾了。下一关是第 40 关「原子计数器 atomic-counters」，讲的是多个 goroutine 并发读写同一个共享变量时，怎么用 sync/atomic 包做无锁原子操作，彻底告别数据竞争。比 mutex 轻量，场景对了性能差距很明显，值得好好看。

👉 [第 40 关：原子计数器](../40-atomic-counters/)
