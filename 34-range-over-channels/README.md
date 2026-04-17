# 第 34 关：通道遍历（range-over-channels）（师兄带你学 Go）

## 🎯 这一关你会学到
Go 允许你直接用 for range 遍历一个通道，每次循环自动取出一个值，直到发送方调用 close 关闭通道为止，循环才会正常退出。这比手动用 v, more := <-ch 再判断 more 要干净得多，是处理「有限数量任务流」时最常见的写法。

## 🤔 先想一个问题
你去快递驿站取包裹，工作人员一件一件从货架上递给你，递完最后一件说一声「没了」，你就知道可以走了。这个「没了」就是 close——如果工作人员一直不说结束，你就得傻站在那里等，永远不敢离开。range-over-channel 做的事情和这一模一样：它帮你省掉了手动问「还有没有？」的那句话。

## 📖 看代码
```go
// 在[前面](range)的例子中，
// 我们讲过 `for` 和 `range` 为基本的数据结构提供了迭代的功能。
// 我们也可以使用这个语法来遍历的从通道中取值。

package main

import "fmt"

func main() {

	// 我们将遍历在 `queue` 通道中的两个值。
	queue := make(chan string, 2)
	queue <- "one"
	queue <- "two"
	close(queue)

	// `range` 迭代从 `queue` 中得到每个值。
	// 因为我们在前面 `close` 了这个通道，所以，这个迭代会在接收完 2 个值之后结束。
	for elem := range queue {
		fmt.Println(elem)
	}
}
```

## 🔍 师兄给你逐行拆
先来看整体结构，这段代码做了三件事：创建一个带缓冲的通道 queue，往里塞两个字符串，关闭它，然后用 for range 把里面的值全部取出来打印。流程极短，但每一行都有值得说道的地方。

**make(chan string, 2) 带缓冲 2 是什么意思**

普通的无缓冲通道（make(chan string)）要求发送方和接收方同时就位，缺一个就阻塞。带缓冲通道则在内部维护一个队列，容量是第二个参数。make(chan string, 2) 表示这个通道最多可以暂存 2 条消息，发送方塞进去就走人，不需要等接收方。如果缓冲区满了，发送方才会阻塞；如果缓冲区空了，接收方才会阻塞。在这个例子里，main 函数先把两条消息都推进去，再关闭通道，之后才开始 range 遍历——因为缓冲区容量刚好是 2，所以两次 queue <- 都能立刻完成，不会卡住。如果缓冲区只有 1，第二次 queue <- "two" 就会阻塞，因为第一条还没人取走，而接收的 range 又还没开始跑，结果就是死锁。

**先发两次 queue <- 再 close**

close(queue) 的语义是：我作为发送方声明「这个通道以后不会再有新数据了」。它不会清空缓冲区里已有的数据，只是插入一个「结束信号」。接收方依然可以把缓冲区里剩余的值一条条取干净，取完之后再收到的就是零值加上 more == false 的信号。所以这里的顺序很重要：先塞数据，再 close，两步缺一不可。如果你先 close 再想往里发，Go 运行时会直接 panic：send on closed channel。另外，close 只能由发送方调用，让接收方来 close 是常见的错误，同样会 panic。

**for elem := range queue 语法**

这是本关的主角。range 作用在通道上时，每次迭代都会阻塞等待，直到通道里有新值可以取，然后把值赋给 elem 执行循环体。当通道被关闭且缓冲区已经清空，range 自动检测到这个信号，正常退出循环，不需要你写任何 break 或 if。这一行抵得上第 33 关里一整个 for { j, more := <-jobs; if more { ... } else { ...; return } } 的结构。

**range 自动检测 close 信号退出循环**

底层机制是：每次从通道接收时，Go 运行时返回两个值——实际数据和一个布尔值 ok，ok == false 代表通道已关闭且无剩余数据。for range 帮你把这个判断封装掉了，循环体里你看不到 ok，但它一直在后台被检查。只要 ok 变成 false，循环立刻退出。这和 range 作用在 slice 或 map 上的行为风格保持一致：你只管用值，边界条件由语言帮你处理。

