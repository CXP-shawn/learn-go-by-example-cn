# 第 73 关：HTTP 客户端（师兄带你学 Go）

## 🎯 这一关你会学到

- `net/http` 包在 Go 标准库里身兼两职，既能当 **HTTP 客户端**发请求，也能当 **HTTP 服务端**处理请求，本关聚焦客户端侧。
- `http.Get(url)` 是最简洁的 GET 快捷方式，底层使用全局 `DefaultClient`，适合脚本和演示场景。
- 拿到 `*http.Response` 后必须 `defer resp.Body.Close()`，否则底层 TCP 连接无法归还连接池，造成连接泄漏。
- `resp.Status` 返回人类可读的字符串（如 `"200 OK"`），`resp.StatusCode` 返回整数（如 `200`），按需选用。
- HTTP 4xx / 5xx 响应**不会**被当作 `error` 返回，`err` 只在网络层面出错时才非 `nil`，业务逻辑须自行检查状态码。
- `bufio.Scanner` 可以逐行读取 `resp.Body`，配合计数器即可方便地截取前 N 行；生产环境建议用自定义 `http.Client` 并配置 `Timeout`。

## 🤔 先想一个问题

你平时有没有用过 `curl` 发 HTTP 请求，或者在 Python 里用过 `requests.get(url)` 一行搞定网络调用？Go 里做同样的事情其实一点都不麻烦。标准库 `net/http` 内置了完整的 HTTP 客户端实现，不需要安装任何第三方依赖，`http.Get(url)` 一行代码就能发出一个 GET 请求，拿回服务器的响应状态和响应体。和 `curl` 相比，Go 的优势在于你可以直接在代码里处理返回结果、解析 JSON、控制超时，而不需要再 shell 转接一道。本关就以请求 `gobyexample.com` 为例，把客户端发请求、读响应状态、逐行打印响应体这套流程跑一遍，让你对 Go 的 HTTP 客户端有个清晰的第一印象。

## 📖 看代码

```go
// Go 标准库的 `net/http` 包为 HTTP 客户端和服务端提供了出色的支持。
// 在这个例子中，我们将使用它发送简单的 HTTP 请求。

package main

import (
	"bufio"
	"fmt"
	"net/http"
)

func main() {

	// 向服务端发送一个 HTTP GET 请求。
	// `http.Get` 是创建 `http.Client` 对象并调用其 `Get` 方法的快捷方式。
	// 它使用了 `http.DefaultClient` 对象及其默认设置。
	resp, err := http.Get("http://gobyexample.com")
	if err != nil {
		panic(err)
	}
	defer resp.Body.Close()

	// 打印 HTTP response 状态.
	fmt.Println("Response status:", resp.Status)

	// 打印 response body 前面 5 行的内容。
	scanner := bufio.NewScanner(resp.Body)
	for i := 0; scanner.Scan() && i < 5; i++ {
		fmt.Println(scanner.Text())
	}

	if err := scanner.Err(); err != nil {
		panic(err)
	}
}
```

## 🔍 师兄给你逐行拆

### net/http 包总览

小师弟，学 Go 网络编程，第一个要认识的就是标准库里的 `net/http` 包。这个包有意思的地方在于，它同时扮演两个角色：**既能当客户端，替你去外面发请求、拿数据；也能当服务端，替你守着端口、接收别人发来的请求**。就好比咱们吃饭用的筷子，夹菜是它，搅汤也是它，一双筷子两种用法，顺手得很。

很多从其他语言转过来的同学会有一个疑问：Python 要发 HTTP 请求，得先 `pip install requests`，装一个第三方库；Node.js 里要么用内置的 `http` 模块写一堆回调，要么引入 `axios` 或者浏览器端的 `fetch`，才算顺手。而在 Go 里，打开标准库文档，`net/http` 就在那里，不需要额外安装任何东西，开箱即用，这是 Go 在工程实践上非常务实的一个设计理念。

这一关我们先只聚焦在**客户端**这一侧，也就是用 Go 去主动发起 HTTP 请求的场景，比如调用第三方接口、抓取数据、对接微服务等等。服务端的用法同样重要，但那是后面的事，先把一双筷子的客户端那一头玩得明白再说。后面的课程里我们会把服务端那一头也补上，到时候你就会发现，Go 原生的网络编程体验真的非常舒服。

### http.Get 快捷方式

