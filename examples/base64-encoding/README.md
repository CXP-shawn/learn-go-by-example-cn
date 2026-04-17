# 第 61 关：Base64 编码（师兄带你学 Go）

## 🎯 这一关你会学到

学会用 Go 标准库 encoding/base64 对字符串进行标准（StdEncoding）和 URL 安全（URLEncoding）两种 Base64 编解码，掌握 EncodeToString、DecodeString 的正确姿势，理解两种编码表的字符差异与适用场景。

## 🤔 先想一个问题

你有没有见过 JWT token 长这个样子：eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyIn0.xxxxx？三段用点分开，每段都是 Base64。还有 HTML 里把图片直接塞进 src 属性、邮件附件 MIME 编码、把二进制数据安全放进 JSON 字段——这些地方全都是 Base64 在背后干活。今天师兄带你把这个用得炉火纯青。

## 📖 看代码

```go
// Go 提供了对 [base64 编解码](http://zh.wikipedia.org/wiki/Base64)的内建支持。

package main

// 这个语法引入了 `encoding/base64` 包，
// 并使用别名 `b64` 代替默认的 `base64`。这样可以节省点空间。
import (
	b64 "encoding/base64"
	"fmt"
)

func main() {

	// 这是要编解码的字符串。
	data := "abc123!?$*&()'-=@~"

	// Go 同时支持标准 base64 以及 URL 兼容 base64。
	// 这是使用标准编码器进行编码的方法。
	// 编码器需要一个 `[]byte`，因此我们将 string 转换为该类型。
	sEnc := b64.StdEncoding.EncodeToString([]byte(data))
	fmt.Println(sEnc)

	// 解码可能会返回错误，如果不确定输入信息格式是否正确，
	// 那么，你就需要进行错误检查了。
	sDec, _ := b64.StdEncoding.DecodeString(sEnc)
	fmt.Println(string(sDec))
	fmt.Println()

	// 使用 URL base64 格式进行编解码。
	uEnc := b64.URLEncoding.EncodeToString([]byte(data))
	fmt.Println(uEnc)
	uDec, _ := b64.URLEncoding.DecodeString(uEnc)
	fmt.Println(string(uDec))
}
```

## 🔍 师兄给你逐行拆

甲、import 别名这个小技巧先说清楚

代码一开头就出现了一个不常见的写法：

```go
import (
    b64 "encoding/base64"
    "fmt"
)
```

这叫做 import 别名（import alias）。正常情况下引入 encoding/base64 之后，你要写 base64.StdEncoding，六个字母加一个点。用别名 b64 之后，写 b64.StdEncoding，省了四个字符。代码量大的时候这个习惯挺好，尤其是包名很长的时候。别名可以是任意合法标识符，甚至可以用 _ 丢弃（不导入符号）或者用 . 把所有符号打散到当前命名空间（不推荐）。这里用 b64 是最实用的写法，师兄建议你在项目里遇到长包名时都可以这样做。

乙、Base64 的本质：从比特位开始理解

Base64 的原理非常简单，但很多人背结论不懂原理，用起来就容易踩坑。

计算机里任何数据最终都是字节，一个字节 8 位。Base64 的思路是：把每 3 个字节（共 24 位）拆成 4 组，每组 6 位，6 位最多表示 64 个不同的值（2 的 6 次方 = 64），然后用一张固定的 64 字符表来映射这 64 个值，这样就把任意二进制数据变成了纯可打印字符。

标准 Base64 字符表（RFC 4648）：A-Z（26个）、a-z（26个）、0-9（10个）、+、/，共 64 个字符。

那如果原始数据字节数不是 3 的倍数怎么办？补 0 凑够，然后在末尾加 = 号作为 padding。1 个多余字节补 2 个 =，2 个多余字节补 1 个 =。

URL 安全的 Base64（Base64URL）：把 + 换成 -，把 / 换成 _。原因是 + 在 URL query string 里会被解释成空格，/ 是路径分隔符，这两个字符放在 URL 里会出问题，所以换掉。

丙、EncodeToString：把字符串编码成 Base64

