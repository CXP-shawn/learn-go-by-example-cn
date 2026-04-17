# 第 49 关：字符串格式化（师兄带你学 Go）

🎯 这一关你会学到
• Printf 家族的**格式化动词**（verb）：%v、%+v、%#v、%T、%d、%b、%c、%x、%f、%e、%s、%q、%p、%t
• 宽度和精度的控制：%6.2f、%-6s 这种带数字修饰的写法
• Printf vs Sprintf vs Fprintf 的区别
• 怎么往 stderr 输出错误信息

🤔 先想一个问题

你玩游戏的时候，背包里的物品栏是不是整整齐齐排成两列？金币数字永远靠右对齐？等级永远两位数？这些都是"**格式化输出**"在幕后出力。写代码打印日志、调试信息、表格数据的时候，如果全靠 Println 凑，那叫一个惨不忍睹。Go 的 fmt.Printf 借鉴了 C 语言的 printf，用一套**动词（verb）**语法告诉编译器"我要按这个格式摆"——啥精度、宽度、对齐方向，都能精细控制。这一关咱们把 Printf 家族挨个拆一遍。

## 📖 看代码

```go
// Go 在传统的 `printf` 中对字符串格式化提供了优异的支持。
// 这儿有一些基本的字符串格式化的任务的例子。

package main

import (
	"fmt"
	"os"
)

type point struct {
	x, y int
}

func main() {

	// Go 提供了一些用于格式化常规值的打印“动词”。
	// 例如，这样打印 `point` 结构体的实例。
	p := point{1, 2}
	fmt.Printf("struct1: %v\n", p)

	// 如果值是一个结构体，`%+v` 的格式化输出内容将包括结构体的字段名。
	fmt.Printf("struct2: %+v\n", p)

	// `%#v` 根据 Go 语法输出值，即会产生该值的源码片段。
	fmt.Printf("struct3: %#v\n", p)

	// 需要打印值的类型，使用 `%T`。
	fmt.Printf("type: %T\n", p)

	// 格式化布尔值很简单。
	fmt.Printf("bool: %t\n", true)

	// 格式化整型数有多种方式，使用 `%d` 进行标准的十进制格式化。
	fmt.Printf("int: %d\n", 123)

	// 这个输出二进制表示形式。
	fmt.Printf("bin: %b\n", 14)

	// 输出给定整数的对应字符。
	fmt.Printf("char: %c\n", 33)

	// `%x` 提供了十六进制编码。
	fmt.Printf("hex: %x\n", 456)

	// 同样的，也为浮点型提供了多种格式化选项。
	// 使用 `%f` 进行最基本的十进制格式化。
	fmt.Printf("float1: %f\n", 78.9)

	// `%e` 和 `%E` 将浮点型格式化为（稍微有一点不同的）科学记数法表示形式。
	fmt.Printf("float2: %e\n", 123400000.0)
	fmt.Printf("float3: %E\n", 123400000.0)

	// 使用 `%s` 进行基本的字符串输出。
	fmt.Printf("str1: %s\n", "\"string\"")

	// 像 Go 源代码中那样带有双引号的输出，使用 `%q`。
	fmt.Printf("str2: %q\n", "\"string\"")

	// 和上面的整型数一样，`%x` 输出使用 base-16 编码的字符串，
	// 每个字节使用 2 个字符表示。
	fmt.Printf("str3: %x\n", "hex this")

	// 要输出一个指针的值，使用 `%p`。
	fmt.Printf("pointer: %p\n", &p)

	// 格式化数字时，您经常会希望控制输出结果的宽度和精度。
	// 要指定整数的宽度，请在动词 "%" 之后使用数字。
	// 默认情况下，结果会右对齐并用空格填充。
	fmt.Printf("width1: |%6d|%6d|\n", 12, 345)

	// 你也可以指定浮点型的输出宽度，同时也可以通过 `宽度.精度` 的语法来指定输出的精度。
	fmt.Printf("width2: |%6.2f|%6.2f|\n", 1.2, 3.45)

	// 要左对齐，使用 `-` 标志。
	fmt.Printf("width3: |%-6.2f|%-6.2f|\n", 1.2, 3.45)

	// 你也许也想控制字符串输出时的宽度，特别是要确保他们在类表格输出时的对齐。
	// 这是基本的宽度右对齐方法。
	fmt.Printf("width4: |%6s|%6s|\n", "foo", "b")

	// 要左对齐，和数字一样，使用 `-` 标志。
	fmt.Printf("width5: |%-6s|%-6s|\n", "foo", "b")

	// 到目前为止，我们已经看过 `Printf` 了，
	// 它通过 `os.Stdout` 输出格式化的字符串。
	// `Sprintf` 则格式化并返回一个字符串而没有任何输出。
	s := fmt.Sprintf("sprintf: a %s", "string")
	fmt.Println(s)

	// 你可以使用 `Fprintf` 来格式化并输出到 `io.Writers` 而不是 `os.Stdout`。
	fmt.Fprintf(os.Stderr, "io: an %s\n", "error")
}
```

🔍 师兄给你逐行拆

**啥是"动词"（verb）？**

Go 的 fmt 包里，**动词（verb）**指的是格式化字符串里以 `%` 开头的占位符，比如 `%d`、`%s`、`%v`。每个动词代表"**把对应位置的参数按某种方式打印出来**"。你可以把动词想象成游戏里的装备槽：位置固定，槽位类型决定能塞什么、会显示啥效果。本关一口气介绍了十几个动词，咱们分门别类拆。

**结构体系列：%v / %+v / %#v / %T**

先看开头：
```go
p := point{1, 2}
fmt.Printf("struct1: %v\n", p)   // struct1: {1 2}
fmt.Printf("struct2: %+v\n", p)  // struct2: {x:1 y:2}
fmt.Printf("struct3: %#v\n", p)  // struct3: main.point{x:1, y:2}
fmt.Printf("type: %T\n", p)      // type: main.point
```

- `%v` 是"默认格式（value）"：结构体打印字段值，大括号里用空格分开
- `%+v` 在 %v 基础上**加上字段名**，调试非常好用
- `%#v` 打印 Go **源码表示**——你直接把它贴到代码里就能编译过的那种形式，包括包名
- `%T` 打印参数的**类型（Type）**

