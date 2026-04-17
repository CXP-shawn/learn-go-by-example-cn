# 第 51 关：正则表达式（师兄带你学 Go）

🎯 这一关你会学到
• Go 的 regexp 包怎么用——MatchString、Compile、MustCompile
• 找匹配的两大家族：FindString 系列（首个） vs FindAllString 系列（全部）
• 捕获组（Submatch）和索引（Index）到底啥区别
• 正则替换：ReplaceAllString / ReplaceAllFunc
• 为啥生产代码都用 MustCompile + 全局变量

🤔 先想一个问题

你收到一封邮件，里面夹杂着三十多个手机号和邮箱地址——现在要把它们全挖出来。手工一个个找？要疯。这就是**正则表达式（regular expression，简称 regex）**派上用场的时候了——用一段"**模式串**"描述你要找的字符长啥样（比如"p 开头、中间全是小写字母、ch 结尾"），正则引擎一口气帮你扫完全文。Go 标准库的 regexp 包就是给这事儿准备的。

## 📖 看代码

```go
// Go 提供了内建的[正则表达式](http://zh.wikipedia.org/wiki/%E6%AD%A3%E5%88%99%E8%A1%A8%E8%BE%BE%E5%BC%8F)支持。
// 这儿有一些在 Go 中与 regexp 相关的常见用法示例。

package main

import (
	"bytes"
	"fmt"
	"regexp"
)

func main() {

	// 测试一个字符串是否符合一个表达式。
	match, _ := regexp.MatchString("p([a-z]+)ch", "peach")
	fmt.Println(match)

	// 上面我们是直接使用字符串，但是对于一些其他的正则任务，
	// 你需要通过 `Compile` 得到一个优化过的 `Regexp` 结构体。
	r, _ := regexp.Compile("p([a-z]+)ch")

	// 该结构体有很多方法。这是一个类似于我们前面看到的匹配测试。
	fmt.Println(r.MatchString("peach"))

	// 查找匹配的字符串。
	fmt.Println(r.FindString("peach punch"))

	// 这个也是查找首次匹配的字符串，
	// 但是它的返回值是，匹配开始和结束位置的索引，而不是匹配的内容。
	fmt.Println("idx:", r.FindStringIndex("peach punch"))

	// `Submatch` 返回完全匹配和局部匹配的字符串。
	// 例如，这里会返回匹配 `p([a-z]+)ch` 和 `([a-z]+)` 的信息。
	fmt.Println(r.FindStringSubmatch("peach punch"))

	// 类似的，这个会返回完全匹配和局部匹配位置的索引。
	fmt.Println(r.FindStringSubmatchIndex("peach punch"))

	// 带 `All` 的这些函数返回全部的匹配项，
	// 而不仅仅是首次匹配项。例如查找匹配表达式全部的项。
	fmt.Println(r.FindAllString("peach punch pinch", -1))

	// `All` 同样可以对应到上面的所有函数。
	fmt.Println("all:", r.FindAllStringSubmatchIndex(
		"peach punch pinch", -1))

	// 这些函数接收一个非负整数作为第二个参数，来限制匹配次数。
	fmt.Println(r.FindAllString("peach punch pinch", 2))

	// 上面的例子中，我们使用了字符串作为参数，
	// 并使用了 `MatchString` 之类的方法。
	// 我们也可以将 `String` 从函数名中去掉，并提供 `[]byte` 的参数。
	fmt.Println(r.Match([]byte("peach")))

	// 创建正则表达式的全局变量时，可以使用 `Compile` 的变体 `MustCompile` 。
	//  `MustCompile` 用 `panic` 代替返回一个错误 ，这样使用全局变量更加安全。
	r = regexp.MustCompile("p([a-z]+)ch")
	fmt.Println("regexp:", r)

	// `regexp` 包也可以用来将子字符串替换为为其它值。
	fmt.Println(r.ReplaceAllString("a peach", "<fruit>"))

	// `Func` 变体允许您使用给定的函数来转换匹配的文本。
	in := []byte("a peach")
	out := r.ReplaceAllFunc(in, bytes.ToUpper)
	fmt.Println(string(out))
}
```

🔍 师兄给你逐行拆

**先说正则表达式是啥**

