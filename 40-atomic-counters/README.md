# 第 40 关：原子计数器（atomic-counters）（师兄带你学 Go）

## 🎯 这一关你会学到
这一关演示如何用 `sync/atomic` 包里的原子操作，替代互斥锁来实现线程安全的计数器。核心场景是 50 个 goroutine 并发地各自对同一个变量累加 1000 次，最终结果必须精确等于 50000，一个都不能少。

## 🤔 先想一个问题
想象群里有人发了一张限量红包图，50 个朋友同时点「赞」，微信要在某个内存格子里把点赞数从 0 一路加到 50000。如果每个人各自「读出当前数→加一→写回去」，两个人恰好同时读到同一个旧值，两人都写回「旧值+1」，就白白丢掉一次点赞——就好像电梯门口 50 人同时按楼层计数按钮，但计数器只记住了其中几下。`sync/atomic` 的作用，就是让每一次「按钮」动作都是硬件级别的原子操作，绝对不会互相踩踏。

## 📖 看代码
```go
// Go 中最主要的状态管理机制是依靠通道间的通信来完成的。
// 我们已经在[工作池](worker-pools)的例子中看到过，并且，还有一些其他的方法来管理状态。
// 这里，我们来看看如何使用 `sync/atomic` 包在多个协程中进行 _原子计数_。

package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {

	// 我们将使用一个无符号整型（永远是正整数）变量来表示这个计数器。
	var ops uint64

	// WaitGroup 帮助我们等待所有协程完成它们的工作。
	var wg sync.WaitGroup

	// 我们会启动 50 个协程，并且每个协程会将计数器递增 1000 次。
	for i := 0; i < 50; i++ {
		wg.Add(1)

		go func() {
			for c := 0; c < 1000; c++ {
				// 使用 `AddUint64` 来让计数器自动增加，
				// 使用 `&` 语法给定 `ops` 的内存地址。
				atomic.AddUint64(&ops, 1)
			}
			wg.Done()
		}()
	}

	// 等待，直到所有协程完成。
	wg.Wait()

	// 现在可以安全的访问 `ops`，因为我们知道，此时没有协程写入 `ops`，
	// 此外，还可以使用 `atomic.LoadUint64` 之类的函数，在原子更新的同时安全地读取它们。
	fmt.Println("ops:", ops)
}
```

## 🔍 师兄给你逐行拆
**第一段：var ops uint64 声明与零值可用**

Go 里声明 `var ops uint64` 会自动初始化为 0，不需要手动赋值。更重要的是，`uint64` 本身在内存里是一块连续的 8 字节，只要地址对齐，硬件就能用原子指令操作它。这就是「零值可用」的精髓——声明即可用，省掉一切初始化仪式。选用无符号整型是因为计数器不会出现负数，`uint64` 最大能表示约 1.8×10¹⁹，对绝大多数计数场景绰绰有余。

**第二段：为什么不能直接写 ops++**

很多人觉得「加一」是一步操作，其实 CPU 层面是三步：① 从内存读出当前值到寄存器，② 在寄存器里加一，③ 把新值写回内存。这三步合称 read-modify-write，在单线程下没有问题，但多个 goroutine 并发执行时，两个 goroutine 可能在「①读」之后、「③写」之前同时持有同一个旧值，最终都只写回「旧值+1」，而不是「旧值+2」。这就是所谓的「丢更新（lost update）」，也是数据竞争（data race）的典型形态。用 `go run -race` 跑含有裸 `ops++` 的并发代码，race detector 立刻就会报警。50 个 goroutine 各跑 1000 次，理论上应得 50000，实际可能只有 48000 甚至更低，而且每次运行结果都不同——这类 bug 在生产环境里极难复现，是最危险的一类并发问题。

**第三段：atomic.AddUint64(&ops, 1) 的底层原理**

`atomic.AddUint64(&ops, 1)` 在 x86-64 上会编译成带 `LOCK` 前缀的 `XADDQ` 或 `LOCK ADDQ` 指令。`LOCK` 前缀告诉 CPU：在执行这条指令的完整周期内，锁住内存总线（现代 CPU 实际上锁的是缓存行），其他核心不能同时读写这块内存。这样三步 read-modify-write 就被压缩成了硬件层面不可分割的一个动作，任何其他 goroutine 看到的要么是加之前的旧值，要么是加之后的新值，永远不会看到中间状态。这就是「原子」的本义：不可再分。对比互斥锁（`sync.Mutex`），`Lock/Unlock` 需要操作系统级别的上下文切换开销，而原子指令完全在用户态完成，性能快一个数量级，非常适合高频计数这类简单操作。

**第四段：必须传指针 &ops**

`atomic.AddUint64` 的签名是 `func AddUint64(addr *uint64, delta uint64) (new uint64)`，第一个参数必须是指针。原因很直接：Go 是值传递，如果传 `ops` 本身，函数拿到的是一份拷贝，在拷贝上加一对原变量毫无影响。传 `&ops` 才能拿到原变量的内存地址，原子指令也是对这个地址操作的。这里还有一个细节：`atomic` 包要求被操作的变量在内存里是 64 位对齐的（地址能被 8 整除），在 64 位系统上声明为包级或函数级变量天然满足，但如果是结构体字段，需要保证结构体本身对齐，否则在 32 位平台上会 panic。这是 `atomic` 使用中不算罕见的陷阱，Go 文档有明确说明。

