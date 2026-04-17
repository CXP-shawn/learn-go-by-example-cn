# 第 30 关：通道选择器（select）（师兄带你学 Go）

## 🎯 这一关你会学到
这关咱们学会用 select 同时盯着多个通道，哪个通道先来数据就先处理哪个，不用自己写一堆 if-else 去轮询。说白了就是 Go 给你内置的「多路等待」神器，配合 goroutine 用起来贼顺手。

## 🤔 先想一个问题
兄弟，你有没有遇到过这种情况——同时点了美团和饿了么两家外卖，不知道哪家先到，只能坐那儿干等。你总不能专门盯着美团 APP 等它送到，再去看饿了么吧？正常人的做法是：哪家骑手先打电话来，就先去拿哪家的。Go 的 select 干的就是这件事——同时耳朵竖着听多个通道，谁先来消息就先处理谁，剩下的继续等。

## 📖 看代码
```go
// Go 的 _选择器（select）_  让你可以同时等待多个通道操作。
// 将协程、通道和选择器结合，是 Go 的一个强大特性。

package main

import (
	"fmt"
	"time"
)

func main() {

	// 在这个例子中，我们将从两个通道中选择。
	c1 := make(chan string)
	c2 := make(chan string)

	// 各个通道将在一定时间后接收一个值，
	// 通过这种方式来模拟并行的协程执行（例如，RPC 操作）时造成的阻塞（耗时）。
	go func() {
		time.Sleep(1 * time.Second)
		c1 <- "one"
	}()
	go func() {
		time.Sleep(2 * time.Second)
		c2 <- "two"
	}()

	// 我们使用 `select` 关键字来同时等待这两个值，
	// 并打印各自接收到的值。
	for i := 0; i < 2; i++ {
		select {
		case msg1 := <-c1:
			fmt.Println("received", msg1)
		case msg2 := <-c2:
			fmt.Println("received", msg2)
		}
	}
}
```

## 🔍 师兄给你逐行拆
### 先把两个通道建出来

```go
c1 := make(chan string)
c2 := make(chan string)
```

这两行没啥花头，就是用 `make` 建了两个**无缓冲通道**（unbuffered channel，意思是发送方和接收方必须同时准备好，才能完成一次传值，不能提前把数据塞进去就走人）。类型是 `chan string`，只能传字符串。两个通道分别叫 `c1` 和 `c2`，后面分别代表两个「外卖骑手」。

师兄提醒一句：无缓冲通道发送数据会**阻塞**发送方，直到有人来取。这个特性在下面的例子里很关键。

### 两个 goroutine 模拟耗时操作

```go
go func() {
    time.Sleep(1 * time.Second)
    c1 <- "one"
}()
go func() {
    time.Sleep(2 * time.Second)
    c2 <- "two"
}()
```

这里开了两个**协程**（goroutine，Go 里轻量级的并发执行单元，你可以理解成两个同时跑的小任务，互不干扰）。每个协程里先 `time.Sleep` 睡一觉，模拟真实场景里那种「要等一段时间才有结果」的操作，比如网络请求、数据库查询、RPC 调用啥的。

第一个协程睡 1 秒，然后往 `c1` 发 `"one"`；第二个协程睡 2 秒，往 `c2` 发 `"two"`。两个协程是**同时跑**的，不是先跑完第一个再跑第二个——这点很重要，不然就没有「同时等」这个意义了。

吐槽一下：有些同学看到这里会问「`time.Sleep` 是不是很low」——对，实际项目里你换成真正的 HTTP 请求或者数据库查询就行，`Sleep` 只是为了让例子简单好懂。

### select 登场，核心来了

```go
for i := 0; i < 2; i++ {
    select {
    case msg1 := <-c1:
        fmt.Println("received", msg1)
    case msg2 := <-c2:
        fmt.Println("received", msg2)
    }
}
```

咱们一层一层拆。

**为啥要循环两次？**

因为咱们总共往通道里发了 2 个值——`c1` 发了一个，`c2` 发了一个。每次 `select` 只能接一个值，所以要循环 2 次才能把两个值都收完。如果你只写一次 `select`，第二个值就永远没人取了。记住：发了几个，就得收几个，否则 goroutine 会卡在那里发不出去，造成**goroutine 泄漏**（goroutine leak，goroutine 永远阻塞无法退出，白白占内存）。

**select 的语法啥意思？**

`select` 长得跟 `switch` 很像，但干的事完全不一样。`switch` 是根据某个值走哪个分支，`select` 是根据哪个通道**已经准备好**走哪个分支。`case msg1 := <-c1:` 这句话的意思是：「如果 `c1` 现在有数据可以读，就把数据赋给 `msg1`，然后执行这个 case 里的代码。」`case msg2 := <-c2:` 同理。

