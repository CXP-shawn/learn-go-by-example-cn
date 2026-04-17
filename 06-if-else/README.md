# 第 6 关：If/Else 分支（师兄带你学 Go）

## 🎯 这一关你会学到

这一关咱们学 Go 里最基础的流程控制——if/else 分支。你会搞懂怎么写条件判断、怎么在 if 里直接声明变量，以及 Go 在语法上和 C/Java 的几个关键不同点。

## 🤔 先想一个问题

你去奶茶店点单，店员会问你：「要加珍珠吗？」——你说要，她就给你加；你说不要，她就跳过这一步。整个点单流程会根据你的回答

## 📖 看代码

```go
// `if` 和 `else` 分支结构在 Go 中非常直接。

package main

import "fmt"

func main() {

 // 这里是一个基本的例子。
 if 7%2 == 0 {
 fmt.Println("7 is even")
 } else {
 fmt.Println("7 is odd")
 }

 // 你可以不要 `else` 只用 `if` 语句。
 if 8%4 == 0 {
 fmt.Println("8 is divisible by 4")
 }

 // 在条件语句之前可以有一个声明语句；在这里声明的变量可以在这个语句所有的条件分支中使用。
 if num := 9; num < 0 {
 fmt.Println(num, "is negative")
 } else if num < 10 {
 fmt.Println(num, "has 1 digit")
 } else {
 fmt.Println(num, "has multiple digits")
 }
}

// 注意，在 Go 中，条件语句的圆括号不是必需的，但是花括号是必需的。
```

## 🔍 师兄给你逐行拆

### 第一段：最基础的 if/else——奇偶判断

```go
if 7%2 == 0 {
    fmt.Println("7 is even")
} else {
    fmt.Println("7 is odd")
}
```

先说这行在干嘛。`7%2` 是取模运算，7 除以 2 余 1，所以 `7%2 == 0` 这个条件是 `false`，程序就会跳过 if 块，执行 else 块，打印出 `7 is odd`。

**条件分支（Conditional Branch）** 是程序根据某个布尔值（true/false）决定走哪条路的机制。这是所有编程语言的核心控制结构之一。

现在咱们来说说 Go 和你以前学的 C、Java、JavaScript 的几个关键区别：

**第一：条件表达式不加圆括号。** 在 C 或 Java 里你要写 `if (7%2 == 0)`，Go 里直接写 `if 7%2 == 0`，不需要括号。Go 的设计哲学是

## 🏃 跑一下试试

```bash
go run if-else.go
7 is odd
8 is divisible by 4
9 has 1 digit
```

## 💡 师兄的碎碎念

- Go 没有三目运算符（`condition ? a : b` 那种），刚从 Java/C++ 转过来的同学要注意，老老实实写完整的 if/else，别想偷懒 😅
- `if num := 9; num < 0 {}` 这个 init statement 写法超实用！比如 `if err := doSomething(); err != nil {}` 是 Go 里最常见的错误处理套路，早点熟悉
- 花括号的左花括号 `{` 必须和 if/else 在同一行，不能换行，否则编译报错——Go 的格式要求比较严，用 `gofmt` 自动格式化就不会踩坑
- else 块必须紧跟在 if 块的右花括号 `}` 后面，不能另起一行，这个和 Python 差别挺大的，新手容易写错

## 🎓 这一关的知识点清单

- **if/else 基本结构**：条件为真走 if 块，否则走 else 块，逻辑清晰无歧义
- **省略 else**：只需要在满足条件时执行某段逻辑，可以直接只写 if，不强制配 else
- **init statement（初始化语句）**：`if num := 9; num < 0 {}` 这种写法可以在 if 的作用域内声明变量，外部访问不到，变量隔离更安全
- **else if 链**：多个条件可以用 else if 串联，按顺序匹配第一个为真的分支
- **无圆括号、必须有花括号**：Go 语法强制要求花括号，但条件表达式不需要加括号，和 C/Java 习惯不同

## ➡️ 下一关

分支多了用 if/else 会显得很啰嗦，下一关我们来看看更优雅的方式 👇

[下一关：Switch 分支结构 →](../07-switch/)
