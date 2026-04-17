# 第 37 关：工作池（worker-pools）（师兄带你学 Go）

## 🎯 这一关你会学到
工作池（worker pool）是一种经典的并发模式：预先启动固定数量的 goroutine 作为「工人」，让它们从同一个任务通道里抢活干，结果统一写进另一个通道。这样既限制了并发数量，又让多个任务能真正并行处理，是 Go 里用 goroutine + channel 实现生产者-消费者最直接的写法。

## 🤔 先想一个问题
想象一家饭店后厨只有 3 个厨师，前台把点单一张一张贴到出票口（jobs 通道）。三个厨师谁空了谁就去撕一张单子开始做菜，做完把菜端到出菜口（results 通道）。前台不用等某个厨师，厨师也不会互相踩脚——这就是工作池的日常。

## 📖 看代码
```go
// 在这个例子中，我们将看到如何使用协程与通道实现一个_工作池_。

package main

import (
	"fmt"
	"time"
)

// 这是 worker 程序，我们会并发的运行多个 worker。
// worker 将在 `jobs` 频道上接收工作，并在 `results` 上发送相应的结果。
// 每个 worker 我们都会 sleep 一秒钟，以模拟一项昂贵的（耗时一秒钟的）任务。
func worker(id int, jobs <-chan int, results chan<- int) {
	for j := range jobs {
		fmt.Println("worker", id, "started  job", j)
		time.Sleep(time.Second)
		fmt.Println("worker", id, "finished job", j)
		results <- j * 2
	}
}

func main() {

	// 为了使用 worker 工作池并且收集其的结果，我们需要 2 个通道。
	const numJobs = 5
	jobs := make(chan int, numJobs)
	results := make(chan int, numJobs)

	// 这里启动了 3 个 worker，
	// 初始是阻塞的，因为还没有传递任务。
	for w := 1; w <= 3; w++ {
		go worker(w, jobs, results)
	}

	// 这里我们发送 5 个 `jobs`，
	// 然后 `close` 这些通道，表示这些就是所有的任务了。
	for j := 1; j <= numJobs; j++ {
		jobs <- j
	}
	close(jobs)

	// 最后，我们收集所有这些任务的返回值。
	// 这也确保了所有的 worker 协程都已完成。
	// 另一个等待多个协程的方法是使用[WaitGroup](waitgroups)。
	for a := 1; a <= numJobs; a++ {
		<-results
	}
}
```

## 🔍 师兄给你逐行拆
我们先看 worker 函数的签名，这是整个模式的核心。

```go
func worker(id int, jobs <-chan int, results chan<- int) {
    for j := range jobs {
        fmt.Println("worker", id, "started  job", j)
        time.Sleep(time.Second)
        fmt.Println("worker", id, "finished job", j)
        results <- j * 2
    }
}
```

**参数里的方向限定：`<-chan int` 和 `chan<- int`**

`jobs <-chan int` 表示这个通道对 worker 来说是「只读」的——箭头指向 chan，意思是数据只能从通道里流出来给 worker 读。worker 在这个参数上只能做 `<-jobs` 这样的接收操作，编译器不允许它往里写。

`results chan<- int` 则相反，箭头从 chan 指出去，表示「只写」——worker 只能往 results 里发数据，不能从里面读。

这两个方向限定不是必须的，你完全可以传一个双向 channel 进来，代码也能跑。但加上方向限定之后，如果你在 worker 里不小心写了 `jobs <- 999` 或者 `<-results`，编译器会直接报错，而不是留一个隐蔽的 bug 到运行时。这是 Go 用类型系统帮你做的一道「护栏」，师兄强烈建议写通道参数时养成加方向的习惯。

**`for j := range jobs` 结合 `close` 自动退出**

`for j := range jobs` 是从通道里循环读值的惯用写法。它会一直阻塞等待新值，直到通道被关闭且里面的值全部取完，循环才自动结束，worker 函数返回。

这里有一个关键点：**必须有人调用 `close(jobs)`**，否则 worker 会永远阻塞在 range 那里，造成 goroutine 泄漏。后面我们会看到，main 函数在发完所有任务之后立刻 `close(jobs)`，正是为了让三个 worker 能在做完活之后正常退出。

**`time.Sleep` 模拟耗时**

`time.Sleep(time.Second)` 在这里代表「做这个任务需要 1 秒」。真实场景里可能是一次数据库查询、一次 HTTP 请求、一次图片压缩，总之是某种 I/O 或计算密集操作。用 Sleep 把逻辑简化掉，让我们能专注看并发结构本身。

**在 main 里启动 3 个 worker**

```go
const numJobs = 5
jobs := make(chan int, numJobs)
results := make(chan int, numJobs)

for w := 1; w <= 3; w++ {
    go worker(w, jobs, results)
}
```

