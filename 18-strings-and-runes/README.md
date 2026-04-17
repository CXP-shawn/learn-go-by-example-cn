# 第 18 关：字符串和 rune 类型（师兄带你学 Go）

## 🎯 这一关你会学到

这一关你会彻底搞懂 Go 字符串的本质——它是**只读的 UTF-8 字节切片**，`len(s)` 返回的是字节数而非字符数。你还会掌握 `rune` 类型的含义：它代表一个 Unicode 码点，也就是人眼看到的一个字符。

## 🤔 先想一个问题

想象你去奶茶店取餐，收银小票上打印着订单号 `AB奶茶`。小票的打印机按**字节**计费：英文字母 `A`、`B` 各占 1 个字节，而汉字 `奶`、`茶` 在 UTF-8 编码下各占 3 个字节。

**Go 里字符串是只读的 UTF-8 字节序列 `[]byte`；`rune` 是 `int32` 的别名，代表一个 Unicode 码点（即人眼看到的一个"字符"）。**

## 📖 看代码

```go
// Go语言中的字符串是一个只读的byte类型的切片。
// Go语言和标准库特别对待字符串 - 作为以
// [UTF-8](https://en.wikipedia.org/wiki/UTF-8) 为编码的文本容器。
// 在其他语言当中， 字符串由"字符"组成。
// 在Go语言当中，字符的概念被称为 `rune` - 它是一个表示
// Unicode 编码的整数。
// [这个Go博客](https://go.dev/blog/strings) 很好的介绍了这个主题。

package main

import (
	"fmt"
	"unicode/utf8"
)

func main() {

	// `s` 是一个 `string` 分配了一个 literal value
	// 表示泰语中的单词 "hello" 。
	// Go 字符串是 UTF-8 编码的文本。
	const s = "สวัสดี"

	// 因为字符串等价于 `[]byte`，
	// 这会产生存储在其中的原始字节的长度。
	fmt.Println("Len:", len(s))

	// 对字符串进行索引会在每个索引处生成原始字节值。
	// 这个循环生成构成`s`中 Unicode 的所有字节的十六进制值。
	for i := 0; i < len(s); i++ {
		fmt.Printf("%x ", s[i])
	}
	fmt.Println()

	// 要计算字符串中有多少rune，我们可以使用`utf8`包。
	// 注意`RuneCountInString`的运行时取决于字符串的大小。
	// 因为它必须按顺序解码每个 UTF-8 rune。
	// 一些泰语字符由多个 UTF-8 code point 表示，
	// 所以这个计数的结果可能会令人惊讶。
	fmt.Println("Rune count:", utf8.RuneCountInString(s))

	// `range` 循环专门处理字符串并解码每个 `rune` 及其在字符串中的偏移量。
	for idx, runeValue := range s {
		fmt.Printf("%#U starts at %d\n", runeValue, idx)
	}

	// 我们可以通过显式使用 `utf8.DecodeRuneInString` 函数来实现相同的迭代。
	fmt.Println("\nUsing DecodeRuneInString")
	for i, w := 0, 0; i < len(s); i += w {
		runeValue, width := utf8.DecodeRuneInString(s[i:])
		fmt.Printf("%#U starts at %d\n", runeValue, i)
		w = width

		// 这演示了将 `rune` value 传递给函数。
		examineRune(runeValue)
	}
}

func examineRune(r rune) {

	// 用单引号括起来的值是 _rune literals_.
	// 我们可以直接将 `rune` value 与 rune literal 进行比较。
	if r == 't' {
		fmt.Println("found tee")
	} else if r == 'ส' {
		fmt.Println("found so sua")
	}
}
```

## 🔍 师兄给你逐行拆

### string 的本质是 UTF-8 字节

```go
const s = "สวัสดี"
fmt.Println(len(s)) // 18，不是 6
```

这段泰语「hello」只有 6 个字符，`len` 却返回 18。原因很简单：Go 的 `string` 底层就是一段 UTF-8 字节序列，`len` 统计的是**字节数**，而非字符数。泰语每个字符在 UTF-8 下占 3 个字节，6 × 3 = 18。

把 `string` 想象成 `[]byte`，`len` 就是在数字节，和你直觉里「数字数」的操作完全不同。

**与 Java 的区别**：Java 的 `String.length()` 返回的是 `char` 个数（UTF-16 码元），同样的泰语字符串会返回 6。Go 不做这层封装，字节就是字节，更透明，但需要你自己留意。

**类比**：快递单上填「北京朝阳区」只有 5 个字，写起来很快；但打印机实际喷的像素远比 5 个英文字母多。`len` 量的是像素（字节），不是字数（字符）。

想正确遍历字符，请用 `range`：

```go
for i, r := range s {
    fmt.Printf("%d → %c\n", i, r) // r 是 rune，即真正的字符
}
```

### 按索引取字符：其实是取字节