小师弟，如果你只是想快速发一个 GET 请求，`http.Get` 就是你的首选。它是 Go 标准库里最简洁的 HTTP 调用方式，你只需要传入一个 URL 字符串，它就会帮你完成整个请求过程，返回 `*http.Response` 和 `error` 两个值。

```go
resp, err := http.Get("https://api.example.com/data")
```

从源码来看，`http.Get` 本质上只是对 `http.DefaultClient.Get` 的一层薄封装，底层用的是包级别的默认客户端 `DefaultClient`。这意味着你不需要自己创建任何客户端实例，直接调用即可，非常适合写原型代码或者做简单的接口验证。

返回的 `resp` 是一个 `*http.Response` 结构体，里面包含了你需要的全部响应信息：`Status` 是可读的状态字符串（比如 `"200 OK"`），`StatusCode` 是整数状态码，`Header` 是响应头的键值集合，`Body` 是可读取的响应体流。

这里有一点师兄要重点提醒你：**拿到返回值之后，第一件事永远是判断 `err` 是否为 `nil`，绝对不能跳过。** 网络请求失败的原因多种多样——DNS 解析失败、TCP 连接被拒绝、请求超时、对端服务器挂掉……任何一个环节出问题都会让 `err` 携带错误信息返回。你可以把它想象成点外卖下单：你下了单不代表餐一定会来，骑手可能路上迟到，可能找不到你的地址，甚至餐厅可能临时关门。你必须等确认送达才能放心，而不是下完单就当吃到了。

```go
if err != nil {
    log.Fatalf("请求失败: %v", err)
}
defer resp.Body.Close()
```

同时别忘了用 `defer resp.Body.Close()` 及时关闭响应体，避免连接泄漏。掌握好这两个习惯，你使用 `http.Get` 就已经稳了。

### defer resp.Body.Close：一定要记得关

师兄今天要跟你聊一个 Go 新手几乎必踩的坑：用 `http.Get` 发完请求，一定要手动关闭响应体。

`http.Get` 成功后会返回一个 `*http.Response`，它里面有个 `Body` 字段，类型是 `io.ReadCloser`。顾名思义，这个接口既能读，也必须关。你可以把它想象成去图书馆借了一本书，借出来之后必须还回去，忘了还的话图书馆会封你的账号，以后再也借不了。`Body` 也是一样的道理——底层其实是一条 TCP 连接，如果你读完内容之后不调用 `Close`，这条连接就不会被归还给连接池，而是白白占着资源。程序跑久了，连接池耗尽，新的请求全部阻塞，最终整个服务挂掉，排查起来还特别费劲。

正确的写法是这样的：

```go
resp, err := http.Get(url)
if err != nil {
    return err
}
defer resp.Body.Close()
```

注意顺序非常关键：**先判断 `err` 是否为 `nil`，确认 `resp` 不是空指针之后，再写 `defer` 关闭 Body。** 很多同学图方便，习惯把 `defer` 写在 `err` 判断之前，像下面这样：

```go
resp, err := http.Get(url)
defer resp.Body.Close() // ❌ 危险！
if err != nil {
    return err
}
```

一旦 `http.Get` 返回错误，`resp` 就是 `nil`，这时候 `defer` 去访问 `nil.Body` 直接触发空指针崩溃，程序当场 panic。这个问题隐蔽之处在于，本地网络好的时候请求总是成功，`resp` 从来不是 `nil`，测试怎么跑都过，一上线遇到偶发网络抖动，程序立刻崩给你看。

所以师兄的建议就是把这个顺序刻进肌肉记忆：**拿到响应、检查错误、紧跟 defer 关闭 Body**，三步缺一不可，顺序不能乱。养成这个习惯，你的 HTTP 客户端代码才算真正稳健。

### Status 和 StatusCode 长啥样

小师弟，拿到 `resp` 之后，你会发现里面有两个长得很像、却用途不同的字段：`Status` 和 `StatusCode`，师兄今天把它们说清楚。

`Status` 是一个字符串，格式固定为「状态码 + 空格 + 描述文字」，比如 `"200 OK"`、`"404 Not Found"`、`"500 Internal Server Error"`。它的好处是一眼就能读懂，非常适合写日志的时候直接打印出来，让排查问题的人不用再去查状态码手册。

