# 第 48 关：字符串函数（师兄带你学 Go）

🎯 这一关你会学到
• strings 包提供的常用字符串操作函数一次性看个遍
• 包别名 import："strings" 用 s 作为短别名
• var p = fmt.Println 这种"函数值当快捷键"的写法
• len 和按索引 [i] 取值是**字节级**操作，多字节字符要另外处理

🤔 先想一个问题

你平时用 Excel 处理姓名，"张三"要拆成姓和名、"order-2024-001"要按中划线分段、"小红书账号"要统一转小写——这些操作在编程里也天天要做。Go 标准库里的 strings 包就像一把瑞士军刀，Contains、Count、HasPrefix、Split、Join、Replace、ToLower/ToUpper……该有的都有。师兄今天带你把这把军刀的每个小工具挨个展示一遍，看完你基本就能应付 80% 的字符串处理需求。

## 📖 看代码

```go
// 标准库的 `strings` 包提供了很多有用的字符串相关的函数。
// 这儿有一些用来让你对 `strings` 包有一个初步了解的例子。

package main

import (
	"fmt"
	s "strings"
)

// 我们给 `fmt.Println` 一个较短的别名，
// 因为我们随后会大量的使用它。
var p = fmt.Println

func main() {

	// 这是一些 `strings` 中有用的函数例子。
	// 由于它们都是包的函数，而不是字符串对象自身的方法，
	// 这意味着我们需要在调用函数时，将字符串作为第一个参数进行传递。
	// 你可以在 [`strings`](http://golang.org/pkg/strings/) 包文档中找到更多的函数。
	p("Contains:  ", s.Contains("test", "es"))
	p("Count:     ", s.Count("test", "t"))
	p("HasPrefix: ", s.HasPrefix("test", "te"))
	p("HasSuffix: ", s.HasSuffix("test", "st"))
	p("Index:     ", s.Index("test", "e"))
	p("Join:      ", s.Join([]string{"a", "b"}, "-"))
	p("Repeat:    ", s.Repeat("a", 5))
	p("Replace:   ", s.Replace("foo", "o", "0", -1))
	p("Replace:   ", s.Replace("foo", "o", "0", 1))
	p("Split:     ", s.Split("a-b-c-d-e", "-"))
	p("ToLower:   ", s.ToLower("TEST"))
	p("ToUpper:   ", s.ToUpper("test"))
	p()

	// 虽然不是 `strings` 的函数，但仍然值得一提的是，
	// 获取字符串长度（以字节为单位）以及通过索引获取一个字节的机制。
	p("Len: ", len("hello"))
	p("Char:", "hello"[1])
}

// 注意，上面的 `len` 以及索引工作在字节级别上。
// Go 使用 UTF-8 编码字符串，因此通常按原样使用。
// 如果您可能使用多字节的字符，则需要使用可识别编码的操作。
// 详情请参考 [strings, bytes, runes and characters in Go](https://blog.golang.org/strings)。
```

🔍 师兄给你逐行拆

**两个"小技巧"先说一下**

代码开头有两个非常 Go 风格的小窍门，咱们先讲明白。

第一个是 `s "strings"`——**包别名（package alias）**。正常写法是 `import "strings"` 然后调用时写 `strings.Contains(...)`。但是这个关里咱们要调一堆 strings 包的函数，每次都写 "strings." 太长。所以 Go 允许给 import 起个短别名，`s "strings"` 意思是"把 strings 包在这个文件里叫 s"，后面就可以写 `s.Contains(...)`，干净很多。注意这个 s 是**局部别名**，只在这个文件里生效。

第二个是 `var p = fmt.Println`——**函数值赋给变量**。在 Go 里，函数是一等公民，可以像值一样赋给变量。p 现在就是 fmt.Println 的一个快捷方式，调 `p(...)` 等于调 `fmt.Println(...)`。这样后面每一行都能省六个字符，写起来爽。但注意：这是"风格问题"而非必须，大项目里一般不推荐这么干，会让代码可读性变差——只在这种 demo 场景用用爽快一下。

**Contains / Count / HasPrefix / HasSuffix：四个"判断类"函数**

- `s.Contains("test", "es")` —— 判断字符串 "test" 里是不是包含 "es"，返回 **true**（bool 类型：Go 的布尔类型，值是 true 或 false）
- `s.Count("test", "t")` —— 数 "test" 里有几个 "t"，返回 **2**（开头一个末尾一个）
- `s.HasPrefix("test", "te")` —— 是不是以 "te" 开头，**true**
- `s.HasSuffix("test", "st")` —— 是不是以 "st" 结尾，**true**

这四个函数的共同特点是**返回布尔值**，常用在 if 条件里。比如检查文件名后缀：`if strings.HasSuffix(filename, ".go") { ... }`。

**Index：找第一次出现的位置**

`s.Index("test", "e")` 返回 **1** —— "e" 在 "test" 里第一次出现的字节位置是下标 1（从 0 开始）。找不到返回 **-1**。师兄提醒：返回的是**字节偏移**，不是字符偏移，后面会专门讲这个坑。

**Join / Repeat / Replace：三个"生成类"函数**

