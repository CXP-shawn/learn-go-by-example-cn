# 第 23 关：泛型（师兄带你学 Go）

## 🎯 这一关你会学到

本关目标是搞懂 Go 1.18 引入的泛型核心概念：用类型参数让函数和类型不再绑定某一具体类型；掌握 `any`（任意类型）与 `comparable`（可比较类型）两种内置约束的含义与使用场景；学会编写泛型函数、泛型类型，并理解编译器如何通过类型推断自动匹配类型参数，从而写出简洁、安全、可复用的通用代码。

## 🤔 先想一个问题

想象小区楼道里的智能外卖柜。柜子本身不在乎你放的是蛋炒饭、炸鸡汉堡还是快递包裹——它只做一件事：**存一个物品，给你一个取件码**。如果这个柜子设计时只能放蛋炒饭，那华为手机、瑞幸咖啡、顺丰快递就全没地方放了，只能各买一台专用柜，成本爆炸。

写代码也一样。没有泛型时，你得为 `int` 写一个 `MinInt`、为 `float64` 写一个 `MinFloat`、为 `string` 写一个 `MinString`……逻辑完全相同，却要复制粘贴 N 份，维护噩梦。"不绑定具体类型"的能力：写一份代码，适配多种类型。

**泛型是"类型参数化"的机制。Go 1.18 起支持用 `[T any]`、`[K comparable, V any]` 等类型参数写出对多种类型通用的函数和容器，编译时由编译器生成各具体类型的实现。**

## 📖 看代码

```go
// 从1.18版本开始，Go添加了对泛型的支持，也即类型参数。

package main

import "fmt"

// 作为泛型函数的示例，`MapKeys` 接受任意类型的Map并返回其Key的切片。
// 这个函数有2个类型参数 - `K` 和 `V`；
// `K` 是 `comparable` 类型，也就是说我们可以通过 `==` 和 `!=`
// 操作符对这个类型的值进行比较。这是Go中Map的Key所必须的。
// `V` 是 `any` 类型，意味着它不受任何限制 (`any` 是 `interface{}` 的别名类型).
func MapKeys[K comparable, V any](m map[K]V) []K {
	r := make([]K, 0, len(m))
	for k := range m {
		r = append(r, k)
	}
	return r
}

// 作为泛型类型的示例， `List` 是一个
// 具有任意类型值的单链表。
type List[T any] struct {
	head, tail *element[T]
}

type element[T any] struct {
	next *element[T]
	val  T
}

// 我们可以像在常规类型上一样定义泛型类型的方法
// 但我们必须保留类型参数。
// 这个类型是 `List[T]`，而不是 `List`
func (lst *List[T]) Push(v T) {
	if lst.tail == nil {
		lst.head = &element[T]{val: v}
		lst.tail = lst.head
	} else {
		lst.tail.next = &element[T]{val: v}
		lst.tail = lst.tail.next
	}
}

func (lst *List[T]) GetAll() []T {
	var elems []T
	for e := lst.head; e != nil; e = e.next {
		elems = append(elems, e.val)
	}
	return elems
}

func main() {
	var m = map[int]string{1: "2", 2: "4", 4: "8"}

	// 当调用泛型函数的时候, 我们经常可以使用类型推断。
	// 注意，当调用 `MapKeys` 的时候，
	// 我们不需要为 `K` 和 `V` 指定类型 - 编译器会进行自动推断
	fmt.Println("keys m:", MapKeys(m))

	// ... 虽然我们也可以明确指定这些类型。
	_ = MapKeys[int, string](m)

	lst := List[int]{}
	lst.Push(10)
	lst.Push(13)
	lst.Push(23)
	fmt.Println("list:", lst.GetAll())
}
```

## 🔍 师兄给你逐行拆

### 泛型函数：MapKeys 的两个类型参数

```go
func MapKeys[K comparable, V any](m map[K]V) []K
```

方括号 `[K comparable, V any]` 声明了两个**类型参数**，这是 Go 1.18 引入的泛型能力。

- **K comparable**：`K` 必须满足 `comparable` 约束，即支持 `==` 运算符。这是必要的，因为 map 的键本身就要求可比较。
- **V any**：`V` 没有额外限制，`any` 等同于 `interface{}`，表示任意类型均可。

有了这两个类型参数，函数体可以对任意 `map[K]V` 统一操作，无需为 `map[int]string`、`map[string]bool` 分别编写实现。

**调用方式**有两种：

```go
// 编译器自动推断 K=int, V=string
keys := MapKeys(m)

// 手动指定类型参数
keys := MapKeys[int, string](m)
```

