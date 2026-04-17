# 第 38 关：WaitGroup（等待协程组）（师兄带你学 Go）

## 🎯 这一关你会学到
sync.WaitGroup 是 Go 标准库里专门用来「等所有 goroutine 都跑完再继续」的计数器。你每启动一个 goroutine 就往里加一，goroutine 结束时减一，主程序在 Wait() 这里卡住，直到计数器归零才放行。它不关心 goroutine 跑出了什么结果，只关心「人到齐了没有」。

## 🤔 先想一个问题
你当过图书馆管理员或者自习室组长就懂这种感觉——快关门了，你站在门口数人头，每走一个人你在本子上划掉一笔，等所有人都出去了你才能锁门走人。sync.WaitGroup 干的就是这件事，它就是你手里那个计数本，Add 是记下「又来一个还没走的人」，Done 是「这个人出去了划掉一笔」，Wait 是「我站在门口等，没走完我不动」。

## 📖 看代码
```go
// 想要等待多个协程完成，我们可以使用 *wait group* 。

package main

import (
	"fmt"
	"sync"
	"time"
)

// 每个协程都会运行该函数。
func worker(id int) {
	fmt.Printf("Worker %d starting\n", id)

	// 睡眠一秒钟，以此来模拟耗时的任务。
	time.Sleep(time.Second)
	fmt.Printf("Worker %d done\n", id)
}

func main() {

	// 这个 WaitGroup 用于等待这里启动的所有协程完成。
	// 注意：如果 WaitGroup 显式传递到函数中，则应使用 *指针* 。
	var wg sync.WaitGroup

	// 启动几个协程，并为其递增 WaitGroup 的计数器。
	for i := 1; i <= 5; i++ {
		wg.Add(1)
		// 避免在每个协程闭包中重复利用相同的 `i` 值
		// 更多细节可以参考 [the FAQ](https://golang.org/doc/faq#closures_and_goroutines)
		i := i

		// 将 worker 调用包装在一个闭包中，可以确保通知 WaitGroup 此工作线程已完成。
		// 这样，worker 线程本身就不必知道其执行中涉及的并发原语。
		go func() {
			defer wg.Done()
			worker(i)
		}()
	}

	// 阻塞，直到 WaitGroup 计数器恢复为 0；
	// 即所有协程的工作都已经完成。
	wg.Wait()

	// 请注意，WaitGroup 这种使用方式没有直观的办法传递来自 worker 的错误。
	// 更高级的用例，请参见 [errgroup package](https://pkg.go.dev/golang.org/x/sync/errgroup)
}
```

## 🔍 师兄给你逐行拆
先回顾一下上一关（第 37 关，Worker Pool）是怎么「等所有 goroutine 完成」的。那里开了一个带缓冲的 results 通道，然后在 main 里用一个 for 循环收了 numJobs 次结果，用收完结果这个动作来隐式地确认所有 worker 都跑完了。这个方案在「每个 goroutine 都要汇报一个结果」的场景下很好用，但如果你的 goroutine 根本不需要返回值——比如只是并发地写文件、打日志、发请求——你为了等它们完成还要专门造一个通道、往里塞占位符，就显得很别扭。WaitGroup 就是为这种场景生的。

**sync.WaitGroup 的零值可用，不需要 new 或 make**

Go 里有一批类型是「零值即可用」的，sync.WaitGroup 是其中之一，sync.Mutex 也是。你只需要声明：

```go
var wg sync.WaitGroup
```

这就够了，wg 内部的计数器初始是 0，直接可以用。不要写 `wg := new(sync.WaitGroup)` 或者 `wg := make(...)` 之类的，虽然 new 版本功能上没问题（返回指针），但惯用写法就是直接 var 声明，简洁清晰。

**wg.Add(1) 要在启动 goroutine 之前调用**

这是一个容易踩的时序问题。正确的顺序是：先 Add，再 go。

```go
for i := 0; i < 5; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        // 做事
    }()
}
```

如果你把 Add 写到 goroutine 内部第一行，理论上看起来也对，但存在竞态：主程序可能在 goroutine 还没来得及执行 Add 之前就已经跑到 Wait 了，此时计数器还是 0，Wait 直接放行，后面的 goroutine 还在跑，程序就提前结束了。所以铁律是：Add 在 go 语句之前，在主程序这一侧调用。

**defer wg.Done() 放在 goroutine 的第一行**

goroutine 内部，第一行就写 `defer wg.Done()`，这是惯用法，原因是 defer 是在函数返回时执行，不管是正常 return 还是 panic 都会触发。如果你把 wg.Done() 写在函数最后一行而不是 defer，那万一中途 panic 或者某个分支提前 return，Done 就永远不会执行，WaitGroup 的计数器永远不会归零，Wait 就永远阻塞，程序就挂死了。defer 帮你兜底，不管出口在哪，Done 一定会被调用。

**wg.Wait() 阻塞到计数器归零**

wg.Wait() 会把当前 goroutine（通常是 main）卡在这一行，直到 WaitGroup 内部的计数器减到 0 为止。每一次 goroutine 调用 Done，计数器减一，当最后一个 Done 让计数器变成 0，Wait 就解除阻塞，主程序继续往下跑。整个语义非常直白，没有任何隐式的魔法。

**i := i——Go 1.22 之前的闭包循环变量坑**

这是 Go 初学者几乎人人都会踩的坑，放在这里必须讲清楚。考虑这段代码（Go 1.21 及以前）：

