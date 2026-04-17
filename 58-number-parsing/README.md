# 第 58 关：数字解析（师兄带你学 Go）

## 🎯 这一关你会学到

学会用 strconv 包把字符串解析成数字——ParseFloat、ParseInt、ParseUint、Atoi 各有适用场景，搞清楚三参数的含义，顺便知道错误长啥样、忽略错误有多危险。

## 🤔 先想一个问题

你用 HTTP 表单提交了一个年龄字段，后端拿到的是字符串 "25"，想做 age + 1，直接写 "25" + 1？编译报错！从 CSV 里读到 "3.14"，从 .env 配置读到 "8080"，都是同一个问题——字符串不是数字，必须

## 📖 看代码

```go
// 从字符串中解析数字在很多程序中是一个基础常见的任务，
// 而在 Go 中，是这样处理的。

package main

// 内建的 `strconv` 包提供了数字解析能力。
import (
	"fmt"
	"strconv"
)

func main() {

	// 使用 `ParseFloat`，这里的 `64` 表示解析的数的位数。
	f, _ := strconv.ParseFloat("1.234", 64)
	fmt.Println(f)

	// 在使用 `ParseInt` 解析整型数时，
	// 例子中的参数 `0` 表示自动推断字符串所表示的数字的进制。
	// `64` 表示返回的整型数是以 64 位存储的。
	i, _ := strconv.ParseInt("123", 0, 64)
	fmt.Println(i)

	// `ParseInt` 会自动识别出字符串是十六进制数。
	d, _ := strconv.ParseInt("0x1c8", 0, 64)
	fmt.Println(d)

	// `ParseUint` 也是可用的。
	u, _ := strconv.ParseUint("789", 0, 64)
	fmt.Println(u)

	// `Atoi` 是一个基础的 10 进制整型数转换函数。
	k, _ := strconv.Atoi("135")
	fmt.Println(k)

	// 在输入错误时，解析函数会返回一个错误。
	_, e := strconv.Atoi("wat")
	fmt.Println(e)
}
```

## 🔍 师兄给你逐行拆

好，我们今天的主角是 strconv 包，全称 string conversion，字符串转换。这个包里有一堆 Parse 开头的函数，专门干一件事：把字符串变成你想要的数字类型。听起来简单，但坑不少，跟我一起过一遍。

**第一个：ParseFloat**

```go
f, _ := strconv.ParseFloat("1.234", 64)
```

两个参数：第一个是字符串，第二个是 bitSize，也就是你希望返回的浮点数精度是多少位。写 64 就是 float64，写 32 就是 float32——但注意，返回值类型永远是 float64，你传 32 只是告诉它「按 float32 的精度范围来校验」，如果你真的要 float32，还得自己手动转 float32(f)。

实际场景：你从配置文件里读到一行 "discount=0.85"，split 完拿到 "0.85" 这个字符串，想做打折计算，就得先 ParseFloat 一下。不然你拿着字符串根本没法乘以价格。

**第二个：ParseInt，重点来了**

```go
i, _ := strconv.ParseInt("123", 0, 64)
```

三个参数，一个一个说：

- 第一个：要解析的字符串，没啥好说的。
- 第二个：base，进制。写 10 就是十进制，写 16 就是十六进制，写 2 就是二进制。**写 0 是重头戏**——表示让 Go 自己看字符串前缀来判断进制：0x 开头就是十六进制，0b 开头是二进制，0o 开头是八进制，没有前缀就当十进制处理。
- 第三个：bitSize，表示你期望的整数位数，可以是 0、8、16、32、64。写 64 就是按 int64 范围来检验，超出范围会报 ErrRange 错误（下面会讲）。写 0 比较特殊，表示跟当前平台的 int 一样大——32 位系统就是 32 位，64 位系统就是 64 位。

**第三个：自动识别十六进制**

```go
d, _ := strconv.ParseInt("0x1c8", 0, 64)
fmt.Println(d) // 输出 456
```

这里 base 传 0，Go 看到 0x 前缀，自动按十六进制解析。0x1c8 换算一下：1×256 + 12×16 + 8 = 256 + 192 + 8 = 456，没问题。

这个功能很实用，比如你读环境变量或者配置文件，用户可能写 "255" 也可能写 "0xFF"，你传 base=0 就都能正确解析，不用自己去判断字符串格式。

**第四个：ParseUint**

```go
u, _ := strconv.ParseUint("789", 0, 64)
```

和 ParseInt 用法一模一样，区别就是：它不接受负号。你传 "-1" 进去，直接报错，返回 0 和 error。

