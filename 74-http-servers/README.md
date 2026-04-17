# 第 74 关：HTTP 服务端（师兄带你学 Go）

## 🎯 这一关你会学到

- 理解 **`Handler` 接口**：只要实现 `ServeHTTP(ResponseWriter, *Request)` 方法，任意类型都能成为路由处理器。
- 掌握 **`HandlerFunc` 适配器**：它把普通函数转换为 `Handler`，`http.HandleFunc` 内部正是借助它完成注册的。
- 熟悉 handler 的 **两参数签名** `(w http.ResponseWriter, r *http.Request)`：`w` 用于写响应，`r` 携带请求信息。
- 学会用 **`http.HandleFunc`** 向默认路由表注册路径与处理函数的映射关系。
- 理解 **`http.ListenAndServe`** 的职责：绑定端口、接受连接、为每个请求启动 goroutine 并分发到对应 handler。
- 区分 **`DefaultServeMux`（默认）与自定义 `ServeMux`**：前者是包级全局变量，方便快速开发；后者隔离路由表，适合库或多服务场景。

## 🤔 先想一个问题

师弟，有没有想过一个 Web 服务器底层到底是怎么跑起来的？在 C 或 Java 里，你可能要手动 `bind`、`listen`、`accept`，写一大堆样板代码，还得自己管线程池。Go 里呢？两个函数调用就够了：`http.HandleFunc` 注册路由，`http.ListenAndServe` 开门迎客——剩下的连接管理、并发调度，标准库全帮你搞定。更妙的是，Go 的 HTTP handler 只是一个普通函数，签名固定、职责单一，测试起来毫不费力。这一关我们从最简单的 `hello` 和 `headers` 两个 handler 出发，把服务端的核心骨架摸清楚，为后续处理真实业务打好地基。

## 📖 看代码

```go
// 使用 `net/http` 包，我们可以轻松实现一个简单的 HTTP 服务器。

package main

import (
	"fmt"
	"net/http"
)

// *handlers* 是 `net/http` 服务器里面的一个基本概念。
// handler 对象实现了 `http.Handler` 接口。
// 编写 handler 的常见方法是，在具有适当签名的函数上使用 `http.HandlerFunc` 适配器。
func hello(w http.ResponseWriter, req *http.Request) {

	// handler 函数有两个参数，`http.ResponseWriter` 和 `http.Request`。
	// response writer 被用于写入 HTTP 响应数据，这里我们简单的返回 "hello\n"。
	fmt.Fprintf(w, "hello\n")
}

func headers(w http.ResponseWriter, req *http.Request) {

	// 这个 handler 稍微复杂一点，
	// 我们需要读取的 HTTP 请求 header 中的所有内容，并将他们输出至 response body。
	for name, headers := range req.Header {
		for _, h := range headers {
			fmt.Fprintf(w, "%v: %v\n", name, h)
		}
	}
}

func main() {

	// 使用 `http.HandleFunc` 函数，可以方便的将我们的 handler 注册到服务器路由。
	// 它是 `net/http` 包中的默认路由，接受一个函数作为参数。
	http.HandleFunc("/hello", hello)
	http.HandleFunc("/headers", headers)

	// 最后，我们调用 `ListenAndServe` 并带上端口和 handler。
	// nil 表示使用我们刚刚设置的默认路由器。
	http.ListenAndServe(":8090", nil)
}
```

## 🔍 师兄给你逐行拆

### handler 到底是啥

师弟，你刚开始写 Go 的 Web 服务时，一定会被 `handler` 这个词绕晕。别慌，师兄今天给你讲清楚。

在 Go 的 `net/http` 包里，handler 是处理 HTTP 请求的最小单元。它的本质是一个实现了 `Handler` 接口的对象。这个接口定义极其简洁，里面只有一个方法，签名长这样：

```go
ServeHTTP(w ResponseWriter, r *Request)
```

只要你的类型实现了这个方法，它就是一个合法的 handler，可以直接交给服务器使用。

服务器的工作流程是这样的：客户端发来一个请求，服务器先看请求的 URL 路径，再去路由表里找匹配的 handler，找到之后就调用这个 handler 的 `ServeHTTP` 方法。在这个方法里，你拿到 `w`（ResponseWriter）往客户端写响应，拿到 `r`（*Request）读取请求里的各种信息，比如请求头、请求体、查询参数等等。整个请求的处理逻辑，全部由这一次 `ServeHTTP` 调用来完成。

