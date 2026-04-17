# 第 22 关：Embedding（师兄带你学 Go）

## 🎯 这一关你会学到

本关目标：理解 Go 的。"组合优于继承"哲学：通过嵌入把一个 struct 塞进另一个 struct，自动获得它的字段和方法，还能顺带满足接口——不用写任何 `extends`。

## 🤔 先想一个问题

拼乐高是最经典的组合思路：你想要一辆带 LED 灯的跑车？把"跑车底盘"和"灯光模块"拼到一起就行，不用从头造一辆。外卖也是这样：方便面自带"怎么煮"的说明，卤蛋自带"怎么剥"的说明，把它们装进一个饭盒，你拿到的就是一份"方便面+卤蛋"套餐，各自的能力都继承过来了，但没有谁是谁的"父类"。Go 的 Embedding 就是程序里的"拼乐高"——把一个类型嵌进另一个里，自动获得它的字段和方法。

**嵌入（embedding）是把一个类型作为字段匿名放进另一个 struct 里，外层 struct 自动获得该类型的字段和方法。Go 没有类继承，用嵌入实现代码复用。**

## 📖 看代码

```go
// Go支持对于结构体(struct)和接口(interfaces)的 _嵌入(embedding)_
// 以表达一种更加无缝的 _组合(composition)_ 类型

package main

import "fmt"

type base struct {
	num int
}

func (b base) describe() string {
	return fmt.Sprintf("base with num=%v", b.num)
}

// 一个 `container` _嵌入_ 了一个 `base`.
// 一个嵌入看起来像一个没有名字的字段
type container struct {
	base
	str string
}

func main() {

	// 当创建含有嵌入的结构体，必须对嵌入进行显式的初始化；
	// 在这里使用嵌入的类型当作字段的名字
	co := container{
		base: base{
			num: 1,
		},
		str: "some name",
	}

	// 我们可以直接在 `co` 上访问 base 中定义的字段,
	// 例如： `co.num`.
	fmt.Printf("co={num: %v, str: %v}\n", co.num, co.str)

	// 或者，我们可以使用嵌入的类型名拼写出完整的路径。
	fmt.Println("also num:", co.base.num)

	// 由于 `container` 嵌入了 `base`，因此`base`的方法
	// 也成为了 `container` 的方法。在这里我们直接在 `co` 上
	// 调用了一个从 `base` 嵌入的方法。
	fmt.Println("describe:", co.describe())

	type describer interface {
		describe() string
	}

	// 可以使用带有方法的嵌入结构来赋予接口实现到其他结构上。
	// 因为嵌入了 `base` ，在这里我们看到 `container`
	// 也实现了 `describer` 接口。
	var d describer = co
	fmt.Println("describer:", d.describe())
}
```

## 🔍 师兄给你逐行拆

### 嵌入写法：没有名字的字段

在 Go 中，结构体字段可以只写类型名，不写字段名：

```go
type base struct{ num int }

type container struct {
    base      // 嵌入，没有字段名
    str string
}
```

这就是**嵌入**（embedding）。`base` 既是类型名，也自动成为字段名。

创建实例时，必须**显式**指定嵌入类型来初始化：

```go
c := container{
    base: base{num: 1},
    str:  "xxx",
}
```

不能省略 `base:`，编译器不会自动猜测。"匿名字段"，但本质上字段名是类型名本身，访问时可以用 `c.base` 把嵌入整体拿出来。

### 字段提升：直接访问里层字段

嵌入结构体之后，外层的 `container` 可以直接通过 `co.num` 访问内层 `base.num`——Go 编译器会自动把嵌入类型的字段。向上**提升**到外层访问路径里。写 `co.base.num` 也合法，两种完全等价。

**遮盖规则**：如果 `container` 本身也定义了一个叫 `num` 的字段，外层的会遮盖内层的——`co.num` 访问的是外层，想访问内层必须用完整路径 `co.base.num`。

### 方法提升：嵌入类型的方法自动归外层所有