三个 `go worker(...)` 启动了三个 goroutine，它们几乎同时开始执行 `for j := range jobs`。但此时 jobs 通道里还没有任何数据，所以三个 worker 全部阻塞在等待上——就像三个厨师站在出票口盯着空空的出票夹。

**发 5 个 job 再 `close(jobs)`**

```go
for j := 1; j <= numJobs; j++ {
    jobs <- j
}
close(jobs)
```

往 jobs 里发 5 个整数。因为 jobs 是带缓冲的（容量 numJobs=5），这个循环可以不阻塞地一口气把 5 个任务塞进去，然后立刻 `close(jobs)`。

close 之后，三个一直阻塞等待的 worker 开始「抢单」：worker 1 拿到任务 1，worker 2 拿到任务 2，worker 3 拿到任务 3，几乎同时开始 Sleep 1 秒。等它们做完，jobs 里还剩任务 4 和 5，谁先做完谁就去拿。close 信号让 range 知道「没有更多任务了」，worker 做完最后一个任务后 range 循环自然退出。

**用循环 `<-results` 收 5 次结果——既拿结果又做同步**

```go
for a := 1; a <= numJobs; a++ {
    <-results
}
```

这个循环做了两件事。第一，把 5 个结果从通道里取走（本例没有用这些值，实际项目里你会处理它们）。第二，也是更重要的：**这里承担了等待所有 worker 完成的同步职责**。

每次 `<-results` 都会阻塞，直到有 worker 往里写了一个结果。5 次接收全部完成，说明 5 个任务全部做完，main 函数才往下走。这是一种轻量的同步技巧——用「消费掉所有结果」来代替显式的 WaitGroup。当然，如果你不在乎结果本身，用 `sync.WaitGroup` 语义更清晰；但当你本来就需要收集结果时，这种写法一举两得。

**为什么总耗时约 2 秒而不是 5 秒**

5 个任务，每个 1 秒，串行跑需要 5 秒。但我们有 3 个 worker 并行：

- 第 1 秒：worker 1/2/3 同时处理任务 1/2/3
- 第 2 秒：worker 1/2/3 里最先做完的两个接着处理任务 4/5

所以总时间约 2 秒。这就是工作池的价值——用有限的并发资源（3 个 goroutine）把总耗时从 O(n) 压到 O(n/worker数)。

**为什么 jobs 用带缓冲的 channel（容量 numJobs）**

如果 jobs 是无缓冲的，`jobs <- j` 每次都需要等某个 worker 来接收才能返回。而 worker 每个任务要花 1 秒，发任务的循环会被严重拖慢——实际上变成了生产者被消费者节奏控制。

用容量为 numJobs（这里是 5）的带缓冲 channel，生产者可以把所有任务一次性塞进去立刻返回，然后调用 `close(jobs)`，worker 们按自己的节奏去取。生产者和消费者在时间上解耦了。

当然，缓冲大小是个权衡：太小会让生产者频繁阻塞，太大会占用更多内存。这里选 numJobs 是「刚好能装下所有任务」的最简单策略，真实系统里你可能会根据内存和延迟要求调整。

## 🏃 跑一下试试
```bash
go run worker-pools.go
```
预期输出（顺序因调度而异，但规律是：先 3 个 worker 各认领 1 个 job，约 1 秒后同时 finished 再各认领剩余 job）：

worker 1 started  job 1
worker 2 started  job 2
worker 3 started  job 3
worker 1 finished job 1
worker 1 started  job 4
worker 2 finished job 2
worker 2 started  job 5
worker 3 finished job 3
worker 1 finished job 4
worker 2 finished job 5

结果收到的值是 job 编号 × 2（2 4 6 8 10），顺序不定。
总耗时约 2 秒（5 个 job ÷ 3 并发 ≈ 2 轮，每轮 1 秒）。

## 💡 师兄的碎碎念
① 并发数控制是工作池的核心价值——只开固定数量的 worker，绝对别对每个 job 无脑 go func()，任务量一大直接炸资源。
② jobs channel 要带缓冲（make(chan int, numJobs)），main 才能一口气把 5 个 job 全塞进去不阻塞，否则塞第 1 个就卡住了。
③ 塞完所有 job 之后必须 close(jobs)，worker 里的 for j := range jobs 才知道没活了可以退出，忘了 close 就永远 range 死等。
④ 这一关用

## 🎓 这一关的知识点清单
✅ make(chan int, numJobs) 带缓冲，main 塞完不阻塞
✅ for w := 1; w <= 3; w++ 启动 3 个 worker goroutine
✅ close(jobs) 在塞完之后调用，worker range 才能正常退出
✅ <-results 循环 numJobs 次，把结果全收走当

## ➡️ 下一关
第 38 关「WaitGroup」——这一关我们靠 results channel 凑数来等 worker，下一关用 sync.WaitGroup 更正式、更通用地等一批 goroutine 全部退出，不依赖结果 channel 的数量，适合不需要返回值的并发任务。

👉 [第 38 关：WaitGroup](../38-waitgroups/)