师兄实战建议：**写日志调试时用 %+v**，看到字段名能省不少脑子。

**布尔、整数系列：%t / %d / %b / %c / %x**

- `%t` → true / false（t 取自 truth）
- `%d` → 十进制整数，最常用
- `%b` → 二进制（binary），`%b` 打印 14 得 `1110`
- `%c` → 按整数值对应的 **Unicode 码点打印字符**，33 是 Unicode U+0021，也就是感叹号 `!`
- `%x` → 十六进制（hex），456 → `1c8`

**浮点数系列：%f / %e / %E**

- `%f` → 普通十进制小数，78.9 默认打 `78.900000`（小数点后 6 位）
- `%e` → 科学记数法小写，`1.234e+08`
- `%E` → 科学记数法大写，`1.234E+08`

**字符串系列：%s / %q / %x / %p**

- `%s` → 原样打印字符串。源码里传的是 `"\"string\""`，也就是包含两个双引号的字面值 `"string"`，打出来就是 `"string"`
- `%q` → **加引号带转义**，变成 Go 源码形式，把 `"string"` 打成 `"\"string\""`
- `%x` 对字符串 → 每个字节打两位十六进制，`"hex this"` 变成 `6865782074686973`
- `%p` → 打印**指针地址**，形如 `0xc0000140a0`

**宽度 & 精度控制**

这是 Printf 最有用的本事之一：

- `%6.2f` → **宽度 6、精度 2** 的浮点数。宽度指"总共至少占 6 个字符位"，精度指"小数点后 2 位"。1.20 变成 `"  1.20"`（前面补两个空格到 6 个字符宽）
- `%-6.2f` → 加 `-` 变**左对齐**，1.20 变成 `"1.20  "`（后面补空格）
- `%6s` → 字符串宽度 6 右对齐，`"foo"` 变成 `"   foo"`
- `%-6s` → 字符串宽度 6 左对齐，`"foo"` 变成 `"foo   "`

这套控制法律师在打印表格时极其好用，比自己一个个算空格强多了。

**Printf vs Sprintf vs Fprintf**

fmt 包里打印有三兄弟，区别在"往哪儿打"：