**正则表达式**是一种用字符组合描述字符串模式的小语言。比如 `p([a-z]+)ch` 意思是"以 p 开头，中间一个或多个小写字母（`[a-z]+` 表示"a-z 中任意字符出现一次或多次"），ch 结尾"。`()` 是**捕获组**——把中间部分单独抓出来能事后取用。

Go 的 regexp 包用的是 **RE2 引擎**（Google 出的安全正则实现，保证线性时间复杂度，不会像 PCRE 那样被恶意正则搞到指数爆炸）。所以 Go 不支持某些回溯相关的高级特性（比如 `(?<=...)` 反向零宽断言），但够用、快、稳。

**第一段：直接 MatchString 判断**

```go
match, _ := regexp.MatchString("p([a-z]+)ch", "peach")
fmt.Println(match)  // true
```

`regexp.MatchString(pattern, str)` 直接判断 str 是否匹配 pattern，返回 (bool, error)。这种一次性检查不需要保存编译结果的场景最方便。`_` 是 **blank identifier**（空标识符），表示"我不关心这个返回值"。

**但要注意**：每次调 MatchString，Go 内部都会帮你 Compile 一次正则——重复调用性能差。所以下面的典型姿势是先 Compile 拿 *Regexp 对象，然后在对象上调各种方法。

**第二段：Compile 得到 *Regexp 对象**

```go
r, _ := regexp.Compile("p([a-z]+)ch")
fmt.Println(r.MatchString("peach"))  // true
```

`regexp.Compile(pattern)` 返回 (*Regexp, error)。*Regexp 是编译后的正则对象，上面挂了一大堆匹配/查找/替换方法。以后重复用就调方法，性能好很多。

**Find 家族：找第一个匹配**

```go
r.FindString("peach punch")               // "peach"（首次匹配的字符串）
r.FindStringIndex("peach punch")          // [0 5]（首次匹配的字节下标范围）
r.FindStringSubmatch("peach punch")       // [peach ea]（完整匹配 + 各捕获组）
r.FindStringSubmatchIndex("peach punch")  // [0 5 1 3]（完整匹配 + 各捕获组 的下标对）
```

- `FindString` 返回**第一个匹配到的字符串本身**
- `FindStringIndex` 返回 `[start, end]` **字节下标**（左闭右开），拿不到内容拿得到位置
- `FindStringSubmatch` 返回**一个切片**：第 0 个是完整匹配，后面每一个是**捕获组**。`p([a-z]+)ch` 有一个捕获组 `([a-z]+)`，所以返回 `["peach", "ea"]`
- `FindStringSubmatchIndex` 把上面那些都转成下标：`[0, 5, 1, 3]` 意思是完整匹配是 [0,5)，捕获组 1 是 [1,3)（"ea"）

**Find All 家族：找全部**

```go
r.FindAllString("peach punch pinch", -1)           // [peach punch pinch]
r.FindAllStringSubmatchIndex("peach punch pinch", -1)  // 所有匹配的 index 数组集合
r.FindAllString("peach punch pinch", 2)            // [peach punch] 只取前 2 个
```

每个 Find 方法都有一个对应的 `FindAll...` 版本，加一个整数参数限制数量——**-1 表示不限，取全部**；正整数表示最多取多少个。

**Match：接受 []byte**

```go
r.Match([]byte("peach"))  // true
```

把方法名里的 "String" 去掉，就是对应的 `[]byte` 版本——处理网络数据、文件字节流时更高效，不用多做一次 string 转换（**string 和 []byte 的定义**：string 在 Go 里是**不可变字节序列**；[]byte 是**可变的字节切片**）。

**MustCompile：初始化时直接 panic**

```go
r = regexp.MustCompile("p([a-z]+)ch")
fmt.Println("regexp:", r)  // regexp: p([a-z]+)ch
```

跟前面学过的 template.Must 一个味儿——`MustCompile` 如果正则语法错直接 panic，适合**全局变量初始化**。生产代码里几乎都是：

```go
var emailRegexp = regexp.MustCompile(`[\w.+-]+@[\w-]+\.[\w.-]+`)
```

放在包级别，程序启动时编一次，之后无限复用。

