# 第 41 关：互斥锁（mutexes）（师兄带你学 Go）

## 🎯 这一关你会学到
sync.Mutex 是保护复杂临界区（比如 map）最直接的工具，同一时刻只允许一个 goroutine 持有锁并操作共享数据。map 在 Go 中并发读写会触发 fatal error，必须用锁包住所有访问路径。掌握这一关，才能放心地在多 goroutine 之间共享任何可变状态。

## 🤔 先想一个问题
想象办公楼里只有一个单人卫生间：门上有一把插销，进去的人把插销一拨，外面的人推门推不开，只能在走廊排队等。里面的人出来时拨开插销，下一个人才能进去。sync.Mutex 就是这把插销——Lock() 相当于进门拨上插销，Unlock() 相当于出门拨开插销。goroutine 们就是走廊里排队的同事，谁也别想在别人还没出来的时候硬闯进去，否则 map 就乱套了，就像两个人同时挤进单人卫生间一样荒唐。

## 📖 看代码
```go
// 在前面的例子中，我们看到了如何使用原子操作(atomic-counters)来管理简单的计数器。
// 对于更加复杂的情况，我们可以使用一个_[互斥量](http://zh.wikipedia.org/wiki/%E4%BA%92%E6%96%A5%E9%94%81)_
// 来在 Go 协程间安全的访问数据。

package main

import (
	"fmt"
	"sync"
)

// Container 中定义了 counters 的 map ，由于我们希望从多个 goroutine 同时更新它，
// 因此我们添加了一个 互斥锁Mutex 来同步访问。
// 请注意不能复制互斥锁，如果需要传递这个 `struct`，应使用指针完成。
type Container struct {
	mu       sync.Mutex
	counters map[string]int
}

func (c *Container) inc(name string) {
	// 在访问 `counters` 之前锁定互斥锁；
	// 使用 [defer]（defer） 在函数结束时解锁。
	c.mu.Lock()
	defer c.mu.Unlock()
	c.counters[name]++
}

func main() {
	c := Container{
		// 请注意，互斥量的零值是可用的，因此这里不需要初始化。
		counters: map[string]int{"a": 0, "b": 0},
	}

	var wg sync.WaitGroup

	// 这个函数在循环中递增对 name 的计数
	doIncrement := func(name string, n int) {
		for i := 0; i < n; i++ {
			c.inc(name)
		}
		wg.Done()
	}

	// 同时运行多个 goroutines;
	// 请注意，它们都访问相同的 `Container`，其中两个访问相同的计数器。
	wg.Add(3)
	go doIncrement("a", 10000)
	go doIncrement("a", 10000)
	go doIncrement("b", 10000)

	// 等待上面的 goroutines 都执行结束
	wg.Wait()
	fmt.Println(c.counters)
}
```

## 🔍 师兄给你逐行拆
### 先说为什么 map 要用 Mutex

师兄先把问题摆清楚。Go 的 map 底层是哈希桶 + 溢出链 + 动态扩容，整个数据结构在写入时要改好几处内存，根本不是一条原子指令能覆盖的。所以 Go 官方直接放话：**map 不是并发安全的**，两个 goroutine 同时往一个 map 写，运行时会直接 panic 抛 `concurrent map writes` 给你看，不是静默数据错乱，是当场崩。

有同学刚学完上一关 `sync/atomic`，可能会想：原子操作能不能救场？不行。`atomic` 只能保护单个整型或者指针，做的是硬件级别的原子加减、比较交换这种简单动作。map 的 `counters[name]++` 一步至少涉及：算 hash → 找桶 → 读旧值 → 加 1 → 写新值，还可能触发扩容搬迁，`atomic` 根本管不过来。

复合结构的临界区要用 `sync.Mutex` 把整段代码块圈起来，保证同一时刻只有一个 goroutine 进去改，改完再让下一个进。

### Container 结构：锁和被保护的字段放在一起

看代码的开头：

```go
type Container struct {
    mu       sync.Mutex
    counters map[string]int
}
```

`sync.Mutex` 直接作为 Container 的一个字段，和它要保护的 `counters` 挨在一起。这是 Go 里非常惯用的写法——锁和数据捆绑，一看结构体就知道"这把锁保护的是哪个字段"，不会写着写着不知道该锁谁。

`sync.Mutex` 是**零值可用**的类型，声明出来就能用，不需要 `new` 或 `make`。所以 `main` 里这样写就行：

```go
c := Container{
    counters: map[string]int{"a": 0, "b": 0},
}
```

`mu` 字段故意不写，让它用零值，干净利落。

### Lock + defer Unlock 的标准姿势