`StatusCode` 则是一个纯整数，比如 `200`、`301`、`404`、`503`。在代码里做条件判断时，整数比字符串方便得多——你可以直接写 `if resp.StatusCode == 200` 或者 `if resp.StatusCode >= 400`，逻辑清晰，也不容易因为字符串拼写错误踩坑。

这里有一个非常重要的提醒，师兄要单独说，请你认真记住：**HTTP 的 4xx 和 5xx 响应，并不会让 `err` 变成非 `nil` 的值。** 也就是说，服务器返回 `404 Not Found` 或者 `500 Internal Server Error` 时，`err` 依然是 `nil`，`resp` 依然是有效的对象。Go 的 `http.Get` 只在网络层面出问题——比如连不上服务器、DNS 解析失败——才会把 `err` 置为非 `nil`。

这就好比你点的外卖，订单状态显示「已送达」，但打开门一看，盒子是空的。快递员确实把东西送到门口了（网络请求成功），但里面有没有货、货对不对，你还得自己打开检查（检查 `StatusCode`）。

所以正确的姿势是：**先判断 `err != nil`，再判断 `resp.StatusCode` 是否符合预期**，两道关卡都过了，才算业务真正成功。

### Scanner：一行一行地读

示例代码里用的是 `bufio` 包的 `NewScanner`，把 `resp.Body` 包进去，然后调用 `Scan` 方法循环读取，每次拿到一行文本，只读前 5 行就主动跳出循环。写法大致如下：

```go
scanner := bufio.NewScanner(resp.Body)
for i := 0; i < 5 && scanner.Scan(); i++ {
    fmt.Println(scanner.Text())
}
```

这种逐行读取的最大好处是**省内存**。整个响应体并不会一次性载入内存，而是读一行、处理一行、丢一行，非常适合处理大文件或按行结构组织的文本响应，比如服务器推送的日志流、NDJSON 数据流等场景。

如果你的需求恰好相反，想把整个 body 一口气吞掉，用 `io` 包的 `ReadAll` 函数就够了，它直接返回一个字节切片，简单直接，适合响应体不大、需要反复读取的情况。

如果响应是标准的 JSON 结构，师兄更推荐用 `json` 包的 `NewDecoder` 直接把 body 解码进你预先定义好的 struct，省去手动把字节切片再传给 `json.Unmarshal` 的中间步骤，代码既干净又少一次内存拷贝。

这里有个坑要特别提醒：`Scanner` 默认每行最大只能处理 **64 KB**，一旦遇到超长的行，内容会被静默截断，不报错、不警告，非常隐蔽。如果你的业务场景可能出现超长行，记得提前调用 `scanner.Buffer` 方法，手动传入一个更大的缓冲区，把上限调高。

用拆快递来打个比方：`ReadAll` 相当于把整个箱子抱回家，一次拆完所有东西；`Scanner` 则是站在门口一层层揭开包装，边拆边看、边看边处理。选哪种方式，取决于你的包裹有多大、你的内存有多宽裕。

### 生产实战：不要用 DefaultClient

小师弟，今天师兄要给你讲一个在生产环境里非常容易踩的坑。

你平时写爬虫或者调外部接口，可能随手就敲了一句 `http.Get(url)`，觉得简洁又方便。但你知道它底层用的是什么吗？答案是 `http.DefaultClient`——一个包级别的全局客户端。问题就出在这里：**`http.DefaultClient` 压根儿没有设置任何超时时间**。

这意味着什么？假设你调用的某个下游服务突然变慢，或者网络抖动，连接迟迟得不到响应，你的 goroutine 就会在那里傻等，永远挂起，既不报错，也不退出。在高并发场景下，这类挂起的 goroutine 会越积越多，轻则内存暴涨，重则整个服务彻底失去响应。

正确的做法是自己创建一个 `http.Client`，并明确设置 `Timeout` 字段。师兄一般习惯设 10 秒，根据业务场景可以适当调整：

```go
client := &http.Client{
    Timeout: 10 * time.Second,
}
resp, err := client.Get(url)
if err != nil {
    // 超时或其他错误，及时处理
    log.Fatal(err)
}
defer resp.Body.Close()
```

有了 `Timeout`，一旦请求超过规定时间，`client.Get` 就会返回一个 error，你的 goroutine 能够正常退出，不会再傻等到天荒地老。