大多数情况下编译器能从实参推断类型，手动指定主要用于消歧义或提升可读性。泛型让 Go 告别了大量重复的样板代码。

### 类型约束：comparable vs any vs 自定义接口

Go 泛型中，**约束**（constraint）的本质是接口——它告诉编译器类型参数 `T` 能执行哪些操作。

**`any`**：即 `interface{}` 的别名（Go 1.18 引入），表示所有类型均可传入。但正因为如此，编译器对 `T` 几乎一无所知，无法对其做加减乘除甚至比较，函数体的操作极为受限。

**`comparable`**：内置关键字约束，表示该类型支持 `==` 与 `!=` 运算。常用于泛型 map 的 key 类型，如 `[K comparable, V any]`。注意它不是用户可定义的接口，而是编译器特殊处理的约束。

**自定义约束**：

```go
type Numeric interface {
    ~int | ~float64
}
```

- `|` 表示类型并集：`int` 或 `float64` 均可。
- `~T` 表示「底层类型为 T」，因此 `type MyInt int` 这样的自定义类型也满足 `~int`。

写 `[T Numeric]` 后，函数体就能对 `T` 做 `+`、`-`、`*`、`/`；若写 `[T any]`，编译器会报错——因为 `any` 涵盖 map、func 等完全不支持算术的类型。约束越精确，编译器能允许的操作就越多。

### 泛型类型：List[T] 通用链表

借助泛型，我们可以定义带类型参数的结构体：

```go
type List[T any] struct {
    head, tail *element[T]
}
```

`[T any]` 声明了类型参数 `T`，约束为 `any`（即任意类型）。这样一来，同一份代码就能派生出 `List[int]`、`List[string]`、`List[myStruct]` 等多种具体类型，无需为每种类型单独编写链表。

**方法签名必须保留类型参数**。接收者写 `*List[T]`，而非 `*List`：

```go
func (lst *List[T]) Push(v T) {
    // 将 v 追加到链表尾部
}
```

参数 `v` 的类型同样是 `T`，与结构体保持一致。

使用时，在实例化阶段指定具体类型：

```go
lst := List[int]{}
lst.Push(1)
lst.Push(2)
```

Go 编译器会根据 `List[int]` 生成一个专门处理 `int` 的实例，类型安全由编译期保证，运行时无需额外的类型断言。泛型类型让通用数据结构的编写变得简洁而严谨。

### 什么时候用泛型？什么时候别用？高阶工具函数（如 `Filter`/`Map`/`Reduce`）。

**不建议用泛型**：
- 具体业务逻辑通常不需要泛型；
- 能用接口（interface）解决的问题，接口往往更简洁、灵活；
- Go 泛型并非零开销：编译器会做 GC shape stenciling，多种具体类型共享实现有一定运行时代价。

**经验法则**：先用非泛型写一版，重复几次确认"真的只有类型不同"了，再考虑重构为泛型。泛型是锤子不是目的——别为了用而用。

## 🏃 跑一下试试

```bash
$ go run generics.go
keys m: [1 2 4]
list: [10 13 23]
```

map 遍历顺序随机，`keys m:` 里的数字顺序每次运行可能不同；`list:` 里按 Push 的顺序稳定输出。

## 💡 师兄的碎碎念

- 泛型从 Go 1.18 起才正式引入，更早版本无法编译含类型参数的代码
- 约束本质是接口，可用 `|` 组合多种类型，用 `~T` 表示以 T 为 underlying type 的所有类型
- 多数情况下编译器可自动推断类型参数，调用时可省略显式的 `[int]` 等标注
- 泛型会增加阅读与维护成本，若用 `interface{}` 或具体类型能满足需求，优先不用泛型

## 🎓 这一关的知识点清单

- **类型参数**：用 `[T any]`、`[K comparable, V any]` 在函数名或类型名后声明，作用域贯穿整个定义。
- **any / comparable**：`any` 是 `interface{}` 的别名（接受所有类型）；`comparable` 是内置关键字约束，表示支持 `==`/`!=`。
- **类型约束**：约束本质是接口，可用 `|` 组合多种类型，`~T` 表示以 T 为 underlying type 的所有自定义类型。
- **泛型函数**：`func Name[T constraint](args) ret`，调用时编译器可自动推断类型参数。
- **泛型类型**：`type Name[T any] struct{}`；方法接收者必须保留类型参数（`*Name[T]`，不是 `*Name`）。

## ➡️ 下一关

泛型让代码更通用，但要控制使用边界，接下来学习 Go 惯用的错误处理方式。[下一关：Errors →](../24-errors/)
