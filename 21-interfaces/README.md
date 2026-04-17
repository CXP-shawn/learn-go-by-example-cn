# 第 21 关：接口（师兄带你学 Go）

## 🎯 这一关你会学到

本关目标：搞懂 Go 接口的定义方式，理解。接口的"隐式实现"机制（鸭子类型），并学会用接口写出一次定义、多种类型通用的函数。

## 🤔 先想一个问题

你买了一根 Type-C 充电线，标准只有两条：能插进去、支持 24W 快充。报告自己"实现了 Type-C 接口"，只要做到标准就自动被认可。Go 的接口就是这种"形状标准"：它规定一组方法要求，任何类型只要实现这些方法就自动满足接口，不用特别声明。

**接口是一组方法签名的集合，规定了类型必须有哪些方法。Go 里接口是隐式实现的——只要类型的方法集包含接口的全部方法，就自动"实现"了该接口，不需要显式声明。**

## 📖 看代码

```go
// 方法签名的集合叫做：_接口(Interfaces)_。

package main

import (
	"fmt"
	"math"
)

// 这是一个几何体的基本接口。
type geometry interface {
	area() float64
	perim() float64
}

// 在这个例子中，我们将为 `rect` 和 `circle` 实现该接口。
type rect struct {
	width, height float64
}
type circle struct {
	radius float64
}

// 要在 Go 中实现一个接口，我们只需要实现接口中的所有方法。
// 这里我们为 `rect` 实现了 `geometry` 接口。
func (r rect) area() float64 {
	return r.width * r.height
}
func (r rect) perim() float64 {
	return 2*r.width + 2*r.height
}

// `circle` 的实现。
func (c circle) area() float64 {
	return math.Pi * c.radius * c.radius
}
func (c circle) perim() float64 {
	return 2 * math.Pi * c.radius
}

// 如果一个变量实现了某个接口，我们就可以调用指定接口中的方法。
// 这儿有一个通用的 `measure` 函数，我们可以通过它来使用所有的 `geometry`。
func measure(g geometry) {
	fmt.Println(g)
	fmt.Println(g.area())
	fmt.Println(g.perim())
}

func main() {
	r := rect{width: 3, height: 4}
	c := circle{radius: 5}

	// 结构体类型 `circle` 和 `rect` 都实现了 `geometry` 接口，
	// 所以我们可以将其实例作为 `measure` 的参数。
	measure(r)
	measure(c)
}
```

## 🔍 师兄给你逐行拆

### 定义接口：只写要做什么，不关心谁做

```go
type geometry interface {
    area()  float64
    perim() float64
}
```

这段代码声明了一个名为 `geometry` 的接口类型，其中包含两个**方法签名**：`area()` 和 `perim()`，分别代表。"计算面积"和"计算周长"。

**接口只规定"要做什么"，完全不关心"谁来做、怎么做"**，这是接口抽象的核心。

类比：公司贴出招聘启事："熟练 Go、懂分布式"。启事不关心应聘者叫张三还是李四、毕业于哪所学校，只看你能不能做到这两件事。接口就是这样的"能力清单"。

### 隐式实现：没有 implements 关键字

在前面的代码里，`rect` 和 `circle` 各自定义了 `area()` 与 `perim()` 两个方法。它们"已实现"接口——**你不用写任何 `implements geometry` 这样的声明**。这是 Go 跟 Java/C# 的根本区别。

**好处**：接口可以**后加**，可以为无法修改的第三方类型定义接口适配，只要它们的方法集恰好吻合。这就是俗称的"鸭子类型"：走路像鸭子、叫起来像鸭子，那它就是鸭子——不问出身，只看行为。

### 用接口写通用函数：measure

定义好接口之后，就可以写出真正「不依赖具体类型」的函数：

```go
func measure(g geometry) {
    fmt.Println(g)
    fmt.Println(g.area())
    fmt.Println(g.perim())
}
```

参数类型是 `geometry` 接口，而不是 `rect` 或 `circle`。函数体里只管调 `g.area()` 和 `g.perim()`，完全不需要知道 `g` 的真实类型。

调用时两者都能传入：

```go
r := rect{width: 3, height: 4}
c := circle{radius: 5}
measure(r)
measure(c)
```

这就是**多态**——同一个函数签名，`rect` 和 `circle` 各自给出自己的实现，运行时 Go 按实际类型分派，调到正确的方法。

其中 `fmt.Println(g)` 会打印底层结构体的值：`{3 4}` 和 `{5}`，Go 知道接口变量背后藏着哪个具体类型，并用该类型的默认格式输出。

接口让你写一次函数，适配所有满足约定的类型，这是 Go 组合设计的核心思想。

### 彩蛋：空接口 interface{} 与类型断言

`interface{}`（Go 1.18 起有了别名 `any`）是一个**没有任何方法**的接口，因此所有类型都自动满足它——它可以接收任意值，类似其他语言的 `Object`。还原成具体类型：

```go
v, ok := g.(rect)           // 单次断言
switch v := g.(type) {      // 类型开关
case rect:   fmt.Println(v.width)
case circle: fmt.Println(v.radius)
}
```

⚠️ **别把 `any` 当万金油**：滥用它会让代码丢失静态类型检查，所有类型问题都拖到运行时才暴露。日常优先用具体接口（哪怕只带 1-2 个方法）。

## 🏃 跑一下试试

```bash
$ go run interfaces.go
{3 4}
12
14
{5}
78.53981633974483
31.41592653589793
```

`measure` 用接口参数，既能测矩形也能测圆——这就是多态。

## 💡 师兄的碎碎念

- **小接口组合**：Go 推崇"小而专"的接口，标准库里 `io.Reader`/`io.Writer` 都只有 1 个方法。用小接口组合而非一个大接口，更灵活。
- **接口可嵌套**：`type ReadWriter interface { Reader; Writer }` 把两个小接口合一，编译器把方法集拼起来。
- **nil 接口 vs 含 nil 值的接口**：这是 Go 的经典坑——一个接口变量里装了 `*rect(nil)` 并不等于 `nil 接口`；前者会让 `v == nil` 返回 false 但调方法可能 panic。
- **零值接口调用方法会 panic**：`var g geometry; g.area()` 直接崩溃，使用前先判 `g != nil`。

## 🎓 这一关的知识点清单

- **接口定义**：`type Name interface { Method() RetType }` 声明一组方法签名，不含实现。
- **隐式实现**：Go 无 `implements` 关键字，类型只要方法集包含接口要求的全部方法，就自动"实现"该接口。
- **方法集与接口关系**：值类型方法集仅含值接收者方法；指针类型方法集含值和指针接收者方法。
- **空接口 any**：`interface{}`（或 `any`）没有方法要求，所有类型都满足，相当于通用容器。
- **类型断言**：`v.(T)` 把接口还原为具体类型，失败会 panic；`v, ok := i.(T)` 带 `ok` 安全版；`i.(type)` 在 switch 里做类型分支。

## ➡️ 下一关

接口让代码面向行为而非类型，是 Go 实现解耦与测试替换的核心武器。

[下一关：Embedding →](../22-embedding/)
