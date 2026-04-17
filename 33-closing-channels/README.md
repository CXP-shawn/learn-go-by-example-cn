# 第 33 关：通道的关闭（closing-channels）（师兄带你学 Go）

## 🎯 这一关你会学到
这一关教你用 close(ch) 告诉接收方「我不再发数据了」，配合 v, ok := <-ch 的双返回值写法，让接收方知道通道是否已关闭、数据是否已被完整消费完毕。这两个机制合在一起，是 Go 里生产者通知消费者「收工」的标准姿势。

## 🤔 先想一个问题
想象一家拉面馆，厨师每做好一碗就通过小窗口递给服务员，服务员站在那边一碗一碗往外端。到了打烊时间，厨师把小窗口的卷帘门「哐」地拉下来——这个动作本身就是一条信号：「今天的面全发完了，你别再等了。」服务员看到卷帘门落下，知道不会再有新面了，就可以安心去擦桌子、关灯、下班。Go 的 close(ch) 做的正是这件事：发送方把「卷帘门」拉下来，接收方看到门落了就知道可以收工了。

## 📖 看代码
```go
// _关闭_ 一个通道意味着不能再向这个通道发送值了。
// 该特性可以向通道的接收方传达工作已经完成的信息。

package main

import "fmt"

// 在这个例子中，我们将使用一个 `jobs` 通道，将工作内容，
// 从 `main()` 协程传递到一个工作协程中。
// 当我们没有更多的任务传递给工作协程时，我们将 `close` 这个 `jobs` 通道。
func main() {
	jobs := make(chan int, 5)
	done := make(chan bool)

	// 这是工作协程。使用 `j, more := <- jobs` 循环的从 `jobs` 接收数据。
	// 根据接收的第二个值，如果 `jobs` 已经关闭了，
	// 并且通道中所有的值都已经接收完毕，那么 `more` 的值将是 `false`。
	// 当我们完成所有的任务时，会使用这个特性通过 `done` 通道通知 main 协程。
	go func() {
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
	}()

	// 使用 `jobs` 发送 3 个任务到工作协程中，然后关闭 `jobs`。
	for j := 1; j <= 3; j++ {
		jobs <- j
		fmt.Println("sent job", j)
	}
	close(jobs)
	fmt.Println("sent all jobs")

	// 使用前面学到的[通道同步](channel-synchronization)方法等待任务结束。
	<-done
}
```

## 🔍 师兄给你逐行拆
我们来一段一段把这份代码拆透。

─────────────────────────────
第一段：make(chan int, 5) — 带缓冲通道
─────────────────────────────

```go
jobs := make(chan int, 5)
done := make(chan bool)
```

jobs 是一个容量为 5 的带缓冲通道。带缓冲和不带缓冲的区别很关键：不带缓冲的通道，发送方每次发数据都必须等接收方就位才能继续；带缓冲的通道，发送方可以先把数据塞进「队列」，只要队列没满就不会阻塞。这里容量设成 5，意味着 main 协程最多可以连续发 5 个 int 进去而不用等工作协程来取。

done 是一个无缓冲的 bool 通道，它的作用不是传数据，而是做「同步」——用来让 main 协程等工作协程彻底干完再退出，后面会细说。

─────────────────────────────
第二段：go func — 工作协程的 for 循环
─────────────────────────────

```go
go func() {
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
}()
```

这里用了一个无限 for 循环，每次迭代都执行 j, more := <-jobs。

关键就在这个双返回值写法。普通的 j := <-jobs 只拿值；j, more := <-jobs 多拿一个布尔值 more。more 的含义是：

- more == true：这次成功收到了一个正常数据，j 就是那个数据。
- more == false：通道已经被关闭，并且通道里所有数据都已经被读完了。此时 j 是该类型的零值（int 的零值是 0），不代表任何有意义的数据。

注意「已关闭 + 数据读完」这两个条件必须同时满足，more 才会变成 false。如果通道已关但里面还剩几个数据没读，这几次接收的 more 依然是 true，你还是能正常拿到数据。这个设计很合理：拉面馆关门了，但窗口里还有三碗面，服务员照样要把这三碗端出去，不能因为关门就扔掉。

当 more 为 false 时，工作协程打印一句

## 🏃 跑一下试试
```bash
# closing_channels.go
cat << 'EOF' > closing_channels.go
package main

import "fmt"

func main() {
    jobs := make(chan int, 5)
    done := make(chan bool)

    go func() {
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
    }()

    for j := 1; j <= 3; j++ {
        jobs <- j
        fmt.Println("sent job", j)
    }
    close(jobs)
    fmt.Println("sent all jobs")

    <-done
}
EOF
go run closing_channels.go
```

预期输出（jobs 有 5 个缓冲槽，sent 那侧不会阻塞，通常全部先打完）：
```
sent job 1
sent job 2
sent job 3
sent all jobs
received job 1
received job 2
received job 3
received all jobs
```
> 注意：goroutine 调度不是铁板钉钉，偶尔 received 会插进 sent 之间，属正常现象，不是 bug。

## 💡 师兄的碎碎念
- **只有发送方能 close**：谁写谁关，接收方调 `close` 是经典的设计错误；关两次或者往已关闭的 channel 发数据，运行时直接 panic，没得商量。
- **读已关闭的 channel 不会 panic**：可以一直读，读完缓冲里的真实数据后会无限返回零值，`more` 同时变成 `false`——这就是 worker 判断「活干完了」的依据。
- **`done` vs `WaitGroup`**：一个 goroutine 收尾用无缓冲 `done` 通道最直接；多个 goroutine 并发跑完再汇总，用 `sync.WaitGroup` 更合适，后面的关卡会专门讲。
- **下一关剧透**：`j, more := <-jobs` 这套写法可以直接换成 `for j := range jobs`，range 内部帮你处理 `more` 的判断，close 之后自动退出循环，少写好几行样板代码。

## 🎓 这一关的知识点清单
- [ ] `close(jobs)` 在发完 3 个 job **之后**调用，且只调用一次
- [ ] worker goroutine 用 `j, more := <-jobs` 判断 `more == false` 来退出循环
- [ ] `done` 通道无缓冲，`done <- true` 与 `<-done` 配合做同步，不是用 sleep 卡时间
- [ ] jobs 通道声明时给了 5 的缓冲，main 发完 3 个不会阻塞就能继续打印 `sent all jobs`
- [ ] 整个程序没有 goroutine 泄漏：worker 一定能跑完并往 done 写

## ➡️ 下一关
第 34 关是「通道遍历 range-over-channels」。上一关你还得手写 `j, more := <-jobs` 加 `if !more { break }`，下一关直接换成 `for j := range jobs`，Go 会自动帮你检测 close 信号然后退出循环，代码瞬间清爽一截。两关放一起对比着看效果最好，走起。

👉 [第 34 关：通道遍历](../34-range-over-channels/)