什么时候用 ParseUint？表示端口号（0-65535）、用户 ID、文件大小这类「不可能是负数」的场景。用 Uint 语义上更明确，而且能多表示一倍的正数范围——uint64 最大能到 18446744073709551615，而 int64 只到 9223372036854775807。

实际例子：你从环境变量读端口号：

```go
portStr := os.Getenv("PORT") // 得到 "8080"
port, err := strconv.ParseUint(portStr, 10, 16) // 16位，端口最大65535
if err != nil {
    log.Fatal("端口号不合法:", err)
}
```

用 16 位 bitSize 是因为端口号最大 65535，正好 uint16 范围，超出这个范围 ParseUint 就会返回 ErrRange。

**第五个：Atoi，最常用的快捷方式**

```go
k, _ := strconv.Atoi("135")
```

Atoi 是 ASCII to Integer 的缩写，等价于 `ParseInt(s, 10, 0)`——十进制，平台原生 int 大小。返回的是 int，不是 int64。

这里有个跨平台要注意的点：Atoi 返回 int，在 32 位系统上是 int32，在 64 位系统上是 int64。如果你的代码有可能跑在 32 位设备上（比如某些嵌入式或老系统），而你解析的数可能超过 21 亿，那就别用 Atoi，老老实实用 ParseInt(..., 10, 64)，拿 int64。

日常用途：表单里的数字字段、URL 参数里的 ID 解析，大多数情况 Atoi 就够了，简洁。

**第六个：错误处理，新手最大的坑**

```go
_, e := strconv.Atoi("wat")
fmt.Println(e)
// 输出：strconv.Atoi: parsing "wat": invalid syntax
```

这个错误信息格式是：`函数名: parsing "输入值": 错误原因`。错误原因主要两种：
- `invalid syntax`：字符串根本不是数字格式
- `value out of range`：格式对但数值超出 bitSize 指定的范围，对应 `strconv.ErrRange`

你可以这样区分错误类型：

```go
if numErr, ok := err.(*strconv.NumError); ok {
    if numErr.Err == strconv.ErrRange {
        fmt.Println("数字超范围了")
    } else if numErr.Err == strconv.ErrSyntax {
        fmt.Println("字符串格式不对")
    }
}
```

**重点吐槽来了**：示例代码里到处都是 `_, _` 忽略错误。这是教学代码为了简洁，**生产环境千万别学**！现实中 Atoi("wat") 你不处理错误，得到的是 0，然后你拿着这个 0 去数据库查 id=0，轻则查出奇怪数据，重则删库。解析失败默默返回零值是 Go 的设计，不是保护你的机制，保护你的是你自己写的 err != nil 判断。

**第七个：Format 系列，反向操作顺带一提**

既然有 Parse，就有对应的 Format，把数字变回字符串：

- `strconv.Itoa(123)` → "123"，Atoi 的反向
- `strconv.FormatInt(456, 16)` → "1c8"，转成十六进制字符串
- `strconv.FormatFloat(3.14, 'f', 2, 64)` → "3.14"，指定格式和精度

Format 和 Parse 是一对，数字和字符串互转，靠这两组函数就搞定了。

**总结一张思维导图**：

- 要解析浮点数 → ParseFloat(s, 64)
- 要解析整数，可能有 0x/0b 前缀 → ParseInt(s, 0, 64)
- 要解析无符号整数（端口、ID）→ ParseUint(s, 10, 64)
- 就是普通十进制 int，懒得写长的 → Atoi(s)
- 无论用哪个，**必须检查 error**，别学示例代码里的 `_`。

**真实事故：端口去哪了**

某同学写服务启动代码时，用 `os.Getenv("PORT")` 读取端口号，然后直接 `port, _ := strconv.Atoi(os.Getenv("PORT"))` 把错误丢掉了。本地联调时环境变量没设，`Atoi` 拿到空字符串，返回 `0` 且 error 被忽略。`net.Listen("tcp", fmt.Sprintf(":%d", port))` 于是监听了 `:0`——这在 Linux 上是合法的，内核会随机分配一个可用端口。服务看似正常启动，但没人知道它究竟跑在哪个端口。同学排查了两小时，最后靠 `ss -tlnp` 才发现真相。教训只有一条：永远不要丢掉 `strconv` 的 error。

**Format 家族：数字变回字符串**