```go
data := "abc123!?$*&()'-=@~"
sEnc := b64.StdEncoding.EncodeToString([]byte(data))
fmt.Println(sEnc)
```

注意 EncodeToString 接收的参数是 []byte，不是 string。Go 里 string 和 []byte 之间转换非常廉价，直接 []byte(data) 就搞定。函数返回一个 string，就是 Base64 编码结果，直接可以打印、存数据库、放 JSON 字段、拼 URL。

对于输入 "abc123!?$*&()'-=@~"，标准编码的结果是 YWJjMTIzIT8kKiYoKSctPUB+，你会发现输出里出现了 + 和 / 这样的字符（具体取决于输入内容的比特位分布）。

丁、DecodeString：解码回原始数据

```go
sDec, _ := b64.StdEncoding.DecodeString(sEnc)
fmt.Println(string(sDec))
```

DecodeString 返回两个值：[]byte 和 error。代码里用 _ 忽略了 error，这在示例代码里没问题，但生产代码里一定要检查 error。什么情况下会出错？输入的 Base64 字符串不合法：比如包含了字符表之外的字符、padding = 号数量不对、字符串长度不是 4 的倍数（对于标准编码）。解码结果是 []byte，要还原成字符串就 string(sDec) 转一下，就能看到原始的 "abc123!?$*&()'-=@~" 了。

戊、URLEncoding：专为 URL 和 JWT 设计

```go
uEnc := b64.URLEncoding.EncodeToString([]byte(data))
fmt.Println(uEnc)
uDec, _ := b64.URLEncoding.DecodeString(uEnc)
fmt.Println(string(uDec))
```

用法和 StdEncoding 完全一样，只是换了编码器对象。URLEncoding 生成的字符串里，凡是标准编码里出现 + 的地方都变成 -，出现 / 的地方都变成 _，其他字符和 padding 规则完全一样。对于本例的输入，你可以把两个输出对比一下，差异就在这两个特殊字符上。

己、Base64 不是加密，这一点必须强调

这是师兄见过最多的误解。Base64 只是编码（encoding），不是加密（encryption）。它是完全公开的算法，任何人拿到 Base64 字符串都可以立即解码，不需要密钥，不需要任何秘密信息。JWT 的 header 和 payload 就是 Base64URL 编码，直接可以解码看内容，JWT 的安全性靠的是第三段 signature 的签名验证，而不是 Base64 本身。所以千万不要用 Base64 来「隐藏」密码或敏感信息，那等于裸奔。

庚、常见坑，师兄帮你提前踩

第一个坑：RawStdEncoding 和 RawURLEncoding。Go 标准库里还有这两个变体，Raw 的意思是不加 padding，也就是末尾没有 = 号。JWT 规范要求 Base64URL 不带 padding，所以实现 JWT 时要用 b64.RawURLEncoding，用了 URLEncoding 带了 = 就不符合规范了，对接其他系统时会踩坑。

第二个坑：= 号的兼容问题。标准 Base64 要求 padding，但很多系统传输时把 = 号丢了（因为 = 在 URL 里是参数分隔符）。你收到一个没有 = 号的 Base64 字符串，用 StdEncoding 解码会报错，这时要么用 RawStdEncoding，要么手动补 = 号。

第三个坑：MIME 编码的 76 列换行。邮件协议里的 Base64 每 76 个字符要插一个换行符，Go 标准库不自动处理这个，如果你在做邮件相关的东西要注意手动处理或者用 mime/quotedprintable 包。

辛、实际使用场景盘点

JWT（JSON Web Token）：header.payload.signature 三段，前两段是 Base64URL 编码的 JSON，第三段是签名。Go 里手写 JWT 就用 b64.RawURLEncoding。

Data URI：HTML 里可以把图片直接内嵌，格式是 data:image/png;base64,iVBORw0KGgo...，这样图片不需要单独请求，适合小图标。用 b64.StdEncoding.EncodeToString 读取图片字节然后拼字符串就能生成。

邮件附件：MIME 协议用 Base64 把二进制附件编码成纯文本，这样邮件服务器就能处理了。

