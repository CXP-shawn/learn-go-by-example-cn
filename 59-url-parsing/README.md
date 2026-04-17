# 第 59 关：URL 解析（师兄带你学 Go）

## 🎯 这一关你会学到

学会用 net/url 包拆解一条完整 URL 的每一个字段：Scheme、User（Username/Password）、Host+Port、Path、Fragment、RawQuery，以及用 url.ParseQuery 把查询串变成可操作的 map，最后顺手学会反向拼出一条合法 URL。

## 🤔 先想一个问题

写爬虫时你拿到一条 "postgres://user:pass@host.com:5432/path?k=v#f"，想取端口号，结果手写字符串分割写了二十行还 panic——其实 Go 标准库一个 url.Parse() 就能把这串天书拆得明明白白。不管是数据库 DSN、OAuth redirect_uri 白名单校验，还是 HTTP 重定向安全审查，只要 URL 出现，net/url 就是你的瑞士军刀。

## 📖 看代码

```go
// URL 提供了[统一资源定位方式](http://adam.heroku.com/past/2010/3/30/urls_are_the_uniform_way_to_locate_resources/)。
// 这里展示了在 Go 中是如何解析 URL 的。

package main

import (
	"fmt"
	"net"
	"net/url"
)

func main() {

	// 我们将解析这个 URL 示例，它包含了一个 scheme、
	// 认证信息、主机名、端口、路径、查询参数以及查询片段。
	s := "postgres://user:pass@host.com:5432/path?k=v#f"

	// 解析这个 URL 并确保解析没有出错。
	u, err := url.Parse(s)
	if err != nil {
		panic(err)
	}

	// 直接访问 scheme。
	fmt.Println(u.Scheme)

	// `User` 包含了所有的认证信息，
	// 这里调用 `Username` 和 `Password` 来获取单独的值。
	fmt.Println(u.User)
	fmt.Println(u.User.Username())
	p, _ := u.User.Password()
	fmt.Println(p)

	//  `Host` 包含主机名和端口号（如果存在）。使用 `SplitHostPort` 提取它们。
	fmt.Println(u.Host)
	host, port, _ := net.SplitHostPort(u.Host)
	fmt.Println(host)
	fmt.Println(port)

	// 这里我们提取路径和 `#` 后面的查询片段信息。
	fmt.Println(u.Path)
	fmt.Println(u.Fragment)

	// 要得到字符串中的 `k=v` 这种格式的查询参数，可以使用 `RawQuery` 函数。
	// 你也可以将查询参数解析为一个 map。已解析的查询参数 map 以查询字符串为键，
	// 已解析的查询参数会从字符串映射到到字符串的切片，
	// 因此如果您只想要第一个值，则索引为 `[0]`。
	fmt.Println(u.RawQuery)
	m, _ := url.ParseQuery(u.RawQuery)
	fmt.Println(m)
	fmt.Println(m["k"][0])
}
```

## 🔍 师兄给你逐行拆

先把今天的主角亮出来：

postgres://user:pass@host.com:5432/path?k=v#f

这一条字符串包含了 URL 规范里几乎所有零件：scheme、用户名密码、主机、端口、路径、查询参数、锚点片段。我们逐段拆开讲。

────────────────────────
第一步：url.Parse() —— 入口只有一个
────────────────────────

u, err := url.Parse(s)
if err != nil {
    panic(err)
}

url.Parse 返回两个值：*url.URL 和 error。*url.URL 是一个结构体，里面每个字段对应 URL 的一个零件，解析完直接按字段取，不用自己切字符串。

重要提醒：面对任何用户输入或外部配置传进来的 URL，err 一定要检查。url.Parse 本身容错性很强，很多残缺 URL 它也不会报错，但如果字符串里有非法的控制字符，它就会 panic 你的服务。养成习惯，解析完先查 err，再访问字段。

────────────────────────
第二步：u.Scheme —— 协议头
────────────────────────

fmt.Println(u.Scheme) // postgres

Scheme 就是冒号前面那一截。HTTP 请求是 "http" 或 "https"，WebSocket 是 "ws"，数据库连接串常见 "postgres"、"mysql"、"mongodb"。这个字段纯字符串，直接用。

业务场景：OAuth redirect_uri 白名单校验，第一步就是检查 u.Scheme 是不是 "https"，不是就拒绝，防止 javascript: 伪协议注入的 XSS 攻击。

────────────────────────
第三步：u.User —— 认证信息
────────────────────────

fmt.Println(u.User)           // user:pass
fmt.Println(u.User.Username()) // user
p, _ := u.User.Password()      // pass
fmt.Println(p)

u.User 的类型是 *url.Userinfo，不是普通字符串。它把用户名和密码封装在一起，提供两个方法：

- Username() string：直接返回用户名。
- Password() (string, bool)：返回密码和一个布尔值。布尔值是 "这条 URL 里有没有密码" 的标志，不是 "密码对不对"。URL 里如果写了 user:@host，密码是空字符串，bool 是 true；如果只写了 user@host，bool 是 false。

这里有个很容易踩的坑：明文密码直接写在 URL 里，一旦你把 u.String() 或者原始字符串打印到日志，密码就泄露了。生产环境的数据库 DSN 千万不要直接 log.Println(dsn)，要么脱敏，要么把 User 字段置 nil 再打印。

────────────────────────
第四步：u.Host + net.SplitHostPort —— 主机和端口
────────────────────────

fmt.Println(u.Host) // host.com:5432
host, port, _ := net.SplitHostPort(u.Host)
fmt.Println(host) // host.com
fmt.Println(port) // 5432

u.Host 是一整个字符串 "host.com:5432"，主机和端口混在一起。要分开取，需要借助 net 包的 SplitHostPort 函数，它会处理 IPv6 地址的方括号格式（比如 [::1]:8080），自己手写 strings.Split(u.Host, ":") 碰到 IPv6 直接崩。

注意：如果 URL 里没有写端口，u.Host 就只是 "host.com"，这时候 SplitHostPort 会报错，port 会是空字符串。实际业务里要加判断，或者用 strings.Contains(u.Host, ":") 先探一下。

业务场景：微服务网关做 HTTP 重定向白名单，解析出 host 之后和允许列表对比，host 不在白名单里的重定向请求直接 403，防止开放重定向漏洞（Open Redirect）。

────────────────────────
第五步：u.Path 和 u.Fragment —— 路径和锚点
────────────────────────

fmt.Println(u.Path)     // /path
fmt.Println(u.Fragment) // f

Path 是 ? 之前、# 之前的那一段，Fragment 是 # 后面的部分。

值得注意的是：Fragment 在 HTTP 请求里是不会发送给服务器的，浏览器自己留着用于页面内跳转。所以如果你在服务端代码里解析 Fragment，往往是在处理前端路由字符串或者做本地 URL 分析，不是真正的 HTTP 请求片段。

────────────────────────
第六步：u.RawQuery + url.ParseQuery —— 查询参数
────────────────────────

fmt.Println(u.RawQuery) // k=v
m, _ := url.ParseQuery(u.RawQuery)
fmt.Println(m)          // map[k:[v]]
fmt.Println(m["k"][0]) // v

u.RawQuery 是 ? 后面的原始字符串，不做任何解码。url.ParseQuery 把它解析成 url.Values，本质是 map[string][]string，也就是一个键可以对应多个值（比如复选框 ?color=red&color=blue，m["color"] 就是 ["red", "blue"]）。

如果你只关心第一个值，直接 m["k"][0]；但如果这个键根本不存在，m["k"] 是 nil，m["k"][0] 会 panic。安全写法是先用 m.Get("k")，它返回第一个值或空字符串，不会 panic。

────────────────────────
第七步：反向操作 —— 构造 URL
────────────────────────

讲完拆，再讲拼。url.Values 提供了三个方法帮你构造查询串：

- Add(key, value)：追加一个值，同一个 key 可以有多个。
- Set(key, value)：覆盖，同一个 key 只保留这一个值。
- Encode()：把整个 map 编码成 k=v&k2=v2 格式，自动处理 URL 编码（空格变 %20，中文变 %E4%B8%AD%E6%96%87）。

拼完查询串，可以赋值回 u.RawQuery，再调用 u.String() 得到完整 URL 字符串：

params := url.Values{}
params.Set("page", "1")
params.Set("size", "20")
u.RawQuery = params.Encode()
fmt.Println(u.String())

或者直接用 url.URL 结构体字面量从零构建：

newURL := &url.URL{
    Scheme:   "https",
    Host:     "api.example.com",
    Path:     "/v1/users",
    RawQuery: params.Encode(),
}
fmt.Println(newURL.String()) // https://api.example.com/v1/users?page=1&size=20

────────────────────────
常见坑汇总
────────────────────────

1. 日志泄露密码：DSN 里带 user:pass 的 URL 千万别直接打日志，解析后把 u.User 置 nil 再 u.String()。

2. URL Encode 忘了处理：查询参数里有中文或特殊字符，手拼字符串不 encode，服务器解析出乱码。用 url.Values.Encode() 自动搞定。

3. SplitHostPort 没判断端口是否存在：URL 不带端口时直接调会拿到空 port，要做容错。

4. ParseQuery 结果直接下标访问：没确认 key 存在就 m["key"][0]，key 不存在时 panic，用 m.Get("key") 更安全。

5. Fragment 服务端收不到：别指望在服务器 Handler 里从 r.URL.Fragment 拿到锚点，浏览器根本不发。

────────────────────────
一句话总结
────────────────────────

url.Parse 把一条 URL 字符串变成结构化的 *url.URL，按字段取值；net.SplitHostPort 拆主机端口；url.ParseQuery 把查询串变 map；url.Values.Encode() 把 map 变查询串；u.String() 把改完的结构体再变回字符串。整套流程闭环，URL 操作从此不手写字符串分割。

---

### 反向拼接 URL

除了解析 URL，`net/url` 包同样支持**反向构造**完整 URL。只需填充 `url.URL` 结构体的各字段，再调用 `.String()` 方法即可得到规范的 URL 字符串：

```go
u := url.URL{
    Scheme:   "https",
    Host:     "a.b",
    Path:     "/x",
    RawQuery: "k=v",
}
fmt.Println(u.String()) // https://a.b/x?k=v
```

若查询参数有多个键值对，推荐使用 `url.Values` 来管理，避免手动拼接出错：

```go
params := url.Values{}
params.Add("page", "1")
params.Set("size", "20")
u.RawQuery = params.Encode() // page=1&size=20
```

`Add` 会追加同名键，`Set` 则会覆盖同名键，最后 `Encode()` 按键名字母序排列并完成百分号编码，输出可直接嵌入 URL 的安全字符串。

---

### URL 编码的常见坑

在手动拼接查询参数时，**编码方式的细节差异**容易踩坑。最典型的是空格的处理：`url.QueryEscape` 将空格编码为 `%20`，而 HTML 表单默认使用 `+` 表示空格（`application/x-www-form-urlencoded` 规范）。两者在不同后端框架中解码行为不同，混用会导致参数值错位。

中文字符同样需要编码。例如 `"北京"` 经 `url.QueryEscape` 后变为 `"%E5%8C%97%E4%BA%AC"`，若直接拼接未编码的中文到 URL 中，部分 HTTP 客户端或服务器会拒绝请求或产生乱码。正确做法是始终通过 `url.QueryEscape` 或 `url.Values.Encode()` 来处理用户输入：

```go
keyword := "Go 语言"
safe := url.QueryEscape(keyword) // Go+%E8%AF%AD%E8%A8%80
fmt.Println("https://search.example.com?q=" + safe)
```

注意 `url.PathEscape` 与 `url.QueryEscape` 适用场景不同：前者用于路径段（不编码 `/`），后者用于查询值（编码 `&`、`=` 等特殊字符），不要混用。

---

### 安全实践：防御开放重定向

在做登录后跳转或第三方回调时，常见的安全漏洞是**开放重定向（Open Redirect）**。开发者往往只校验重定向目标的 `Host` 是否在白名单内，却忽略了 `Scheme`，导致攻击者可以构造如下 URL 绕过校验：

```
https://your-site.com/login?next=javascript://your-site.com/%0aalert(1)
```

或者利用 `//evil.com` 这类协议相对 URL，在仅比较 Host 的情况下悄悄跳转到恶意站点。正确的白名单校验应**同时验证 Scheme 和 Host**：