```go
for i := 0; i < 5; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        fmt.Println(i) // 危险！
    }()
}
```

for 循环里的 i 在整个循环过程中是同一个变量，goroutine 里的闭包捕获的是这个变量的引用，而不是它当时的值。等这些 goroutine 真正开始执行的时候，循环早就跑完了，i 已经是 5，所以你大概率会看到打印出来五个 5，而不是 0 1 2 3 4。

修法有两种。第一种，在循环体内用 `i := i` 创建一个同名的局部变量，遮蔽外层的 i，让闭包捕获这个局部副本：

```go
for i := 0; i < 5; i++ {
    i := i // 关键行
    wg.Add(1)
    go func() {
        defer wg.Done()
        fmt.Println(i) // 捕获的是局部副本，安全
    }()
}
```

第二种，把 i 作为参数传给匿名函数，传参是值拷贝，同样安全：

```go
for i := 0; i < 5; i++ {
    wg.Add(1)
    go func(id int) {
        defer wg.Done()
        fmt.Println(id)
    }(i)
}
```

Go 1.22 开始，官方修改了 for 循环的语义，每次迭代 i 都是一个新变量，这个坑被从语言层面填掉了。但如果你的项目 go.mod 里写的是 go 1.21 或更早，这个坑依然存在，老代码也要注意。

**传 WaitGroup 必须传指针**

如果你要把 WaitGroup 传进一个函数或者 goroutine，必须传指针，绝对不能传值。

```go
// 错误写法
func doWork(wg sync.WaitGroup) {
    defer wg.Done() // 操作的是副本，主程序的计数器根本没变
}

// 正确写法
func doWork(wg *sync.WaitGroup) {
    defer wg.Done() // 操作的是同一个 WaitGroup
}
```

sync.WaitGroup 是结构体，Go 传参是值拷贝，传值进去相当于每个 goroutine 拿到了一个全新的独立 WaitGroup，它们各自 Done 各自的副本，主程序 Wait 的那个计数器根本感知不到这些操作，永远不会归零，死锁。传指针才能让所有人共享同一个计数器。直接声明在 main 里然后用闭包捕获也是可以的，闭包捕获的是变量本身（引用语义），不存在这个问题。

**和第 37 关 results channel 凑数方案对比**

第 37 关用 `for a := 1; a <= numJobs; a++ { <-results }` 来等所有 worker 完成，本质是靠「结果的数量等于任务的数量」这个假设来凑数。这要求每个 goroutine 必须恰好往 results 里发一个值，多了少了都会出问题，而且你必须提前知道总数。WaitGroup 方案完全不依赖「有没有结果」「结果有几个」，只问「人走完了没有」，两者的关注点完全不同，各有适用场景，理解清楚了按需选用就好。

## 🏃 跑一下试试
```bash
cat << 'EOF' > waitgroup.go
package main

import (
    "fmt"
    "sync"
    "time"
)

func worker(id int, wg *sync.WaitGroup) {
    defer wg.Done()
    fmt.Printf("Worker %d starting\n", id)
    time.Sleep(time.Second)
    fmt.Printf("Worker %d done\n", id)
}

func main() {
    var wg sync.WaitGroup
    start := time.Now()
    for i := 1; i <= 5; i++ {
        wg.Add(1)
        go worker(i, &wg)
    }
    wg.Wait()
    fmt.Printf("总耗时：%v\n", time.Since(start).Round(time.Millisecond))
}
EOF
go run waitgroup.go

# 预期输出（starting 顺序随机，done 顺序也随机）：
# Worker 1 starting
# Worker 2 starting
# Worker 3 starting
# Worker 4 starting
# Worker 5 starting
# Worker 3 done
# Worker 1 done
# Worker 5 done
# Worker 2 done
# Worker 4 done
# 总耗时：1.001s   ← 5 个并发，约 1 秒而不是 5 秒
```

## 💡 师兄的碎碎念
💡 Add 必须在 go 语句之前调用——如果放进 goroutine 里面，主协程可能在 Add 执行前就跑到 Wait，直接漏掉等待，属于典型竞态。
💡 goroutine 里永远用 defer wg.Done()，别手写在函数末尾——一旦中途 panic 或提前 return，Done 没调到，Wait 就永远卡住。
💡 Go 1.22+ 已修复 for 循环变量捕获问题，但老项目（go.mod 里 go 1.21 及以下）还是要写 i := i 再传进去，否则所有 goroutine 拿到的是同一个变量的最终值。
💡 需要收集 goroutine 错误？别自己用 channel 手拼，直接上 golang.org/x/sync/errgroup，代码干净一半；wg.Add 传负数或 Done 多调一次会直接 panic，errgroup 帮你规避了不少这类坑。

## 🎓 这一关的知识点清单
✅ wg.Add(n) 在 goroutine 启动前调用了吗？
✅ goroutine 内部用了 defer wg.Done() 吗？
✅ main 里 wg.Wait() 放在所有 go 语句之后了吗？
✅ 跑了几次，确认输出顺序确实每次不一样（调度随机）？
✅ 把 sleep 去掉后再跑一次，感受下没有耗时任务时的调度行为？

## ➡️ 下一关
第 39 关「速率限制 rate-limiting」——下一关咱们看怎么用 time.Ticker 配合 channel，把请求速率压在一个固定节奏上。限流是生产里非常常见的手段，ticker 怎么搭、带缓冲 channel 怎么做「突发允许」，都会讲到，跟上来。

👉 [第 39 关：速率限制](../39-rate-limiting/)
