# 第 75 关：Context（师兄带你学 Go）

## 🎯 这一关你会学到

- 理解 `context.Context` 是 Go 里跨 goroutine 传递

## 🤔 先想一个问题

你有没有遇到过这种情况：手机外卖 App 下完单，突然反悔了，手速飞快点了「取消订单」，结果骑手已经出发了，平台还在傻乎乎地处理你的地址？Go 的 HTTP Server 也会遇到一样的问题——客户端早就关掉浏览器走人了，后台的 goroutine 还在埋头苦干，白白消耗 CPU 和数据库连接。Context 就是 Go 给这个场景设计的

## 📖 看代码

```go
// 在前面的示例中，我们研究了配置简单的 [HTTP 服务器](http-servers)。
// HTTP 服务器对于演示 `context.Context` 的用法很有用的，
// `context.Context` 被用于控制 cancel。
// `Context` 跨 API 边界和协程携带了：deadline、取消信号以及其他请求范围的值。

package main

import (
	"fmt"
	"net/http"
	"time"
)

func hello(w http.ResponseWriter, req *http.Request) {

	// `net/http` 机制为每个请求创建了一个 `context.Context`，
	// 并且可以通过 `Context()` 方法获取并使用它。
	ctx := req.Context()
	fmt.Println("server: hello handler started")
	defer fmt.Println("server: hello handler ended")

	// 等待几秒钟，然后再将回复发送给客户端。
	// 这可以模拟服务器正在执行的某些工作。
	// 在工作时，请密切关注 context 的 `Done()` 通道的信号，
	// 一旦收到该信号，表明我们应该取消工作并尽快返回。
	select {
	case <-time.After(10 * time.Second):
		fmt.Fprintf(w, "hello\n")
	case <-ctx.Done():
		// context 的 `Err()` 方法返回一个错误，
		// 该错误说明了 `Done` 通道关闭的原因。
		err := ctx.Err()
		fmt.Println("server:", err)
		internalError := http.StatusInternalServerError
		http.Error(w, err.Error(), internalError)
	}
}

func main() {

	// 跟前面一样，我们在 `/hello` 路由上注册 handler，然后开始提供服务。
	http.HandleFunc("/hello", hello)
	http.ListenAndServe(":8090", nil)
}
```

## 🔍 师兄给你逐行拆

### Context 是什么，解决什么问题

师兄先给你一个直觉：Context 在 Go 里扮演的是「对讲机」的角色。

想象你是一个施工队队长，手下有十几个工人分散在工地各处干活。突然甲方打电话说今天不施工了，你得通知到每一个工人停下来。如果每个工人都有一台对讲机，你对着总频道喊一声「收工」，所有人立刻就能听到。Context 就是这台对讲机：一个取消信号从根 Context 往下传，所有持有这个 Context 的 goroutine 都能收到。

Go 标准库里 `context.Context` 是一个接口，它主要解决三类问题：

1. **取消信号（Cancellation）**：某个操作不需要继续了，通知所有下游 goroutine 停下来。
2. **截止时间（Deadline / Timeout）**：设定一个最晚完成时间，超时自动触发取消。
3. **请求域的值（Values）**：在整个调用链上传递一些请求级别的元数据，比如 Trace ID、用户身份信息。

本关的示例以 HTTP Server 场景为主，师兄会带你逐行拆解，弄清楚客户端取消请求时，服务端如何优雅地感知并停止工作。

---

### 从 http.Request 拿 Context

在 Go 的 HTTP 处理函数里，`*http.Request` 上自带了一个 Context，可以用 `req.Context()` 取出来：

```go
func hello(w http.ResponseWriter, req *http.Request) {
    ctx := req.Context()
    fmt.Println("server: hello handler started")
    defer fmt.Println("server: hello handler ended")
    // ...
}
```

这个 `ctx` 的生命周期和 HTTP 请求绑定在一起——客户端一旦断开连接，Go 的 HTTP 框架会自动取消这个 Context。所以你不需要自己创建什么，直接拿来用就好。

`defer fmt.Println(...)` 那行是个好习惯，不管 handler 怎么退出，都会打印结束日志，方便排查。

---

### ctx.Done() 是什么

`ctx.Done()` 返回一个只读的 channel（类型是 `<-chan struct{}`）。当 Context 被取消或者超时的时候，这个 channel 会被关闭。

你可以把它理解成一个「哨兵 channel」：平时没人往里发消息，channel 是阻塞的；一旦有取消信号来了，channel 关闭，所有监听它的 `select` 分支都会立刻被触发。

关键点：**channel 关闭 ≠ 收到了某个值**，而是「读取一个已关闭 channel 会立即返回零值」，这是 Go 语言利用 channel 语义传播信号的经典技巧。

---

### select 实现「工作中可被打断」

示例里最核心的部分是这一段逻辑：

```go
select {
case <-time.After(5 * time.Second):
    fmt.Fprintf(w, "hello\n")
case <-ctx.Done():
    err := ctx.Err()
    fmt.Println("server:", err)
    internalError := http.StatusInternalServerError
    http.Error(w, err.Error(), internalError)
}
```

师兄逐行给你拆：

**`select` 语句**：Go 里的 `select` 和 `switch` 长得像，但它是专门为 channel 设计的，会等待多个 channel 操作，哪个先就绪就走哪个分支。

**`case <-time.After(5 * time.Second)`**：`time.After` 会创建一个计时器，5 秒后往 channel 里发一个值。如果 5 秒内没有取消信号，这个分支先触发，正常给客户端写回 `hello`。