再看 `inc` 方法：

```go
func (c *Container) inc(name string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.counters[name]++
}
```

三行代码，每一行都有讲究。

第一行 `c.mu.Lock()`：尝试拿到锁。如果锁已经被别人拿走，当前 goroutine 就在这里阻塞，什么都不干，等锁被释放再继续。

第二行 `defer c.mu.Unlock()`：注册一个"函数退出时解锁"的钩子。这是 Go 里处理锁的铁律——Lock 下一行立刻写 defer Unlock，两行之间不插别的逻辑。为什么？因为 defer 保证不管你是正常 return 还是中途 panic，Unlock 都会被执行。要是你手写 `c.mu.Unlock()` 放在函数末尾，中间某行 panic 了，锁就永远丢了，其他所有等这把锁的 goroutine 全部死锁，运行时会报 `all goroutines are asleep - deadlock!` 然后整个程序挂掉。

第三行 `c.counters[name]++` 才是真正的业务逻辑，这时候锁已经拿到，可以安全地写 map。

### 为什么必须用指针接收器

注意 `inc` 的接收器写的是 `c *Container`，不是 `c Container`。这是硬性要求，不能偷懒写值接收器。

原因：值接收器每次调用方法都会**复制**整个 Container 结构体，包括里面的 `mu` 字段。复制出来的 `mu` 和原始的 `mu` 就是两把完全独立的锁了，你在副本上 Lock，原始 Container 还是开着的，相当于没锁。

`sync.Mutex` 自己也有规定：**Once used, a Mutex must not be copied**（一旦被使用过就不能被复制）。用 `go vet` 扫代码，只要检测到 Mutex 值传递或结构体值拷贝，直接报警 `passes lock by value`。记住：任何内嵌 `sync.Mutex` 的 struct，方法都要用指针接收器，结构体也要用指针传递。

### main 里的并发场景和预期输出

最后看 main 里的逻辑：

```go
wg.Add(3)
go doIncrement("a", 10000)
go doIncrement("a", 10000)
go doIncrement("b", 10000)
wg.Wait()
fmt.Println(c.counters)
```

启动 3 个 goroutine：两个都往 "a" 这个 key 各累加 10000 次，一个往 "b" 累加 10000 次。每次 `c.inc(name)` 都要 Lock → ++ → Unlock，所以哪怕两个 goroutine 抢着写 "a"，也会被互斥锁排成队一个一个来，不会相互踩踏。

WaitGroup 保证主 goroutine 等三个子 goroutine 都完成再继续。最后打印结果稳定是：

```
map[a:20000 b:10000]
```

**a 恰好是 20000、b 恰好是 10000，一个都不少**。把 `c.mu.Lock()` 和 `defer c.mu.Unlock()` 这两行注释掉再跑，大概率直接 panic 出 `concurrent map writes`——这就是锁存在的价值。

## 🏃 跑一下试试
```bash
go run mutexes.go
```
预期输出：
map[a:20000 b:10000]

## 💡 师兄的碎碎念
① 养成 Lock() 之后立即 defer Unlock() 的习惯，哪怕函数只有三行，defer 能保证 panic 时锁也会被释放，不会把后续 goroutine 全部卡死。
② 绝对不要把 sync.Mutex 按值传递或赋值，go vet 会报 "passes lock by value" 错误；结构体含锁时，方法接收者要用指针 *Container 而非 Container。
③ 如果场景是「读多写少」（比如配置缓存），可以换用 sync.RWMutex：RLock/RUnlock 允许多个读者同时进入，只有写操作才独占，吞吐量更高。
④ 避免嵌套加锁（持有锁 A 的情况下再去抢锁 B），两个 goroutine 互相等对方释放就会永久死锁；如果业务确实复杂，可以考虑 Go 1.9+ 提供的 sync.Map，它内部已做好分段锁优化，适合键集合相对稳定、并发读写均衡的场景。

## 🎓 这一关的知识点清单
☐ Container 结构体嵌入 sync.Mutex 字段
☐ 所有对 counters map 的读写都在 Lock/Unlock 之间
☐ 启动 3 个 goroutine 分别调用 inc，WaitGroup 等待全部完成
☐ 最终打印 map 确认 a:20000 b:10000
☐ go run -race 运行无数据竞争警告

## ➡️ 下一关
第 42 关「状态协程 stateful-goroutines」会把这把插销彻底藏起来，用一个专职的 goroutine 独占状态、通过 channel 收发指令来替代显式加锁，让并发设计更符合 Go 的哲学。

👉 [第 42 关：状态协程](../42-stateful-goroutines/)
