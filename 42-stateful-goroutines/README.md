# 第 42 关：状态协程（stateful-goroutines）（师兄带你学 Go）

## 🎯 这一关你会学到
这一关展示 Go「通过通信共享内存」的核心哲学：把原本需要 mutex 保护的 state map 交给一个专职 goroutine 独占持有，所有读写都变成向 channel 发消息、等 channel 回消息的协议。这样做的好处是状态的所有权始终清晰，你永远不会忘记加锁或者加错锁，因为根本没有锁。

## 🤔 先想一个问题
你去银行取钱，柜台只有一个柜员坐镇，他面前放着一本账本（state map）。你（reader goroutine）走到叫号窗口，把一张写着「查余额，账户 42」的纸条（readOp）推进去，同时递进去一个回执信封（resp chan）；柜员看完纸条，把余额写进信封推回来，你拆开信封拿到数字，离开。另一个人（writer goroutine）排队递进去「存款 100，账户 7」的纸条（writeOp），柜员处理完同样推回确认信封。关键在于：账本永远只在柜员手里，任何人都不能绕过窗口直接翻账本——这就是状态协程模型，账本不共享，只共享通信窗口。

## 📖 看代码
```go
// 在前面的例子中，我们用 [互斥锁](mutexes) 进行了明确的锁定，
// 来让共享的 state 跨多个 Go 协程同步访问。
// 另一个选择是，使用内建协程和通道的同步特性来达到同样的效果。
// Go 共享内存的思想是，通过通信使每个数据仅被单个协程所拥有，即通过通信实现共享内存。
// 基于通道的方法与该思想完全一致！

package main

import (
	"fmt"
	"math/rand"
	"sync/atomic"
	"time"
)

// 在这个例子中，state 将被一个单独的协程拥有。
// 这能保证数据在并行读取时不会混乱。
// 为了对 state 进行读取或者写入，
// 其它的协程将发送一条数据到目前拥有数据的协程中，
// 然后等待接收对应的回复。
// 结构体 `readOp` 和 `writeOp` 封装了这些请求，并提供了响应协程的方法。
type readOp struct {
	key  int
	resp chan int
}
type writeOp struct {
	key  int
	val  int
	resp chan bool
}

func main() {

	// 和前面的例子一样，我们会计算操作执行的次数。
	var readOps uint64
	var writeOps uint64

	// 其他协程将通过 `reads` 和 `writes` 通道来发布 `读` 和 `写` 请求。
	reads := make(chan readOp)
	writes := make(chan writeOp)

	// 这就是拥有 `state` 的那个协程，
	// 和前面例子中的 map 一样，不过这里的 state 是被这个状态协程私有的。
	// 这个协程不断地在 `reads` 和 `writes` 通道上进行选择，并在请求到达时做出响应。
	// 首先，执行请求的操作；然后，执行响应，在响应通道 `resp` 上发送一个值，表明请求成功（`reads` 的值则为 state 对应的值）。
	go func() {
		var state = make(map[int]int)
		for {
			select {
			case read := <-reads:
				read.resp <- state[read.key]
			case write := <-writes:
				state[write.key] = write.val
				write.resp <- true
			}
		}
	}()

	// 启动 100 个协程通过 `reads` 通道向拥有 state 的协程发起读取请求。
	// 每个读取请求需要构造一个 `readOp`，发送它到 `reads` 通道中，
	// 并通过给定的 `resp` 通道接收结果。
	for r := 0; r < 100; r++ {
		go func() {
			for {
				read := readOp{
					key:  rand.Intn(5),
					resp: make(chan int)}
				reads <- read
				<-read.resp
				atomic.AddUint64(&readOps, 1)
				time.Sleep(time.Millisecond)
			}
		}()
	}

	// 用相同的方法启动 10 个写操作。
	for w := 0; w < 10; w++ {
		go func() {
			for {
				write := writeOp{
					key:  rand.Intn(5),
					val:  rand.Intn(100),
					resp: make(chan bool)}
				writes <- write
				<-write.resp
				atomic.AddUint64(&writeOps, 1)
				time.Sleep(time.Millisecond)
			}
		}()
	}

	// 让协程们跑 1s。
	time.Sleep(time.Second)

	// 最后，获取并报告 `ops` 值。
	readOpsFinal := atomic.LoadUint64(&readOps)
	fmt.Println("readOps:", readOpsFinal)
	writeOpsFinal := atomic.LoadUint64(&writeOps)
	fmt.Println("writeOps:", writeOpsFinal)
}
```

## 🔍 师兄给你逐行拆
### 开场：为什么要学这个模式

师兄先给你立一下这关的意义。前面咱们用 sync.Mutex 解决了并发读写 map 的问题，代码也能跑，为什么还要再学一个新套路？

因为 Go 官方有一句非常出名的箴言：**"Do not communicate by sharing memory; instead, share memory by communicating."**（不要通过共享内存来通信，而要通过通信来共享内存）。mutex 是典型的「共享内存 + 加锁」，而状态协程是典型的「通过通信共享内存」——两种思路都能达到目的，但哲学完全不同。

状态协程的核心玩法是：**让一个专属 goroutine 独占持有 state，其他所有 goroutine 想读写 state，必须通过 channel 发送请求、等 channel 回复**。state 的所有权永远只在一个 goroutine 手里，你连加锁的机会都没有——因为根本没锁可加。

### 请求结构体：readOp 和 writeOp

```go
type readOp struct {
    key  int
    resp chan int
}
type writeOp struct {
    key  int
    val  int
    resp chan bool
}
```