**谁先 ready 谁先被选**

`select` 会同时监听所有 case 里的通道操作。哪个通道先有数据到来（也就是先「就绪」），`select` 就执行哪个 case。在咱们这个例子里，`c1` 等 1 秒，`c2` 等 2 秒，所以第一次循环肯定是 `c1` 先就绪，打印 `received one`；第二次循环 `c2` 就绪，打印 `received two`。整个程序大概跑 2 秒出结果，而不是 1+2=3 秒，因为两个 goroutine 是并行的。

**select 的随机性**

这里有个坑师兄要提前说：如果同一时刻**多个** case 都就绪了，`select` 会**随机**选一个执行，不是固定选第一个。Go 故意这么设计的，避免某个通道被饿死（starvation，某个 case 永远得不到执行机会）。咱们这个例子因为两个通道就绪时间差了 1 秒，基本不会碰到随机性问题，但你在写真实代码的时候要记住这个特性，别假设 select 总是按顺序执行。

**不带 default 时 select 会阻塞**

现在这段代码的 `select` 没有 `default` 分支。没有 `default` 意味着：如果所有 case 的通道都没数据，`select` 就乖乖**阻塞**在那里等，直到有某个通道就绪为止。这正是咱们想要的行为——等着就行，别瞎转。

如果你加了 `default`，`select` 在没有通道就绪时就会立刻跑 `default` 分支，不会等待。这个用法适合「非阻塞尝试读取」的场景，但那是后面的事，这关先搞清楚阻塞版本。

**和 switch 的区别总结一下**

`switch` 是「看值走分支」，条件是普通表达式；`select` 是「看通道走分支」，条件必须是通道的发送或接收操作。`switch` 按顺序匹配，找到第一个满足的就走；`select` 同时监听所有 case，多个就绪时随机选。两个用途完全不同，别搞混了。

### 程序跑起来的时间线

简单梳理一下整个执行流程：主 goroutine 建完通道、启动两个子 goroutine 之后，进入 for 循环，第一次 `select` 阻塞等待。大约 1 秒后，第一个子 goroutine 睡醒，往 `c1` 发 `"one"`，`select` 的 `case msg1 := <-c1` 就绪，打印 `received one`，第一次循环结束。进入第二次循环，`select` 继续阻塞，又过了大约 1 秒（总共约 2 秒），第二个子 goroutine 睡醒，往 `c2` 发 `"two"`，打印 `received two`，循环结束，程序退出。整个程序耗时约 2 秒，不是 3 秒，这就是并发的好处。

## 🏃 跑一下试试
```bash
go run select.go

# 预期输出（约 2 秒后全部打印完）：
# received one
# received two
```

注意：c1 睡 1 秒先到，c2 睡 2 秒后到，但两个 sleep 是并行跑的，所以总耗时≈2 秒，不是 3 秒——这就是并发的好处，师兄再强调一遍！

## 💡 师兄的碎碎念
⚡ 多个 case 同时 ready，select 随机挑一个执行——这不是 bug，是故意设计的，保证公平性，别想着依赖顺序，兄弟，会翻车的

🚫 channel 被关闭之后，对应的 case 会立刻拿到零值（比如 string 就是 ""，int 就是 0），而且每次 select 都会命中它，一不小心就死循环，记得用 `v, ok := <-ch` 判断一下 ok

😶 `select {}` 空写法会让当前 goroutine 永远阻塞，main 函数里这么写程序就挂在那了，守护进程、单元测试里偶尔用到，见到别懵

🎯 default 分支让 select 变「探一下就走」的非阻塞模式，想做「有消息就处理、没消息继续干别的」的逻辑，default 是你的好兄弟

## 🎓 这一关的知识点清单
• select 语句会同时监听多个 channel，哪个 case 的通道先 ready，就执行哪个，其余继续等
• 两个 goroutine 并行 sleep，所以总耗时约 2 秒，而不是 1+2=3 秒，这是并发的精髓
• 不带 default 的 select 会一直阻塞，直到至少一个 case 就绪——这是「等待多路信号」的标准姿势
• 带上 default 分支后，select 立刻变成非阻塞：没有 case 就绪就直接走 default，不等了
• 空的 select{} 会让当前 goroutine 永久挂起，常见于主函数守护或测试场景

## ➡️ 下一关
第 31 关「超时处理 timeouts」来了——select 配上 time.After 就能给任意操作加超时，下一关咱们见！

👉 [第 31 关：超时处理](../31-timeouts/)
