# 第 31 关：超时处理（timeouts）（师兄带你学 Go）

## 🎯 这一关你会学到
这关咱们学的是怎么用 select 配合 time.After 给异步操作加一个「等待上限」。核心思路就一句话：同时监听「结果通道」和「计时器通道」，谁先来就处理谁。超时了就直接走超时分支，不傻等了。

## 🤔 先想一个问题
兄弟，你有没有遇到过这种情况——点了外卖，等了四十分钟还没动静，你肯定不会无限等下去对吧？等到某个点你就直接取消订单走人了。程序里也一样，调一个外部接口，网络抖一下可能卡住，你总不能让整个服务就这么挂在那儿傻等。「等太久就不等了」，这就是超时处理要干的事，Go 用 select + time.After 把这件事做得特别优雅。

## 📖 看代码
```go
// _超时_ 对于一个需要连接外部资源，
// 或者有耗时较长的操作的程序而言是很重要的。
// 得益于通道和 `select`，在 Go 中实现超时操作是简洁而优雅的。

package main

import (
	"fmt"
	"time"
)

func main() {

	// 在这个例子中，假如我们执行一个外部调用，
	// 并在 2 秒后使用通道 `c1` 返回它的执行结果。
	c1 := make(chan string, 1)
	go func() {
		time.Sleep(2 * time.Second)
		c1 <- "result 1"
	}()

	// 这里是使用 `select` 实现一个超时操作。
	// `res := <- c1` 等待结果，`<-time.After` 等待超时（1秒钟）以后发送的值。
	// 由于 `select` 默认处理第一个已准备好的接收操作，
	// 因此如果操作耗时超过了允许的 1 秒的话，将会执行超时 case。
	select {
	case res := <-c1:
		fmt.Println(res)
	case <-time.After(1 * time.Second):
		fmt.Println("timeout 1")
	}

	// 如果我们允许一个长一点的超时时间：3 秒，
	// 就可以成功的从 `c2` 接收到值，并且打印出结果。
	c2 := make(chan string, 1)
	go func() {
		time.Sleep(2 * time.Second)
		c2 <- "result 2"
	}()
	select {
	case res := <-c2:
		fmt.Println(res)
	case <-time.After(3 * time.Second):
		fmt.Println("timeout 2")
	}
}
```

## 🔍 师兄给你逐行拆
### 第一段：带缓冲的通道——防止 goroutine 泄漏

```go
c1 := make(chan string, 1)
```

师兄先盯着这个 `1` 说一下，别以为随手写的。这里用的是**带 1 个缓冲槽的通道**，不是无缓冲通道。为什么要带缓冲？

你想想，goroutine 里会在 2 秒后往 `c1` 发一个 `"result 1"`。但如果超时分支先触发了，`select` 已经跑去执行 `fmt.Println("timeout 1")` 了，主流程压根就不再接收 `c1` 里的值了。

这时候那个 goroutine 还活着，还在 `time.Sleep`，2 秒到了它想往 `c1` 写，但如果 `c1` 是无缓冲的，没人接收，它就会**永远阻塞在那里**，这就是经典的 goroutine 泄漏。

换成缓冲为 1 的通道就不一样了——goroutine 写值的时候不需要有人同时在接收，直接把值塞进缓冲槽，写完就结束，goroutine 正常退出。虽然那个值后来没人读，但 goroutine 本身释放了，内存不会一直占着。这个细节在生产代码里非常重要，师兄见过不少人写超时逻辑把这个 `1` 漏掉，压力大了之后 goroutine 数量蹭蹭往上涨。

---

### 第二段：goroutine 模拟耗时外部调用

```go
go func() {
    time.Sleep(2 * time.Second)
    c1 <- "result 1"
}()
```

这段就是在模拟「调一个很慢的外部服务」。用 `go func()` 开一个 goroutine，里面先睡 2 秒，再把结果塞进 `c1`。

现实场景里，这个 goroutine 里放的可能是 HTTP 请求、数据库查询、RPC 调用之类的。这里用 `time.Sleep` 来模拟延迟，干净直观。注意这是异步的，`go` 关键字一写，主流程不会等它，直接往下走到 `select`。

---

### 第三段：第一次 select——1 秒超时，等不到就跑

```go
select {
case res := <-c1:
    fmt.Println(res)
case <-time.After(1 * time.Second):
    fmt.Println("timeout 1")
}
```

这里是整个例子的核心，咱们拆开慢慢说。

`select` 会**同时监听多个通道**，哪个先准备好就走哪个分支。这里监听了两个：

**分支一**：`res := <-c1`，等 goroutine 把结果塞进来。但那个 goroutine 要 2 秒才会发，所以这个分支短期内不会就绪。

**分支二**：`<-time.After(1 * time.Second)`，这才是超时控制的关键。

`time.After` 这个函数师兄重点说一下。它的签名是这样的：