每个请求都带一个 `resp` 通道。读请求的 resp 是 `chan int`，用来接收读到的值；写请求的 resp 是 `chan bool`，用来接收"写完了"的确认信号。

注意这个 `resp` 是**每次请求新建的**，而不是复用的全局 channel。为什么？因为有成百上千个 goroutine 并发发请求，如果共用一个 resp 通道，A 发的请求被 B 收到的概率极大，回复就串线了。每个请求自带一个私有回执信封，各回各家，绝对不会错。

### 状态协程主体：独占 state 的 goroutine

```go
go func() {
    var state = make(map[int]int)
    for {
        select {
        case read := <-reads:
            read.resp <- state[read.key]
        case write := <-writes:
            state[write.key] = write.val
            write.resp <- true
        }
    }
}()
```

这个 goroutine 启动后会做两件事，周而复始：① 在 reads 和 writes 两个通道之间 select，谁先来消息处理谁；② 处理完通过请求里自带的 resp 通道把结果或确认写回去。

注意 `state` 是声明在这个 goroutine 内部的局部变量，外面根本没法碰到它。这就是状态协程模型的精髓——**state 的生命周期完全被这个 goroutine 独占**，别人只能发请求，不能直接操作。

### 100 个 reader + 10 个 writer 并发

main 里紧接着起了两批 goroutine：

```go
for r := 0; r < 100; r++ {
    go func() {
        for {
            read := readOp{
                key:  rand.Intn(5),
                resp: make(chan int),
            }
            reads <- read
            <-read.resp
            atomic.AddUint64(&readOps, 1)
            time.Sleep(time.Millisecond)
        }
    }()
}
```

100 个 reader 各自在一个无限循环里：构造 readOp（每次新建一个 resp channel）→ 推进 reads 通道 → 阻塞在 <-read.resp 等回复 → 拿到值后把原子计数器 readOps 加 1 → 休息 1ms 继续。

writer 类似，只是 10 个，操作改成写。

### 最后等 1 秒打印统计

main 主协程 `time.Sleep(time.Second)` 睡 1 秒，让 reader/writer 狂跑，然后用 `atomic.LoadUint64` 安全读出 readOps 和 writeOps 打印。

因为 reader 有 100 个、sleep 1ms，writer 只有 10 个、sleep 1ms，1 秒内理论上 reader 大约执行 `100 * 1000 / 1 ≈ 100000` 次，writer 约 10000 次，实际会因为 state goroutine 串行处理变小一点，常见结果 readOps 约 70000、writeOps 约 7000，每次跑数字都略有浮动。

### 状态协程 vs mutex 的选择

简单归纳一下：

- **mutex**：临界区短、逻辑简单（比如计数器、简单 map）时首选，代码最少，性能也好
- **状态协程**：当 state 是一个**复杂的状态机**、有多种请求类型、处理逻辑分散时更清晰——所有状态相关逻辑集中在一个 `for { select { } }` 里，职责明确

你不用两者都用，场景合适才用。Go 给了你选择权。

## 🏃 跑一下试试
```bash
# 运行命令
go run stateful-goroutines.go

# 预期输出（并发随机，数字每次略有不同）
readOps: 71708
writeOps: 7177

# 说明：程序跑约 1 秒后退出，readOps 约 7 万级别（100 个 reader）
# writeOps 约 7 千级别（10 个 writer），比例稳定在 10:1 左右
```

## 💡 师兄的碎碎念
① 何时选 mutex、何时选状态协程：简单的临界区（保护一个计数器、一个标志位）用 sync.Mutex 更直接，代码更少；当你的「状态」本身是一个复杂的状态机、需要根据请求类型做不同处理时，状态协程更清晰，因为所有逻辑集中在一个 select 里，不会散落在各处的 Lock/Unlock 之间。
② reads 和 writes channel 建议带缓冲：如果 state goroutine 处理一条消息时稍微慢一拍，无缓冲 channel 会让所有 reader/writer 全部阻塞排队；给 channel 加适当缓冲（容量等于并发量级别）可以起到「消峰」作用，吞吐更平稳。
③ resp channel 必须每个请求独立新建且容量为 1：若复用同一个 resp channel，多个请求的回应会互相覆盖或错位；容量设为 1 保证 state goroutine 写完就走，不会因为发送方还没来得及读而阻塞整条流水线。
④ 读写混合场景要注意「读到旧值」问题：状态协程模型是串行处理请求的，但如果你在读完之后、用读到的值做判断再发写请求，两次请求之间 map 可能已经被其他 writer 改过，这和数据库的「不可重复读」是同一个问题，必要时需要把「读-判断-写」合并成一条原子操作消息类型。

## 🎓 这一关的知识点清单
☑ readOp / writeOp 结构体各含 resp chan，每次请求都新建容量为 1 的 channel，防止 state goroutine 回写时互相串台
☑ state goroutine 用 for { select } 独占 map，绝不在其他 goroutine 里直接读写该 map
☑ reads 和 writes channel 建议设缓冲（如 make(chan readOp, 100)），避免大量并发 reader/writer 把 state goroutine 撑爆
☑ atomic 计数器（readOps / writeOps）只用于统计总量，不涉及 state map 本身，职责分离
☑ time.Sleep(time.Millisecond) 控制并发压力，去掉后 goroutine 会把 CPU 打满，数字飙升但不影响正确性

## ➡️ 下一关
第 42 关是本系列 30-42 关的收官，我这个 2 号 agent 的任务到这里圆满结束；从第 43 关「排序 sorting」开始，会有新的 agent 师兄接力继续带你拆解 Go by Example，放心跟上就好。