二进制数据存 JSON：JSON 本身只支持文本，如果你想把图片、音频、证书等二进制数据放进 JSON 字段安全传输，把二进制 Base64 编码成字符串放进去，接收方再解码，是最通用的做法。

壬、对照本例的输出差异

输入 "abc123!?$*&()'-=@~" 同时用两种编码，标准编码结果和 URL 编码结果大部分字符相同，差异仅在 + 变 - 和 / 变 _ 这两处。你可以自己跑一下代码，然后把两行输出做字符级别的对比，直观感受两种编码的区别就在字符表的这两个位置。这也提醒你：StdEncoding 和 URLEncoding 的结果不能混用，用哪个编码就要用哪个解码，否则解码会出错或者得到错误结果。

总结一句话：Base64 是把任意字节变成安全可打印字符的标准手段，Go 的 encoding/base64 用起来极简单，记住标准 vs URL-safe 的字符表差异、有无 padding 的 Raw 变体、以及它只是编码不是加密，你就掌握了这个知识点的精髓。

## 🏃 跑一下试试

跑 `go run base64-encoding.go`。

第 1 行：`encoding/base64` 包直接内置，无需第三方依赖。
第 2 行：`sEnc := base64.StdEncoding.EncodeToString([]byte(data))` 把字符串转 `[]byte` 再做标准 Base64 编码，输出 `YWJjMTIzIT8kKiYoKSctPUB+`，其中 `+` 和 `/` 是标准字母表的合法字符。
第 3 行：`StdEncoding.DecodeString(sEnc)` 解码回原始 `[]byte`，再转 `string` 打印。
第 4 行：`base64.URLEncoding.EncodeToString` 使用 URL-safe 字母表，把 `+` 换成 `-`、`/` 换成 `_`，输出 `YWJjMTIzIT8kKiYoKSctPUB-`，这样结果可以安全嵌进 URL 或 HTTP Header。
第 5 行：同样用 `URLEncoding.DecodeString` 解码，还原原字符串。两路解码结果相同，说明只是字母表不同，信息完全等价。

## 💡 师兄的碎碎念

💡 Base64 速记

① **编码 ≠ 加密**：Base64 只是把二进制转成可打印 ASCII，任何人都能一键解码，别用它保护敏感数据。

② **大小膨胀约 33%**：每 3 字节原始数据变成 4 个字符，所以 `encoded 长度 = ceil(n/3) × 4`。18 字节输入 → 24 字符输出，刚好整除不需要补 `=`。

③ **Padding `=`**：字节数不是 3 的倍数时末尾补 `=`（1 或 2 个）。`base64.RawStdEncoding` / `RawURLEncoding` 省掉 padding，常见于 JWT Header/Payload 段。

④ **URL-safe 的必要性**：标准编码里的 `+` 在 URL 中会被解析成空格，`/` 会被当成路径分隔符，所以 JWT、OAuth token、URL 参数一律用 URLEncoding。

⑤ **用途清单**：图片嵌 HTML（data URI）、HTTP Basic Auth、JSON 传二进制字段、TLS 证书（PEM）……只要需要

## 🎓 知识点清单

- 新建 base64-encoding.go 并粘贴源码
- go run base64-encoding.go 跑通，无报错
- 观察标准编码输出含 + / 字符（YWJjMTIzIT8kKiYoKSctPUB+）
- 观察 URL 编码输出，+ → - 、/ → _（YWJjMTIzIT8kKiYoKSctPUB-）
- 解码两路均还原出原字符串 abc123!?$*&()'-=@~
- 尝试修改输入字符串，验证长度变化约为原来 4/3
- 故意传错误 base64 字符串给 DecodeString，确认 err 不为 nil
- 理解 StdEncoding vs URLEncoding 的使用场景区别

## ➡️ 下一关

搞定 Base64，下一关是 **读文件（reading-files）**。我们将用 os.Open + bufio.Scanner 逐行读取文件，以及用 os.ReadFile 一次性把整个文件读进内存——Go 处理本地文件的两把利器，准备好了吗？🗂️