- `fmt.Printf(...)` → 打到 **os.Stdout**（标准输出，也就是终端默认那个）
- `fmt.Sprintf(...)` → **不打印**，而是**返回字符串**，你拿到后可以自己处理
- `fmt.Fprintf(w, ...)` → 打到你指定的 `io.Writer`（**io.Writer 是标准库里一个接口，任何实现了 Write(p []byte) (n int, err error) 方法的类型都算**），常见的写入目标：os.Stdout、os.Stderr、os.File、bytes.Buffer、http.ResponseWriter 都算。

代码里最后两行：

```go
s := fmt.Sprintf("sprintf: a %s", "string")
fmt.Println(s)
fmt.Fprintf(os.Stderr, "io: an %s\n", "error")
```

第一行用 Sprintf 格式化成字符串 `"sprintf: a string"` 赋给 s，再 Println 出去。第二行 Fprintf 写到 `os.Stderr`——**标准错误输出**，它和 stdout 是两个独立的流，常用来输出错误信息，在终端里一般都会显示，但可以被单独重定向（比如 `go run prog > out.log 2> err.log`）。

**总结：动词记忆口诀**

- v 通用、+v 带名、#v 源码、T 类型
- d 十进制、b 二进制、x 十六进制、c 字符、t 布尔
- f 浮点、e/E 科学记数法
- s 字符串、q 加引号、p 指针
- 数字控制宽度和精度，带 - 左对齐

🏃 跑一下试试

保存为 string-formatting.go，然后：

```bash
go run string-formatting.go
```

预期输出（顺序和代码里 Printf 调用顺序完全一致）：

```text
struct1: {1 2}
struct2: {x:1 y:2}
struct3: main.point{x:1, y:2}
type: main.point
bool: true
int: 123
bin: 1110
char: !
hex: 1c8
float1: 78.900000
float2: 1.234e+08
float3: 1.234E+08
str1: "string"
str2: "\"string\""
str3: 6865782074686973
pointer: 0xc0000140a0
width1: |    12|    345|
width2: |  1.20|  3.45|
width3: |1.20  |3.45  |
width4: |   foo|     b|
width5: |foo   |b     |
sprintf: a string
io: an error
```

注意：
- `char: !` —— 33 是感叹号的 Unicode 码点
- `pointer:` 那行的具体地址**每次运行都不一样**，因为堆栈上的地址是动态分配的
- 最后一行 `io: an error` 实际会去到 stderr；在终端里通常能看到，但如果你重定向了 stderr 就看不到了

💡 师兄的碎碎念

Printf 系列用起来爽，但有几个坑你要知道：

1. **%d 对应的参数类型不对会怎样？** 不会编译错，runtime 时 fmt 会打印 `%!d(string=...)` 之类的东西。Go 的 fmt 是**反射驱动**的，能宽容但会难看。

2. **Sprintf 比字符串拼接慢**。性能敏感场景下循环里别用 Sprintf 拼大串，用 `strings.Builder` 更快。

3. **%v 对嵌套复杂结构体可能信息不全**。调试时用 `fmt.Printf("%+v\n", obj)` 或 `fmt.Printf("%#v\n", obj)`，甚至用 `spew` 第三方库。

4. **想打日志用 log 包而不是 fmt**。log 带时间戳、前缀、可配置输出位置。更进阶的用 zap / slog（Go 1.21+ 标准库内置 slog）。

5. **国际化数字（千位分隔符、本地化小数点）**：fmt 本身不支持，想要用 `golang.org/x/text/message` 里的 Printer。

🎓 知识点清单

- Printf 用动词（%开头的占位符）按位填参数
- 结构体打印：%v / %+v / %#v / %T 挑需要的用
- 数字：%d / %b / %x / %f / %e / %E / %c，按进制/精度挑
- 字符串：%s 普通、%q 加引号转义、%x 按字节打 hex
- 指针：%p
- 宽度和精度：`%[-]宽度.精度动词`，- 表示左对齐
- Printf 输出到 stdout、Sprintf 返回字符串、Fprintf 写到任意 io.Writer
- os.Stderr 是独立的错误流，可单独重定向
- 日志场景优先用 log / slog，不要裸 fmt

➡️ 下一关

会用 Printf 之后，如果要填复杂的模板——比如邮件正文、HTML 片段、配置文件模板——硬拼会累死。下一关 50「文本模板 text-templates」教你用 Go 原生的 text/template 优雅地填坑，走起！