```go
type base struct{ id int }

func (b base) describe() string {
    return fmt.Sprintf("base id=%d", b.id)
}

type container struct {
    base
    label string
}
```

因为 `container` 嵌入了 `base`，外层结构体会自动**提升**（promote）`base` 的所有方法。因此你可以直接写：

```go
co := container{base: base{id: 1}, label: "box"}
fmt.Println(co.describe()) // base id=1
```

不需要写 `co.base.describe()`，虽然那样也合法。**嵌入类型的方法自动变成外层类型的方法**——这正是 Go 用嵌入代替继承的核心机制：想让一个类型拥有另一个类型的全部能力，嵌入它即可，无需 `extends`。

外层还可以**重写**：在 `container` 上定义同名方法，就会遮蔽 `base` 的版本：

```go
func (c container) describe() string {
    return fmt.Sprintf("container label=%s", c.label)
}
```

此时 `co.describe()` 调用的是 `container` 的版本；若仍需原版本，显式写 `co.base.describe()` 即可。

### 嵌入 + 接口：组合实现

`describer` 接口只有一个要求：实现 `describe() string` 方法。`container` 结构体本身并没有显式定义 `describe`，但它嵌入了 `base`，而 `base` 拥有 `describe` 方法。

Go 的**方法提升**机制会将 `base` 的方法自动纳入 `container` 的方法集，因此 `container` 无需任何额外代码，便已满足 `describer` 接口。赋值语句 `var d describer = co` 可以顺利编译，调用 `d.describe()` 时，实际执行的是 `base.describe()`。

```go
type describer interface {
    describe() string
}
type base struct{ name string }
func (b base) describe() string { return "base: " + b.name }
type container struct{ base }

co := container{base{"go"}}
var d describer = co   // 编译通过
fmt.Println(d.describe()) // "base: go"
```

Go 的接口满足检查完全依赖**方法集**，而嵌入正是向外层结构贡献方法的途径。"免费接口实现"是 Go 很常见的组合技巧——想给已有类型扩一个接口，写一个嵌入它的新类型，再补齐差的方法即可。

## 🏃 跑一下试试

```bash
$ go run embedding.go
co={num: 1, str: some name}
also num: 1
describe: base with num=1
describer: base with num=1
```

注意第 4 行——`container` 通过嵌入 base 自动实现了 describer 接口。

## 💡 师兄的碎碎念

- **嵌入不是继承**：Go 没有父子类、没有 `extends` 关键字；嵌入只是字段和方法的自动提升，外层和嵌入类型之间没有 is-a 关系。
- **可以嵌入接口**：`type X interface { Reader; Writer }` 把多个接口组合成新接口，标准库里 `io.ReadWriter` 就是这样写的。
- **同名方法 ambiguous**：同时嵌入多个类型且它们有同名方法时，编译器直接报 `ambiguous selector`，必须显式指定调哪个。
- **装饰器模式**：通过嵌入一个已有类型并只覆盖部分方法，你就得到了一个"带新行为"的装饰器——零样板扩展功能。

## 🎓 这一关的知识点清单

- **嵌入语法**：`type Outer struct { Inner }` —— 字段位置只写类型名，没有字段名，这就是嵌入。
- **字段提升**：嵌入后 `Outer` 的实例可以直接访问 `Inner` 的字段（`o.fieldOfInner`），也能用完整路径 `o.Inner.fieldOfInner`。
- **方法提升**：`Inner` 的方法自动变成 `Outer` 的方法，`Outer` 实例可直接调用。
- **方法遮盖**：`Outer` 定义同名方法会遮盖嵌入类型的方法；想调原版用 `o.Inner.Method()`。
- **嵌入与接口实现**：嵌入类型的方法进入外层方法集，外层可能因此自动满足某个接口。

## ➡️ 下一关

嵌入让代码组合更自然，下一关继续探索 Go 的类型系统新特性。

[下一关：Generics →](../23-generics/)