解析是把字符串变数字，反向操作同样由 `strconv` 包承包。`strconv.Itoa(42)` 直接返回 `"42"`，是 `Atoi` 的镜像，最常用。需要指定进制时用 `FormatInt`：`strconv.FormatInt(456, 16)` 返回 `"1c8"`，和第 58 关的 `0x1c8` 遥相呼应。浮点数则用 `FormatFloat`：`strconv.FormatFloat(3.14159, 'f', 2, 64)` 返回 `"3.14"`，第三个参数控制小数位数，第四个参数与 `ParseFloat` 的 `bitSize` 语义一致。Format 家族和 Parse 家族合在一起，构成了 Go 数字与字符串互转的完整闭环。

**fmt.Sscanf 的替代路径**

对于简单场景，`fmt.Sscanf("123", "%d", &i)` 也能把字符串解析成整数，写法直观，熟悉 C 的同学上手即会。但选择时有几条原则：若只需解析单个数字，优先用 `strconv`，因为它性能更好、错误信息更精确，能区分「溢出」和「非法字符」。若输入是带固定格式的混合字符串，比如 `"x=42, y=7"`，`fmt.Sscanf(s, "x=%d, y=%d", &x, &y)` 一行搞定，比手动切割再 `ParseInt` 简洁得多。总结：纯数字解析用 `strconv`，带格式模板的混合解析用 `fmt.Sscanf`，按场景选工具，不要教条。

## 🏃 跑一下试试

执行 go run number-parsing.go，你会看到六行输出：

1. 1.234 —— ParseFloat("1.234", 64) 直接解析为 float64，原样打印。
2. 123 —— ParseInt("123", 0, 64)，字符串无前缀，自动推断为十进制，结果 123。
3. 456 —— ParseInt("0x1c8", 0, 64)，0x 前缀触发十六进制模式：1×16²=256，c=12，12×16=192，8×1=8，合计 256+192+8=456。
4. 789 —— ParseUint("789", 0, 64)，无符号十进制，直接 789。
5. 135 —— Atoi("135") 是 ParseInt(s,10,0) 的快捷版，返回平台原生 int，值 135。
6. strconv.Atoi: parsing "wat": invalid syntax —— "wat" 不是合法数字，返回 *strconv.NumError，fmt.Println 自动调用其 Error() 方法打印错误描述。

## 💡 碎碎念

① 别忽略 error：代码里 _, _ 是教学用的偷懒写法，生产代码每次解析都要检查 err，尤其来自用户输入或外部接口的字符串，一个「0x」前缀就能让你的逻辑走岔。

② 对外参数做上下界校验：ParseInt 本身不管业务语义，解析成功不代表值合理。年龄传来个 -999 或 99999，语法上没错，但你的数据库会很受伤，记得在解析后加 range check。

③ Atoi 的 int 宽度随平台变：32 位系统 int 是 32 位，64 位系统是 64 位。对精度有要求的场景，显式用 ParseInt(s, 10, 32) 或 ParseInt(s, 10, 64) 更安全，别依赖 Atoi 的隐式宽度。

④ 严格防溢出：ParseInt("2147483648", 10, 32) 会返回 ErrRange，正好可以用来做边界拦截，比自己写比较判断更地道。

⑤ JSON 大整数陷阱：json.Unmarshal 默认把数字解析为 float64，超过 2⁵³ 的整数会丢精度。遇到雪花 ID 这类大整数字段，记得用 json.Number 或先解成字符串再 ParseInt，这是踩过才懂的坑。

## 🎓 知识点清单

- 运行 go run number-parsing.go，确认六行输出与预期一致
- 理解 ParseFloat 第二参数 64 代表 float64 精度，32 则为 float32
- 理解 ParseInt 进制参数：0 = 自动推断，8 = 八进制，10 = 十进制，16 = 十六进制
- 手算验证 0x1c8：1×256 + 12×16 + 8 = 256 + 192 + 8 = 456 ✓
- 理解 Atoi 等价于 ParseInt(s, 10, 0)，返回的 int 宽度随平台变化
- 故意传入非法字符串（如 "3.14" 给 Atoi），观察 error 类型 *strconv.NumError 的 Func/Num/Err 字段
- 尝试 ParseInt("256", 10, 8)，观察溢出时返回的 ErrRange 错误
- 对比 ParseUint 与 ParseInt 在负数输入时的行为差异

## ➡️ 下一关

数字能解析，URL 当然也能拆。下一关「URL 解析 (url-parsing)」我们用 net/url 把一条完整的 URL 大卸八块——Scheme、Host、Path、Query 参数逐一取出，顺便摸清 Go 标准库里 URL 结构体的字段布局，处理 Web 请求时这招非常实用，师弟走起 👇