**第五段：WaitGroup 组织 50 个 goroutine**

代码用 `sync.WaitGroup` 来等待所有 goroutine 完成。先 `wg.Add(50)` 告知组里有 50 个任务，然后 for 循环启动 50 个 goroutine，每个 goroutine 体内循环 1000 次调用 `atomic.AddUint64(&ops, 1)`，结束后调用 `wg.Done()`（即 `wg.Add(-1)`）。主 goroutine 在 `wg.Wait()` 处阻塞，直到计数归零才继续往下走。WaitGroup 保证了一个重要的 happen-before 关系：`wg.Wait()` 返回之后，所有 goroutine 的所有写操作对主 goroutine 都可见。也就是说，在 `wg.Wait()` 之后读取 `ops`，一定能看到完整的 50000，不会因为 CPU 缓存没刷新而看到旧值。

**第六段：最终结果必须等于 50000**

50 个 goroutine × 每个 1000 次 = 50000 次原子加一，由于每次加一都是硬件原子操作，不存在丢更新，最终 `ops` 一定等于 50000。这是这道题想验证的核心结论。你可以把 `atomic.AddUint64` 换成裸 `ops++`，再用 `-race` 跑一遍，就能亲眼看到两种截然不同的行为。

**第七段：atomic.LoadUint64 用于运行时读取**

如果你想在所有 goroutine 还在运行时就「偷看」当前计数，不能直接写 `fmt.Println(ops)`，因为那是一次普通的内存读，可能读到撕裂值（torn read）或缓存中的旧值。应该用 `atomic.LoadUint64(&ops)`，它同样是原子指令，保证读到一个完整的、最新的值。在本题里，`wg.Wait()` 已经建立了足够的同步屏障，所以最后打印直接读 `ops` 其实也安全，但习惯上仍然推荐用 `atomic.LoadUint64`，意图更清晰，也避免 race detector 误报。

**第八段：Go 1.19+ 的 atomic.Uint64 类型**

Go 1.19 引入了 `atomic.Bool`、`atomic.Int32`、`atomic.Int64`、`atomic.Uint32`、`atomic.Uint64` 等封装类型。用法是 `var ops atomic.Uint64`，然后 `ops.Add(1)`、`ops.Load()`，不再需要手动取地址，也不用记 `*uint64` 指针传递规则，编译器能在类型层面拦截误用。新类型还刻意把 `noCopy` 嵌进去，防止被意外值拷贝——直接赋值给另一个变量会让 `go vet` 报错，比裸 `uint64` 更安全。如果你的项目 Go 版本 ≥ 1.19，优先选这套新 API，代码可读性和安全性都更好；旧版本项目则继续用 `atomic.AddUint64` 那套函数式 API 即可。

## 🏃 跑一下试试
```bash
# 保存为 atomic_counter.go 后执行
go run atomic_counter.go
# 预期输出：
# ops: 50000

# 同时建议跑竞态检测，应该看到 0 个 data race
go run -race atomic_counter.go
# 预期输出：
# ops: 50000
# (无 WARNING: DATA RACE 字样)
```

## 💡 师兄的碎碎念
💡 go run -race 是你的好朋友：它会在运行时插桩，一旦有两个 goroutine 裸读写同一内存就立刻报警，开发阶段养成习惯，比事后排查省十倍力气。

💡 atomic 只适合「单个整型或指针」的简单操作；如果你需要同时更新 count 和 lastUpdated 两个字段，哪怕各自都是原子的，组合在一起也不原子——这种情况必须用 sync.Mutex 把整个临界区包起来。

💡 Go 1.19+ 新增了类型安全的 atomic.Int64 / atomic.Uint64，不用再传指针、不容易写错，新项目优先用这套 API，比 atomic.AddInt64(&counter, 1) 更清晰。

💡 原子操作比 Mutex 快，但不是银弹：高竞争场景下 Mutex + 单次批量更新 反而可能更快；遇到性能瓶颈先 benchmark，别过早优化。

## 🎓 这一关的知识点清单
✅ 用 sync/atomic.AddInt64 或 atomic.Uint64 保护计数器，而不是裸写 counter++
✅ 所有 goroutine 启动后必须 wg.Wait()，否则主进程退出时数据还没算完
✅ 打印前不需要额外加锁，直接 atomic.LoadInt64 读最终值即可
✅ 跑一遍 go run -race 确认零 data race 警告
✅ 理解 atomic 操作的局限：只适合单个整型/指针，多字段联动就得换 Mutex

## ➡️ 下一关
atomic 搞定了单个计数器，但现实中经常要同时保护好几个字段、或者先读再写的复合操作——这时候 atomic 就力不从心了。第 41 关「互斥锁 Mutexes」就是专门解决这个问题的：用 sync.Mutex 把整段临界区锁住，让复杂的共享状态也能安全更新。我们下一关见。

👉 [第 41 关：互斥锁](../41-mutexes/)