**和第 33 关 v, more := <-ch 的对比**

第 33 关的写法是这样的：

```go
for {
    j, more := <-jobs
    if more {
        fmt.Println("received job", j)
    } else {
        fmt.Println("received all jobs")
        done <- true
        return
    }
}
```

这种写法的优点是灵活——你可以在 more == false 的分支里做额外的清理工作，比如通知其他协程、记录日志等。但如果你不需要在「通道关闭」这个事件上附加任何逻辑，这段代码就显得啰嗦。for range 版本变成：

```go
for elem := range queue {
    fmt.Println(elem)
}
```

两行，意图一目了然。代码越短，出错的地方就越少。实际项目里，能用 range 的场景尽量用 range，把 v, more := <-ch 留给真正需要区分「正常值」和「关闭事件」的场合。

**如果不 close 会怎样——永远阻塞，死锁**

这是初学者最容易踩的坑。假设你把 close(queue) 这行删掉，range 会在取完两条消息之后继续阻塞等待，因为它不知道你是真的没数据了还是数据还在路上。如果程序里没有其他协程会往 queue 里发数据，Go 运行时会检测到所有协程都卡住了，抛出 fatal error: all goroutines are asleep - deadlock!。如果是在一个单独的 goroutine 里跑 range 而 main 继续执行，那个 goroutine 就会泄漏——它永远挂在那里，占着内存，直到进程退出。所以规则很简单：谁负责往通道发数据，谁负责在发完之后 close 它；接收方只管 range，不管 close。

**什么时候适合用带缓冲通道 + range**

带缓冲通道 + range 这个组合最适合的场景是：发送方的生产速度和接收方的消费速度不一定匹配，但任务总量是有限的。典型例子：批量处理文件，主协程把文件路径一条条推进带缓冲通道，工作协程用 range 依次处理，主协程推完之后 close。缓冲区的作用是平滑两端速度差，避免发送方因接收方处理慢而频繁阻塞。缓冲区大小通常根据「你愿意在内存里预存多少条待处理任务」来定，没有固定公式，但设成 0（无缓冲）也完全合法，只是发送方每推一条就要等接收方取走才能继续。无论缓冲区大小，range 的写法不变，关闭语义不变，这正是这个模式的优雅之处。

## 🏃 跑一下试试
```bash
cat << 'EOF' > channel_range.go
package main

import "fmt"

func main() {
    queue := make(chan string, 2)
    queue <- "one"
    queue <- "two"
    close(queue)

    for elem := range queue {
        fmt.Println(elem)
    }
}
EOF
go run channel_range.go
```
预期输出：
one
two

## 💡 师兄的碎碎念
- 忘记 close 是新手最常踩的坑：for range 会一直等新消息，程序直接死锁，runtime 报 deadlock，记住「发完就 close」
- range channel 只有一个循环变量，是消息本身，不像 slice 的 range 能拿到 index，多写一个变量编译直接报错
- 这是生产者-消费者模型的标准写法：生产者往 channel 塞数据、close，消费者 for range 消费，干净利落
- 想中途退出用 break 就够了，千万别让接收方去 close channel——那是反模式，发送方往已关闭的 channel 发数据会直接 panic

## 🎓 这一关的知识点清单
- [ ] 发送完两条消息后有没有调用 close(queue)，没有的话 for range 会永远堵着
- [ ] for range 只写了一个变量（消息本身），没有多写 index 导致编译报错
- [ ] goroutine 里发送时记得确认 channel 有足够缓冲，否则还没 range 就堵死了
- [ ] 程序输出顺序是 one 在前、two 在后，符合 FIFO 特性

## ➡️ 下一关
第 35 关咱们聊「Timer」——time 包里专门做单次定时的小工具，到点触发、可以手动取消，比 time.Sleep 灵活多了，跟着我走一遍就明白它和 ticker 的区别在哪。

👉 [第 35 关：Timer](../35-timers/)