打个比方，你可以把整个 Web 服务想象成一个客服中心。客户打进来的电话，前台会先听一下诉求，然后按业务分工转接——投诉的转给投诉组，咨询产品的转给咨询组，要办退款的转给退款组。每个小组只负责自己那块业务，互不干扰。这里的每一个小组，就对应着我们代码里的一个 handler。路由器就是那个负责转接的前台，它按 URL 路径判断该把请求交给哪个小组来接待。

理解了这个分工模型，你再去看 `http.HandleFunc`、`http.Handle` 这些函数，就会发现它们不过是在往路由表里登记一条条"路径到处理逻辑"的对应关系，本质上都是在构建这张分工转接表。

### HandlerFunc 适配器：让普通函数变成 handler

师兄来给你讲一个 Go 标准库里设计得非常优雅的小东西——`http.HandlerFunc`。

先回顾一下 `Handler` 接口的定义：

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

按照接口规矩，你得先定义一个结构体，再给它实现 `ServeHTTP` 方法，才能把它注册成路由处理器。每写一个 handler 就要造一个结构体，代码量一大，光结构体定义就能把你淹死。

`HandlerFunc` 就是来解决这个麻烦的。它的定义只有两行：

```go
type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)
}
```

你看明白了吗？`HandlerFunc` 是一个**函数类型**，而不是结构体。只要一个普通函数的签名是 `func(ResponseWriter, *Request)`，把它强制转换成 `HandlerFunc`，它就自动满足了 `Handler` 接口——因为 `HandlerFunc` 已经帮你实现了 `ServeHTTP`，内部就是调用函数本身。

师兄打个比方：这就像一个**万能转接头**。你手里有一根普通的双脚插头（普通函数），插座要求三孔（Handler 接口），`HandlerFunc` 就是那个转接头，套上去就能用，不需要你重新换一根线。

在实际代码里，`hello` 和 `headers` 就是这样的普通函数：

```go
func hello(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "Hello, 师弟!")
}

func headers(w http.ResponseWriter, r *http.Request) {
    for k, v := range r.Header {
        fmt.Fprintf(w, "%v: %v\n", k, v)
    }
}
```

当你调用 `http.HandleFunc("/hello", hello)` 注册路由时，标准库内部做的事情正是 `mux.Handle(pattern, HandlerFunc(handler))`，悄悄地帮你套上了那个转接头。

理解了 `HandlerFunc` 之后，你会发现 Go 的接口设计哲学非常务实：能用类型系统解决的问题，就不需要额外的脚手架代码。写 handler，直接写函数就好，结构体留给真正需要携带状态的场景。

### ResponseWriter 和 Request 两个好兄弟

小师弟，每次你注册一个 handler 函数，Go 都会把两个参数塞到你手里：`http.ResponseWriter` 和 `*http.Request`。这两位可是搭档关系，一个负责「说话」，一个负责「听话」，缺了谁都转不起来。

先说 `ResponseWriter`。它其实是一个接口，不是具体的结构体。你往它里面写字节，HTTP 框架就会把这些字节作为响应体发送给客户端。最常用的姿势是 `fmt.Fprintf(w, "Hello, %s", name)`，简洁又顺手，跟平时写格式化输出没什么两样。除了写响应体，它还提供了两个重要方法：`Header()` 返回一个 `http.Header`，你可以在这里设置 `Content-Type`、`Set-Cookie` 之类的响应头，**注意要在写响应体之前设置**，否则头部已经发出去了再改就晚了；`WriteHeader(statusCode int)` 用来设置 HTTP 状态码，比如 `w.WriteHeader(http.StatusNotFound)` 就会返回 404，同样要在写响应体之前调用。如果你不手动调用 `WriteHeader`，第一次调用 `Write` 时会自动发出 200。

再看 `*http.Request`。它是一个指针，指向 `http.Request` 结构体，里面装着客户端请求的全部信息。`r.Method` 告诉你是 GET 还是 POST；`r.URL` 是请求路径，`r.URL.Query()` 可以直接拿到查询参数的键值对；`r.Header` 存放请求头，比如 `r.Header.Get("Authorization")`；`r.Body` 是请求体，读完记得关闭。表单数据先调用 `r.ParseForm()`，再用 `r.FormValue("key")` 取值，非常方便。

