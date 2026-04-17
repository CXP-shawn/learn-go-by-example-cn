# 第 32 关：非阻塞通道操作（non-blocking-channel-operations）（师兄带你学 Go）

## 🎯 这一关你会学到
select 加一个 default 分支，就能把原本会死等的通道操作变成「有就拿、没有就走」。没有 default，select 会一直挂着等某个 case 就绪；加了 default，只要当前没有任何 case 能立即执行，就直接走 default，整个操作瞬间返回，不阻塞调用方。

## 🤔 先想一个问题
你半夜饿了，打开冰箱看看有没有剩菜——有就拿出来热，没有就关门去泡面，不会站在冰箱前傻等剩菜凭空变出来。通道的非阻塞操作就是这个逻辑：去看一眼，有数据就取走，没有就走 default 这条路，绝不站在那里干等。

## 📖 看代码
```go
// 常规的通过通道发送和接收数据是阻塞的。
// 然而，我们可以使用带一个 `default` 子句的 `select`
// 来实现 _非阻塞_ 的发送、接收，甚至是非阻塞的多路 `select`。

package main

import "fmt"

func main() {
	messages := make(chan string)
	signals := make(chan bool)

	// 这是一个非阻塞接收的例子。
	// 如果在 `messages` 中存在，然后 `select` 将这个值带入 `<-messages` `case` 中。
	// 否则，就直接到 `default` 分支中。
	select {
	case msg := <-messages:
		fmt.Println("received message", msg)
	default:
		fmt.Println("no message received")
	}

	// 一个非阻塞发送的例子，代码结构和上面接收的类似。
	// `msg` 不能被发送到 `message` 通道，因为这是
	// 个无缓冲区通道，并且也没有接收者，因此， `default`
	// 会执行。
	msg := "hi"
	select {
	case messages <- msg:
		fmt.Println("sent message", msg)
	default:
		fmt.Println("no message sent")
	}

	// 我们可以在 `default` 前使用多个 `case` 子句来实现一个多路的非阻塞的选择器。
	// 这里我们试图在 `messages` 和 `signals` 上同时使用非阻塞的接收操作。
	select {
	case msg := <-messages:
		fmt.Println("received message", msg)
	case sig := <-signals:
		fmt.Println("received signal", sig)
	default:
		fmt.Println("no activity")
	}
}
```

## 🔍 师兄给你逐行拆
### 先回顾一下：常规通道操作为什么会阻塞？

兄弟，咱们先把基础捋清楚。Go 里面，无论是往通道发数据还是从通道收数据，默认都是阻塞的。比如你写 `v := <-messages`，如果 messages 里面没有数据，这行代码就会把当前 goroutine 挂起来，一直等到有人往 messages 发东西为止。同理，`messages <- msg` 如果没有接收方，也会挂住，等到有人来收。

这种机制在多 goroutine 协作的场景下很有用，能做到精准同步。但有时候你并不想等，你只是想「看一眼，有就拿，没有就算」，这时候阻塞就成了麻烦。select + default 就是为这种需求设计的。

### select 加了 default 之后发生了什么？

普通的 select 是这样的：它会检查每个 case，如果有多个 case 同时就绪就随机选一个，如果一个都没就绪就一直阻塞等待。而一旦你在 select 里加了 default 分支，规则就变了：**如果扫一遍所有 case，没有任何一个能立即执行，就立刻走 default，整个 select 直接返回，不等**。这就是非阻塞的核心语义——default 就是「没有就绪就走这里」的出口。

好，下面咱们把代码里的三段 select 一段一段拆开看。

---

### 第一段：非阻塞接收

```go
select {
case msg := <-messages:
    fmt.Println("received message", msg)
default:
    fmt.Println("no message received")
}
```

此时 messages 是一个刚创建的无缓冲通道，没有任何 goroutine 往里面发数据，所以 `<-messages` 这个 case 根本没办法立即就绪——没数据可收。

select 扫了一圈，发现唯一的 case 没法执行，立刻走 default，打印 `no message received`，然后整个 select 结束。整个过程没有任何等待，就是去看了一眼冰箱，空的，关门走人。

如果你去掉 default，这里就会永久阻塞，因为没有任何 goroutine 会来发数据，程序会死锁，Go runtime 最终会报 `all goroutines are asleep - deadlock!`。

所以 default 的意义就在这里：它给了 select 一条「没有就绪时的退路」，让操作可以立即返回。

---

### 第二段：非阻塞发送