用下标直接遍历字符串，拿到的是**字节**，不是字符：

```go
s := "สวัสดี"
for i := 0; i < len(s); i++ {
    fmt.Printf("%x ", s[i])
}
// 输出：e0 b8 aa e0 b8 a7 ...
```

输出是一堆十六进制数，每个数字代表一个 `byte`（`uint8`）。

**坑在哪里？** `s[0]` 得到的是第 1 个字节 `0xe0`，而泰语字符 `ส` 实际占 3 个字节。

### 数字符要用 utf8.RuneCountInString 或 for range

`len(s)` 返回的是**字节数**，对多字节语言毫无意义。要数字符，应使用 `utf8.RuneCountInString(s)`，它返回字符串中 Unicode 码点（rune）的个数。例如一段包含 6 个泰语字符的字符串，该函数会返回 `6`，而 `len` 可能返回 18。

```go
import "unicode/utf8"

s := "สวัสดี"
fmt.Println(utf8.RuneCountInString(s)) // 6
```

`for idx, runeValue := range s` 会自动沿 UTF-8 边界将字符串切分为 rune，并给出每个 rune 的**起始字节位置** `idx`。因为泰语每个码点占 3 字节，相邻的 `idx` 之差通常为 3。

```go
for idx, r := range s {
    fmt.Printf("idx=%d rune=%c\n", idx, r)
}
```

⚠️ 泰语存在**组合字符**（combining mark）：一个

### 手动解码：utf8.DecodeRuneInString

更底层的姿势是用 `utf8.DecodeRuneInString`：

```go
import "unicode/utf8"

s := "Go语言"
for i := 0; i < len(s); {
    r, size := utf8.DecodeRuneInString(s[i:])
    fmt.Printf("%#U 占 %d 字节\n", r, size)
    i += size
}
```

`r` 是当前位置的 rune，`size` 是它占了几个字节；每次 `i += size` 就跳到下一个字符的起点。

**什么时候用它？** 手写解析器、需要同时拿到字节偏移量和字符值、或者做边界检查时，这个函数能给你最精细的控制权。平常遍历字符串，`for range` 已经够用——Go 会自动替你解码 UTF-8，无需操心。

两个调试利器记一下：`%x` 打印原始字节（看编码长啥样），`%#U` 打印 Unicode 码点（如 `U+8BED '语'`），排查乱码问题时非常顺手。

## 🏃 跑一下试试

```bash
$ go run strings-and-runes.go
Len: 18
e0 b8 aa e0 b8 a7 e0 b8 b1 e0 b8 aa e0 b8 94 e0 b8 b5
Rune count: 6
U+0E2A 'ส' starts at 0
U+0E27 'ว' starts at 3
U+0E31 'ั' starts at 6
U+0E2A 'ส' starts at 9
U+0E14 'ด' starts at 12
U+0E35 'ี' starts at 15

Using DecodeRuneInString
U+0E2A 'ส' starts at 0
found so sua
U+0E27 'ว' starts at 3
U+0E31 'ั' starts at 6
U+0E2A 'ส' starts at 9
found so sua
U+0E14 'ด' starts at 12
U+0E35 'ี' starts at 15
```

## 💡 师兄的碎碎念

- **`len(s)` 返回字节数**：对 UTF-8 多字节字符串，`len(s)` 得到的是字节总数而非字符个数，需用 `utf8.RuneCountInString(s)` 获取真实字符数。
- **字符串不可变**：string 底层字节不可修改，若需按字符编辑，应先转为 `[]rune`，修改后再用 `string(runes)` 转回。
- **rune 本质是 int32**：`type rune = int32`，可直接参与整数运算，也可用 `%c` 格式化为字符、`%U` 格式化为 Unicode 码点。
- **拼接用 strings.Builder**：循环中用 `+` 拼接字符串每次都会分配新内存，改用 `strings.Builder` 的 `WriteRune`/`WriteString` 可显著减少内存分配，提升性能。

## 🎓 这一关的知识点清单

- **string 本质**：Go 的 string 是只读字节切片，底层存储 UTF-8 编码的原始字节，不是字符数组。
- **rune**：`rune` 是 `int32` 的别名，代表一个 Unicode 码点，可完整表示任意 Unicode 字符。
- **byte**：`byte` 是 `uint8` 的别名，`len(s)` 返回的是字节数，对多字节字符（如泰文）字节数 > 字符数。
- **UTF-8 编码**：Go 源文件默认 UTF-8，一个非 ASCII 字符可能占 2–4 个字节，需用 `utf8.DecodeRuneInString` 正确解码。
- **for range 遍历**：`for i, r := range s` 自动按 rune 解码，`i` 是字节偏移，`r` 是对应的 rune 值。

## ➡️ 下一关

掌握字符串与 rune 的关系后，下一关学习如何用结构体组织复合数据。[下一关：Structs →](../19-structs/)