记住这个口诀：**w 写出去，r 读进来**。两兄弟分工明确，配合默契，把这对搭档摸透，handler 开发就稳了一大半。

### HandleFunc：注册路由就一行

小师弟，路由注册是写 HTTP 服务绕不开的第一步，而 `http.HandleFunc` 就是 Go 标准库给你准备的最省事的入口。

调用 `http.HandleFunc` 时，你只需要传入两个参数：一个路径字符串，以及一个签名为 `func(http.ResponseWriter, *http.Request)` 的函数。底层它会把这对映射关系写进一个叫做 `DefaultServeMux` 的默认路由器里。后续 `http.ListenAndServe` 如果第二个参数传 `nil`，用的就是这个默认路由器。示例代码里注册了两条路由：`/hello` 路径交给 `hello` 函数来响应，`/headers` 路径交给 `headers` 函数来响应，两行代码，清晰直白。

不过师兄要提醒你几个容易踩的坑，记住了能省不少调试时间。

**第一个坑：重复注册同一个路径会直接 panic。** 标准库不会给你返回一个 error 让你优雅地处理，而是运行时直接崩溃。所以同一个路径只能注册一次，别想着覆盖。

**第二个坑：`DefaultServeMux` 是包级全局变量。** 这意味着你引入的任何第三方库，只要它在 `init` 函数里调用了 `http.HandleFunc`，就会悄无声息地往这个全局路由器上挂载路由。你自己都不知道服务上多了哪些路径，存在一定的安全隐患。

**第三个坑也是最重要的建议：复杂项目里，请自己 `new` 一个 `ServeMux`。** 用 `mux := http.NewServeMux()` 创建独立的路由器实例，然后把它传给 `http.ListenAndServe`，路由注册的范围就完全由你自己掌控，与全局状态彻底隔离，项目结构也会更清晰可维护。

简单项目用 `DefaultServeMux` 足够，但养成隔离的习惯，从一开始就受益无穷。

### ListenAndServe：服务真正跑起来

小师弟，路由注册好之后，服务还没有真正跑起来，还差最后一步——调用 `http.ListenAndServe(addr, handler)`。这个函数做的事情说白了就两件：**绑定端口**，然后**开始接受请求**。

先说第一个参数 `addr`。你写 `":8090"` 的时候，冒号前面是空的，意思是监听本机**所有网卡**的 8090 端口。不管你的机器有几块网卡、几个 IP，只要流量打到 8090，服务都能收到。如果你只想监听本地回环，可以写 `"127.0.0.1:8090"`，这样外部流量就进不来了，调试的时候偶尔会用到。

再说第二个参数 `handler`。这里传 `nil`，Go 会自动用内置的 `DefaultServeMux` 来做路由分发。你之前用 `http.HandleFunc` 注册的那些路由，默认就挂在 `DefaultServeMux` 上，所以传 `nil` 完全没问题，省事又直接。

有一点要特别记住：`ListenAndServe` 是**阻塞**的。它一旦开始监听，就会一直卡在那里，直到出错才会返回。所以你会看到很多项目在最后用 `log.Fatal(http.ListenAndServe(...))` 来处理错误，一旦函数返回就把错误信息打出来，程序随即退出。最常见的错误是端口被占用，错误信息会提示 `address already in use`，遇到这个直接换个端口或者把占用进程干掉就好。

最后同步一个重要细节：Go 的 HTTP 服务器对**每一个进来的请求**都会开一个新的 goroutine 去处理，这是默认行为，不需要你写任何额外代码。天然并发，轻量高效，这也是 Go 做 Web 服务的一大优势。理解了这一点，你写处理函数的时候也要有并发意识，共享变量记得加锁。

### 两个 handler 各司其职

回到本关的代码，我们一共注册了两个路由，各自挂了一个 handler，两者分工明确、互不干扰。

先说 `hello` 这个 handler。它的逻辑极其精简，只做一件事：往响应里写一行 `hello` 加换行符，然后返回。别看它简单，它的价值在于让你第一次把服务真正跑起来，浏览器或者 `curl` 打过来，你能看到响应，心里就有了底——框架没问题、端口没问题、路由没问题，后续一切都可以在这个基础上继续堆叠。