还有第二个坑要提醒你：**不要在每次请求时都 `new` 一个新的 `http.Client`**。`http.Client` 内部维护着一个连接池（通过 `http.Transport` 实现），复用同一个实例可以有效复用 TCP 连接，大幅降低握手开销，提升吞吐量。正确的姿势是在包初始化或服务启动时创建一个全局的 `http.Client`，然后在所有请求中共享这一个实例。

记住这两条原则：**一定要设超时，一定要复用客户端**。把它们刻进脑子里，生产环境才能睡得安稳。

## 🏃 跑一下试试

确保当前机器可以访问外网，然后直接运行源文件：

```bash
go run http-clients.go
```

一切正常的话，终端会先打印响应状态行，再输出响应体的前 5 行 HTML，大致效果如下：

```
Response status: 200 OK
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Go by Example</title>
```

`200 OK` 说明请求成功，后面几行是 `gobyexample.com` 首页 HTML 的开头部分，Scanner 读到第 5 行就停了，不会把整页内容都倒出来。

如果你的网络不通或者 DNS 解析失败，`http.Get` 会返回一个非 `nil` 的 `error`，示例代码里用了 `panic(err)` 直接崩掉，终端会看到一长串错误堆栈。这在演示代码里无所谓，但生产环境里记得换成正经的错误处理，别让一次网络抖动把整个进程干掉。另外，如果你在公司内网或容器里跑，记得检查代理设置，`net/http` 默认会读取 `HTTP_PROXY` 和 `HTTPS_PROXY` 环境变量。

## 💡 师兄的碎碎念

**师兄碎碎念**

第一件事：**生产代码绝对不要用 `http.DefaultClient`**。`DefaultClient` 没有设置 `Timeout`，一旦服务端不响应，你的 goroutine 就会永远卡在那里，连接池慢慢耗尽，整个服务就挂了。正确姿势是自己 new 一个：`client := &http.Client{Timeout: 10 * time.Second}`，然后用 `client.Get(url)`，省心很多。

第二件事：**`defer resp.Body.Close()` 要放在 `err != nil` 判断之后**。如果 `http.Get` 返回了错误，`resp` 可能是 `nil`，这时候对 `nil` 调用 `Close()` 会直接 panic。正确写法是先 `if err != nil { ... }`，确认 `resp` 非 nil 再 defer。

第三件事：`bufio.Scanner` 适合逐行处理，但如果你只是想把整个响应体读进内存，直接用 `io.ReadAll(resp.Body)` 更简洁，返回 `[]byte`，再 `string()` 转一下就能打印。如果响应体是 JSON，更推荐 `json.NewDecoder(resp.Body).Decode(&result)`，直接把 body 流式解码到结构体，不用先读成字节再 Unmarshal，减少一次内存拷贝。

第四件事：如果你需要**设置请求头、发 POST、带 Cookie**，`http.Get` 就不够用了，换成 `http.NewRequest(method, url, body)` 构造请求，拿到 `*http.Request` 后往 `req.Header` 里塞 KV，再用 `client.Do(req)` 发出去，灵活得多。这是实际项目里用得最多的姿势，值得早点熟悉。

最后：Go 的 HTTP 客户端默认会**复用底层 TCP 连接**（Keep-Alive），只要你把 `resp.Body` 完整读完再关闭，连接就能归还连接池给下次请求用。如果你中途不读完就 `Close()`，连接会被丢弃而非复用，高并发场景下会产生大量 TIME_WAIT，留意一下。

## 🎓 知识点清单

- [ ] 能说清楚 `net/http` 包为什么同时支持客户端和服务端两种角色
- [ ] 理解 `http.Get` 只是快捷方式，底层仍走 `DefaultClient`
- [ ] 记得用 `defer resp.Body.Close()` 防止连接泄漏，且要在判断 `err == nil` 之后才调用
- [ ] 明白 4xx / 5xx 响应不会让 `err != nil`，需自行检查 `resp.StatusCode`
- [ ] 能区分 `resp.Status`（字符串 `"200 OK"`）与 `resp.StatusCode`（整数 `200`）的使用场景
- [ ] 知道生产环境要自定义 `http.Client` 并设置 `Timeout`，避免请求永久挂起

## ➡️ 下一关

下一关我们把角色翻转过来——从发请求变成**接受请求**。`net/http` 同样内置了 HTTP 服务端能力，几行代码就能跑起一个真正的 Web 服务器，处理路由、读取参数、写回响应。快去看看吧：[第 74 关：HTTP 服务端](../74-http-servers/README.md)。