```go
func After(d Duration) <-chan Time
```

它返回的是一个**只读通道**（`<-chan Time`），在指定的时间 `d` 过后，它会往这个通道发送一个 `time.Time` 类型的值。咱们这里不关心那个值是什么，只关心「它发了」这个信号。

所以 `select` 开始等待的时候，同时开了两个计时：一边等 `c1` 有没有结果进来，一边等 1 秒计时器到没到。

结果是：goroutine 要 2 秒，计时器只要 1 秒，**计时器先到**，所以走第二个 case，打印 `timeout 1`，`select` 结束，程序继续往下走。

这就是「等太久就不等了」的实现。整个逻辑清晰，没有任何回调嵌套，没有专门起一个「看门狗」线程，一个 select 搞定。

---

### 第四段：第二次 select——3 秒超时，能等到结果

```go
c2 := make(chan string, 1)
go func() {
    time.Sleep(2 * time.Second)
    c2 <- "result 2"
}()
select {
case res := <-c2:
    fmt.Println(res)
case <-time.After(3 * time.Second):
    fmt.Println("timeout 2")
}
```

这段逻辑和第一段几乎一样，但把超时时间改成了 3 秒。goroutine 还是 2 秒后发结果，但超时允许等 3 秒，所以 `c2` 里的 `"result 2"` 在 2 秒的时候就准备好了，**比 3 秒的计时器更早就绪**，select 走第一个 case，打印 `result 2`。

两段代码对比一看，逻辑一目了然：超时设短了等不到，超时设长了能拿到结果。实际项目里，这个超时时间是你根据业务来定的，调接口一般是几百毫秒到几秒不等。

---

### 和第 30 关（select）有啥区别？

第 30 关咱们学的是 select 的基础用法——监听多个通道，谁准备好走谁，没准备好就阻塞着。核心在于「多路复用」本身的机制。

这关是 select 的一个**具体应用场景**：把 `time.After` 返回的通道作为 select 的一个 case，专门用来做超时控制。

换句话说，第 30 关告诉你 select 能干什么，这关告诉你 select 在工程里怎么用。`time.After` 把「时间」也变成了一个通道信号，和业务结果通道并列参与竞争，这是 Go 通道模型设计哲学的一个很典型的体现：**一切皆通道，时间也是**。

这种模式兄弟要熟，后面写 HTTP 中间件、写 RPC 客户端、做任务队列，到处都会用到。

## 🏃 跑一下试试
```bash
# 跑一下，感受两次 select 的不同命运
go run timeouts.go

# 预期输出：
# timeout 1      ← 第一次：1秒超时到了，result 还没来，直接超时
# result 2       ← 第二次：3秒够用，2秒时 result 先到，成功拿到结果
```

## 💡 师兄的碎碎念
- ⚠️ **time.After 在高频循环里慎用**：每次调用都会创建一个新 timer 对象，超时前这个 timer 不会被 GC 回收，循环跑多了内存蹭蹭涨。Go 1.23 之前尤其明显，建议换成 `time.NewTimer(d)` + 用完记得 `timer.Stop()`，或者直接上 `context.WithTimeout`
- 🔒 **缓冲大小直接决定 goroutine 会不会泄漏**：咱们例子里 `make(chan string, 1)` 是有缓冲的，超时后接收方跑路了，发送方还是能把结果塞进去然后正常退出；要是改成无缓冲的，发送方就会永远卡在那儿等人来收，goroutine 泄漏就来了
- 🏆 **真实业务请用 context 传超时**：`context.WithTimeout` / `context.WithDeadline` 可以跨函数、跨 goroutine 传递超时信号，整条调用链都能感知，比在每个 select 里单独写 time.After 优雅多了，也方便统一取消

## 🎓 这一关的知识点清单
- [ ] `select` 同时监听多个 channel，哪个先来数据就走哪个分支，这是超时控制的核心
- [ ] `time.After(d)` 返回一个 channel，d 时间后会往里塞一个值，配合 select 就能做超时
- [ ] 第一次 select：操作 channel 要 2 秒才有结果，但超时设 1 秒，所以走 timeout 分支
- [ ] 第二次 select：操作 channel 要 2 秒才有结果，超时设 3 秒，2 秒时结果先到，所以拿到 result 2
- [ ] 发送方 goroutine 里用带缓冲的 channel（`make(chan string, 1)`），即使接收方已经因超时离开，goroutine 也不会永远卡着——这是防泄漏的关键细节

## ➡️ 下一关
下一关第 32 关「非阻塞通道操作 non-blocking-channel-operations」，师兄带你看 select + default 这个组合——不想傻等 channel 有没有数据，default 分支直接让你优雅地溜走，兄弟们准备好了吗？

👉 [第 32 关：非阻塞通道操作](../32-non-blocking-channel-operations/)