```go
func isSafeRedirect(raw string) bool {
    u, err := url.Parse(raw)
    if err != nil {
        return false
    }
    // 必须同时限定 Scheme 和 Host
    return (u.Scheme == "https") && (u.Host == "trusted.example.com")
}
```

此外，对于站内跳转，最安全的方式是**只保留 Path 和 Query 部分**，彻底丢弃 Scheme 和 Host，从根本上杜绝跨域重定向的可能。

## 🏃 跑一下试试

逐行输出解释如下：
1. `postgres` —— u.Scheme，URL 协议头。
2. `user:pass` —— u.User 整体打印，格式为 username:password。
3. `user` —— u.User.Username() 单独取出用户名。
4. `pass` —— u.User.Password() 单独取出密码（第二个返回值 bool 表示密码是否存在）。
5. `host.com:5432` —— u.Host，主机名与端口合在一起。
6. `host.com` —— net.SplitHostPort 拆出主机名。
7. `5432` —— net.SplitHostPort 拆出端口号，类型是 string。
8. `/path` —— u.Path，注意保留了前导斜杠。
9. `f` —— u.Fragment，# 后面的片段，不含 # 本身。
10. `k=v` —— u.RawQuery，原始查询字符串，未解码。
11. `map[k:[v]]` —— url.ParseQuery 返回 url.Values，值是字符串切片。
12. `v` —— m["k"][0]，取第一个值。