**替换：ReplaceAllString 和 ReplaceAllFunc**

```go
r.ReplaceAllString("a peach", "<fruit>")  // "a <fruit>"
```

把所有匹配到的地方替换成指定字符串。这个 API 在替换模板里还支持 `$1`、`$2` 这种引用捕获组的语法（本例没演示）。

```go
in := []byte("a peach")
out := r.ReplaceAllFunc(in, bytes.ToUpper)
fmt.Println(string(out))  // "a PEACH"
```

`ReplaceAllFunc` 更灵活——**每个匹配到的字节片段喂给一个函数，函数返回啥就替换成啥**。这里用 `bytes.ToUpper` 把匹配到的 "peach" 转大写，结果 "a PEACH"。这种 API 在处理复杂替换（比如匹配到日期要格式化、匹配到金额要千位分隔）时无敌好用。

🏃 跑一下试试

保存为 regular-expressions.go，然后：

```bash
go run regular-expressions.go
```

预期输出：

```text
true
true
peach
idx: [0 5]
[peach ea]
[0 5 1 3]
[peach punch pinch]
all: [[0 5 1 3] [6 11 7 9] [12 17 13 15]]
[peach punch]
true
regexp: p([a-z]+)ch
a <fruit>
a PEACH
```

重点看几行：
- 第 1-2 行 true：MatchString / r.MatchString 都确认 "peach" 匹配
- `peach` / `idx: [0 5]`：首个匹配的字符串和下标
- `[peach ea]`：Submatch 返回 [完整匹配, 捕获组1]
- `[0 5 1 3]`：SubmatchIndex 同上但全转下标
- `all:` 那行三组下标分别对应 "peach"(0,5)、"punch"(6,11)、"pinch"(12,17) 及其捕获组
- `[peach punch]`：限量 2 个
- `a <fruit>`：ReplaceAllString 把 "peach" 换成了 "<fruit>"
- `a PEACH`：ReplaceAllFunc 把 "peach" 交给 bytes.ToUpper 转成大写

💡 师兄的碎碎念

师兄的正则生存指南：

1. **生产代码永远 MustCompile + 全局变量**。在函数里反复 Compile 是常见反模式，每次匹配都要重新解析正则 AST，性能下降一个数量级。

2. **RE2 不支持回溯相关**，比如反向断言 `(?<=abc)`、反向引用 `\1`。要这些特性得用 `github.com/dlclark/regexp2` 第三方库。

3. **正则不是万能钥匙**。HTML / XML / JSON 这种结构化数据别用正则硬啃，用专门的解析库。经典梗"用正则解析 HTML 会召唤古神"不是开玩笑。

4. **可读性**：复杂正则用 ``` 反引号原生字符串``` 包起来，加注释 `(?P<year>\d{4})` 命名捕获组，比一堆 `\\w\\d` 不知道清楚多少。

5. **字节 vs 字符** 又出现了：regexp 默认按 UTF-8 字节处理，输入中文正则也能工作，但 Index 类返回的是**字节下标**，不是 rune 下标。

吐槽一句：Go 的 regexp 方法名一大堆，Find / Match / Replace 再组合 String / Index / Submatch / All ——光看名字就能排列组合出十几个。但熟了之后发现规律性很强：名字越长、返回的信息越细。

🎓 知识点清单

- Go 的 regexp 包底层是 RE2 引擎，保证线性时间复杂度
- 一次性匹配用 regexp.MatchString；重复使用先 Compile 拿 *Regexp
- MustCompile 适合全局变量初始化，失败直接 panic
- Find 系列 = 首个匹配，FindAll 系列 = 全部匹配（-1 不限）
- Submatch 返回 [完整匹配, 捕获组1, 捕获组2, ...]
- Index 版本返回字节下标对，而不是内容
- 方法名带 String 处理 string，不带处理 []byte
- ReplaceAllString 做普通替换；ReplaceAllFunc 让你用函数动态决定替换内容
- 日常记 pattern 用反引号字符串（避免反斜杠转义地狱）

➡️ 下一关

会解析字符串后，咱们该学**跟外部世界交换数据**了——下一关 52「JSON」教你用 Go 的 encoding/json 做序列化和反序列化，API 接口对接全靠它，走起！