- `s.Join([]string{"a", "b"}, "-")` —— 用 "-" 把 ["a", "b"] 连起来，得到 **"a-b"**。这是 Python 用户最熟悉的一个，相当于 "-".join(["a", "b"])
- `s.Repeat("a", 5)` —— 重复 "a" 五次，得到 **"aaaaa"**。画分隔线、搓 emoji 墙的时候好用
- `s.Replace("foo", "o", "0", -1)` —— 把 "foo" 里的 "o" 全换成 "0"，第四个参数 **-1 表示"全部替换"**。结果 **"f00"**
- `s.Replace("foo", "o", "0", 1)` —— 只替换第一个 "o"，结果 **"f0o"**

顺便说一下 strings 包还有个 `Replacer` 类型可以一次性做多对替换，比 Replace 高效，项目里真用得多。

**Split：拆分**

`s.Split("a-b-c-d-e", "-")` —— 按 "-" 拆，得到切片 `[a b c d e]`（这里 [] 里展示的是 `[]string` 的默认 Println 格式）。实际业务里处理 URL、CSV 这种分隔符数据是日常。

**ToLower / ToUpper：大小写转换**

- `s.ToLower("TEST")` → **"test"**
- `s.ToUpper("test")` → **"TEST"**

注意这俩只处理 ASCII 字母以及 Unicode 里有大小写概念的字符。中文、标点这些没大小写的原样返回。

**最后两行：len 和索引取字节**

`len("hello")` 返回 **5**——但注意，len 返回的是**字节数**，不是字符数。英文 ASCII 里一个字母一个字节，所以 5 没问题；但如果你写 `len("你好")`，返回的是 **6**（UTF-8 里一个汉字占 3 字节），不是 2。

`"hello"[1]` 返回 **101**——"e" 的 ASCII 码。Go 里对字符串用下标访问拿到的是**一个字节（byte 类型，uint8 的别名）**，不是"字符"。打印 byte 默认是数字。再强调一遍：如果字符串含中文，用 [i] 索引会把一个汉字切出三段 byte，乱码！

**字节 vs 字符这个坑**

Go 的字符串底层是 UTF-8 编码（定义：Unicode 字符集的一种变长编码方式，ASCII 字符 1 字节，常见汉字 3 字节，emoji 通常 4 字节）。要按字符处理，得转成 `[]rune`（rune 是 Go 里表示 Unicode 码点的类型，int32 别名）。这块内容下一关咱们重点再提，本关先知道有这事儿。

🏃 跑一下试试

保存为 string-functions.go，然后：

```bash
go run string-functions.go
```

预期输出：

```text
Contains:   true
Count:      2
HasPrefix:  true
HasSuffix:  true
Index:      1
Join:       a-b
Repeat:     aaaaa
Replace:    f00
Replace:    f0o
Split:      [a b c d e]
ToLower:    test
ToUpper:    TEST

Len:  5
Char: 101
```

重点看：
- `Index: 1` —— 字节位置，从 0 开始数
- `Split: [a b c d e]` —— `[]string` 的默认打印格式，不是带引号的列表
- `Char: 101` —— 这是 'e' 的 ASCII 码，证明 "hello"[1] 拿到的是 byte 不是 string
- 中间有个空行对应源码里的 `p()` 调用

💡 师兄的碎碎念

strings 包的函数师兄建议你对着官方文档 pkg.go.dev/strings 翻一遍——大部分工作里用到的都是里面那二十几个老面孔。再补几个本关没演示但特别常用的：

- **TrimSpace(s)**：砍掉首尾的空白（空格、制表符、换行），处理用户输入必备
- **Trim(s, cutset)**：砍掉首尾指定字符集合
- **Fields(s)**：按任意空白拆分，比 Split 更智能
- **EqualFold(a, b)**：不区分大小写比较两个字符串是否相等（比先 ToLower 再 == 更快更省内存）
- **Builder**：拼大串时别再 `+=` 了，用 `strings.Builder` 性能高一个数量级

最后再强调一次那个坑：Go 里字符串操作默认是**字节级**的，中文、emoji 这种多字节字符用 len、[i]、Index 都要小心。需要按"字符"处理，要么转 `[]rune`，要么用 `unicode/utf8` 包里的函数（比如 `utf8.RuneCountInString`）。

🎓 知识点清单

- strings 包是 Go 处理字符串的标准武器库
- 常用函数：Contains/Count/HasPrefix/HasSuffix/Index/Join/Repeat/Replace/Split/ToLower/ToUpper
- 包别名 `s "strings"` 可以缩短调用
- 函数值赋给变量（`var p = fmt.Println`）可以做快捷方式，demo 里好用
- len 返回字节数、`s[i]` 返回 byte，都是字节级操作
- 处理多字节字符要用 `[]rune` 或 utf8 包
- 拼大串优先用 `strings.Builder`，比 += 快一个数量级
- 官方文档 pkg.go.dev/strings 值得通读一遍

➡️ 下一关

光会调函数还不够，咱们还得学会**把东西漂亮地打印出来**。下一关 49「字符串格式化 string-formatting」带你玩转 fmt 家族的 %d / %s / %v / %+v / %q 这些占位符，让你的 Println 变成 Printf 的艺术。