## 💡 小贴士：
1. **Password() 双返回值**：u.User.Password() 返回 (string, bool)，bool 为 false 说明 URL 中根本没有设置密码，与空字符串密码是两种不同状态，别用 _ 忽略后者。
2. **RawQuery vs Query()**：u.RawQuery 是原始字符串（如 k=v%20x），不做任何解码；u.Query() 等价于 url.ParseQuery(u.RawQuery)，返回已 URL 解码的 url.Values，日常推荐用 Query()。
3. **明文密码别进日志**：含认证信息的 URL 直接 fmt.Println(u) 就会把密码打出来，生产环境应先 u.User = nil 或用 u.Redacted() 方法（Go 1.15+）脱敏后再记录。
4. **url.Parse 纯字符串操作**：它不会 DNS 解析，不建立连接，解析失败才会报错，因此可以放心在初始化阶段调用。
5. **相对路径合并**：url.URL.ResolveReference(ref) 能将相对 URL 相对于 base 解析为完整 URL，爬虫或跳转逻辑里非常实用。

## 🎓 知识点清单

- 理解 url.Parse 只做字符串解析，不发起任何网络请求
- u.User 是 *url.Userinfo 类型，可调用 .Username() 和 .Password() 方法拆分认证信息
- u.Host 含端口时用 net.SplitHostPort 拆分，不要手动 strings.Split(":")
- u.Path 保留前导斜杠 /path，u.Fragment 不含 # 符号本身
- u.RawQuery 是原始字符串 k=v；url.ParseQuery 返回 url.Values（即 map[string][]string）
- Values 的值是切片，多个同名参数会追加到同一个 key 下，取第一个用 [0]
- 生产代码中不要将含密码的 URL 直接写进日志，应脱敏后再输出
- url.URL.ResolveReference 可将相对 URL 解析为绝对 URL，处理页面跳转场景很方便

## ➡️ 下一关

搞定了 URL 的拆解，下一关来点「不可逆」的操作——第 60 关 **SHA256 散列（sha256-hashes）**。我们会用 crypto/sha256 对任意字符串生成固定长度的哈希摘要，一起看看 Go 标准库里的密码学工具箱。