```go
msg := "hi"
select {
case messages <- msg:
    fmt.Println("sent message", msg)
default:
    fmt.Println("no message sent")
}
```

这次是发送方向。咱们有一个字符串 `"hi"`，想把它发到 messages 通道里。但是 messages 是无缓冲通道，无缓冲的意思是：发送方和接收方必须同时在场，数据才能传递过去，通道本身不存数据。

现在没有任何 goroutine 在等着从 messages 里收数据，所以 `messages <- msg` 这个 case 也没办法立即就绪，没有接收方接住这个 `"hi"`，发送操作无法完成。

select 扫一遍，case 不就绪，走 default，打印 `no message sent`，直接结束。msg 这个数据就这么被丢弃了，没发出去。

你可能会问：如果是有缓冲的通道，缓冲区没满，这个 case 能立即就绪吗？能。有缓冲通道只要缓冲区没满，发送操作就是立即完成的，不需要等接收方，所以 case 就绪，会走 `sent message` 那个分支而不是 default。这个区别师兄特别提一下，面试常考。

---

### 第三段：多路非阻塞 select

```go
select {
case msg := <-messages:
    fmt.Println("received message", msg)
case sig := <-signals:
    fmt.Println("received signal", sig)
default:
    fmt.Println("no activity")
}
```

这里咱们把两个通道 messages 和 signals 都放进来，同时做非阻塞的接收尝试。select 会同时检查两个 case：

- `<-messages`：messages 里有没有数据？没有，不就绪。
- `<-signals`：signals 里有没有数据？也没有，也不就绪。

两个 case 都没有就绪，走 default，打印 `no activity`，结束。

这段代码展示了 default 在多路场景下的威力：你可以同时监听很多个通道，只要其中有一个有数据，就走对应的 case；如果一个都没有，走 default。整个过程依然是瞬间完成，不等任何人。

这个模式在实际项目里非常常见，比如你有一个主循环，每次循环里想看看有没有新消息、有没有新信号、有没有退出指令，但不想在这里卡住，就可以用这种多路非阻塞 select，快速轮询一圈，有就处理，没有就继续做别的事情。

---

### 总结一下 default 的语义

师兄帮你把核心语义浓缩成一句话：**default 就是「没有任何 case 在这一刻能立即执行」时的兜底出口，一毫秒都不等**。有了它，select 从「同步等待」变成了「瞬时检查」。去掉它，select 回到阻塞模式，会一直挂着直到某个 case 就绪。理解这一点，非阻塞通道操作这关就彻底拿下了。

## 🏃 跑一下试试
```bash
# 文件已经是 go-by-example 里的 non-blocking-channel-operations.go
go run non-blocking-channel-operations.go
```

预期输出：
```
no message received
no message sent
no activity
```

## 💡 师兄的碎碎念
- **best-effort 投递场景**：非阻塞发送/接收适合「发得出去最好，发不出去也无所谓」的场景，比如事件采样、心跳通知，别用在必须保证送达的关键路径上
- **小心错过真实信号**：default 让 select 立刻逃跑，如果信号就在「下一纳秒」才 ready，你已经走了——这不是 bug 是设计，但一定要清楚自己在做取舍
- **别在裸 for 循环里无脑用非阻塞 select**：`for { select { case ...: default: } }` 没有任何 sleep/yield，会把一个核跑满，CPU 100%，老老实实加 `time.Sleep` 或换成带超时的 `select`
- **和 `len(ch)` 对比着理解**：`len(ch)` 能看 buffered channel 当前有几条消息，但读完之前数量可能变——非阻塞 select 才是原子地「试一下」，两者不能互换

## 🎓 这一关的知识点清单
- [ ] 三段 select 都加了 `default`，程序不会在任何一段阻塞
- [ ] 第一段尝试从 `messages` 接收，没数据 → 走 default 打印 `no message received`
- [ ] 第二段尝试往 `messages` 发送，没接收方 → 走 default 打印 `no message sent`
- [ ] 第三段同时监听 `messages` 和 `signals`，两个都没就绪 → 走 default 打印 `no activity`
- [ ] 理解「非阻塞」本质：select 加 default 就是给 channel 操作加了一个「超时时间 = 0」的快速失败出口

## ➡️ 下一关
搞清楚非阻塞发送之后，下一关（第 33 关）讲 **closing channels**——发送方主动关闭 channel，告诉接收方「我没数据了，别等了」，是比 default 更正式的收尾手段。

👉 [第 33 关：通道的关闭](../33-closing-channels/)