再说 `headers` 这个 handler，它就稍微有点意思了。HTTP 请求到达服务端时，除了请求行和请求体，还携带了一批请求头。在 Go 里，`r.Header` 的类型是 `map[string][]string`，键是 header 的名字，值是一个字符串切片——之所以是切片，是因为同一个 header 名在协议层面允许出现多次。`headers` 这个 handler 会把这张 map 完整遍历一遍，把每个 header 的名字和对应的值都格式化之后写回响应，客户端收到的就是自己发出去的全部请求头，相当于一个"请求头回显工具"。工作中排查问题时，这种小工具其实非常实用——不管是前端的 Content-Type、反向代理加的 X-Forwarded-For，还是各种自定义的认证头，挂一个类似的 handler 就能一眼看穿客户端到底往你这儿发了什么，比翻日志或者自己 printf 方便得多。

## 🏃 跑一下试试

先在终端启动服务。由于 `http.ListenAndServe` 会持续监听，命令执行后**终端会阻塞**，这是正常现象——服务就在后台默默等待请求。

```text
$ go run http-servers.go
```

不要关闭这个终端。**新开一个终端窗口**，用 `curl` 分别访问两个路由。

访问 `/hello`：

```text
$ curl localhost:8090/hello
hello
```

访问 `/headers`，会把你的请求头原样打印出来：

```text
$ curl localhost:8090/headers
User-Agent: curl/8.6.0
Accept: */*
```

实际输出的字段顺序和内容会随 `curl` 版本、系统环境略有不同，不必纠结。

**端口占用的情况**：如果你忘记关掉上一次运行的进程，再次执行 `go run` 会看到类似这样的报错：

```text
listen tcp :8090: bind: address already in use
```

遇到这条信息，用 `lsof -i :8090`（macOS/Linux）或 `netstat -ano | findstr 8090`（Windows）找到占用进程，结束它后再重试即可。养成每次测试结束用 `Ctrl+C` 停掉服务的习惯，能省不少麻烦。

## 💡 师兄的碎碎念

师兄碎碎念几句，都是踩过坑才懂的东西。

**生产环境别用裸 `http.ListenAndServe`。** 它内部创建的服务器没有任何超时配置，客户端只要不关连接，服务器就得一直等。正确做法是显式构造 `http.Server` 结构体：

```go
srv := &http.Server{
    Addr:         ":8090",
    ReadTimeout:  5 * time.Second,
    WriteTimeout: 10 * time.Second,
    IdleTimeout:  60 * time.Second,
}
srv.ListenAndServe()
```

`ReadTimeout` 防止客户端慢慢发请求耗光连接；`WriteTimeout` 限制 handler 写响应的时间；`IdleTimeout` 控制 keep-alive 连接的最大空闲时长。三个参数一起上，基本能挡住大多数慢速攻击。

**优雅关闭别忘了。** 收到 `SIGINT` / `SIGTERM` 信号时，调用 `srv.Shutdown(ctx)` 而不是直接 `os.Exit`，这样能等已有请求处理完再退出，避免连接被强制断开。

**路由进阶。** 标准库的 `ServeMux` 功能有限，路径参数、中间件链都要自己实现。项目稍微复杂一点，推荐看看 `chi`（轻量、兼容标准库接口）或 `gin`（功能更全、生态丰富），按需选择。

**并发安全。** 每个请求都跑在独立 goroutine 里，handler 内部如果读写共享状态，务必加锁或用 channel 协调，否则数据竞争随时可能出现。

最后，`fmt.Fprintf(w, ...)` 是有返回值的，错误处理不要省略，线上环境记得 `log` 一下。

## 🎓 知识点清单

- [ ] 能用 `go run` 启动 HTTP 服务，并在 8090 端口正常监听
- [ ] 理解 `http.HandleFunc` 与 `Handler` 接口的关系，知道 `HandlerFunc` 适配器做了什么
- [ ] 能区分 `DefaultServeMux` 与自定义 `ServeMux`，知道各自的使用场景
- [ ] 用 `curl` 验证 `/hello` 和 `/headers` 两个路由均返回正确内容
- [ ] 了解 `http.ListenAndServe` 阻塞的原因，以及端口已被占用时的报错含义
- [ ] 知道生产环境应使用 `http.Server` 结构体并配置超时参数

## ➡️ 下一关

下一关 [第 75 关：Context](../75-context/README.md) 将带你掌握 Go 并发与网络编程中的核心利器——`context`。它让你能优雅地在调用链中传递取消信号、截止时间和请求域的数据，彻底告别各种全局变量和手动 channel 通知的混乱写法，是每个 Go 工程师都绕不开的必修课。