**`case <-ctx.Done()`**：如果在 5 秒之内客户端断开了，`ctx.Done()` 的 channel 被关闭，这个分支就会先触发。进入这个分支后，我们调用 `ctx.Err()` 拿到具体的取消原因。

这个 `select` 的结构就是 Go 里处理「可打断工作」的标准模式，你以后在数据库查询、RPC 调用、文件 IO 里会反复见到它。

---

### ctx.Err() 告诉你为什么取消

`ctx.Err()` 返回一个 `error`，只有两种可能：

- `context.Canceled`：Context 被主动取消（比如客户端关掉了连接，或者上层代码调用了 `cancel()` 函数）。
- `context.DeadlineExceeded`：超过了设定的截止时间，Context 自动过期。

在本关的 HTTP 示例里，客户端断开连接对应的是 `context.Canceled`。你在日志里会看到 `server: context canceled`，就是它的错误信息。

区分这两种原因在实际项目里很有用：前者说明用户主动放弃了，后者说明系统响应太慢，需要优化性能或者调整超时配置。

---

### http.Error 返回错误状态码

`http.Error(w, errorMessage, statusCode)` 是 Go 标准库里一个简单的工具函数，它帮你做三件事：

1. 设置 `Content-Type` 为 `text/plain`。
2. 设置 HTTP 状态码为你传入的 `statusCode`。
3. 把 `errorMessage` 写进响应 body。

示例里用的是 `http.StatusInternalServerError`，也就是 500。你可以根据业务情况换成其他状态码，比如 `http.StatusServiceUnavailable`（503）表示服务暂时不可用，语义上更贴切一些。

注意：在写入 body 之前，Header 必须还没有被写出去，否则状态码会被忽略。`http.Error` 内部会先调用 `w.Header().Set(...)` 再写 body，所以你不能在它之前调用 `w.Write(...)`，否则 Header 已经发出去了，状态码改不了。

---

### 完整流程串联

把上面几个知识点串起来，整个 handler 的运行流程是这样的：

1. 请求进来，从 `req.Context()` 拿到和这个请求绑定的 Context。
2. 进入 `select`，同时监听两个 channel：5 秒计时器和 `ctx.Done()`。
3. 情况 A：5 秒内客户端一直等着，计时器先触发，正常返回 `hello`，请求完成。
4. 情况 B：5 秒内客户端断开连接，Go HTTP 框架自动取消 Context，`ctx.Done()` 先触发，调用 `ctx.Err()` 打日志，然后用 `http.Error` 返回 500（虽然这时候客户端可能已经不在了，但服务端该做的清理动作还是应该做）。
5. `defer` 保证不管哪种情况，「handler ended」的日志都会打出来。

这个模式的精髓在于：**业务逻辑和取消逻辑是并行监听的**，任何一个先就绪就优先处理，互不阻塞，代码结构非常清晰。

## 🏃 跑一下试试

把示例代码保存成 `context.go`，然后用下面的方式启动 HTTP 服务器：

```shell
$ go run context.go &
```

服务跑在后台之后，我们用 `curl` 模拟两种情况。

**情况一：正常等待完成（5 秒后返回）**

```shell
$ curl localhost:8090/hello
```

```text
hello\n
```
等了大概 5 秒，终端才打印出 `hello`，服务器日志会显示 `server: hello handler started` 和 `server: hello handler ended`，一切正常。

**情况二：客户端中途取消（模拟用户关掉浏览器）**

```shell
$ curl localhost:8090/hello &
$ sleep 2 && kill %1
```

```text
[1]  + 12345 terminated  curl localhost:8090/hello
```

这时候服务器日志会打出类似这样的内容：

```text
server: hello handler started
server: context canceled
server: hello handler ended
```

没有给客户端返回 `hello`，而是提前感知到了取消信号，打印了 `context canceled`，然后用 `http.Error` 返回了 500。这就是 Context 的价值所在：后台该停的时候真的停了，不会继续跑无意义的逻辑，也不会让数据库白忙活一场。记得跑完用 `kill` 关掉后台的服务进程。

## 💡 师兄的碎碎念

**师兄碎碎念：三个坑你早晚会踩**

**第一个坑：Context 不要塞进结构体。**

Go 官方文档里有一句话说得很直白：不要把 Context 存到结构体字段里，而应该显式地把它作为函数的第一个参数传进去，通常命名为 `ctx`。很多初学者图省事，把 `ctx` 存到某个 Service 结构体里，结果整个请求链路共享同一个 Context，一个请求取消了，其他请求也跟着受影响，排查起来非常头疼。记住，Context 是

## 🎓 知识点清单

- [ ] 能解释 `context.Context` 在 Go 里解决的三类问题：取消、超时、值传递
- [ ] 能从 `*http.Request` 上用 `req.Context()` 拿到当前请求的 Context
- [ ] 能用 `select` 同时监听 `ctx.Done()` 和业务 channel，实现可打断的工作循环
- [ ] 知道 `ctx.Err()` 返回的两种错误：`context.Canceled` 和 `context.DeadlineExceeded`
- [ ] 能用 `http.Error` 向客户端返回非 200 的错误状态码
- [ ] 理解 Context 只应向下传、不应存在结构体里的惯例

## ➡️ 下一关

下一关我们要玩点刺激的——直接从 Go 程序里启动外部进程，读它的输出、传参数给它，就像在代码里内嵌了一个小终端一样。感兴趣的话就去看 [第 76 关：生成进程](../76-spawning-processes/README.md) 吧，师兄在那里等你。
